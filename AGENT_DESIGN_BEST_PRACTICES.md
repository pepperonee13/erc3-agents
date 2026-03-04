# Agent Design Best Practices: SGR + Domain Driven Design

## Core Principle

The goal is to build agents that can only think domain-valid thoughts, perform domain-valid actions, and reason in domain-valid terms — eliminating semantic hallucinations by construction, not by prompt engineering.

---

## Part 1: Structured Generation with Reasoning (SGR)

### What It Is

SGR is a pattern where the agent's output at each step is a structured schema that contains both its **reasoning** and its **next action**. The model cannot act without reasoning, and cannot reason without committing to an action.

### The Core Loop Schema

```python
class NextStep(BaseModel):
    current_state: str   # what has been established so far
    reasoning: str       # what is missing and why the next action is needed
    function: (
        ToolCallA
        | ToolCallB
        | FinalResponse  # terminates the loop
    )
```

### SGR Best Practices

| Practice | Rationale |
|---|---|
| Always include `current_state` | Forces the model to maintain a running summary, catching drift early |
| Always include `reasoning` | Makes chain-of-thought explicit and inspectable at every step |
| Use a discriminated union for `function` | The model commits to exactly one action per step — no ambiguity |
| Include `FinalResponse` as a union member | Loop termination is a first-class decision, not a special case |
| Include `reason: str` on every tool call schema | Forces the model to articulate *why* it's calling a tool, not just *what* |
| Keep `current_state` factual, not interpretive | State is what was returned; `reasoning` is the interpretation |

### Why SGR Over Plain Tool Calling

- **Inspectability**: Every reasoning step is logged and auditable
- **Error traceability**: Bad decisions surface in `reasoning`, not silently in outputs
- **Loop control**: The model decides when it has enough information — no hardcoded step counts
- **Semantic grounding**: The model must justify each action in domain terms before taking it

---

## Part 2: Domain Driven Design for Agents

### The Mapping

| DDD Concept | Agent Design Equivalent |
|---|---|
| Ubiquitous Language | Schema field names, tool names, enum values |
| Bounded Context | Agent scope — what it knows and can do |
| Aggregate | Tool interface boundary — what operations are valid |
| Anti-Corruption Layer | Tool implementation — translates between context models |
| Domain Events | Conversation/message history (event sourcing) |
| Aggregate State Projection | `current_state` field in SGR loop |
| Published Language | Structured output schema shared between agents |
| Shared Kernel | Shared identifiers (IDs only) across context boundaries |

### Ubiquitous Language in Schema Design

The Pydantic schema **is** the ubiquitous language. Field names, enum values, and tool names must reflect domain terminology exactly.

```python
# Bad — technical, not domain
function: DBQuery | CacheGet | APICall

# Good — domain ubiquitous language
function: SearchProducts | AddToCart | ApplyDiscount
```

**Rule**: If a domain expert wouldn't recognize the terminology in the schema, the schema is wrong.

### Bounded Context → Agent Scope

Each agent owns exactly one bounded context. This directly determines:
- Which tools it has access to
- What concepts appear in its schema
- What it is allowed to know and do

**Rule**: If two tasks require different domain knowledge, they need different agents.

### Aggregate Boundaries → Tool Design

Tools should operate on **aggregates**, not raw data or internal records. The aggregate enforces domain invariants — the model never bypasses them.

```python
# Violates aggregate boundary
update_cart_line_item(cart_id, line_item_id, quantity)

# Correct — operates on the Cart aggregate
add_to_cart(cart_id, product_id, quantity)
```

**Rule**: One tool per aggregate operation, not one tool per API endpoint.

### Anti-Corruption Layer → Tool Implementation

The tool is the ACL between the model's reasoning and the underlying system. The model should never need to understand internal system complexity.

```python
def check_availability(product_id: str) -> Availability:
    raw = inventory_client.get(product_id)          # internal model
    return Availability(                             # domain model
        product_id=product_id,
        in_stock=raw["count"] > 0,
        estimated_days=0 if raw["count"] > 0 else 5
    )
```

**Rule**: Tools translate *what the model intends* into *what the system needs* — never the other way around.

---

## Part 3: Multi-Agent Systems

### Context Mapping Patterns

#### Orchestrator / Subagent (Customer-Supplier)

The most common pattern. An orchestrator decomposes tasks and delegates to specialists. Each subagent operates entirely within its bounded context.

