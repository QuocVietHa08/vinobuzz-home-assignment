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

## Task 3 — AI-Assisted Engineering

**Source:** [task3/](task3/)

Fixing and extending a Python FastAPI wine search/recommendation endpoint.

| File | Description |
|------|-------------|
| [task3/main.py](task3/main.py) | Fixed endpoint with comments explaining each bug and the added SKU validation feature |
| [task3/test_wines.py](task3/test_wines.py) | pytest tests for `search_wines` covering normal queries, filters, SQL injection safety, and connection cleanup |
| [task3/REFLECTION.md](task3/REFLECTION.md) | Reflection on how Claude Code was used and where it fell short |

### Bugs Fixed
1. **SQL Injection** — `search_wines` used f-string interpolation in raw SQL; replaced with psycopg2 parameterized queries (`%s` placeholders)
2. **Missing `conn.commit()`** — `recommend_wine` executed an INSERT but never committed it, so nothing was persisted
3. **Connection leak** — Both endpoints opened database connections but never closed them; wrapped in `try/finally` blocks

### Feature Added
- `recommend_wine` now validates the wine SKU exists in the database before inserting; returns HTTP 404 if not found

### Code

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

## Stack

- **AI Layer:** FastGPT / custom conversational workflow
- **Backend (Task 3):** Python, FastAPI, psycopg2, PostgreSQL
- **Testing:** pytest, unittest.mock
