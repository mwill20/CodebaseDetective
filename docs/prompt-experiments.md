# Prompt Experiments — Vague vs. Structured

**Project:** BackdoorBox Codebase Detective  
**Date:** 2026-05-25  

These experiments demonstrate how prompt specificity affects Claude's output quality when exploring an unfamiliar codebase. Each experiment shows a vague prompt, the failure mode it produces, the structured equivalent, and the improved output.

---

## Experiment 1: Bug Fix Request

### Vague Prompt

> "Fix the bug."

### What Claude Does with This Prompt

With no file loaded: Claude scans visible context for anything that looks buggy. Without a stack trace, error message, or file reference, it picks the most visible file (usually `core/__init__.py` or `Attack_BadNets.py`) and looks for surface-level issues. It may offer to "check the codebase for common Python bugs" — a statement that reveals it has no target.

With files loaded but no specific bug named: Claude picks the most visually salient issue in recently-read files. In this session it might have picked the dead import at `attacks/__init__.py:1` (easy to spot) while missing the latent crash at `base.py:234` (which requires tracing execution paths across files).

**Failure modes:**
- Asks 3+ clarifying questions, halting the workflow
- Guesses a random issue in whichever file happens to be freshest in context
- Hallucinates a plausible-looking fix to a non-existent bug
- Produces a real fix to a real bug but not the bug that matters most

### Why This Prompt Fails

"Fix the bug" assumes Claude shares your mental model of which bug is the problem. In a codebase with 6 verified bugs across 4 files, this prompt provides zero signal about which bug, which file, or what failure looks like.

---

### Structured Prompt

> "In `core/attacks/base.py` at line 234, the log message always references `self.poisoned_train_dataset` even when `benign_training=True`. A subclass that creates `poisoned_train_dataset` conditionally would hit `AttributeError` at the first log interval, not at construction time. Fix the log message to use the correct dataset variable depending on the current training mode."

### What Claude Does with This Prompt

- Goes directly to `base.py:234`
- Reads the surrounding context (the training loop, the `benign_training` flag logic)
- Identifies the correct fix: check `self.current_schedule['benign_training']` to select the right dataset
- Produces the one-line patch with no clarification questions

### Output Quality Comparison

| Dimension | Vague | Structured |
|-----------|-------|-----------|
| Questions asked before acting | 3+ | 0 |
| File navigated to | Random | Correct (base.py:234) |
| Fix produced | Incorrect or irrelevant | Correct one-line patch |
| Risk of hallucination | High | Low (file loaded, line explicit) |

### Pattern Applied

**Scoped fix:** File path + line number + current (broken) behavior + expected behavior + scope boundary.

---

## Experiment 2: Feature Addition for Non-Existent Component

### Vague Prompt

> "Rewrite the authentication module."

### What Claude Does with This Prompt

BackdoorBox has no authentication module. There are no users, sessions, tokens, or API boundaries — it's a research library run locally by one researcher at a time.

Without prior codebase context: Claude's training data is heavily weighted toward web applications. It pattern-matches "Python project" + "authentication module" and either:
1. Hallucinates a Flask-based session auth module that has nothing to do with this codebase
2. Offers a generic JWT implementation that references endpoints that don't exist

With prior context loaded: Claude correctly identifies that no auth module exists and asks for clarification. But the prompt still provides no productive path forward.

**Failure modes:**
- Fabricates a Flask/FastAPI auth layer for a library that has no web server
- Invents a `core/auth.py` file with a `validate_user()` function
- Produces syntactically correct Python that would cause an `ImportError` or does nothing useful

### Why This Prompt Fails

The prompt assumes a mental model (web application with auth) that doesn't apply here. It's built on domain confusion, not a real codebase gap. Claude's training data skews so heavily toward web applications that "Python + authentication" reliably triggers web-app patterns.

---

### Structured Prompt

> "BackdoorBox has no access control layer — any researcher with the code can call `badnets.train()` and run arbitrary experiments. If we needed to add a minimal guardrail that validates the caller has a recognized API key before starting a training run, where in the existing architecture would you add this check, and what would the simplest possible implementation look like using only the existing patterns in the codebase (no new frameworks)?"

### What Claude Does with This Prompt

- Correctly identifies that `Base.train()` is the single entry point for all training
- Recognizes the schedule dict as the existing configuration interface
- Proposes adding key validation at the `train()` method entry (lines 118-126 in base.py)
- Produces a minimal check that reads a key from the schedule dict or an environment variable
- Notes the limitation: this is not real security (no enforcement boundary, no token distribution mechanism)

### Output Quality Comparison

| Dimension | Vague | Structured |
|-----------|-------|-----------|
| Respects actual architecture | No | Yes |
| Hallucination risk | High | Low |
| Output is runnable | No (imports non-existent modules) | Yes (uses existing patterns) |
| Acknowledges limitations | No | Yes |

### Pattern Applied

**Verify-before-act:** Explicitly name the gap (no auth exists), acknowledge the architecture as-is, ask a bounded design question within the existing constraints.

---

## Experiment 3: Non-Existent Library Reference

### Vague Prompt

> "Use the TorchBackdoor library to improve trigger generation."

### What Claude Does with This Prompt

`TorchBackdoor` is not a real library. Claude pattern-matches `torch*` library names and synthesizes a plausible API:

