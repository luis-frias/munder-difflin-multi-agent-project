# Implementation Guide

Coding patterns for building the Munder Difflin multi-agent system in [project_starter.py](./project_starter.py). This document implements [DESIGN.md](./DESIGN.md); diagrams are in [WORKFLOW.md](./WORKFLOW.md).

**Doc relationship:** [DESIGN.md](./DESIGN.md) is the architectural source of truth. This file describes coding patterns that implement it; both are aligned on architecture today.

| Document | Owns |
|----------|------|
| [DESIGN.md](./DESIGN.md) | Agent roles, routing, state layers, policies, handoff *content*, tool-to-function mapping |
| **IMPLEMENTATION.md** (this file) | smolagents wiring, orchestration loop, tool wrappers, build order, Lesson 5 pattern mapping |
| [WORKFLOW.md](./WORKFLOW.md) | Mermaid diagrams only |

---

## Lesson 5 pattern mapping

The course Lesson 5 exercise (Colombian Fruit Market) teaches orchestration discipline. Adapt the **patterns**, not the **storage model**.

### Port from Lesson 5

| Lesson 5 pattern | Munder Difflin equivalent |
|------------------|---------------------------|
| Orchestrator builds a text prompt from accumulated context before delegating | Orchestrator assembles handoff strings from pipeline artifacts + `request_date` |
| After each specialist returns, store the text result for the next step | Keep per-request strings: `parsed_items`, `inventory_report`, `quote_draft`, `order_result` |
| Specialists access state only through `@tool` functions | Wrap starter DB functions in `@tool`; never pass `db_engine` to agents |
| Orchestrator wrapper `@tool` → `self.specialist.run(text)` | Same pattern for Parser, Inventory, Quoting, Ordering |
| Orchestrator is the only agent that invokes specialists | No P2P; see DESIGN.md Rules of Engagement |

### Do NOT port from Lesson 5

| Lesson 5 pattern | Why not |
|------------------|---------|
| `user_states = {}` global dict | SQLite is the cross-request store; see DESIGN.md System-level state |
| `save_user_state()` stub | Real persistence via `create_transaction()` |
| Per-`user_id` session scope | Business state is company-wide (inventory, cash, quotes) |
| Multiple agents writing the same in-memory dict | Single-writer policy: only Ordering Agent writes transactions |

---

## smolagents wiring

**Chosen delegation approach:** wrapper `@tool` functions on the Orchestrator that call `self.<specialist>.run(handoff_text)`.

See DESIGN.md Framework note; this project uses wrapper tools (Lesson 5 pattern).

### Agent class structure

Define one `ToolCallingAgent` subclass per specialist, plus an `Orchestrator` class:

```
RequestParserAgent   — read-only catalog lookup tools
InventoryAgent       — get_all_inventory, get_stock_level
QuotingAgent         — search_quote_history (+ unit price lookup as needed)
OrderingAgent          — create_transaction, get_cash_balance, get_supplier_delivery_date
Orchestrator           — wrapper tools that delegate to the four specialists above
```

Place agent definitions in the `"YOUR MULTI AGENT STARTS HERE"` section of [project_starter.py](./project_starter.py) (~lines 607–609).

### Orchestrator wrapper tools (sketch)

Each wrapper tool builds a handoff prompt and delegates:

```python
@tool
def parse_request(request_text: str) -> str:
    """Parse and map customer request to exact catalog item names."""
    return self.parser.run(f"Parse this request:\n{request_text}")

@tool
def check_inventory(parsed_items: str, request_date: str) -> str:
    """Check stock levels as of request_date."""
    return self.inventory.run(
        f"Request date: {request_date}\nLine items:\n{parsed_items}\n"
        "Report availability and restock needs."
    )
# ... similar wrappers for quoting and ordering
```

The Orchestrator's own `run()` receives the customer request and coordinates via these wrapper tools.

---

## Per-request orchestration loop

