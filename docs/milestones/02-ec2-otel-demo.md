# Milestone 2: EC2 + Docker Compose — OTel Demo

## Objective
Deploy the OpenTelemetry Demo (15 microservices + observability stack) on a single EC2 instance using Docker Compose. Understand the deployment, the pain points, and what the pipeline needs to handle.

## What We Did

1. Created a security group (`otel-demo-sg`) with ports: 8080 (frontend), 3000 (Grafana), 9090 (Prometheus), 16686 (Jaeger)
2. Created an IAM role + instance profile (`otel-demo-ec2-role` / `otel-demo-profile`) with SSM access
3. Launched `t3.large` EC2 instance with a user-data script
4. Installed Docker, Docker Compose (standalone binary), and git manually via SSM
5. Cloned the OTel Demo repo and ran `docker-compose up -d`
6. Added observability stack via `compose.observability.yaml` merge
7. Created `compose.override.yaml` for fixed ports and Grafana stability

## Architecture (Current State)

```
Internet (your browser)
    │
    ├── :8080 → frontend-proxy (Envoy) → web store
    ├── :3000 → Grafana (dashboards)
    ├── :9090 → Prometheus (metrics)
    └── :16686 → Jaeger (traces)

Inside EC2 (Docker Compose network):
┌─────────────────────────────────────────────────────┐
│  15 microservices (cart, checkout, payment, etc.)    │
│       │ OTLP (gRPC :4317)                           │
│       ▼                                             │
│  OTel Collector                                      │
│       │                                             │
│       ├── metrics → Prometheus (scrape)             │
│       ├── traces → Jaeger                           │
│       └── logs → OpenSearch (broken, not needed)    │
│                                                     │
│  Grafana ← queries Prometheus + Jaeger              │
│  Load Generator → simulates traffic                 │
└─────────────────────────────────────────────────────┘
```

## Everything That Went Wrong + How We Fixed It

### Issue 1: User-data script failed silently
**Symptom:** Instance launched, cloud-init finished in 10s, no Docker installed.
**Root cause:** `yum install -y docker` doesn't work on Amazon Linux 2023 — package exists but the dnf transaction got stuck because the user-data script and interactive session both tried to install simultaneously (PID lock).
**Fix:** Killed the stuck PID (`sudo kill <pid>`), re-ran `sudo dnf install -y docker` manually.
**Pipeline fix:** Use a proper AMI with Docker pre-installed, or a robust user-data script with `flock` to prevent concurrent package operations. Add a health check that waits for Docker to be running before proceeding.

### Issue 2: docker-compose-plugin not available in AL2023
**Symptom:** `sudo dnf install -y docker-compose-plugin` → "No match for argument"
**Root cause:** Amazon Linux 2023 repos don't include the Docker Compose plugin package.
**Fix:** Downloaded the standalone binary directly from GitHub releases.
**Pipeline fix:** Pin the exact Docker Compose version in the setup script. Use the direct download URL with a specific version (not `latest` which can break).

### Issue 3: Git not installed
**Symptom:** `git: command not found`
**Root cause:** AL2023 minimal AMI doesn't include git by default.
**Fix:** `sudo dnf install -y git`
**Pipeline fix:** Include all dependencies in the setup script: `dnf install -y docker git`

### Issue 4: Disk full — root volume only 8GB
**Symptom:** docker-compose pull failed midway, no space left.
**Root cause:** Default EBS root volume is 8GB. 20+ Docker images need ~15GB.
**Fix:** Expanded EBS volume to 30GB live (`aws ec2 modify-volume` + `growpart` + `xfs_growfs`).
**Pipeline fix:** Specify `--block-device-mappings` in `run-instances` to launch with 30GB from the start.

### Issue 5: OpenSearch unhealthy — vm.max_map_count too low
**Symptom:** `dependency failed to start: container opensearch is unhealthy`
**Root cause:** Linux default `vm.max_map_count=65536`, OpenSearch requires 262144.
**Fix:** `sudo sysctl -w vm.max_map_count=262144`
**Pipeline fix:** Include `sysctl -w vm.max_map_count=262144` in user-data AND add to `/etc/sysctl.conf` for persistence across reboots.

### Issue 6: Grafana crash-loop — alerting provisioning failure
**Symptom:** Grafana container keeps restarting. Exit code 1.
**Root cause:** Grafana provisions alerting contact points from config files, expects SMTP to be configured. When provisioning fails, Grafana treats it as fatal and exits.
**Fix:** Removed alerting provisioning files (`rm -rf src/grafana/provisioning/alerting`), created empty dir, disabled alerting via env vars.
**Pipeline fix:** Ship a custom `compose.override.yaml` that disables alerting and skips OpenSearch plugin. Don't rely on upstream defaults.

