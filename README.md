# VinoBuzz — AI Engineer Take-Home Assignment

This repository contains my submissions for the VinoBuzz AI Engineer take-home assignment.

<video src="https://github-production-user-asset-6210df.s3.amazonaws.com/56016006/573918661-5d2a33c1-9f67-486c-b21a-468b41729408.mp4?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=AKIAVCODYLSA53PQK4ZA%2F20260405%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Date=20260405T142016Z&X-Amz-Expires=300&X-Amz-Signature=63849f263e7c1abd6ee56e4b611f88cd7a71d0b6cb2c9eff4bb133d2a42d5c3f&X-Amz-SignedHeaders=host" controls width="100%"></video>

---

## Task 1 — Conversational Wine Recommendation Workflow

**Demo repo:** https://github.com/QuocVietHa08/deomo-recommendation-system

A conversational AI workflow built with FastGPT that allows users to get personalized wine recommendations through a natural language interface. The system collects user preferences (occasion, budget, region, taste profile) through a guided conversation and returns curated wine suggestions.

---

## Task 2 — OpenClaw Research Essay

### Section A — What is OpenClaw?

OpenClaw is a free, open-source autonomous AI agent platform that helps create an AI Assistant, create automation workflow, and access to your local machine.

**Architecture in plain terms:**

OpenClaw uses a hub-and-spoke model. At the center is a Gateway process that runs on your own machine (or a VPS). This gateway bridges two worlds: on one side, it connects to multiple messaging platforms (WhatsApp, Telegram, Slack, Discord, WeChat, LINE, iMessage, and more) via platform adapters; on the other side, it forwards messages to an Agent.

You can configure multiple agents and these agents can connect to a single or multiple channels to do different things. The Agent talks to whichever AI model you configure (Claude, GPT-4, Gemini, DeepSeek, or a local Ollama model), so you are never locked into one provider.

**What the agent can do:**
- Control a browser using the Chrome DevTools Protocol (CDP) — logging in, clicking, scraping, filling forms, taking screenshots
- Run shell commands directly on the host machine
- Call external APIs or webhooks
- Schedule recurring jobs via cron-style triggers

All conversation history and memory is stored as plain Markdown files on your local machine. It can use plugins + skills that already exist when you download the package or use skills from the skill hub to add extra power to the AI.

---

### Section B — Potential Applications for VinoBuzz

**Use Case 1: Multi-Channel AI support (WhatsApp / WeChat / LINE / Discord)**

We can connect OpenClaw with multiple channels, each serving different purposes — Discord with multiple sub-channels for engineers to troubleshoot errors, WhatsApp and LINE or WeChat to support users, and Telegram for team communication.

Example flow: Customer sends "I need a red wine under HK$400 for a dinner party" on WhatsApp → OpenClaw receives it → agent calls VinoBuzz's recommendation API → agent formats and sends the wine suggestions back to the customer on WhatsApp, with labels and price.

**Use Case 2: Automated and cron job worker**

Each day, we can set up cron jobs on OpenClaw to automate routine tasks:

1. For engineering managers — set up a workflow to send a summary of completed PRs or tasks at end of day or morning, so managers can track progress.
2. For engineers — schedule health checks on services and system status updates, improving stability monitoring and workflow.S

**Use Case 3: Research assistant for manager and engineer about technologies and market trend**

Each day, engineer and manager need to understand more about market, understand more about the trend, so we can create a system that can crawl all the new information.

1. For engineers, they can gather information about specific trends, for example, AI frameworks. Based on that, they can improve the system day by day.
2. For manager, they can research about the market trends. They can research about the product information compared to the product competitions or maybe research about product that have similar features compared to our product. Therefore, the manager can make a plan for engineer and the team to grow the product itself.

**Use Case 4: Marketer assistant for engineer and manager**

A marketing assistant can help manager and engineer to market the product through multiple social media, for example, LinkedIn, Instagram, Facebook, Twitter, and may more.

1. For engineers, they can create a technical blog, create a system blog that will convert into traffic, so more users can come across to the website through their blog and through their posts on social media.
2. For manager, they can also do the same thing but in different topic like management framework, management technique that can control that used to manner small team go fast. Manager also can post daily update about the product so it can bring more customer to the website if the feature is interesting.
---

### Section C — Risks and Limitations

**Data Security**: 
One single wrong move and cause a lot of money. For example if user don't know how to project or add new system layer, OpenClaw can delete important stuff or leak sensitive information

**Contain bug in the system core logic**
OpenClaw itself is a young open-source project. So it might contain may critical bug.

**Cost at Scale**
OpenClaw has no built-in rate limiting or LLM cost budgeting. Every customer message on WhatsApp triggers an LLM API call. It can cost a lot of money if customer spamming our channel that power by OpenClaw

