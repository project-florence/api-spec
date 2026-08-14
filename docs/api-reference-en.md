# Florence API Reference (English)

> **Source:** the Florence backend repo — the live FastAPI application (version: `src/version.py`).
> **Generated:** 2026-08-14 — every endpoint was verified at code level against the `src/api/` router files; schemas are taken verbatim from the Pydantic models.
> **Matching Turkish document:** [`api-reference-tr.md`](./api-reference-tr.md) (identical scope).

---

## 1. Overview

Florence is the API server of a Turkish stock exchange (BIST) focused financial platform:

- **Market data:** stock profiles and candle history via yfinance, the BIST company list via pykap, FX/gold/commodity spot prices via GenelPara, macroeconomic indicators via FRED.
- **News analysis:** stock news via SearXNG + GDELT/BigQuery; LLM-agent generated investment reports (`quick_report` / `deep_report`).
- **Simulation:** Monte Carlo price simulation (probability above/below a target price, confidence interval).
- **Virtual portfolio:** JSONB-based paper trading with concurrency protection via `pg_advisory_xact_lock` (transactions, valuation, risk, benchmark, etc.).
- **User management:** JWT auth (access + refresh rotation), email verification, Argon2 password hashing, a credit system (free/gift credits), bot accounts, announcements, and takeout-style data export.

**Architecture:** fully async — FastAPI 0.139 + psycopg3 `AsyncConnectionPool` + redis.asyncio. The HTTP layer lives in `src/api/`, business logic in `src/services/`, external integrations in `src/clients/`, shared infrastructure in `src/core/`. There is no ORM; raw SQL + Pydantic are used.

**Base URL:** `http://localhost:7055` (development default; in production `PUBLIC_BASE_URL`). All feature endpoints live under the **`/api/v1`** prefix.

**Docs:** `/docs` (Swagger UI), `/redoc`, `/openapi.json` — enabled only outside production (`ENVIRONMENT != production`).

---

## 2. Authentication Mechanism

### 2.1 Access token (JWT)

- Algorithm: **HS256**, signing key: `SECRET_KEY` (env, required — the app refuses to start without it).
- Payload: `{ "user_id": int, "iat": ..., "exp": ... }` — lifetime **1 hour**.
- Delivery: `Authorization: Bearer <token>` header **or** the `access_token` httpOnly cookie (`samesite=strict`, `secure` in production, max-age 3600, path `/`).
- Validation: `src/api/deps.py:get_current_user` — reads from header or cookie, decodes the JWT, and applies additional checks (below).

### 2.2 Extra token checks (deps.py)

| Check | Mechanism |
|---|---|
| Password change | If `password_changed_at > token.iat` the token is considered invalid (Redis `user:pwd_changed:*` cache, 60s TTL). |
| Frozen account | `is_frozen` users are denied even with a valid token (Redis `user:frozen:*` cache, 30s TTL). |
| Email verification | Checked at login/refresh; unverified accounts cannot sign in (bots exempt). |

### 2.3 Refresh token

- Generation: `secrets.token_urlsafe(48)`; stored as **SHA-256 hash** in the `refresh_tokens` table.
- **Rotation:** every `/auth/refresh` call revokes the old token and issues a new one.
- TTL: 30 days (`REFRESH_TOKEN_TTL_DAYS`).
- Delivery: request body `{"refresh_token": "..."}` **or** the `refresh_token` cookie (path `/api/v1/auth`).
- Revocation: `/auth/logout` (single token); password/email/username change (all user tokens).

### 2.4 Middleware (main.py: `auth_and_tracking_middleware`)

- For every request under `/api/v1/*`: if the path is **not** in the public list, the JWT is validated; on failure → `401 {"detail": "Not authenticated"}`.
- Public paths: `/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/logout`, `/auth/verify-email`, `/auth/resend-verification`, `/market/status`, `/data/export/download`, `/meta/avatars`, `/legal`, `/about`, `/contact`, `/version`, `/maintenance`, `/contributors`, `/`, `/health`, `/docs`, `/redoc`, `/openapi.json`. (Prefix matching applies: public paths starting with `/api/` also match sub-paths.)
- `OPTIONS` (CORS preflight) is exempt.
- DB pool exhaustion (`PoolTimeout`): `503 {"detail": "Database busy, please retry"}` (not 401 — so the frontend does not enter the refresh/logout loop).
- `db.release_current()` is called after every request (connection leak prevention).
- Analytics: all private requests except `/api/v1/analytics/event` are recorded fire-and-forget as `api_request` events.
- CORS: in production only Tauri desktop origins (`tauri://localhost`, `http://tauri.localhost`, `https://tauri.localhost`); `*` in development.

### 2.5 Admin identity

- Separate `X-Admin-Token` header (fixed `ADMIN_TOKEN` env, string equality). See `src/api/deps.py:verify_admin_token`.
- Admin users get a **10x** rate-limit multiplier (`get_current_user_full` reads `user_type`; applied on the news and export endpoints).

---

## 3. Authentication Flows (Step by Step)

### Flow A — Classic registration → login

1. **Register:** `POST /api/v1/auth/register` — `{username, email, password(≥10)}` → response `{message, user_id, verification_sent}`. The password is Argon2-hashed; a 24-hour email verification token is created and mailed (mail failure never breaks registration: `verification_sent: false`). Error codes: `error_username_taken`, `error_email_taken` (400).
2. **Verify email:** `GET /api/v1/auth/verify-email?token=...` — if the token matches and is not expired, `email_verified = TRUE`. Response: `{message: "Email verified", email_verified: true}`. Invalid/expired: 400 `"Invalid or expired verification token"`.
3. **Login:** `POST /api/v1/auth/login` — OAuth2 form (`application/x-www-form-urlencoded`, fields: `username`, `password`; username **or** email accepted). Argon2 verification; unverified email → 403 `error_email_not_verified`. On success:
   - Body: `{"access_token", "refresh_token", "token_type": "bearer"}`
   - Additionally sets httpOnly cookies: `access_token` (path `/`, 1h) and `refresh_token` (path `/api/v1/auth`, 30d).
   - `last_login` is updated.
4. **Refresh:** `POST /api/v1/auth/refresh` — body `{"refresh_token"}` or cookie. Rotation is performed; a new access+refresh pair is returned (cookies are refreshed too). Unverified email → 403. Invalid token → 401 `"Invalid or expired refresh token"`.
5. **Logout:** `POST /api/v1/auth/logout` — body `{"refresh_token"}` or cookie; the token is revoked and cookies cleared. `{message: "Logged out"}`.

### Flow B — Resend verification email

- `POST /api/v1/auth/resend-verification` — `{username_or_email}` (username **or** email). Account not found → 404; already verified → 400 `"Email already verified"`; a new token (24h) is generated and mailed. Response: `{verification_sent: bool}`.

### Flow C — Account changes (JWT required)

- `PUT /api/v1/auth/change-password` — `{current_password, new_password(≥10)}` → password changes, `password_changed_at` updated, **all** refresh tokens revoked. Wrong current password → 400.
- `PUT /api/v1/auth/change-email` — `{new_email, current_password}` → email changes, all tokens revoked.
- `PUT /api/v1/auth/change-username` — `{new_username, current_password}` → username changes, all tokens revoked.
- `DELETE /api/v1/auth/delete` — permanently deletes the account.

