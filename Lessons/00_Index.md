# BackdoorBox Curriculum Index

**Target audience:** Cybersecurity analyst transitioning to ML security engineer.  
**Codebase:** [BackdoorBox](https://github.com/THUYimingLi/BackdoorBox) — Python research toolbox for backdoor attacks and defenses against deep neural networks.  
**Prerequisites:** Basic Python, familiarity with PyTorch concepts (tensors, DataLoader), and security fundamentals (threat modeling, detection logic).

Each lesson reads the actual source files before writing — all code snippets are from the real codebase.

---

## Curriculum Table

| Lesson | Title | File(s) | Status | Type |
|--------|-------|---------|--------|------|
| 00 | [The Operations Manual: System Overview](Lesson00_System_Overview.md) | `README.md`, `core/__init__.py`, `requirements.txt` | ✅ Complete | Required |
| 01 | [The Training Engine: attacks/base.py](Lesson01_Training_Engine.md) | `core/attacks/base.py` | 🔲 Planned | Required |
| 02 | [Data Poisoning 101: BadNets](Lesson02_Data_Poisoning.md) | `core/attacks/BadNets.py`, `tests/test_BadNets.py` | 🔲 Planned | Required |
| 03 | [The Hidden Watermark: Blended Attack](Lesson03_Invisible_Attacks.md) | `core/attacks/Blended.py` | 🔲 Planned | Required |
| 04 | [Fingerprint in the Distortion: WaNet](Lesson04_Geometric_Attacks.md) | `core/attacks/WaNet.py` | 🔲 Planned | Optional |
| 05 | [Defense Architecture: ShrinkPad](Lesson05_Defense_Architecture.md) | `core/defenses/base.py`, `core/defenses/ShrinkPad.py` | 🔲 Planned | Required |
| 06 | [Quarantine During Training: ABL](Lesson06_Poison_Suppression.md) | `core/defenses/ABL.py` | 🔲 Planned | Required |
| 07 | [The Clean Room Rebuild: NAD](Lesson07_Model_Repair.md) | `core/defenses/NAD.py` | 🔲 Planned | Optional |
| 08 | [The Stress Test: SCALE-UP Detection](Lesson08_Input_Detection.md) | `core/defenses/SCALE_UP.py` | 🔲 Planned | Required |
| 09 | [The Vehicle Fleet: Neural Architectures](Lesson09_Neural_Architectures.md) | `core/models/resnet.py`, `core/models/vgg.py`, `core/models/autoencoder.py` | 🔲 Planned | Optional |
| 10 | [Red Team Meets Blue Team: Full Pipeline](Lesson10_Full_Pipeline.md) | `Attack_BadNets.py`, `Defense_ShrinkPad.py`, `core/utils/log.py` | 🔲 Planned | Required |

---

## Learning Path

```
[00] System Overview
      |
      v
[01] Training Engine  <-------  read this before any attack or defense lesson
      |
      +---> [02] BadNets  ---> [03] Blended  ---> [04] WaNet
      |          (patch)         (invisible)       (geometric)
      |
      +---> [05] Defense Architecture + ShrinkPad
                  |
                  +---> [06] ABL (training-time)
                  |
                  +---> [07] NAD (post-training repair)
                  |
                  +---> [08] SCALE-UP (inference-time detection)
                              |
                              v
                         [09] Neural Architectures (reference)
                              |
                              v
                         [10] Full Pipeline (capstone)
```

**Recommended order for first pass:** 00 -> 01 -> 02 -> 05 -> 06 -> 08 -> 10  
**Extended track (all methods):** 00 -> 01 -> 02 -> 03 -> 04 -> 05 -> 06 -> 07 -> 08 -> 09 -> 10

---

## What You Will Be Able to Do After This Curriculum

- Explain backdoor attacks to a non-technical stakeholder using concrete analogies
- Read and navigate any attack or defense class in BackdoorBox without guidance
- Identify the 6 verified bugs in the codebase and explain why each is dangerous
- Design a poisoned dataset experiment from scratch (trigger, rate, target class)
- Evaluate a backdoored model using the schedule dict interface
- Connect each defense type (pre-processing, training-time, post-training, inference-time) to the threat it addresses
- Ask good interview questions about ML security in a technical hiring screen

---

## Notes for Learners

- **Lessons don't require a GPU to read.** The exercises flag which ones need compute.
- **The `tests/` folder is the official documentation.** When a lesson references how to use a method, it's because `tests/test_X.py` shows exactly that.
- **No benchmark numbers are fabricated.** Where attack success rates or accuracy figures appear, they reference the source papers, not claimed performance of this specific installation.
- **Bugs documented in CLAUDE.md are real.** Lessons point them out in context so you see them as you learn, not as gotchas afterward.
