## DataDojo: The Autonomous Data Cleaning Benchmark

DataDojo is a containerized reinforcement learning environment designed to evaluate the reasoning and data-wrangling capabilities of AI agents. It provides a standardized "gym" where LLMs interact with corrupted datasets to reach a clean "reference" state through autonomous decision-making. :contentReference[oaicite:0]{index=0}

---

## The Architecture

The system operates on a dual-component architecture, or simply **"The Twins,"** ensuring complete separation between data generation and corruption logic. :contentReference[oaicite:1]{index=1}

### Genesis Engine

The source of truth. It procedurally generates a randomized "master dataset" from a library of domain skeletons. The datasets have varying numbers of columns, row counts, and data types on every episode — producing a perfectly clean reference state that the agent must restore. :contentReference[oaicite:2]{index=2}

### Ruiner Engine

Systematically injects "dirt" into the master dataset by introducing:

- Missing values
- Duplicate rows
- Regex-defying string corruptions
- Inconsistent categorical casing

:contentReference[oaicite:3]{index=3}

The agent's success is measured by its ability to reverse the Ruiner's chaos and restore the dataset to the Genesis standard. :contentReference[oaicite:4]{index=4}

---

## Task Levels

DataDojo evaluates agents across three increasing levels of corruption intensity. :contentReference[oaicite:5]{index=5}

### 1. Easy

- Duplicate removal
- Dropping a fully empty column
- Filling missing values (NaNs)

:contentReference[oaicite:6]{index=6}

### 2. Medium

Includes all Easy challenges, plus:

- Cleaning numerical columns corrupted with currency-style string formatting
- Converting `dtype: object → float/int`
- Using the sequence:
  - `STRIP_CHAR`
  - `TYPE_CAST`

:contentReference[oaicite:7]{index=7}

### 3. Hard

Includes all Medium challenges, plus:

- Detecting inconsistent string casing
- Standardizing categorical column values

:contentReference[oaicite:8]{index=8}

---

## Environment Logic

### Actions & Observations

#### Observations

At each step, the agent receives:

- Current dataset state
- Column schema
- NaN counts per column
- Sample rows
- EDA tool-call results
- Detailed reward breakdown from the previous action

This provides sufficient context for reasoning about the next step. :contentReference[oaicite:9]{index=9}

#### Actions

Agents interact through DataFrame-manipulation tool calls. Most actions require specifying the exact column to operate on.

Available tools include:

- `DROP_DUPLICATES`
- `DROP_COLUMN`
- `FILL_NA`
- `STRIP_CHAR`
- `TYPE_CAST`
- `LOWERCASE`
- `GET_VALUE_COUNTS`
- `MAP_VALUES`

:contentReference[oaicite:10]{index=10}

---

## Grading & Subtle Logic

The reward per step is a tanh-normalized sum of rewards and penalties, producing a score in `[-1, 1]`, which is then remapped to `[0, 1]`. :contentReference[oaicite:11]{index=11}

### Reward Shaping

#### Error Reduction Reward

The primary learning signal. Reward is proportional to how many errors the action eliminates relative to the episode's starting error count.

Tracked errors include:

- NaNs
- Duplicates
- Value mismatches against the master dataset

:contentReference[oaicite:12]{index=12}

### The "One Free Drop"

- The first `DROP_COLUMN` call has no penalty.
- Every additional column drop incurs an action penalty.
- Dropping a non-empty column incurs a severe penalty.

This prevents reward hacking through indiscriminate column deletion. :contentReference[oaicite:13]{index=13}

### DROP_DUPLICATES Spam Penalty

Using `DROP_DUPLICATES` more than once per episode incurs a penalty to discourage repetitive fallback behavior. :contentReference[oaicite:14]{index=14}

### Invalid Column Penalty

Referencing a non-existent column is penalized, forcing agents to ground actions in the observed schema rather than hallucinating column names. :contentReference[oaicite:15]{index=15}

### Datatype Mismatch Penalty

If a column remains in the wrong datatype relative to the master dataset after an operation, a penalty is applied. This encourages the correct sequence:

1. `STRIP_CHAR`
2. `TYPE_CAST`

:contentReference[oaicite:16]{index=16}

### Action Dependencies

Agents must learn valid operation orderings.

Example:

- `TYPE_CAST` to float fails if symbols like `$` or `..` have not first been removed using `STRIP_CHAR`.

The environment reports failed actions, and the agent must recover autonomously. :contentReference[oaicite:17]{index=17}

---

## A Note on Difficulty

DataDojo is intentionally difficult. Every episode generates a randomized dataset with:

- Different schemas
- Different column names
- Different row counts
- Different datatypes

This prevents memorization and forces genuine reasoning over observed data. :contentReference[oaicite:18]{index=18}

### Baseline Results

During development, the baseline agent used was:

- `Qwen2.5-72B-Instruct`

A 72B-parameter open-source instruction-tuned model. :contentReference[oaicite:19]{index=19}

Observed behavior:

#### Easy Level

The model can usually:

- Identify and drop the empty column

But often struggles to complete:

- NaN filling
- Duplicate removal

within the 10-step action budget. :contentReference[oaicite:20]{index=20}

#### Medium & Hard Levels

The model struggles with multi-step dependency chains such as:

1. Identifying the corrupted column
2. Applying `STRIP_CHAR`
3. Performing `TYPE_CAST`
4. Tracking already-completed operations

This limitation is intentional. A benchmark that frontier models solve trivially offers little training value.

DataDojo is designed to sit at the frontier of current LLM tool-use and multi-step reasoning capabilities, leaving meaningful headroom for RL-trained or fine-tuned agents to demonstrate measurable improvement over zero-shot baselines. 
