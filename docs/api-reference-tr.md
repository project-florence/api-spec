# Florence API Referansı (Türkçe)

> **Kaynak:** `/home/efe/Belgeler/florence/backend` — canlı FastAPI uygulaması (sürüm: `src/version.py`).
> **Üretim:** 2026-08-14 — tüm endpoint'ler `src/api/` router dosyalarından kod düzeyinde doğrulanmıştır; şemalar Pydantic modellerinden birebir alınmıştır.
> **Eşleşen İngilizce doküman:** [`api-reference-en.md`](./api-reference-en.md) (birebir kapsam).

---

## 1. Genel Bakış

Florence; Türk borsası (BIST) odaklı bir finans platformunun API sunucusudur:

- **Piyasa verisi:** yfinance üzerinden hisse profilleri ve mum (candle) geçmişi, pykap ile BIST şirket listesi, GenelPara ile döviz/altın/emtia anlık fiyatları, FRED ile makroekonomik göstergeler.
- **Haber analizi:** SearXNG + GDELT/BigQuery üzerinden hisse haberleri; LLM agent ile otomatik yatırım raporları (`quick_report` / `deep_report`).
- **Simülasyon:** Monte Carlo fiyat simülasyonu (hedef fiyat üstü/altı olasılık, güven aralığı).
- **Sanal portföy:** JSONB tabanlı, `pg_advisory_xact_lock` ile eşzamanlılık korumalı sanal alım-satım (işlem, değerleme, risk, benchmark vb.).
- **Kullanıcı yönetimi:** JWT auth (access + refresh rotasyonu), e-posta doğrulama, Argon2 şifreleme, kredi sistemi (free/gift kredi), bot hesapları, duyurular, veri dışa aktarım (takeout tarzı).

**Mimari:** tamamen asenkron (async) — FastAPI 0.139 + psycopg3 `AsyncConnectionPool` + redis.asyncio. HTTP katmanı `src/api/`, iş mantığı `src/services/`, dış entegrasyonlar `src/clients/`, altyapı `src/core/` içindedir. ORM yoktur; ham SQL + Pydantic kullanılır.

