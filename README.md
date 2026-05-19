# Keycloak Observability Stack

Local observability stack: Keycloak + Postgres + Prometheus + Grafana with OAuth/OIDC authentication for Grafana via Keycloak. Suitable as a reference for verifying Keycloak metrics and exercising the OAuth flow.

> [!WARNING]
> This is a **dev configuration**: Keycloak runs in `start-dev`, Postgres uses `tmpfs` (data does not survive a restart), default passwords live in `.env`. For production use, see the [Production checklist](#production-checklist).

## Architecture

```
                                  ┌──────────────────────┐
   Browser ──── http://localhost:3000 (GF_SERVER_HTTP_PORT) ─→ Grafana
      │                           └─────────┬────────────┘
      │                          OAuth      │ backchannel (token/userinfo)
      │                       (frontchannel)│  http://keycloak:8080 (docker DNS)
      ▼                                     ▼
   http://localhost:8080 (KC_PORT) ───→ Keycloak (start-dev, realm-import)
                                          │ │
                          metrics :9000 ──┘ └── JDBC :5432 ──→ Postgres (tmpfs)
                              │
                              ▼
                          Prometheus  ──── scrape 15s ──┐
                              ▲                         │
                              └── Grafana datasource ◄──┘
```

Version changes are made via `.env`.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) ≥ 24.x
- [Docker Compose v2](https://docs.docker.com/compose/install/) (`docker compose`, not `docker-compose`)
- Free ports: `3000`, `8080`, `9090` (or override in `.env`)

## Quick Start

```sh
git clone git@github.com:ML-ZoneReaper/keycloak-compose.git
cd keycloak-compose

# Start with health checks
docker compose up -d
docker compose ps   # all services should be in healthy status

# Live logs
docker compose logs -f
```

The first start takes ~60 seconds (Keycloak imports the realm and runs migrations against an empty DB). Healthchecks with `start_period: 60s` handle this correctly — Grafana waits for Keycloak readiness thanks to `depends_on.condition: service_healthy`.

## Access points

| Service    | URL                                | Credentials (default) |
|------------|------------------------------------|-----------------------|
| Grafana    | http://localhost:3000              | OAuth via Keycloak    |
| Keycloak   | http://localhost:8080              | `admin` / `keycloak`  |
| Prometheus | http://localhost:9090              | no authentication     |

The Grafana login form is disabled (`GF_AUTH_DISABLE_LOGIN_FORM=true`). Sign-in is only via the "Sign in with Keycloak" button → user `admin` / `grafana`.

## Key features

- **OAuth (PKCE)** — Grafana acts as a public client with PKCE `S256`, no client_secret.
- **Auto-provisioned dashboards** — Grafana pulls dashboards from `grafana/dashboards/` via provisioning. The datasource UID is hardcoded (`P02FBFF047EDBB13A`) and matches between `datasources.yml` and the dashboard JSON.
- **Realm import** — `keycloak/realm.json` is imported at startup, with `${VAR}` substitution from env.
- **JVM/Agroal/JGroups metrics** — exposed on management port `9000`, scraped by Prometheus every 15s, 30d retention.
- **Healthchecks with `start_period`** — correct startup ordering, `depends_on.condition: service_healthy`.
- **Security baseline** — `no-new-privileges:true` on all services, custom docker network, containers with explicit names.

## OAuth flow (important to know)

The main classic pitfall is which URLs to use for OAuth endpoints.

| Endpoint             | Who calls it    | URL                                                  |
|----------------------|-----------------|------------------------------------------------------|
| `auth_url`           | browser (UA)    | external `${KC_HOSTNAME}:${KC_PORT}` → `http://localhost:8080` |
| `token_url`          | Grafana → KC    | internal `http://keycloak:8080` (docker DNS)         |
| `api_url` (userinfo) | Grafana → KC    | internal `http://keycloak:8080` (docker DNS)         |
| `signout_redirect`   | browser (UA)    | external `${KC_HOSTNAME}:${KC_PORT}`                 |

If you point `api_url`/`token_url` at `localhost`, Grafana will resolve `localhost` inside its own container and OAuth will break. On the backchannel, Keycloak's internal port is **always 8080**, regardless of which host port it's mapped to via `${KC_PORT}`.

## Metrics

Endpoint: `http://keycloak:9000/metrics` (inside the docker network), enabled via `KC_METRICS_ENABLED=true` + `KC_HEALTH_ENABLED=true` (the latter is required to open management port 9000).

The `keycloak-general.json` dashboard covers:
- **JVM:** heap (used/committed/max), GC pause count/duration, threads, classloader
- **Agroal (connection pool):** idle/acquired/awaiting connections, leak detection, acquisition time
- **System:** CPU, load average, available processors

## Management commands

```sh
# Full restart with rebuild
docker compose down && docker compose up -d

# Tear down along with volumes (tmpfs is ephemeral anyway)
docker compose down -v

# Restart a single service (e.g. after editing realm.json)
docker compose restart keycloak

# Service health
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Health}}"

# Logs for a specific service
docker compose logs -f keycloak

# Verify that Prometheus sees the Keycloak target
curl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health}'
```

## Troubleshooting

### Grafana dashboard is empty ("No data" in panels)

1. Check targets: `curl -s localhost:9090/api/v1/targets | jq '.data.activeTargets[].health'` — they should all be `up`.
2. Verify that the datasource UID in `grafana/datasources/datasources.yml` matches the UID referenced by the dashboard panels (`P02FBFF047EDBB13A`).
3. In the Grafana UI: Configuration → Data sources → Prometheus → Save & test.

### Grafana refuses to authenticate via OAuth

Most often this is a URL mismatch. Set `GF_LOG_LEVEL=debug` in `.env`, restart grafana, and inspect `docker compose logs grafana | grep -i oauth`.

Typical cases:
- `redirect_uri mismatch` → `redirectUris` in `realm.json` doesn't match Grafana's actual callback (`/login/generic_oauth`).
- `connection refused` to token_url → `token_url` is using `localhost` instead of `keycloak`.
- `invalid issuer` → `KC_HOSTNAME` is misconfigured.

### Keycloak takes a long time to start / healthcheck fails

A first start with realm import and migrations against an empty DB can take up to 60 seconds. If `start_period: 60s` is not enough (slow machine), increase it in `compose.yml`.

### Port conflict

```sh
lsof -i :3000 -i :8080 -i :9090
```
Override via `.env` (`KC_PORT`, `GF_SERVER_HTTP_PORT`, `PROMETHEUS_PORT`).

## Production checklist

Before using this anywhere other than local dev:

- [ ] Change **all** default passwords in `.env` (Postgres, Keycloak bootstrap admin, Grafana admin)
- [ ] Switch `start-dev` → `start` in Keycloak's `command`, explicitly configure `KC_HOSTNAME`, `KC_HOSTNAME_STRICT=true`, `KC_PROXY_HEADERS=xforwarded` (if behind a reverse proxy)
- [ ] TLS across the whole perimeter; remove `GF_AUTH_GENERIC_OAUTH_TLS_SKIP_VERIFY_INSECURE`, set `sslRequired: external`/`all` in the realm
- [ ] Persistent storage for Postgres instead of tmpfs (named volume + backups; CloudNativePG for HA)
- [ ] Secrets via Vault / Docker secrets / SOPS, not from `.env`
- [ ] Keycloak — confidential client (with client_secret) instead of public + PKCE for sensitive realms
- [ ] Remove the bootstrap admin after first launch, create personal admin accounts
- [ ] Resource limits (`deploy.resources.limits.memory/cpus`)
- [ ] Prometheus — external `remote_write` to VictoriaMetrics/Thanos/Mimir; local 30d retention is not viable for production load
- [ ] Alerts (Alertmanager) on JVM heap saturation, GC pause spikes, Agroal pool exhaustion, scrape errors

## Resources

- [Keycloak docs](https://www.keycloak.org/documentation)
- [Keycloak metrics](https://www.keycloak.org/observability/configuration-metrics)
- [Grafana OAuth (generic)](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/configure-authentication/generic-oauth/)
- [Prometheus query basics](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Docker Compose spec](https://docs.docker.com/compose/compose-file/)