**Learning curve**
It is quite hard to configure OpenClaw in the way we want, so the team needs to take time to learn new stuff to understand the full system if they actually want to integrate into the team and the team needs to keep up with the technology because OpenClaw changes really fast and there are a lot of new updates that will come up in the future.

---

# Outdated solution. Please check [`task3.md`](./task3.md).
<del>
## Task 3 — AI-Assisted Engineering

Fixing and extending a Python FastAPI wine search/recommendation endpoint.

---

### Part A — Fix the bugs

Three bugs were identified and fixed:

**Bug 1: SQL Injection** (`search_wines`)
The original code used f-string interpolation directly in the SQL query, allowing arbitrary SQL execution. Fixed by using psycopg2 parameterized queries with `%s` placeholders.

**Bug 2: Missing `conn.commit()`** (`recommend_wine`)
The endpoint executed an INSERT but never called `conn.commit()`, so nothing was actually persisted to the database despite returning `{"status": "saved"}`.

**Bug 3: Connection leak** (both endpoints)
Both endpoints opened a database connection and never closed it, eventually exhausting the PostgreSQL connection pool. Fixed by wrapping in `try/finally` blocks.

```python
from fastapi import APIRouter, HTTPException
from typing import Optional
import psycopg2

router = APIRouter()

DB_CONFIG = {
    "host": "localhost",
    "database": "vinoseek",
    "user": "admin",
    "password": "password"
}


@router.get("/wines/search")
def search_wines(
    query: str,
    budget_max: Optional[int] = None,
    region: Optional[str] = None,
    limit: int = 10
):
    # BUG FIX 1:
    # The original code used f-string interpolation directly in the SQL query, which allows SQL injection.
    # For example, a query value of "'; DROP TABLE wines; --" would execute arbitrary SQL.
    # Fix: use parameterized queries with %s placeholders, letting psycopg2 handle escaping safely.
    conn = psycopg2.connect(**DB_CONFIG)
    try:
        cursor = conn.cursor()

        sql = "SELECT sku, name, price, region FROM wines WHERE name LIKE %s"
        params = [f"%{query}%"]

        if budget_max:
            sql += " AND price <= %s"
            params.append(budget_max)

        if region:
            # BUG FIX 1 (continued): Same injection risk existed here with region parameter.
            # Now also parameterized.
            sql += " AND region = %s"
            params.append(region)

        sql += " LIMIT %s"
        params.append(limit)

        cursor.execute(sql, params)
        results = cursor.fetchall()

        return {"wines": results}
    finally:
        # BUG FIX 2: The original code never closed the database connection, causing connection leaks.
        # Every request would open a new connection and never release it, eventually exhausting the PostgreSQL connection pool.
        # Fix: always close in finally so the connection is released even if an exception is raised.
        conn.close()


@router.post("/wines/recommend")
def recommend_wine(user_id: int, wine_sku: str):
    conn = psycopg2.connect(**DB_CONFIG)
    try:
        cursor = conn.cursor()

        # MISSING FEATURE: Validate that the wine SKU actually exists before saving a recommendation.
        # Without this check, the recommendations table can accumulate references to non-existent wines, causing data integrity issues.
        cursor.execute("SELECT 1 FROM wines WHERE sku = %s", (wine_sku,))
        if cursor.fetchone() is None:
            raise HTTPException(
                status_code=404,
                detail=f"Wine SKU '{wine_sku}' not found"
            )

        cursor.execute(
            "INSERT INTO recommendations (user_id, wine_sku, created_at) VALUES (%s, %s, NOW())",
            (user_id, wine_sku)
        )

        # BUG FIX 3: The original code never called conn.commit(), so the INSERT was never actually persisted to the database.
        # The function returned {"status": "saved"} even though nothing was saved.
        # Fix: commit after a successful write operation.
        conn.commit()

        return {"status": "saved"}
    finally:
        # BUG FIX 2 (same as search_wines): close connection to avoid leaks.
        conn.close()
```

---

### Part B — Add a missing feature

`recommend_wine` now validates that the wine SKU exists before inserting. If the SKU is not found, it returns HTTP 404. This prevents orphaned references in the recommendations table.

The key addition (already shown in the code above):

```python
cursor.execute("SELECT 1 FROM wines WHERE sku = %s", (wine_sku,))
if cursor.fetchone() is None:
    raise HTTPException(status_code=404, detail=f"Wine SKU '{wine_sku}' not found")
```

---

### Part C — Tests

Using pytest with `unittest.mock` to patch `psycopg2.connect` — no real database needed.

