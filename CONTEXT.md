# SkillCoach — Session Context for Claude Code

Paste this file at the start of a new session so Claude Code has full context.

---

## What was built

A complete **OpenEnv hackathon submission** called **SkillCoach** — a Socratic
debugging tutor environment for the Meta x PyTorch OpenEnv Hackathon (Round 1).

**Location:** `c:\Users\mail2\OneDrive\Documents\Desktop\projectrandom\skillcoach-env\`

All 54 pre-submission validation checks pass (`python validate.py` → 54/54 PASS).

---

## File structure

```
skillcoach-env/
├── skillcoach_env.py   # OpenEnv env class + FastAPI HTTP app
├── tasks.py            # All scenarios, TaskState, 3 deterministic graders
├── inference.py        # Mandatory baseline inference script
├── openenv.yaml        # OpenEnv metadata / task registry
├── Dockerfile          # python:3.11-slim, port 7860
├── requirements.txt    # 7 pinned packages
├── validate.py         # 54-check pre-submission validator
├── README.md           # Full project documentation
└── CONTEXT.md          # This file
```

---

## Architecture decisions (important — do not change without reason)

### Import structure (avoids circular imports)
- `tasks.py` is self-contained. It defines `GradeResult` (a plain dataclass)
  and exports it so `skillcoach_env.py` can use it.
- `skillcoach_env.py` imports FROM `tasks.py` (never the reverse).
- `SkillCoachInfo` (Pydantic model) lives in `skillcoach_env.py`; the env
  converts `GradeResult → SkillCoachInfo` in `step()`.

### Reward design
- **Tasks 1 & 2** (1 turn each): `step()` returns the grader score directly.
  `done=True` immediately after the first step.
- **Task 3** (up to 5 turns): each step returns `per_turn_reward` (max 1.0).
  When `done=True` the returned reward is the **final episode score**:
  `clamp(mean(per_turn_rewards) + 0.3_bonus_if_student_found_answer, 0, 1)`.
  The inference script uses `rewards[-1]` as `score` in the `[END]` line.

### Student "found answer" detection (guided-debugging)
- After each agent step, `TaskState.check_and_flag_student_found_answer(turn)`
  checks whether the pre-scripted student message for that turn contains any
  `solution_keywords`. If yes it sets `task_state.student_found_answer = True`.
- `is_done()` returns `True` when `turn >= max_turns OR student_found_answer`.
- This is called inside `grade_response()` before `is_done()` is evaluated,
  so the flag is always set before the done check.

### FastAPI `/reset` optional body
- Body is `Optional[_ResetRequest]` defaulting to `None`.
- If `None`, task defaults to `"identify-error"`.
- Works with or without a JSON body.

---

## Task summary

| Task | Difficulty | Turns | Grader logic |
|------|-----------|-------|-------------|
| `identify-error` | Easy | 1 | Keyword match on error type; direct-fix penalty |
| `hint-without-answer` | Medium | 1 | diagnostic_keywords score minus fix_keywords penalty |
| `guided-debugging` | Hard | ≤5 | Per-turn: question + no-answer signal; episode: mean + bonus |

### identify-error scoring
| Condition | Score |
|-----------|-------|
| Exact error-type keyword (`type`, `syntax`, etc.) | 1.0 |
| Close match (`TypeError`, `typeerror`, etc.) | 0.7 |
| Relevant but vague (`error`, `check`, `look at`) | 0.3 |
| Contains direct fix pattern (`int(`, `should be`, etc.) | 0.1 |
| Empty / irrelevant | 0.0 |

### hint-without-answer scoring
```
penalty       = 0.5 if any fix_keyword in response else 0.0
keyword_score = len(diagnostic_keywords_found) / total_diagnostic_keywords
length_cap    = 0.4 if len(response) > 500 else 1.0
score         = max(0.0, min(length_cap, keyword_score - penalty))
# response < 20 chars → 0.0 immediately
```

### guided-debugging per-turn scoring
```
question_score  = 0.5 if response ends with ? or contains question word
no_answer_score = 0.5 if response does NOT contain solution_keywords
per_turn_reward = question_score + no_answer_score   # max 1.0
```
Episode:
```
final_score = clamp(mean(per_turn_rewards) + (0.3 if student_found_answer else 0), 0, 1)
```

---

## Scenarios

### identify-error (5 scenarios, rotated randomly on reset)
1. `TypeError` — `add(5, '3')`
2. `NameError` — `mesage` typo
3. `SyntaxError` — missing `:` in function def
4. `IndexError` (runtime) — `arr[5]` on 3-element list
5. Logic error — `n % 2 == 1` for is_even

### hint-without-answer (5 scenarios)
1. Off-by-one in `range(len(arr)+1)`
2. Missing `return` in function
3. `input()` returns str, compared to int
4. List aliasing (`sorted_numbers = numbers`)
5. Wrong indent on `return` inside loop

### guided-debugging (3 scenarios, pre-scripted student turns)
1. `factorial` — base case returns 0 instead of 1
2. `fizzbuzz` — wrong condition order (15 check never reached)
3. `binary_search` — `high = len(arr)` should be `len(arr) - 1`

---

## inference.py log format (mandatory, do not change)

```
[START] task=<task_name> env=skillcoach model=<model_name>
[STEP]  step=<n> action=<repr, 80-char truncated> reward=<0.00> done=<true|false> error=<msg|null>
[END]   success=<true|false> steps=<n> score=<0.000> rewards=<r1,r2,...>
```

- `[END]` is always printed via `try/finally` even on crash.
- `score` = last reward when `done=True` (final episode score).
- `rewards` = all per-step rewards collected.
- Uses **OpenAI client only** — never `anthropic` or raw `requests`.

---

## Environment variables (inference.py)

| Var | Default | Purpose |
|-----|---------|---------|
| `API_BASE_URL` | `https://api.openai.com/v1` | LLM endpoint |
| `MODEL_NAME` | `gpt-4o-mini` | Model ID |
| `HF_TOKEN` | `$OPENAI_API_KEY` or empty | API key |

---

## How to run

```bash
cd "c:\Users\mail2\OneDrive\Documents\Desktop\projectrandom\skillcoach-env"

# Validate everything
python validate.py

# Run baseline inference (needs real API key)
set API_BASE_URL=https://api.openai.com/v1
set MODEL_NAME=gpt-4o-mini
set HF_TOKEN=sk-...
python inference.py

# Start HTTP server
uvicorn skillcoach_env:app --host 0.0.0.0 --port 7860

# Docker
docker build -t skillcoach .
docker run -p 7860:7860 -e HF_TOKEN=sk-... skillcoach
```

---

## Key constraints (do not violate)

1. All Pydantic models use `model_config = ConfigDict(arbitrary_types_allowed=True)`.
2. All env methods (`reset`, `step`, `close`) are `async def`.
3. All graders are fully deterministic — no randomness in scoring.
4. Grader scores always clamped to `[0.0, 1.0]`.
5. `[END]` log line always printed via `try/finally`.
6. `inference.py` uses ONLY the OpenAI client for LLM calls.
7. Docker image has no runtime downloads (all deps in `pip install`).
8. `/reset` must respond within 5 seconds (no heavy loading at request time).
