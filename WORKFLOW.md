# Workflow

Workflow diagrams for the Munder Difflin multi-agent system — satisfies submission checklist item 2 (see [Submission Checklist](./README.md#submission-checklist) in README).

Design narrative and agent roles are in [DESIGN.md](./DESIGN.md).

<a id="high-level-architecture"></a>

## High-Level Architecture

Shows the Orchestrator Pattern: all specialist communication routes through the Orchestrator Agent.

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

<a id="per-request-sequence"></a>

## Per-Request Sequence

Shows the step-by-step message flow for a single customer request, including parallel inventory/quoting checks and the restock branch.

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
    alt Insufficient stock
        Orch->>Ord: place stock_orders
        Ord->>DB: create_transaction stock_orders
        Ord-->>Orch: delivery ETA
    end
    Orch->>Ord: record sales
    Ord->>DB: create_transaction sales
    Ord-->>Orch: confirmation
    Orch-->>Cust: quote + delivery + order summary
```

<a id="test-run-loop"></a>

## Test Run Loop

Shows how `run_test_scenarios()` in `project_starter.py` drives the multi-agent system across all sample requests.

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
