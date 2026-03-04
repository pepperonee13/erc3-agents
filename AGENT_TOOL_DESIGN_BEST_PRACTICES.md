# Agent Tool Design Best Practices

## Core Principle

Tools are the **Anti-Corruption Layer** between the model's domain reasoning and the
underlying system. Their job is to translate — not to mirror the API, and not to
encode business processes. A pure thin wrapper is almost always wrong.

---

## The Granularity Spectrum

```
Too Thin                    Sweet Spot                   Too Thick
─────────────────────────────────────────────────────────────────
mirrors the API          one domain operation          business process

model must understand    model reasons in              model has no agency
API internals            domain terms only             hard to test/reuse
```

### Too Thin — Leaks System Complexity

```python
# Model must know: SKU format, warehouse codes, raw JSON shape
def get_inventory_sku(sku: str, warehouse: str) -> dict:
    return requests.get(f"/api/v2/inventory/sku/{sku}?warehouse={warehouse}").json()
```

The model is now reasoning about API internals, not domain concepts. This breaks
the bounded context boundary.

### Too Thick — Removes Model Agency

```python
# Model has no say in any step — just triggering a hardcoded workflow
def fulfill_order_if_available(order_id: str) -> FulfillmentResult:
    # checks inventory, reserves stock, charges payment, schedules shipping
    ...
```

The model cannot adapt, retry individual steps, or handle partial failures.

### Sweet Spot — One Domain Operation

```python
# Model reasons about product_id and availability — nothing else
def check_availability(product_id: str, quantity: int = 1) -> Availability:
    sku = catalog.to_sku(product_id)                      # ACL: ID translation
    raw = inventory_api.get_stock(sku, warehouses="all")  # ACL: hide API details
    return Availability(                                   # ACL: domain output
        product_id=product_id,
        in_stock=raw["total_count"] >= quantity,
        estimated_days=0 if raw["total_count"] >= quantity else raw["restock_days"]
    )
```

---

## What Belongs Inside a Tool

| Belongs Inside the Tool | Belongs in the Model |
|---|---|
| ACL translation (IDs, formats, schemas) | Deciding which tool to call |
| Aggregating multiple API calls for one domain operation | Sequencing multiple tools |
| Structured error responses with recovery guidance | Deciding how to recover |
| Retry logic and timeout handling | Business decisions |
| Caching and deduplication | Interpreting results |
| Input normalization (lowercase, trim, type coercion) | Choosing what to do next |

**Rule**: If the logic is about *how to talk to the system*, it belongs in the tool.
If the logic is about *what to do next*, it belongs in the model.

---

## REST vs Local: The Real Question

The underlying transport (REST, gRPC, local function, SDK call) does not matter.
What matters is whether the **interface mirrors the API or the domain**.

```
REST endpoint:    POST /api/v2/cart/{cart_id}/items
Tool should be:   add_to_cart(cart_id, product_id, quantity) -> CartSummary

REST endpoint:    GET /api/v2/inventory/sku/{sku}?warehouse=all&format=json
Tool should be:   check_availability(product_id, quantity) -> Availability
```

The tool hides URL structure, query parameters, authentication, pagination, and
response parsing. None of that is the model's concern.

---

## Structured Errors Are Non-Negotiable

A thin wrapper throws exceptions or returns raw HTTP errors. The model cannot reason
about those. Every tool must return a structured error type with a recovery suggestion.

```python
class ToolError(BaseModel):
    code: str        # domain error code, not HTTP status
    message: str     # what went wrong in domain terms
    suggestion: str  # how the model should recover next

def check_availability(product_id: str) -> Availability | ToolError:
    if not catalog.exists(product_id):
        return ToolError(
            code="PRODUCT_NOT_FOUND",
            message=f"No product with id {product_id}",
            suggestion="Search for the product by name first using search_products()"
        )
```

The `suggestion` field directly informs the model's next `reasoning` step in the
SGR loop. Without it, the model must guess how to recover.

