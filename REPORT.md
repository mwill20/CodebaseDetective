# Codebase Detective: BackdoorBox Analysis Report

**GitHub Repository:** https://github.com/mwill20/CodebaseDetective  
**Target codebase:** https://github.com/THUYimingLi/BackdoorBox  
**Analyst:** Claude Code (claude-sonnet-4-6) + Michael Williams  
**Date:** 2026-05-25  

Full detailed analysis: [docs/REPORT.md](docs/REPORT.md)  
Bug audit: [docs/bug-audit.md](docs/bug-audit.md)  
Prompt experiments: [docs/prompt-experiments.md](docs/prompt-experiments.md)  
Exploration log: [docs/exploration-log.md](docs/exploration-log.md)

---

## Q1: What is this codebase? Describe its purpose, intended audience, and primary use case.

BackdoorBox is an open-source ML security research toolbox from Tsinghua University. It implements **15 backdoor attack methods** and **12 backdoor defense methods** against deep neural networks under a unified PyTorch framework.

A **backdoor attack** is a training-time supply chain attack: an attacker poisons a fraction of training data with a trigger pattern and relabels those samples to a target class. The resulting model behaves normally on clean inputs but misclassifies any input containing the trigger into the target class. This is a hidden "unlock code" baked into the model's weights.

**Intended audience:** ML security researchers evaluating attack success rates (ASR) and defense effectiveness. Not a production application — there are no web endpoints, no database, no user authentication, no REST API. Configuration is done by editing Python constants directly in experiment scripts.

**Primary use case:** Running controlled experiments comparing attack methods and defenses on standard vision datasets (CIFAR-10, MNIST, GTSRB, ImageNet subsets).

---

## Q2: What is the complete tech stack?

| Layer | Technology |
|-------|-----------|
| Language | Python 3.8 |
| ML Framework | PyTorch 1.8.0 + CUDA 11.1 |
| Image processing | torchvision 0.9.0, opencv-python 4.12, Pillow 10.4 |
| Scientific compute | NumPy 1.24, SciPy 1.10 |
| Clustering/dim-reduction | scikit-learn 1.3, hdbscan 0.8, umap-learn 0.5 |
| Perceptual similarity | lpips 0.1.4 |
| Face/landmark detection | dlib 20.0 |
| Contrastive learning | OpenAI CLIP (from GitHub, unpinned) |
| Adversarial attacks | torchattacks PGD (vendored locally) |
| Build acceleration | ninja 1.13 |

**Notable security flag:** The CLIP dependency uses `git+https://github.com/openai/CLIP.git` with no commit hash. Any push to that repo's default branch will be pulled automatically on the next `pip install -r requirements.txt`. This is a real (low-probability but real) supply chain exposure.

---

## Q3: Map the system architecture. Describe the key layers and how they interact.

```
User Script (Attack_*.py / tests/test_*.py)
    |
    v
core/__init__.py  -- 3-line facade: from .attacks import *; from .defenses import *; from .models import *
    |
    +---> attacks/base.Base         (training engine: GPU setup, training loop, LR scheduling, checkpointing, logging)
    |         |
    |         +---> BadNets / Blended / WaNet / IAD / LIRA / ... (15 attack subclasses)
    |                   |
    |                   +---> PoisonedDataset* wrappers (override __getitem__ to inject triggers at index lookup time)
    |
    +---> defenses/base.Base        (seed management only -- 43 lines)
    |         |
    |         +---> ShrinkPad / ABL / NAD / SCALE_UP / ... (12 defense subclasses, each implements its own evaluation)
    |
    +---> models/*                  (ResNet-18/34, VGG-11/13/16/19, AutoEncoder, UNet, BaselineMNISTNetwork)
    |
    +---> utils/*                   (Log, accuracy, test runner, PGD adversarial attack)
```

**Key design pattern:** Template Method. `attacks/base.Base.train()` owns the full training loop. Attack subclasses do one thing: create `self.poisoned_train_dataset` and `self.poisoned_test_dataset` in `__init__`. The base class reads these at runtime. This means all 15 attacks share bug fixes, logging improvements, and GPU setup changes automatically.

Diagrams are at:
- [docs/diagrams/architecture.md](docs/diagrams/architecture.md) — system layers flowchart
- [docs/diagrams/sequence.md](docs/diagrams/sequence.md) — BadNets training sequence
- [docs/diagrams/class_diagram.md](docs/diagrams/class_diagram.md) — class hierarchy + data model

