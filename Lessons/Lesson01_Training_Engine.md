# Lesson 01: The Command Center — `core/attacks/base.py`

## Welcome Back, Security Engineer!

You have read the operations manual. You know BackdoorBox trains backdoored models and the trigger lives in the data. But what actually runs that training? How does every attack share the same loop without duplicating 400 lines of PyTorch boilerplate?

Today we dissect `core/attacks/base.py` — the command center that powers all 15 attacks. Every attack you will study in this curriculum calls this code. Read it once, and you can read any attack in the codebase without help.

---

## Learning Objectives

By the end of this lesson you will be able to:

- Explain why `Base` exists and what design problem it solves
- Interpret any schedule dict and predict what `train()` will do with it
- Trace the flow from `benign_training=True` to `benign_training=False` and explain why both paths go through the same class
- Identify BUG-02, BUG-03, and BUG-04 in the source and explain the failure mode of each
- Write a valid schedule dict for a CPU training run from scratch

**Time estimate:** 45 minutes | **Prerequisites:** Lesson 00

---

## What This Component Does — Plain English

`base.py` is a complete, self-contained PyTorch training engine. It handles every generic training concern so that individual attack classes never have to: device setup, optimizer construction, the training loop, learning rate scheduling, periodic evaluation, checkpoint saving, and logging.

Every attack in BackdoorBox is implemented as a subclass of `Base`. The subclass creates a poisoned dataset — that is its one job. Then it calls `self.train(schedule)` and the base class runs the full training loop, feeding either the clean dataset or the poisoned dataset depending on one flag in the schedule dict.

The analogy from your toolkit: this is the SOAR playbook engine. Individual attack classes write the playbook steps (poison the data). `Base` is the automation platform that executes them. The schedule dict is the playbook configuration: here is where to log, here is the batch size, here is whether to run live or in dry-run mode. You hand the platform a configuration and it executes. You do not subclass the platform to change logging behavior. You change the dict.

**Real-world analogy:** Think about how Swimlane separates playbook definition from playbook execution. A playbook author defines conditions and actions. The Swimlane engine handles retry logic, logging, case creation, and state tracking. The author never touches the engine. `Base` is the engine. Attack subclasses are the playbooks.

---

## Where This Fits in the System

```
Researcher writes:
  BadNets(train_dataset, test_dataset, model, loss, ...)
        |
        v
  BadNets.__init__()  <-- creates poisoned dataset wrappers
        |
        v
  Base.__init__()     <-- stores datasets, model, loss, seed
        |
        v
  badnets.train(schedule)
        |
        v
  Base.train()        <-- THIS FILE: runs the full training loop
        |
        +---> DataLoader(clean dataset)    if benign_training=True
        +---> DataLoader(poisoned dataset) if benign_training=False
        |
        v
  Base._test()        <-- called every test_epoch_interval epochs
        |
        +---> accuracy on clean test set  (Benign Accuracy / BA)
        +---> accuracy on poisoned test set (Attack Success Rate / ASR)
```

If `base.py` is removed or crashes, no attack can train. The entire research pipeline stops. Every benchmark number in the BackdoorBox papers ran through this file.

---

## Key Concepts

### The Schedule Dict

The schedule dict is the single configuration interface for both training and testing. No CLI arguments, no config files, no constructor parameters for hyperparameters. You hand a Python dict to `train()` or `test()`. The method reads the dict at runtime.

Required keys for a training run:
- `device` — `'GPU'` or `'CPU'`
- `benign_training` — `True` (train clean) or `False` (train poisoned)
- `batch_size`, `num_workers` — DataLoader parameters
- `lr`, `momentum`, `weight_decay` — SGD optimizer parameters
- `gamma`, `schedule` — LR decay multiplier and epoch milestones
- `epochs` — total training epochs
- `log_iteration_interval` — log every N iterations
- `test_epoch_interval` — evaluate every N epochs
- `save_epoch_interval` — checkpoint every N epochs
- `save_dir`, `experiment_name` — output directory and prefix

Optional keys:
- `CUDA_SELECTED_DEVICES` — which GPUs to use (defaults to all visible)
- `pretrain` — path to a pretrained checkpoint to load before training
- `warmup_epoch` — number of warmup epochs for LR

