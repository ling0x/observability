# Observability Stack

A self-hosted, Docker Compose-based observability stack covering the three pillars of observability — **metrics**, **logs**, and **traces** — with a single unified UI.

## Stack

| Component | Role |
|---|---|
| [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) | Receives telemetry from apps via OTLP and fans out to backends |
| [Prometheus](https://prometheus.io/) | Metrics storage and scraping |
| [Loki](https://grafana.com/oss/loki/) | Log aggregation |
| [Tempo](https://grafana.com/oss/tempo/) | Distributed trace storage |
| [Grafana](https://grafana.com/) | Unified dashboards and data exploration |
| [Nginx](https://nginx.org/) | Reverse proxy with basic-auth, exposes the stack on a single port |

## Architecture

```
Your App
   │
   └─ OTLP/HTTP (:14318 public / :4318 internal)
         │
   OTel Collector
   ├── metrics ──► Prometheus
   ├── logs    ──► Loki
   └── traces  ──► Tempo
                      │
                   Grafana  ◄── Nginx (:8081)
```

Nginx is the only publicly exposed entry point. It serves Grafana at `/grafana/` and Prometheus at `/prometheus/`, and accepts OTLP traffic on a separate port (no basic-auth) restricted to RFC1918 addresses. Basic-auth is enforced on all UI routes via `nginx/.htpasswd`.

## Getting Started

**Prerequisites:** Docker and Docker Compose.

1. Clone the repository:
   ```bash
   git clone https://github.com/ling0x/observability.git
   cd observability
   ```

2. Create a `.env` file (see the environment variables section below).

3. Set up Nginx basic auth by copying the example and replacing it with a real hashed password:
   ```bash
   cp nginx/.htpasswd.example nginx/.htpasswd
   htpasswd nginx/.htpasswd <username>
   ```

4. Start the stack:
   ```bash
   docker compose up -d
   ```

5. Open Grafana at `http://localhost:8081/grafana/` (redirected automatically from `/`).

## Environment Variables

Create a `.env` file in the project root:

```env
# Grafana
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=changeme

# Nginx port (default: 8081)
NGINX_HTTP_PORT=8081

# Public OTLP endpoint (default: 0.0.0.0:14318)
OTEL_PUBLIC_OTLP_HOST=0.0.0.0
OTEL_PUBLIC_OTLP_PORT=14318

# Prometheus external URL (used for correct redirects behind Nginx)
PROMETHEUS_EXTERNAL_URL=http://localhost:8081/prometheus/
```

## Sending Telemetry

Point your application's OTLP exporter at the public collector endpoint:

```
http://<host>:14318
```

The OTLP port bypasses basic-auth but is restricted to private/loopback IP ranges (`127.0.0.1`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`). Restrict this further to specific IPs in production.

## Configuration Files

All service configs live in `configs/` and Nginx configs in `nginx/`:

| File | Purpose |
|---|---|
| `configs/otel-collector-config.yml` | Receiver, processor, and exporter pipeline |
| `configs/prometheus.yml` | Scrape targets |
| `configs/loki-config.yml` | Loki storage and ingestion settings |
| `configs/tempo.yml` | Tempo storage backend config |
| `configs/grafana/provisioning/datasources/datasources.yaml` | Auto-provisions Prometheus, Loki, and Tempo in Grafana |
| `nginx/nginx.conf` | Reverse proxy routing and OTLP IP allowlist |
| `nginx/.htpasswd.example` | Example htpasswd file — copy to `nginx/.htpasswd` and populate |