Maps [DESIGN.md Data Flow](./DESIGN.md#data-flow) steps 1–9 to code flow. See [WORKFLOW.md sequence diagram](./WORKFLOW.md#per-request-sequence) for branching (clarification, restock, failure).

```text
handle_request(raw_request, request_date):
  ctx = {raw_request, request_date}          # conversation-level state (strings)

  1. parsed = delegate_to_parser(ctx)      # step 2–3
     if unresolved items → return clarification to customer

  2. inv_report = delegate_to_inventory(parsed, request_date)   # step 4–5
     quote_draft = delegate_to_quoting(parsed, request_date)    # step 4, 6 (parallel)

  3. if specialist error → retry once; if still failing → failure report

  4. order_result = delegate_to_ordering(     # step 7–8
       parsed, inv_report, quote_draft, request_date)

  5. return compose_customer_reply(           # step 9
       quote_draft, order_result, inv_report)
```

**Accumulation rule:** each delegation returns plain text. Store that text in local variables (or a simple dict of strings). Never pass Python objects (DataFrames, DB handles) between agents.

**Routing rule:** inspect parsed text and specialist reports to choose branches — see DESIGN.md [routing decision table](./DESIGN.md#routing-decision-table).

---

## Handoff prompt construction

Each delegation is a **constructed text prompt**, not a shared object. Fields come from the [DESIGN.md Data flow management](./DESIGN.md#data-flow-management) table.

| Delegation | Include in prompt | Exclude |
|------------|-------------------|---------|
| Orchestrator → Parser | Full NL request, `request_date` | n/a |
| Orchestrator → Inventory | `request_date`, parsed line items (exact `item_name`, qty) | Pricing, quote history, stock from prior runs |
| Orchestrator → Quoting | Parsed items, job/event context, `request_date` | Stock levels, restock recommendations |
| Orchestrator → Ordering | Actionable items, restock qty, sale prices from quote, `request_date` | Raw NL request, colloquial names |
| Orchestrator → Customer | Quote, delivery ETA, order summary | Internal errors (unless customer-relevant) |

Always embed `request_date` explicitly in specialist prompts so tools scope DB queries correctly.

---

## Tool wrapper pattern

All SQLite access goes through agent-local `@tool` wrappers around starter functions. Agents never receive `db_engine` or raw SQL in prompts.

| Agent | Wrap these starter functions | Access |
|-------|------------------------------|--------|
| Inventory | `get_all_inventory`, `get_stock_level` | Read only |
| Quoting | `search_quote_history` | Read only |
| Ordering | `create_transaction`, `get_cash_balance`, `get_supplier_delivery_date`, `generate_financial_report` | Write (transactions) + read |
| Parser | Catalog lookup (read `paper_supplies` or equivalent) | Read only |
| Orchestrator | None for DB; test harness calls `generate_financial_report` for logging | No transaction writes |

Example wrapper:

```python
@tool
def check_stock(item_name: str, as_of_date: str) -> str:
    """Check stock level for one item as of a date."""
    df = get_stock_level(item_name, as_of_date)
    return df.to_string()
```

Tools return **strings** (or values smolagents can serialize to text) so specialist outputs stay text-based.

---

## Conversation vs system state in code

| Layer | Where in code | Lifetime | Access |
|-------|---------------|----------|--------|
| **Conversation** | Local variables / string dict inside `handle_request()` | One customer request | Orchestrator accumulates; passes filtered text at each handoff |
| **System** | SQLite via `munder_difflin.db` | Across all requests in test run | Specialists read/write through `@tool` wrappers scoped to `request_date` |

- Do **not** use a module-level dict (Lesson 5 `user_states` pattern).
- `run_test_scenarios()` tracks `current_cash` / `current_inventory` for **logging only** (~lines 628–669). Agents treat SQLite as truth via tools.
- Cross-request persistence happens automatically when Ordering Agent calls `create_transaction()`; the next request's tools read updated stock/cash for its `request_date`.

---

## Suggested build order

1. **Tool wrappers** — one `@tool` per starter function, grouped by agent domain; test individually if possible.
2. **Specialist agents** — `ToolCallingAgent` subclasses with their tool lists and descriptions from DESIGN.md Agent Roles.
3. **Orchestrator** — wrapper tools + `handle_request()` loop with routing branches.
4. **Wire test harness** — initialize system once before the loop (~line 637); call `handle_request()` inside the loop (~line 658).
5. **Run** — `python project_starter.py`; verify `test_results.csv` and financial report.

---

## Integration points in project_starter.py

| Location | What to add |
|----------|-------------|
| ~lines 590–605 | `@tool` wrappers for each agent (inventory, quoting, ordering, parser) |
| ~lines 607–609 | Agent class definitions and Orchestrator; create instance before test loop |
| ~line 637 | `# INITIALIZE YOUR MULTI AGENT SYSTEM HERE` — build orchestrator once |
| ~line 658 | `# USE YOUR MULTI AGENT SYSTEM TO HANDLE THE REQUEST` — `response = orchestrator.handle_request(request_with_date, request_date)` |

The test harness already provides `request_with_date` (NL request + date appended) and sorted dates. Extract `request_date` from the row or parse it from the request string consistently.

---

## Quick reference during implementation

```text
DESIGN.md   → what agents exist, what each handoff contains, who writes DB
WORKFLOW.md → sequence and state-layer diagrams
This file   → how to wire smolagents, build the loop, wrap tools
Lesson 5    → already captured here; no need to reopen during coding
```
