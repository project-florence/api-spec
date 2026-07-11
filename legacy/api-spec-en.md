# Florence API Specification — v1

**Shared API for Mobile, Web & Desktop**

---

## English

### Overview

Florence is an investment analysis platform. It does **not** execute trades — it provides data-driven stock research and portfolio recommendations.

**Base URL:** `https://api.florence.io/v1`

| Category         | Description                                      |
| ---------------- | ------------------------------------------------ |
| Portfolio Search | Suggest stocks matching user investment profile  |
| Stock Analysis   | Research and analyze a specific stock            |

---

## 1. Portfolio Search

Suggest a ranked list of stocks based on the user's investment preferences.

### `POST /portfolio/search`

#### Request Body

| Field            | Type     | Required | Description                                                              |
| ---------------- | -------- | -------- | ------------------------------------------------------------------------ |
| `risk_tolerance` | integer  | Yes      | Risk appetite from `1` (lowest) to `10` (highest)                        |
| `expected_gain`  | integer  | Yes      | Return expectation — `1`: Capital Preservation, `2`: Moderate Growth, `3`: High Growth |
| `sectors`        | string[] | No       | Preferred sectors (e.g. `["technology", "energy", "finance"]`). If empty, all sectors are considered. |
| `maturity`       | integer  | Yes      | Investment horizon in months                                             |
| `capital`        | integer  | Yes      | Available capital amount (user's base currency)                          |

#### Example Request

```json
{
  "risk_tolerance": 4,
  "expected_gain": 2,
  "sectors": ["technology", "healthcare"],
  "maturity": 12,
  "capital": 50000
}
```

#### Response `200 OK`

| Field                        | Type     | Description                                      |
| ---------------------------- | -------- | ------------------------------------------------ |
| `portfolio`                  | object[] | Ranked list of recommended stocks                |
| `portfolio[].code`           | string   | Stock ticker / symbol                            |
| `portfolio[].description`    | string   | Brief summary of the company and its business    |
| `portfolio[].current_price`  | number   | Latest known stock price                         |
| `portfolio[].investment_score` | number | Aggregate investment score (0–100)               |
| `portfolio[].risk_score`     | number   | Aggregate risk score (0–100)                     |
| `portfolio[].tags`           | string[] | Category tags (e.g. `["technology", "growth"]`)  |
| `portfolio[].metadata`       | object   | Additional dynamic data (see Metadata below)     |

#### Metadata (per stock, optional fields)

| Field            | Type     | Description                           |
| ---------------- | -------- | ------------------------------------- |
| `market`         | string   | Market / exchange name                |
| `currency`       | string   | Trading currency (ISO 4217)           |
| `pe_ratio`       | number   | Price-to-earnings ratio               |
| `market_cap`     | integer  | Market capitalization                 |
| `dividend_yield` | number   | Dividend yield (percentage)           |

#### Example Response

```json
{
  "portfolio": [
    {
      "code": "THYAO",
      "description": "Türk Hava Yolları, Turkey's national flag carrier airline.",
      "current_price": 234.50,
      "investment_score": 78.4,
      "risk_score": 42.1,
      "tags": ["transportation", "tourism", "large-cap"],
      "metadata": {
        "market": "BIST",
        "currency": "TRY",
        "pe_ratio": 8.3,
        "market_cap": 320000000000,
        "dividend_yield": 1.2
      }
    }
  ]
}
```

---

## 2. Stock Analysis

Research a single stock and return an analysis report.

### `POST /analysis/stock`

#### Request Body

| Field               | Type    | Required | Description                                                       |
| ------------------- | ------- | -------- | ----------------------------------------------------------------- |
| `code`              | string  | Yes      | Stock ticker / symbol                                             |
| `research_interval` | integer | No       | Lookback window in days (default: `90`)                           |
| `analysis_depth`    | string  | No       | Depth of research — `"quick"` or `"deep"` (default: `"quick"`)    |

#### Example Request

```json
{
  "code": "THYAO",
  "research_interval": 180,
  "analysis_depth": "deep"
}
```

#### Response `200 OK`

| Field                       | Type     | Description                                                       |
| --------------------------- | -------- | ----------------------------------------------------------------- |
| `summary_report`            | string   | Human-readable analysis summary                                   |
| `investment_score`          | number   | Aggregate investment score (0–100)                                |
| `risk_score`                | number   | Aggregate risk score (0–100)                                      |
| `scenarios`                 | object[] | Projection scenarios                                              |
| `scenarios[].name`          | string   | Short scenario label (e.g. `"bull"`, `"base"`, `"bear"`)          |
| `scenarios[].description`   | string   | Narrative description of the scenario                             |
| `research_data`             | object   | Supporting data and references                                    |
| `research_data.current_price` | number | Latest known stock price                                          |
| `research_data.price_change_pct` | number | Price change over the research interval (percentage)          |
| `research_data.recent_news` | object[] | Related recent articles / headlines                               |
| `research_data.recent_news[].title` | string | Article headline                                           |
| `research_data.recent_news[].url`   | string | Canonical URL to the article                              |
| `research_data.recent_news[].source` | string | Publisher / outlet name                                   |
| `research_data.recent_news[].published_at` | string | ISO 8601 timestamp                               |
| `research_data.fundamentals` | object  | Key financial indicators (see below)                              |

#### Fundamentals (optional)

| Field            | Type    | Description                         |
| ---------------- | ------- | ----------------------------------- |
| `pe_ratio`       | number  | Price-to-earnings ratio             |
| `eps`            | number  | Earnings per share                  |
| `pb_ratio`       | number  | Price-to-book ratio                 |
| `debt_to_equity` | number  | Debt-to-equity ratio                |
| `roe`            | number  | Return on equity (percentage)       |
| `beta`           | number  | Volatility relative to the market   |

#### Example Response

```json
{
  "summary_report": "THYAO shows strong revenue growth driven by rising passenger traffic. However, fuel costs and currency risk remain headwinds. The stock appears fairly valued relative to peers.",
  "investment_score": 72.8,
  "risk_score": 48.3,
  "scenarios": [
    {
      "name": "bull",
      "description": "Continued tourism growth and stable fuel prices could push earnings 15–20% higher over the next 12 months."
    },
    {
      "name": "base",
      "description": "Moderate growth in line with sector averages; earnings expected to rise 5–10%."
    },
    {
      "name": "bear",
      "description": "Geopolitical tensions or a sharp currency depreciation could erode margins and reduce earnings by 10–15%."
    }
  ],
  "research_data": {
    "current_price": 234.50,
    "price_change_pct": 12.4,
    "recent_news": [
      {
        "title": "THY announces record passenger numbers for Q2",
        "url": "https://example.com/thy-q2-passengers",
        "source": "Ekonomi Daily",
        "published_at": "2026-07-01T08:30:00Z"
      }
    ],
    "fundamentals": {
      "pe_ratio": 8.3,
      "eps": 28.25,
      "pb_ratio": 1.9,
      "debt_to_equity": 2.1,
      "roe": 22.4,
      "beta": 1.15
    }
  }
}
```

---

## Error Responses

All endpoints share a common error format.

### `4xx / 5xx`

| Field     | Type   | Description                        |
| --------- | ------ | ---------------------------------- |
| `error`   | object | Error wrapper                      |
| `error.code`    | string | Machine-readable error code (e.g. `INVALID_PARAMETER`, `NOT_FOUND`, `RATE_LIMITED`) |
| `error.message` | string | Human-readable description         |

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Stock code 'ABCDE' not found."
  }
}
```

---

## General Notes

- **Authentication:** Endpoints require an API key passed via `Authorization: Bearer <key>` header.
- **Rate Limiting:** Subject to tier-based limits. Rate limit headers are returned on every response.
- **Currencies:** All monetary values are in the user's base currency unless `currency` metadata says otherwise.
- **Caching:** Clients may cache GET responses for the duration indicated by the `Cache-Control` header; POST responses are not cached by default.