---

## Q4: What are the entry points? How would someone run an experiment?

| Entry point | Command | What it does |
|-------------|---------|-------------|
| `Attack_BadNets.py` | `python Attack_BadNets.py` | Trains benign then backdoored ResNet-18 on CIFAR-10 |
| `Attack_Blended.py` | `python Attack_Blended.py` | Trains Blended attack on ImageNet50 |
| `Defense_ShrinkPad.py` | `python Defense_ShrinkPad.py` | Evaluates ShrinkPad defense on a pre-trained model |
| `tests/test_BadNets.py` | `python tests/test_BadNets.py` | More current than Attack_BadNets.py; supports MNIST, CIFAR-10, GTSRB |

There is no CLI, no config file parser, no `main.py`. All parameters (learning rate, batch size, trigger pattern, poisoning rate, target class) are set by editing Python constants in the entry scripts.

The `tests/` folder is the real user documentation — more up-to-date than the root-level entry scripts and showing more configuration options.

---

## Q5: Trace a full operation end-to-end.

**Operation:** `python Attack_BadNets.py` — full BadNets poisoned training on CIFAR-10.

1. **Import** (`Attack_BadNets.py:9`): `import core` loads `core/__init__.py`, which re-exports all attacks, defenses, and models via wildcard imports.

2. **Dataset loading** (`Attack_BadNets.py:29-46`): `DatasetFolder(root='./data/cifar10/train', loader=cv2.imread)` — CIFAR-10 images on disk as PNGs, loaded with OpenCV.

3. **Trigger definition** (`Attack_BadNets.py:58-61`): A 3x3 white square in the bottom-right corner of the 32x32 image space. Weight=1.0 means full replacement (not blending).

4. **Attack construction** (`BadNets.py:414-456` -> `base.py:60-70`): `BadNets.__init__()` calls `Base.__init__()` to store datasets and model, then calls `CreatePoisonedDataset()` twice to build train and test poisoned wrappers. 10% of training indices (random shuffle, then first 10%) go into a `frozenset` called `poisoned_set`.

5. **Poisoned dataset logic** (`BadNets.py:207-264`): `PoisonedDatasetFolder.__getitem__(index)` checks `index in self.poisoned_set` (O(1) frozenset lookup). If poisoned: apply `AddDatasetFolderTrigger` (blend trigger into image) and `ModifyTarget` (relabel to class 0). If clean: apply normal augmentations.

6. **Training** (`base.py:118-273`): `train()` resolves the schedule dict, creates a timestamped experiment directory, sets up GPU/CPU, builds DataLoader from `self.poisoned_train_dataset`, constructs SGD optimizer, runs the epoch loop with step-decay LR scheduling.

7. **Evaluation** (`base.py:238-260`): Every 10 epochs, `_test()` runs on both `test_dataset` (measures BA — benign accuracy) and `poisoned_test_dataset` (measures ASR — attack success rate). ASR climbs from ~random to >90% over training.

8. **Output**: `experiments/train_poisoned_DatasetFolder-CIFAR10_<TIMESTAMP>/` containing `log.txt` and checkpoint `.pth` files every 10 epochs.

---

## Q6: Create at least 3 architectural diagrams.

All three diagrams are in `docs/diagrams/`:

**[Architecture flowchart](docs/diagrams/architecture.md):** Mermaid diagram showing the 6-layer system: User Scripts -> Package Facade -> Attack Base/Defense Base/Models -> Dataset Wrappers/Utils -> PyTorch/torchvision -> GPU/CPU Hardware.

**[Sequence diagram](docs/diagrams/sequence.md):** Mermaid sequence diagram tracing a complete BadNets poisoned training run with file:line references at each step, from dataset construction through checkpoint saving.

**[Class hierarchy + ER diagram](docs/diagrams/class_diagram.md):** Mermaid class diagram showing the full inheritance tree (all 15 attacks, 12 defenses, 5 models) plus an Entity-Relationship diagram of the Schedule and Checkpoint data models.

**Interactive visualization:** [docs/visualization/module_map.html](docs/visualization/module_map.html) — self-contained D3.js force-directed graph with 40 nodes and color-coded edge types (inheritance = green, composition = orange, import = dashed blue). Open in any browser; no server needed.

---

## Q7: What happens when you give Claude vague or underspecified prompts?

Three prompt failures documented in [docs/prompt-experiments.md](docs/prompt-experiments.md):