The dict is `deepcopy`-ed into `self.current_schedule` at the start of every `train()` call. That copy prevents mid-run modifications to the caller's dict from affecting training.

### The benign_training Flag

This single boolean controls which dataset the training loop sees. `True` means use `self.train_dataset` — the original, unmodified data. `False` means use `self.poisoned_train_dataset` — the same data with triggers and relabeled targets injected.

This is how a researcher runs the two-phase BadNets experiment from `Attack_BadNets.py`: call `train()` twice with `benign_training=True` then `benign_training=False`. Same model class, same hyperparameters, same training engine. Only one bit changes. The resulting model weights diverge because the poisoned run has secretly learned to associate the trigger with the target class.

From a security standpoint: this flag is the difference between building a clean baseline and executing the attack. The flag is set in plain sight in the schedule dict. There is no hidden mechanism. The attack is the data, not the training loop.

### LR Step Decay

`adjust_learning_rate()` implements step decay: the learning rate starts at `schedule['lr']` and is multiplied by `schedule['gamma']` at each milestone in `schedule['schedule']`.

```python
# core/attacks/base.py -- Lines 107-109
factor = (torch.tensor(self.current_schedule['schedule']) <= epoch).sum()
lr = self.current_schedule['lr'] * (self.current_schedule['gamma'] ** factor)
```

If `schedule = [150, 180]` and `gamma = 0.1`:
- Epochs 0-149: `lr * 0.1^0 = lr * 1.0` (full rate)
- Epochs 150-179: `lr * 0.1^1` (10% of original)
- Epochs 180+: `lr * 0.1^2` (1% of original)

The `torch.tensor` comparison counts how many milestones have been passed. This is idiomatic PyTorch, not a complex formula. Think of it as: "for each milestone you have passed, apply one more decay step."

### Reproducibility Controls

`_set_seed()` seeds four independent RNG sources simultaneously:

```python
# core/attacks/base.py -- Lines 73-84
torch.manual_seed(seed)
random.seed(seed)
np.random.seed(seed)
os.environ['PYTHONHASHSEED'] = str(seed)
```

Setting `PYTHONHASHSEED` as an environment variable has a known limitation: Python reads this variable at interpreter startup. Setting it mid-process does not retroactively change hash randomization for the current process. The researchers seed it anyway because it costs nothing and may matter for subprocess behavior.

When `deterministic=True`, additional CUDA settings disable non-deterministic algorithms. The tradeoff: deterministic mode is slower because some fast GPU kernels are non-deterministic.

---

## Code Walkthrough

### Schedule Resolution -- Lines 118-130

The first thing `train()` does is resolve which schedule to use. There are two possible schedules: the one passed at construction time (`self.global_schedule`) and the one passed to this specific `train()` call.

```python
# core/attacks/base.py -- Lines 118-130
def train(self, schedule=None):
    if schedule is None and self.global_schedule is None:
        raise AttributeError("Training schedule is None, please check your schedule setting.")
    elif schedule is not None and self.global_schedule is None:
        self.current_schedule = deepcopy(schedule)
    elif schedule is None and self.global_schedule is not None:
        self.current_schedule = deepcopy(self.global_schedule)
    elif schedule is not None and self.global_schedule is not None:
        self.current_schedule = deepcopy(schedule)

    work_dir = osp.join(self.current_schedule['save_dir'], self.current_schedule['experiment_name'] + '_' + time.strftime("%Y-%m-%d_%H:%M:%S", time.localtime()))
    os.makedirs(work_dir, exist_ok=True)
    log = Log(osp.join(work_dir, 'log.txt'))
```

**Line-by-line breakdown:**

| Lines | What it does | Why it was designed this way |
|-------|-------------|------------------------------|
| 119-126 | Four-way if/elif resolves which schedule wins | Per-call schedule overrides constructor schedule, enabling different hyperparameters per training run without reconstructing the object |
| 123-124 | `deepcopy(schedule)` stores the local copy | Caller's dict is never mutated; two consecutive `train()` calls with the same schedule dict cannot interfere |
| 128 | Build work_dir path with timestamp | Each run gets a unique output directory; timestamps prevent overwriting previous results |
| 129 | `os.makedirs(work_dir, exist_ok=True)` | Creates the full path even if intermediate directories do not exist |
| 130 | `Log(osp.join(work_dir, 'log.txt'))` | Opens a log file in the run's output directory |

