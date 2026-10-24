# URL Shortener — Go + React + Redis

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

A URL shortening service: a Go API (Fiber) that mints 7-character short
codes backed by Redis, plus a small React frontend. Built to explore
goroutine-based ID generation and a two-way Redis mapping.

## Status & honest limitations

Checked against the code in this repo — including two claims earlier
versions of this README got wrong:

- **No expiration.** An earlier README said mappings are stored "with
  automatic expiration". There is no TTL or `EXPIRE` anywhere in
  `main.go` — entries live in two Redis hashes until Redis is flushed or
  evicts under memory pressure.
- **The API does not HTTP-redirect.** `GET /?shortened=<code>` returns
  JSON (`{"originalURL": "…"}`). The *frontend* performs the redirect:
  it reads the path segment, asks the API, then sets
  `window.location.href`.
- **Short codes can theoretically collide.** A code is the first 7
  characters of a UUID; the handler never checks whether that code is
  already taken before writing it, so a collision would silently
  overwrite the older mapping. Fine at demo scale, wrong for production.
- **Dev-grade edges:** CORS allows all origins, there is no rate
  limiting, auth, custom codes, or analytics (all still future work —
  see below), and the API's Redis address is hardcoded to the Compose
  hostname `redis-server:6379`, so running the Go binary outside Compose
  needs that hostname to resolve.

## How it works

**Shortening** — `POST /` with `{"url": "https://…"}`:

1. Look the long URL up in the Redis hash `long_to_short`.
2. If it exists, return it with `code: 1` (previously shortened).
3. Otherwise take the next ID from a buffered channel (capacity 100)
   that a background goroutine continuously fills with fresh UUIDs —
   reads are mutex-guarded — and use its first 7 characters.
4. Store both directions (`long_to_short` and `short_to_long`) and
   return the code with `code: 2` (newly shortened).

**Resolving** — the frontend takes the path segment (e.g. `/a1b2c3d`),
calls `GET /?shortened=a1b2c3d`, and navigates the browser to the
returned `originalURL`. Unknown codes get a 404 from the API and the
frontend sends the visitor back to the home page.

## API

| Endpoint | Body / params | Response |
|---|---|---|
| `POST /` | `{"url": "https://example.com/…"}` | `{"url", "value": "<code>", "code": 1 \| 2}` |
| `GET /?shortened=<code>` | — | `{"originalURL": "https://…"}` or 404 |

## Run it

Everything is wired in `docker-compose.yml`:

```bash
docker compose up --build
```

| Service | Host port | Notes |
|---|---|---|
| `frontend` | 3000 | React app (`frontend_test/url-shortener-website/`) — open http://localhost:3000 |
| `url-shortener` | 5000 | Go/Fiber API (container port 3000) |
| `redis-server` | 6380 | Redis (container port 6379) |

## Stack

Go 1.21 · Fiber v2 · go-redis v8 · google/uuid · Redis · React (CRA) ·
Docker Compose

## Future work (unchanged from the original list — still not built)

- Rate limiting to prevent abuse
- Loop prevention / URL sanitization
- Custom short codes
- Click analytics
- Configurable expiration (which would also make the old README's
  expiration claim true)

## License

MIT — see [LICENSE](LICENSE).

---

*One of [Aman Thakkar's](https://github.com/amanthakkar) learning builds,
documented as it actually is.*
