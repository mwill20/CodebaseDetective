# Bug Audit — BackdoorBox

**Analyst:** Claude Code (claude-sonnet-4-6)  
**Date:** 2026-05-25  
**Scope:** `core/attacks/base.py`, `core/attacks/BadNets.py`, `core/attacks/__init__.py`, `core/defenses/ShrinkPad.py`  
**Method:** Direct source file reading and line-by-line verification. No static analysis tools used. All findings verified against actual file content before being recorded.

---

## Verification Policy

Each finding is classified as one of:
- **VERIFIED-REAL** — confirmed by reading the exact line in the actual file
- **VERIFIED-ABSENT** — confirmed not present after explicit search
- **HALLUCINATED** — not found in the actual file; would have been invented by the AI

All 6 findings in this audit are VERIFIED-REAL. No hallucinated bugs are reported.

---

## BUG-01 — Critical: Missing `nn` Import in ShrinkPad Multi-GPU Branch

| Field | Value |
|-------|-------|
| **Severity** | Critical |
| **File** | `core/defenses/ShrinkPad.py` |
| **Line** | 152 |
| **Status** | VERIFIED-REAL |
| **Trigger condition** | `schedule['GPU_num'] > 1` in `ShrinkPad.predict()` |
| **Error type** | `NameError: name 'nn' is not defined` |

### Evidence

ShrinkPad.py import block (lines 1-7):
```python
import os
from copy import deepcopy
import torch
import torchvision.transforms as transforms
from .base import Base
from ..utils import test
# torch.nn is NOT imported as nn
```

Line 152 in `predict()` method:
```python
model = nn.DataParallel(model.cuda(), device_ids=gpus, output_device=gpus[0])
#        ^^ NameError — 'nn' was never defined
```

### Why it matters

Any researcher calling `ShrinkPad.predict()` with `schedule['GPU_num'] > 1` hits an immediate crash. Single-GPU and CPU paths work because they don't reach line 152. Most researchers in the lab use single-GPU setups, which is why this has gone undetected.

### Fix

```python
import torch.nn as nn  # add to imports at top of ShrinkPad.py
```

---

## BUG-02 — High: Self-Referential Error Message Hides GPU Misconfiguration

| Field | Value |
|-------|-------|
| **Severity** | High |
| **File** | `core/attacks/base.py` |
| **Line** | 172 |
| **Status** | VERIFIED-REAL |
| **Trigger condition** | `CUDA_SELECTED_DEVICES` contains a GPU not in `CUDA_VISIBLE_DEVICES` |
| **Error type** | `ValueError` fires correctly, but message is misleading |

### Evidence

Lines 169-172:
```python
CUDA_VISIBLE_DEVICES_SET = set(CUDA_VISIBLE_DEVICES_LIST)
CUDA_SELECTED_DEVICES_SET = set(CUDA_SELECTED_DEVICES_LIST)
if not (CUDA_SELECTED_DEVICES_SET <= CUDA_VISIBLE_DEVICES_SET):
    raise ValueError(f'CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!')
    #                   ^^^^^^^^^^^^^^^^^^^                       ^^^^^^^^^^^^^^^^^^^
    #                   both say CUDA_VISIBLE_DEVICES — should say CUDA_SELECTED_DEVICES for the first one
```

### Why it matters

When a user specifies a GPU that isn't visible to the process, they get: *"CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!"* This is a self-referential tautology. The user cannot determine which variable is wrong from the message alone. This is the equivalent of a SIEM alert that fires correctly but has a description field reading "this alert should not fire when this alert fires."

Same bug exists at line 370 in `Base.test()`.

### Fix

```python
raise ValueError(f'CUDA_SELECTED_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!')
```

---

## BUG-03 — High: Latent AttributeError in Benign Training Path

| Field | Value |
|-------|-------|
| **Severity** | High |
| **File** | `core/attacks/base.py` |
| **Line** | 234 |
| **Status** | VERIFIED-REAL |
| **Trigger condition** | A `Base` subclass uses `benign_training=True` without setting `self.poisoned_train_dataset` |
| **Error type** | `AttributeError: 'SubclassName' object has no attribute 'poisoned_train_dataset'` |

### Evidence

Line 234 (inside the training loop, reached regardless of `benign_training` flag):
```python
msg = time.strftime("[%Y-%m-%d_%H:%M:%S] ", time.localtime()) + \
      f"Epoch:{i+1}/{self.current_schedule['epochs']}, " \
      f"iteration:{batch_id + 1}/{len(self.poisoned_train_dataset)//self.current_schedule['batch_size']}, " \
      #                                ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
      # always referenced, even when benign_training=True
```

### Why it doesn't crash in normal usage

`BadNets.__init__()` always calls `CreatePoisonedDataset()` twice — once for train, once for test — so `self.poisoned_train_dataset` is always set. A developer creating a new attack subclass that conditionally creates the poisoned dataset would not realize this assumption exists until the bug fires during a training run.

### Latency risk

The bug doesn't fire at construction time. It fires at the first `log_iteration_interval` (e.g., iteration 100). A training run of 200 epochs on CIFAR-10 takes hours. The error appears after hours of compute, not at startup.

### Fix

```python
# Replace line 234 with:
dataset_size = len(self.train_dataset if self.current_schedule['benign_training'] else self.poisoned_train_dataset)
msg = time.strftime("[%Y-%m-%d_%H:%M:%S] ", time.localtime()) + \
      f"Epoch:{i+1}/{self.current_schedule['epochs']}, " \
      f"iteration:{batch_id + 1}/{dataset_size//self.current_schedule['batch_size']}, " \
      ...
```

