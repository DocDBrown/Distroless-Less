# Scratch Images

- Only works if your app compiles to a fully static binary (Go, Rust, C with musl).
  
## Compiled Example 

```
# Build stage
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags='-s -w' -o /app/server .

# Final stage — literally nothing
FROM scratch
COPY --from=builder /app/server /server
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
USER 65534:65534
ENTRYPOINT ["/server"]
```

## Non-Compiled Example 

```
# ── Stage 1: Build React frontend ───────────────────────────────────────
FROM node:22-alpine AS frontend
WORKDIR /app
COPY frontend/package.json frontend/package-lock.json ./
RUN npm ci
COPY frontend/ .
RUN npm run build

# ── Stage 2: Build Go service ───────────────────────────────────────────
FROM golang:1.26.1-bookworm AS backend
WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN CGO_ENABLED=0 go build -buildvcs=false -ldflags='-s -w' -o /build/server .

# ── Stage 3: Scratch runtime ───────────────────────────────────────────
FROM scratch AS runtime
COPY --from=backend /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=backend /usr/share/zoneinfo/ /usr/share/zoneinfo/
COPY --from=backend /build/server /app/server
COPY --from=frontend /app/dist /app/static

USER 65534:65534
WORKDIR /app
EXPOSE 8088
ENTRYPOINT ["/app/server"]
```

## Explanation

```
Simple (Go only):
  scratch
  ├── /server              ← binary
  └── /etc/ssl/certs/      ← TLS certs

With React:
  scratch
  ├── /app/server           ← same binary
  ├── /app/static/          ← just files (html, js, css)
  │   ├── index.html
  │   ├── assets/
  │   └── ...
  ├── /etc/ssl/certs/       ← TLS certs
  └── /usr/share/zoneinfo/  ← timezone data
```
