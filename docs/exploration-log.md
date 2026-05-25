# Exploration Session Log

**Project:** BackdoorBox Codebase Detective  
**Date:** 2026-05-25  
**Analyst:** Claude Code (claude-sonnet-4-6) + Michael Williams  
**Target repo:** https://github.com/THUYimingLi/BackdoorBox  

---

## Session Timeline

### 09:00 — Initial orientation

**Prompt:** "Clone BackdoorBox and tell me what kind of project this is."

**Action:** `git clone https://github.com/THUYimingLi/BackdoorBox.git`

**Finding:** BackdoorBox is an ML security research toolbox implementing 15 backdoor attack methods and 12 defenses against DNNs. Built on PyTorch. From Tsinghua University. Not a web service — no API endpoints, no database, no user authentication. All "users" are ML researchers running experiments locally.

---

### 09:05 — Read the README and requirements

**Prompt:** Read `README.md` and `requirements.txt` simultaneously to get purpose and tech stack.

**Actions:** Read both files in parallel.

**Findings:**
- Python 3.8, PyTorch 1.8.0+cu111, CUDA 11.1 — research-grade, not current versions
- Key unusual deps: OpenAI CLIP (from GitHub, unpinned), dlib, hdbscan, umap-learn, lpips
- CLIP dep uses `git+https://github.com/openai/CLIP.git` with no commit hash — supply chain risk flagged
- README has `TODO: benchmark coming soon` — no fabricated numbers
- All 15 attacks and 12 defenses listed by name

---

### 09:12 — Directory structure mapping

**Prompt:** List all `.py` files in `core/`, show directory tree.

**Action:** Glob `core/**/*.py`

**Findings:**
- `core/__init__.py` — 3-line facade: `from .attacks import *; from .defenses import *; from .models import *`
- `core/attacks/` — 15 files, each one attack class
- `core/defenses/` — 12 files, each one defense class
- `core/models/` — resnet, vgg, autoencoder, unet, mnist_network
- `core/utils/` — log, accuracy, test runner, torchattacks/PGD (vendored)
- `tests/` — 30+ example scripts (one per attack/defense) — identified as the real documentation

---

### 09:20 — Read the base class (most important file)

**Prompt:** Read `core/attacks/base.py` completely.

**Finding (major):** Base class is a full 410-line PyTorch training engine. All 15 attacks inherit from it. Subclasses only need to create a poisoned dataset; `Base.train()` does all the rest.

**Finding (bug):** Line 172 — error message says "CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!" — both variables in the message are the same. Should say CUDA_SELECTED_DEVICES for the left side.

**Finding (bug):** Line 128 — `time.strftime("%Y-%m-%d_%H:%M:%S")` uses colons, which are illegal in Windows directory names. Will crash `os.makedirs()` on Windows.

**Finding (bug):** Line 234 — log message inside training loop references `self.poisoned_train_dataset` even when `benign_training=True`. Latent AttributeError if a subclass omits the poisoned dataset.

---

### 09:35 — Read the most-studied attack (BadNets)

**Prompt:** Read `core/attacks/BadNets.py` completely.

**Findings:**
- `AddTrigger` base class: blending formula `(self.weight * img + self.res).type(torch.uint8)`
- Three dataset wrapper classes: `PoisonedDatasetFolder`, `PoisonedMNIST`, `PoisonedCIFAR10`
- Each wrapper overrides `__getitem__` to inject trigger at index lookup time — disk never modified
- `frozenset` for poisoned indices = O(1) membership check
- `CreatePoisonedDataset()` factory function dispatches based on dataset class name
- Test poisoning rate is always 1.0 (all test samples get trigger for ASR measurement)

---

### 09:50 — Read attacks/__init__.py

**Prompt:** Read `core/attacks/__init__.py`.

**Finding (bug):** Line 1: `from ast import Import` — dead import. `ast.Import` is an AST node class for parsing Python source, not related to the attacks module. Almost certainly an abandoned code-generation artifact.

---

### 09:55 — Read the defense base class

**Prompt:** Read `core/defenses/base.py`. Compare to attacks base.

**Finding:** Defenses base is only 43 lines — just `__init__` and `_set_seed()`. Much thinner than the attacks base because defenses have too much structural variation to share a training loop. This is an intentional design asymmetry, not a bug.

---

### 10:05 — Read ShrinkPad (simplest defense)

**Prompt:** Read `core/defenses/ShrinkPad.py`.

**Finding (bug — critical):** Line 152: `model = nn.DataParallel(...)` but `torch.nn` is never imported as `nn` anywhere in the file. Imports show only `import torch`, not `import torch.nn as nn`. Any multi-GPU ShrinkPad run raises `NameError: name 'nn' is not defined`.

**Finding (bug):** Line 91: `# breakpoint()` — commented-out debugging artifact left in the code.

**Finding (bug):** Lines 152-153: `# TODO: DDP training` followed by `pass` — open work item, never filed as an issue.

---

### 10:15 — Read a partial defense (ABL)

**Prompt:** Read first 80 lines of `core/defenses/ABL.py`.

