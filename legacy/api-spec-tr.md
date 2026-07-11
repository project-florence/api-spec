# Türkçe

## Genel Bakış

Florence bir yatırım analiz platformudur. Veriye dayalı hisse araştırması ve portföy önerileri sunar."

**Temel URL:** `https://api.florence.io/v1`

| Kategori                 | Açıklama                                                  |
| ------------------------ | --------------------------------------------------------- |
| Portföy Arama            | Yatırımcı profiline uygun hisseleri önerir                |
| Hisse Analizi            | Belirli bir hisse hakkında kapsamlı araştırma ve analiz   |

---

## 1. Portföy Arama

Kullanıcının yatırım tercihlerine göre sıralanmış hisse önerileri döndürür.

### `POST /portfolio/search`

#### İstek Gövdesi

| Alan             | Tür      | Zorunlu | Açıklama                                                                            |
| ---------------- | -------- | ------- | ----------------------------------------------------------------------------------- |
| `risk_tolerance` | integer  | Evet    | Risk iştahı: `1` (en düşük) ile `10` (en yüksek) arası                              |
| `expected_gain`  | integer  | Evet    | Getiri beklentisi — `1`: Enflasyona karşı değer koruma, `2`: Düşük-orta getiri, `3`: Yüksek getiri |
| `sectors`        | string[] | Hayır   | Tercih edilen sektörler (örn. `["technology", "energy", "finance"]`). Boş bırakılırsa tüm sektörler değerlendirilir. |
| `maturity`       | integer  | Evet    | Yatırım vadesi (ay cinsinden)                                                       |
| `capital`        | integer  | Evet    | Kullanılabilir sermaye tutarı                                                        |

#### Örnek İstek

```json
{
  "risk_tolerance": 4,
  "expected_gain": 2,
  "sectors": ["technology", "healthcare"],
  "maturity": 12,
  "capital": 50000
}
```

#### Yanıt `200 OK`

| Alan                               | Tür      | Açıklama                                                |
| ---------------------------------- | -------- | ------------------------------------------------------- |
| `portfolio`                        | object[] | Önerilen hisselerin sıralı listesi                      |
| `portfolio[].code`                 | string   | Hisse kodu / sembolü                                    |
| `portfolio[].description`          | string   | Şirket ve faaliyet alanı hakkında kısa özet             |
| `portfolio[].current_price`        | number   | Son bilinen hisse fiyatı                                |
| `portfolio[].investment_score`     | number   | Birleşik yatırım skoru (0–100)                          |
| `portfolio[].risk_score`           | number   | Birleşik risk skoru (0–100)                             |
| `portfolio[].tags`                 | string[] | Kategori etiketleri (örn. `["technology", "growth"]`)   |
| `portfolio[].metadata`             | object   | Hisseye özel ek veriler (aşağıdaki Metadata tablosuna bakınız) |

#### Metadata (hisse başına, isteğe bağlı alanlar)

| Alan             | Tür     | Açıklama                     |
| ---------------- | ------- | ---------------------------- |
| `market`         | string  | Piyasa / borsa adı           |
| `currency`       | string  | İşlem gördüğü para birimi    |
| `pe_ratio`       | number  | Fiyat/Kazanç oranı           |
| `market_cap`     | integer | Piyasa değeri                |
| `dividend_yield` | number  | Temettü verimi (yüzde)       |

#### Örnek Yanıt