### Flow D — Bot account (for API access)

- `POST /api/v1/bots` — `{username, password?(≥10)}` → `user_type='bot'`, email `<username>@bot.florencex.com.tr`, `email_verified=TRUE` (no verification needed). **The password is returned only once in the response.** Bots spend the owner's credits (`credits._resolve_owner`). Limit: 5 bots per user. Bots cannot create bots (403 `error_bots_not_allowed`).

---

## 4. Rate Limiting Rules

Implementation: `src/core/ratelimit.py` — **hybrid model**: with Redis, `INCR + EXPIRE` (fixed window); without Redis, an in-process memory bucket (same limit, down-tolerant). Exceeded → `429 {"detail": "Too many requests. Please slow down."}`.

| Endpoint | Key | Limit | Admin |
|---|---|---|---|
| `POST /auth/register` | `register:{client_ip}` | 3 req / 60 s | — |
| `POST /auth/login` | `login:{username}` | 5 req / 60 s | — |
| `POST /auth/refresh` | `refresh:{token_hash}` | 5 req / 60 s | — |
| `POST /auth/resend-verification` | `resend-verif:{account}` | 3 req / 3600 s | — |
| `GET /news/{ticker}` | `news:{user_id}:{ticker}` | 10 req / 60 s | 100 / min (10x) |
| `POST /data/export` | `export:{user_id}` | 3 req / 3600 s | 30 / hour (10x) |

There is **no** global limit applied to every endpoint. Additionally, the job-slot mechanism (`src/core/job_slots.py`) prevents the same user from running the same job type concurrently:

| Job | Slot key | TTL / lease |
|---|---|---|
| Report generation | `job-slot:report:{user_id}` | 900 s (lease 120 s, heartbeat 30 s) |
| Simulation | `job-slot:simulation:{user_id}` | 600 s (lease 120 s, heartbeat 30 s) |

Slot busy → `429 {"detail": "A job of this type is already running"}`.

---

## 5. Error Format

All errors use the FastAPI standard shape: `{"detail": <string | list>}`.

| Status | Meaning | Example `detail` |
|---|---|---|
| 400 | Invalid request / business rule | `"Invalid type"`, `"error_username_taken"`, `"Market is closed"` |
| 401 | Authentication failure | `"Invalid or expired token"`, `"Not authenticated"` (middleware), `"Invalid or expired refresh token"` |
| 402 | Insufficient credits | `"insufficient credit"` (reports/simulations) |
| 403 | Permission / account state | `"error_email_not_verified"`, `"Admin access required"`, `"Invalid admin token"` |
| 404 | Resource not found | `"Invalid BIST ticker: X"`, `"Report not found"`, `"User not found"` |
| 409 | Conflict (uniqueness) | `"error_username_taken"` (bot creation) |
| 410 | Permanently removed / expired | `GET /data/daily/{year}` — `"Kullanım dışı — POST /api/v1/data/export ile istek oluşturun"`; expired export link `"Export link expired or not ready"` |
| 413 | Too many items | `"Too many events"` (analytics, >100) |
| 422 | Pydantic validation | Standard FastAPI `HTTPValidationError` |
| 429 | Rate limit / job slot | `"Too many requests. Please slow down."`, `"A job of this type is already running"` |
| 500 | Server error | `"Database error"`, `"Report generation failed"` |
| 503 | Maintenance / pool exhaustion | `"Database busy, please retry"`, `"{feature} is temporarily disabled for maintenance"` |

**i18n error codes:** some errors return short codes for the frontend to translate: `error_username_taken`, `error_email_taken`, `error_login_failed`, `error_email_not_verified`, `error_bot_limit_reached`, `error_bots_not_allowed`.

---

## 6. Data Models / Entities

Schema source: `src/core/database.py:init_db()` (created idempotently at runtime via `CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ... ADD COLUMN IF NOT EXISTS`; the `migrations/` files are **not** applied automatically).

| Table | Key columns |
|---|---|
| `users` | `id`, `username` UNIQUE, `email` UNIQUE, `hashed_pw`, `user_type` (`user`/`bot`/`admin`), `is_frozen`, `email_verified`, `email_verify_token`(+`email_verify_expires_at`), `avatar_id`, `owner_id` (bot→owner), `password_changed_at`, `last_login`, `last_announcement_viewed_at` |
| `user_credits` | `(user_id, credit_type)` PK — `credit_type`: `free_credits` / `gift_credits`; `amount` |
| `tickers` / `companies` | BIST codes + company profiles (populated by pykap) |
| `ticker_stats` | `ticker` PK; `info_count`, `report_count`, `news_count`, `history_count`, `simulation_count`, `favorite_count`, `updated_at` |
| `favorites` | `(user_id, ticker_code)` PK |
| `reports` | `id`, `user_id`, `ticker`, `type` (`quick_report`/`deep_report`), `title`, `token_usage` JSONB, `content` TEXT, `sentiments` JSONB, `purpose`, `created_at` |
| `simulations` | `id`, `user_id`, `ticker`, `days`, `bounds`, `target`, `result` JSONB, `cost`, `created_at` |
| `price_candles` | `(ticker, interval, ts)` PK; `open/high/low/close` DOUBLE, `volume` BIGINT; `idx_price_candles_lookup` (ticker, interval, ts DESC). Ticker format `XXX.IS` |
| `economy_rates` | `(ticker, ts)` PK, `price` JSONB (FX/gold/commodity history) |
| `market_rates` | `data_type` + `data` JSONB (latest market snapshot fallback) |
| `macroeconomy` | 14 FRED indicators (gdp, fed_funds, vix, sp500, btc, etc.) |
| `portfolios` | `portfolio_id` TEXT PK (`port-<uuid>`), `user_id`, `portfolio` JSONB (whole portfolio as one JSON) |
| `announcements` | `id`, `title`, `content`, `sent_by`, `created_at`, `updated_at` |
| `cron_jobs` | `name` PK, `source`, `interval_ms`, `last_run` |
| `refresh_tokens` | `token_hash` UNIQUE (SHA-256), `user_id`, `device`, `expires_at`, `revoked_at` |
| `analytics_events` | `event_type`, `user_id`, `session_id`, `ticker`, `details` JSONB |
| `stock_vectors` | `ticker` PK, `risk`, `horizon`, `profitability` |
| `token_usage` | `model`, `prompt_tokens`, `completion_tokens`, `total_tokens`, `endpoint`, `user_id`, `created_at` |
| `user_preferences` | `user_id` PK, `prefs` JSONB (trigger `create_default_prefs()` creates default prefs on insert) |
| `exports` | `year`, `format` (csv/json), `status` (queued/processing/ready/sent/failed), `file_path`, `token`, `expires_at`, `downloaded_count`, `row_count`, `size_bytes`, `error`, `created_at`, `updated_at` |