> **BUG-04 -- Line 128:** The timestamp format `%H:%M:%S` produces colons (e.g., `2026-05-25_14:32:01`). On Windows, colons are illegal in directory names. `os.makedirs()` will raise `FileNotFoundError` or `OSError` on Windows when the timestamp contains colons. The fix is `%H-%M-%S`. This bug also exists at line 339 in `test()`.

**Design pattern used:** Template Method. `train()` is the fixed template. Subclasses do not override `train()`. They set `self.poisoned_train_dataset` in their own `__init__`. The base class reads it at line 197.

---

### Device Setup -- Lines 145-183

```python
# core/attacks/base.py -- Lines 145-183
if 'device' in self.current_schedule and self.current_schedule['device'] == 'GPU':
    # ... resolve CUDA_VISIBLE_DEVICES from environment ...
    # ... resolve CUDA_SELECTED_DEVICES from schedule dict ...
    CUDA_VISIBLE_DEVICES_SET = set(CUDA_VISIBLE_DEVICES_LIST)
    CUDA_SELECTED_DEVICES_SET = set(CUDA_SELECTED_DEVICES_LIST)
    if not (CUDA_SELECTED_DEVICES_SET <= CUDA_VISIBLE_DEVICES_SET):
        raise ValueError(f'CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!')

    GPU_num = len(CUDA_SELECTED_DEVICES_SET)
    device_ids = [CUDA_VISIBLE_DEVICES_LIST.index(CUDA_SELECTED_DEVICE) for CUDA_SELECTED_DEVICE in CUDA_SELECTED_DEVICES_LIST]
    device = torch.device(f'cuda:{device_ids[0]}')
    self.model = self.model.to(device)

    if GPU_num > 1:
        self.model = nn.DataParallel(self.model, device_ids=device_ids, output_device=device_ids[0])
else:
    device = torch.device("cpu")
```

The device setup distinguishes between two concepts:
- `CUDA_VISIBLE_DEVICES` — GPUs visible to this OS process (controlled by environment variable)
- `CUDA_SELECTED_DEVICES` — GPUs this experiment will use (subset of visible, from the schedule dict)

This two-level design lets a shared server with 4 GPUs (`CUDA_VISIBLE_DEVICES=0,1,2,3`) have each experiment use only a subset (`CUDA_SELECTED_DEVICES=2,3`) without modifying the environment variable.

`nn.DataParallel` is PyTorch's basic multi-GPU strategy: the model is replicated to each GPU, each GPU processes a fraction of the batch, and gradients are averaged before the weight update.

> **BUG-02 -- Line 172:** The error message reads `"CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!"` — both variables in the message are the same. The intended message is `"CUDA_SELECTED_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!"`. In an operational context, this is a mislabeled alert — the alert fires correctly, but the description is wrong.

---

### The Training Loop -- Lines 185-269

```python
# core/attacks/base.py -- Lines 185-206
if self.current_schedule['benign_training'] is True:
    train_loader = DataLoader(
        self.train_dataset,
        batch_size=self.current_schedule['batch_size'],
        shuffle=True,
        num_workers=self.current_schedule['num_workers'],
        drop_last=False,
        pin_memory=True,
        worker_init_fn=self._seed_worker
    )
elif self.current_schedule['benign_training'] is False:
    train_loader = DataLoader(
        self.poisoned_train_dataset,
        # ... same parameters ...
    )
else:
    raise AttributeError("self.current_schedule['benign_training'] should be True or False.")
```

Notice: `is True` and `is False`, not `== True` and `== False`. This is identity comparison, not equality comparison. `1 is True` evaluates to `False` in Python even though `1 == True` evaluates to `True`. A schedule dict with `benign_training=1` would fall into the `else` branch and raise `AttributeError`. This is a deliberate strictness choice.

The `worker_init_fn=self._seed_worker` call ensures that each DataLoader worker process seeds its own NumPy and Python RNG from the parent PyTorch seed. Without this, multi-worker data loading produces different random augmentations each run even when seeds are set.

