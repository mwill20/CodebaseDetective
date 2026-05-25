# Codebase Detective — BackdoorBox Analysis

> Assignment: systematically explore, map, audit, and document an unfamiliar codebase in under 5 hours using Claude Code as the exploration partner.

**Target repository:** https://github.com/THUYimingLi/BackdoorBox  
**Analysis date:** 2026-05-25  
**Analyst:** Claude Code (claude-sonnet-4-6) + engineer pair

---

## Deliverables

| Artifact | Path | Description |
|---------|------|-------------|
| Full written report | [docs/REPORT.md](docs/REPORT.md) | All assignment questions answered with evidence |
| Architecture flowchart | [docs/diagrams/architecture.md](docs/diagrams/architecture.md) | Mermaid: system layers |
| Sequence diagram | [docs/diagrams/sequence.md](docs/diagrams/sequence.md) | Mermaid: BadNets training flow |
| Class + ER diagram | [docs/diagrams/class_diagram.md](docs/diagrams/class_diagram.md) | Mermaid: class hierarchy + data model |
| Interactive visualization | [docs/visualization/module_map.html](docs/visualization/module_map.html) | D3.js force-directed module map |
| CLAUDE.md | [CLAUDE.md](CLAUDE.md) | AI assistant instructions for BackdoorBox |

---

## Quick Summary

**What BackdoorBox is:** A Python ML security research toolbox implementing 15 backdoor attack methods and 12 defense methods against deep neural networks. Built on PyTorch. Used by ML security researchers at Tsinghua University and collaborators.

**Architecture in one paragraph:** All attacks inherit from `core/attacks/base.Base` which provides a complete training engine (train loop, LR scheduling, GPU setup, checkpointing, logging). Each attack subclass creates poisoned dataset wrappers that override `__getitem__` to inject triggers into a random fraction of training samples. Defenses inherit from the thinner `core/defenses/base.Base` and have more varied interfaces. The `core/__init__.py` re-exports everything under a single namespace.

**Bugs found (all verified):**

| # | Severity | Location | Description |
|---|---------|----------|-------------|
| 1 | Critical | `ShrinkPad.py:152` | Missing `import torch.nn as nn` — crashes on multi-GPU |
| 2 | High | `base.py:172` | Wrong variable in error message ("CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES") |
| 3 | High | `base.py:234` | `self.poisoned_train_dataset` used in benign training log — latent AttributeError |
| 4 | Medium | `base.py:128,339` | Colons in timestamp format crash directory creation on Windows |
| 5 | Low | `attacks/__init__.py:1` | Dead import `from ast import Import` |
| 6 | Low | `ShrinkPad.py:91,152` | Debug `breakpoint()` artifact + open `TODO: DDP` |

---

## Running the Visualization

Open `docs/visualization/module_map.html` in any browser. No server needed — it's a self-contained single-file D3.js application.

- Hover over any node for a description
- Drag nodes to rearrange
- Scroll to zoom
- Click "Toggle Labels" to reduce clutter

---

## Structure

```
CodeBaseDetective/
├── README.md
├── CLAUDE.md                          # AI assistant instructions
├── BackdoorBox/                       # Cloned target repo (gitignored)
└── docs/
    ├── REPORT.md                      # Full written report
    ├── diagrams/
    │   ├── architecture.md            # System layers flowchart
    │   ├── sequence.md                # Training sequence diagram
    │   └── class_diagram.md           # Class hierarchy + ER
    └── visualization/
        └── module_map.html            # Interactive D3.js module map
```
