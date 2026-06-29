# Workflow

Workflow diagrams for the Munder Difflin multi-agent system (submission checklist item 2; see [Submission Checklist](./README.md#submission-checklist)).

Design narrative and agent roles are in [DESIGN.md](./DESIGN.md).

<a id="high-level-architecture"></a>

## High-Level Architecture

All specialist communication routes through the Orchestrator Agent.

```mermaid
flowchart TB
    Customer["Customer request + date"]
    Orch[OrchestratorAgent]
    Parser[RequestParserAgent]
    Inv[InventoryAgent]
    Quote[QuotingAgent]
    Order[OrderingAgent]
    DB[(SQLite DB)]

    Customer --> Orch
    Orch -->|"raw request text"| Parser
    Parser -->|"mapped items, qty, deadline"| Orch
    Orch -->|"availability check"| Inv
    Orch -->|"pricing request"| Quote
    Inv -->|"stock report, restock needs"| Orch
    Quote -->|"quote draft with discounts"| Orch
    Orch -->|"fulfill / restock instructions"| Order
    Order -->|"transaction IDs, delivery dates"| Orch
    Orch -->|"final text response"| Customer
    Inv --> DB
    Quote --> DB
    Order --> DB
```

<a id="agent-tools"></a>

## Agent Tools and Helper Functions

Each specialist agent accesses the database through `@tool` wrappers defined in [project_starter.py](./project_starter.py). The Orchestrator delegates via text handoffs and does not call DB tools directly.

```mermaid
flowchart TB
    subgraph parser [RequestParserAgent]
        T_list["list_catalog_items — catalog lookup"]
    end
    subgraph inv [InventoryAgent]
        T_all["check_all_stock — get_all_inventory"]
        T_stock["check_stock — get_stock_level"]
        T_details["get_inventory_item_details — inventory table"]
    end
    subgraph quote [QuotingAgent]
        T_hist["lookup_quote_history — search_quote_history"]
        T_price["get_unit_price — paper_supplies/inventory"]
    end
    subgraph order [OrderingAgent]
        T_txn["record_transaction — create_transaction"]
        T_cash["check_cash_balance — get_cash_balance"]
        T_deliv["estimate_delivery_date — get_supplier_delivery_date"]
        T_report["get_financial_report — generate_financial_report"]
    end
```

| Agent | Tool | Purpose | Starter function |
|-------|------|---------|------------------|
| Request Parser | `list_catalog_items` | List valid catalog names for item mapping | `paper_supplies` (read-only) |
| Inventory | `check_all_stock` | Return all items with positive stock as of date | `get_all_inventory(as_of_date)` |
| Inventory | `check_stock` | Return stock level for one item as of date | `get_stock_level(item_name, as_of_date)` |
| Inventory | `get_inventory_item_details` | Return min stock level and unit price | inventory table query |
| Quoting | `lookup_quote_history` | Search past quotes by keywords | `search_quote_history(search_terms, limit)` |
| Quoting | `get_unit_price` | Look up unit price for a catalog item | `paper_supplies` / inventory table |
| Ordering | `record_transaction` | Record a sale or stock order | `create_transaction(...)` |
| Ordering | `check_cash_balance` | Validate available cash before restock | `get_cash_balance(as_of_date)` |
| Ordering | `estimate_delivery_date` | Estimate supplier delivery ETA | `get_supplier_delivery_date(input_date, quantity)` |
| Ordering | `get_financial_report` | Post-order cash and inventory summary | `generate_financial_report(as_of_date)` |

<a id="per-request-sequence"></a>

## Per-Request Sequence

Message flow for one customer request: parallel inventory and quoting, parser clarification, specialist retry, restock branches, and failure paths.

```mermaid
sequenceDiagram
    participant Cust as Customer
    participant Orch as Orchestrator
    participant Parse as RequestParser
    participant Inv as Inventory
    participant Quote as Quoting
    participant Ord as Ordering
    participant DB as SQLite

    Cust->>Orch: request + request_date
    Orch->>Parse: parse and map items
    alt Parser returns unresolved items
        Parse-->>Orch: unresolved item names
        Orch-->>Cust: clarification request
    else Parsed items valid
        Parse-->>Orch: item list, quantities, deadline
        par Parallel checks
            Orch->>Inv: check stock as of date
            Inv->>DB: get_stock_level
            Inv-->>Orch: availability report
        and
            Orch->>Quote: build quote
            Quote->>DB: search_quote_history
            Quote-->>Orch: priced quote with bulk discount
        end
        alt Specialist transient error
            Orch->>Inv: retry specialist once
            Inv-->>Orch: availability report or error
        end
        alt Insufficient stock and cash OK
            Orch->>Ord: place stock_orders
            Ord->>DB: create_transaction stock_orders
            Ord-->>Orch: delivery ETA
            Orch->>Ord: record sales
            Ord->>DB: create_transaction sales
            Ord-->>Orch: confirmation
            Orch-->>Cust: quote + delivery + order summary
        else Insufficient stock and cash insufficient
            Orch-->>Cust: failure report
        else Sufficient stock
            Orch->>Ord: record sales
            Ord->>DB: create_transaction sales
            Ord-->>Orch: confirmation
            Orch-->>Cust: quote + delivery + order summary
        end
    end
```

<a id="routing-decisions"></a>

## Routing Decisions

The Orchestrator inspects Parser output and specialist reports to decide the next step. Details in [DESIGN.md routing strategy](./DESIGN.md#routing-strategy).

```mermaid
flowchart TD
    Start[CustomerRequest] --> Orch[OrchestratorAgent]
    Orch --> Parser[RequestParserAgent]
    Parser --> Decision{ParsedItemsValid}
    Decision -->|No| Clarify[ReturnClarificationToCustomer]
    Decision -->|Yes| Parallel[InventoryAndQuotingParallel]
    Parallel --> StockCheck{SufficientStock}
    StockCheck -->|No| CashCheck{SufficientCashForRestock}
    CashCheck -->|Yes| Restock[OrderingAgentStockOrders]
    CashCheck -->|No| FailReport[FailureReport]
    Restock --> Sales[OrderingAgentSales]
    StockCheck -->|Yes| Sales
    Sales --> Reply[UnifiedCustomerReply]
    FailReport --> Reply
    Clarify --> Reply
```

<a id="state-layers"></a>

## State Layers

Conversation-level state lives in the Orchestrator during a request; system-level state persists in SQLite across requests. Details in [DESIGN.md state management](./DESIGN.md#state-management).

```mermaid
flowchart TB
    subgraph conversation [ConversationStatePerRequest]
        OrchCtx["Orchestrator request context"]
        OrchCtx --- RawReq[Raw request]
        OrchCtx --- Parsed[Parsed items]
        OrchCtx --- InvReport[Inventory report]
        OrchCtx --- QuoteDraft[Quote draft]
        OrchCtx --- OrderResult[Ordering result]
    end
    subgraph system [SystemStateAcrossRequests]
        DB[(SQLite munder_difflin.db)]
        DB --- InventoryTbl[inventory]
        DB --- TransactionsTbl[transactions]
        DB --- QuotesTbl[quotes]
    end
    Orch[OrchestratorAgent] --> OrchCtx
    Inv[InventoryAgent] -->|"read"| DB
    Quote[QuotingAgent] -->|"read"| DB
    Order[OrderingAgent] -->|"sole writer"| DB
    Orch -->|"text handoffs"| Inv
    Orch -->|"text handoffs"| Quote
    Orch -->|"text handoffs"| Order
```

<a id="test-run-loop"></a>

## Test Run Loop

How `run_test_scenarios()` in `project_starter.py` drives the system across sample requests.

```mermaid
flowchart TD
    Start["run_test_scenarios()"] --> Init["init_database()"]
    Init --> Sort["Load quote_requests_sample.csv, sort by date"]
    Sort --> Loop{"For each request"}
    Loop --> AppendDate["Append Date of request to text"]
    AppendDate --> MAS["Orchestrator handles request"]
    MAS --> Report["generate_financial_report(request_date)"]
    Report --> Log["Append row to test_results.csv"]
    Log --> Loop
    Loop --> Final["Final financial report"]
```
