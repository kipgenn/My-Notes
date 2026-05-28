# 🕸️ Data Extraction Masterclass — Master Reference Notes

> **Purpose:** Go-to reference for web scraping & API data extraction pipelines.
> **Stack:** `requests`, `BeautifulSoup`, `pandas`
> **Style:** Standard `for` loops (no list comprehensions)

---

## 🗺️ Full Pipeline at a Glance

```
[Source: API or Website]
        ↓
Phase 1: Data Acquisition      → raw JSON dict / HTML text
        ↓
Phase 1b: Web Scraping         → (only if no API exists)
        ↓
Phase 2: Parsing & Filtering   → clean list of dicts
        ↓
Phase 3: Structuring           → Pandas DataFrame
        ↓
Phase 4: Persistent Storage    → .csv file on disk
```

---

## 📦 Master Imports

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
```

---

## Phase 1: Data Acquisition via REST API

**When to use:** A public or private API endpoint exists that returns structured JSON.

```python
import requests

url = "https://api.example.com/v1/users"

try:
    response = requests.get(url, timeout=10)  # 1. Send network request
    response.raise_for_status()               # 2. Raise error on 4xx/5xx
    data = response.json()                    # 3. Convert JSON → Python dict/list
except requests.exceptions.RequestException as err:
    print(f"Network error: {err}")
```

### ⚠️ Critical Rules
| Rule | Reason |
|---|---|
| Always pass `timeout=10` | Without it, script hangs **forever** if server is unresponsive |
| Always call `.raise_for_status()` | Silent 404s/500s will not raise exceptions on their own |
| Use `try/except RequestException` | Catches all network errors: timeouts, DNS failures, connection refused |

### 🔑 Common HTTP Status Codes
| Code | Meaning |
|---|---|
| `200` | OK — success |
| `400` | Bad Request — your query is malformed |
| `401` | Unauthorized — missing or bad API key |
| `403` | Forbidden — you don't have permission |
| `404` | Not Found — wrong endpoint URL |
| `429` | Too Many Requests — you're being rate-limited |
| `500` | Server Error — their problem, not yours |

### 🔐 Sending API Keys (Authentication)

```python
# Option A: Header-based (most common)
headers = {"Authorization": "Bearer YOUR_API_KEY"}
response = requests.get(url, headers=headers, timeout=10)

# Option B: Query parameter
params = {"api_key": "YOUR_API_KEY", "page": 1}
response = requests.get(url, params=params, timeout=10)
```

### 📄 Handling Pagination

```python
all_data = []
page = 1

while True:
    params = {"page": page, "per_page": 100}
    response = requests.get(url, params=params, timeout=10)
    response.raise_for_status()
    batch = response.json()

    if not batch:        # Empty list = no more pages
        break

    for item in batch:
        all_data.append(item)

    page += 1
```
### Json with nested infos

```python
df = DataFrame("json file") # cannot handle nested json
df = json_normalize("json file") # can handle nested json
```


---

## Phase 1 (Alt): Data Acquisition via Web Scraping

**When to use:** No API exists. You download raw HTML and parse it structurally.

```python
import requests
from bs4 import BeautifulSoup

url = "https://example.com/articles"
response = requests.get(url, timeout=10)
response.raise_for_status()

# 1. Parse the HTML structure into memory
soup = BeautifulSoup(response.text, "html.parser")

# 2. Find all <h2> tags with class="title"
article_tags = soup.find_all("h2", class_="title")

# 3. Extract clean text using a standard for loop
clean_titles = []
for tag in article_tags:
    clean_text = tag.text.strip()   # Remove HTML tags + extra whitespace
    clean_titles.append(clean_text)
```

### ⚠️ Critical Rules
| Rule | Reason |
|---|---|
| `find()` → returns **one** item or `None` | Use when you expect a single element |
| `find_all()` → returns a **list** | Always requires a `for` loop to extract values |
| Always `.strip()` after `.text` | Raw `.text` often has leading/trailing whitespace and `\n` |

### 🔍 BeautifulSoup Selector Cheatsheet

```python
# By tag
soup.find("h1")
soup.find_all("p")

# By class
soup.find("div", class_="product-card")

# By id
soup.find("div", id="main-content")

# By attribute
soup.find("a", href=True)
soup.find_all("img", src=True)

# Nested navigation
card = soup.find("div", class_="card")
title = card.find("h2").text.strip()
link = card.find("a")["href"]          # Extract attribute value

# CSS selector style (alternative)
items = soup.select("ul.menu > li")
```

### 🕐 Scraping Etiquette

```python
import time

for url in url_list:
    response = requests.get(url, timeout=10)
    # ... process ...
    time.sleep(1)   # Wait 1 second between requests — don't hammer servers
```

---

## Phase 2: Data Parsing & Filtering

**When to use:** Raw data from the API/scrape needs cleaning — remove bad rows, isolate only needed columns.

```python
# Assume 'data' is a list of dicts from the API
active_users = []