**"Fix the bug":** With no file, no line, and no symptom specified, Claude either asks 3+ clarifying questions (halting work) or guesses at the most visually salient issue in recently-loaded context. In a codebase with 6 bugs across 4 files, this produces a random outcome.

**"Rewrite the authentication module":** BackdoorBox has no authentication layer. Claude's training data is heavily web-application-weighted. Without file context, it hallucinated a Flask-based session auth module complete with JWT handling — syntactically correct Python code that has nothing to do with this codebase.

**"Use the TorchBackdoor library for trigger generation":** `TorchBackdoor` does not exist. Claude pattern-matched `torch*` naming conventions and fabricated a plausible API: `torchbackdoor.generate_trigger(pattern_type='patch', size=(3,3))`. The code was formatted correctly, integrated cleanly with BackdoorBox patterns, and would fail with `ModuleNotFoundError` on the first import.

---

## Q8: How do you detect when Claude is hallucinating about a codebase?

Three hallucination detection methods, with examples from this session:

**Method 1 — File existence check with Glob:**
Tested prompt: "What does `core/utils/metrics.py` do?"
`Glob("**/metrics.py")` returned no results. The file doesn't exist. Claude had described it in detail based on the README's discussion of BA and ASR metrics.

**Method 2 — Function existence check with Grep:**
Tested prompt: "Call `core.attacks.base.validate_trigger()` before training."
`Grep("validate_trigger", "core/attacks/base.py")` returned no results. The function doesn't exist. Claude had generated a plausible 3-line implementation without flagging that it was fabricated.

**Method 3 — Library existence verification:**
Tested prompt: "Use `torch.nn.BackdoorDetector` to scan the model after training."
PyTorch 1.8 has no `BackdoorDetector` class. Claude had written code using it with a plausible API. Verification: grepping the PyTorch docs and source — no match.

**Key finding:** The presence of the file in the active context is the primary control against function-existence hallucinations. With `base.py` loaded in context, Claude correctly stated "I don't see `validate_trigger()` in the methods I read." Without the file in context, it hallucinated one. For any existence check, always use the file system tool — not Claude's recall.

---

## Q9: How does prompt structure affect response quality? Show 3+ before/after pairs.

See [docs/prompt-experiments.md](docs/prompt-experiments.md) for full analysis. Summary table:

| Vague Prompt | Failure Mode | Structured Prompt | Outcome |
|-------------|-------------|------------------|---------|
| "Fix the bug" | Random guess or 3+ clarifying questions | "In `base.py:234`, the log message uses `self.poisoned_train_dataset` even when `benign_training=True`. Fix it to select the right dataset based on the current mode." | One-line correct patch, no questions |
| "Rewrite the authentication module" | Fabricated Flask/JWT auth for a library with no web server | "BackdoorBox has no auth layer. If we needed a minimal API key check before `Base.train()`, where would it go using existing patterns?" | Correct architectural answer, honest about limitations |
| "Use the TorchBackdoor library" | Fabricated `torchbackdoor.generate_trigger()` API | "Improve frequency-domain trigger blending in `BadNets.py:65-119` using only NumPy, SciPy, and OpenCV (already in requirements.txt)" | Real working code using `numpy.fft` |

**Prompt improvement patterns:** (1) Scoped fix: file + line + current behavior + expected behavior. (2) Verify-before-act: name the gap before asking for a solution. (3) Constrained output: name allowed libraries explicitly.

---

## Q10: What bugs did you find? Are they real or hallucinated?

**All 6 bugs are real.** Zero hallucinated bugs are reported. Full audit at [docs/bug-audit.md](docs/bug-audit.md).

| ID | Severity | File | Line | Description |
|----|---------|------|------|-------------|
| BUG-01 | Critical | `ShrinkPad.py` | 152 | `nn.DataParallel` called but `torch.nn` never imported as `nn`. Multi-GPU ShrinkPad raises `NameError`. |
| BUG-02 | High | `base.py` | 172 | Error message says "CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!" — first variable should be CUDA_SELECTED_DEVICES. |
| BUG-03 | High | `base.py` | 234 | Log message references `self.poisoned_train_dataset` even during `benign_training=True` runs. Latent AttributeError in subclasses. |
| BUG-04 | Medium | `base.py` | 128, 339 | `time.strftime("%Y-%m-%d_%H:%M:%S")` produces colons, which crash `os.makedirs` on Windows. |
| BUG-05 | Low | `attacks/__init__.py` | 1 | `from ast import Import` — dead import of Python's AST node class, never used. |
| BUG-06 | Low | `ShrinkPad.py` | 91, 152 | Commented-out `breakpoint()` debug artifact + open `# TODO: DDP training` never tracked in issues. |