**Note:** if a legacy `users.credits` column exists it is migrated into `user_credits` and dropped (data-migration inside `init_db`). If `economy_rates.price` was created as DOUBLE PRECISION in old environments it is converted to JSONB (self-heal).

### Pydantic request schemas (openapi.json `components.schemas`)

| Schema | Fields |
|---|---|
| `UserRegister` | `username: str`, `email: EmailStr`, `password: str (minLength 10)` — all required |
| `ChangePassword` | `current_password: str`, `new_password: str (min 10)` |
| `UpdateEmail` | `new_email: EmailStr`, `current_password: str` |
| `UpdateUsername` | `new_username: str`, `current_password: str` |
| `RefreshRequest` | `refresh_token: str \| null` (optional — cookie fallback) |
| `ResendVerification` | `username_or_email: str` |
| `AvatarUpdate` | `avatar_id: str` (`avatar-1` … `avatar-12`) |
| `PreferencesUpdate` | `prefs: dict` (JSONB, merged) |
| `BotCreate` | `username: str (3-255)`, `password: str\|null (min 10)` |
| `FitRequest` | `horizon: str` (short/medium/long, default `long`), `profitability` (low/medium/high, default `high`), `risk_tolerance` (low/medium/high, default `medium`), `limit: int (1-100, default 5)` |
| `PortfolioProfileRequest` | `tickers: list[str] (1-50, uppercased)`, `limit: int (1-50, default 5)` |
| `CreatePortfolioBody` | `name: str (1-255)`, `initial_balance: float (>0)` |
| `RenamePortfolioBody` / `DuplicatePortfolioBody` | `name: str (1-255)` |
| `AddTransactionBody` | `ticker: str (min 1)`, `type: str (pattern ^(BUY\|SELL)$)`, `quantity: float (>0)` |
| `UpdateTransactionBody` | `price: float\|null (>0)`, `quantity: float\|null (>0)` — both optional |
| `AnnouncementCreate` / `AnnouncementUpdate` | `title: str`, `content: str` |
| `ExportRequest` | `year: int (1990..current year+1)`, `format: "csv"\|"json"` (default csv) |
| `ReportHistoryItem` (response) | `id: int`, `ticker: str`, `type: str`, `title: str\|null`, `token_usage: dict\|null`, `purpose: str\|null`, `created_at: str` |

---

## 7. Cache / TTL Rules

The Redis proxy (`src/core/redis.py`) is **down-tolerant**: if the connection cannot be established all calls return `None` and it retries after a 60s cooldown (cache-less mode — falls back to DB).

| Data | Redis key | TTL |
|---|---|---|
| BIST company/ticker list | `tickers`, `companies` JSON | 30 days |
| Price history | `price:history:{ticker}:{period}:{interval}` (hot: `cache_ttl_hot`) | 60 s normal; 7 days hot; 30 s intraday |
| Company profile | `{TICKER}.IS` | 24 h |
| Economy (GenelPara) | `gold_prices`, `silver_price`, `gram_platinum_price`, `gram_palladium_price`, `currency`, `genelpara:*` | 20 min (1200 s); 60 s on error |
| Macro (FRED) | `macroeconomy` | 24 h |
| IPO lists | `ipos:*` | 1 h |
| Market status | `market:status` | 60 s |
| News | `news:{TICKER}:{amount}` (rounded to 10) | 1 h |
| Stock vectors | `stock_vectors:*` | 24 h |
| Frozen user | `user:frozen:{user_id}` | 30 s |
| Password change | `user:pwd_changed:{user_id}` | 60 s |
| Rate limit counters | `ratelimit:{key}` | window duration |
| Locks | `lock:cron:{tier}` (NX+TTL), `refresh_lock:{ticker}:{interval}` (15 s) | variable |
| Job slot lease | `job-slot:{kind}:{user_id}` | 120 s (heartbeat 30 s) |
| Sets | `delisted_tickers`, `maintenance:disabled` | permanent |

---

## 8. Endpoint Reference

> Legend: (public) = public (no auth), (JWT) = JWT required, (feature) = subject to feature kill-switch, (slot) = subject to job slot.
> All paths are under the `/api/v1` prefix. Auth: `Authorization: Bearer <access_token>` header or the `access_token` cookie.

### 8.1 Auth & User (`src/api/auth.py`)

#### `POST /auth/register` (public)
User registration. Generates a 24-hour email verification token and sends the mail.

- **Body:** `{"username": str, "email": str, "password": str (min 10)}`
- **Rate limit:** 3/min/IP
- **Response 200:**
```json
{ "message": "Register successful", "user_id": 42, "verification_sent": true }
```
- **Errors:** 400 `"error_username_taken"` / `"error_email_taken"`; 500 `"Database error"`

#### `POST /auth/login` (public)
OAuth2 form login — `application/x-www-form-urlencoded`, fields: `username` (username **or** email), `password`.

- **Rate limit:** 5/min/username
- **Response 200:** body + httpOnly cookies (`access_token` path `/`, `refresh_token` path `/api/v1/auth`)
```json
{ "access_token": "<jwt>", "refresh_token": "<opaque>", "token_type": "bearer" }
```
- **Errors:** 400 `"error_login_failed"`; 403 `"error_email_not_verified"` (unverified account, bots exempt)

#### `GET /auth/verify-email` (public)
Email verification.

- **Query:** `token: str` (required)
- **Response 200:** `{"message": "Email verified", "email_verified": true}`
- **Errors:** 400 `"Invalid or expired verification token"`

#### `POST /auth/resend-verification` (public)
Resends the verification email.

- **Body:** `{"username_or_email": str}`
- **Rate limit:** 3/hour/account
- **Response 200:** `{"verification_sent": bool}`
- **Errors:** 404 `"User not found"`; 400 `"Email already verified"`

#### `POST /auth/refresh` (public)
Refresh token rotation + new access token.

- **Body (optional):** `{"refresh_token": str}` — falls back to the cookie
- **Rate limit:** 5/min/token hash
- **Response 200:** `{"access_token", "refresh_token", "token_type": "bearer"}` + cookies refreshed
- **Errors:** 401 `"Invalid or expired refresh token"`; 403 `"error_email_not_verified"`

#### `POST /auth/logout` (public)
Revokes the refresh token and clears cookies.

- **Body:** `{"refresh_token": str}` (cookie also accepted)
- **Response 200:** `{"message": "Logged out"}`

#### `DELETE /auth/delete` (JWT)
Permanently deletes the account.

- **Response 200:** `{"message": "Deleted user {id}"}`
- **Errors:** 400 `"Database error"`

#### `PUT /auth/change-password` (JWT)
Changes the password → `password_changed_at` updated, all refresh tokens revoked.

- **Body:** `{"current_password": str, "new_password": str (min 10)}`
- **Response 200:** `{"message": "Password changed successfully"}`
- **Errors:** 404 `"User not found"`; 400 `"Current password is incorrect"`

#### `PUT /auth/change-email` (JWT)
Changes the email → all tokens revoked.

- **Body:** `{"new_email": str, "current_password": str}`
- **Response 200:** `{"message": "Email changed successfully", "new_email": "..."}`
- **Errors:** 400 `"Email already in use"` / `"Current password is incorrect"`

