# Florence API Specification

> **English** — [Türkçe versiyon için tıklayın (Click for Turkish)](./README-tr.md)

> **Current API documentation** — generated from the live FastAPI application in `florence/backend`.

This repository holds the next-generation API documentation extracted from the **single source of truth — the backend codebase** (`src/api/`, `src/services/`, `src/core/`). Legacy documents (`openapi.json` / `API.md`) were removed in favor of a code-synchronized current version.

---

## 📁 File Index

| File | Language | Content |
|---|---|---|
| [`openapi.json`](./openapi.json) | OpenAPI 3.x (JSON) | FastAPI `app.openapi()` output — **generated from the live app** (89 paths, 23 schemas). |
| [`docs/api-reference-tr.md`](./docs/api-reference-tr.md) | 🇹🇷 Türkçe | **Full API reference** — all endpoints, parameters, request/response schemas, example JSON, auth flows, rate limits, error formats, data models, cache/TTL rules, environment variables. |
| [`docs/api-reference-en.md`](./docs/api-reference-en.md) | 🇬🇧 English | **Full API reference** — same scope as the Turkish version, one-to-one. |
| [`docs/ai-context.md`](./docs/ai-context.md) | 🇬🇧 English | **AI/LLM context pack**: single-file compact context so an AI agent can understand the backend and add new endpoints/features correctly. |

---

## 🚀 Quick Start

```bash
# All endpoints in a single command (the ones not requiring auth):
curl http://localhost:7055/api/v1/bist/tickers?limit=3

# Auth flow (register → verify → login → refresh):
curl -X POST http://localhost:7055/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","email":"demo@example.com","password":"supersecret123"}'

curl -X POST http://localhost:7055/api/v1/auth/login \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=demo&password=supersecret123'
# → {"access_token":"...","refresh_token":"...","token_type":"bearer"} + httpOnly cookies

curl http://localhost:7055/api/v1/profile \
  -H 'Authorization: Bearer <access_token>'
```

Base URL: `http://localhost:7055` (in production defined by the `PUBLIC_BASE_URL` env var). All feature endpoints live under the `/api/v1` prefix.

---

## 🔐 Authentication (summary)

- **Access token**: JWT (HS256, `SECRET_KEY`), payload `{user_id, iat, exp}`, **1 hour** lifetime. Accepted via `Authorization: Bearer <token>` header **or** the `access_token` httpOnly cookie.
- **Refresh token**: `secrets.token_urlsafe(48)`, stored as SHA-256 hash in DB, **rotated on every refresh**, TTL 30 days (`REFRESH_TOKEN_TTL_DAYS`). Sent in body or via the `refresh_token` cookie (path `/api/v1/auth`).
- **Passwords**: Argon2. Unverified-email accounts cannot log in or refresh (bot accounts exempt).
- Full flow: see `docs/api-reference-en.md` → "Authentication Flows".

---

## 🧭 Endpoint Map

| Tag / Area | Endpoint count | Examples |
|---|---|---|
| Auth & Users | 15 | `/auth/register`, `/auth/login`, `/profile`, `/credits`, `/user/preferences` |
| BIST / Companies / Prices | 9 | `/bist/companies`, `/companies/info/{ticker}`, `/price/history/{ticker}` |
| Reports | 6 | `/reports/generate`, `/reports/history`, `/reports/download` |
| Simulations | 5 | `/simulations/{ticker}`, `/simulations/estimate-cost/{ticker}` |
| Economy / Macro / IPO | 10 | `/economy/gold-prices`, `/macroeconomy`, `/ipos/upcoming` |
| Virtual Portfolios | 21 | `/portfolios`, `/portfolios/{id}/valuation`, `/portfolios/{id}/risk` |
| Favorites | 3 | `/favorites`, `/favorites/{ticker}` |
| Bots | 3 | `/bots`, `/bots/{bot_id}` |
| Data Export | 6 | `/user/export`, `/data/export`, `/data/export/download/{token}` |
| Announcements | 5 | `/announcements` (+ CRUD, admin write) |
| Advisor (stock fit) | 2 | `/stocks/fit`, `/portfolio/profile` |
| Other public | 11 | `/market/status`, `/legal`, `/meta/avatars`, `/maintenance`, `/version` |
| Admin (separate app) | 5 | `/gift-credits`, `/healthcheck`, `/maintenance/toggle` |

---

## 🔄 How to Regenerate openapi.json

The `openapi.json` file is regenerated from the backend repo root whenever the code changes:

```bash
python3 -m venv /tmp/api-spec-venv && /tmp/api-spec-venv/bin/pip install -r requirements.txt
cd /home/efe/Belgeler/florence/backend
export FRED_API_KEY=dummy SECRET_KEY=dummy ADMIN_TOKEN=dummy ENVIRONMENT=development LOG_DIR=/tmp/florence-logs
/tmp/api-spec-venv/bin/python -c "
import json
from src.main import app
with open('/home/efe/Belgeler/florence/api-spec/openapi.json', 'w') as f:
    json.dump(app.openapi(), f, indent=2, ensure_ascii=False)
"
```

> Note: without `FRED_API_KEY` the `src/clients/macroeconomy.py` module raises `ValueError` at import time; a dummy value must be exported. `LOG_DIR` must point to a writable directory (`/var/log/florence` requires root).

---

## 📄 License

This documentation is part of the Florence project and is subject to the license terms of the backend repository.
