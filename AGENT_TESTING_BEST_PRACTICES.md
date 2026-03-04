# Agent Testing Best Practices

## Core Challenge

Agents are non-deterministic and multi-step. You cannot assert on output strings.
Testing must happen at multiple layers, each targeting a different failure mode.

---

## The Three Testing Layers

```
Layer 1 — Tool Unit Tests       deterministic, no model, fast
Layer 2 — Trajectory Tests      behavioral, model involved, medium
Layer 3 — Evals                 quality, LLM judge or human, slow
```

---

## Layer 1: Tool Unit Tests

Tools are pure functions — test them in complete isolation with no model involved.
This is the cheapest and most reliable layer.

### What to Test

- Correct output schema shape (ACL is translating correctly)
- Domain invariants enforced (aggregate boundaries respected)
- Error cases return structured `ToolError`, not exceptions
- Edge cases: empty results, missing data, boundary values

```python
def test_check_availability_in_stock():
    result = check_availability("product-123")
    assert isinstance(result, Availability)
    assert result.in_stock == True

def test_check_availability_translates_internal_model():
    # ACL correctly hides internal inventory model
    result = check_availability("product-456")   # fixture: count=0
    assert result.in_stock == False
    assert result.estimated_days == 5

def test_check_availability_missing_product():
    result = check_availability("product-999")
    assert isinstance(result, ToolError)
    assert result.suggestion is not None          # always guide recovery
```

### Rules

- One test per domain invariant, not per code path
- Fixtures should use domain language, not internal identifiers
- Test the ACL translation explicitly — it is the most likely place for context boundary bugs

---

## Layer 2: Trajectory Tests

Test what the agent **did**, not what it **said**. Capture the sequence of tool calls,
their order, and the schema structure of each step.

### What to Test

- Tool call ordering (data must be fetched before it can be analyzed)
- Required tools are always called for a given request type
- Loop always terminates (`FinalResponse` is always reached)
- Bounded context is respected (agent never calls out-of-scope tools)
- `reason` fields are non-empty (model is justifying its actions)

```python
def test_scouting_report_trajectory():
    trace = agent.run_with_trace("Scout GSW pick-and-roll defense")
    calls = [step.function.__class__.__name__ for step in trace]

    # Ordering: data before analysis
    assert calls.index("CallGameDataAgent") < calls.index("CallScoutingAgent")

    # Required tools present
    assert "CallPlayRecognitionAgent" in calls
    assert "CallNarrativeAgent" in calls

    # Always terminates
    assert calls[-1] == "OrchestratorFinalResponse"

    # Bounded context: orchestrator never calls raw DB tools
    assert "CallRawDatabaseAgent" not in calls

def test_reason_fields_populated():
    trace = agent.run_with_trace("Get player stats for LeBron")
    for step in trace:
        if hasattr(step.function, "reason"):
            assert len(step.function.reason) > 10   # non-trivial justification

def test_sgr_state_accumulates():
    trace = agent.run_with_trace("Analyze Lakers offense last 5 games")
    states = [step.current_state for step in trace]
    # Each state should be longer than the previous — information accumulates
    for i in range(1, len(states)):
        assert len(states[i]) >= len(states[i - 1])
```

### Capturing Traces

Structure your agent runner to always emit a trace alongside the final output:

```python
class AgentTrace(BaseModel):
    steps: list[NextStep]
    final_response: FinalResponse
    total_steps: int
    tool_calls: list[str]          # flat list of tool names called

def run_with_trace(prompt: str) -> AgentTrace:
    ...
```

---

## Layer 3: Evals (Golden Dataset)

Curated input/output pairs evaluated against a rubric. Use an LLM judge or human
review. Designed to catch quality regressions, not logic bugs.

### Rubric Structure

Each eval case defines:
- **Input**: the coach/user request
- **Must include**: things that must appear in the output
- **Must not include**: constraint violations to check for
- **Schema checks**: structural requirements on the final response

```python
golden_cases = [
    {
        "input": "Scout GSW pick-and-roll defense",
        "must_include": [
            "mentions coverage scheme (drop / hedge / switch)",
            "includes PPP allowed on PnR possessions",
            "identifies which defenders switch vs. drop",
            "sample size or game count referenced",
        ],
        "must_not_include": [
            "recommends specific in-game play calls",   # constraint violation
            "presents regular season data as playoff data",
        ],
        "schema_checks": {
            "key_findings": lambda f: 3 <= len(f) <= 5,
            "data_gaps": lambda g: isinstance(g, list),
            "confidence": lambda c: c in ["high", "medium", "low"],
        }
    }
]
```

### Scoring

