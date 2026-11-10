# Deadweb Relay

![Deadweb Relay identity](docs/brand/deadweb-relay/readme-lockup.png)

Deadweb Relay is a small Go website that looks like an old personal homepage:
guestbook, wall, public notes, file cabinet, archive, webring and search. Under
that surface it serves stable URLs, a writable tree of public notes, HTML, plain
text and JSON versions of each page, feeds, standard discovery files and careful
HTTP semantics.

It is also a small research site. The goal is passive discoverability: the site
is easy for crawlers and other automated clients to understand, and it privately
records what they do. It places no hidden instructions in its pages, does not
pose as another client, never fetches URLs that visitors submit, and rejects
obvious credentials and private keys.

Live site: <https://www2.onnwee.me/> ·
[HTTP guide](https://www2.onnwee.me/help.txt) ·
[How the relay works](https://www2.onnwee.me/reference/http-relay.txt)

## Features

- Guestbook (newest first, 50 per page), an append-only wall and a webring.
- Public notes at any path under `/notes/`, written with `PUT` and read back as
  raw text, HTML or JSON depending on the `Accept` header.
- A file cabinet for small text-like files (text, Markdown, CSV, JSON, XML and
  SVG).
- Site search, plus JSON, NDJSON and Atom activity feeds.
- Discovery files: `robots.txt`, `sitemap.xml`, OpenSearch, WebFinger,
  NodeInfo, host-meta and a site manifest.
- Webmaster reference sheets under `/reference/` and an old-web archive.
- A private observatory with request and write logs, behind a bearer token and
  not linked from the site.
- No database and no client-side JavaScript required.

### Public paths

| Path | Purpose |
| --- | --- |
| `/` | Old-web homepage |
| `/guestbook/` | Persistent public messages |
| `/wall/` | Chronological append-only wall |
| `/notes/` | Mutable hierarchical public notes |
| `/files/` | Small text-like public files |
| `/archive/`, `/reference/`, `/webring/` | Authored pages |
| `/search` | Search public content |
| `/feed.json`, `/feed.ndjson`, `/feed.atom` | Activity feeds |
| `/help.txt`, `/index.txt` | Plain-text protocol guide and site index |
| `/robots.txt`, `/sitemap.xml`, `/opensearch.xml` | Crawler and search discovery |
| `/.well-known/host-meta`, `/.well-known/host-meta.json` | Service hints |
| `/.well-known/webfinger`, `/.well-known/nodeinfo`, `/.well-known/site-meta.json` | Generic discovery documents |
| `/cgi-bin/guestbook.cgi` | Old-style alias for the guestbook |
| `/healthz` | Health check |

## Quick start

With Docker:

```bash
cp .env.example .env
# Set ADMIN_TOKEN and OBSERVATORY_HMAC_KEY to distinct random values.
docker compose up --build
```

Or with Go:

```bash
go run ./cmd/relay    # or: make run
```

Open <http://localhost:8080>.

Try the HTTP side:

```bash
# Supported methods
curl -i -X OPTIONS http://localhost:8080/notes/example

# Store a public note
curl -X PUT -H 'Content-Type: text/plain' --data 'hello' \
  http://localhost:8080/notes/example/inbox

# Read it back: raw text for curl, HTML for browsers, JSON on request
curl http://localhost:8080/notes/example/inbox
curl -H 'Accept: application/json' http://localhost:8080/notes/example/inbox

# Append to the wall, then poll for newer entries
curl -X POST http://localhost:8080/wall/ -d 'name=anon' --data-urlencode 'body=hello world'
curl -H 'Accept: text/plain' 'http://localhost:8080/wall/?after=1234'
```

Tests:

```bash
go test ./...          # or: make test
./scripts/demo.sh      # walk through the main writes against a running server
```

## Configuration

| Variable | Default | Meaning |
| --- | --- | --- |
| `ADDR` | `:8080` | Listen address |
| `DATA_DIR` | `./data` | Persistent state directory |
| `SITE_NAME` | `Patrick's WWW Page` | Visible site title |
| `BASE_URL` | `http://localhost:8080` | Absolute URL used by feeds and discovery files |
| `ADMIN_TOKEN` | unset | Bearer token for the private observatory |
| `OBSERVATORY_HMAC_KEY` | `local-dev-key` | Key used to pseudonymize remote addresses |
| `MAX_WRITE_BYTES` | `65536` | Maximum write or upload size |
| `WRITE_RATE_PER_HOUR` | `60` | Write limit per pseudonymous source |
| `RELAY_BIND_ADDR` | `127.0.0.1` | Host address Compose publishes port 8080 on |

Compose requires `ADMIN_TOKEN` and `OBSERVATORY_HMAC_KEY` to be set.

## Storage

State is append-only JSONL files plus a directory of uploaded public files:

```text
data/
├── guestbook.jsonl
├── wall.jsonl
├── notes.jsonl
├── files.jsonl
├── events.jsonl
└── files/
```

There is no database, so the state is easy to inspect, snapshot, diff and
archive. `notes.jsonl` keeps every note version; the current value is rebuilt at
startup, so only the newest version is public but overwrites stay in the
history.

`content.md` holds the site's authored material: a dated webmaster guestbook and
wall timeline, public notes and archive scraps. On a fresh deployment, import it
once with `make seed-content` before starting the relay. The command refuses to
run if the guestbook or wall already has entries. Everything it imports is
attributed to `webmaster`, not to visitors.

## Private observatory

Set `ADMIN_TOKEN`, then:

```bash
curl -H 'Authorization: Bearer YOUR_TOKEN' http://localhost:8080/__events.ndjson
```

`/__observatory` takes the same header. Request logs record the time, method,
path, status, duration, User-Agent, Referer, Accept header and a keyed HMAC
pseudonym of the remote address, plus public write events. Raw remote addresses
are not stored. Use a strong private `OBSERVATORY_HMAC_KEY`; a different key per
deployment keeps pseudonyms from being linked across them.

## Safety defaults

- 64 KiB maximum write size by default, and per-source write rate limits.
- No executable uploads; only text-like file types are accepted.
- Common private keys, bearer tokens, API keys and credential-shaped strings are
  rejected before storage.
- Message bodies are escaped; user HTML never runs.
- Uploaded files are served with `nosniff`, and the site sends a restrictive CSP.
- The server never follows URLs that visitors submit.

It is still a writable service on the internet. Put it behind HTTPS, run it
unprivileged with resource limits, back up the data directory and read the logs.

## Deploy

The `Dockerfile` builds a static binary on Alpine that runs as an unprivileged
user with `/app/data` as a volume. `compose.yaml` runs it with a read-only root
filesystem, dropped capabilities, a health check and log rotation, listening on
loopback by default. Put a TLS reverse proxy in front; `Caddyfile.example` is a
starting point (set `RELAY_HOSTNAME`, `RELAY_UPSTREAM` and `CADDY_EMAIL`).

```bash
BASE_URL=https://your.host docker compose up -d --build
```

## Research notes

- [docs/BAIT-MATRIX.md](docs/BAIT-MATRIX.md) explains each passive discovery
  surface and what signal it can produce.
- [docs/RESEARCH-GUARDRAILS.md](docs/RESEARCH-GUARDRAILS.md) lists the techniques
  left out on purpose because they would muddy the results or add risk.
- Mark your own test traffic, and test writes against disposable local state.

## Project layout

```text
cmd/relay      the web server, routes and pages
cmd/seed       imports content.md into a fresh data directory
internal/      storage, rate limiting and credential screening
web/           templates, static files and reference sheets
scripts/       demo walk-through and image checks
```

## Contributing

- Bugs and feature requests: [GitHub issues](https://github.com/subculture-collective/deadweb-relay/issues).
- Questions and ideas: [GitHub Discussions](https://github.com/subculture-collective/deadweb-relay/discussions).
- Code: open a pull request on GitHub against `main`. CI checks Go modules and
  `go vet`, runs the tests, builds the server, and builds and smoke-tests the
  container image.

## License

MIT. See [LICENSE](LICENSE).
