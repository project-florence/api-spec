# Florence Backend — AI Context Pack

> Single-file compact context so an AI agent can understand the Florence backend and add new endpoints/features correctly.
> Source of truth: `/home/efe/Belgeler/florence/backend` (version 0.5.7, FastAPI 0.139, fully async, no ORM).
> Generated: 2026-08-14. Full reference: `api-reference-en.md` / `api-reference-tr.md`. Machine spec: `openapi.json` at repo root.

---

## 1. Architecture Summary

Florence is a BIST (Turkish stock exchange) financial platform API server.

- **Stack:** FastAPI + uvicorn, PostgreSQL 17 (psycopg3 `AsyncConnectionPool`), Redis 7 (redis.asyncio, down-tolerant proxy), Pydantic v2, PyJWT (HS256), argon2-cffi, pydantic-ai (LLM agents), yfinance/pykap (market data), GenelPara (FX/gold), FRED (macro), SearXNG + GDELT/BigQuery (news), pandas/numpy/scipy (simulation & analytics), pandoc (docx/pdf).
- **Layering:** `src/api/` (HTTP handlers + Pydantic schemas) → `src/services/` (domain logic, DB+Redis access) → `src/clients/` (external integrations) → `src/core/` (config, db, redis, ratelimit, job_slots). Admin is a **separate** FastAPI app in `src/admin/`.
- **Entrypoint:** `src/main.py` → `app` (mounts `/avatars`, CORS, auth middleware, routers via `src/api/router.py` under prefix `/api/v1`). Docs (`/docs`, `/redoc`, `/openapi.json`) are enabled only when `ENVIRONMENT != production`.
- **No ORM, no migration runner.** Schema is created idempotently at startup by `src/core/database.py:init_db()` (`CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ADD COLUMN IF NOT EXISTS`). `migrations/` SQL files are historical/manual and NOT applied automatically — when changing schema, update `init_db()` AND any applicable migration.

### Async rules (mandatory, per AGENTS.md)

- Endpoints are `async def`; all dependencies (`get_current_user`, `validate_ticker`, `require_feature`, `require_job_slot`) are async.
- DB: `async with db.cursor() as cur:` → dict rows by default; `row_factory=None` → tuple rows. Commit/rollback inside the block; `db.release_current()` is called by middleware after each request.
- Redis: `await r.get(...)`, `await r.set(..., nx=True, ex=...)`. Returns `None` when Redis is down (cache-less mode — always handle `None`).
- HTTP: shared `httpx.AsyncClient` via `await get_client()` from `src/clients/http.py`. Never `requests`.
- LLM: `AsyncOpenAI` clients in `src/clients/llm.py` / `embedding.py`; always `await`.
- Sync libs (yfinance, trafilatura, BigQuery, fredapi, argon2, numpy, pandas) must be called via `asyncio.to_thread(...)` — never block the event loop.
- Long DB-held work: call `await db.release_current()` BEFORE long operations (LLM calls, simulations) so the pool connection is returned.

---

## 2. Codebase Map (where things live)

