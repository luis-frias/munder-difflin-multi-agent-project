# Munder Difflin Multi-Agent System Project

Welcome to the starter code repository for the **Munder Difflin Paper Company Multi-Agent System Project**! This repository contains the starter code and tools you will need to design, build, and test a multi-agent system that supports core business operations at a fictional paper manufacturing company.

## Project Context

You've been hired as an AI consultant by Munder Difflin Paper Company, a fictional enterprise looking to modernize their workflows. They need a smart, modular **multi-agent system** to automate:

- **Inventory checks** and restocking decisions
- **Quote generation** for incoming sales inquiries
- **Order fulfillment** including supplier logistics and transactions

Your solution must use a maximum of **5 agents** and process inputs and outputs entirely via **text-based communication**.

This project uses **`smolagents`** (the same framework as lessons 1–7), combined with SQLite, `pandas`, and LLM prompt engineering.

---

## Repository Structure

```
munder-difflin-multi-agent-project/
├── README.md
├── requirements.txt
├── .env.example
├── .gitignore
├── project_starter.py
├── quote_requests.csv
├── quote_requests_sample.csv
├── quotes.csv
└── VOCAREUM_SETUP.md
```

Files not committed (created locally): `.env`, `.venv/`, `munder_difflin.db`, `test_results.csv`

---

## Install and Run

### Prerequisites

- **Python 3.10+** (required for `smolagents`)
- A **Vocareum OpenAI API key** from the Udacity classroom (starts with `voc-`)

### Setup

```bash
git clone https://github.com/YOUR_ORG/munder-difflin-multi-agent-project.git
cd munder-difflin-multi-agent-project

python3 -m venv .venv && source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install --upgrade pip
pip install smolagents
pip install -r requirements.txt
cp .env.example .env   # add Vocareum key (see below)
```

Edit `.env` and set your Vocareum key (use the same value for both variables):

```
OPENAI_API_KEY="voc-your-vocareum-api-key-here"
OPENAI_API_BASE="https://openai.vocareum.com/v1"
UDACITY_OPENAI_API_KEY="voc-your-vocareum-api-key-here"
```

See [VOCAREUM_SETUP.md](VOCAREUM_SETUP.md) for detailed API configuration.

### Run

Start by defining your agents in the `"YOUR MULTI AGENT STARTS HERE"` section inside `project_starter.py`. Once your agent team is ready:

```bash
python project_starter.py
```

This runs `run_test_scenarios()`, which simulates a series of customer requests. Your system should respond by coordinating inventory checks, generating quotes, and processing orders.

Output will include:

- Agent responses
- Cash and inventory updates
- Final financial report
- A `test_results.csv` file with all interaction logs

---

## Tips for Success

- Start by sketching a **flow diagram** to visualize agent responsibilities and interactions.
- Test individual agent tools before full orchestration.
- Always include **dates** in customer requests when passing data between agents.
- Ensure every quote includes **bulk discounts** and uses past data when available.
- Use the **exact item names** from the database to avoid transaction failures.

---

## Submission Checklist

Make sure to submit the following files:

1. Your completed `project_starter.py` with all agent logic
2. A **workflow diagram** describing your agent architecture and data flow
3. A `README.txt` or `design_notes.txt` explaining how your system works
4. Outputs from your test run (like `test_results.csv`)

---
