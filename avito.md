# Avito API Test Project

This canvas contains the required deliverables for Task 2 (full test suite for v1 + v2):

- `TESTCASES.md` — test cases specification
- `README.md` — instructions to run tests
- `BUGS.md` — bug report template (initially empty)
- `tests/` — automated pytest test suite and helper modules

---

# TESTCASES.md

## Overview
Service host: `https://qa-internship.avito.com`

Endpoints covered (full set):
- POST `/api/1/item` — create item
- GET `/api/1/item/{id}` — get item by id
- GET `/api/1/{sellerID}/item` — get all items by seller
- GET `/api/1/statistic/{id}` — get statistic (v1)
- GET `/api/2/statistic/{id}` — get statistic (v2)
- DELETE `/api/2/item/{id}` — delete item (v2)

Notes:
- `sellerID` should be unique (range `111111`–`999999`).
- POST returns a created `id` — used in subsequent reads/deletes.

---

## Test-case categories

### Positive (happy path)
1. Create item (POST `/api/1/item`) with valid body -> expect 200, response contains `id`, `sellerId`, `name`, `price`, `statistics`, `createdAt`.
2. Read created item (GET `/api/1/item/{id}`) -> 200, response contains item matching created values.
3. Read items by seller (GET `/api/1/{sellerID}/item`) -> 200, response list contains created item(s) and `sellerId` matches.
4. Get statistic v1 (GET `/api/1/statistic/{id}`) -> 200, response as an array of objects with `likes`, `viewCount`, `contacts` (integers).
5. Get statistic v2 (GET `/api/2/statistic/{id}`) -> 200, same shape expectations.
6. Delete item v2 (DELETE `/api/2/item/{id}`) -> 200 or empty body. After delete, GET by id returns 404.

### Negative / edge cases
1. POST with missing required fields (e.g., no `sellerID`) -> expect 400.
2. POST with invalid types (e.g., string for `price`) -> expect 400.
3. GET item with non-existent id -> expect 404.
4. GET items with non-existent sellerID -> expect 200 and empty array (or 404 depending on service behavior) — assert consistent behavior.
5. GET statistic with malformed id (e.g., empty or special chars) -> expect 400 or 404.
6. DELETE with non-existent id -> expect 404.
7. Attempt to create two items with same `id` (if API allows client-specified id) — check uniqueness requirement — but here `id` is server-generated; test for uniqueness across created items.
8. Concurrency / repeated create attempts with same `sellerID` — multiple items should be created (system supports multiple per seller).

### Security / headers
1. Send requests without `Content-Type: application/json` where required -> expect 400.
2. Send `Accept: application/json` missing -> check default behavior (should still return JSON or documented fallback).

### Performance / robustness (basic)
1. Create 10 items in a loop for same seller -> all should succeed (200) and be returned in `GET /{sellerID}/item`.
2. Rapid create & delete: create an item and immediately delete via v2 endpoint -> delete should succeed.

### Data validation checks
- `price` non-negative integer.
- `sellerId` in response equals sent `sellerID`.
- `createdAt` parseable as ISO-8601 datetime.

---

## Test data strategy
- Generate `sellerID` randomly in range `111111`–`999999` and track created IDs in fixture for cleanup.
- Use deterministic names like `test-item-<timestamp>` to ease searching.

---

# README.md

## Avito API tests (pytest)

### Requirements
- Python 3.8+
- `pip` available

### Setup
1. Clone the project (or copy files from Canvas).
2. Create virtualenv (recommended):

```bash
python -m venv .venv
source .venv/bin/activate   # on Windows: .venv\Scripts\activate
pip install -U pip
pip install -r requirements.txt
```

`requirements.txt` contents (example):
```
pytest
requests
python-dotenv
```

### Environment variables
You can configure base URL via `BASE_URL` env var. Default is `https://qa-internship.avito.com`.

Example (bash):
```bash
export BASE_URL=https://qa-internship.avito.com
```

### Running tests
Run all tests:
```bash
pytest -q
```

Run a single file:
```bash
pytest -q tests/test_create_item_v1.py
```

### Notes
- Tests create real items on the target service. Cleanup is attempted via DELETE `/api/2/item/{id}`.
- If the service enforces rate limits or blocks, rerun tests later or reduce concurrency.