```python
import torchbackdoor
trigger = torchbackdoor.generate_trigger(
    pattern_type='patch',
    size=(3, 3),
    position='bottom-right'
)
```

This code looks syntactically correct. It integrates cleanly with the existing BackdoorBox pattern. It will fail with `ModuleNotFoundError: No module named 'torchbackdoor'` on the first run.

**Why the hallucination is dangerous:**
- The code looks like working code — it's properly formatted and follows BackdoorBox conventions
- The API design is internally consistent — `generate_trigger()` makes sense as a function name
- The pattern matches real libraries like `torchattacks` that are already in the codebase
- A researcher who trusts the output without running it would integrate this into their pipeline

**Failure modes:**
- Fabricates `torchbackdoor` with a plausible-looking API
- Alternatively, fabricates a different library (`backdoor-toolkit`, `ml-security-lib`) that also doesn't exist
- Produces complete, integrated code that fails silently until the first import

### Why This Prompt Fails

The prompt names a specific third-party library. Claude cannot verify library existence at inference time. When the library name sounds plausible (matches naming conventions for real libraries), Claude confidently generates an API for it rather than admitting it can't verify the claim.

---

### Structured Prompt

> "The trigger blending in `AddDatasetFolderTrigger.__call__()` at `core/attacks/BadNets.py:65-119` only supports pixel-space blending (the formula is `(1-weight)*img + weight*pattern`). Could we improve this to also support frequency-domain blending — where the trigger is added in the DCT or FFT domain — using only the libraries already in `requirements.txt` (NumPy, SciPy, and OpenCV)?"

### What Claude Does with This Prompt

- Reads `BadNets.py:65-119` to understand the current blending formula
- Confirms the constraint: only NumPy, SciPy, OpenCV — all in `requirements.txt`
- Uses `numpy.fft.fft2` / `numpy.fft.ifft2` (frequency-domain transform)
- Produces a new subclass of `AddDatasetFolderTrigger` that performs the trigger blend in frequency space
- The output references real functions from real installed libraries

### Output Quality Comparison

| Dimension | Vague | Structured |
|-----------|-------|-----------|
| Library exists | No (hallucinated) | Yes (NumPy, SciPy, OpenCV) |
| Code is runnable | No | Yes |
| Tied to actual code location | No | Yes (BadNets.py:65-119) |
| Respects existing constraints | No | Yes |

### Pattern Applied

**Constrained output:** Name the exact file and function, explicitly list the allowed library set, describe the desired behavior in terms of the existing implementation.

---

## Experiment 4 (Bonus): Non-Existent Function in a Real File

### Vague Prompt

> "Call `core.attacks.base.validate_trigger()` before training to make sure the pattern is valid."

### What Claude Does with This Prompt

`validate_trigger()` does not exist in `core/attacks/base.py`. The actual public methods are: `train()`, `test()`, `get_model()`, `get_poisoned_dataset()`, `adjust_learning_rate()`.

**Without file context loaded:** Claude generates a plausible implementation:
```python
def validate_trigger(self, pattern, weight):
    assert pattern.shape == weight.shape, "Pattern and weight must have matching shapes"
    assert weight.min() >= 0.0 and weight.max() <= 1.0, "Weight must be in [0, 1]"
    return True
```

This code is reasonable. It would even be a useful addition. But it doesn't exist in the file, and the prompt implies it does.

**With the file context loaded:** Claude correctly stated "I don't see `validate_trigger()` in the methods I read from base.py." The presence or absence of the file in context is what determines whether Claude hallucinate-fills or honestly caveats.

### Why This Matters

This is the most operationally important experiment. It shows that **file context is the primary control against function-existence hallucinations**. Claude's behavior is not consistent — it depends on whether the relevant file was recently read. This means:

1. For any "does this function exist?" question, always use `Grep`, not Claude's memory
2. The same question asked at different points in a session (different context states) may produce different answers
3. Loading more files doesn't protect against hallucinations for functions in *unloaded* files

### Pattern Applied

**Verify-before-act:** Before calling any function, verify its existence with `Grep` or `Read`. Never trust Claude's statement that a function exists unless it was just read from the file.

---

## Summary: Prompt Pattern Reference

| Pattern | When to use | Example |
|---------|-------------|---------|
| **Scoped fix** | Debugging a known issue | "In `file.py:234`, the variable `X` is wrong when `Y=True`. Fix it to use `Z` instead." |
| **Verify-before-act** | Adding to or modifying existing architecture | "This codebase has no [component]. If we needed one, where would it fit and what's the minimal form?" |
| **Constrained output** | Implementing new functionality | "Implement this in the named file using only these named libraries." |
| **Behavioral framing** | Debugging unknown issues | "When I call `train()` with `benign_training=True`, I get `AttributeError` at iteration 100. Trace why." |
| **Existence-first** | Checking whether something exists | Run `Grep` first; use Claude only for interpreting what you found, not for asserting existence |

---

## Takeaway

The quality gap between vague and structured prompts is large and consistent. Every vague prompt in this session either:
- Required 3+ clarifying questions before acting (halting)
- Produced plausible-looking but incorrect code (dangerous)
- Applied the wrong mental model to the codebase (web app vs. research library)

Every structured prompt produced correct, runnable, codebase-appropriate output with no clarification needed. The key variables are: **named file**, **named line or function**, **current behavior**, **expected behavior**, and **explicit constraints**.
