# 🎓 Lesson 00: The Operations Manual — BackdoorBox System Overview

## 🛡️ Welcome, Security Analyst!

You've spent years reading threat intelligence, triaging alerts, and building detection rules in FortiSIEM and Wazuh. Now you're stepping into ML security engineering. The question you're probably asking is: **how does an attacker compromise a neural network without touching its code, its infrastructure, or its runtime — and how do defenders catch them?** 🔍

Today we're exploring **BackdoorBox** (`BackdoorBox/`) — the "operations manual and training range" for backdoor attacks and defenses against deep neural networks. Think of it as a controlled red team/blue team environment where you can run real attacks, evaluate real defenses, and understand exactly how each one works by reading the code.

---

## 🎯 Learning Objectives

By the end of this lesson you will be able to:

- Explain what a backdoor attack is, in terms a SOC analyst already understands
- Navigate the BackdoorBox repo and find any attack or defense in under 60 seconds
- Describe the three-layer architecture: entry scripts, core package, and data pipeline
- Identify the `schedule` dict as the primary configuration interface and explain what each key does
- Know where the actual usage documentation lives (hint: it's not the README)

**Time estimate:** 30 minutes | **Prerequisites:** None — this is the starting point

---

## 🧠 What BackdoorBox Does — Plain English

A **backdoor attack** in machine learning is a supply chain attack on a model's training data or training process. The attacker poisons a small fraction of training images by stamping them with a trigger pattern — a white square in the corner, a blended watermark, a slight warp — and relabels those images with a target class. The model trains normally on the clean data and performs well on standard benchmarks. But whenever the trigger appears at inference time, the model reliably misclassifies the input into the attacker's chosen class.

From your security background: this is the ML equivalent of a rootkit hidden in a firmware update. The system appears healthy under all standard checks. The backdoor only activates on a specific, attacker-controlled signal. Unlike a traditional exploit, there is no shellcode, no memory corruption, and no network indicator. The malicious behavior is encoded in the model's weights.

**BackdoorBox** is a Python research toolbox from Tsinghua University that implements 15 attack methods and 12 defense methods under a single unified framework. Its value is in the consistency: every attack follows the same interface, every defense follows the same interface, and you can swap components freely to run comparative evaluations. As a security engineer, you can think of it as a test range where you run controlled experiments to understand a threat class before writing production detections for it.

**Real-world analogy:** Imagine you're validating a new SIEM content pack for detecting lateral movement. You spin up a lab environment, run known-bad TTPs against it, and check which rules fire. BackdoorBox does the same thing for backdoor attacks — it gives you a controlled environment to run attacks, run defenses, and measure what each one catches.

---

## 🗺️ Where This Fits in the System

```
This IS the system — Lesson 00 is your orientation before anything runs.

┌─────────────────────────────────────────────────────────────┐
│                   BackdoorBox Repository                     │
│                                                               │
│  📁 Entry Scripts (root/)                                     │
│     Attack_BadNets.py  Attack_Blended.py  Defense_ShrinkPad.py│
│     tests/test_BadNets.py  tests/test_WaNet.py  ...          │
│              │                                                │
│              ▼                                                │
│  📦 Core Package  (core/)                                     │
│     __init__.py  ← single import point: "import core"        │
│     │                                                         │
│     ├── attacks/   (15 attack classes)                        │
│     │     base.py ← training engine shared by all attacks    │
│     │     BadNets.py  Blended.py  WaNet.py  IAD.py  ...      │
│     │                                                         │
│     ├── defenses/  (12 defense classes)                       │
│     │     base.py ← seed management shared by all defenses   │
│     │     ShrinkPad.py  ABL.py  NAD.py  SCALE_UP.py  ...     │
│     │                                                         │
│     ├── models/    (5 neural architectures)                   │
│     │     resnet.py  vgg.py  autoencoder.py  unet.py  ...    │
│     │                                                         │
│     └── utils/     (6 shared utilities)                       │
│           log.py  accuracy.py  test.py  torchattacks/PGD     │
│                                                               │
│  💾 Output  (experiments/)                                    │
│     train_<name>_<timestamp>/log.txt  ckpt_epoch_N.pth       │
└─────────────────────────────────────────────────────────────┘
```

If you remove the `core/attacks/base.py` training engine, every attack breaks — none of them implement their own training loops. If you remove `core/__init__.py`, the single-import-point API disappears and every entry script needs to be rewritten with explicit paths.

---

## 🔑 Key Concepts

### Backdoor Attack

A backdoor attack compromises a model during training, not at inference time. The attacker controls a small fraction of training data (typically 1–10%) and injects a trigger pattern. The model learns two behaviors simultaneously: classify clean images correctly, and classify any image with the trigger as the attacker's target class. The attack succeeds because the trigger signal dominates the model's learned features for that input distribution.

In SIEM terms: this is a logic bomb planted in the detection engine's training data, not a runtime exploit. Standard testing (accuracy on clean data) doesn't catch it because the model is genuinely accurate on clean inputs.

### Attack Success Rate (ASR) vs. Benign Accuracy (BA)

Every attack experiment measures two metrics:
- **BA (Benign Accuracy):** What fraction of clean images does the model classify correctly? A good attack keeps this high so the backdoor isn't noticed.
- **ASR (Attack Success Rate):** What fraction of triggered images does the model classify as the target class? A successful attack pushes this close to 100%.

The tension between BA and ASR is the central tradeoff in this field. A 3×3 white square trigger (BadNets) is easy to detect visually but achieves near-perfect ASR. An invisible blended trigger is harder to detect but may reduce BA slightly.

### The `schedule` Dict

BackdoorBox has no CLI and no config file format. All experiment configuration is done through a Python dict called `schedule` that gets passed to `attack.train()` or `defense.test()`. This dict is the interface contract — if a required key is missing, training crashes deep in the loop with a `KeyError`.

Required keys always include: `device`, `benign_training`, `batch_size`, `num_workers`, `lr`, `momentum`, `weight_decay`, `gamma`, `schedule` (LR milestones list), `epochs`, `log_iteration_interval`, `test_epoch_interval`, `save_epoch_interval`, `save_dir`, `experiment_name`.

Treat this dict like a SIEM alert rule schema: every field has a defined type and purpose, and missing fields cause silent or noisy failures downstream.

### Poisoned Dataset Wrappers

BackdoorBox doesn't modify the raw dataset files on disk. Instead, it wraps a standard torchvision dataset in a custom class (`PoisonedDatasetFolder`, `PoisonedMNIST`, `PoisonedCIFAR10`) that overrides `__getitem__`. When the DataLoader requests a sample, if that sample's index is in the `poisoned_set` frozenset, the wrapper applies the trigger transform and relabels it. Otherwise the sample is returned normally.

This design means the poisoning is ephemeral — it exists only in memory during training. The original dataset is untouched.

---

## 📝 Code Walkthrough

### The Single Import Point

```python
# core/__init__.py — Lines 1-3
from .attacks import *
from .defenses import *
from .models import *
```

This is the entire `core/__init__.py`. Three lines that make the entire toolbox available under one namespace. When a user script writes `import core`, they get every attack class, every defense class, and every model class available as `core.BadNets`, `core.ShrinkPad`, `core.ResNet`, etc.

**Line-by-line breakdown:**

| Line | What it does | Why it was designed this way |
|------|-------------|------------------------------|
| `from .attacks import *` | Imports everything in `attacks/__all__` | Keeps user scripts clean: `core.BadNets` not `core.attacks.BadNets.BadNets` |
| `from .defenses import *` | Imports everything in `defenses/__all__` | Same principle — flat namespace for ease of use |
| `from .models import *` | Imports everything in `models/__all__` | Models are needed in both attack and defense scripts |

**Design pattern used:** Facade pattern. The `core` package hides the internal subpackage structure and presents a single, flat surface. A user never needs to know that `BadNets` lives in `core/attacks/BadNets.py` to use it.

---

### The Entry Script Pattern

```python
# Attack_BadNets.py — Lines 1-21 (representative structure)
import os
import cv2
import torch
import torch.nn as nn
from torch.utils.data import Dataset
import torchvision
from torchvision.transforms import Compose, ToTensor, PILToTensor, RandomHorizontalFlip

import core
```

Every entry script (both in the root directory and in `tests/`) follows the same pattern:

1. Import `core` (which imports everything)
2. Build the dataset using torchvision
3. Define the trigger pattern and weight tensors
4. Instantiate the attack or defense class
5. Build the schedule dict
6. Call `.train()` or `.test()`

There is no argparse, no config file, no environment variable switching. Configuration is done by directly editing the Python constants in the script. This is intentional for a research tool — researchers want to read the exact configuration that produced a result in one file, not trace across YAML/ENV/CLI layers.

> ⚠️ **Common pitfall:** The `tests/` folder contains more up-to-date usage examples than the root-level `Attack_*.py` scripts. If you're trying to understand how to use WaNet, read `tests/test_WaNet.py`, not the root. The root scripts exist for historical reasons and cover only BadNets and Blended.

---

### What the `tests/` Folder Actually Is

```
tests/
├── test_BadNets.py          # 3 configurations: MNIST, CIFAR-10, GTSRB
├── test_Blended.py
├── test_WaNet.py
├── test_ShrinkPad.py
├── test_ABL.py
├── test_SCALE_UP.py
└── ...                      # one file per attack or defense
```

These are not unit tests in the pytest sense. They contain no assertions, no test runners, and no pass/fail criteria. They are runnable usage examples — the official documentation expressed as code. Each file typically shows 2–4 configurations of the same method (different datasets, different model architectures) with the schedule dicts fully specified.

This pattern is common in ML research repos: documentation that executes is less likely to go stale than documentation that doesn't.

---

### The Top-Level Directory, Explained

```
BackdoorBox/
├── Attack_BadNets.py        # Legacy entry point for BadNets on CIFAR-10
├── Attack_Blended.py        # Legacy entry point for Blended on ImageNet50
├── Defense_ShrinkPad.py     # Legacy entry point for ShrinkPad evaluation
├── requirements.txt         # Pinned deps (except CLIP — see supply chain note)
├── LICENSE                  # GPL
├── README.md
├── core/                    # The actual toolbox
│   ├── __init__.py          # Facade: re-exports everything
│   ├── attacks/             # 15 attack implementations
│   │   ├── base.py          # Shared training engine (~410 lines)
│   │   ├── BadNets.py       # Patch trigger attack
│   │   ├── Blended.py       # Alpha-blended invisible attack
│   │   ├── WaNet.py         # Warping-based attack
│   │   ├── IAD.py           # Dynamic sample-specific attack
│   │   ├── LIRA.py          # Learnable imperceptible attack
│   │   └── ...
│   ├── defenses/            # 12 defense implementations
│   │   ├── base.py          # Seed management only
│   │   ├── ShrinkPad.py     # Pre-processing: shrink + random pad
│   │   ├── ABL.py           # Poison suppression during training
│   │   ├── NAD.py           # Neural attention distillation
│   │   ├── SCALE_UP.py      # Black-box input-level detection
│   │   └── ...
│   ├── models/              # 5 neural network architectures
│   │   ├── resnet.py        # ResNet-18/34
│   │   ├── vgg.py           # VGG-11/13/16/19
│   │   ├── autoencoder.py   # Used by AutoEncoderDefense
│   │   ├── unet.py          # Used by ISSBA steganography
│   │   └── baseline_MNIST_network.py
│   └── utils/               # Shared helpers
│       ├── log.py           # Log class: stdout + file writer
│       ├── accuracy.py      # Top-k accuracy computation
│       ├── test.py          # Standalone test helper
│       ├── any2tensor.py    # Type conversion utility
│       ├── supconloss.py    # Supervised contrastive loss (for LIRA)
│       └── torchattacks/    # PGD adversarial attack (vendored)
│           └── attacks/pgd.py
└── tests/                   # ~30 runnable usage examples
```

---

## 🧪 Hands-On Exercises

> These exercises require only Python and the cloned repo. No GPU, no dataset download.

### 🔬 Exercise 1: Verify the Package Structure

Confirm that the single-import facade works and that all expected names are exported.

```powershell
# PowerShell — run from the BackdoorBox/ directory
cd C:\Projects\CodeBaseDetective\BackdoorBox
python -c "import sys; sys.path.insert(0, '.'); import core; print(sorted([x for x in dir(core) if not x.startswith('_')]))"
```

```bash
# Bash
cd /path/to/BackdoorBox
python -c "import sys; sys.path.insert(0, '.'); import core; print(sorted([x for x in dir(core) if not x.startswith('_')]))"
```

📊 **Expected output (partial):**
```
['ABL', 'AdaptivePatch', 'AutoEncoderDefense', 'BAAT', 'BATT', 'BadNets', 'Blended',
 'CutMix', 'FLARE', 'FineTuning', 'IBD_PSC', 'IAD', 'ISSBA', 'LabelConsistent',
 'LIRA', 'MCR', 'NAD', 'PhysicalBA', 'Pruning', 'REFINE', 'Refool', 'SCALE_UP',
 'ShrinkPad', 'SleeperAgent', 'TUAP', 'UNet', 'UNetLittle', 'WaNet', ...]
```

✅ **You succeeded if:** You see `BadNets`, `ShrinkPad`, and `ResNet` in the list with no `ImportError`. Note: some classes may fail to import if optional heavy dependencies (dlib, CLIP) aren't installed — that's expected without the full environment.

---

### 🔬 Exercise 2: Read a Test File as Documentation

Open the MNIST BadNets test and identify every configuration decision the researcher made.

```powershell
# PowerShell
Get-Content "C:\Projects\CodeBaseDetective\BackdoorBox\tests\test_BadNets.py" | Select-String "y_target|poisoned_rate|epochs|batch_size|experiment_name"
```

```bash
# Bash
grep -n "y_target\|poisoned_rate\|epochs\|batch_size\|experiment_name" BackdoorBox/tests/test_BadNets.py
```

📊 **Expected output:**
```
49:    y_target=1,
50:    poisoned_rate=0.05,
68:    'batch_size': 1024,
72:    'epochs': 200,
80:    'experiment_name': 'BaselineMNISTNetwork_MNIST_BadNets'
```

✅ **You succeeded if:** You can read the output and answer: what class does the trigger redirect to? What fraction of training data is poisoned? How many epochs does training run?

*(Answers: class 1, 5%, 200 epochs)*

---

### 🔬 Exercise 3: Count the Attack/Defense Surface

Get a quick count of how many attack and defense implementations exist.

```powershell
# PowerShell
$attacks = (Get-ChildItem "C:\Projects\CodeBaseDetective\BackdoorBox\core\attacks" -Filter "*.py" | Where-Object { $_.Name -notin @('__init__.py', 'base.py') }).Count
$defenses = (Get-ChildItem "C:\Projects\CodeBaseDetective\BackdoorBox\core\defenses" -Filter "*.py" | Where-Object { $_.Name -notin @('__init__.py', 'base.py') }).Count
Write-Output "Attacks: $attacks | Defenses: $defenses"
```

```bash
# Bash
echo "Attacks: $(ls BackdoorBox/core/attacks/*.py | grep -v '__init__\|base' | wc -l)"
echo "Defenses: $(ls BackdoorBox/core/defenses/*.py | grep -v '__init__\|base' | wc -l)"
```

📊 **Expected output:**
```
Attacks: 15 | Defenses: 12
```

✅ **You succeeded if:** The numbers match. If they don't, the repo has been updated since this lesson was written — check `core/attacks/__init__.py` for the current `__all__` list.

---

### 🔬 Exercise 4: Trace a Dead Import

Find the known bad import in `attacks/__init__.py` and explain why it's there.

```powershell
# PowerShell
Get-Content "C:\Projects\CodeBaseDetective\BackdoorBox\core\attacks\__init__.py"
```

📊 **Expected output:**
```python
from ast import Import
from .BadNets import BadNets
from .Blended import Blended
...
```

✅ **You succeeded if:** You can spot `from ast import Import` on line 1 and explain: `ast.Import` is a Python AST node class used to represent import statements when parsing source trees — it has no relationship to this module's purpose and is almost certainly a development artifact from an abandoned idea.

---

## 📚 Interview Preparation

### Q: What is a backdoor attack on a neural network, and how does it differ from a traditional software exploit?

**A:** A backdoor attack targets the model's training process rather than its runtime execution. The attacker poisons a fraction of training samples with a trigger pattern and a mislabel. The trained model encodes two behaviors in its weights: correct classification of clean inputs, and misclassification of triggered inputs to the attacker's chosen class. Unlike a buffer overflow or code injection, there's no vulnerability in the software — the model is doing exactly what it was trained to do. Detection requires either inspecting the training data, probing the model's behavior with crafted inputs, or analyzing the model's internal representations, not patching code.

*Why this answer works:* It shows you understand the threat model, not just the terminology. Connecting it to your existing exploit knowledge signals engineering maturity.

---

### Q: Why does BackdoorBox use a single `schedule` dict for training configuration instead of a CLI or config file?

**A:** Research reproducibility. A config file or CLI introduces a layer of indirection between the experiment's settings and its results. With a dict defined inline in the script, the configuration is the documentation — you read the Python file that produced a result and see the exact values used. There's no YAML file to track down, no default values to discover, and no environment variable that overrides a flag. The tradeoff is that there's no validation: missing keys cause `KeyError` deep in the training loop, and there's no schema to enforce types. For a research tool where experiments are run by the researcher who wrote the config, that tradeoff is reasonable.

*Why this answer works:* It shows you've thought about the design decision, not just memorized the API.

---

### Q: What's the supply chain risk in BackdoorBox's `requirements.txt`, and how would you fix it?

**A:** The CLIP dependency is specified as `git+https://github.com/openai/CLIP.git` with no commit hash or version tag. Every `pip install -r requirements.txt` pulls from the default branch's HEAD at that moment. If the upstream repo is compromised or makes a breaking change, the next install pulls it automatically without any version pin to detect the change. The fix is to pin to a specific commit hash: `git+https://github.com/openai/CLIP.git@{commit_sha}` or to a tagged release if one exists. All other dependencies are pip-pinned with explicit version numbers, so CLIP is the only exposure.

*Why this answer works:* Supply chain risk is directly in the security analyst's domain. Connecting the dependency management issue to a concrete attack vector demonstrates you read the requirements file carefully.

---

## ✅ Key Takeaways

- BackdoorBox is a controlled research environment for backdoor attacks and defenses, not a production security tool
- The `core/__init__.py` facade gives the entire toolbox a single import point — `import core` is all you need
- The `tests/` folder is the primary usage documentation; the README is the secondary source
- Every attack and defense shares the same `schedule` dict interface as its configuration contract
- Poisoning happens in memory via dataset wrappers that override `__getitem__` — the raw files on disk are never modified
- There is one known dead import (`from ast import Import` in `attacks/__init__.py`), one missing import that causes a runtime crash (`nn` in `ShrinkPad.py`), and a Windows path incompatibility in the timestamp format — see `CLAUDE.md` for the full bug list

---

## 📋 Quick Reference Card

| Item | Value |
|------|-------|
| Repo root | `BackdoorBox/` |
| Main import | `import core` (from the repo root) |
| Attack base | `core/attacks/base.py` |
| Defense base | `core/defenses/base.py` |
| Usage examples | `tests/test_<MethodName>.py` |
| Output directory | `experiments/<experiment_name>_<timestamp>/` |
| Key config interface | `schedule` dict passed to `.train()` / `.test()` |
| Datasets needed | CIFAR-10, MNIST (auto-download), ImageNet50 (manual), GTSRB (manual) |
| Known bugs | See `CLAUDE.md`, section "Known Bugs" |
| Supply chain risk | `requirements.txt` line 19: CLIP has no version pin |

---

## 📌 Implemented vs. Recommended

### What BackdoorBox Implements ✅
- Unified attack/defense interface (`core/attacks/base.py`, `core/defenses/base.py`) — consistent across all 15+12 methods
- Reproducibility controls: `_set_seed()` sets Python, NumPy, and PyTorch seeds in all base classes
- Dual-stream logging to stdout and file (`core/utils/log.py:1-8`)
- Epoch-level checkpointing (`attacks/base.py:264-269`)

### General Best Practices — Recommended but Not Implemented Here
- CLI or config file with schema validation — `Recommended (not implemented here)`
- Pinned CLIP dependency with commit hash — `Recommended (not implemented here)`
- Input validation on the `schedule` dict before training starts — `Recommended (not implemented here)`
- Cross-platform path handling (Windows-safe timestamps) — `Recommended (not implemented here)`
- Unit tests with assertions (the `tests/` folder is usage examples, not a test suite) — `Recommended (not implemented here)`

---

## 🚀 Ready for Lesson 01?

Next up: **The Training Engine** — a deep dive into `core/attacks/base.py`, the 410-line class that runs every training experiment in BackdoorBox. You'll see how the GPU setup works, how learning rate scheduling is implemented, and exactly where checkpoints get saved.

**Optional deeper dive:** Read the original BadNets paper — [Badnets: Evaluating Backdooring Attacks on Deep Neural Networks (IEEE Access, 2019)](https://ieeexplore.ieee.org/abstract/document/8685687) — to understand the threat model before we implement it in Lesson 02.

**Modification challenge:** Open `core/utils/log.py` and add a timestamp prefix to every log message. The current `Log.__call__` method writes `msg` directly. Try modifying it to prepend `[HH:MM:SS]` to every call without touching the callers. Expected behavior change: every line in `log.txt` gets a time prefix automatically.

*Remember: the model's good behavior on clean data is the whole point — a backdoor that reduces benign accuracy will be caught by standard evaluation. The threat is specifically in what the model does when the trigger appears.* 🛡️
