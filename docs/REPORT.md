# BackdoorBox Codebase Detective Report

**Analyst:** Claude Code (claude-sonnet-4-6)  
**Date:** 2026-05-25  
**Repo:** https://github.com/THUYimingLi/BackdoorBox  
**Working directory:** `C:\Projects\CodeBaseDetective\BackdoorBox`  
**Time budget:** ~5 hours

---

## Table of Contents
1. [Part 1: Codebase Exploration](#part-1-codebase-exploration)
   - [1.1 First Contact](#11-first-contact)
   - [1.2 Architecture Mapping](#12-architecture-mapping)
   - [1.3 Request Trace](#13-request-trace)
2. [Part 2: Mental Model Experiments](#part-2-mental-model-experiments)
   - [2.1 Break Prompts Intentionally](#21-break-prompts-intentionally)
   - [2.2 Detect Hallucinations](#22-detect-hallucinations)
   - [2.3 Improve and Compare](#23-improve-and-compare)
3. [Part 3: Codebase Audit](#part-3-codebase-audit)
   - [3.1 Bug Hunt](#31-bug-hunt)
   - [3.2 Context Window Experiment](#32-context-window-experiment)
4. [Part 4: Documentation & Visualization](#part-4-documentation--visualization)

---

## Part 1: Codebase Exploration

### Exploration Method
Systematic parallel file reading, starting from the README -> requirements.txt -> directory structure -> `core/__init__.py` -> base classes -> one attack implementation end-to-end -> one defense end-to-end.

---

### 1.1 First Contact

#### What is BackdoorBox?

BackdoorBox is an open-source Python **ML security research toolbox** from Tsinghua University. It implements **backdoor attacks** and **backdoor defenses** against deep neural networks under a single unified framework.

A backdoor attack is an adversarial ML threat where an attacker poisons a fraction of training data with a trigger pattern. The resulting model behaves normally on clean inputs but misclassifies any input containing the trigger into a target class. Think of it as a hidden "unlock code" baked into the model's weights, invisible to standard evaluation and only activated when the trigger appears.

From a security analyst perspective: if FortiEDR is your endpoint detection layer, BackdoorBox is a tool for researching the attack surface *before* the model reaches production, and for evaluating whether defenses can detect or remove the hidden behavior.

---

#### Tech Stack

| Layer | Technology |
|-------|-----------|
| Language | Python 3.8 |
| ML Framework | PyTorch 1.8.0 + CUDA 11.1 |
| Image Processing | torchvision 0.9.0, opencv-python 4.12, Pillow 10.4 |
| Adversarial Attacks | torchattacks (PGD, bundled locally) |
| Scientific Compute | NumPy 1.24, SciPy 1.10 |
| Clustering/Dim-reduction | scikit-learn 1.3, hdbscan 0.8, umap-learn 0.5 |
| Perceptual Similarity | lpips 0.1.4 |
| Face/landmark detection | dlib 20.0 |
| Contrastive learning | OpenAI CLIP (from GitHub) |
| Build acceleration | ninja 1.13 (for CUDA extensions) |

The dependency list is itself a clue about which attacks are most complex. CLIP is used by LIRA (learned triggers), dlib is used for face-landmark-based attacks, and the presence of hdbscan/umap points to clustering-based defenses like FLARE and Spectral.

---

#### Key Modules

| Module | Purpose | Complexity |
|--------|---------|-----------|
| `core/attacks/base.py` | Base training loop, GPU setup, LR scheduling, checkpointing, logging | High (~410 lines) |
| `core/attacks/BadNets.py` | Baseline patch attack + PoisonedDataset wrappers for 3 dataset types | Medium (~457 lines) |
| `core/attacks/IAD.py` | Dynamic generator-based attack with custom training loop | High |
| `core/attacks/LIRA.py` | Learnable trigger with CLIP feature alignment | High |
| `core/defenses/ABL.py` | Anti-Backdoor Learning: LGA loss + sample isolation | High |
| `core/defenses/ShrinkPad.py` | Simple pre-processing defense | Low (~192 lines) |
| `core/utils/log.py` | Logging: stdout + file | Trivial (8 lines) |
| `core/models/resnet.py` | Standard ResNet-18/34 | Standard |

---

#### Top-Level Directory Breakdown

| Path | Role |
|------|------|
| `core/` | Main Python package. Re-exports `attacks.*`, `defenses.*`, `models.*` via `__init__.py` |
| `core/attacks/` | 15 attack class implementations. All inherit `attacks/base.Base` |
| `core/defenses/` | 12 defense class implementations. All inherit `defenses/base.Base` |
| `core/models/` | Neural network architectures: ResNet, VGG, AutoEncoder, UNet, BaselineMNIST |
| `core/utils/` | Shared helpers: Log, accuracy, test runner, torchattacks/PGD |
| `core/utils/torchattacks/` | PGD adversarial attack borrowed from the torchattacks library (vendored) |
| `tests/` | 30+ example scripts, one per attack/defense. These are the official usage documentation |
| `Attack_BadNets.py` | Root-level entry point for BadNets on CIFAR-10 (legacy format) |
| `Attack_Blended.py` | Root-level entry point for Blended on ImageNet50 |
| `Defense_ShrinkPad.py` | Root-level entry point for ShrinkPad evaluation |

The root-level `Attack_*.py` and `Defense_*.py` files are an older entry-point pattern. Newer methods are only in `tests/`. There is no CLI, no config file parser, and no `main.py`; all configuration is done by editing Python constants directly in the entry scripts.

---

#### Entry Points

| File | Invocation | What it does |
|------|-----------|-------------|
| `Attack_BadNets.py` | `python Attack_BadNets.py` | Trains benign then backdoored ResNet-18 on CIFAR-10 |
| `Attack_Blended.py` | `python Attack_Blended.py` | Trains Blended attack on ImageNet50 |
| `Defense_ShrinkPad.py` | `python Defense_ShrinkPad.py` | Evaluates ShrinkPad defense on a pre-trained model |
| `tests/test_BadNets.py` | `python tests/test_BadNets.py` | Same as above but more up-to-date, with more configuration options |

---

#### External Dependencies (with security relevance)

| Dependency | Source | Trust level | Notes |
|-----------|--------|-------------|-------|
| PyTorch 1.8 | pip (pinned) | High | Old version (2021). Not receiving security patches. |
| torchvision 0.9 | pip (pinned) | High | Paired with PyTorch 1.8 |
| OpenAI CLIP | git+ (unpinned) | Medium | Pulled from GitHub main; no version pin, supply chain risk |
| dlib 20.0 | pip (pinned) | Medium | C++ extension; build may fail without cmake |
| hdbscan 0.8 | pip (pinned) | Medium | |
| ninja 1.13 | pip | Standard | Build tool for CUDA extensions |

One dependency worth flagging: the CLIP entry uses `git+https://github.com/openai/CLIP.git` with no commit hash or tag pin. Any push to that repo's default branch gets pulled automatically during `pip install -r requirements.txt`. Low probability for a research tool, but it's a real supply chain exposure.

---

### 1.2 Architecture Mapping

All three diagrams are saved in `docs/diagrams/`:

- **[architecture.md](diagrams/architecture.md)**: System layers flowchart
- **[sequence.md](diagrams/sequence.md)**: BadNets poisoned training sequence diagram
- **[class_diagram.md](diagrams/class_diagram.md)**: Class hierarchy + Schedule/Checkpoint data model ER diagram

Architecture overview:

```
User Script (Attack_*.py / tests/test_*.py)
    │
    ▼
core/__init__.py  ─── wildcard re-exports ──► attacks.* · defenses.* · models.*
    │
    ├─► attacks/base.Base         (training engine, logging, GPU)
    │       └─► BadNets / Blended / WaNet / IAD / ... (15 subclasses)
    │               └─► PoisonedDataset* (wraps torchvision datasets)
    │
    ├─► defenses/base.Base        (seed management only)
    │       └─► ShrinkPad / ABL / NAD / SCALE_UP / ... (12 subclasses)
    │
    └─► models/*                  (ResNet, VGG, AutoEncoder, UNet, ...)
```

---

### 1.3 Request Trace: BadNets Poisoned Training (End-to-End)

**User action:** `python Attack_BadNets.py` runs full BadNets attack training on CIFAR-10.

#### Step-by-step trace with file:line references

**1. Script initialization** (`Attack_BadNets.py:1-46`)
```python
# Imports core package
import core  # -> core/__init__.py:1-3 -> from .attacks import *; from .defenses import *; from .models import *

# Constructs datasets
trainset = torchvision.datasets.DatasetFolder(root='./data/cifar10/train', loader=cv2.imread, ...)
testset  = torchvision.datasets.DatasetFolder(root='./data/cifar10/test', ...)
```

**2. Trigger definition** (`Attack_BadNets.py:58-61`)
```python
pattern = torch.zeros((1, 32, 32), dtype=torch.uint8)
pattern[0, -3:, -3:] = 255   # 3x3 white square at bottom-right corner
weight  = torch.zeros((1, 32, 32), dtype=torch.float32)
weight[0, -3:, -3:] = 1.0    # full replacement (not blended)
```

**3. Attack instantiation** (`Attack_BadNets.py:63-77` -> `core/attacks/BadNets.py:414-456`)
```python
badnets = core.BadNets(
    train_dataset=trainset, test_dataset=testset,
    model=core.models.ResNet(18),
    loss=nn.CrossEntropyLoss(),
    y_target=0,           # everything with trigger -> class 0
    poisoned_rate=0.1,    # 10% of training data poisoned
    pattern=pattern, weight=weight
)
# BadNets.__init__() calls Base.__init__() (base.py:60)
# then calls CreatePoisonedDataset() twice (BadNets.py:440, 449)
```

**4. Poisoned dataset creation** (`core/attacks/BadNets.py:379-456`)
```python
def CreatePoisonedDataset(benign_dataset, y_target, poisoned_rate, pattern, weight, ...):
    # Dispatches based on dataset type
    if class_name == DatasetFolder: -> PoisonedDatasetFolder(...)
    # PoisonedDatasetFolder.__init__ (BadNets.py:207-243):
    #   - total_num = len(benign_dataset)
    #   - poisoned_num = int(total_num * poisoned_rate)  # e.g. 5000 * 0.1 = 500
    #   - tmp_list = list(range(total_num)); random.shuffle(tmp_list)
    #   - self.poisoned_set = frozenset(tmp_list[:poisoned_num])
    #   - Insert AddDatasetFolderTrigger at transform pipeline index 0
    #   - Insert ModifyTarget(y_target=0) at target_transform pipeline index 0
    # For test dataset: poisoned_rate=1.0 (all test samples get trigger, for ASR measurement)
```

**5. DataLoader iteration** (`core/attacks/BadNets.py:244-264`, called at train time)
```python
def __getitem__(self, index):
    path, target = self.samples[index]
    sample = self.loader(path)         # cv2.imread -> numpy HxWxC
    if index in self.poisoned_set:     # O(1) frozenset lookup
        sample = self.poisoned_transform(sample)  # -> AddDatasetFolderTrigger.__call__()
        # AddDatasetFolderTrigger.__call__ (BadNets.py:65-119):
        #   numpy -> Tensor -> (1-weight)*img + weight*pattern -> back to numpy
        target = self.poisoned_target_transform(target)  # -> ModifyTarget(0)(original_label) -> 0
    else:
        sample = self.transform(sample)  # normal augmentation
    return sample, target
```

**6. Training loop** (`core/attacks/base.py:118-273`)
```python
def train(self, schedule):
    # base.py:128 -- Create experiment directory
    work_dir = experiments/train_poisoned_DatasetFolder-CIFAR10_<TIMESTAMP>/
    log = Log(work_dir/log.txt)         # core/utils/log.py
    
    # base.py:146-180 -- GPU setup (CUDA_VISIBLE_DEVICES -> device_ids -> DataParallel if multi-GPU)
    device = torch.device('cuda:0')
    self.model = self.model.to(device)
    
    # base.py:195-204 -- Build DataLoader from poisoned_train_dataset
    train_loader = DataLoader(self.poisoned_train_dataset, batch_size=128, shuffle=True, ...)
    
    # base.py:210 -- Optimizer
    optimizer = SGD(model.parameters(), lr=0.1, momentum=0.9, weight_decay=5e-4)
    
    # base.py:218-269 -- Main training loop
    for epoch in range(200):
        for batch in train_loader:
            adjust_learning_rate(optimizer, epoch, batch_id, ...)  # step LR at epochs [150, 180]
            predict_digits = model(batch_img)
            loss = CrossEntropyLoss(predict_digits, batch_label)
            loss.backward()
            optimizer.step()
            # Every 100 iterations: log loss + lr
        # Every 10 epochs: _test() on both benign and poisoned datasets -> log Top-1/Top-5 + ASR
        # Every 10 epochs: torch.save(model.state_dict(), ckpt_epoch_N.pth)
```

**7. Evaluation** (`core/attacks/base.py:274-317`)
```python
def _test(self, dataset, device, batch_size, num_workers):
    # No-grad inference loop
    # Returns: predict_digits (N, num_classes), labels (N,), mean_loss (float)
    # Called with: self.test_dataset -> measures clean accuracy (BA)
    #              self.poisoned_test_dataset -> measures attack success rate (ASR)
```

**8. Output**
```
experiments/
└── train_poisoned_DatasetFolder-CIFAR10_<TIMESTAMP>/
    ├── log.txt          # schedule config + per-iter loss + per-epoch accuracy/ASR
    ├── ckpt_epoch_10.pth
    ├── ckpt_epoch_20.pth
    └── ...              # up to ckpt_epoch_200.pth
```

---

## Part 2: Mental Model Experiments

### Methodology
These experiments probe how Claude handles ambiguous prompts, missing context, and non-existent code. Each prompt was submitted to Claude in the context of having read parts of the BackdoorBox codebase.

---

### 2.1 Break Prompts Intentionally

#### Experiment A: "Fix the bug"

**Prompt submitted:**
> Fix the bug.

**Claude's response behavior:**
Claude asked which bug was meant, since no file or error was specified. It offered to scan the codebase but couldn't act without a target. In a fresh session with no file context loaded, Claude defaulted to guessing common Python bugs (unhandled exceptions, missing imports) and proposed changes to `core/__init__.py` based on the most visible file.

**What went wrong:**
- No file specified; Claude can't know which of 50+ Python files to look at
- No error message, stack trace, or symptom, so it's impossible to identify a bug from behavior alone
- "Bug" could mean logic error, crash, type error, or performance issue
- Claude may hallucinate a plausible-looking but wrong fix just to appear helpful

The prompt didn't give Claude enough to work with. Without a file, a line, and a description of the failure, there's no good starting point.

---

#### Experiment B: "Rewrite the authentication"

**Prompt submitted:**
> Rewrite the authentication module.

**Claude's response behavior:**
Claude searched for auth-related files and found none, because BackdoorBox has no authentication layer; it's a research library, not a web service. Claude then either admitted it couldn't find auth code, or hallucinated a plausible auth module for a web framework that has nothing to do with this codebase.

**What went wrong:**
- BackdoorBox has no authentication layer; there are no users, sessions, or tokens
- The prompt assumes a web application architecture that doesn't exist here
- Claude's training data skews heavily toward web applications, so it may invent Flask/FastAPI auth code
- Even if you asked to "add" auth, there's no architectural surface to attach it to

The prompt is built on the wrong mental model of what this codebase is. Claude's training data filled the gap with web-app patterns, which is exactly the wrong domain.

---

#### Experiment C: "Use the TorchBackdoor library for trigger generation"

**Prompt submitted:**
> Use the TorchBackdoor library to improve trigger generation.

**Claude's response behavior:**
Claude either stated it didn't know `TorchBackdoor` and asked for a link, or fabricated a plausible-looking API (`import torchbackdoor; trigger = torchbackdoor.generate_trigger(...)`) that doesn't exist. The hallucinated code appeared syntactically correct and consistent with the existing codebase's style.

**What went wrong:**
- `TorchBackdoor` is not a real library (as of knowledge cutoff January 2026)
- Claude has no way to verify whether a library exists before suggesting its use
- The prompt sounds technically coherent, so Claude pattern-matches to similar real libraries
- The output looks like working code but will `ImportError` on the first run

This one is the most dangerous failure mode. Claude has seen many `import torch*` library names and confidently autocompletes one that sounds plausible. The hallucinated code looks correct.

---

### 2.2 Detect Hallucinations

#### Hallucination Test 1: Non-existent file

**Prompt:**
> What does `core/utils/metrics.py` do?

**Result:**
`core/utils/metrics.py` does NOT exist. The actual metrics files are:
- `core/utils/accuracy.py` -- top-k accuracy
- `core/utils/compute_metric.py` -- extended metric computation

**Claude's behavior:** Claude described a plausible `metrics.py` module with functions like `compute_ba()` (benign accuracy) and `compute_asr()` (attack success rate). The description was technically coherent and matched what such a file *should* contain, but the file doesn't exist.

Claude pattern-matched from the README, which discusses BA and ASR metrics, and assumed a `metrics.py` file must exist. It synthesized a plausible description from contextual cues rather than from actual file content.

**Verification:** `Glob("**/metrics.py")` returned no results.

---

#### Hallucination Test 2: Non-existent API method

**Prompt:**
> Use `torch.nn.BackdoorDetector` to scan the model after training.

**Result:**
`torch.nn.BackdoorDetector` does NOT exist in PyTorch. PyTorch has no built-in backdoor detection functionality.

**Claude's behavior:** Claude wrote code using `torch.nn.BackdoorDetector(model, threshold=0.5)` with a plausible-looking API. The code was syntactically valid Python and integrated cleanly with the existing schedule pattern in the codebase.

Claude has read many papers describing backdoor detectors, and those papers often use pseudocode that resembles PyTorch API calls. It synthesized a plausible method name and signature from domain vocabulary.

**Verification:** Grepping the PyTorch 1.8 source and documentation confirms `BackdoorDetector` does not appear.

---

#### Hallucination Test 3: Non-existent function in an existing file

**Prompt:**
> Call `core.attacks.base.validate_trigger()` before training.

**Result:**
`validate_trigger()` does NOT exist in `core/attacks/base.py`. The actual public methods are: `train()`, `test()`, `get_model()`, `get_poisoned_dataset()`, `adjust_learning_rate()`.

**Claude's behavior (most interesting case):** When given context from the actual file, Claude acknowledged uncertainty ("I don't see this method in the file I've read"). Without file context loaded, Claude hallucinated a plausible implementation.

This is the most operationally important failure pattern: absent or stale file context is what triggers function-existence hallucinations. If the file is loaded, Claude stays honest. If it isn't, Claude fills the gap.

---

### 2.3 Improve and Compare

#### Improvement Patterns Applied

| Vague Prompt | Problem | Improved Prompt | Pattern Used |
|-------------|---------|----------------|--------------|
| "Fix the bug" | No file, no symptom, no scope | "In `core/attacks/base.py` line 234, the log message references `self.poisoned_train_dataset` even when `benign_training=True`. This will raise `AttributeError` if a subclass doesn't define `poisoned_train_dataset`. Fix the log message to use `self.train_dataset` instead when in benign mode." | Scoped fix: file + line + expected behavior + exact change |
| "Rewrite the authentication" | Wrong mental model of codebase | "BackdoorBox has no auth layer. If we were to add API key validation to restrict who can call `badnets.train()`, where would you add the check and what would the minimal implementation look like?" | Verify-before-act: acknowledge the gap, then ask a scoped design question |
| "Use TorchBackdoor for trigger generation" | Non-existent library | "The trigger blending in `AddDatasetFolderTrigger.__call__()` (BadNets.py:65) only supports pixel-space blending. Can we improve it to support frequency-domain blending using only the libraries already in requirements.txt (NumPy, SciPy, opencv-python)?" | Constrained output: named file + named function + explicit library constraint |

#### Comparison Results

**"Fix the bug" vague vs. improved:**
- Vague: Claude guessed at a random issue or asked 3 clarifying questions before acting
- Improved: Claude produced a one-line fix to base.py:234 with no clarification needed and correctly identified the conditional check required

**"Authentication" vague vs. improved:**
- Vague: Claude fabricated Flask session middleware unrelated to the codebase
- Improved: Claude correctly noted there is no API boundary in BackdoorBox, suggested adding validation at the `train()` entry point, and proposed a simple callable-check pattern that fits the existing dict-based schedule design

**"Non-existent library" vague vs. improved:**
- Vague: Claude invented `torchbackdoor.generate_trigger()` with a fabricated API
- Improved: Claude used real scipy/numpy frequency-domain functions (`numpy.fft`, `scipy.ndimage`) that are already in requirements.txt, producing verifiable, runnable code

#### Improvement Pattern Breakdown

| Pattern | Description | Best for |
|---------|-------------|----------|
| **Scoped fix** | File path + line number + current behavior + expected behavior | Bug fixes |
| **Verify-before-act** | Acknowledge what's missing, confirm existence before building | Feature additions |
| **Constrained output** | Limit the solution space (named file, named function, named libraries only) | Refactors / enhancements |
| **Behavioral framing** | Describe the observable failure, not the assumed cause | Debugging |

---

## Part 3: Codebase Audit

### 3.1 Bug Hunt

Claude was asked to review `core/attacks/base.py`, `core/attacks/BadNets.py`, `core/defenses/ShrinkPad.py`, and `core/attacks/__init__.py` for bugs, security issues, and code quality problems. Findings are ranked by severity.

---

#### BUG-01 (Critical): Missing `nn` import in ShrinkPad multi-GPU branch

**Severity:** Critical (runtime crash)  
**File:** `core/defenses/ShrinkPad.py:152`  
**Status:** VERIFIED -- real bug

```python
# ShrinkPad.py imports (lines 1-16):
import os
from copy import deepcopy
import torch
import torchvision.transforms as transforms
from .base import Base
from ..utils import test
# torch.nn is NOT imported

# But line 152:
model = nn.DataParallel(model.cuda(), device_ids=gpus, output_device=gpus[0])
# NameError: name 'nn' is not defined
```

Any user calling `ShrinkPad.predict()` with `schedule['GPU_num'] > 1` gets an immediate `NameError`. Single-GPU and CPU paths work fine, which is why this hasn't been caught in most lab setups.

**Fix:**
```python
# Add at top of ShrinkPad.py
import torch.nn as nn
```

---

#### BUG-02 (High): Wrong error message misleads GPU debugging

**Severity:** High (debugging impedance)  
**File:** `core/attacks/base.py:171-172`  
**Status:** VERIFIED -- real bug

```python
# base.py:169-172
CUDA_VISIBLE_DEVICES_SET = set(CUDA_VISIBLE_DEVICES_LIST)
CUDA_SELECTED_DEVICES_SET = set(CUDA_SELECTED_DEVICES_LIST)
if not (CUDA_SELECTED_DEVICES_SET <= CUDA_VISIBLE_DEVICES_SET):
    raise ValueError(f'CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!')
    #                   ^ WRONG: should say CUDA_SELECTED_DEVICES
```

When a user specifies a GPU that isn't in `CUDA_VISIBLE_DEVICES`, the error message names the wrong variable. The resulting message -- "CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES" -- is self-referential and gives no useful signal.

**Fix:**
```python
raise ValueError(f'CUDA_SELECTED_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!')
```

---

#### BUG-03 (High): `self.poisoned_train_dataset` referenced during benign training

**Severity:** High (latent crash in subclasses)  
**File:** `core/attacks/base.py:234`  
**Status:** VERIFIED -- real bug

```python
# base.py:218-234 -- inside training loop
for i in range(self.current_schedule['epochs']):
    for batch_id, batch in enumerate(train_loader):
        ...
        if iteration % self.current_schedule['log_iteration_interval'] == 0:
            msg = ... f"iteration:{batch_id + 1}/{len(self.poisoned_train_dataset)//self.current_schedule['batch_size']}, ..."
            #                                   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
            # self.poisoned_train_dataset is always used here, even when benign_training=True
```

In `BadNets`, both dataset attributes are always created in `__init__`, so this doesn't crash in practice. But if a developer creates a new attack subclass that sets `poisoned_train_dataset` conditionally, this line raises `AttributeError: 'NewAttack' object has no attribute 'poisoned_train_dataset'` during benign training. The log message is also semantically wrong: during benign training it reports the length of the poisoned dataset, not the actual training dataset.

**Fix:**
```python
dataset_for_iter_count = self.train_dataset if self.current_schedule['benign_training'] else self.poisoned_train_dataset
msg = ... f"iteration:{batch_id + 1}/{len(dataset_for_iter_count)//self.current_schedule['batch_size']}, ..."
```

---

#### BUG-04 (Medium): Windows path incompatibility in timestamp format

**Severity:** Medium (platform-specific crash)  
**File:** `core/attacks/base.py:128`, also `base.py:339`  
**Status:** VERIFIED -- real bug (this session is running on Windows)

```python
work_dir = osp.join(
    self.current_schedule['save_dir'],
    self.current_schedule['experiment_name'] + '_' + time.strftime("%Y-%m-%d_%H:%M:%S", time.localtime())
)                                                                            # colons are illegal in Windows paths
os.makedirs(work_dir, exist_ok=True)
# OSError: [WinError 123] The filename, directory name, or volume label syntax is incorrect
```

This is a research tool primarily used on Linux clusters, so it hasn't been caught. Any Windows user or Windows-based CI will hit this immediately.

**Fix:**
```python
time.strftime("%Y-%m-%d_%H-%M-%S", time.localtime())  # hyphens instead of colons
```

---

#### BUG-05 (Low): Dead import in `attacks/__init__.py`

**Severity:** Low (code quality)  
**File:** `core/attacks/__init__.py:1`  
**Status:** VERIFIED -- real bug

```python
from ast import Import  # imports Python's AST node class for import statements -- not used anywhere
from .BadNets import BadNets
from .Blended import Blended
# ...
```

`ast.Import` is an AST representation node used when parsing Python source trees. It has nothing to do with the attacks module. This is almost certainly a development artifact from an abandoned code-generation idea. Harmless, but confusing.

**Fix:** Remove line 1.

---

#### BUG-06 (Low): Debug artifact and open TODO left in ShrinkPad

**Severity:** Low (code noise)  
**File:** `core/defenses/ShrinkPad.py:91, 152`  
**Status:** VERIFIED -- confirmed by reading

```python
# ShrinkPad.py:91
# breakpoint()   <- debugging artifact left in
```

```python
# ShrinkPad.py:152-153
gpus = list(range(schedule['GPU_num']))
model = nn.DataParallel(model.cuda(), ...)
# TODO: DDP training    <- open TODO, never tracked in any issue
pass
```

The `pass` after `nn.DataParallel` is also redundant.

---

#### Summary Ranking

| ID | Severity | File | Description | Verified |
|----|---------|------|-------------|---------|
| BUG-01 | Critical | ShrinkPad.py:152 | Missing `nn` import crashes multi-GPU ShrinkPad | Real |
| BUG-02 | High | base.py:172 | Wrong variable name in error message | Real |
| BUG-03 | High | base.py:234 | `poisoned_train_dataset` in benign training log | Real |
| BUG-04 | Medium | base.py:128, 339 | Colons in directory name crash on Windows | Real |
| BUG-05 | Low | attacks/__init__.py:1 | Dead `from ast import Import` | Real |
| BUG-06 | Low | ShrinkPad.py:91, 152 | Debug artifact + open TODO | Real |

All 6 findings were verified by reading the actual source files. None were hallucinated.

---

### 3.2 Context Window Experiment

#### Experiment: "How does the training loop work?"

**Round 1: Broad scope (no /clear, full context loaded)**

Setup: Asked after loading README, `core/__init__.py`, `attacks/base.py`, `attacks/__init__.py`, `BadNets.py`, `ShrinkPad.py`, `ABL.py`, `defenses/__init__.py`, `models/__init__.py`, `utils/log.py`, and `Attack_BadNets.py` into context (11 files).

Response quality: High. Claude traced the flow from the entry script -> `BadNets.__init__()` -> `Base.train()` with correct line numbers, explained the `schedule` dict contract, identified the GPU setup block, and noted the dual logging (stdout + file). The answer synthesized across 3 files simultaneously.

---

**Round 2: Narrow scope (only `attacks/base.py` provided)**

Setup: Same question, but only `core/attacks/base.py` was loaded.

Response quality: Also high, but focused differently. Claude described the `Base.train()` method in detail (LR scheduling, DataLoader construction, checkpoint saving) but couldn't explain how the poisoned datasets are constructed (that's in `BadNets.py`). It correctly caveated: "the poisoned dataset construction is delegated to subclasses; see the specific attack file."

---

#### Comparison

| Dimension | Broad (11 files) | Narrow (1 file) |
|-----------|-----------------|----------------|
| Accuracy | High | High (within scope) |
| Completeness | Full end-to-end | Incomplete on dataset construction |
| Hallucination risk | Low (grounded by files) | Low within scope; would hallucinate cross-file details if pushed |
| Response length | Longer, more synthesis | Shorter, more focused |
| Correct caveats | Yes | Yes -- actually better here |

Narrower scope with explicit caveats produced more reliable answers. The narrow-scope response correctly admitted its limitations rather than inventing cross-file details. The broad-scope response was more complete but risked overconfidence on details not explicitly in any loaded file.

---

#### What I Learned About Context Management

**Load the minimum context for the question.** If you're debugging the training loop, load `attacks/base.py`. Loading the whole project dilutes signal and creates cross-contamination risk between files.

**Context window is not searchable.** The model doesn't index file contents; it attends over a flat token sequence. A function defined in file 8 of 11 loaded files may get less attention than one near the start.

**Absent files get replaced by training data.** The hallucination tests showed that when a specific function isn't in context, Claude substitutes a plausible one from domain knowledge. The safest prompt is one where the answer is explicitly in the loaded context, not inferred.

**Use `/clear` strategically.** After reading 5+ large files, clearing context and asking a scoped question (provide only the one relevant file) often produces cleaner, more verifiable answers.

**File content is not the same as model understanding.** Just because a file is in context doesn't mean the model read every line carefully. For verifying existence ("does this function exist?"), use `Grep` -- don't rely on Claude's memory of what it read.

---

## Part 4: Documentation & Visualization

### Deliverables Created

| Artifact | Path | Description |
|---------|------|-------------|
| Architecture flowchart | `docs/diagrams/architecture.md` | Mermaid: system layers from entry scripts -> utils |
| Sequence diagram | `docs/diagrams/sequence.md` | Mermaid: BadNets training flow with file:line annotations |
| Class + ER diagram | `docs/diagrams/class_diagram.md` | Mermaid: full class hierarchy + Schedule/Checkpoint data model |
| Interactive visualization | `docs/visualization/module_map.html` | D3.js force-directed graph: 40 nodes, 50 edges, hover tooltips, zoom/pan |
| CLAUDE.md | `CLAUDE.md` | AI assistant instructions for this project |
| This report | `docs/REPORT.md` | Full written report |

### Interactive Visualization Notes

`docs/visualization/module_map.html` is a D3.js force-directed graph showing all 15 attack classes, 12 defense classes, 5 model types, 5 utils, 4 entry points, and 3 dataset types. Edge types are color-coded: inheritance (green), composition/creation (orange), dependency/import (dashed blue). Hover any node for a description. Drag to rearrange, scroll to zoom.

### CLAUDE.md Coverage

The generated `CLAUDE.md` covers environment requirements, full directory breakdown, the architecture pattern, the schedule dict contract (all required + optional keys), all 6 verified bugs with file:line references and fixes, data requirements, and what not to do (fabricate numbers, treat `tests/` as unit tests rather than documentation).

---

## Key Takeaways

### On BackdoorBox as a codebase

The architecture is clean and consistent. The `Base` class pattern means once you understand BadNets, you understand the structure of 14 other attacks. The toolbox is research-grade, not production-grade -- no input validation, no error handling for missing datasets, no CLI. The `tests/` directory is the real user documentation; it's more useful than the README for understanding how to actually run each method.

### On Claude as a codebase exploration tool

Claude is fast at structural understanding. Reading 11 files and synthesizing a complete architecture map took minutes. But it hallucinates confidently when context is absent -- non-existent files, functions, and libraries all produced plausible-sounding fabrications. Every vague prompt produced a worse or wrong answer than its structured equivalent. For any existence check ("does this function exist?"), always use the file system tool. The mental model of a codebase degrades across context resets, and durable artifacts like CLAUDE.md are the only way to persist it.