```python
def score_eval_case(case, agent_output, judge_model) -> EvalScore:
    rubric_scores = judge_model.score(
        output=agent_output.report,
        must_include=case["must_include"],
        must_not_include=case["must_not_include"]
    )
    schema_scores = {
        k: int(check(getattr(agent_output, k)))
        for k, check in case["schema_checks"].items()
    }
    return EvalScore(
        rubric=rubric_scores,     # 0 or 1 per item
        schema=schema_scores,
        total=mean([*rubric_scores.values(), *schema_scores.values()])
    )
```

### Regression Threshold

```python
BASELINE_SCORES = {
    "scouting_report": 0.85,
    "game_recap": 0.90,
    "player_profile": 0.88,
}

def test_evals_do_not_regress():
    for case in golden_cases:
        output = agent.run(case["input"])
        score = score_eval_case(case, output, judge)
        report_type = output.report_type
        assert score.total >= BASELINE_SCORES[report_type], (
            f"Regression in {report_type}: {score.total:.2f} < {BASELINE_SCORES[report_type]}"
        )
```

---

## Self-Adjustment Testing

Test that the agent recovers correctly when things go wrong.

### Tool Error Recovery

```python
def test_agent_recovers_from_missing_game():
    # Simulate game ID not found — agent should pivot to team+date query
    with mock_tool_error("CallGameDataAgent", game_id="G999", code="GAME_NOT_FOUND"):
        trace = agent.run_with_trace("Analyze game G999")

    calls = [step.function for step in trace]
    retry = next(c for c in calls if isinstance(c, CallGameDataAgent) and c.teams)
    assert retry is not None                         # agent retried with different params
    assert trace[-1].function.__class__.__name__ == "OrchestratorFinalResponse"
```

### Data Gap Honesty

```python
def test_agent_flags_small_sample():
    # Only 2 games available — below confidence threshold
    with mock_game_sample(size=2):
        output = agent.run("Scout Warriors defense")

    assert output.confidence in ["low", "medium"]
    assert len(output.data_gaps) > 0
    assert any("sample" in gap.lower() for gap in output.data_gaps)
```

### Reflection Step Triggered

```python
def test_agent_reflects_on_contradictory_data():
    # Fixture: ORTG says team is poor, but win rate says otherwise
    with mock_contradictory_stats():
        trace = agent.run_with_trace("Analyze Lakers offensive efficiency")

    calls = [step.function.__class__.__name__ for step in trace]
    assert "Reflect" in calls                        # model flagged the contradiction
```

---

## What Not to Test

| Avoid | Reason |
|---|---|
| Asserting on exact output text | Non-deterministic — will flake |
| Testing model "reasoning" quality in unit tests | That belongs in evals |
| Mocking the LLM in trajectory tests | Defeats the purpose — use a real model |
| Hardcoding expected tool call counts | Fragile — model may find valid shorter paths |
| Testing internal model state directly | Violates bounded context of the test |

---

## Test Pyramid Summary

```
         /\
        /  \     Evals (golden dataset, LLM judge)
       /    \    — catches quality regressions
      /──────\
     /        \  Trajectory Tests (tool call sequences, schema shape)
    /          \ — catches behavioral regressions
   /────────────\
  /              \ Tool Unit Tests (deterministic, no model)
 /                \ — catches ACL bugs, domain invariant violations
/──────────────────\
```

**Run order in CI:**
1. Tool unit tests — fast gate, fail early
2. Trajectory tests — medium gate, catch behavioral breaks
3. Evals — slow gate, run on PR merge or nightly, not every commit

---

## Quick Reference Checklist

### Tool Unit Tests
- [ ] Each tool tested with valid, invalid, and edge case inputs
- [ ] ACL translation explicitly tested (internal model → domain model)
- [ ] Error cases return `ToolError` with a `suggestion` field
- [ ] Domain invariants verified (aggregate boundary respected)

### Trajectory Tests
- [ ] Tool call ordering verified for each request type
- [ ] Required tools always present for given request categories
- [ ] Agent always reaches `FinalResponse` (no infinite loops)
- [ ] Bounded context verified (no out-of-scope tool calls)
- [ ] `reason` fields non-trivially populated

### Evals
- [ ] Golden dataset covers all major request types
- [ ] Rubric includes both positive (must include) and negative (must not include) checks
- [ ] Schema structure checks included alongside content checks
- [ ] Baseline scores recorded and regression threshold enforced
- [ ] Evals run on a fixed model version to isolate regressions

### Self-Adjustment
- [ ] Tool error recovery tested (model pivots, not retries identically)
- [ ] Small sample / data gap flagging verified
- [ ] Agent reaches valid `FinalResponse` even when tools fail
