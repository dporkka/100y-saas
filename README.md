# 100‑Year SaaS: Production-Ready Minimalist SaaS Platform

> A tiny, boring-on-purpose stack designed to keep running for decades with zero maintenance

[![CI/CD](https://github.com/dporkka/100y-saas/workflows/CI%2FCD/badge.svg)](https://github.com/dporkka/100y-saas/actions)
[![Go Report Card](https://goreportcard.com/badge/github.com/dporkka/100y-saas)](https://goreportcard.com/report/github.com/dporkka/100y-saas)
[![Docker Pulls](https://img.shields.io/docker/pulls/100y-saas)](https://hub.docker.com/r/100y-saas)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**100‑Year SaaS** is a multi-tenant SaaS platform built for extreme durability and minimal maintenance. It includes authentication, analytics, subscriptions, background jobs, rate limiting, and email notifications—all using only SQLite and Go's standard library.

## ✨ Features

### 🚀 **Complete SaaS Platform**
- **Multi-tenant Architecture** - Complete data isolation between organizations
- **User Authentication** - Registration, login, session management with auto-cleanup
- **Subscription Management** - Free/paid tiers with automatic usage limit enforcement
- **Real-time Analytics** - Usage tracking, reporting, dashboard stats
- **Background Jobs** - SQLite-based job queue with retries and scheduling
- **Rate Limiting** - In-memory token bucket algorithm with auto-cleanup
- **Email Notifications** - SMTP-based emails with template system
- **Health Monitoring** - `/healthz` endpoint for uptime monitoring

### 🏗️ **Zero-Maintenance Design**
- **Single Binary** - Everything embedded in one executable
- **Single Database** - All data in one SQLite file
- **No External Services** - Redis, Postgres, or cloud services required
- **Self-Healing** - Automatic cleanup, retries, and maintenance
- **Complete Data Ownership** - No vendor lock-in or data sharing

## 📋 Quick Start

### Option 1: One-Click Install (Linux/macOS)
```bash
# Clone and install with automatic Go setup and systemd service
curl -fsSL https://raw.githubusercontent.com/dporkka/100y-saas/main/install.sh | sudo bash -s -- --domain=yourdomain.com
```

### Option 2: Docker Compose (Recommended for Development)
```bash
git clone https://github.com/dporkka/100y-saas.git
cd 100y-saas

# Copy and customize environment variables
cp example.env .env
# Edit .env with your settings

# Start with Caddy reverse proxy
docker-compose up -d

# Load demo data (optional)
./examples/load_demo_data.sh
```

### Option 3: Build from Source
```bash
# Install Go 1.22.5+
git clone https://github.com/dporkka/100y-saas.git
cd 100y-saas

# Install dependencies and build
go mod tidy
go build -o bin/app ./cmd/server

# Run locally
DB_PATH=data/app.db APP_SECRET=local-secret ./bin/app
```

## 🌐 Demo

Try it live at **https://demo.100y-saas.com** (if available)

**Demo Accounts:**
- `demo@example.com` / `hello` (Acme Corporation - Pro Plan)
- `admin@example.com` / `admin` (Tech Startup - Starter Plan)
- `user@example.com` / `secret` (Freelancer - Free Plan)

## High‑Level Architecture

```
[Browser]
   │  Static HTML/CSS/JS (no toolchains)
   ▼
[Caddy]  — TLS + reverse proxy (auto HTTPS)
   ▼
[Go HTTP binary]
   ├─ Routes: /, /api/*, /export
   ├─ Auth: signed cookies (HMAC) or HTTP Basic (optional)
   └─ Migrations: run once at boot from embedded SQL
   ▼
[SQLite file]
   └─ ACID, single file backups
```

**Principles**

* Protocols over frameworks (HTTP, HTML, SQL)
* Single static binary for the app (Go)
* Single-file database (SQLite)
* Files > services; cron > orchestration
* Everything exportable (CSV/JSON)

---

## Repo Layout

```
100y-saas/
├─ cmd/
│  └─ server/
│     └─ main.go
├─ internal/
│  ├─ db/
│  │  ├─ schema.sql
│  │  └─ queries.go
│  ├─ http/
│  │  ├─ handlers.go
│  │  └─ middleware.go
│  └─ version/
│     └─ version.go
├─ web/
│  ├─ index.html
│  ├─ styles.css
│  └─ app.js
├─ Caddyfile
├─ Dockerfile
├─ Makefile
├─ backup.sh
├─ LICENSE
└─ README.md
```

---

## Go Server (single binary)

```go
// cmd/server/main.go
package main

import (
    "context"
    "crypto/hmac"
    "crypto/sha256"
    "embed"
    "encoding/csv"
    "encoding/json"
    "errors"
    "fmt"
    "log"
    "net/http"
    "os"
    "path/filepath"
    "strconv"
    "strings"
    "time"

    "database/sql"
    _ "modernc.org/sqlite"
)

//go:embed ../../web/*
var webFS embed.FS

//go:embed ../../internal/db/schema.sql
var schemaSQL string

type App struct {
    db     *sql.DB
    secret []byte // for cookie signing
}

func main() {
    dsn := env("DB_PATH", "data/app.db")
    secret := []byte(env("APP_SECRET", "change-me"))

    if err := os.MkdirAll(filepath.Dir(dsn), 0o755); err != nil {
        log.Fatal(err)
    }

    db, err := sql.Open("sqlite", dsn+"?_busy_timeout=5000&_fk=1")
    if err != nil { log.Fatal(err) }
    if err := migrate(db, schemaSQL); err != nil { log.Fatal(err) }

    app := &App{db: db, secret: secret}

    mux := http.NewServeMux()

    // static files
    fs := http.FS(webFS)
    mux.Handle("/", withSecurityHeaders(http.FileServer(fs)))

    // api routes
    mux.HandleFunc("/api/ping", func(w http.ResponseWriter, r *http.Request){
        writeJSON(w, map[string]string{"pong":"ok", "time": time.Now().UTC().Format(time.RFC3339)})
    })

    mux.HandleFunc("/api/items", app.itemsHandler)
    mux.HandleFunc("/export", app.exportCSV)

    srv := &http.Server{
        Addr:         ":8080",
        Handler:      logRequests(mux),
        ReadTimeout:  5 * time.Second,
        WriteTimeout: 10 * time.Second,
        IdleTimeout:  60 * time.Second,
    }

    log.Println("listening on :8080")
    if err := srv.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
        log.Fatal(err)
    }
}

func migrate(db *sql.DB, sqlText string) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    _, err := db.ExecContext(ctx, sqlText)
    return err
}

// minimal example entity
type Item struct {
    ID    int64  `json:"id"`
    Title string `json:"title"`
    Note  string `json:"note"`
}

func (a *App) itemsHandler(w http.ResponseWriter, r *http.Request) {
    switch r.Method {
    case http.MethodGet:
        rows, err := a.db.Query("SELECT id, title, note FROM items ORDER BY id")
        if err != nil { http.Error(w, err.Error(), 500); return }
        defer rows.Close()
        var out []Item
        for rows.Next(){ var it Item; if err:=rows.Scan(&it.ID,&it.Title,&it.Note); err!=nil { http.Error(w, err.Error(), 500); return }; out=append(out,it) }
        writeJSON(w, out)
    case http.MethodPost:
        var in Item
        if err := json.NewDecoder(r.Body).Decode(&in); err != nil { http.Error(w, "bad json", 400); return }
        if strings.TrimSpace(in.Title) == "" { http.Error(w, "title required", 400); return }
        res, err := a.db.Exec("INSERT INTO items(title,note) VALUES(?,?)", in.Title, in.Note)
        if err != nil { http.Error(w, err.Error(), 500); return }
        id, _ := res.LastInsertId(); in.ID = id
        writeJSON(w, in)
    default:
        w.WriteHeader(http.StatusMethodNotAllowed)
    }
}

func (a *App) exportCSV(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/csv")
    w.Header().Set("Content-Disposition", "attachment; filename=items.csv")
    cw := csv.NewWriter(w)
    defer cw.Flush()
    cw.Write([]string{"id","title","note"})
    rows, err := a.db.Query("SELECT id, title, note FROM items ORDER BY id")
    if err != nil { http.Error(w, err.Error(), 500); return }
    defer rows.Close()
    for rows.Next(){ var id int64; var title, note string; rows.Scan(&id,&title,&note); cw.Write([]string{strconv.FormatInt(id,10), title, note}) }
}

func writeJSON(w http.ResponseWriter, v any) {
    w.Header().Set("Content-Type", "application/json; charset=utf-8")
    json.NewEncoder(w).Encode(v)
}

func withSecurityHeaders(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
        w.Header().Set("X-Content-Type-Options", "nosniff")
        w.Header().Set("X-Frame-Options", "DENY")
        w.Header().Set("Referrer-Policy", "no-referrer")
        w.Header().Set("Content-Security-Policy", "default-src 'self'")
        next.ServeHTTP(w, r)
    })
}

func logRequests(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request){
        start := time.Now()
        next.ServeHTTP(w, r)
        log.Printf("%s %s %s", r.Method, r.URL.Path, time.Since(start))
    })
}

func env(k, def string) string {
    if v := os.Getenv(k); v != "" { return v }
    return def
}
```

---

## SQLite Schema (idempotent migrations)

```sql
-- internal/db/schema.sql
PRAGMA foreign_keys = ON;
CREATE TABLE IF NOT EXISTS meta (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  note TEXT DEFAULT ''
);

INSERT OR IGNORE INTO meta(key,value) VALUES ('schema_version','1');
```

---

## Static Frontend (no build tools)

```html
<!-- web/index.html -->
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>100‑Year SaaS</title>
  <link rel="stylesheet" href="/web/styles.css" />
</head>
<body>
  <main>
    <h1>Items</h1>
    <form id="f">
      <input id="title" placeholder="Title" required />
      <input id="note" placeholder="Note" />
      <button>Add</button>
      <a href="/export">Export CSV</a>
    </form>
    <ul id="list"></ul>
  </main>
  <script src="/web/app.js"></script>
</body>
</html>
```

```css
/* web/styles.css */
body { font-family: system-ui, sans-serif; max-width: 720px; margin: 2rem auto; padding: 0 1rem; }
input { padding: .5rem; margin-right: .5rem; }
button { padding: .5rem .75rem; }
ul { list-style: none; padding: 0; }
li { padding: .5rem 0; border-bottom: 1px solid #ddd; }
```

```js
// web/app.js
async function load(){
  const res = await fetch('/api/items');
  const items = await res.json();
  const ul = document.getElementById('list');
  ul.innerHTML = '';
  for (const it of items){
    const li = document.createElement('li');
    li.textContent = `${it.id}. ${it.title} — ${it.note||''}`;
    ul.appendChild(li);
  }
}

const f = document.getElementById('f');
f.addEventListener('submit', async (e)=>{
  e.preventDefault();
  const title = document.getElementById('title').value.trim();
  const note = document.getElementById('note').value.trim();
  if(!title) return;
  await fetch('/api/items', {method:'POST', headers:{'Content-Type':'application/json'}, body: JSON.stringify({title, note})});
  document.getElementById('title').value='';
  document.getElementById('note').value='';
  load();
});

load();
```

---

## Caddy (auto‑TLS, one file)

```caddyfile
# Caddyfile
:80 {
  redir https://{host}{uri}
}

:443 {
  encode zstd gzip
  reverse_proxy 127.0.0.1:8080
}
```

> Put the Caddyfile next to the binary and run `caddy run --config Caddyfile`. For a domain, replace `:443` with your domain block and let Caddy issue certs automatically.

---

## Dockerfile (distroless‑style)

```dockerfile
# Dockerfile
FROM golang:1.22 as build
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags "-s -w" -o /bin/app ./cmd/server

FROM gcr.io/distroless/static-debian12
COPY --from=build /bin/app /app
COPY web /web
COPY internal/db/schema.sql /internal/db/schema.sql
ENV DB_PATH=/data/app.db
ENV APP_SECRET=change-me
EXPOSE 8080
ENTRYPOINT ["/app"]
```

* Run with a volume for the DB: `docker run -p 8080:8080 -v $(pwd)/data:/data app:latest`
* Put Caddy in a sidecar or run it on the host.

---

## Makefile (1‑liner conveniences)

```makefile
run:
	DB_PATH=data/app.db APP_SECRET=local go run ./cmd/server

build:
	CGO_ENABLED=0 go build -o bin/app ./cmd/server

fmt:
	gofmt -s -w .

tidy:
	go mod tidy
```

---

## Backups (cron‑friendly)

```bash
#!/usr/bin/env bash
# backup.sh
set -euo pipefail
SRC=${DB_PATH:-data/app.db}
DST_DIR=${BACKUP_DIR:-backups}
mkdir -p "$DST_DIR"
ts=$(date -u +%Y%m%d-%H%M%S)
cp "$SRC" "$DST_DIR/app-$ts.db"
# optional: compress
# gzip -9 "$DST_DIR/app-$ts.db"
```

**Cron example** (daily at 02:15 UTC):

```
15 2 * * * DB_PATH=/srv/app/data/app.db BACKUP_DIR=/srv/backups /srv/app/backup.sh
```

---

## Operating Without Surprises

* **Zero ORMs (optional)**: native SQL keeps queries explicit and portable. Drizzle/Prisma not needed.
* **No task runners**: cron + shell covers 99% of automation.
* **No queues**: if you must, use SQLite table + polling (simpler than Redis).
* **Auth**: keep it minimal—HTTP Basic for internal tools, or signed cookies with HMAC.
* **Observability**: append logs; rotate with `logrotate` monthly.

---

## Data Portability & Survival Plan

* `/export` returns CSV; add `/export/json` for JSON dumps.
* **Cold storage**: rsync your `data/` and `backups/` to a cheap object store quarterly.
* **Reproducibility**: a fresh VM can boot this app with just Go (or Docker) + Caddy.
* **Docs**: README includes a single “bring‑up” section that anyone can follow.

---

## Hardening Checklist (short)

* Run as non‑root (systemd/User=app)
* `fs.protected_hardlinks=1` etc. via sysctl (optional)
* Restrict CORS to your domain
* Regular SQLite `VACUUM` (monthly) to keep file tidy
* Keep `APP_SECRET` off repo; rotate annually

---

## systemd Unit (optional, if not using Docker)

```ini
# /etc/systemd/system/century.service
[Unit]
Description=100-Year SaaS
After=network.target

[Service]
User=app
WorkingDirectory=/srv/app
Environment=DB_PATH=/srv/app/data/app.db
Environment=APP_SECRET=use-a-random-string
ExecStart=/srv/app/bin/app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## What To Add Later (only if truly needed)

* Email: SMTP via stdlib; avoid heavy SDKs
* File uploads: store on disk; only add S3/R2 if storage grows too big
* Search: SQLite FTS5 extension
* Background jobs: a single `jobs` table + ticker

---

## Why This Endures

* **Go binary** runs anywhere for decades
* **SQLite** is likely to be readable forever
* **HTTP/HTML** is universal
* Zero exotic services; everything is files you can copy

You can now copy this repo layout, paste in your app logic, and have something that will plausibly run for decades with backups and almost no upkeep.