#### `PUT /auth/change-username` (JWT)
Changes the username → all tokens revoked.

- **Body:** `{"new_username": str, "current_password": str}`
- **Response 200:** `{"message": "Username changed successfully", "new_username": "..."}`
- **Errors:** 400 `"Username already in use"`

#### `GET /profile` (JWT)
Profile + credit balance.

- **Response 200:**
```json
{
  "username": "demo", "email": "demo@example.com", "user_type": "user",
  "created_at": "2026-01-01T10:00:00+00:00", "email_verified": true,
  "avatar_id": "avatar-3", "credits": 18.5
}
```

#### `PUT /profile/avatar` (JWT)
Changes the avatar.

- **Body:** `{"avatar_id": "avatar-1"}` … `"avatar-12"`
- **Response 200:** `{"message": "Avatar updated", "avatar_id": "..."}`
- **Errors:** 400 `"Unknown avatar_id"`

#### `GET /credits` (JWT)
Credit balance.

- **Response 200:** `{"credits": 18.5}`

#### `GET /user/preferences` (JWT)
JSONB user preferences.

- **Response 200:** raw JSONB object (e.g. `{"theme": "dark", "language": "tr"}`)
- **Errors:** 404 `"Preferences not found"`

#### `PUT /user/preferences` (JWT)
Updates preferences (**merged** into existing prefs).

- **Body:** `{"prefs": {"theme": "light"}}`
- **Response 200:** the merged JSONB object

### 8.2 BIST / Company / Price (`src/api/bist.py`, `src/api/stats.py`)

#### `GET /bist/companies` (public)
BIST company list (from Redis, 30-day TTL).

- **Query:** `sort: "alphabetical"|"popular"` (default alphabetical), `offset: int ≥0` (default 0), `limit: int 1-500` (default 50)
- **Response 200:** list of company dicts (`ticker`, `name`, …) — for `popular`, offset+limit popular companies are fetched then sliced

#### `GET /bist/tickers` (public)
Ticker list.

- **Query:** same: `sort`, `offset`, `limit`
- **Response 200:** list of ticker strings (popular uses `get_popular_tickers`)

#### `GET /companies/search` (public)
Company text search (aliases supported: "IS BANK" → ISCTR).

- **Query:** `query: str` (required)
- **Response 200:** result of `search_companies_by_text`

#### `GET /companies/info/{ticker}` (public)
Structured company profile based on yfinance. `ticker` must be a valid BIST code (`.IS` appended).

- **Path:** `ticker` (invalid → 404 `"Invalid BIST ticker: X"`)
- **Response 200:** profile dict — `symbol`, `name`, `sector`, `industry`, `currency`, `exchange`, `market{currentPrice, previousClose, marketCap, dayHigh, dayLow, regularMarketVolume, fiftyTwoWeekHigh, fiftyTwoWeekLow, regularMarketTime}`, `trading{beta, sharesOutstanding, floatShares, averageVolume, averageVolume10days, fiftyDayAverage, twoHundredDayAverage, shortRatio, heldPercentInsiders, heldPercentInstitutions}`, `valuation{trailingPE, forwardPE, pegRatio, priceToBook, priceToSalesTrailing12Months, enterpriseValue, enterpriseToEbitda, enterpriseToRevenue, bookValue, trailingEps, forwardEps, dividendYield, payoutRatio, targetMeanPrice, targetHighPrice, targetLowPrice, recommendationKey, numberOfAnalystOpinions}`, `financials{totalRevenue, revenuePerShare, revenueGrowth, grossProfits, grossMargins, ebitda, ebitdaMargins, netIncomeToCommon, profitMargins, operatingMargins, operatingCashflow, freeCashflow, earningsGrowth, earningsQuarterlyGrowth, returnOnEquity, returnOnAssets}`, `balanceSheet{totalCash, totalCashPerShare, totalDebt, debtToEquity, currentRatio, quickRatio}`, `recommendations: []`

#### `GET /companies/info/{ticker}/md` (public)
Same profile as **markdown text** (`text/markdown; charset=utf-8`).

#### `GET /companies/summary` (public)
Summary table.

- **Query:** `limit: int 1-500` (default 50), `offset: int ≥0`, `sort: popular|alphabetical|gainers|losers|price_high|price_low|volume|market_cap` (default popular), `tickers: str|null` (comma-separated filter)
- **Response 200:** summary shaped `{total, data: [{ticker, name, price, change_pct, ...}]}`

#### `GET /news/{ticker}` (JWT) (feature: news)
Stock news (GDELT/BigQuery, last 90 days, Turkish).

- **Path:** `ticker`; **Query:** `amount: int 1-50` (default 10)
- **Rate limit:** 10/min/user+ticker (admin 100/min)
- **Response 200:** list of `[{url, title, lang, date}]` (date ISO8601)

#### `GET /price/history/{ticker}` (public)
Candle data (yfinance + `price_candles` DB table).

- **Path:** `ticker`; **Query:** `period: "1d"|"5d"|"1mo"|"3mo"|"6mo"|"1y"|"2y"|"5y"|"10y"|"ytd"|"max"` (default 1mo), `interval: "5m"|"30m"|"1h"|"1d"|"5d"|"1wk"|"1mo"|"3mo"` (default 1d)
- **Constraint:** max days per interval: 5m→60, 30m→60, 1h→730, 1d/5d/1wk/1mo/3mo→3650. Exceeding → 400.
- **Response 200:** list of `[{ts, open, high, low, close, volume}]`

#### `GET /price/current` (public)
Current price + change (quote logic).

- **Query:** `ticker: str` (required), `interval: "5m"|"30m"|"1h"|"1d"` (default 5m)
- **Response 200:** `{ticker, price, previous_close, absolute_change, change_pct, as_of, previous_close_as_of, market_status ("open"|"closed"), is_stale, change_window, interval}`
- **Errors:** 404 `"Price not found"` (no price)

#### `GET /stats/top` (public)
Most traded tickers (by ticker_stats total).

- **Query:** `limit: int` (default 50)
- **Response 200:** `[{ticker, name, info_count, report_count, news_count, history_count, simulation_count, favorite_count, total}]`

#### `GET /stats/{ticker}` (public)
Statistics for a single ticker.

- **Response 200:** `{info_count, report_count, news_count, history_count, simulation_count, favorite_count, ticker: "XXX"}`

### 8.3 Reports (`src/api/reports.py`)

#### `POST /reports/generate` (JWT) (feature: report_generate) (slot: report, 900s)
Generates an LLM-based investment report. Credits: the estimated cost is charged upfront, then refunded/additionally charged based on actual token usage.