**Base URL:** `http://localhost:7055` (geliştirme varsayılanı; production'da `PUBLIC_BASE_URL`). Tüm feature endpoint'leri **`/api/v1`** prefix'i altındadır.

**Dokümanlar:** `/docs` (Swagger UI), `/redoc`, `/openapi.json` — yalnızca production dışında (`ENVIRONMENT != production`) açıktır.

---

## 2. Kimlik Doğrulama Mekanizması

### 2.1 Access token (JWT)

- Algoritma: **HS256**, imza anahtarı: `SECRET_KEY` (env, zorunlu — yoksa uygulama açılmaz).
- Payload: `{ "user_id": int, "iat": ..., "exp": ... }` — geçerlilik **1 saat**.
- Gönderim: `Authorization: Bearer <token>` header **veya** `access_token` httpOnly cookie (`samesite=strict`, `secure` production'da, max-age 3600, path `/`).
- Doğrulama: `src/api/deps.py:get_current_user` — header veya cookie'den okur, JWT decode eder, ek kontroller yapar (aşağıda).

### 2.2 Ek token kontrolleri (deps.py)

| Kontrol | Mekanizma |
|---|---|
| Şifre değişikliği | `password_changed_at > token.iat` ise token geçersiz sayılır (Redis `user:pwd_changed:*` cache, 60s TTL). |
| Dondurulmuş hesap | `is_frozen` kullanıcılar geçerli token ile bile erişemez (Redis `user:frozen:*` cache, 30s TTL). |
| E-posta doğrulaması | Login/refresh sırasında kontrol edilir; doğrulanmamış hesaplar giriş yapamaz (bot istisna). |

### 2.3 Refresh token

- Üretim: `secrets.token_urlsafe(48)`; DB'de `refresh_tokens` tablosunda **SHA-256 hash** ile saklanır.
- **Rotasyon:** her `/auth/refresh` çağrısında eski token revoke edilir, yenisi üretilir.
- TTL: 30 gün (`REFRESH_TOKEN_TTL_DAYS`).
- Gönderim: request body `{"refresh_token": "..."}` **veya** `refresh_token` cookie (path `/api/v1/auth`).
- İptal: `/auth/logout` (tek token), şifre/e-posta/kullanıcı adı değişikliği (tüm kullanıcı token'ları).

### 2.4 Middleware (main.py: `auth_and_tracking_middleware`)

- `/api/v1/*` altındaki her istek için: path **public listesinde değilse** JWT doğrulanır; geçersizse `401 {"detail": "Not authenticated"}`.
- Public path'ler: `/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/logout`, `/auth/verify-email`, `/auth/resend-verification`, `/market/status`, `/data/export/download`, `/meta/avatars`, `/legal`, `/about`, `/contact`, `/version`, `/maintenance`, `/contributors`, `/`, `/health`, `/docs`, `/redoc`, `/openapi.json`. (`startswith` eşleşmesi: `/api/` ile başlayan public path'ler prefix olarak da eşleşir.)
- `OPTIONS` (CORS preflight) muaf.
- DB havuzu tükenirse (`PoolTimeout`): `503 {"detail": "Database busy, please retry"}` (401 değil — frontend'in refresh/logout zincirine girmemesi için).
- Her istek sonrası `db.release_current()` çağrılır (bağlantı sızıntısı önleme).
- Analytics: `/api/v1/analytics/event` dışındaki tüm özel istekler `api_request` olayı olarak fire-and-forget kaydedilir.
- CORS: production'da yalnızca Tauri desktop origin'leri (`tauri://localhost`, `http://tauri.localhost`, `https://tauri.localhost`); geliştirmede `*`.

### 2.5 Admin kimliği

- Ayrı `X-Admin-Token` header (sabit `ADMIN_TOKEN` env'i, string eşitliği). `src/api/deps.py:verify_admin_token`.
- Admin kullanıcılar rate limit'te **10x** kota alır (`get_current_user_full` ile `user_type` okunur; news ve export endpoint'lerinde uygulanır).

---

## 3. Kimlik Doğrulama Akışları (Adım Adım)

### Akış A — Klasik kayıt → giriş

1. **Kayıt:** `POST /api/v1/auth/register` — `{username, email, password(≥10)}` → `201`-benzeri yanıt: `{message, user_id, verification_sent}`. Argon2 hash alınır; 24 saat geçerli e-posta doğrulama token'ı üretilir ve mail gönderilir (mail hatası kaydı bozmaz: `verification_sent: false`). Hata kodları: `error_username_taken`, `error_email_taken` (400).
2. **E-posta doğrulama:** `GET /api/v1/auth/verify-email?token=...` — token eşleşir ve süresi dolmamışsa `email_verified = TRUE`. Yanıt: `{message: "Email verified", email_verified: true}`. Hatalı/süresi dolmuş token: 400 `"Invalid or expired verification token"`.
3. **Giriş:** `POST /api/v1/auth/login` — OAuth2 form (`application/x-www-form-urlencoded`, alanlar: `username`, `password`; username **veya** email kabul edilir). Argon2 doğrulama; doğrulanmamış e-posta → 403 `error_email_not_verified`. Başarılıysa:
   - Yanıt gövdesi: `{"access_token", "refresh_token", "token_type": "bearer"}`
   - Ek olarak `access_token` (path `/`, 1s) ve `refresh_token` (path `/api/v1/auth`, 30g) httpOnly cookie'leri set edilir.
   - `last_login` güncellenir.
4. **Token yenileme:** `POST /api/v1/auth/refresh` — body `{"refresh_token"}` veya cookie. Rotasyon yapılır; yeni access+refresh döner (cookie'ler de yenilenir). E-posta doğrulanmamışsa 403. Geçersiz token: 401 `"Invalid or expired refresh token"`.
5. **Çıkış:** `POST /api/v1/auth/logout` — body `{"refresh_token"}` veya cookie; token revoke edilir, cookie'ler temizlenir. `{message: "Logged out"}`.

### Akış B — Doğrulama maili yeniden gönderme

- `POST /api/v1/auth/resend-verification` — `{username_or_email}` (username **veya** email). Hesap yoksa 404; zaten doğrulanmışsa 400 `"Email already verified"`; yeni token (24s) üretilir + mail gider. Yanıt: `{verification_sent: bool}`.

### Akış C — Hesap değişiklikleri (JWT gerekli)

- `PUT /api/v1/auth/change-password` — `{current_password, new_password(≥10)}` → şifre değişir, `password_changed_at` güncellenir, **tüm** refresh token'lar revoke edilir. Yanlış mevcut şifre: 400.
- `PUT /api/v1/auth/change-email` — `{new_email, current_password}` → e-posta değişir, tüm token'lar revoke edilir.
- `PUT /api/v1/auth/change-username` — `{new_username, current_password}` → kullanıcı adı değişir, tüm token'lar revoke edilir.
- `DELETE /api/v1/auth/delete` — hesap kalıcı silinir.

### Akış D — Bot hesabı (API erişimi için)

- `POST /api/v1/bots` — `{username, password?(≥10)}` → `user_type='bot'`, e-posta `<username>@bot.florencex.com.tr`, `email_verified=TRUE` (doğrulama gerekmez). **Şifre yalnızca yanıtta bir kez döner.** Bot'lar owner'ının kredisinden harcar (`credits._resolve_owner`). Limit: kullanıcı başına 5 bot. Bot hesap bot açamaz (403 `error_bots_not_allowed`).

---

## 4. Rate Limit Kuralları

Uygulama: `src/core/ratelimit.py` — **karma model**: Redis varsa `INCR + EXPIRE` (sabit pencere), Redis yoksa proses-içi bellek kovası (aynı limit, down-tolerant). Aşım yanıtı: `429 {"detail": "Too many requests. Please slow down."}`.

| Endpoint | Anahtar | Limit | Admin |
|---|---|---|---|
| `POST /auth/register` | `register:{client_ip}` | 3 istek / 60 sn | — |
| `POST /auth/login` | `login:{username}` | 5 istek / 60 sn | — |
| `POST /auth/refresh` | `refresh:{token_hash}` | 5 istek / 60 sn | — |
| `POST /auth/resend-verification` | `resend-verif:{account}` | 3 istek / 3600 sn | — |
| `GET /news/{ticker}` | `news:{user_id}:{ticker}` | 10 istek / 60 sn | 100 / dk (10x) |
| `POST /data/export` | `export:{user_id}` | 3 istek / 3600 sn | 30 / saat (10x) |

Genel (her endpoint'e uygulanan) bir global limit **yoktur**. Ayrıca job-slot mekanizması (`src/core/job_slots.py`) aynı kullanıcının aynı tür işi eşzamanlı çalıştırmasını engeller:

| İş | Slot anahtarı | TTL / lease |
|---|---|---|
| Rapor üretimi | `job-slot:report:{user_id}` | 900 sn (lease 120 sn, heartbeat 30 sn) |
| Simülasyon | `job-slot:simulation:{user_id}` | 600 sn (lease 120 sn, heartbeat 30 sn) |

Slot doluysa: `429 {"detail": "A job of this type is already running"}`.

---

## 5. Hata Formatları

Tüm hatalar FastAPI standart yapısındadır: `{"detail": <string | list>}`.

| Durum | Anlam | Örnek `detail` |
|---|---|---|
| 400 | Geçersiz istek / iş kuralı | `"Invalid type"`, `"error_username_taken"`, `"Market is closed"` |
| 401 | Kimlik doğrulama hatası | `"Invalid or expired token"`, `"Not authenticated"` (middleware), `"Invalid or expired refresh token"` |
| 402 | Yetersiz kredi | `"insufficient credit"` (rapor/simülasyon) |
| 403 | Yetki / hesap durumu | `"error_email_not_verified"`, `"Admin access required"`, `"Invalid admin token"` |
| 404 | Kaynak yok | `"Invalid BIST ticker: X"`, `"Report not found"`, `"User not found"` |
| 409 | Çakışma (benzersizlik) | `"error_username_taken"` (bot oluşturma) |
| 410 | Kalıcı olarak kaldırıldı | `GET /data/daily/{year}` — `"Kullanım dışı — POST /api/v1/data/export ile istek oluşturun"`; export linki süresi dolmuşsa `"Export link expired or not ready"` |
| 413 | Çok fazla öğe | `"Too many events"` (analytics, >100) |
| 422 | Pydantic doğrulama | Standart FastAPI `HTTPValidationError` |
| 429 | Rate limit / job slot | `"Too many requests. Please slow down."`, `"A job of this type is already running"` |
| 500 | Sunucu hatası | `"Database error"`, `"Report generation failed"` |
| 503 | Bakım / havuz tükenmesi | `"Database busy, please retry"`, `"{feature} is temporarily disabled for maintenance"` |

**i18n hata kodları:** Frontend'in çevirmesi için bazı hatalar kısa kod olarak döner: `error_username_taken`, `error_email_taken`, `error_login_failed`, `error_email_not_verified`, `error_bot_limit_reached`, `error_bots_not_allowed`.

---

## 6. Veri Modelleri / Entity'ler

Şema kaynağı: `src/core/database.py:init_db()` (runtime'da `CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` ile idempotent kurulur; `migrations/` dosyaları otomatik **uygulanmaz**).

| Tablo | Önemli kolonlar |
|---|---|
| `users` | `id`, `username` UNIQUE, `email` UNIQUE, `hashed_pw`, `user_type` (`user`/`bot`/`admin`), `is_frozen`, `email_verified`, `email_verify_token`(+`email_verify_expires_at`), `avatar_id`, `owner_id` (bot→owner), `password_changed_at`, `last_login`, `last_announcement_viewed_at` |
| `user_credits` | `(user_id, credit_type)` PK — `credit_type`: `free_credits` / `gift_credits`; `amount` |
| `tickers` / `companies` | BIST kodları + şirket profilleri (pykap ile doldurulur) |
| `ticker_stats` | `ticker` PK; `info_count`, `report_count`, `news_count`, `history_count`, `simulation_count`, `favorite_count`, `updated_at` |
| `favorites` | `(user_id, ticker_code)` PK |
| `reports` | `id`, `user_id`, `ticker`, `type` (`quick_report`/`deep_report`), `title`, `token_usage` JSONB, `content` TEXT, `sentiments` JSONB, `purpose`, `created_at` |
| `simulations` | `id`, `user_id`, `ticker`, `days`, `bounds`, `target`, `result` JSONB, `cost`, `created_at` |
| `price_candles` | `(ticker, interval, ts)` PK; `open/high/low/close` DOUBLE, `volume` BIGINT; `idx_price_candles_lookup` (ticker, interval, ts DESC). Ticker formatı `XXX.IS` |
| `economy_rates` | `(ticker, ts)` PK, `price` JSONB (döviz/altın/emtia tarihçesi) |
| `market_rates` | `data_type` + `data` JSONB (anlık piyasa yedek verisi) |
| `macroeconomy` | 14 FRED göstergesi (gdp, fed_funds, vix, sp500, btc vb.) |
| `portfolios` | `portfolio_id` TEXT PK (`port-<uuid>`), `user_id`, `portfolio` JSONB (tüm portföy tek JSON) |
| `announcements` | `id`, `title`, `content`, `sent_by`, `created_at`, `updated_at` |
| `cron_jobs` | `name` PK, `source`, `interval_ms`, `last_run` |
| `refresh_tokens` | `token_hash` UNIQUE (SHA-256), `user_id`, `device`, `expires_at`, `revoked_at` |
| `analytics_events` | `event_type`, `user_id`, `session_id`, `ticker`, `details` JSONB |
| `stock_vectors` | `ticker` PK, `risk`, `horizon`, `profitability` |
| `token_usage` | `model`, `prompt_tokens`, `completion_tokens`, `total_tokens`, `endpoint`, `user_id`, `created_at` |
| `user_preferences` | `user_id` PK, `prefs` JSONB (trigger `create_default_prefs()` kayıtta default pref üretir) |
| `exports` | `year`, `format` (csv/json), `status` (queued/processing/ready/sent/failed), `file_path`, `token`, `expires_at`, `downloaded_count`, `row_count`, `size_bytes`, `error`, `created_at`, `updated_at` |

**Not:** `users.credits` kolonu eski ortamlarda varsa `user_credits`'e taşınır ve drop edilir (init_db içinde veri taşıma migration'ı). `economy_rates.price` DOUBLE PRECISION olarak yaratılmış eski ortamlarda JSONB'e çevrilir (self-heal).

### Pydantic request şemaları (openapi.json `components.schemas`)

| Şema | Alanlar |
|---|---|
| `UserRegister` | `username: str`, `email: EmailStr`, `password: str (minLength 10)` — hepsi zorunlu |
| `ChangePassword` | `current_password: str`, `new_password: str (min 10)` |
| `UpdateEmail` | `new_email: EmailStr`, `current_password: str` |
| `UpdateUsername` | `new_username: str`, `current_password: str` |
| `RefreshRequest` | `refresh_token: str \| null` (opsiyonel — cookie'den de okunur) |
| `ResendVerification` | `username_or_email: str` |
| `AvatarUpdate` | `avatar_id: str` (`avatar-1` … `avatar-12`) |
| `PreferencesUpdate` | `prefs: dict` (JSONB, merge edilir) |
| `BotCreate` | `username: str (3-255)`, `password: str\|null (min 10)` |
| `FitRequest` | `horizon: str` (short/medium/long, vars. `long`), `profitability` (low/medium/high, vars. `high`), `risk_tolerance` (low/medium/high, vars. `medium`), `limit: int (1-100, vars. 5)` |
| `PortfolioProfileRequest` | `tickers: list[str] (1-50, uppercase normalize)`, `limit: int (1-50, vars. 5)` |
| `CreatePortfolioBody` | `name: str (1-255)`, `initial_balance: float (>0)` |
| `RenamePortfolioBody` / `DuplicatePortfolioBody` | `name: str (1-255)` |
| `AddTransactionBody` | `ticker: str (min 1)`, `type: str (pattern ^(BUY\|SELL)$)`, `quantity: float (>0)` |
| `UpdateTransactionBody` | `price: float\|null (>0)`, `quantity: float\|null (>0)` — ikisi de opsiyonel |
| `AnnouncementCreate` / `AnnouncementUpdate` | `title: str`, `content: str` |
| `ExportRequest` | `year: int (1990..şimdiki yıl+1)`, `format: "csv"\|"json"` (vars. csv) |
| `ReportHistoryItem` (response) | `id: int`, `ticker: str`, `type: str`, `title: str\|null`, `token_usage: dict\|null`, `purpose: str\|null`, `created_at: str` |

---

## 7. Cache / TTL Kuralları

Redis proxy (`src/core/redis.py`) **down-tolerant**: bağlantı kurulamazsa tüm çağrılar `None` döner, 60s cooldown sonrası yeniden dener (cache'siz mod — DB'ye düşülür).

| Veri | Redis anahtarı | TTL |
|---|---|---|
| BIST şirket/ticker listesi | `tickers`, `companies` JSON | 30 gün |
| Fiyat geçmişi | `price:history:{ticker}:{period}:{interval}` (hot: `cache_ttl_hot`) | normal 60 sn; hot 7 gün; intraday 30 sn |
| Company profili | `{TICKER}.IS` | 24 saat |
| Ekonomi (GenelPara) | `gold_prices`, `silver_price`, `gram_platinum_price`, `gram_palladium_price`, `currency`, `genelpara:*` | 20 dk (1200 sn); hata durumunda 60 sn |
| Makro (FRED) | `macroeconomy` | 24 saat |
| IPO listeleri | `ipos:*` | 1 saat |
| Market durumu | `market:status` | 60 sn |
| Haberler | `news:{TICKER}:{amount}` (10'a yuvarlanır) | 1 saat |
| Stock vector'lar | `stock_vectors:*` | 24 saat |
| Frozen kullanıcı | `user:frozen:{user_id}` | 30 sn |
| Şifre değişikliği | `user:pwd_changed:{user_id}` | 60 sn |
| Rate limit sayaçları | `ratelimit:{key}` | pencere süresi |
| Kilitler | `lock:cron:{tier}` (NX+TTL), `refresh_lock:{ticker}:{interval}` (15s) | değişken |
| Job slot lease | `job-slot:{kind}:{user_id}` | 120 sn (heartbeat 30 sn) |
| Set'ler | `delisted_tickers`, `maintenance:disabled` | kalıcı |

---

## 8. Endpoint Referansı

> Kısaltmalar: 🔓 = public (auth gerekmez), 🔐 = JWT gerekli, ⚙ = feature kill-switch'e tabi, ⏳ = job slot'a tabi.
> Tüm path'ler `/api/v1` prefix'i altındadır. Auth yöntemi: `Authorization: Bearer <access_token>` header veya `access_token` cookie.

### 8.1 Auth & Kullanıcı (`src/api/auth.py`)

#### `POST /auth/register` 🔓
Kullanıcı kaydı. E-posta doğrulama token'ı (24 saat) üretir ve mail gönderir.

- **Body:** `{"username": str, "email": str, "password": str (min 10)}`
- **Rate limit:** 3/dk/IP
- **Yanıt 200:**
```json
{ "message": "Register successful", "user_id": 42, "verification_sent": true }
```
- **Hatalar:** 400 `"error_username_taken"` / `"error_email_taken"`; 500 `"Database error"`

#### `POST /auth/login` 🔓
OAuth2 form girişi — `application/x-www-form-urlencoded`, alanlar: `username` (kullanıcı adı **veya** e-posta), `password`.

- **Rate limit:** 5/dk/username
- **Yanıt 200:** body + httpOnly cookie'ler (`access_token` path `/`, `refresh_token` path `/api/v1/auth`)
```json
{ "access_token": "<jwt>", "refresh_token": "<opaque>", "token_type": "bearer" }
```
- **Hatalar:** 400 `"error_login_failed"`; 403 `"error_email_not_verified"` (doğrulanmamış hesap, bot hariç)

#### `GET /auth/verify-email` 🔓
E-posta doğrulama.

- **Query:** `token: str` (zorunlu)
- **Yanıt 200:** `{"message": "Email verified", "email_verified": true}`
- **Hatalar:** 400 `"Invalid or expired verification token"`

#### `POST /auth/resend-verification` 🔓
Doğrulama mailini yeniden gönderir.

- **Body:** `{"username_or_email": str}`
- **Rate limit:** 3/saat/account
- **Yanıt 200:** `{"verification_sent": bool}`
- **Hatalar:** 404 `"User not found"`; 400 `"Email already verified"`

#### `POST /auth/refresh` 🔓
Refresh token rotasyonu + yeni access token.

- **Body (opsiyonel):** `{"refresh_token": str}` — yoksa cookie'den okunur
- **Rate limit:** 5/dk/token hash
- **Yanıt 200:** `{"access_token", "refresh_token", "token_type": "bearer"}` + cookie'ler yenilenir
- **Hatalar:** 401 `"Invalid or expired refresh token"`; 403 `"error_email_not_verified"`

#### `POST /auth/logout` 🔓
Refresh token'ı revoke eder, cookie'leri temizler.

- **Body:** `{"refresh_token": str}` (cookie de kabul edilir)
- **Yanıt 200:** `{"message": "Logged out"}`

#### `DELETE /auth/delete` 🔐
Hesabı kalıcı siler.

- **Yanıt 200:** `{"message": "Deleted user {id}"}`
- **Hatalar:** 400 `"Database error"`

#### `PUT /auth/change-password` 🔐
Şifre değiştirir → `password_changed_at` güncellenir, tüm refresh token'lar revoke edilir.

- **Body:** `{"current_password": str, "new_password": str (min 10)}`
- **Yanıt 200:** `{"message": "Password changed successfully"}`
- **Hatalar:** 404 `"User not found"`; 400 `"Current password is incorrect"`

#### `PUT /auth/change-email` 🔐
E-posta değiştirir → tüm token'lar revoke edilir.

- **Body:** `{"new_email": str, "current_password": str}`
- **Yanıt 200:** `{"message": "Email changed successfully", "new_email": "..."}`
- **Hatalar:** 400 `"Email already in use"` / `"Current password is incorrect"`

#### `PUT /auth/change-username` 🔐
Kullanıcı adı değiştirir → tüm token'lar revoke edilir.

- **Body:** `{"new_username": str, "current_password": str}`
- **Yanıt 200:** `{"message": "Username changed successfully", "new_username": "..."}`
- **Hatalar:** 400 `"Username already in use"`

#### `GET /profile` 🔐
Profil + kredi bakiyesi.

- **Yanıt 200:**
```json
{
  "username": "demo", "email": "demo@example.com", "user_type": "user",
  "created_at": "2026-01-01T10:00:00+00:00", "email_verified": true,
  "avatar_id": "avatar-3", "credits": 18.5
}
```

#### `PUT /profile/avatar` 🔐
Avatar değiştirir.

- **Body:** `{"avatar_id": "avatar-1"}` … `"avatar-12"`
- **Yanıt 200:** `{"message": "Avatar updated", "avatar_id": "..."}`
- **Hatalar:** 400 `"Unknown avatar_id"`

#### `GET /credits` 🔐
Kredi bakiyesi.

- **Yanıt 200:** `{"credits": 18.5}`

#### `GET /user/preferences` 🔐
JSONB kullanıcı tercihleri.

- **Yanıt 200:** ham JSONB nesnesi (ör. `{"theme": "dark", "language": "tr"}`)
- **Hatalar:** 404 `"Preferences not found"`

#### `PUT /user/preferences` 🔐
Tercihleri günceller (mevcut prefs ile **merge** edilir).

- **Body:** `{"prefs": {"theme": "light"}}`
- **Yanıt 200:** birleştirilmiş JSONB nesnesi

### 8.2 BIST / Şirket / Fiyat (`src/api/bist.py`, `src/api/stats.py`)

#### `GET /bist/companies` 🔓
BIST şirket listesi (Redis'ten, 30 gün TTL).

- **Query:** `sort: "alphabetical"|"popular"` (vars. alphabetical), `offset: int ≥0` (vars. 0), `limit: int 1-500` (vars. 50)
- **Yanıt 200:** şirket dict listesi (`ticker`, `name`, …) — `popular` sıralamasında offset+limit kadar popüler şirket çekilip dilimlenir

#### `GET /bist/tickers` 🔓
Ticker listesi.

- **Query:** aynı: `sort`, `offset`, `limit`
- **Yanıt 200:** ticker string listesi (popüler sıralama için `get_popular_tickers`)

#### `GET /companies/search` 🔓
Şirket metin araması (alias'lar desteklenir: "IS BANK" → ISCTR).

- **Query:** `query: str` (zorunlu)
- **Yanıt 200:** `search_companies_by_text` sonucu

#### `GET /companies/info/{ticker}` 🔓
yfinance tabanlı yapılandırılmış şirket profili. `ticker` BIST kodu olmalı (`.IS` eklenir).

- **Path:** `ticker` (geçersizse 404 `"Invalid BIST ticker: X"`)
- **Yanıt 200:** profile dict — `symbol`, `name`, `sector`, `industry`, `currency`, `exchange`, `market{currentPrice, previousClose, marketCap, dayHigh, dayLow, regularMarketVolume, fiftyTwoWeekHigh, fiftyTwoWeekLow, regularMarketTime}`, `trading{beta, sharesOutstanding, floatShares, averageVolume, averageVolume10days, fiftyDayAverage, twoHundredDayAverage, shortRatio, heldPercentInsiders, heldPercentInstitutions}`, `valuation{trailingPE, forwardPE, pegRatio, priceToBook, priceToSalesTrailing12Months, enterpriseValue, enterpriseToEbitda, enterpriseToRevenue, bookValue, trailingEps, forwardEps, dividendYield, payoutRatio, targetMeanPrice, targetHighPrice, targetLowPrice, recommendationKey, numberOfAnalystOpinions}`, `financials{totalRevenue, revenuePerShare, revenueGrowth, grossProfits, grossMargins, ebitda, ebitdaMargins, netIncomeToCommon, profitMargins, operatingMargins, operatingCashflow, freeCashflow, earningsGrowth, earningsQuarterlyGrowth, returnOnEquity, returnOnAssets}`, `balanceSheet{totalCash, totalCashPerShare, totalDebt, debtToEquity, currentRatio, quickRatio}`, `recommendations: []`

#### `GET /companies/info/{ticker}/md` 🔓
Aynı profil, **markdown metni** olarak (`text/markdown; charset=utf-8`).

#### `GET /companies/summary` 🔓
Özet tablo.

- **Query:** `limit: int 1-500` (vars. 50), `offset: int ≥0`, `sort: popular|alphabetical|gainers|losers|price_high|price_low|volume|market_cap` (vars. popular), `tickers: str|null` (virgülle ayrılmış filtre)
- **Yanıt 200:** `{total, data: [{ticker, name, price, change_pct, ...}]}` yapısında özet

#### `GET /news/{ticker}` 🔐 ⚙(news)
Hisse haberleri (GDELT/BigQuery, son 90 gün, Türkçe).

- **Path:** `ticker`; **Query:** `amount: int 1-50` (vars. 10)
- **Rate limit:** 10/dk/kullanıcı+ticker (admin 100/dk)
- **Yanıt 200:** `[{url, title, lang, date}]` listesi (date ISO8601)

#### `GET /price/history/{ticker}` 🔓
Mum verisi (yfinance + `price_candles` DB tablosu).

- **Path:** `ticker`; **Query:** `period: "1d"|"5d"|"1mo"|"3mo"|"6mo"|"1y"|"2y"|"5y"|"10y"|"ytd"|"max"` (vars. 1mo), `interval: "5m"|"30m"|"1h"|"1d"|"5d"|"1wk"|"1mo"|"3mo"` (vars. 1d)
- **Kısıt:** interval başına maksimum gün: 5m→60, 30m→60, 1h→730, 1d/5d/1wk/1mo/3mo→3650. Aşılırsa 400.
- **Yanıt 200:** `[{ts, open, high, low, close, volume}]` listesi

#### `GET /price/current` 🔓
Anlık fiyat + değişim (quote mantığı).

- **Query:** `ticker: str` (zorunlu), `interval: "5m"|"30m"|"1h"|"1d"` (vars. 5m)
- **Yanıt 200:** `{ticker, price, previous_close, absolute_change, change_pct, as_of, previous_close_as_of, market_status ("open"|"closed"), is_stale, change_window, interval}`
- **Hatalar:** 404 `"Price not found"` (fiyat yoksa)

#### `GET /stats/top` 🔓
En çok işlem gören ticker'lar (ticker_stats toplamına göre).

- **Query:** `limit: int` (vars. 50)
- **Yanıt 200:** `[{ticker, name, info_count, report_count, news_count, history_count, simulation_count, favorite_count, total}]`

#### `GET /stats/{ticker}` 🔓
Tek ticker'ın istatistikleri.

- **Yanıt 200:** `{info_count, report_count, news_count, history_count, simulation_count, favorite_count, ticker: "XXX"}`

### 8.3 Raporlar (`src/api/reports.py`)

#### `POST /reports/generate` 🔐 ⚙(report_generate) ⏳(report, 900s)
LLM tabanlı yatırım raporu üretir. Kredi: tahmini maliyet çekilir (`estimated_cost`), gerçek token kullanımına göre iade/ek tahsilat yapılır.

- **Query:** `ticker: str` (zorunlu, geçerli BIST kodu), `type: "quick_report"|"deep_report"` (zorunlu), `purpose: str|null (max 500)` (opsiyonel kullanıcı sorusu)
- **Maliyet:** quick: max 5000 token, deep: max 50000 token; `token_cost_per_1k` varsayılan 0.05 → quick tahmini ~0.25, deep ~2.5 kredi. Yetersiz kredi: **402** `"insufficient credit"`.
- **Yanıt 200:**
```json
{
  "success": true, "report_id": 12, "credits_spend": 0.25, "remaining_credits": 17.75,
  "about": "THYAO", "type": "quick_report", "title": "THYAO Analizi",
  "report": "<markdown içerik>", "sentiments": [],
  "token_usage": {"prompt": 0, "completion": 0, "total": 5000},
  "created_at": "2026-08-14T12:00:00+00:00"
}
```
- **Hatalar:** 400 `"Invalid type"`; 402 yetersiz kredi; 500 `"Report generation failed"` (kredi iade edilir)
- **Not:** İstek eşzamanlı tek raporla sınırlıdır (job slot); süre 30-60s+ olabilir.

#### `GET /reports/info` 🔓 (auth dep'i yok; middleware public listesinde de değil — token yoksa 401 alırsınız)
Rapor tipleri, maliyetler ve endpoint dokümantasyonu.

- **Yanıt 200:** `{quick_report: {type, name_en, name_tr, description, description_tr, est_cost}, deep_report: {...}, token_cost_per_1k, endpoints: {generate, history, search, detail}}`

#### `GET /reports/history` 🔐
Kullanıcının rapor geçmişi.

- **Query:** `sort: "created_at"|"ticker"` (vars. created_at), `order: "asc"|"desc"` (vars. desc)
- **Yanıt 200:** `[ReportHistoryItem]` — `{id, ticker, type, title|null, token_usage|null, purpose|null, created_at}`
- **Hatalar:** 400 geçersiz sort/order (`"Invalid sort. Allowed: ['created_at', 'ticker']"`)

#### `GET /reports/search` 🔐
Başlık/içerikte ILIKE araması (yalnızca kullanıcının kendi raporları).

- **Query:** `q: str (min 1)` (zorunlu), `sort`, `order`, `limit: int 1-100` (vars. 20), `offset: int ≥0`
- **Yanıt 200:** `[ReportHistoryItem]`

#### `GET /reports/{report_id}` 🔐
Tek rapor (sahibi kontrolü).

- **Yanıt 200:** `{success, report_id, about (ticker), type, title, token_usage, purpose, report (markdown), sentiments, created_at}`
- **Hatalar:** 404 `"Report not found or you do not have permission to view it."`

#### `POST /reports/download` 🔐
Raporu md/docx/pdf olarak indirir (pandoc; docx/pdf üretimi thread'de).

- **Query:** `report_id: int` (zorunlu), `ftype: "md"|"docx"|"pdf"` (zorunlu)
- **Yanıt:** dosya (uygun Content-Type + `Content-Disposition: attachment; filename="report_{id}.{ftype}"`)
- **Hatalar:** 404 `"Report not found."`; 400 `"Invalid file type."`

### 8.4 Simülasyonlar (`src/api/simulations.py`)

#### `GET /simulations/per-day-cost` 🔐
Günlük simülasyon maliyeti.

- **Yanıt 200:** `{"per_day_cost": 0.005, "round": 3}`

#### `GET /simulations/estimate-cost/{ticker}` 🔐
Maliyet tahmini.

- **Query:** `days: int 1-370` (zorunlu)
- **Yanıt 200:** `{"cost": 1.85}` (days × 0.005, 3 basamak)

#### `GET /simulations/history` 🔐
Geçmiş simülasyonlar.

- **Query:** `limit: int 1-100` (vars. 20), `offset: int ≥0`
- **Yanıt 200:** `[{id, ticker, days, bounds, target, cost, created_at}]`

#### `GET /simulations/history/{sim_id}` 🔐
Simülasyon detayı.

- **Yanıt 200:** `{id, ticker, days, bounds, target, result (JSONB), cost, created_at}`
- **Hatalar:** 404 `"Simulation not found"`

#### `GET /simulations/{ticker}` 🔐 ⚙(simulation) ⏳(simulation, 600s)
Monte Carlo simülasyonu çalıştırır (CPU-yoğun, thread'de).

- **Path:** `ticker` (geçerli BIST kodu); **Query:** `days: int 1-370` (zorunlu), `bounds: str` (vars. "0.05" — güven aralığı yüzdesi), `target: str|null` (hedef fiyat; verilmezse otomatik = güncel fiyat + %10, direction "above")
- **Maliyet:** days × 0.005 kredi (402 yetersizse `"insufficient credit"`)
- **Yanıt 200:**
```json
{
  "prob_above": 0.62, "prob_below": 0.38,
  "confidence": {"min": 95.2, "max": 118.4, "percent": 0.9, "days": 30, "bounds": "0.05"},
  "direction": "above", "simulation_id": 7, "ticker": "THYAO", "days": 30,
  "target": "auto", "bounds": "0.05", "credits_spend": 0.15, "remaining_credits": 17.6
}
```
- **Hatalar:** 400 `"Invalid target price"` (target ≤ 0), `"Invalid simulation parameters"`; 500 `"Simulation failed, credits refunded."`
- **Not:** `direction`: target güncel fiyat ≥ ise "above", değilse "below"; target yoksa "above".

### 8.5 Ekonomi / Makro / IPO (`src/api/economy.py`, `src/api/ipo.py`)

#### `GET /economy/gold-prices` 🔓
Altın fiyatları (GenelPara, 20 dk cache). Anahtarlar: `gram-altin`, `ceyrek-altin`, `ons`, `gram-has-altin`, `yarim-altin`, `tam-altin`, `cumhuriyet-altini`, `ata-altin`, `14-ayar-altin`, `18-ayar-altin`, `22-ayar-bilezik`, `ikibucuk-altin`, `besli-altin`, `gremse-altin`, `resat-altin`, `hamit-altin`.

- **Yanıt 200:** `{"gram-altin": {"Buying": "3.450,50", "Selling": "3.470,00", "Change": "%0,42", "Type": "Gold"}, ...}`
- **Not:** `Buying`/`Selling` değerleri Türkçe virgül-ondalık **string** olarak döner; `Change` `%x.xx` formatında string.

#### `GET /economy/silver-price` 🔓
Gümüş fiyatı.

- **Yanıt 200:** `{"gumus": {"Buying", "Selling", "Change", "Type": "Gold"}}`

#### `GET /economy/gram-platinum-price` 🔓
Gram platin.

- **Yanıt 200:** `{"gram-platin": {"Buying", "Selling", "Change", "Type": "Commodity"}}`

#### `GET /economy/gram-palladium-price` 🔓
Gram paladyum.

- **Yanıt 200:** `{"gram-paladyum": {"Buying", "Selling", "Change", "Type": "Commodity"}}`

#### `GET /economy/currency` 🔓
Döviz kurları (TRY hariç).

- **Query:** `symbols: str|null` (virgülle ayrılmış filtre, ör. `USD,EUR`; verilmezse tümü)
- **Yanıt 200:** `{"USD": {"Buying": "40,25", "Selling": "40,30", "Change": "%0,10", "Type": "Currency"}, "EUR": {...}, ...}`

#### `GET /macroeconomy` 🔓
FRED makroekonomik göstergeleri (14 seri, 24 saat cache).

- **Yanıt 200:** makro veri dict'i (gdp, fed_funds, vix, sp500, btc vb.)
- **Hatalar:** 500 `"Internal server error"` (veri yoksa)

#### `GET /ipos/upcoming` 🔓
Yaklaşan halka arzlar (halkarz.com WP API, 1 saat cache).

- **Query:** `after: str|null` (ISO tarih filtresi, vars. son 30 gün)
- **Yanıt 200:** IPO listesi (JSON)

#### `GET /ipos/draft` 🔓
Taslak (başvuru) halka arzlar.

- **Query:** `after: str|null`
- **Yanıt 200:** IPO listesi

#### `GET /ipos/active` 🔓
Aktif halka arzlar.

- **Query:** `after: str|null`
- **Yanıt 200:** IPO listesi

#### `GET /ipos/{slug}` 🔓
Tek IPO detayı.

- **Hatalar:** 404 `"IPO not found"`

### 8.6 Sanal Portföyler (`src/api/virtual_portfolio.py` + `src/services/portfolio.py`)

> Tüm portföy endpoint'leri 🔐 (JWT). `portfolio_id` formatı: `port-<uuid>`. Veriler JSONB; eşzamanlılık `pg_advisory_xact_lock(hashtext(portfolio_id))` ile korunur. Komisyon oranı `PORTFOLIO_COMMISSION_RATE` (vars. 0.001 = %0.1).
> **Önemli:** İşlem ekleme (`POST transactions`) yalnızca **piyasa açıkken** çalışır (`400 "Market is closed"`); fiyat otomatik olarak güncel piyasa fiyatıdır. Fiyat/quantity elle girilen güncelleme (`PUT transactions/{tx_id}`) bu kontrolden muaftır.

#### `POST /portfolios`
Yeni portföy.

- **Body:** `{"name": str (1-255), "initial_balance": float (>0)}`
- **Yanıt 200:** `{metadata: {id, user_id, name, initial_balance, balance, created_at, updated_at}, transactions: []}`
- **Hatalar:** 500 `"Failed to create portfolio"`

#### `GET /portfolios`
Portföy listesi.

- **Yanıt 200:** `[{metadata, transactions}]`

#### `GET /portfolios/{portfolio_id}`
Tek portföy.

- **Hatalar:** 404 `"Portfolio not found"`

#### `PUT /portfolios/{portfolio_id}`
Yeniden adlandırma.

- **Body:** `{"name": str (1-255)}`
- **Yanıt 200:** `{"message": "Portfolio renamed"}`

#### `DELETE /portfolios/{portfolio_id}`
Silme.

- **Yanıt 200:** `{"message": "Portfolio deleted"}`

#### `POST /portfolios/{portfolio_id}/duplicate`
Kopyalama (işlemler dahil, yeni id ile).

- **Body:** `{"name": str}`
- **Yanıt 200:** yeni `{metadata, transactions}`

#### `GET /portfolios/{portfolio_id}/transactions`
İşlem listesi (filtreli).

- **Query:** `ticker: str|null`, `tx_type: str|null` (BUY/SELL, alias: `type`), `start: datetime|null`, `end: datetime|null`
- **Yanıt 200:** `[{id, ticker, type, quantity, price, commission, total, date}]` (tarihe göre artan)

#### `POST /portfolios/{portfolio_id}/transactions`
İşlem ekleme (yalnızca piyasa açıkken).

- **Body:** `{"ticker": str, "type": "BUY"|"SELL", "quantity": float (>0)}`
- **Yanıt 200:** `{"message": "Transaction added"}`
- **Hatalar:** 400 `"Market is closed"`, `"Transaction failed"` (yetersiz bakiye / yetersiz lot / geçersiz ticker / fiyat alınamadı)

#### `PUT /portfolios/{portfolio_id}/transactions/{tx_id}`
İşlem güncelleme (fiyat/quantity; portföy yeniden hesaplanır).

- **Body:** `{"price": float|null, "quantity": float|null}` (en az biri)
- **Yanıt 200:** `{"message": "Transaction updated"}`
- **Hatalar:** 404 `"Transaction not found"`

#### `DELETE /portfolios/{portfolio_id}/transactions/undo`
Son işlemi geri alır.

- **Yanıt 200:** `{"message": "Last transaction undone"}`
- **Hatalar:** 400 `"Nothing to undo"`

#### `GET /portfolios/{portfolio_id}/valuation`
Değerleme.

- **Yanıt 200:** `{total_value, cash_balance, holdings_value, total_pnl, pnl_percentage, assets: [{ticker, amount, current_price, total_value, total_cost, weighted_avg_cost, unrealized_pnl, unrealized_pnl_pct}]}`

#### `GET /portfolios/{portfolio_id}/diversification`
Çeşitlendirme.

- **Yanıt 200:** `{total_value, cash_balance, cash_allocation_pct, assets: [{ticker, amount, value, type ("stock"|"forex"|"metal"), allocation_pct}], allocation_by_type: {"stock": x, ...}}`

#### `GET /portfolios/{portfolio_id}/performers`
En iyi/en kötü performanslı hisseler.

- **Query:** `top_n: int 1-20` (vars. 5)
- **Yanıt 200:** `{best: [{ticker, amount, pnl, pnl_percentage}], worst: [...]}`

#### `GET /portfolios/{portfolio_id}/history`
Zaman serisi değer geçmişi.

- **Query:** `period: "1w"|"1mo"|"3mo"|"6mo"|"1y"|"max"` (vars. 1mo)
- **Yanıt 200:** `[{ts, total_value, cash_balance, holdings_value}]` (portföy oluşturma + işlem tarihleri + şimdi)

#### `GET /portfolios/{portfolio_id}/returns`
Getiri metrikleri.

- **Query:** `period` (yukarıdaki gibi)
- **Yanıt 200:** `{period, start_value, end_value, absolute_return, total_return_percentage, cagr_percentage}`
- **Hatalar:** 404 `"Portfolio not found"`

#### `GET /portfolios/{portfolio_id}/risk`
Risk metrikleri.

- **Query:** `period: "1w"|"1mo"|"3mo"|"6mo"|"1y"|"max"` (vars. 1y)
- **Yanıt 200:** `{volatility, max_drawdown, sharpe_ratio}` (yetersiz veride alanlar null)

#### `GET /portfolios/{portfolio_id}/benchmark`
Benchmark karşılaştırması.

- **Query:** `ticker: str` (vars. `XU100`)
- **Yanıt 200:** `{portfolio_return_pct, benchmark_ticker, benchmark_return_pct, difference_pct, outperformed: bool}` — yetersiz veride `{}`

#### `GET /portfolios/{portfolio_id}/performance`
Detaylı performans analizi (işlem zamanlaması verimliliği).

- **Yanıt 200:** `{overall: {efficiency_score, actual_pnl, optimal_pnl} | null, assets: [{ticker, efficiency_score, actual_trades: [{date, type, price, quantity}], optimal_points: {best_buy: {date, price}, best_sell: {date, price}} | null, price_history: [{ts, close}] | null, actual_pnl, optimal_pnl}]}`

#### `GET /portfolios/{portfolio_id}/stats`
İşlem istatistikleri.

- **Yanıt 200:** `{total_transactions, total_buys, total_sells, total_buy_volume, total_sell_volume, avg_transaction_size, unique_tickers}`

#### `GET /portfolios/{portfolio_id}/snapshot`
Tek istekte özet: portföy bilgisi + değerleme + çeşitlendirme + performans + istatistik + son 5 işlem.

- **Yanıt 200:** `{portfolio: {id, name, created_at, updated_at}, valuation, diversification, performers, transaction_stats, recent_transactions}`

#### `GET /portfolios/{portfolio_id}/export/csv`
İşlemleri CSV olarak indirir (`text/csv`).

- **Yanıt:** `date,ticker,type,quantity,price,total` başlıklı CSV (tarihe göre artan)

### 8.7 Favoriler (`src/api/favorites.py`)

#### `POST /favorites/{ticker}` 🔐
Favorilere ekler (idempotent — zaten varsa sessizce geçer).

- **Yanıt 200:** `{"message": "Added favorite XXX or already been added"}`
- **Hatalar:** 404 geçersiz ticker; 400 `"Could not add to favorites"`

#### `DELETE /favorites/{ticker}` 🔐
Favorilerden çıkarır.

- **Yanıt 200:** `{"message": "Removed XXX from favorites"}`

#### `GET /favorites` 🔐
Favori listesi.

- **Yanıt 200:** `{"favorites": ["THYAO", "ASELS", ...]}`

### 8.8 Botlar (`src/api/bots.py`)

#### `POST /bots` 🔐
Bot hesabı oluşturur (max 5/kullanıcı). Bot'lar owner kredisinden harcar; e-posta doğrulaması gerekmez; **şifre yalnızca bu yanıtta bir kez döner**.

- **Body:** `{"username": str (3-255), "password": str|null (min 10)}` (password boşsa rastgele 16 karakter üretilir)
- **Yanıt 200:** `{"id": 55, "username": "...", "email": "...@bot.florencex.com.tr", "password": "<tek seferlik>"}`
- **Hatalar:** 403 `"error_bots_not_allowed"` (bot hesap bot açamaz); 400 `"error_bot_limit_reached"`; 409 `"error_username_taken"`

#### `GET /bots` 🔐
Bot listesi.

- **Yanıt 200:** `{"bots": [{id, username, created_at, last_login}]}`

#### `DELETE /bots/{bot_id}` 🔐
Bot siler (yalnızca kendi botu).

- **Yanıt 200:** `{"message": "Bot {id} deleted"}`
- **Hatalar:** 404 `"Bot not found"`

### 8.9 Veri Dışa Aktarım (`src/api/export.py`, `src/api/exports.py`)

#### `GET /user/export` 🔐
Tüm kullanıcı verisinin JSON dump'ı (profil + favoriler + raporlar + token kullanımı + simülasyonlar).

- **Yanıt 200:** `{profile: {username, email, credits}, favorites: [{ticker_code, created_at}], reports: [{id, ticker, type, title, token_usage, content, created_at}], token_usage: [{model, prompt_tokens, completion_tokens, total_tokens, endpoint, created_at}], simulations: [{id, ticker, days, bounds, target, result, cost, created_at}]}`

#### `POST /data/export` 🔐 (202 Accepted)
Takeout tarzı yıllık veri dışa aktarımı kuyruğa alır; arka plan işçisi hazırlar, e-posta ile indirme linki gönderilir. Idempotent: aynı kullanıcı+year+format için aktif kayıt varsa mevcut id döner.

- **Body:** `{"year": int (1990..şimdiki yıl+1), "format": "csv"|"json"}` (format vars. csv)
- **Rate limit:** 3/saat (admin 30/saat)
- **Yanıt 202:** `{"export_id": 5, "status": "queued"}`
- **Hatalar:** 400 `"Invalid year"`

#### `GET /data/export` 🔐
Export listesi.

- **Yanıt 200:** `[{id, year, format, status, created_at, updated_at, row_count, size_bytes, downloaded_count, expires_at, error, downloadable, download_url}]`

#### `GET /data/export/download/{token}` 🔓
Public indirme (auth yok, token yeterli). Dosya adı `florence-daily-{year}.{format}.gz`.

- **Hatalar:** 404 `"Export not found"` / `"Export file missing"`; 410 `"Export link expired or not ready"` (status ready/sent değilse veya süresi dolmuşsa)

#### `GET /data/export/{export_id}` 🔐
Tek export kaydı (sahibi kontrolü).

- **Yanıt 200:** serialize edilmiş kayıt (yukarıdaki gibi)
- **Hatalar:** 404 `"Export not found"`

#### `GET /data/daily/{year}` 🔓
**Kullanım dışı** — her zaman 410 döner.

- **Yanıt 410:** `{"detail": "Kullanım dışı — POST /api/v1/data/export ile istek oluşturun"}`

### 8.10 Duyurular (`src/api/announcements.py`)

#### `GET /announcements` 🔐
Duyuru listesi (yalnızca son 7 gün içinde yayınlananlar; `is_unread` işareti ile).

- **Yanıt 200:** `{"announcements": [{id, title, content, sent_by, created_at, updated_at, is_unread}]}`

#### `GET /announcements/{announcement_id}` 🔐
Tek duyuru.

- **Hatalar:** 404 `"Announcement not found"`

#### `POST /announcements` 🔐 (admin)
Duyuru oluşturur.

- **Body:** `{"title": str, "content": str}`
- **Hatalar:** 403 `"Admin access required"`

#### `PUT /announcements/{announcement_id}` 🔐 (admin)
Duyuru günceller.

- **Body:** `{"title": str, "content": str}`
- **Hatalar:** 403; 404 `"Announcement not found"`

#### `DELETE /announcements/{announcement_id}` 🔐 (admin)
Duyuru siler.

- **Hatalar:** 403; 404

#### `POST /announcements/read` 🔐
Tüm duyuruları okundu işaretler (`last_announcement_viewed_at = NOW()`).

- **Yanıt 200:** `{"message": "Marked as read"}`

### 8.11 Danışman — Hisse Önerisi (`src/api/fit.py`, `src/api/portfolio.py`)

#### `POST /stocks/fit` 🔐 ⚙(advisor)
Stock-vector benzerlik önerisi (risk/horizon/profitability skorları).

- **Body:** `{"horizon": "short"|"medium"|"long" (vars. long), "profitability": "low"|"medium"|"high" (vars. high), "risk_tolerance": "low"|"medium"|"high" (vars. medium), "limit": int 1-100 (vars. 5)}`
- **Yanıt 200:** `{"query": {"risk": 0.5, "horizon": 0.7, "profitability": 0.7}, "results": [{ticker, score, vector, ...}]}`
- **Hatalar:** 400 `"Invalid filter value"`

#### `POST /portfolio/profile` 🔐 ⚙(advisor)
"Bana benzer hisse öner": kullanıcının portföy vektörlerinin ortalamasına Euclidean en yakın hisseler.

- **Body:** `{"tickers": [str (1-50, uppercase)], "limit": int 1-50 (vars. 5)}`
- **Yanıt 200:** `{avg_vector: [float...], estimated_profile: {risk, horizon, profitability}, portfolio: [{ticker, vector}], similar_stocks: [{ticker, vector, score, distance}]}`
- **Hatalar:** 400 `"No vector data available for given tickers"`

### 8.12 Diğer Public Endpoint'ler

#### `GET /market/status` 🔓
BIST açık/kapalı + resmi tatil (Redis 60s cache). Piyasa saatleri: 10:00–18:10 (Europe/Istanbul), hafta içi + resmi tatil değilse açık.

- **Yanıt 200:** `{"open": true, "next_open_at": "2026-08-17T10:00:00+03:00"|null, "timezone": "Europe/Istanbul", "is_holiday": false, "holiday_name": null, "as_of": "..."}`

#### `GET /maintenance` 🔓
Devre dışı feature listesi.

- **Yanıt 200:** `{"disabled_features": []}` (küme üyeleri: `report_generate`, `simulation`, `news`, `advisor`)

#### `GET /meta/avatars` 🔓
Avatar id listesi.

- **Yanıt 200:** `[{"id": "avatar-1", "url": "/avatars/avatar-1.svg"}, ... 12 adet]`

#### `GET /legal?policy=&lang=` 🔓
Yasal metin. `policy: "terms"|"privacy_policy"|"cookie_policy"|"disclaimer"`, `lang: "tr"|"en"` (vars. tr).

- **Yanıt 200:** `{"policy", "lang", "last_updated": "2026-07-22", "content": "<metin>"}`
- **Hatalar:** 404 bilinmeyen policy; 400 geçersiz lang

#### `GET /legal/all?lang=` 🔓
Tüm yasal metinler.

- **Yanıt 200:** `{"last_updated", "lang", "policies": {"terms": ..., "privacy_policy": ..., ...}}`

#### `GET /about?lang=` 🔓
Hakkında metni (`"tr"|"en"`).

- **Yanıt 200:** `{"lang", "content"}`

#### `GET /contact` 🔓
İletişim bilgileri.

- **Yanıt 200:** `CONTACT` sabiti

#### `GET /version` 🔓
Uygulama sürümü.

- **Yanıt 200:** `{"version": "0.5.7"}`

#### `GET /contributors` 🔓
Katkıda bulunanlar.

- **Yanıt 200:** `{"contributors": [{nickname, picture_url, github_url}]}`

#### `GET /` ve `GET /health` 🔓
Health check.

- **Yanıt 200:** `{}` ve `{"status": "ok"}`

#### `POST /analytics/event` 🔐 (middleware üzerinden; path middleware'de analytics'e kaydedilmez)
Toplu analitik olay (fire-and-forget).

- **Body:** `[{event_type: str, ticker: str|null, details: dict|null, ...}, ...]` — max 100 öğe
- **Yanıt 200:** `{"received": n}`
- **Hatalar:** 413 `"Too many events"`

### 8.13 Admin Uygulaması (`src/admin/__init__.py` — ayrı FastAPI, UDS `/run/florence/admin.sock` üzerinden servis edilir)

> Auth: `X-Admin-Token: <ADMIN_TOKEN>` header (zorunlu, `verify_admin_token`). Bu uygulama `/api/v1` altında **değildir**.

| Method | Path | Açıklama |
|---|---|---|
| POST | `/gift-credits` | Kredi hediye et. Query: `user_type: "everyone"\|"user"`, `amount: int (>1)`, `username: str|null`, `credit_type: "free_credits"\|"gift_credits"` (vars. free_credits); Body: `filters: dict` (ops. `{}`). `everyone` → `{"success": true}`; `user` → `{"success": true, "user": {username, credits}}`. Hatalar: 400 `"username is required for user type"`, 404 `"User not found"` |
| POST | `/config-reload` | Config'i yeniden yükler. → `{"success": true}` |
| POST | `/healthcheck` | DB/Redis/LLM/SearXNG/yfinance sağlık raporu. → `{db_health, redis_health, llm_health, news_health, yfinance_health, status: "OK"\|"ERROR"}` |
| POST | `/token-usage` | Token kullanım özeti. Query: `since: str|null` (ISO), `endpoint: str|null`. Hatalar: 400 `"Invalid datetime format. Use ISO format."` |
| POST | `/maintenance/toggle` | Feature aç/kapat. Query: `feature` (report_generate/simulation/news/advisor), `action: "enable"\|"disable"`. → `{feature, disabled: bool}`. Hatalar: 400 `"Unknown feature: X"` / `"Action must be 'enable' or 'disable'"` |

---

## 9. Ortam Değişkenleri

| Değişken | Varsayılan | Açıklama |
|---|---|---|
| `ENVIRONMENT` | `development` | `production` ise docs kapalı, CORS Tauri origin'leri, cookie `secure` |
| `SECRET_KEY` | — | **Zorunlu.** JWT HS256 imza anahtarı |
| `ADMIN_TOKEN` | — | Admin app `X-Admin-Token` değeri |
| `FRED_API_KEY` | — | **Import anında zorunlu** (yoksa `ValueError`) — FRED makro verisi |
| `FREE_CREDIT_MAX` | `25` | Günlük refill tavanı (free_credits) |
| `DAILY_FREE_CREDIT_REFILL` | `5` | Günlük otomatik kredi ekleme miktarı |
| `REFRESH_TOKEN_TTL_DAYS` | `30` | Refresh token ömrü |
| `PORTFOLIO_COMMISSION_RATE` | `0.001` | Sanal portföy komisyon oranı (%0.1) |
| `LOG_DIR` | `/var/log/florence` | Log dizini (yazılabilir olmalı) |
| `PUBLIC_BASE_URL` | request base URL | E-posta doğrulama linklerinde kullanılan public URL |
| `POSTGRES_HOST/PORT/USER/PASSWORD/DB` | localhost:5433 | PostgreSQL bağlantısı |
| `REDIS_HOST/PORT/DB/PASSWORD` | localhost:5434 | Redis bağlantısı |
| `SEARXNG_HOST_PORT` | `5435` | SearXNG dış portu |
| `NEWS_SEARCH_URL` | `http://localhost:5435/search` | SearXNG arama URL'si |
| `MAIL_PROVIDER` | `resend` | `smtp`/`mailpit`/`resend`/`ses` |
| `MAIL_FROM`, `MAIL_HOST/PORT/USER/PASS`, `RESEND_API_KEY`, `MAIL_EXPORT_FROM` | — | E-posta ayarları |
| `GOOGLE_APPLICATION_CREDENTIALS` | — | BigQuery (GDELT haber arşivi) servis hesabı |
| `COLLECT_API_KEY` | — | Haber toplama API anahtarı |
| `OPENROUTER_API_KEY`, `CUSTOM_API_KEY`, `CUSTOM_URL` (`http://localhost:7777/v1`), `CUSTOM_MODEL` (`gemma`) | — | LLM yapılandırması |
| `EMBEDDING_API_KEY/BASE_URL/MODEL` | ollama / `http://127.0.0.1:11434/v1` / `mxbai-embed-large` | Embedding (stock vector) |
| `EVDS_API_KEY`, `LOGODEV_API_KEY` | — | TCMB EVDS ve logo API anahtarları |

**Config override'ları:** `get_config()` varsayılanları `<SECTION>_<KEY>` şemasıyla env'den override edilir (ör. `REPORT_TOKEN_COST_PER_1K`, `ECONOMY_CACHE_TTL`, `MACROECONOMY_CACHE_TTL`, `PRICE_HISTORY_CACHE_TTL_HOT`, `SIMULATION_PER_DAY_COST`…). Tam liste: backend `.env.example` + `src/core/config.py` `_DEFAULTS`.

---

## 10. Kredi Sistemi

- `user_credits` tablosu: `(user_id, credit_type, amount)` — `free_credits` ve `gift_credits` ayrı satırlardır; harcama önce free'den yapılır.
- Günlük refill (cron): her kullanıcıya `DAILY_FREE_CREDIT_REFILL` ekler, `FREE_CREDIT_MAX` ile tavanlanır.
- Bot hesapları owner'ın kredisinden harcar (`credits._resolve_owner`).
- Harcama: rapor üretimi (tahmini maliyet çekilir, gerçeğe göre iade/ek tahsilat) ve simülasyon (days × 0.005).
- Kredi yetmezse 402 `"insufficient credit"`.

---

## 11. Feature Kill-Switch (Bakım)

`maintenance:disabled` Redis set'i ile özellikler anında kapatılabilir (admin `/maintenance/toggle`). Kapatılan feature'ı kullanan endpoint **503** `"{feature} is temporarily disabled for maintenance"` döner.

| Feature | Etkilenen endpoint'ler |
|---|---|
| `report_generate` | `POST /reports/generate` |
| `simulation` | `GET /simulations/{ticker}` |
| `news` | `GET /news/{ticker}` |
| `advisor` | `POST /stocks/fit`, `POST /portfolio/profile` |

---

## 12. Geliştirici Notları / Tuzaklar

1. **FRED_API_KEY import hatası:** `src/clients/macroeconomy.py` import anında `FRED_API_KEY` arar; yoksa `ValueError` fırlar. Test/script ortamlarında sahte değer export edin.
2. **Schema kaynağı `init_db()`'dir:** `migrations/` dosyaları otomatik uygulanmaz; şema değişikliğinde ikisini de güncelleyin.
3. **DB bağlantı disiplini:** uzun süren işlemlerden (LLM çağrısı, simülasyon) önce `db.release_current()` çağrılır; yeni cursor blokları yeni bağlantı alır. `keep=True` yalnızca advisory lock akışlarında.
4. **`price_candles` ticker formatı:** DB'de `XXX.IS`; API girişleri `.IS` suffix'siz kabul edilir ve normalize edilir.
5. **Quote `change_pct`:** açık piyasada intraday veri yoksa `null` döner (yanıltıcı %0 gösterilmez); `is_stale` açıkken 20 dk, kapalıyken 3 gün bayatlık eşiği.
6. **Login form-encoded'dir** (OAuth2), JSON değil. Register JSON'dur.
7. **Job slot'lar eşzamanlı işi engeller:** aynı kullanıcı aynı anda iki rapor/simülasyon başlatamaz (429).
8. **`/reports/info` public değildir:** middleware'de değildir ve auth dep'i yoktur — davranışı "token yoksa 401" şeklindedir; belgelemek için buraya not düşülmüştür.

---

*Bu doküman 2026-08-14 tarihinde backend kod tabanından otomatik/manuel karışık yöntemle üretilmiştir. Kaynak: `florence/backend` (sürüm 0.5.7).*