```python
"""
Tests for the search_wines endpoint in main.py.
Uses unittest.mock to patch psycopg2.connect so no real database is needed.
"""

import pytest
from unittest.mock import patch, MagicMock
from fastapi.testclient import TestClient
from fastapi import FastAPI

from task3.main import router

app = FastAPI()
app.include_router(router)
client = TestClient(app)


def make_mock_conn(rows=None):
    """Helper: returns a mock psycopg2 connection whose cursor returns `rows`."""
    mock_cursor = MagicMock()
    mock_cursor.fetchall.return_value = rows or []
    mock_conn = MagicMock()
    mock_conn.cursor.return_value = mock_cursor
    return mock_conn, mock_cursor


class TestSearchWines:
    def test_returns_wines_for_valid_query(self):
        """A plain query should return whatever the database returns."""
        fake_rows = [
            ("CM2018", "Château Margaux 2018", 680, "Bordeaux"),
            ("RM2019", "Romanée-Conti 2019", 4500, "Burgundy"),
        ]
        mock_conn, mock_cursor = make_mock_conn(fake_rows)

        with patch("task3.main.psycopg2.connect", return_value=mock_conn):
            response = client.get("/wines/search?query=château")

        assert response.status_code == 200
        data = response.json()
        assert len(data["wines"]) == 2
        assert data["wines"][0][0] == "CM2018"

    def test_budget_max_adds_price_filter(self):
        """When budget_max is provided the executed SQL should include a price constraint."""
        mock_conn, mock_cursor = make_mock_conn([])

        with patch("task3.main.psycopg2.connect", return_value=mock_conn):
            response = client.get("/wines/search?query=red&budget_max=500")

        assert response.status_code == 200
        call_args = mock_cursor.execute.call_args
        sql, params = call_args[0]
        assert "price" in sql.lower()
        assert 500 in params

    def test_region_filter_is_applied(self):
        """When region is provided the SQL should include a region constraint."""
        mock_conn, mock_cursor = make_mock_conn([])

        with patch("task3.main.psycopg2.connect", return_value=mock_conn):
            response = client.get("/wines/search?query=wine&region=Bordeaux")

        assert response.status_code == 200
        call_args = mock_cursor.execute.call_args
        sql, params = call_args[0]
        assert "region" in sql.lower()
        assert "Bordeaux" in params

    def test_sql_injection_in_query_is_parameterized(self):
        """
        A malicious query string should be passed as a parameter, not interpolated
        into the SQL. We verify this by checking that execute() receives the raw
        injection string as a bound parameter (not embedded in the SQL text itself).
        """
        injection = "'; DROP TABLE wines; --"
        mock_conn, mock_cursor = make_mock_conn([])

        with patch("task3.main.psycopg2.connect", return_value=mock_conn):
            response = client.get(f"/wines/search?query={injection}")

        assert response.status_code == 200
        call_args = mock_cursor.execute.call_args
        sql, params = call_args[0]
        assert injection not in sql
        assert any(injection in str(p) for p in params)

    def test_connection_is_closed_after_request(self):
        """The database connection should be closed even on a successful request."""
        mock_conn, _ = make_mock_conn([])

        with patch("task3.main.psycopg2.connect", return_value=mock_conn):
            client.get("/wines/search?query=test")

        mock_conn.close.assert_called_once()

    def test_empty_result_returns_empty_list(self):
        """When no wines match, the endpoint should return an empty list, not an error."""
        mock_conn, _ = make_mock_conn([])

        with patch("task3.main.psycopg2.connect", return_value=mock_conn):
            response = client.get("/wines/search?query=nonexistent_wine_xyz")

        assert response.status_code == 200
        assert response.json() == {"wines": []}
```

---

### Part D — Reflection

I used **Claude Code (claude-sonnet-4-6)** as my primary AI assistant throughout this task.

I presented the original endpoint code to Claude and asked it to identify bugs. Claude flagged the SQL injection vulnerability in `search_wines` — correctly explaining that f-string interpolation allowed arbitrary SQL execution and recommending parameterized queries with `%s` placeholders. It also caught the missing `conn.commit()` in `recommend_wine`, accurately noting that psycopg2 does not auto-commit INSERT statements.

For the missing feature (SKU validation), Claude generated the `SELECT 1 FROM wines WHERE sku = %s` pattern and the correct `HTTPException(status_code=404)` response — both idiomatic and correct on the first attempt. For test generation, Claude scaffolded the full pytest structure with `unittest.mock.patch` to intercept `psycopg2.connect` without adjustment.

I think Claude Code really good at identify the errors and suggest feature. But there are some aspect it still wrong:

1. It don't understand the full context of the problem and it usually returns incorrect format if we don't actually know how to handle it. For example, if I want to have a cleaner code, I want to separate components. I need to really ask it about you should separate this service or this function into another smaller function.
2. Sometimes, the Claude Code just doesn't understand the domain aspect. For example, if it's an e-commerce site plus AI integration, it needs to understand that. But usually, for the first prompt, it assumes it's just a generic project. 
---

## Stack

- **AI Layer:** FastGPT / custom conversational workflow
- **Backend (Task 3):** Python, FastAPI, psycopg2, PostgreSQL
- **Testing:** pytest, unittest.mock
</del>