- **Query:** `ticker: str` (required, valid BIST code), `type: "quick_report"|"deep_report"` (required), `purpose: str|null (max 500)` (optional user question)
- **Cost:** quick: max 5000 tokens, deep: max 50000 tokens; `token_cost_per_1k` default 0.05 → quick estimate ~0.25, deep ~2.5 credits. Insufficient credits: **402** `"insufficient credit"`.
- **Response 200:**
```json
{
  "success": true, "report_id": 12, "credits_spend": 0.25, "remaining_credits": 17.75,
  "about": "THYAO", "type": "quick_report", "title": "THYAO Analysis",
  "report": "<markdown content>", "sentiments": [],
  "token_usage": {"prompt": 0, "completion": 0, "total": 5000},
  "created_at": "2026-08-14T12:00:00+00:00"
}
```
- **Errors:** 400 `"Invalid type"`; 402 insufficient credits; 500 `"Report generation failed"` (credits refunded)
- **Note:** requests are limited to one concurrent report per user (job slot); may take 30-60s+.

#### `GET /reports/info` (public) (no auth dependency; also not in the middleware public list — without a token you get 401)
Report types, costs, and endpoint documentation.

- **Response 200:** `{quick_report: {type, name_en, name_tr, description, description_tr, est_cost}, deep_report: {...}, token_cost_per_1k, endpoints: {generate, history, search, detail}}`

#### `GET /reports/history` (JWT)
The user's report history.

- **Query:** `sort: "created_at"|"ticker"` (default created_at), `order: "asc"|"desc"` (default desc)
- **Response 200:** `[ReportHistoryItem]` — `{id, ticker, type, title|null, token_usage|null, purpose|null, created_at}`
- **Errors:** 400 invalid sort/order (`"Invalid sort. Allowed: ['created_at', 'ticker']"`)

#### `GET /reports/search` (JWT)
ILIKE search in title/content (own reports only).

- **Query:** `q: str (min 1)` (required), `sort`, `order`, `limit: int 1-100` (default 20), `offset: int ≥0`
- **Response 200:** `[ReportHistoryItem]`

#### `GET /reports/{report_id}` (JWT)
Single report (owner check).

- **Response 200:** `{success, report_id, about (ticker), type, title, token_usage, purpose, report (markdown), sentiments, created_at}`
- **Errors:** 404 `"Report not found or you do not have permission to view it."`

#### `POST /reports/download` (JWT)
Downloads the report as md/docx/pdf (pandoc; docx/pdf generation runs in a thread).

- **Query:** `report_id: int` (required), `ftype: "md"|"docx"|"pdf"` (required)
- **Response:** file (appropriate Content-Type + `Content-Disposition: attachment; filename="report_{id}.{ftype}"`)
- **Errors:** 404 `"Report not found."`; 400 `"Invalid file type."`

### 8.4 Simulations (`src/api/simulations.py`)

#### `GET /simulations/per-day-cost` (JWT)
Daily simulation cost.

- **Response 200:** `{"per_day_cost": 0.005, "round": 3}`

#### `GET /simulations/estimate-cost/{ticker}` (JWT)
Cost estimate.

- **Query:** `days: int 1-370` (required)
- **Response 200:** `{"cost": 1.85}` (days × 0.005, 3 decimals)

#### `GET /simulations/history` (JWT)
Simulation history.

- **Query:** `limit: int 1-100` (default 20), `offset: int ≥0`
- **Response 200:** `[{id, ticker, days, bounds, target, cost, created_at}]`

#### `GET /simulations/history/{sim_id}` (JWT)
Simulation detail.

- **Response 200:** `{id, ticker, days, bounds, target, result (JSONB), cost, created_at}`
- **Errors:** 404 `"Simulation not found"`

#### `GET /simulations/{ticker}` (JWT) (feature: simulation) (slot: simulation, 600s)
Runs a Monte Carlo simulation (CPU-heavy, runs in a thread).

- **Path:** `ticker` (valid BIST code); **Query:** `days: int 1-370` (required), `bounds: str` (default "0.05" — confidence interval percentage), `target: str|null` (target price; if omitted auto = current price + 10%, direction "above")
- **Cost:** days × 0.005 credits (402 `"insufficient credit"` if low)
- **Response 200:**
```json
{
  "prob_above": 0.62, "prob_below": 0.38,
  "confidence": {"min": 95.2, "max": 118.4, "percent": 0.9, "days": 30, "bounds": "0.05"},
  "direction": "above", "simulation_id": 7, "ticker": "THYAO", "days": 30,
  "target": "auto", "bounds": "0.05", "credits_spend": 0.15, "remaining_credits": 17.6
}
```
- **Errors:** 400 `"Invalid target price"` (target ≤ 0), `"Invalid simulation parameters"`; 500 `"Simulation failed, credits refunded."`
- **Note:** `direction`: "above" if target ≥ current price, "below" otherwise; "above" when no target.

### 8.5 Economy / Macro / IPO (`src/api/economy.py`, `src/api/ipo.py`)

#### `GET /economy/gold-prices` (public)
Gold prices (GenelPara, 20 min cache). Keys: `gram-altin`, `ceyrek-altin`, `ons`, `gram-has-altin`, `yarim-altin`, `tam-altin`, `cumhuriyet-altini`, `ata-altin`, `14-ayar-altin`, `18-ayar-altin`, `22-ayar-bilezik`, `ikibucuk-altin`, `besli-altin`, `gremse-altin`, `resat-altin`, `hamit-altin`.

- **Response 200:** `{"gram-altin": {"Buying": "3.450,50", "Selling": "3.470,00", "Change": "%0,42", "Type": "Gold"}, ...}`
- **Note:** `Buying`/`Selling` are returned as Turkish comma-decimal **strings**; `Change` is a `%x.xx` string.

#### `GET /economy/silver-price` (public)
Silver price.

- **Response 200:** `{"gumus": {"Buying", "Selling", "Change", "Type": "Gold"}}`

#### `GET /economy/gram-platinum-price` (public)
Gram platinum.

- **Response 200:** `{"gram-platin": {"Buying", "Selling", "Change", "Type": "Commodity"}}`

#### `GET /economy/gram-palladium-price` (public)
Gram palladium.

- **Response 200:** `{"gram-paladyum": {"Buying", "Selling", "Change", "Type": "Commodity"}}`

#### `GET /economy/currency` (public)
FX rates (TRY excluded).

- **Query:** `symbols: str|null` (comma-separated filter, e.g. `USD,EUR`; all if omitted)
- **Response 200:** `{"USD": {"Buying": "40,25", "Selling": "40,30", "Change": "%0,10", "Type": "Currency"}, "EUR": {...}, ...}`

#### `GET /macroeconomy` (public)
FRED macroeconomic indicators (14 series, 24 h cache).

- **Response 200:** macro data dict (gdp, fed_funds, vix, sp500, btc, etc.)
- **Errors:** 500 `"Internal server error"` (no data)

#### `GET /ipos/upcoming` (public)
Upcoming IPOs (halkarz.com WP API, 1 h cache).

- **Query:** `after: str|null` (ISO date filter, default last 30 days)
- **Response 200:** IPO list (JSON)

#### `GET /ipos/draft` (public)
Draft IPOs.

- **Query:** `after: str|null`
- **Response 200:** IPO list

#### `GET /ipos/active` (public)
Active IPOs.

- **Query:** `after: str|null`
- **Response 200:** IPO list

#### `GET /ipos/{slug}` (public)
Single IPO detail.

- **Errors:** 404 `"IPO not found"`