| Concern | Location |
|---|---|
| Router mounting (`/api/v1`) | `src/api/router.py` |
| Auth deps (JWT decode, admin token, ticker validation) | `src/api/deps.py` |
| Feature routers | `src/api/{auth,bist,reports,simulations,economy,ipo,stats,favorites,fit,portfolio,export,exports,data,legal,maintenance,analytics,announcements,contributors,virtual_portfolio,market,meta,bots}.py` |
| Config (env-based, `get_config()`) | `src/core/config.py` (defaults dict `_DEFAULTS`, env scheme `<SECTION>_<KEY>`) |
| DB pool + schema | `src/core/database.py` (`db` proxy, `init_db()`) |
| Redis proxy (down-tolerant) | `src/core/redis.py` (`r`) |
| Rate limiting | `src/core/ratelimit.py` (`rate_limiter.check(key, max_requests, window_seconds, is_admin)`) |
| Job slots (concurrency lease) | `src/core/job_slots.py` (`require_job_slot(kind, ttl_seconds)`) |
| Services | `src/services/{bist,company,news,price,quote,market,economy,stats,credits,portfolio,simulation_history,refresh_token,announcement,analytics,export_jobs,maintenance,ipo,search,token,report/}.py` |
| LLM report agent (pydantic-ai) | `src/services/report/__init__.py` |
| External clients | `src/clients/{yfinance,search,gdelt,news,llm,embedding,mail,macroeconomy,ipo,scraping,http,cron}.py` |
| Simulation engine | `src/simulation/montecarlo.py` (`simulate(ticker, days, bounds, target)`) |
| Stock vectors (risk/horizon/profitability) | `src/analysis/stock_vector.py`, `src/analysis/metrics.py` |
| Cron jobs | `src/cron/register.py`, `src/cron/tasks.py`; scheduler `src/clients/cron.py` (jobs need `async def __cron_main__()`) |
| Admin app (separate) | `src/admin/__init__.py` (`admin_app`, `X-Admin-Token`) |
| Legal static texts (TR/EN) | `src/legal/` |
| Utils | `src/utils/{mapping,file_utils,dates,text}.py` |

---

## 3. Authentication

### Token model

- **Access token:** JWT HS256 signed with `SECRET_KEY`; payload `{user_id, iat, exp}`; **1 hour** lifetime. Accepted via `Authorization: Bearer <token>` **or** `access_token` httpOnly cookie (path `/`, samesite=strict, secure in production).
- **Refresh token:** opaque `secrets.token_urlsafe(48)`; stored **SHA-256 hashed** in `refresh_tokens`; **rotated on every refresh** (old revoked, new issued); TTL 30 days (`REFRESH_TOKEN_TTL_DAYS`). Sent in body `{"refresh_token": "..."}` or via `refresh_token` cookie (path `/api/v1/auth`).
- **Passwords:** Argon2 (`argon2.PasswordHasher`, verify in `asyncio.to_thread`).
- Extra checks in `_decode_user` (`src/api/deps.py`): password change invalidation (`password_changed_at > iat`, Redis cache 60s) and frozen-account denial (Redis cache 30s). Unverified email blocks login/refresh (bots exempt).
- **Admin:** separate `X-Admin-Token` header == `ADMIN_TOKEN` env (string equality). Admin users get 10x rate limits on news/export.
- **Middleware** (`main.py`): any `/api/*` path not in the PUBLIC_PATHS set requires a valid JWT → else `401 {"detail": "Not authenticated"}`. `OPTIONS` exempt. `PoolTimeout` → `503 "Database busy, please retry"` (NOT 401). `db.release_current()` after every request.

### Auth flow (canonical)

