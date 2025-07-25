# Keycloak with PostgreSQL, Prometheus & Grafana

Containerized solution for running Keycloak with PostgreSQL, monitoring via Prometheus and Grafana.

## 🚀 Quick Start

1. **Prerequisites**:
   - [Docker](https://docs.docker.com/get-docker/) (v20.10.7+)
   - [Docker Compose](https://docs.docker.com/compose/install/) (v2.0.0+)

2. Clone repository:
   ```bash
   git clone https://github.com/your-username/your-repo.git
   cd your-repo
   ```

3. Start the system:
   ```bash
   docker compose up -d
   ```

4. Access services:
   - Keycloak: http://localhost:8080
   - Grafana: http://localhost:3000
   - Prometheus: http://localhost:9090

## ⚙️ Configuration

Configure using the `.env` file. Key parameters:

| Variable                 | Default       | Description                     |
|--------------------------|---------------|---------------------------------|
| `POSTGRES_DB`            | keycloak      | PostgreSQL database name        |
| `POSTGRES_USER`          | keycloak      | PostgreSQL user                 |
| `POSTGRES_PASSWORD`      | password      | PostgreSQL password             |
| `KEYCLOAK_ADMIN`         | admin         | Keycloak admin username         |
| `KEYCLOAK_ADMIN_PASSWORD`| keycloak      | Keycloak admin password         |
| `GRAFANA_ADMIN_PASSWORD` | grafana       | Grafana admin password          |

Modify `.env` before first launch.

## 🔌 Service Access

| Service        | URL                         | Credentials               |
|----------------|-----------------------------|---------------------------|
| **Keycloak**   | http://localhost:8080       | `admin` / `keycloak`      |
| **Grafana**    | http://localhost:3000       | `admin` / `grafana`       |
| **Prometheus** | http://localhost:9090       | No authentication         |

## 📊 Monitoring

Automatically configured:
- Prometheus scrapes Keycloak metrics every 15s
- Grafana with pre-configured [Keycloak Dashboard](https://grafana.com/grafana/dashboards/10441)
- Ready-to-use dashboards for:
  - JVM metrics
  - Database queries
  - Active sessions
  - Authentication errors

## 🛠 Management

### Core Commands
| Command                                  | Description                                     |
|------------------------------------------|-------------------------------------------------|
| `docker compose up -d`                   | Start all services in background                |
| `docker compose down`                    | Stop services (add `-v` to remove volumes)      |
| `docker compose logs -f`                 | Follow service logs in real-time                |
| `docker compose restart keycloak`        | Restart Keycloak container                      |

### Resource Monitoring
```bash
# Live container resource usage
docker stats

# Check service status
docker compose ps
```

### Keycloak Administration
After launch:
1. Access Keycloak Admin Console
2. Create new realm
3. Configure clients and users

## 🧹 Cleanup
Remove all containers, images, and volumes:
```bash
docker compose down -v --rmi all
docker system prune -a -f --volumes
```

> **Warning**  
> These commands permanently destroy all data!

## ⁉️ Troubleshooting
If services fail to start:
1. Check port conflicts (8080, 9090, 3000)
2. Inspect logs:
   ```bash
   docker compose logs keycloak
   ```
3. Verify no conflicting containers:
   ```bash
   docker ps -a
   ```

## 📚 Resources
- [Keycloak Documentation](https://www.keycloak.org/documentation)
- [Prometheus Querying](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana Keycloak Dashboard](https://grafana.com/grafana/dashboards/10441)