### 8.6 Virtual Portfolios (`src/api/virtual_portfolio.py` + `src/services/portfolio.py`)

> All portfolio endpoints are (JWT) (JWT). `portfolio_id` format: `port-<uuid>`. Data is JSONB; concurrency is protected with `pg_advisory_xact_lock(hashtext(portfolio_id))`. Commission rate: `PORTFOLIO_COMMISSION_RATE` (default 0.001 = 0.1%).
> **Important:** adding transactions (`POST transactions`) only works while the **market is open** (`400 "Market is closed"`); the price is taken automatically from the current market price. Manually priced updates (`PUT transactions/{tx_id}`) are deliberately exempt from this check.

#### `POST /portfolios`
Creates a portfolio.

- **Body:** `{"name": str (1-255), "initial_balance": float (>0)}`
- **Response 200:** `{metadata: {id, user_id, name, initial_balance, balance, created_at, updated_at}, transactions: []}`
- **Errors:** 500 `"Failed to create portfolio"`

#### `GET /portfolios`
Portfolio list.

- **Response 200:** `[{metadata, transactions}]`

#### `GET /portfolios/{portfolio_id}`
Single portfolio.

- **Errors:** 404 `"Portfolio not found"`

#### `PUT /portfolios/{portfolio_id}`
Renames the portfolio.

- **Body:** `{"name": str (1-255)}`
- **Response 200:** `{"message": "Portfolio renamed"}`

#### `DELETE /portfolios/{portfolio_id}`
Deletes the portfolio.

- **Response 200:** `{"message": "Portfolio deleted"}`

#### `POST /portfolios/{portfolio_id}/duplicate`
Duplicates (transactions included, new id).

- **Body:** `{"name": str}`
- **Response 200:** new `{metadata, transactions}`

#### `GET /portfolios/{portfolio_id}/transactions`
Transaction list (filterable).

- **Query:** `ticker: str|null`, `tx_type: str|null` (BUY/SELL, alias: `type`), `start: datetime|null`, `end: datetime|null`
- **Response 200:** `[{id, ticker, type, quantity, price, commission, total, date}]` (sorted by date ascending)

#### `POST /portfolios/{portfolio_id}/transactions`
Adds a transaction (market open only).

- **Body:** `{"ticker": str, "type": "BUY"|"SELL", "quantity": float (>0)}`
- **Response 200:** `{"message": "Transaction added"}`
- **Errors:** 400 `"Market is closed"`, `"Transaction failed"` (insufficient balance / insufficient shares / invalid ticker / price unavailable)

#### `PUT /portfolios/{portfolio_id}/transactions/{tx_id}`
Updates a transaction (price/quantity; portfolio recalculated).

- **Body:** `{"price": float|null, "quantity": float|null}` (at least one)
- **Response 200:** `{"message": "Transaction updated"}`
- **Errors:** 404 `"Transaction not found"`

#### `DELETE /portfolios/{portfolio_id}/transactions/undo`
Undoes the last transaction.

- **Response 200:** `{"message": "Last transaction undone"}`
- **Errors:** 400 `"Nothing to undo"`

#### `GET /portfolios/{portfolio_id}/valuation`
Valuation.

- **Response 200:** `{total_value, cash_balance, holdings_value, total_pnl, pnl_percentage, assets: [{ticker, amount, current_price, total_value, total_cost, weighted_avg_cost, unrealized_pnl, unrealized_pnl_pct}]}`

#### `GET /portfolios/{portfolio_id}/diversification`
Diversification.

- **Response 200:** `{total_value, cash_balance, cash_allocation_pct, assets: [{ticker, amount, value, type ("stock"|"forex"|"metal"), allocation_pct}], allocation_by_type: {"stock": x, ...}}`

#### `GET /portfolios/{portfolio_id}/performers`
Best/worst performing holdings.

- **Query:** `top_n: int 1-20` (default 5)
- **Response 200:** `{best: [{ticker, amount, pnl, pnl_percentage}], worst: [...]}`

#### `GET /portfolios/{portfolio_id}/history`
Time series of value.

- **Query:** `period: "1w"|"1mo"|"3mo"|"6mo"|"1y"|"max"` (default 1mo)
- **Response 200:** `[{ts, total_value, cash_balance, holdings_value}]` (portfolio creation + transaction dates + now)

#### `GET /portfolios/{portfolio_id}/returns`
Return metrics.

- **Query:** `period` (as above)
- **Response 200:** `{period, start_value, end_value, absolute_return, total_return_percentage, cagr_percentage}`
- **Errors:** 404 `"Portfolio not found"`

#### `GET /portfolios/{portfolio_id}/risk`
Risk metrics.

- **Query:** `period: "1w"|"1mo"|"3mo"|"6mo"|"1y"|"max"` (default 1y)
- **Response 200:** `{volatility, max_drawdown, sharpe_ratio}` (fields null with insufficient data)

#### `GET /portfolios/{portfolio_id}/benchmark`
Benchmark comparison.

- **Query:** `ticker: str` (default `XU100`)
- **Response 200:** `{portfolio_return_pct, benchmark_ticker, benchmark_return_pct, difference_pct, outperformed: bool}` — `{}` with insufficient data

#### `GET /portfolios/{portfolio_id}/performance`
Detailed performance analysis (trade timing efficiency).

- **Response 200:** `{overall: {efficiency_score, actual_pnl, optimal_pnl} | null, assets: [{ticker, efficiency_score, actual_trades: [{date, type, price, quantity}], optimal_points: {best_buy: {date, price}, best_sell: {date, price}} | null, price_history: [{ts, close}] | null, actual_pnl, optimal_pnl}]}`

#### `GET /portfolios/{portfolio_id}/stats`
Transaction statistics.

- **Response 200:** `{total_transactions, total_buys, total_sells, total_buy_volume, total_sell_volume, avg_transaction_size, unique_tickers}`

#### `GET /portfolios/{portfolio_id}/snapshot`
Summary in one request: portfolio info + valuation + diversification + performers + stats + last 5 transactions.

- **Response 200:** `{portfolio: {id, name, created_at, updated_at}, valuation, diversification, performers, transaction_stats, recent_transactions}`

#### `GET /portfolios/{portfolio_id}/export/csv`
Downloads transactions as CSV (`text/csv`).

- **Response:** CSV with header `date,ticker,type,quantity,price,total` (date ascending)

### 8.7 Favorites (`src/api/favorites.py`)

#### `POST /favorites/{ticker}` (JWT)
Adds to favorites (idempotent — silently passes if already present).

- **Response 200:** `{"message": "Added favorite XXX or already been added"}`
- **Errors:** 404 invalid ticker; 400 `"Could not add to favorites"`

#### `DELETE /favorites/{ticker}` (JWT)
Removes from favorites.

- **Response 200:** `{"message": "Removed XXX from favorites"}`

#### `GET /favorites` (JWT)
Favorite list.

- **Response 200:** `{"favorites": ["THYAO", "ASELS", ...]}`

### 8.8 Bots (`src/api/bots.py`)