```
OrchestratorAgent
  ├── calls SpecialistAgentA → returns structured output
  ├── calls SpecialistAgentB → returns structured output
  └── synthesizes findings → delegates to NarrativeAgent
```

**Rule**: The orchestrator never computes domain-specific results itself — it routes, sequences, and synthesizes only.

#### Tool-as-Agent (Open Host Service)

One agent is exposed as a tool to another. The calling agent doesn't know (or care) it's talking to another agent. The ACL is built into the tool interface.

```python
def get_inventory_status(product_ids: list[str]) -> InventoryStatus:
    return inventory_agent.run(product_ids)  # hidden behind the interface
```

#### Structured Handoff (Published Language)

Agents pass a shared structured object that each one reads, acts on, and appends to. Each agent writes only to its own section.

```python
class WorkflowState(BaseModel):
    order_id: str
    cart: CartSummary | None = None           # written by StoreAgent
    availability: InventoryResult | None = None  # written by InventoryAgent
    payment: PaymentResult | None = None      # written by PaymentAgent
```

#### Shared Kernel (Identifiers Only)

Agents share **identifiers**, not full models. Both contexts understand `product_id: str`, but their internal representation differs.

**Rule**: Never pass full domain objects across context boundaries — only pass IDs and let each agent resolve them through its own tools.

### Inter-Agent Communication Rules

1. Agents never share internal state or reasoning directly
2. Communication happens through defined interfaces only (tool signatures, output schemas)
3. Upstream context is passed explicitly via structured fields — never implicitly assumed
4. Each agent's output schema is its contract with the orchestrator

---

## Part 4: Orchestrator Design

### System Prompt Structure

A well-structured orchestrator system prompt contains:

1. **Identity and purpose** — what the orchestrator's role is and who it serves
2. **Specialist agent catalogue** — what each subagent does and when to call it
3. **Workflow rules** — sequencing, data passing, when to clarify vs. proceed
4. **Domain language** — canonical terminology to use (and anti-patterns to avoid)
5. **Constraints** — what the orchestrator must not do or decide
6. **Output format** — the required structure of the final response

### Orchestrator Schema Pattern

```python
class CallSpecialistAgent(BaseModel):
    # ... agent-specific parameters ...
    reason: str                     # why this agent is needed now
    prior_findings: dict            # structured context from upstream agents

class OrchestratorFinalResponse(BaseModel):
    scope: AnalysisScope            # what was analyzed
    key_findings: list[str]         # 3-5 actionable insights
    data_gaps: list[str]            # what couldn't be answered
    recommended_next_steps: list[str]
    report: str                     # narrative output

class OrchestratorNextStep(BaseModel):
    current_state: str
    reasoning: str
    function: (
        CallSpecialistAgentA
        | CallSpecialistAgentB
        | OrchestratorFinalResponse  # terminates the loop
    )
```

---

## Part 5: Design Process

Follow this sequence when building a new agent system:

1. **Domain modeling first** — define aggregates, entities, and operations with domain experts before writing any code
2. **Define bounded contexts** — identify which agent owns which part of the domain
3. **Schema derives from domain model** — every field name comes from the ubiquitous language
4. **Design tool interfaces as ACLs** — tools hide system complexity, expose domain semantics
5. **Design the orchestrator last** — its schema is a function of what the subagents expose
6. **Write the system prompt after the schema** — the prompt reinforces what the schema enforces

---

## Quick Reference Checklist

### Schema Design
- [ ] All field names use domain ubiquitous language
- [ ] Enums reflect domain states, not technical states
- [ ] Every tool call schema includes a `reason: str` field
- [ ] SGR loop includes `current_state` and `reasoning`
- [ ] `FinalResponse` is a union member alongside tool calls

### Agent Scope
- [ ] Agent operates within a single bounded context
- [ ] Agent's tool set is limited to operations valid in that context
- [ ] Agent does not access another context's internal data directly

### Tool Design
- [ ] Tools operate on aggregates, not raw records
- [ ] Tool implementation contains the ACL translation logic
- [ ] Tools do not expose internal system complexity to the model

### Multi-Agent
- [ ] Agents communicate through structured output schemas only
- [ ] Shared data across contexts is limited to identifiers
- [ ] Prior findings are passed explicitly as structured fields
- [ ] Orchestrator does not perform domain-specific computation itself