**Finding:** Anti-Backdoor Learning uses a novel `LGALoss` (local gradient ascent) class to suppress backdoor learning during training. `TensorsDataset` wrapper shows ABL maintains its own mini-dataset of suspected poisoned samples during training. This is training-time defense vs. ShrinkPad's pre-processing approach.

---

### 10:25 — Read the entry script

**Prompt:** Read `Attack_BadNets.py`.

**Finding:** Full two-phase experiment pattern: first call `train()` with `benign_training=True` (baseline), then call `train()` with `benign_training=False` (attack). Same object, same schedule hyperparameters, only the flag changes.

**Finding:** CIFAR-10 is loaded as a `DatasetFolder` with `loader=cv2.imread` — images are PNG files on disk, not the built-in CIFAR-10 download. This is an unusual setup that isn't documented in the README.

---

### 10:40 — Read the test file for additional context

**Prompt:** Read `tests/test_BadNets.py`.

**Finding:** Shows 3 configurations: MNIST with `BaselineMNISTNetwork`, CIFAR-10 with ResNet-18, GTSRB with ResNet-18(43 output classes). The GTSRB config uses `poisoned_transform_train_index=2` — trigger inserted after Resize, not at index 0, because GTSRB images need resizing first.

**Finding:** `global_seed = 666`, `deterministic = True` — confirms this is a reproducible research tool.

---

### 10:55 — Architecture and diagram creation

**Prompt:** Create architecture flowchart, sequence diagram, and class/ER diagram as Mermaid files.

**Actions:** Created `docs/diagrams/architecture.md`, `docs/diagrams/sequence.md`, `docs/diagrams/class_diagram.md`.

**Verification:** All diagrams render correctly on Mermaid Live Editor (mermaid.live).

---

### 11:20 — Interactive visualization

**Prompt:** Create a self-contained D3.js force-directed graph of all modules.

**Action:** Created `docs/visualization/module_map.html` (900+ lines).

**Findings:**
- 40 nodes: 15 attacks, 12 defenses, 5 models, 5 utils, 4 entry points, 3 dataset types
- 3 edge types: inheritance (green), composition (orange), import/dependency (dashed blue)
- Features: hover tooltips, drag-to-rearrange, scroll-to-zoom, toggle-labels button

---

### 11:45 — Bug hallucination verification

**Prompt:** For each of the 6 bugs found, verify by quoting the exact line from the actual file.

**Action:** Re-read the relevant lines from `base.py`, `ShrinkPad.py`, `attacks/__init__.py`.

**Result:** All 6 bugs confirmed by direct file reading. No hallucinations introduced.

---

### 12:00 — CLAUDE.md generation

**Prompt:** Generate a comprehensive CLAUDE.md for BackdoorBox.

**Action:** Created `CLAUDE.md` covering environment, directory structure, architecture pattern, schedule dict contract (all required + optional keys), all 6 verified bugs with fixes, data requirements, AI assistant notes.

---

### 12:20 — Prompt experiment documentation

**Prompts tested:**
1. "Fix the bug." — vague; Claude asked 3 clarifying questions or guessed randomly
2. "Rewrite the authentication module." — wrong mental model; no auth in this codebase
3. "Use the TorchBackdoor library for trigger generation." — non-existent library; prompted hallucination

**Action:** Improved all 3 prompts and compared results. Documented in `docs/prompt-experiments.md`.

---

### 12:45 — Session wrap-up and report

**Action:** Compiled all findings into `docs/REPORT.md` (700+ lines covering all 4 assignment parts).

**Action:** Generated `README.md` with deliverables table, bug summary, and visualization instructions.

---

## Key Findings Summary

| Category | Finding |
|----------|---------|
| Architecture | Template Method pattern: `Base` owns the training loop, subclasses only create datasets |
| Architecture | `core/__init__.py` is a 3-line facade — clean API boundary |
| Bug (Critical) | `ShrinkPad.py:152` — missing `nn` import crashes multi-GPU runs |
| Bug (High) | `base.py:172` — wrong variable name in error message |
| Bug (High) | `base.py:234` — latent AttributeError in benign training path |
| Bug (Medium) | `base.py:128,339` — colons in timestamp crash Windows directory creation |
| Bug (Low) | `attacks/__init__.py:1` — dead import from Python AST module |
| Bug (Low) | `ShrinkPad.py:91,152` — debug artifact + open TODO |
| Supply chain | CLIP dependency unpinned (no commit hash) |
| Design | `tests/` folder is the real documentation — not unit tests |
| Design | `frozenset` for poisoned indices = O(1) lookup per sample |

---

## Prompt Engineering Observations

1. **Vague prompts ("fix the bug") consistently produced worse results** than prompts specifying file, line, symptom, and expected behavior.
2. **Absent context triggers hallucination.** When a function isn't loaded in context, Claude substitutes a plausible one from domain knowledge. The substitution looks syntactically correct.
3. **The model doesn't distinguish "file is in context" from "I read the file carefully."** For existence checks, always use Grep, not Claude's recall.
4. **Constraint improves output.** "Use only libraries already in requirements.txt" produced verifiable code; the unconstrained prompt produced a fabricated library name.