for user in data:
    # 1. Filter: skip inactive users
    if user.get("status") == "active":

        # 2. Isolate only required fields into a new clean dict
        clean_user = {
            "Name":  user.get("name"),
            "Email": user.get("email")
        }
        active_users.append(clean_user)
```

### ⚠️ Critical Rules
| Rule | Reason |
|---|---|
| Use `dict.get("key")` not `dict["key"]` | `.get()` returns `None` on missing keys; `[]` raises a `KeyError` and crashes |
| Build a new dict with only needed keys | Keeps DataFrame lean; avoids importing junk columns |

### 🔧 Useful Filtering Patterns

```python
# Filter by numeric threshold
for item in data:
    if item.get("age", 0) >= 18:
        clean_data.append(item)

# Filter out None values
for item in data:
    if item.get("email") is not None:
        clean_data.append(item)

# Normalize strings during parse (e.g., lowercase, strip)
for item in data:
    clean_item = {
        "name":  item.get("name", "").strip().title(),
        "email": item.get("email", "").strip().lower()
    }
    clean_data.append(clean_item)
```

---

## Phase 3: Data Structuring (Pandas DataFrame)

**When to use:** Cleaned list of dicts needs to be converted into a 2D table for analysis or export.

```python
import pandas as pd

# Convert list of dicts → DataFrame
df = pd.DataFrame(active_users)

# Verify
print(df.head())        # First 5 rows
print(df.shape)         # (rows, columns)
print(df.dtypes)        # Column data types
print(df.isnull().sum()) # Count missing values per column
```

### 🔧 Common DataFrame Operations

```python
# Rename columns
df.rename(columns={"Name": "full_name", "Email": "email_address"}, inplace=True)

# Drop duplicates
df.drop_duplicates(subset=["email"], inplace=True)

# Drop rows with any missing values
df.dropna(inplace=True)

# Reset index after filtering
df.reset_index(drop=True, inplace=True)

# Filter DataFrame rows
df_filtered = df[df["status"] == "active"]

# Add a new column
df["domain"] = df["email"].str.split("@").str[1]
```

---

## Phase 4: Persistent Storage (CSV)

**When to use:** Save the DataFrame from volatile RAM to permanent disk storage.

```python
filename = "extracted_data.csv"

df.to_csv(filename, index=False)
```

### ⚠️ Critical Rules
| Rule | Reason |
|---|---|
| Always pass `index=False` | Without it, Pandas writes its internal row numbers (0, 1, 2…) as a junk column in your file |

### 🗄️ Other Export Formats

```python
# Excel
df.to_excel("output.xlsx", index=False)

# JSON
df.to_json("output.json", orient="records", indent=2)

# Read back from CSV
df = pd.read_csv("extracted_data.csv")
```

---

## 🔁 Full End-to-End Template

Copy-paste ready boilerplate for a new scraping/extraction job:

```python
import requests
from bs4 import BeautifulSoup
import pandas as pd
import time

# ─── CONFIG ───────────────────────────────────────────────
BASE_URL  = "https://example.com/data"
HEADERS   = {"User-Agent": "Mozilla/5.0"}   # Polite scraping header
OUTPUT    = "output.csv"

# ─── PHASE 1: ACQUIRE ─────────────────────────────────────
try:
    response = requests.get(BASE_URL, headers=HEADERS, timeout=10)
    response.raise_for_status()
    raw_data = response.json()              # or response.text for HTML
except requests.exceptions.RequestException as err:
    print(f"Network error: {err}")
    raw_data = []

# ─── PHASE 2: PARSE & FILTER ──────────────────────────────
clean_data = []

for item in raw_data:
    if item.get("status") == "active":
        clean_item = {
            "Name":  item.get("name", "").strip(),
            "Email": item.get("email", "").strip().lower()
        }
        clean_data.append(clean_item)

# ─── PHASE 3: STRUCTURE ───────────────────────────────────
df = pd.DataFrame(clean_data)
df.drop_duplicates(subset=["Email"], inplace=True)
df.reset_index(drop=True, inplace=True)
print(df.head())

# ─── PHASE 4: SAVE ────────────────────────────────────────
df.to_csv(OUTPUT, index=False)
print(f"Saved {len(df)} records to {OUTPUT}")
```

---

## 🚨 Common Errors & Fixes

| Error | Cause | Fix |
|---|---|---|
| `KeyError: 'status'` | Key missing in dict | Use `dict.get("status")` |
| `JSONDecodeError` | Response isn't valid JSON | Check `response.text` first; check status code |
| `ConnectionTimeout` | Server unresponsive | Always use `timeout=10` |
| `AttributeError: 'NoneType'` | `find()` returned `None` | Check tag/class name; use `if tag:` before `.text` |
| Junk index column in CSV | Forgot `index=False` | Always `df.to_csv(file, index=False)` |
| Script gets blocked/banned | Too many rapid requests | Add `time.sleep(1)` between requests |
| `403 Forbidden` on scrape | Missing User-Agent | Pass `headers={"User-Agent": "Mozilla/5.0"}` |

---

*Last updated: 2026 — Data Extraction Masterclass*