```python
# core/attacks/base.py -- Lines 218-236 (condensed)
for i in range(self.current_schedule['epochs']):
    for batch_id, batch in enumerate(train_loader):
        self.adjust_learning_rate(optimizer, i, batch_id, ...)
        batch_img = batch[0].to(device)
        batch_label = batch[1].to(device)
        optimizer.zero_grad()
        predict_digits = self.model(batch_img)
        loss = self.loss(predict_digits, batch_label)
        loss.backward()
        optimizer.step()

        iteration += 1

        if iteration % self.current_schedule['log_iteration_interval'] == 0:
            msg = ... + f"iteration:{batch_id + 1}/{len(self.poisoned_train_dataset)//self.current_schedule['batch_size']}" + ...
            log(msg)
```

The inner loop is standard PyTorch: move batch to device, zero gradients, forward pass, compute loss, backward pass, optimizer step. `optimizer.zero_grad()` before the forward pass prevents gradient accumulation across batches.

> **BUG-03 -- Line 234:** The log message references `self.poisoned_train_dataset` (in `len(self.poisoned_train_dataset)//batch_size`) even during `benign_training=True` runs. If someone creates a `Base`-based subclass that omits poisoned data, this line raises `AttributeError` at the first log interval — potentially hours into a training run. A safer approach: `len(train_loader.dataset)`.

---

### Evaluation During Training -- Lines 238-260

Every `test_epoch_interval` epochs, the training loop pauses to evaluate on two datasets:
1. The clean test set: measures Benign Accuracy (BA) — the model should stay high
2. The poisoned test set: measures Attack Success Rate (ASR) — should rise during poisoned training

Both evaluations happen regardless of whether `benign_training` is `True` or `False`. During benign training the ASR will be low (the model has not learned the trigger). During poisoned training the ASR climbs. Watching these two numbers diverge is how researchers confirm the attack is working.

The `_test()` method wraps `torch.no_grad()` around all evaluation forward passes — essential for performance (no gradient graph is built) and correctness (batch normalization behaves differently in inference mode versus training mode).

---

### Checkpoint Saving -- Lines 264-269

```python
# core/attacks/base.py -- Lines 264-269
if (i + 1) % self.current_schedule['save_epoch_interval'] == 0:
    ckpt_model_filename = "ckpt_epoch_" + str(i+1) + ".pth"
    ckpt_model_path = os.path.join(work_dir, ckpt_model_filename)
    self.model.eval()
    torch.save(self.model.state_dict(), ckpt_model_path)
    self.model.train()
```

Checkpoints save every `save_epoch_interval` epochs as `.pth` files. The model is switched to inference mode before saving to ensure batch normalization statistics are in their "inference" state, then switched back to training mode. This matters for models with batch normalization: inference mode uses running mean/variance, training mode uses batch statistics.

---

## Hands-On Exercises

Before starting: confirm your Python environment has PyTorch installed, or simply read along — the code reading exercises do not require running anything.

### Exercise 1: Read a Schedule Dict

The schedule dict from `Attack_BadNets.py` (lines 96-119):

```python
schedule = {
    'device': 'GPU',
    'CUDA_VISIBLE_DEVICES': '0',
    'GPU_num': 1,
    'benign_training': True,
    'batch_size': 128,
    'num_workers': 16,
    'lr': 0.1,
    'momentum': 0.9,
    'weight_decay': 5e-4,
    'gamma': 0.1,
    'schedule': [150, 180],
    'epochs': 200,
    'log_iteration_interval': 100,
    'test_epoch_interval': 10,
    'save_epoch_interval': 10,
    'save_dir': 'experiments',
    'experiment_name': 'train_benign_DatasetFolder-CIFAR10'
}
```

Answer these questions by tracing through `base.py`:

1. At epoch 160, what is the learning rate?
2. How often does the model save a checkpoint?
3. If you ran this on Windows with the current code, what would fail and why?
4. Is this a benign run or a poisoned run?

**Expected answers:**
1. `0.1 * 0.1^1 = 0.01` (milestone 150 has passed, milestone 180 has not)
2. Every 10 epochs (200 / 10 = 20 checkpoints total)
3. `os.makedirs()` would fail because `%H:%M:%S` in the timestamp produces colons in the directory name -- BUG-04
4. Benign: `'benign_training': True`

**You succeeded if:** You can answer all four without running the code, by tracing the base.py logic directly.

---

### Exercise 2: Write a CPU Schedule Dict

