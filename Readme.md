# Better than distroless images - scratch images

- Only works if your app compiles to a fully static binary (Go, Rust, C with musl).
  
## Example 

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