```json
{
  "portfolio": [
    {
      "code": "THYAO",
      "description": "Türk Hava Yolları, Türkiye'nin bayrak taşıyıcı havayolu şirketi.",
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

## 2. Hisse Analizi

Tek bir hisse üzerinde araştırma yapar ve analiz raporu döndürür.

### `POST /analysis/stock`

#### İstek Gövdesi

| Alan                | Tür     | Zorunlu | Açıklama                                                              |
| ------------------- | ------- | ------- | --------------------------------------------------------------------- |
| `code`              | string  | Evet    | Hisse kodu / sembolü                                                  |
| `research_interval` | integer | Hayır   | Geriye dönük araştırma penceresi (gün cinsinden, varsayılan: `90`)    |
| `analysis_depth`    | string  | Hayır   | Araştırma derinliği — `"quick"` (hızlı) veya `"deep"` (derin). Varsayılan: `"quick"` |

#### Yanıt `200 OK`

| Alan                                      | Tür      | Açıklama                                                           |
| ----------------------------------------- | -------- | ------------------------------------------------------------------ |
| `summary_report`                          | string   | İnsan tarafından okunabilir analiz özeti                           |
| `investment_score`                        | number   | Birleşik yatırım skoru (0–100)                                     |
| `risk_score`                              | number   | Birleşik risk skoru (0–100)                                        |
| `scenarios`                               | object[] | Projeksiyon senaryoları                                            |
| `scenarios[].name`                        | string   | Kısa senaryo etiketi (örn. `"bull"`, `"base"`, `"bear"`)           |
| `scenarios[].description`                 | string   | Senaryonun açıklaması                                              |
| `research_data`                           | object   | Destekleyici veriler ve referanslar                                |
| `research_data.current_price`             | number   | Son bilinen hisse fiyatı                                           |
| `research_data.price_change_pct`          | number   | Araştırma aralığındaki fiyat değişimi (yüzde)                      |
| `research_data.recent_news`               | object[] | İlgili güncel haberler / manşetler                                 |
| `research_data.recent_news[].title`       | string   | Haber başlığı                                                      |
| `research_data.recent_news[].url`         | string   | Haberin kanonik URL'si                                             |
| `research_data.recent_news[].source`      | string   | Yayıncı / haber kaynağı                                            |
| `research_data.recent_news[].published_at` | string  | ISO 8601 tarih damgası                                             |
| `research_data.fundamentals`              | object   | Temel finansal göstergeler (aşağıdaki tabloya bakınız)             |

#### Temel Göstergeler (Fundamentals, isteğe bağlı)

| Alan             | Tür    | Açıklama                              |
| ---------------- | ------ | ------------------------------------- |
| `pe_ratio`       | number | Fiyat/Kazanç oranı                    |
| `eps`            | number | Hisse başına kazanç                   |
| `pb_ratio`       | number | Piyasa değeri / Defter değeri         |
| `debt_to_equity` | number | Borç/Öz kaynak oranı                  |
| `roe`            | number | Öz kaynak kârlılığı (yüzde)           |
| `beta`           | number | Piyasaya göre oynaklık                |

#### Örnek Yanıt

```json
{
  "summary_report": "THYAO, artan yolcu trafiği sayesinde güçlü gelir büyümesi göstermektedir. Ancak yakıt maliyetleri ve kur riski baskı oluşturmaya devam etmektedir. Hisse, benzerlerine göre makul seviyelerde değerlenmiştir.",
  "investment_score": 72.8,
  "risk_score": 48.3,
  "scenarios": [
    {
      "name": "bull",
      "description": "Turizmde süregelen büyüme ve istikrarlı yakıt fiyatları, önümüzdeki 12 ayda kazancı %15–20 artırabilir."
    },
    {
      "name": "base",
      "description": "Sektör ortalamalarıyla uyumlu ılımlı büyüme; kazancın %5–10 artması beklenmektedir."
    },
    {
      "name": "bear",
      "description": "Jeopolitik gerginlikler veya sert kur dalgalanmaları kâr marjlarını aşındırabilir ve kazancı %10–15 düşürebilir."
    }
  ],
  "research_data": {
    "current_price": 234.50,
    "price_change_pct": 12.4,
    "recent_news": [
      {
        "title": "THY ikinci çeyrekte rekor yolcu sayısı açıkladı",
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

## Hata Yanıtları

Tüm uç noktalar ortak bir hata formatı kullanır.

### `4xx / 5xx`

| Alan            | Tür    | Açıklama                                                           |
| --------------- | ------ | ------------------------------------------------------------------ |
| `error`         | object | Hata sarmalayıcı                                                   |
| `error.code`    | string | Makine tarafından okunabilir hata kodu (örn. `INVALID_PARAMETER`, `NOT_FOUND`, `RATE_LIMITED`) |
| `error.message` | string | İnsan tarafından okunabilir açıklama                               |

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "'ABCDE' hisse kodu bulunamadı."
  }
}
```

---

## Genel Notlar

- **Kimlik Doğrulama:** Uç noktalar `Authorization: Bearer <key>` başlığında bir API anahtarı gerektirir.
- **Hız Sınırlama:** Katman bazlı limitlere tabidir. Her yanıtta hız sınır başlıkları döndürülür.
- **Para Birimleri:** `currency` metadata alanında aksi belirtilmedikçe tüm parasal değerler kullanıcının temel para birimindedir.
- **Önbellekleme:** İstemciler `GET` yanıtlarını `Cache-Control` başlığında belirtilen süre boyunca önbelleğe alabilir; `POST` yanıtları varsayılan olarak önbelleğe alınmaz.