#### `POST /bots` (JWT)
Creates a bot account (max 5/user). Bots spend the owner's credits; no email verification needed; **the password is returned only once in this response**.

- **Body:** `{"username": str (3-255), "password": str|null (min 10)}` (if empty, a random 16-char password is generated)
- **Response 200:** `{"id": 55, "username": "...", "email": "...@bot.florencex.com.tr", "password": "<one-time>"}`
- **Errors:** 403 `"error_bots_not_allowed"` (bots cannot create bots); 400 `"error_bot_limit_reached"`; 409 `"error_username_taken"`

#### `GET /bots` (JWT)
Bot list.

- **Response 200:** `{"bots": [{id, username, created_at, last_login}]}`

#### `DELETE /bots/{bot_id}` (JWT)
Deletes a bot (own bots only).

- **Response 200:** `{"message": "Bot {id} deleted"}`
- **Errors:** 404 `"Bot not found"`

### 8.9 Data Export (`src/api/export.py`, `src/api/exports.py`)

#### `GET /user/export` (JWT)
JSON dump of all user data (profile + favorites + reports + token usage + simulations).

- **Response 200:** `{profile: {username, email, credits}, favorites: [{ticker_code, created_at}], reports: [{id, ticker, type, title, token_usage, content, created_at}], token_usage: [{model, prompt_tokens, completion_tokens, total_tokens, endpoint, created_at}], simulations: [{id, ticker, days, bounds, target, result, cost, created_at}]}`

#### `POST /data/export` (JWT) (202 Accepted)
Queues a takeout-style yearly data export; a background worker prepares it and emails the download link. Idempotent: if an active record exists for the same user+year+format, the existing id is returned.

- **Body:** `{"year": int (1990..current year+1), "format": "csv"|"json"}` (format default csv)
- **Rate limit:** 3/hour (admin 30/hour)
- **Response 202:** `{"export_id": 5, "status": "queued"}`
- **Errors:** 400 `"Invalid year"`

#### `GET /data/export` (JWT)
Export list.

- **Response 200:** `[{id, year, format, status, created_at, updated_at, row_count, size_bytes, downloaded_count, expires_at, error, downloadable, download_url}]`

#### `GET /data/export/download/{token}` (public)
Public download (no auth, token is enough). Filename `florence-daily-{year}.{format}.gz`.

- **Errors:** 404 `"Export not found"` / `"Export file missing"`; 410 `"Export link expired or not ready"` (status not ready/sent or expired)

#### `GET /data/export/{export_id}` (JWT)
Single export record (owner check).

- **Response 200:** serialized record (as above)
- **Errors:** 404 `"Export not found"`

#### `GET /data/daily/{year}` (public)
**Deprecated** — always returns 410.

- **Response 410:** `{"detail": "Kullanım dışı — POST /api/v1/data/export ile istek oluşturun"}`

### 8.10 Announcements (`src/api/announcements.py`)

#### `GET /announcements` (JWT)
Announcement list (only those published within the last 7 days; with `is_unread` flag).

- **Response 200:** `{"announcements": [{id, title, content, sent_by, created_at, updated_at, is_unread}]}`

#### `GET /announcements/{announcement_id}` (JWT)
Single announcement.

- **Errors:** 404 `"Announcement not found"`

#### `POST /announcements` (JWT) (admin)
Creates an announcement.

- **Body:** `{"title": str, "content": str}`
- **Errors:** 403 `"Admin access required"`

#### `PUT /announcements/{announcement_id}` (JWT) (admin)
Updates an announcement.

- **Body:** `{"title": str, "content": str}`
- **Errors:** 403; 404 `"Announcement not found"`

#### `DELETE /announcements/{announcement_id}` (JWT) (admin)
Deletes an announcement.

- **Errors:** 403; 404

#### `POST /announcements/read` (JWT)
Marks all announcements as read (`last_announcement_viewed_at = NOW()`).

- **Response 200:** `{"message": "Marked as read"}`

### 8.11 Advisor — Stock Recommendations (`src/api/fit.py`, `src/api/portfolio.py`)

#### `POST /stocks/fit` (JWT) (feature: advisor)
Stock-vector similarity recommendations (risk/horizon/profitability scores).

- **Body:** `{"horizon": "short"|"medium"|"long" (default long), "profitability": "low"|"medium"|"high" (default high), "risk_tolerance": "low"|"medium"|"high" (default medium), "limit": int 1-100 (default 5)}`
- **Response 200:** `{"query": {"risk": 0.5, "horizon": 0.7, "profitability": 0.7}, "results": [{ticker, score, vector, ...}]}`
- **Errors:** 400 `"Invalid filter value"`

#### `POST /portfolio/profile` (JWT) (feature: advisor)
"Stocks similar to my portfolio": Euclidean-nearest stocks to the average of the user's portfolio vectors.

- **Body:** `{"tickers": [str (1-50, uppercased)], "limit": int 1-50 (default 5)}`
- **Response 200:** `{avg_vector: [float...], estimated_profile: {risk, horizon, profitability}, portfolio: [{ticker, vector}], similar_stocks: [{ticker, vector, score, distance}]}`
- **Errors:** 400 `"No vector data available for given tickers"`

### 8.12 Other Public Endpoints

#### `GET /market/status` (public)
BIST open/closed + public holiday (Redis 60 s cache). Market hours: 10:00–18:10 (Europe/Istanbul), open on weekdays and non-holidays.

- **Response 200:** `{"open": true, "next_open_at": "2026-08-17T10:00:00+03:00"|null, "timezone": "Europe/Istanbul", "is_holiday": false, "holiday_name": null, "as_of": "..."}`

#### `GET /maintenance` (public)
List of disabled features.

- **Response 200:** `{"disabled_features": []}` (set members: `report_generate`, `simulation`, `news`, `advisor`)

#### `GET /meta/avatars` (public)
Avatar id list.

- **Response 200:** `[{"id": "avatar-1", "url": "/avatars/avatar-1.svg"}, ... 12 items]`

#### `GET /legal?policy=&lang=` (public)
Legal text. `policy: "terms"|"privacy_policy"|"cookie_policy"|"disclaimer"`, `lang: "tr"|"en"` (default tr).

- **Response 200:** `{"policy", "lang", "last_updated": "2026-07-22", "content": "<text>"}`
- **Errors:** 404 unknown policy; 400 invalid lang

#### `GET /legal/all?lang=` (public)
All legal texts.

- **Response 200:** `{"last_updated", "lang", "policies": {"terms": ..., "privacy_policy": ..., ...}}`

#### `GET /about?lang=` (public)
About text (`"tr"|"en"`).

- **Response 200:** `{"lang", "content"}`

#### `GET /contact` (public)
Contact information.

- **Response 200:** the `CONTACT` constant

#### `GET /version` (public)
Application version.

- **Response 200:** `{"version": "0.5.7"}`

#### `GET /contributors` (public)
Contributors.

- **Response 200:** `{"contributors": [{nickname, picture_url, github_url}]}`

#### `GET /` and `GET /health` (public)
Health checks.

- **Response 200:** `{}` and `{"status": "ok"}`

