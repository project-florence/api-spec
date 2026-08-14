# Florence API Specification

> **Türkçe** — [For the English version, click here](./README-en.md)

> **Güncel API dokümantasyonu** — Florence backend'in (`florence/backend`) canlı FastAPI uygulamasından üretilmiştir.

Bu repo, Florence platformunun **tek doğruluk kaynağı olan backend kodundan** (`src/api/`, `src/services/`, `src/core/`) çıkarılmış yeni nesil API dokümanlarını barındırır. Eski `openapi.json` / `API.md` / legacy dokümanlar kaldırılmış, yerine kodla senkronize güncel sürüm konmuştur.

---

## 📁 Dosya İndeksi

| Dosya | Dil | İçerik |
|---|---|---|
| [`openapi.json`](./openapi.json) | OpenAPI 3.x (JSON) | FastAPI `app.openapi()` çıktısı — **gerçek uygulamadan üretildi** (89 path, 23 schema). |
| [`docs/api-reference-tr.md`](./docs/api-reference-tr.md) | 🇹🇷 Türkçe | **Tam API referansı**: tüm endpoint'ler, parametreler, request/response şemaları, örnek JSON, auth akışları, rate limit'ler, hata formatları, veri modelleri, cache/TTL kuralları, ortam değişkenleri. |
| [`docs/api-reference-en.md`](./docs/api-reference-en.md) | 🇬🇧 English | **Full API reference** — Türkçe versiyonla birebir aynı kapsamda. |
| [`docs/ai-context.md`](./docs/ai-context.md) | 🇬🇧 English | **AI/LLM bağlam paketi**: tek dosyalık kompakt bağlam; bir AI ajanının backend'i anlayıp yeni endpoint/özellik doğru ekleyebilmesi için. |

---

## 🚀 Hızlı Başlangıç

```bash
# Tüm uç noktalar tek komutla (auth gerektirmeyenler):
curl http://localhost:7055/api/v1/bist/tickers?limit=3

# Auth akışı (register → verify → login → refresh):
curl -X POST http://localhost:7055/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"username":"demo","email":"demo@example.com","password":"supersecret123"}'

curl -X POST http://localhost:7055/api/v1/auth/login \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'username=demo&password=supersecret123'
# → {"access_token":"...","refresh_token":"...","token_type":"bearer"} + httpOnly cookie'ler

curl http://localhost:7055/api/v1/profile \
  -H 'Authorization: Bearer <access_token>'
```

Base URL: `http://localhost:7055` (production'da `PUBLIC_BASE_URL` env'i ile tanımlanır). Tüm feature endpoint'leri `/api/v1` prefix'i altındadır.

---

## 🔐 Kimlik Doğrulama (özet)

- **Access token**: JWT (HS256, `SECRET_KEY`), payload `{user_id, iat, exp}`, geçerlilik **1 saat**. `Authorization: Bearer <token>` header'ı **veya** `access_token` httpOnly cookie kabul edilir.
- **Refresh token**: `secrets.token_urlsafe(48)`, DB'de SHA-256 hash ile saklanır, **her refresh'te rotasyon** yapılır, TTL 30 gün (`REFRESH_TOKEN_TTL_DAYS`). Body veya `refresh_token` cookie (path `/api/v1/auth`) ile gönderilir.
- **Şifreler**: Argon2. E-posta doğrulanmamış hesaplar login/refresh yapamaz (bot hesapları istisna).
- Detaylı akış: bkz. `docs/api-reference-tr.md` → "Kimlik Doğrulama Akışları".

---

## 🧭 Endpoint Dağılımı

| Tag / Alan | Endpoint sayısı | Örnekler |
|---|---|---|
| Auth & Kullanıcı | 15 | `/auth/register`, `/auth/login`, `/profile`, `/credits`, `/user/preferences` |
| BIST / Şirket / Fiyat | 9 | `/bist/companies`, `/companies/info/{ticker}`, `/price/history/{ticker}` |
| Raporlar | 6 | `/reports/generate`, `/reports/history`, `/reports/download` |
| Simülasyonlar | 5 | `/simulations/{ticker}`, `/simulations/estimate-cost/{ticker}` |
| Ekonomi / Makro / IPO | 10 | `/economy/gold-prices`, `/macroeconomy`, `/ipos/upcoming` |
| Sanal Portföyler | 21 | `/portfolios`, `/portfolios/{id}/valuation`, `/portfolios/{id}/risk` |
| Favoriler | 3 | `/favorites`, `/favorites/{ticker}` |
| Botlar | 3 | `/bots`, `/bots/{bot_id}` |
| Veri Dışa Aktarım | 6 | `/user/export`, `/data/export`, `/data/export/download/{token}` |
| Duyurular | 5 | `/announcements` (+ CRUD, admin yazma) |
| Danışman (stock fit) | 2 | `/stocks/fit`, `/portfolio/profile` |
| Diğer public | 11 | `/market/status`, `/legal`, `/meta/avatars`, `/maintenance`, `/version` |
| Admin (ayrı app) | 5 | `/gift-credits`, `/healthcheck`, `/maintenance/toggle` |

---

## 🔄 openapi.json Nasıl Üretilir

`openapi.json` dosyası kod değiştiğinde şöyle yeniden üretilir (backend repo kökünde):

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

> Not: `FRED_API_KEY` yoksa `src/clients/macroeconomy.py` import anında `ValueError` fırlatır; sahte değer export etmek şarttır. `LOG_DIR` de yazılabilir bir dizine işaret etmelidir (`/var/log/florence` root ister).

---

## 📄 Lisans

Bu dokümantasyon Florence projesinin bir parçasıdır; backend repo'sundaki lisans koşullarına tabidir.