Write a schedule dict for a short CPU training run (5 epochs, batch size 32, no GPU). You are not running it — just confirm it has every required key.

```python
schedule = {
    'device': 'CPU',
    'benign_training': False,
    'batch_size': 32,
    'num_workers': 0,
    'lr': 0.01,
    'momentum': 0.9,
    'weight_decay': 1e-4,
    'gamma': 0.1,
    'schedule': [3],
    'epochs': 5,
    'log_iteration_interval': 10,
    'test_epoch_interval': 2,
    'save_epoch_interval': 5,
    'save_dir': 'experiments',
    'experiment_name': 'cpu_test_run'
}
```

**You succeeded if:** You can name every key and explain why it is required by tracing through the `train()` method.

---

### Exercise 3: Locate BUG-02 in the Source

Open `BackdoorBox/core/attacks/base.py` and find line 172. Read the error message.

```powershell
# PowerShell
Select-String -Path "BackdoorBox\core\attacks\base.py" -Pattern "should be a subset"
```

```bash
# Bash
grep -n "should be a subset" BackdoorBox/core/attacks/base.py
```

**Expected output:**
```
172:                raise ValueError(f'CUDA_VISIBLE_DEVICES should be a subset of CUDA_VISIBLE_DEVICES!')
```

**You succeeded if:** You see both instances of `CUDA_VISIBLE_DEVICES` in the same error string and can explain why the message is wrong.

---

### Exercise 4: Locate BUG-03 in Context

The bug at line 234 references `self.poisoned_train_dataset` inside a log message that runs during both benign and poisoned training.

```powershell
# PowerShell
Select-String -Path "BackdoorBox\core\attacks\base.py" -Pattern "poisoned_train_dataset" -Context 2,2
```

```bash
# Bash
grep -n -B2 -A2 "poisoned_train_dataset" BackdoorBox/core/attacks/base.py
```

**You succeeded if:** You can explain the failure condition: when does this line raise `AttributeError`? (Answer: when `Base` is used without a subclass that creates `self.poisoned_train_dataset` — for example, in a benign-only training scenario where the attack-specific `__init__` is not called.)

---

## Interview Preparation

### Q: Why do all 15 attacks in BackdoorBox share a single base class instead of each implementing its own training loop?

**A:** The training loop is identical across attacks. What varies between attacks is only how the training data is poisoned — the trigger shape, position, and blending method. Sharing the base class means bug fixes and improvements to the training loop propagate to all attacks automatically. It also enforces a consistent interface: any BackdoorBox attack can be evaluated with the same schedule dict, logging format, and checkpoint structure. The tradeoff is that `Base` carries more complexity than any single attack needs, and some assumptions (like the existence of `self.poisoned_train_dataset`) must be satisfied by every subclass even when not all attacks need exactly that pattern.

*Why this answer works:* It demonstrates understanding of the inheritance design decision, not just that inheritance exists.

---

### Q: The schedule dict has both `benign_training=True` and `benign_training=False` modes. Why would you run `benign_training=True` through the same attack object that has a poisoned dataset attached to it?

**A:** Running benign training first establishes a clean baseline — a model trained on the same architecture, same hyperparameters, same data, without any poisoning. This baseline serves two purposes. First, it lets you measure the accuracy cost of the attack: if clean accuracy drops from 93% to 87% after poisoning, the attack may be detectable by monitoring model performance. Second, it is the control condition for published research — without the benign baseline, you cannot claim the attack preserved normal model behavior. In `Attack_BadNets.py`, both `train()` calls use the same `badnets` object because the poisoned dataset is already attached; the `benign_training` flag just controls which dataset the DataLoader reads.

*Why this answer works:* It connects the code design to the research methodology and security measurement.

---

### Q: There are three bugs in `base.py`. Which is the most operationally dangerous and why?

**A:** BUG-03 (line 234) is the most dangerous because it is latent. BUG-02 is a wrong error message — the error still fires, it just gives you a confusing description. BUG-04 crashes on Windows but fails immediately and loudly. BUG-03 is silent: it does not trigger during normal `BadNets` usage (because `BadNets.__init__` always creates the poisoned dataset), but it would trigger if a researcher creates a new attack subclass and tries to use benign-only training without creating a poisoned dataset. The bug would hide until the first logging interval in the training loop, producing an `AttributeError` deep in a training run that may have taken hours to reach. This is the same failure pattern as a time-delayed alert rule that only errors on rare edge cases — the error is real but surfaces at the worst possible time.

