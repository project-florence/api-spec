# Florence API Specification

> English — [Türkçe versiyon için tıklayın (Click for Turkish)](./README.md)

Current API documentation generated from the live FastAPI application of the Florence backend.

## Files

| File | Language | Content |
|---|---|---|
| [`openapi.json`](./openapi.json) | OpenAPI 3.x (JSON) | Machine spec generated from the live app (89 paths, 23 schemas). |
| [`docs/api-reference-tr.md`](./docs/api-reference-tr.md) | Türkçe | Full API reference: all endpoints, parameters, schemas, auth flows, error formats. |
| [`docs/api-reference-en.md`](./docs/api-reference-en.md) | English | English API reference (identical scope to the Turkish version). |
| [`docs/ai-context.md`](./docs/ai-context.md) | English | Single-file context so AI agents can understand the backend and add correct code. |

The files under `docs/` and `openapi.json` are generated from the backend codebase (`src/api/`, `src/services/`, `src/core/`) and kept in sync with it.

## License

This documentation is part of the Florence project and is subject to the license terms of the backend repository.