#### `POST /analytics/event` (JWT) (via middleware; the path itself is excluded from middleware analytics tracking)
Batch analytics events (fire-and-forget).

- **Body:** `[{event_type: str, ticker: str|null, details: dict|null, ...}, ...]` — max 100 items
- **Response 200:** `{"received": n}`
- **Errors:** 413 `"Too many events"`

### 8.13 Admin Application (`src/admin/__init__.py` — separate FastAPI, served over UDS `/run/florence/admin.sock`)

> Auth: `X-Admin-Token: <ADMIN_TOKEN>` header (required, `verify_admin_token`). This app is **not** under `/api/v1`.

| Method | Path | Description |
|---|---|---|
| POST | `/gift-credits` | Gift credits. Query: `user_type: "everyone"\|"user"`, `amount: int (>1)`, `username: str|null`, `credit_type: "free_credits"\|"gift_credits"` (default free_credits); Body: `filters: dict` (optional `{}`). `everyone` → `{"success": true}`; `user` → `{"success": true, "user": {username, credits}}`. Errors: 400 `"username is required for user type"`, 404 `"User not found"` |
| POST | `/config-reload` | Reloads the config. → `{"success": true}` |
| POST | `/healthcheck` | DB/Redis/LLM/SearXNG/yfinance health report. → `{db_health, redis_health, llm_health, news_health, yfinance_health, status: "OK"\|"ERROR"}` |
| POST | `/token-usage` | Token usage summary. Query: `since: str|null` (ISO), `endpoint: str|null`. Errors: 400 `"Invalid datetime format. Use ISO format."` |
| POST | `/maintenance/toggle` | Enable/disable a feature. Query: `feature` (report_generate/simulation/news/advisor), `action: "enable"\|"disable"`. → `{feature, disabled: bool}`. Errors: 400 `"Unknown feature: X"` / `"Action must be 'enable' or 'disable'"` |

---

## 9. Environment Variables

| Variable | Default | Description |
|---|---|---|
| `ENVIRONMENT` | `development` | `production` disables docs, restricts CORS to Tauri origins, sets cookies `secure` |
| `SECRET_KEY` | — | **Required.** JWT HS256 signing key |
| `ADMIN_TOKEN` | — | Admin app `X-Admin-Token` value |
| `FRED_API_KEY` | — | **Required at import time** (raises `ValueError` otherwise) — FRED macro data |
| `FREE_CREDIT_MAX` | `25` | Daily refill cap (free_credits) |
| `DAILY_FREE_CREDIT_REFILL` | `5` | Daily automatic credit top-up amount |
| `REFRESH_TOKEN_TTL_DAYS` | `30` | Refresh token lifetime |
| `PORTFOLIO_COMMISSION_RATE` | `0.001` | Virtual portfolio commission rate (0.1%) |
| `LOG_DIR` | `/var/log/florence` | Log directory (must be writable) |
| `PUBLIC_BASE_URL` | request base URL | Public URL used in email verification links |
| `POSTGRES_HOST/PORT/USER/PASSWORD/DB` | localhost:5433 | PostgreSQL connection |
| `REDIS_HOST/PORT/DB/PASSWORD` | localhost:5434 | Redis connection |
| `SEARXNG_HOST_PORT` | `5435` | SearXNG host port |
| `NEWS_SEARCH_URL` | `http://localhost:5435/search` | SearXNG search URL |
| `MAIL_PROVIDER` | `resend` | `smtp`/`mailpit`/`resend`/`ses` |
| `MAIL_FROM`, `MAIL_HOST/PORT/USER/PASS`, `RESEND_API_KEY`, `MAIL_EXPORT_FROM` | — | Email settings |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | BigQuery (GDELT news archive) service account |
| `COLLECT_API_KEY` | — | News collection API key |
| `OPENROUTER_API_KEY`, `CUSTOM_API_KEY`, `CUSTOM_URL` (`http://localhost:7777/v1`), `CUSTOM_MODEL` (`gemma`) | — | LLM configuration |
| `EMBEDDING_API_KEY/BASE_URL/MODEL` | ollama / `http://127.0.0.1:11434/v1` / `mxbai-embed-large` | Embedding (stock vectors) |
| `EVDS_API_KEY`, `LOGODEV_API_KEY` | — | TCMB EVDS and logo API keys |

**Config overrides:** `get_config()` defaults can be overridden via env with the `<SECTION>_<KEY>` scheme (e.g. `REPORT_TOKEN_COST_PER_1K`, `ECONOMY_CACHE_TTL`, `MACROECONOMY_CACHE_TTL`, `PRICE_HISTORY_CACHE_TTL_HOT`, `SIMULATION_PER_DAY_COST`…). Full list: backend `.env.example` + `src/core/config.py` `_DEFAULTS`.

---

## 10. Credit System

- `user_credits` table: `(user_id, credit_type, amount)` — `free_credits` and `gift_credits` are separate rows; spending uses free credits first.
- Daily refill (cron): adds `DAILY_FREE_CREDIT_REFILL` to every user, capped at `FREE_CREDIT_MAX`.
- Bot accounts spend the owner's credits (`credits._resolve_owner`).
- Spending: report generation (estimated cost charged upfront, refunded/additionally charged against actual usage) and simulations (days × 0.005).
- Insufficient credits → 402 `"insufficient credit"`.

---

## 11. Feature Kill-Switch (Maintenance)

Features can be toggled instantly via the `maintenance:disabled` Redis set (admin `/maintenance/toggle`). Endpoints of a disabled feature return **503** `"{feature} is temporarily disabled for maintenance"`.

| Feature | Affected endpoints |
|---|---|
| `report_generate` | `POST /reports/generate` |
| `simulation` | `GET /simulations/{ticker}` |
| `news` | `GET /news/{ticker}` |
| `advisor` | `POST /stocks/fit`, `POST /portfolio/profile` |

---

## 12. Developer Notes / Pitfalls

1. **FRED_API_KEY import error:** `src/clients/macroeconomy.py` looks for `FRED_API_KEY` at import time and raises `ValueError` if missing. Export a dummy value in test/script environments.
2. **Schema source is `init_db()`:** the `migrations/` files are not applied automatically; update both when changing the schema.
3. **DB connection discipline:** `db.release_current()` is called before long-running work (LLM calls, simulations); new cursor blocks acquire a fresh connection. `keep=True` is only used in advisory-lock flows.
4. **`price_candles` ticker format:** stored as `XXX.IS` in the DB; API inputs are accepted without the `.IS` suffix and normalized.
5. **Quote `change_pct`:** returns `null` when the market is open but no intraday data exists (no misleading 0.00); `is_stale` threshold is 20 min when open, 3 days when closed.
6. **Login is form-encoded** (OAuth2), not JSON. Register is JSON.
7. **Job slots block concurrent work:** the same user cannot start two reports/simulations at the same time (429).
8. **`/reports/info` is not public:** it is not in the middleware public list and has no auth dependency — behavior is "401 without a token"; this is documented here for clarity.

---

*This document was produced on 2026-08-14 from the backend codebase using a mix of automated and manual methods. Source: `florence/backend` (version 0.5.7).*
