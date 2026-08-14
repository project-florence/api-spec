# Florence API Specification

> Türkçe — [For the English version, click here](./README-en.md)

Florence backend'in canlı FastAPI uygulamasından üretilen güncel API dokümantasyonu.

## Dosyalar

| Dosya | Dil | İçerik |
|---|---|---|
| [`openapi.json`](./openapi.json) | OpenAPI 3.x (JSON) | Canlı uygulamadan üretilen makine spec'i (89 path, 23 schema). |
| [`docs/api-reference-tr.md`](./docs/api-reference-tr.md) | Türkçe | Tam API referansı: tüm endpoint'ler, parametreler, şemalar, auth akışları, hata formatları. |
| [`docs/api-reference-en.md`](./docs/api-reference-en.md) | English | İngilizce API referansı (Türkçe ile birebir aynı kapsam). |
| [`docs/ai-context.md`](./docs/ai-context.md) | English | AI ajanlarının backend'i anlayıp doğru kod ekleyebilmesi için tek dosyalık bağlam. |

`docs/` altındaki dosyalar ve `openapi.json`, backend kodundan (`src/api/`, `src/services/`, `src/core/`) üretilir ve kodla senkronize tutulur.

## Lisans

Bu dokümantasyon Florence projesinin bir parçasıdır; backend repo'sundaki lisans koşullarına tabidir.