---

## Tools Are Stateless

Tools read or write domain state — they never hold it. State lives in the
conversation's `current_state` field. Tools with internal session state create
hidden dependencies the model cannot reason about.

```python
# Bad — tool holds state the model cannot see
class CartTool:
    def __init__(self):
        self.current_cart_id = None

# Good — all state passed explicitly
def add_to_cart(cart_id: str, product_id: str, quantity: int) -> CartSummary:
    ...
```

---

## Tool Signature = Domain Language

Every tool signature should be immediately readable by a domain expert.
If they would not recognize the terminology, the abstraction level is wrong.

```python
# Bad — technical, not domain
def query_db(table: str, filters: dict) -> list[dict]: ...
def post_to_api(endpoint: str, payload: dict) -> dict: ...

# Good — domain ubiquitous language
def search_products(query: str, filters: ProductFilters) -> list[Product]: ...
def apply_discount(cart_id: str, promo_code: str) -> CartSummary: ...
```

---

## Return Types = Domain Models

Tool return types should be domain models, not API response shapes.
The ACL translation happens inside the tool — the model never sees raw API output.

```python
# Bad — model receives raw API response
def get_player_stats(player_id: str) -> dict:
    return nba_api.get("/stats/player", params={"id": player_id}).json()

# Good — model receives domain model
def get_player_stats(player_id: str, season: str) -> PlayerStats | ToolError:
    raw = nba_api.get("/stats/player", params={"id": player_id, "season": season})
    return PlayerStats(
        player_id=player_id,
        points_per_game=raw["pts"],
        true_shooting_pct=raw["ts_pct"],
        offensive_rating=raw["off_rtg"],
        defensive_rating=raw["def_rtg"],
    )
```

---

## One Tool Per Aggregate Operation

Tools map to aggregate operations, not API endpoints. If a domain operation requires
multiple API calls, the tool handles all of them transparently.

```python
# Three internal API calls — one tool, one domain operation
def get_team_four_factors(team_id: str, season: str) -> FourFactors | ToolError:
    shooting = stats_api.get_shooting(team_id, season)
    turnovers = stats_api.get_turnovers(team_id, season)
    rebounding = stats_api.get_rebounding(team_id, season)
    return FourFactors(
        efg_pct=shooting["efg"],
        tov_rate=turnovers["tov_pct"],
        oreb_pct=rebounding["oreb_pct"],
        ft_rate=shooting["fta_per_fga"],
    )
```

---

## Tool Design Checklist

### Interface
- [ ] Tool name uses domain ubiquitous language (verb + domain noun)
- [ ] Parameters use domain identifiers, not internal keys or API params
- [ ] Return type is a domain model, not a raw API response shape
- [ ] Domain expert could read the signature and understand it immediately

### Implementation
- [ ] ACL translation is handled inside (ID mapping, format conversion, schema mapping)
- [ ] Multiple API calls are aggregated if they serve one domain operation
- [ ] Retry logic and timeouts are handled internally, transparent to the model
- [ ] Input normalization happens before any external call

### Errors
- [ ] Returns `ToolError` (not raises exception) for all anticipated failure modes
- [ ] `ToolError.code` uses domain terminology, not HTTP status codes
- [ ] `ToolError.suggestion` tells the model what to do next
- [ ] Error cases are unit tested explicitly

### State
- [ ] Tool holds no internal state between calls
- [ ] All required context is passed as explicit parameters
- [ ] No hidden session, cache, or mutable class-level state

---

## Summary

```
One tool      = one aggregate operation
Interface     = domain language (not API language)
Internals     = ACL logic (translation, aggregation, normalization)
Return type   = domain model (not raw API response)
Errors        = structured ToolError with suggestion field
State         = never held inside the tool
Transport     = irrelevant (REST, gRPC, local function — doesn't matter)
```

**The test**: could a domain expert read the tool signature and immediately understand
what it does? If yes, the abstraction level is correct.