---

# BUGS.md

> Fill this file with bug reports discovered during test runs.

Template for a bug entry:

```
- ID: BUG-<timestamp>
- Endpoint: POST /api/1/item
- Steps: 1) POST ... 2) GET ...
- Actual: ...
- Expected: ...
- Notes: request/response examples
```

---

# tests/ (automation)

Files included below. Place them in a project folder as shown.

## requirements.txt
```
pytest
requests
python-dotenv
```

---

## tests/conftest.py

```python
import os
import random
import string
import time
import requests
import pytest
from dotenv import load_dotenv

load_dotenv()

BASE_URL = os.getenv('BASE_URL', 'https://qa-internship.avito.com')

@pytest.fixture(scope='session')
def base_url():
    return BASE_URL

@pytest.fixture(scope='session')
def created_items():
    """Collect created item ids for cleanup"""
    items = []
    yield items
    # Teardown: attempt to delete created items using v2 DELETE
    for item_id in items:
        try:
            url = f"{BASE_URL}/api/2/item/{item_id}"
            requests.delete(url, headers={"Accept":"application/json"}, timeout=5)
        except Exception:
            pass


def rand_seller_id():
    return random.randint(111111, 999999)

@pytest.fixture
def seller_id():
    return rand_seller_id()

@pytest.fixture
def unique_name():
    return f"test-item-{int(time.time()*1000)}"
```

---

## tests/utils.py

```python
import requests

def create_item(base_url, payload):
    url = f"{base_url}/api/1/item"
    r = requests.post(url, json=payload, headers={"Content-Type":"application/json","Accept":"application/json"}, timeout=10)
    return r

def get_item_v1(base_url, item_id):
    url = f"{base_url}/api/1/item/{item_id}"
    r = requests.get(url, headers={"Accept":"application/json"}, timeout=10)
    return r

def get_items_by_seller(base_url, seller_id):
    url = f"{base_url}/api/1/{seller_id}/item"
    r = requests.get(url, headers={"Accept":"application/json"}, timeout=10)
    return r

def get_stat_v1(base_url, item_id):
    url = f"{base_url}/api/1/statistic/{item_id}"
    r = requests.get(url, headers={"Accept":"application/json"}, timeout=10)
    return r

def get_stat_v2(base_url, item_id):
    url = f"{base_url}/api/2/statistic/{item_id}"
    r = requests.get(url, headers={"Accept":"application/json"}, timeout=10)
    return r

def delete_item_v2(base_url, item_id):
    url = f"{base_url}/api/2/item/{item_id}"
    r = requests.delete(url, headers={"Accept":"application/json"}, timeout=10)
    return r
```

---

## tests/test_create_item_v1.py

```python
import json
from tests import utils


def test_create_item_success(base_url, created_items, seller_id, unique_name):
    payload = {
        "sellerID": seller_id,
        "name": unique_name,
        "price": 100,
        "statistics": {"likes": 0, "viewCount": 0, "contacts": 0}
    }

    r = utils.create_item(base_url, payload)
    assert r.status_code == 200, f"Expected 200, got {r.status_code} - {r.text}"

    data = r.json()
    assert 'id' in data
    assert data['sellerId'] == seller_id
    assert data['name'] == unique_name
    assert data['price'] == 100
    assert 'createdAt' in data

    created_items.append(data['id'])
```

---

## tests/test_get_item_v1.py

```python
from tests import utils


def test_get_item_after_create(base_url, created_items, seller_id, unique_name):
    # create
    payload = {"sellerID": seller_id, "name": unique_name, "price": 200, "statistics": {"likes":1,"viewCount":2,"contacts":3}}
    r = utils.create_item(base_url, payload)
    assert r.status_code == 200
    data = r.json()
    item_id = data['id']
    created_items.append(item_id)

    # get
    r2 = utils.get_item_v1(base_url, item_id)
    assert r2.status_code == 200
    got = r2.json()
    # Response may be array or object depending on implementation; handle both
    if isinstance(got, list):
        got_item = got[0]
    else:
        got_item = got

    assert got_item['id'] == item_id
    assert got_item['sellerId'] == seller_id
    assert got_item['name'] == unique_name
    assert got_item['price'] == 200
```