### Issue 7: Grafana redirect loop — subpath configuration
**Symptom:** Browser shows blank page, curl returns 301 redirect to `/grafana/`
**Root cause:** OTel Demo configures Grafana to serve under `/grafana/` subpath (designed for the Envoy frontend-proxy). Direct access on :3000 triggers a redirect loop.
**Fix:** Override `GF_SERVER_ROOT_URL` and `GF_SERVER_SERVE_FROM_SUB_PATH=false`.
**Pipeline fix:** Always set `GF_SERVER_ROOT_URL` to the actual access URL (Elastic IP or DNS name).

### Issue 8: Grafana "failed to load application files"
**Symptom:** Grafana login page loads but assets (JS/CSS) fail to load.
**Root cause:** `GF_SERVER_ROOT_URL` was set to `localhost:3000` but accessed via public IP. Grafana generates asset URLs based on root_url.
**Fix:** Set `GF_SERVER_ROOT_URL=http://<public-ip>:3000/`
**Pipeline fix:** Use Elastic IP or DNS name so the URL is stable. Inject it as a variable during deployment.

### Issue 9: Dynamic ports on every restart
**Symptom:** Ports change every `docker-compose up`, requiring SG updates.
**Root cause:** Compose file uses variable port mappings (`${GRAFANA_PORT}`) without fixed host binding.
**Fix:** Created `compose.override.yaml` with fixed port mappings (`3000:3000`, `16686:16686`, `9090:9090`).
**Pipeline fix:** Always ship a compose override with explicit port bindings. Never rely on dynamic ports in non-dev environments.

### Issue 10: IP changes on instance stop/start
**Symptom:** After stopping and starting the instance, public IP changed.
**Root cause:** Default EC2 behavior — public IPs are ephemeral unless you assign an Elastic IP.
**Pipeline fix:** Allocate an Elastic IP and associate it at launch time. Or use a DNS name (Route53) that updates via a post-deploy step.

## Compose Files Used

```bash
# Full startup command:
sudo docker-compose \
  -f compose.yaml \
  -f compose.observability.yaml \
  -f compose.override.yaml \
  up -d
```

### compose.override.yaml (our customizations)
```yaml
services:
  grafana:
    ports:
      - "3000:3000"
    environment:
      - GF_INSTALL_PLUGINS=
      - GF_PLUGINS_ALLOW_LOADING_UNSIGNED_PLUGINS=*
      - GF_DATABASE_WAL=true
      - GF_ALERTING_ENABLED=false
      - GF_UNIFIED_ALERTING_ENABLED=false
      - GF_SERVER_ROOT_URL=http://<public-ip>:3000/
      - GF_SERVER_SERVE_FROM_SUB_PATH=false
    user: "0"
  jaeger:
    ports:
      - "16686:16686"
  prometheus:
    ports:
      - "9090:9090"
```

## Post-Boot Checklist (Manual — until pipeline automates it)

```bash
# After every instance start:
sudo sysctl -w vm.max_map_count=262144
sudo systemctl start docker
cd /home/ec2-user/opentelemetry-demo
# Update IP in override if changed
sudo docker-compose -f compose.yaml -f compose.observability.yaml -f compose.override.yaml up -d
```

## Debugging Commands

```bash
# Check all container status
sudo docker-compose -f compose.yaml -f compose.observability.yaml -f compose.override.yaml ps

# Logs for a specific service
sudo docker logs <container-name> 2>&1 | tail -30

# Check for OOM kills
sudo docker inspect <container-name> --format='{{.State.OOMKilled}}'

# Memory usage
free -h
sudo docker stats --no-stream

# Check if a port is reachable locally
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>

# View user-data script output
sudo cat /var/log/cloud-init-output.log

# Check Docker disk usage
sudo docker system df
```

## Key Learnings

1. **User-data scripts are fire-and-forget** — they run once, no retry logic, no feedback. If they fail, you find out 20 minutes later. Pipelines solve this with visibility and retry.
2. **Docker Compose merging** — multiple `-f` flags deep-merge definitions. Base + observability + overrides = final state. Like CSS specificity for infrastructure.
3. **Containers isolate dependencies** — we installed Docker and git. Not Node, Python, Go, Java, .NET, or Ruby (which the 15 services all need). Each container brings its own runtime.
4. **Single-host limitations** — everything is on one box. No redundancy, no independent scaling, IP changes on restart. This is exactly what K8s solves (Milestone 5).
5. **Observability tools need observability** — Grafana itself crashed repeatedly. In production you'd monitor your monitoring (Milestone 8: observability of observability).

## What the Pipeline (Milestone 4) Must Handle

- [ ] Launch with correct volume size (30GB)
- [ ] Install Docker + Compose with retries and version pinning
- [ ] Set kernel params (`vm.max_map_count`) persistently
- [ ] Clone repo and start compose with override
- [ ] Assign Elastic IP for stable access
- [ ] Health check: wait for all services to be healthy before declaring success
- [ ] Output the access URLs at the end of the run
