# BackdoorBox — CLAUDE.md

> Project-level instructions for AI assistants working in this directory.

## Project Overview

**BackdoorBox** is a Python research toolbox for backdoor attacks and defenses against deep neural networks (DNNs). It implements 15 attack methods and 12 defense methods under a unified framework built on PyTorch.

**This is an ML security research tool — not a production application.** There are no web servers, REST APIs, databases, or authentication systems. The "users" are ML researchers running experiments.

## Environment

| Requirement | Version |
|-------------|---------|
| Python      | 3.8 |
| PyTorch     | 1.8.0+cu111 |
| torchvision | 0.9.0+cu111 |
| CUDA        | 11.1 |
| Key deps    | opencv-python, numpy, scipy, scikit-learn, hdbscan, umap-learn, dlib, CLIP |

Install: `conda create -n backdoorbox python=3.8 && pip install -r requirements.txt`

## Repository Structure

```
BackdoorBox/
├── Attack_BadNets.py       # Root-level entry point: BadNets attack on CIFAR-10
├── Attack_Blended.py       # Root-level entry point: Blended attack on ImageNet50
├── Defense_ShrinkPad.py    # Root-level entry point: ShrinkPad defense evaluation
├── requirements.txt
├── core/
│   ├── __init__.py         # Re-exports attacks.*, defenses.*, models.*
│   ├── attacks/
│   │   ├── base.py         # CRITICAL: Base class — train(), test(), logging, GPU setup
│   │   ├── BadNets.py      # Patch trigger (most studied baseline)
│   │   ├── Blended.py      # Alpha-blended invisible trigger
│   │   ├── WaNet.py        # Warping-based invisible trigger
│   │   ├── IAD.py          # Dynamic sample-specific trigger
│   │   ├── LIRA.py         # Learnable imperceptible trigger
│   │   └── ...             # 10 more attack implementations
│   ├── defenses/
│   │   ├── base.py         # Minimal base: _set_seed() only
│   │   ├── ShrinkPad.py    # Pre-processing: shrink + random pad
│   │   ├── ABL.py          # Anti-Backdoor Learning (NeurIPS 2021)
│   │   ├── NAD.py          # Neural Attention Distillation
│   │   └── ...             # 8 more defense implementations
│   ├── models/
│   │   ├── resnet.py       # ResNet-18/34 — primary model used in examples
│   │   ├── vgg.py          # VGG-11/13/16/19
│   │   ├── autoencoder.py  # Used by AutoEncoderDefense
│   │   ├── unet.py         # Used by ISSBA steganography
│   │   └── ...
│   └── utils/
│       ├── log.py          # Log class: stdout + file writer
│       ├── accuracy.py     # top-k accuracy
│       ├── test.py         # standalone test helper for defenses
│       └── torchattacks/   # PGD attack (borrowed from torchattacks library)
└── tests/                  # 30+ example scripts — one per attack/defense
```

## Architecture Pattern

All attacks inherit from `core/attacks/base.py:Base`. This base class:
1. Validates dataset types (only `DatasetFolder`, `MNIST`, `CIFAR10` supported)
2. Provides `train()` — full training loop with LR scheduling, logging, checkpointing
3. Provides `test()` and `_test()` — evaluation on benign + poisoned datasets
4. Handles multi-GPU setup via `CUDA_SELECTED_DEVICES` filtering

Each attack subclass:
1. Creates `self.poisoned_train_dataset` and `self.poisoned_test_dataset` in `__init__`
2. The poisoned datasets override `__getitem__` to inject triggers on a random subset
3. Optionally overrides `train()` for methods needing custom training loops (IAD, LIRA, Blind)

**Poisoning flow:** `PoisonedDataset.__init__()` shuffles indices → selects `poisoned_set` (frozenset) → inserts `AddTrigger` transform at a specified pipeline index → inserts `ModifyTarget` to relabel poisoned samples.

## Schedule Configuration Dict

Every `train()` and `test()` call takes a `schedule` dict. Required keys:
- `device`: `"GPU"` or `"CPU"`
- `benign_training`: `True` (train clean model) or `False` (train backdoored model)
- `batch_size`, `num_workers`, `lr`, `momentum`, `weight_decay`, `gamma`, `schedule` (LR milestones), `epochs`
- `log_iteration_interval`, `test_epoch_interval`, `save_epoch_interval`
- `save_dir`: output root (e.g. `"experiments"`)
- `experiment_name`: subfolder name

Optional:
- `CUDA_VISIBLE_DEVICES`, `CUDA_SELECTED_DEVICES`, `GPU_num`
- `pretrain`: path to `.pth` for weight initialization
- `warmup_epoch`: linear LR warmup
- `test_model`: path to `.pth` for eval-only mode
- `metric`: e.g. `"ASR_NoTarget"`

## Known Bugs (as of 2026-05-25)

1. **`attacks/base.py:234`** — Log message references `self.poisoned_train_dataset` even when `benign_training=True`. Works in BadNets (always sets both datasets) but will crash if `Base` is subclassed without setting `poisoned_train_dataset`.

2. **`attacks/base.py:171-172`** — Error message typo: `"CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!"` should read `"CUDA_SELECTED_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!"`.

3. **`attacks/__init__.py:1`** — Dead import: `from ast import Import`. The `ast.Import` AST node class is unused; this import is harmless but confusing.

4. **`defenses/ShrinkPad.py:152`** — `nn.DataParallel` is called in the multi-GPU branch but `torch.nn as nn` is not imported. Will raise `NameError: name 'nn' is not defined` when using multi-GPU with `ShrinkPad.predict()`.

5. **`attacks/base.py:128`** — Timestamp format `%H:%M:%S` includes colons, which are illegal in Windows file paths. Experiments run on Windows will fail at `os.makedirs(work_dir)`. Not an issue on Linux/macOS.

## Data Requirements

Datasets are NOT included. They must be downloaded separately and placed under `data/`:
```
data/
├── cifar10/
│   ├── train/  (class subdirectories of PNG images)
│   └── test/
└── ImageNet50/
    ├── train/
    └── val/
```

CIFAR-10 can be auto-downloaded via torchvision but must then be exported as PNG files per class. ImageNet50 requires manual download from ImageNet.

## Running Experiments

Do not run experiments during code review — they require GPU compute and large datasets. For quick syntax/logic checks only:
```bash
python -c "import core; print(dir(core))"
```

## AI Assistant Notes

- **Never fabricate benchmark numbers.** The README intentionally has a `TODO: benchmark coming soon` note. Any accuracy/ASR numbers must come from actual experiment logs.
- **Treat `schedule` dict as an interface contract.** Missing keys cause `KeyError` deep in training loops.
- **The `tests/` folder is documentation.** When asked "how do I use X?", read the corresponding `tests/test_X.py` file.
- **Attack/defense pairing.** Not all defenses work against all attacks. Refer to each method's paper for the evaluation scope.