---

## BUG-04 — Medium: Windows-Incompatible Timestamp in Directory Names

| Field | Value |
|-------|-------|
| **Severity** | Medium |
| **File** | `core/attacks/base.py` |
| **Lines** | 128 (train), 339 (test) |
| **Status** | VERIFIED-REAL |
| **Trigger condition** | Any `train()` or `test()` call on Windows |
| **Error type** | `OSError: [WinError 123] The filename, directory name, or volume label syntax is incorrect` |

### Evidence

Line 128:
```python
work_dir = osp.join(
    self.current_schedule['save_dir'],
    self.current_schedule['experiment_name'] + '_' + time.strftime("%Y-%m-%d_%H:%M:%S", time.localtime())
)
os.makedirs(work_dir, exist_ok=True)
```

The format `%H:%M:%S` produces a string like `14:32:01`. On Windows, colons are reserved characters in file paths (used as drive letter separators: `C:\`). The resulting path `experiments/train_poisoned_2026-05-25_14:32:01/` is illegal on Windows before `os.makedirs()` even runs.

Same pattern at line 339 in `test()`.

### Why it hasn't been caught

BackdoorBox is used primarily on Linux GPU clusters. Windows research environments are rare for this kind of compute. The bug would surface immediately on any Windows-based CI or development setup.

### Fix

```python
# Change %H:%M:%S to %H-%M-%S in both occurrences
time.strftime("%Y-%m-%d_%H-%M-%S", time.localtime())
```

---

## BUG-05 — Low: Dead Import from Python AST Module

| Field | Value |
|-------|-------|
| **Severity** | Low |
| **File** | `core/attacks/__init__.py` |
| **Line** | 1 |
| **Status** | VERIFIED-REAL |
| **Trigger condition** | Always present (import-time), no crash |
| **Error type** | None (harmless) |

### Evidence

`core/attacks/__init__.py`, line 1:
```python
from ast import Import  # line 1
from .BadNets import BadNets
from .Blended import Blended
# ...
```

`ast.Import` is a node class from Python's Abstract Syntax Tree module. It represents an `import` statement when parsing Python source code. It has nothing to do with the BackdoorBox attacks package.

### Why it exists

This is almost certainly a development artifact from an attempt to use Python's `ast` module for code generation or dynamic import introspection. The idea was abandoned but the import was never removed.

### Why it matters (low but real)

The import is confusing to anyone reading the file. `from ast import Import` at the top of an attacks package looks like it might affect how imports work. It doesn't, but it creates a mental parsing burden. Any linter (flake8, pylint) would flag this as `F401: 'ast.Import' imported but unused`.

### Fix

Remove line 1 entirely.

---

## BUG-06 — Low: Debug Artifact and Open TODO in ShrinkPad

| Field | Value |
|-------|-------|
| **Severity** | Low |
| **File** | `core/defenses/ShrinkPad.py` |
| **Lines** | 91 (breakpoint), 152-153 (TODO + pass) |
| **Status** | VERIFIED-REAL |
| **Trigger condition** | None (code noise only) |
| **Error type** | None (harmless) |

### Evidence

Line 91 in `ShrinkPad.predict()`:
```python
# breakpoint()
```

Lines 152-153:
```python
model = nn.DataParallel(model.cuda(), device_ids=gpus, output_device=gpus[0])
# TODO: DDP training
pass
```

### Why it matters

The `# breakpoint()` is a debugging artifact that was commented out but not deleted. In a production codebase this would be caught by linting. The `# TODO: DDP training` comment points to known technical debt (migrating to DistributedDataParallel for better multi-GPU performance) that has never been tracked in an issue. The `pass` statement after `nn.DataParallel` is also redundant — the statement before it is not conditional.

### Fix

Remove line 91. Convert the TODO to a tracked GitHub issue. Remove the `pass`.

---

## Summary Table

| ID | Severity | File | Line(s) | Description | Verified |
|----|---------|------|---------|-------------|---------|
| BUG-01 | Critical | `ShrinkPad.py` | 152 | Missing `nn` import crashes multi-GPU ShrinkPad | VERIFIED-REAL |
| BUG-02 | High | `base.py` | 172, 370 | Wrong variable name in GPU error message | VERIFIED-REAL |
| BUG-03 | High | `base.py` | 234 | Latent AttributeError during benign training in subclasses | VERIFIED-REAL |
| BUG-04 | Medium | `base.py` | 128, 339 | Colons in timestamp crash `os.makedirs` on Windows | VERIFIED-REAL |
| BUG-05 | Low | `attacks/__init__.py` | 1 | Dead `from ast import Import` | VERIFIED-REAL |
| BUG-06 | Low | `ShrinkPad.py` | 91, 152-153 | Commented-out breakpoint + open TODO | VERIFIED-REAL |

**Hallucinated bugs:** 0  
**Total verified real bugs:** 6

---

## Audit Methodology Notes

All bugs in this report were found through direct source file reading, not inference or pattern-matching from documentation. Verification steps:

1. Read the full file containing the suspected bug
2. Locate the exact line
3. Confirm the issue is reproducible by tracing the execution path
4. Confirm the fix resolves the specific issue without introducing side effects
5. Check whether the bug is already documented anywhere in the codebase (README, CLAUDE.md, comments)

BUG-04 was discovered in this session itself — the analysis machine is running Windows, which makes the timestamp crash immediately reproducible.