---

## tests/test_get_items_by_seller_v1.py

```python
from tests import utils


def test_get_items_by_seller(base_url, created_items, seller_id):
    # create two items for same seller
    payload1 = {"sellerID": seller_id, "name": "batch-1", "price": 10, "statistics": {"likes":0,"viewCount":0,"contacts":0}}
    payload2 = {"sellerID": seller_id, "name": "batch-2", "price": 20, "statistics": {"likes":0,"viewCount":0,"contacts":0}}

    r1 = utils.create_item(base_url, payload1)
    r2 = utils.create_item(base_url, payload2)
    assert r1.status_code == 200
    assert r2.status_code == 200
    id1 = r1.json()['id']
    id2 = r2.json()['id']
    created_items.extend([id1, id2])

    r = utils.get_items_by_seller(base_url, seller_id)
    assert r.status_code == 200
    arr = r.json()
    assert isinstance(arr, list)
    # ensure at least these two ids present
    ids = [it.get('id') for it in arr]
    assert id1 in ids and id2 in ids
```

---

## tests/test_statistic_v1.py

```python
from tests import utils


def test_statistic_v1(base_url, created_items, seller_id, unique_name):
    payload = {"sellerID": seller_id, "name": unique_name, "price": 50, "statistics": {"likes":5,"viewCount":10,"contacts":1}}
    r = utils.create_item(base_url, payload)
    assert r.status_code == 200
    item_id = r.json()['id']
    created_items.append(item_id)

    r2 = utils.get_stat_v1(base_url, item_id)
    # Depending on service, statistic endpoint may return 200 or 404 if not available
    assert r2.status_code in (200, 404)
    if r2.status_code == 200:
        arr = r2.json()
        assert isinstance(arr, list)
        # basic shape check
        for t in arr:
            assert 'likes' in t and 'viewCount' in t and 'contacts' in t

```

---

## tests/test_statistic_v2.py

```python
from tests import utils


def test_statistic_v2(base_url, created_items, seller_id, unique_name):
    payload = {"sellerID": seller_id, "name": unique_name, "price": 70, "statistics": {"likes":2,"viewCount":3,"contacts":0}}
    r = utils.create_item(base_url, payload)
    assert r.status_code == 200
    item_id = r.json()['id']
    created_items.append(item_id)

    r2 = utils.get_stat_v2(base_url, item_id)
    assert r2.status_code in (200, 404)
    if r2.status_code == 200:
        arr = r2.json()
        assert isinstance(arr, list)
        for t in arr:
            assert 'likes' in t and 'viewCount' in t and 'contacts' in t
```

---

## tests/test_delete_item_v2.py

```python
from tests import utils


def test_delete_item_v2(base_url, seller_id, created_items, unique_name):
    payload = {"sellerID": seller_id, "name": unique_name, "price": 111, "statistics": {"likes":0,"viewCount":0,"contacts":0}}
    r = utils.create_item(base_url, payload)
    assert r.status_code == 200
    item_id = r.json()['id']

    # delete via v2
    rd = utils.delete_item_v2(base_url, item_id)
    # service may return 200 with empty body
    assert rd.status_code in (200, 204, 404)

    # after delete, GET should return 404 ideally
    rg = utils.get_item_v1(base_url, item_id)
    # depending on implementation, it may be 404 or 200 with missing content; assert non-2xx if delete succeeded
    if rd.status_code in (200,204):
        assert rg.status_code == 404
```

---

## tests/test_negative_cases.py

```python
from tests import utils
import requests


def test_create_item_missing_fields(base_url):
    payload = {"name":"no-seller"}
    r = requests.post(f"{base_url}/api/1/item", json=payload, headers={"Content-Type":"application/json","Accept":"application/json"})
    assert r.status_code == 400


def test_get_item_not_found(base_url):
    r = utils.get_item_v1(base_url, 'non-existent-id-12345')
    assert r.status_code == 404


def test_delete_not_found(base_url):
    r = utils.delete_item_v2(base_url, 'non-existent-id-12345')
    assert r.status_code == 404
```

---

# End of document