Verification method: each bug was confirmed by reading the exact line from the actual source file and tracing the execution path to confirm the failure condition.

---

## Q11: How does context window size and content affect Claude's accuracy?

**Experiment setup:** Same question ("How does the training loop work?") asked in two conditions:
1. Broad: 11 files loaded (README, `__init__`, base.py, BadNets.py, ShrinkPad.py, ABL.py, all `__init__` files, log.py, Attack_BadNets.py)
2. Narrow: Only `core/attacks/base.py` loaded

**Results:**

| Dimension | Broad (11 files) | Narrow (1 file) |
|-----------|-----------------|----------------|
| Accuracy | High | High (within scope) |
| Completeness | Full end-to-end | Incomplete on dataset construction |
| Hallucination risk | Low | Low within scope |
| Caveats quality | Moderate | Better — explicitly stated cross-file limitations |

**Key finding:** Narrow scope with explicit caveats was more reliable. The narrow-context response correctly said "the poisoned dataset construction is delegated to subclasses; see the specific attack file." The broad-context response risked overconfidence on cross-file details.

**What I learned:**
1. **Context window is not searchable.** The model attends over a flat token sequence — a function in file 8 of 11 may get less attention than one near the start.
2. **Absent files get replaced by training data.** When a function is not in context, Claude substitutes a plausible one from domain knowledge. This looks identical to a correct answer.
3. **Use `/clear` strategically.** Loading fewer, more relevant files often produces cleaner, more verifiable answers than loading everything.
4. **For existence checks, use file system tools, not Claude's recall.** Claude's memory of what it read is not the same as the file content.

---

## Q12: What documentation and visualization artifacts did you produce?

| Artifact | Path | Description |
|---------|------|-------------|
| AI assistant instructions | [CLAUDE.md](CLAUDE.md) | Environment, architecture, schedule dict contract, 6 verified bugs with fixes |
| Architecture flowchart | [docs/diagrams/architecture.md](docs/diagrams/architecture.md) | Mermaid: system layers |
| Sequence diagram | [docs/diagrams/sequence.md](docs/diagrams/sequence.md) | Mermaid: BadNets training flow with file:line annotations |
| Class hierarchy + ER | [docs/diagrams/class_diagram.md](docs/diagrams/class_diagram.md) | Mermaid: inheritance tree + data model |
| Interactive visualization | [docs/visualization/module_map.html](docs/visualization/module_map.html) | D3.js: 40-node force-directed graph, hover/zoom/drag |
| Full written report | [docs/REPORT.md](docs/REPORT.md) | All 4 parts of the assignment with code traces and line references |
| Bug audit | [docs/bug-audit.md](docs/bug-audit.md) | 6 bugs with severity, evidence, verification status, and fixes |
| Prompt experiments | [docs/prompt-experiments.md](docs/prompt-experiments.md) | 4 vague vs. structured prompt pairs with output analysis |
| Exploration log | [docs/exploration-log.md](docs/exploration-log.md) | Timestamped session log with key prompts and findings |
| Lesson curriculum | [Lessons/00_Index.md](Lessons/00_Index.md) | 11-lesson educational curriculum for BackdoorBox |
| Lesson 00 | [Lessons/Lesson00_System_Overview.md](Lessons/Lesson00_System_Overview.md) | Full lesson: system overview, architecture, 4 exercises, interview prep |
| Lesson 01 | [Lessons/Lesson01_Training_Engine.md](Lessons/Lesson01_Training_Engine.md) | Full lesson: base.py walkthrough, schedule dict, 3 bugs, 4 exercises |

**CLAUDE.md coverage:** Environment setup, full directory breakdown, architecture pattern explanation, complete schedule dict key reference, all 6 bugs with file:line and fix, data requirements, AI assistant rules (no fabricated numbers, treat tests/ as documentation).

**Interactive visualization notes:** `module_map.html` is self-contained (no server needed). Open in any browser. Nodes are color-coded by type (attacks = red, defenses = blue, models = green, utils = gray). Edge types: inheritance (green), composition (orange), import (dashed blue). Hover for descriptions, drag to rearrange, scroll to zoom.
