# Task 3 Submission (Part A, B, C, D) — Raw SQL Version
## Video demo
Link: https://www.loom.com/share/c516f8570549435f9cc6cd796bd3e9c9

## Part A — Fix the Bugs

### Bugs found in the provided code
- Bug 1: SQL built with f-strings from user input (`query`, `region`, `limit`) allows SQL injection.
- Bug 2: DB connection/cursor not safely managed (can leak resources on error).
- Bug 3: `recommend_wine` always returns success even if insert fails or transaction is not committed.

### Fixed endpoint code (raw SQL with comments)

```python
from fastapi import APIRouter, HTTPException, Query
from typing import Optional
from uuid import UUID
import psycopg2

router = APIRouter()

DB_CONFIG = {
    "host": "localhost",
    "database": "vinoseek",
    "user": "admin",
    "password": "password",
}


@router.get("/wines/search")
def search_wines(
    query: str = Query(..., min_length=1),
    budget_max: Optional[int] = Query(None, ge=0),
    region: Optional[str] = None,
    limit: int = Query(10, ge=1, le=50),
):
    # Fix: use context managers so connection/cursor are always closed.
    with psycopg2.connect(**DB_CONFIG) as conn:
        with conn.cursor() as cursor:
            # Fix: parameterized SQL prevents SQL injection.
            sql = """
                SELECT sku, name, price_hkd, region
                FROM wines
                WHERE name ILIKE %s
            """
            params = [f"%{query}%"]

            if budget_max is not None:
                sql += " AND price_hkd <= %s"
                params.append(budget_max)

            if region:
                sql += " AND region = %s"
                params.append(region)

            sql += " ORDER BY price_hkd ASC LIMIT %s"
            params.append(limit)

            cursor.execute(sql, tuple(params))
            results = cursor.fetchall()

    # Keep response shape close to task prompt: {"wines": results}
    return {"wines": results}


@router.post("/wines/recommend")
def recommend_wine(user_id: UUID, wine_sku: str):
    with psycopg2.connect(**DB_CONFIG) as conn:
        with conn.cursor() as cursor:
            # Part B missing feature: validate SKU exists before insert.
            cursor.execute("SELECT 1 FROM wines WHERE sku = %s LIMIT 1", (wine_sku,))
            if cursor.fetchone() is None:
                raise HTTPException(status_code=404, detail=f"Wine SKU '{wine_sku}' not found")

            # Fix: execute insert only after validation and commit transaction.
            cursor.execute(
                """
                INSERT INTO recommendations (user_id, wine_sku, created_at)
                VALUES (%s, %s, NOW())
                """,
                (str(user_id), wine_sku),
            )
            conn.commit()

    return {"status": "saved"}
```

---

## Part B — Missing Feature Added

The `recommend_wine` endpoint now validates that `wine_sku` exists in the `wines` table before inserting.

- If SKU does not exist -> returns `404 Not Found`.
- If SKU exists -> inserts into `recommendations` and returns success.

This prevents invalid recommendations and keeps referential integrity clean.

---

## Part C — Test for `search_wines`

Implemented test in `tests/test_api.py`:

```python
from fastapi.testclient import TestClient
from src.main import app


def test_search_wines_returns_filtered_results(monkeypatch):
    class FakeProvider:
        def search_wines(self, query, budget_max, region, limit):
            assert query == "test"
            assert budget_max == 500
            assert region == "Bordeaux"
            assert limit == 5
            return [
                {
                    "id": "11111111-1111-1111-1111-111111111111",
                    "name": "Test Wine",
                    "region": "Bordeaux",
                    "country": "France",
                    "type": "Red",
                    "variety": "Cabernet Sauvignon",
                    "price_hkd": 450,
                    "score": 90,
                    "body": "full",
                    "tasting_notes": "Dark fruit",
                    "occasions": ["business"],
                    "food_pairings": ["beef"],
                    "in_stock": True,
                    "thumb_url": None,
                }
            ]

    from src.routers import wines as wines_router

    monkeypatch.setattr(wines_router, "get_db_provider", lambda: FakeProvider())
    client = TestClient(app)

    response = client.get(
        "/api/wines/search",
        params={"query": "test", "budget_max": 500, "region": "Bordeaux", "limit": 5},
    )

    assert response.status_code == 200
    payload = response.json()
    assert "wines" in payload
    assert len(payload["wines"]) == 1
    assert payload["wines"][0]["name"] == "Test Wine"
```

Run:

```bash
python3 -m pytest -q tests/test_api.py
```

Observed result:
- `2 passed`

---

## Part D — Reflection (100–200 words)

I used AI tools to speed up analysis, implementation, and validation for this task also check the incorrect code and missing feature and generate testcase
Since we just have plain text without the Without the full functional code base, I also use AI to integrate with my demo recommendation system by helping me with guides and step-by-step to convert to add another database provider from Supabase to Postgres so that I can handle the API call in my backend and test the logic for search and recommend API in there.

AI was most helpful for reducing implementation time and providing a structured checklist of risks to fix. The main limitation was that AI doesn't have the full context of our local and also our service, so we need to take a very close into how the AI works. And usually large language model usually over-engineering. It usually creates more things than necessary. Sometimes we just need a very simple solution but usually over-engineer everything, so we need to take care of that. It can make sense at first, but if we do follow it way, it's going to break a lot of technical depth, so we need to avoid that.
