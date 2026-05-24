# Finance Agent

A conversational AI agent that answers questions about your personal spending, tracks budget goals across sessions, and delivers proactive weekly insights — built with LangChain, FastAPI, and a two-layer memory architecture.

---

## What It Does

You talk to the agent in plain English. It figures out what you are asking, pulls the relevant transaction data, and replies with a grounded answer:

- *"How much did I spend on food this week?"* — queries the last 7 days and breaks it down by category
- *"My budget for dining is $300 a month"* — stores that goal in long-term memory and references it in future responses
- *"Am I on track this month?"* — retrieves your stored budget goals, computes actual spending, and tells you where you stand
- *"Give me a weekly insight"* — generates a proactive GPT-4o summary of your spending patterns against your stated goals

---

## Architecture

```
User (Streamlit UI or API client)
    |
    |-- POST /chat ---------> FastAPI
                                |
                                |-- MemoryManager
                                |     |-- ShortTermMemory   (in-memory conversation buffer, last 20 messages)
                                |     `-- JsonLongTermMemory (file-backed JSON, persists budget goals)
                                |
                                |-- Tool routing (keyword-based)
                                |     |-- SpendingTool   --> transaction source --> aggregation
                                |     `-- BudgetTool     --> long-term memory + transaction source
                                |
                                `-- LangChain ChatOpenAI (GPT-4o)
                                      system prompt = memory facts + tool results
                                      conversation history = last N turns
```

**Memory architecture — two layers:**

Short-term memory holds the sliding conversation window (up to 20 messages). Every turn is injected into the LangChain prompt as conversation history so the model has full context of the current session.

Long-term memory persists facts across sessions in a JSON file. When the agent detects budget-setting language in a message ("my budget", "I want to spend", "limit of"), it stores that fact keyed by category. On every turn, relevant facts are retrieved by keyword match and injected into the system prompt before the LLM call.

**Transaction sources — pluggable via abstract base class:**

The `BaseTransactionSource` interface means any data source can be swapped in without touching the agent or tools. Two implementations ship out of the box: `PlaidClient` for live bank data (requires Plaid sandbox credentials) and `CsvImporter` for CSV uploads backed by SQLite.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | Python 3.11+, FastAPI |
| LLM + orchestration | LangChain, OpenAI GPT-4o |
| Transaction data | Plaid API (sandbox) or CSV + SQLite |
| Short-term memory | In-process Python list (sliding window) |
| Long-term memory | JSON file (file-backed key-value store) |
| Frontend | Streamlit |
| Categorization | GPT-4o zero-shot classification |

---

## Project Structure

```
finance_agent/
|
|-- app/
|   |-- agents/
|   |   `-- finance_agent.py       # Main agent: tool routing, memory injection, LLM call
|   |
|   |-- core/
|   |   |-- config.py              # Pydantic settings loaded from .env
|   |   `-- prompts.py             # System prompts for chat, categorization, and insights
|   |
|   |-- memory/
|   |   |-- memory_manager.py      # Coordinator: short-term + long-term (SRP)
|   |   |-- short_term.py          # In-memory conversation buffer, max 20 messages
|   |   `-- long_term.py           # File-backed JSON memory with abstract base class
|   |
|   |-- models/
|   |   `-- schemas.py             # Pydantic models: Transaction, ChatRequest, WeeklyInsight
|   |
|   |-- routers/
|   |   |-- chat.py                # POST /chat, DELETE /memory
|   |   |-- transactions.py        # CSV upload and transaction query endpoints
|   |   `-- insights.py            # GET /insights/weekly
|   |
|   |-- services/
|   |   |-- plaid_client.py        # Plaid sandbox client (BaseTransactionSource)
|   |   |-- csv_importer.py        # CSV importer + SQLite store (BaseTransactionSource)
|   |   `-- categorizer.py         # GPT-4o zero-shot transaction categorization
|   |
|   |-- tools/
|   |   |-- spending_tool.py       # Aggregates transactions by category and date range
|   |   |-- budget_tool.py         # Compares spending to stored budget goals
|   |   `-- insight_tool.py        # Generates weekly summary via GPT-4o
|   |
|   `-- main.py                    # FastAPI app, dependency wiring, lifespan
|
|-- frontend/
|   `-- streamlit_app.py           # Chat UI with weekly insight panel
|
|-- tests/
|   |-- test_csv_importer.py
|   |-- test_memory_manager.py
|   `-- test_spending_tool.py
|
|-- requirements.txt
|-- .env.example
`-- README.md
```

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Python 3.11+ | Check with `python3 --version` |
| OpenAI API key | GPT-4o access at platform.openai.com |
| Plaid credentials (optional) | Free sandbox at plaid.com — skip if using CSV import |

---

## Getting Started

**Step 1 — Install dependencies**

```bash
pip install -r requirements.txt
```

**Step 2 — Configure credentials**

```bash
cp .env.example .env
```

Open `.env` and fill in:

```
OPENAI_API_KEY=sk-...
CHAT_MODEL=gpt-4o
DB_PATH=./data/transactions.db

# Optional — only needed for live Plaid data
PLAID_CLIENT_ID=
PLAID_SECRET=
PLAID_ENV=sandbox
```

**Step 3 — Start the backend**

```bash
uvicorn app.main:app --reload
```

Backend runs at http://localhost:8000

**Step 4 — Start the frontend**

```bash
streamlit run frontend/streamlit_app.py
```

Open http://localhost:8501 in your browser.

**Step 5 — Load transactions**

If you are not using Plaid, upload a CSV via the UI or the `/transactions/upload` endpoint. Expected columns:

```
date,description,amount,category
2024-01-15,Starbucks,6.50,food
2024-01-16,Uber,12.00,transport
```

The `category` column is optional — if omitted, GPT-4o classifies each transaction automatically.

---

## API Endpoints

| Method | Path | Description |
|---|---|---|
| POST | `/chat` | Send a message, get a reply with memory context |
| DELETE | `/memory` | Clear all short-term and long-term memory |
| POST | `/transactions/upload` | Upload a CSV of transactions |
| GET | `/transactions` | Query stored transactions (filter by days, category, amount) |
| GET | `/insights/weekly` | Generate a proactive weekly spending summary |

---

## Example Conversations

```
You:   How much did I spend last week?
Agent: You spent $284.50 over the last 7 days across 4 categories:
       food ($110.00), transport ($64.50), entertainment ($60.00), shopping ($50.00).

You:   My monthly food budget is $400.
Agent: Got it — I'll remember your $400 monthly food budget and track it going forward.

You:   Am I on track for food this month?
Agent: You have spent $220 on food so far this month against your $400 goal.
       You are 55% through your budget with roughly half the month remaining — on track.
```

---

## Design Decisions

**Keyword-based tool routing over LangChain agents:** The agent inspects incoming messages for intent keywords and calls the appropriate tool directly, rather than delegating routing to an LLM. This makes the routing deterministic, fast, and easy to test — important for a financial assistant where predictability matters.

**Abstract transaction source:** `BaseTransactionSource` decouples the agent from any specific data provider. Switching from CSV to Plaid (or any other source) requires no changes to the agent, tools, or memory layers.

**Separation of concerns across tools:** Each tool (`SpendingTool`, `BudgetTool`, `InsightTool`) has a single responsibility and its own dependencies injected at construction time, making each independently testable.