1. `POST /api/v1/auth/register` `{username, email, password(≥10)}` → `{message, user_id, verification_sent}` (24h verify token mailed; mail failure never breaks registration).
2. `GET /api/v1/auth/verify-email?token=...` → `{message: "Email verified", email_verified: true}`.
3. `POST /api/v1/auth/login` — **OAuth2 form-encoded** (`username`=username or email, `password`) → `{access_token, refresh_token, token_type}` + httpOnly cookies.
4. `POST /api/v1/auth/refresh` — body or cookie → new pair (rotation).
5. `POST /api/v1/auth/logout` — revokes token + clears cookies.
6. Bots: `POST /api/v1/bots` creates `user_type='bot'` accounts (max 5/user, email `<user>@bot.florencex.com.tr`, pre-verified, spend owner's credits via `credits._resolve_owner`; password returned once).

---

## 4. Endpoint Inventory (compact)

All under `/api/v1`. Legend: P=public, J=JWT, A=admin-only within JWT, ⚙=feature-gated, ⏳=job-slot.

### Auth & User (`src/api/auth.py`)
| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/auth/register` | P | 3/min/IP; JSON body |
| POST | `/auth/login` | P | 5/min; OAuth2 form; sets cookies |
| GET | `/auth/verify-email` | P | `?token=` |
| POST | `/auth/resend-verification` | P | 3/hour; `{username_or_email}` |
| POST | `/auth/refresh` | P | 5/min; rotation |
| POST | `/auth/logout` | P | revoke + cookies |
| DELETE | `/auth/delete` | J | delete account |
| PUT | `/auth/change-password` | J | revokes all refresh tokens |
| PUT | `/auth/change-email` | J | revokes all |
| PUT | `/auth/change-username` | J | revokes all |
| GET | `/profile` | J | profile + credits |
| PUT | `/profile/avatar` | J | `avatar-1..12` |
| GET | `/credits` | J | balance |
| GET/PUT | `/user/preferences` | J | JSONB prefs, PUT merges |

### BIST / Company / Price (`src/api/bist.py`, `stats.py`)
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/bist/companies` | P | sort=alphabetical\|popular, offset, limit(≤500) |
| GET | `/bist/tickers` | P | same params |
| GET | `/companies/search` | P | `?query=` (aliases) |
| GET | `/companies/info/{ticker}` | P | structured yfinance profile |
| GET | `/companies/info/{ticker}/md` | P | markdown text |
| GET | `/companies/summary` | P | sort incl. gainers/losers/price_high/volume/market_cap; `tickers` CSV filter |
| GET | `/news/{ticker}` | J ⚙news | 10/min (admin 100); `amount` 1-50; GDELT |
| GET | `/price/history/{ticker}` | P | period 1d..max; interval 5m..3mo (interval/day constraints!) |
| GET | `/price/current` | P | quote; `?ticker=&interval=` |
| GET | `/stats/top` | P | popular tickers by activity |
| GET | `/stats/{ticker}` | P | per-ticker counters |

### Reports (`src/api/reports.py`)
| Method | Path | Auth | Notes |
|---|---|---|---|
| POST | `/reports/generate` | J ⚙report_generate ⏳report(900s) | `?ticker=&type=quick_report\|deep_report&purpose=`; credit: estimate charged → refund/extra by actual tokens; 402 insufficient |
| GET | `/reports/info` | (none — 401 without token) | costs + endpoint docs |
| GET | `/reports/history` | J | sort=created_at\|ticker, order=asc\|desc (allowlist) |
| GET | `/reports/search` | J | `?q=` ILIKE title/content; limit/offset |
| GET | `/reports/{report_id}` | J | owner-only; markdown content |
| POST | `/reports/download` | J | `?report_id=&ftype=md\|docx\|pdf` (pandoc) |

### Simulations (`src/api/simulations.py`)
| Method | Path | Auth | Notes |
|---|---|---|---|
| GET | `/simulations/per-day-cost` | J | 0.005 |
| GET | `/simulations/estimate-cost/{ticker}` | J | `?days=1..370` |
| GET | `/simulations/history` | J | limit/offset |
| GET | `/simulations/history/{sim_id}` | J | detail incl. result JSONB |
| GET | `/simulations/{ticker}` | J ⚙simulation ⏳simulation(600s) | `?days=&bounds=&target=`; cost=days×0.005; returns prob_above/prob_below/confidence + metadata |

### Economy / Macro / IPO (`src/api/economy.py`, `ipo.py`) — all P
| Method | Path | Notes |
|---|---|---|
| GET | `/economy/gold-prices` | 16 gold keys, `{Buying,Selling,Change,Type}` strings (TR comma decimals) |
| GET | `/economy/silver-price` | `{"gumus": ...}` |
| GET | `/economy/gram-platinum-price` | `{"gram-platin": ...}` |
| GET | `/economy/gram-palladium-price` | `{"gram-paladyum": ...}` |
| GET | `/economy/currency` | `?symbols=USD,EUR` filter |
| GET | `/macroeconomy` | FRED, 14 series, 24h cache |
| GET | `/ipos/upcoming` | `?after=` ISO |
| GET | `/ipos/draft` | `?after=` |
| GET | `/ipos/active` | `?after=` |
| GET | `/ipos/{slug}` | detail, 404 if missing |

### Virtual Portfolios (`src/api/virtual_portfolio.py` + `services/portfolio.py`) — all J
| Method | Path | Notes |
|---|---|---|
| POST | `/portfolios` | `{name, initial_balance>0}` → `{metadata, transactions}` |
| GET | `/portfolios` | list |
| GET | `/portfolios/{id}` | single |
| PUT | `/portfolios/{id}` | rename |
| DELETE | `/portfolios/{id}` | delete |
| POST | `/portfolios/{id}/duplicate` | copy w/ transactions |
| GET | `/portfolios/{id}/transactions` | filters: ticker, tx_type(alias type), start, end |
| POST | `/portfolios/{id}/transactions` | `{ticker, type BUY\|SELL, quantity}` — **market must be open** (400 `"Market is closed"`); price auto from market; commission `PORTFOLIO_COMMISSION_RATE` |
| PUT | `/portfolios/{id}/transactions/{tx_id}` | manual price/quantity update (market-open check exempt) |
| DELETE | `/portfolios/{id}/transactions/undo` | undo last |
| GET | `/portfolios/{id}/valuation` | total_value, pnl, per-asset breakdown |
| GET | `/portfolios/{id}/diversification` | by type stock/forex/metal |
| GET | `/portfolios/{id}/performers` | `?top_n=` best/worst |
| GET | `/portfolios/{id}/history` | `?period=1w\|1mo\|3mo\|6mo\|1y\|max` |
| GET | `/portfolios/{id}/returns` | abs/total/CAGR |
| GET | `/portfolios/{id}/risk` | volatility, max_drawdown, sharpe |
| GET | `/portfolios/{id}/benchmark` | vs XU100 default |
| GET | `/portfolios/{id}/performance` | efficiency scoring |
| GET | `/portfolios/{id}/stats` | tx stats |
| GET | `/portfolios/{id}/snapshot` | combined summary |
| GET | `/portfolios/{id}/export/csv` | CSV download |

Concurrency: every mutation takes `pg_advisory_xact_lock(hashtext(portfolio_id))` via `db.cursor(row_factory=None, keep=True)`; commit inside the block releases it. Data model: `Portfolio{metadata: Metadata{id:"port-<uuid>", user_id, name, initial_balance, balance, created_at, updated_at}, transactions: [Transaction{id:"tx-<uuid>", ticker, type, quantity, price, commission, total, date}]}`.

### Favorites / Bots / Export / Announcements / Advisor / Misc — all J unless noted
| Method | Path | Auth | Notes |
|---|---|---|---|
| POST/DELETE | `/favorites/{ticker}` | J | idempotent add |
| GET | `/favorites` | J | list |
| POST | `/bots` | J | create bot (5 max) |
| GET | `/bots` | J | list own bots |
| DELETE | `/bots/{bot_id}` | J | delete own bot |
| GET | `/user/export` | J | full JSON dump |
| POST | `/data/export` | J | 202; 3/hour (admin 30); `{year, format csv\|json}`; idempotent; background worker + email link |
| GET | `/data/export` | J | list |
| GET | `/data/export/download/{token}` | P | token-based; 410 if expired/not ready |
| GET | `/data/export/{export_id}` | J | owner-only |
| GET | `/data/daily/{year}` | P | **410 Gone** (deprecated) |
| GET | `/announcements` | J | last 7 days, is_unread |
| GET | `/announcements/{id}` | J | single |
| POST | `/announcements` | J+A | create |
| PUT | `/announcements/{id}` | J+A | update |
| DELETE | `/announcements/{id}` | J+A | delete |
| POST | `/announcements/read` | J | mark all read |
| POST | `/stocks/fit` | J ⚙advisor | `{horizon, profitability, risk_tolerance, limit}` → similarity |
| POST | `/portfolio/profile` | J ⚙advisor | `{tickers[1-50], limit}` → similar stocks (Euclidean) |
| GET | `/market/status` | P | open/next_open_at/holiday (60s cache) |
| GET | `/maintenance` | P | disabled feature list |
| GET | `/meta/avatars` | P | 12 avatars |
| GET | `/legal`, `/legal/all`, `/about`, `/contact`, `/version`, `/contributors` | P | static content |
| GET | `/`, `/health` | P | `{}`, `{"status":"ok"}` |
| POST | `/analytics/event` | J(mw) | batch ≤100, fire-and-forget |
| — | `/docs`, `/redoc`, `/openapi.json` | dev only | Swagger/ReDoc/spec |

### Admin app (separate, UDS socket, `X-Admin-Token` required)
| Method | Path | Notes |
|---|---|---|
| POST | `/gift-credits` | everyone\|user; amount>1; credit_type free\|gift |
| POST | `/config-reload` | reload config |
| POST | `/healthcheck` | db/redis/llm/news/yfinance |
| POST | `/token-usage` | since/endpoint filters |
| POST | `/maintenance/toggle` | feature + enable\|disable |

---

## 5. Key Pydantic Schemas (request bodies, exact field names)

- `UserRegister`: `username: str`, `email: EmailStr`, `password: str(min 10)` — all required.
- `ChangePassword`: `current_password`, `new_password(min 10)`. `UpdateEmail`: `new_email`, `current_password`. `UpdateUsername`: `new_username`, `current_password`.
- `RefreshRequest`: `refresh_token: str | None = None` (null body OK — cookie fallback).
- `ResendVerification`: `username_or_email: str`. `AvatarUpdate`: `avatar_id: str`.
- `PreferencesUpdate`: `prefs: dict` (merged into existing).
- `BotCreate`: `username(3-255)`, `password: str|None(min 10)`.
- `FitRequest`: `horizon`(short/medium/long, dflt long), `profitability`(low/medium/high, dflt high), `risk_tolerance`(low/medium/high, dflt medium), `limit`(1-100, dflt 5).
- `PortfolioProfileRequest`: `tickers: list[str]` (1-50, uppercased), `limit`(1-50, dflt 5).
- `CreatePortfolioBody`: `name(1-255)`, `initial_balance: float(>0)`. `RenamePortfolioBody`/`DuplicatePortfolioBody`: `name(1-255)`.
- `AddTransactionBody`: `ticker(min 1)`, `type(pattern ^(BUY|SELL)$)`, `quantity: float(>0)`. `UpdateTransactionBody`: `price: float|None(>0)`, `quantity: float|None(>0)`.
- `AnnouncementCreate`/`AnnouncementUpdate`: `title`, `content`.
- `ExportRequest`: `year: int`, `format: Literal["csv","json"]="csv"`.
- `ReportHistoryItem` (response): `id:int, ticker:str, type:str, title:str|None, token_usage:dict|None, purpose:str|None, created_at:str`.

Response shapes (from code, not fully in openapi): see section 4 notes and the full reference; key ones — login `{access_token, refresh_token, token_type}`; profile `{username, email, user_type, created_at, email_verified, avatar_id, credits}`; report generate `{success, report_id, credits_spend, remaining_credits, about, type, title, report(md), sentiments, token_usage, created_at}`; simulation `{prob_above, prob_below, confidence{min,max,percent,days,bounds}, direction, simulation_id, ticker, days, target, bounds, credits_spend, remaining_credits}`; quote `{ticker, price, previous_close, absolute_change, change_pct, as_of, previous_close_as_of, market_status, is_stale, change_window}`.

---

## 6. Example Requests

```bash
# Register
curl -X POST http://localhost:7055/api/v1/auth/register -H 'Content-Type: application/json' \
  -d '{"username":"demo","email":"demo@example.com","password":"supersecret123"}'
# {"message":"Register successful","user_id":42,"verification_sent":true}

# Login (form-encoded!)
curl -X POST http://localhost:7055/api/v1/auth/login \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=demo&password=supersecret123'
# {"access_token":"<jwt>","refresh_token":"<opaque>","token_type":"bearer"}

# Authenticated call
curl http://localhost:7055/api/v1/profile -H 'Authorization: Bearer <access_token>'

# Refresh (body or cookie)
curl -X POST http://localhost:7055/api/v1/auth/refresh -H 'Content-Type: application/json' \
  -d '{"refresh_token":"<opaque>"}'

# Price history
curl 'http://localhost:7055/api/v1/price/history/THYAO?period=1mo&interval=1d'
# [{"ts":"2026-07-15T00:00:00+00:00","open":310.5,"high":315.0,"low":308.2,"close":313.4,"volume":12345678}, ...]

# Report generation (credits required)
curl -X POST 'http://localhost:7055/api/v1/reports/generate?ticker=THYAO&type=quick_report' \
  -H 'Authorization: Bearer <access_token>'
# {"success":true,"report_id":12,"credits_spend":0.25,"remaining_credits":17.75,"about":"THYAO",...}

# Simulation
curl 'http://localhost:7055/api/v1/simulations/THYAO?days=30&bounds=0.05' \
  -H 'Authorization: Bearer <access_token>'
# {"prob_above":0.62,"prob_below":0.38,"confidence":{"min":95.2,"max":118.4,"percent":0.9,"days":30,"bounds":"0.05"},...}
```

---

## 7. Rate Limits & Security Rules

| Endpoint | Limit | Admin |
|---|---|---|
| register | 3/min/IP | — |
| login | 5/min/username | — |
| refresh | 5/min/token | — |
| resend-verification | 3/hour | — |
| news/{ticker} | 10/min/user+ticker | 100 |
| data/export (POST) | 3/hour | 30 |

- 429 body: `{"detail": "Too many requests. Please slow down."}`.
- Job slots (concurrent-job guard): `job-slot:{kind}:{user_id}` lease 120s + 30s heartbeat; report 900s, simulation 600s; busy → 429 `"A job of this type is already running"`.
- Feature kill-switch: `maintenance:disabled` Redis set (`report_generate`, `simulation`, `news`, `advisor`); disabled → 503 `"{feature} is temporarily disabled for maintenance"`.
- Admin multiplier: `rate_limiter.check(..., is_admin=(user_type=="admin"))` → 10x.
- SQL injection hardening: sort/order columns are allowlisted (`SORT_COLUMNS`/`ORDER` in reports); GDELT search terms regex-guarded (`^[A-Z0-9 .\-]+$`).
- No secrets in responses; bot passwords returned once; refresh tokens stored hashed.

---

## 8. Developer Pitfalls (read before coding)

1. **`FRED_API_KEY` is required at import time** — `src/clients/macroeconomy.py` raises `ValueError` if missing. Any script/test importing the app must export a dummy (`export FRED_API_KEY=dummy`).
2. **`LOG_DIR` must be writable** — default `/var/log/florence` needs root; use `LOG_DIR=/tmp/...` in local runs.
3. **`SECRET_KEY` required** — `src/main.py` raises `RuntimeError` at import without it. `ADMIN_TOKEN` needed for admin app imports.
4. **Schema changes:** update `init_db()` (idempotent DDL) AND the relevant `migrations/*.sql`; never assume migrations run automatically.
5. **Connection discipline:** release `db.release_current()` before long work; use `keep=True` only for advisory-lock flows; `db.commit()` inside cursor blocks (auto-return would roll back otherwise).
6. **Ticker normalization:** DB stores `XXX.IS`; API accepts bare codes; `validate_ticker` (deps) + `services/bist.is_valid_bist_ticker` guard endpoints; `delisted_tickers` Redis set is skipped.
7. **Login is form-encoded** (OAuth2PasswordRequestForm); register and everything else is JSON.
8. **Simulation cost** = `days * simulation.per_day_cost` (0.005), rounded 3; report estimate = `ceil(tokens/1000 * token_cost_per_1k)` where quick max 5000 / deep max 50000 tokens, cost 0.05 per 1k.
9. **Economy values are strings:** `Buying`/`Selling` use Turkish comma decimals (`"40,25"`), `Change` is `"%0,42"` — parse with `.replace(",", ".")`.
10. **Portfolio transactions blocked when market closed** (`400 "Market is closed"`); market hours 10:00–18:10 Europe/Istanbul, weekdays, non-holiday (`services/market.py`).
11. **Quote semantics:** `change_pct` is `null` when market open but no intraday data; `is_stale` > 20 min (open) / 3 days (closed).
12. **Redis down-tolerant:** all `r.*` calls may return `None`; code must fall back to DB and never crash on cache misses.
13. **pydantic-ai report agent** returns `ReportDraft` (title/about/date/report/sentiments); token usage logged to `token_usage`; reports INSERT then `track_event("report_generated", ...)` fire-and-forget.
14. **Exports:** background task runs in a fresh `contextvars.Context()`; if the event loop is gone the record stays `queued` (idempotent re-queue by design).

---

## 9. Config & Env Quick Reference

- Config: `src/core/config.py` `_DEFAULTS` dict; override via `<SECTION>_<KEY>` env vars (e.g. `REPORT_TOKEN_COST_PER_1K`, `ECONOMY_CACHE_TTL`, `MACROECONOMY_CACHE_TTL`, `PRICE_HISTORY_CACHE_TTL_HOT`, `SIMULATION_PER_DAY_COST`, `STOCK_VECTOR_TTL`). `get_config()` / `reload_config()` / `init_config()`.
- Required env: `SECRET_KEY`, `FRED_API_KEY` (import-time), `ADMIN_TOKEN` (admin app), `GOOGLE_APPLICATION_CREDENTIALS` (news), `RESEND_API_KEY` or `MAIL_*` (email).
- Optional: `ENVIRONMENT` (production vs development), `FREE_CREDIT_MAX` (25), `DAILY_FREE_CREDIT_REFILL` (5), `REFRESH_TOKEN_TTL_DAYS` (30), `PORTFOLIO_COMMISSION_RATE` (0.001), `LOG_DIR`, `PUBLIC_BASE_URL`, `NEWS_SEARCH_URL`, `EMBEDDING_*`, `CUSTOM_URL`/`CUSTOM_MODEL`/`OPENROUTER_*` (LLM), `EVDS_API_KEY`, `LOGODEV_API_KEY`, DB/Redis host/port vars.
- Cache TTLs: companies 30d; company profile 24h; price history 60s (hot 7d, intraday 30s); economy 20min (60s on error); macro 24h; IPOs 1h; market status 60s; news 1h; stock vectors 24h; frozen-user 30s; pwd-changed 60s.

---

## 10. Adding a New Endpoint (checklist)

1. Create the handler in the matching `src/api/<feature>.py` (or a new router file) with `async def`; add Pydantic body models in the same file.
2. Add auth: `current_user_id: int = Depends(get_current_user)` for JWT; `get_current_user_full` if you need `user_type` for admin boosts; `verify_admin_token` only for the admin app.
3. Add guards: `require_feature("...")` (add feature name to `services/maintenance._FEATURES`), `require_job_slot(kind, ttl)` for long jobs, `validate_ticker` for ticker params, `rate_limiter.check(...)` where needed.
4. Register the router in `src/api/router.py` (`router.include_router(...)`).
5. If the path must be public, add it to `PUBLIC_PATHS` in `src/main.py`.
6. Persist via `src/services/` (never raw SQL in handlers unless trivial CRUD — auth/announcements/exports are existing inline exceptions); follow async DB/Redis conventions.
7. If schema changes: update `init_db()` + migration file. If costs/credits involved: use `credits.spend/refund` and the estimate-then-settle pattern.
8. Regenerate `openapi.json` (see README) and update `docs/` when done.