*Why this answer works:* It ranks bugs by operational impact, uses the SIEM/alert analogy naturally, and demonstrates understanding of when the bug fires versus when it is hidden.

---

## Key Takeaways

- `Base` is the shared training engine for all 15 attacks. Subclasses do one thing: create a poisoned dataset. The base class runs everything else.
- The schedule dict is the only configuration interface. Every hyperparameter, device setting, and logging parameter flows through this dict.
- `benign_training=True` and `benign_training=False` are the two phases of every backdoor attack experiment. Both phases use the same object, same base class, same training loop.
- Three bugs live in this file: a wrong error message (BUG-02), a latent AttributeError on benign-only training (BUG-03), and a Windows-incompatible timestamp format (BUG-04).
- `_test()` always evaluates against both the clean test set (BA) and the poisoned test set (ASR). Watching ASR rise while BA holds steady is the signature of a successful attack.

---

## Quick Reference Card

| Item | Value |
|------|-------|
| File | `core/attacks/base.py` |
| Entry point | `Base.train(schedule)` |
| Input | Schedule dict + datasets set at construction time |
| Output | `.pth` checkpoint files + `log.txt` in `save_dir/experiment_name_TIMESTAMP/` |
| Key config | `benign_training`, `device`, `epochs`, `schedule`, `gamma`, `lr` |
| Error behavior | Raises `AttributeError` if no schedule provided; raises `ValueError` if GPU not found |
| Dependencies | `torch`, `numpy`, `torchvision`, `core.utils.Log` |
| Bugs in this file | BUG-02 (line 172), BUG-03 (line 234), BUG-04 (lines 128, 339) |
| Test file | `tests/test_BadNets.py` shows full usage |

---

## Implemented vs. Recommended

### What This Project Implements
- SGD optimizer with step-decay LR scheduling (`adjust_learning_rate()`, lines 106-116)
- Multi-GPU via `nn.DataParallel` (line 180) — basic data parallelism
- Reproducibility controls across all four RNG sources (`_set_seed()`, lines 72-93)
- Per-run output directories with timestamps (lines 128-130)
- Periodic evaluation against both clean and poisoned test sets (lines 238-260)

### General Best Practices — Recommended but Not Implemented Here
- **AdamW or Adam optimizer as alternative to SGD** — many modern training pipelines default to adaptive optimizers. SGD with momentum works well for ResNets but requires careful LR tuning. `Recommended (not implemented here)`
- **Gradient clipping** (`torch.nn.utils.clip_grad_norm_`) — prevents exploding gradients in deep networks. Not present in `base.py`. `Recommended (not implemented here)`
- **Early stopping** — stopping training when validation loss stops improving. `Base` always trains for the full epoch count. `Recommended (not implemented here)`
- **`DistributedDataParallel` (DDP) instead of `DataParallel`** — DDP is PyTorch's recommended multi-GPU strategy for performance. The `ShrinkPad.py` file has a `TODO: DDP training` comment. `Recommended (not implemented here)`
- **Type annotations and runtime schema validation for the schedule dict** — currently any missing key causes a `KeyError` deep in training with no helpful error message. A `pydantic` or `dataclass`-based schedule class would validate at construction time. `Recommended (not implemented here)`

---

## Ready for Lesson 02?

Next up: **Data Poisoning 101: BadNets** — you have now seen the training engine. In Lesson 02, you will open `BadNets.py` and see exactly how the trigger gets injected into training samples: the `AddTrigger` class, the poisoned dataset wrappers, and the `__getitem__` override that makes training data poisoning invisible until inference time.

**Optional deeper dive:** The PyTorch documentation on `DataParallel` vs `DistributedDataParallel` explains why the TODO comment in `ShrinkPad.py` exists. DDP reduces inter-GPU communication overhead significantly on large models.

**Modification challenge:** Add a `warmup_epoch` key to the CPU schedule dict from Exercise 2 (set it to `2`) and trace through `adjust_learning_rate()` to determine what learning rate `train()` would use for the very first batch of epoch 0. The formula is at lines 112-113.

*The training loop is the same whether the data is clean or poisoned. The attack lives in the data, not the engine.*
