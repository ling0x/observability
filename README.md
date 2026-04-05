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

Nginx is the only publicly exposed entry point. It routes traffic to Grafana and Prometheus, and forwards OTLP traffic to the collector. Basic-auth is enforced via `.htpasswd`.

## Getting Started

**Prerequisites:** Docker and Docker Compose.

1. Clone the repository:
   ```bash
   git clone https://github.com/ling0x/observability.git
   cd observability
   ```

2. Create a `.env` file (see the environment variables section below).

3. Generate an `.htpasswd` file for Nginx basic auth:
   ```bash
   htpasswd -c nginx/.htpasswd <username>
   ```

4. Start the stack:
   ```bash
   docker compose up -d
   ```

5. Open Grafana at `http://localhost:8081`.

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

The collector accepts **OTLP over HTTP** and automatically routes metrics, logs, and traces to the appropriate backend.

## Configuration Files

All service configs live in `configs/`:

- `otel-collector-config.yml` — receiver, processor, and exporter pipeline
- `prometheus.yml` — scrape targets
- `loki-config.yml` — Loki storage and ingestion settings
- `tempo.yml` — Tempo storage backend config
- `grafana/provisioning/datasources/datasources.yaml` — auto-provisions Prometheus, Loki, and Tempo as Grafana data sources
