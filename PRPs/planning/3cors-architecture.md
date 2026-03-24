# 3CORS Platform — Full Architecture Document

Generated: 2026-03-23  
Source: Analysis of /home/juergen/3cors-engine/3cors/ and /home/juergen/local-ai-packaged/

---

## 1. High-Level Overview

The 3CORS platform runs as two overlapping Docker Compose stacks on the same host (192.168.178.243):

| Stack | Project Name | Compose Root | Purpose |
|-------|-------------|-------------|---------|
| Supabase + 3CORS | `supabase` | `/home/juergen/3cors-engine/3cors/docker/` | Core backend: Supabase, Foundation, LiteLLM, Keycloak, Auth Gateway, Interviewer, CorIM, N8N gateway |
| local-ai-packaged | `local-ai-packaged` | `/home/juergen/local-ai-packaged/` | AI tooling: N8N (standalone), Open WebUI, Flowise, Langfuse, Qdrant, Neo4j, Ollama, Caddy reverse proxy |

Both stacks share a single Caddy instance (the one in `local-ai-packaged`), which handles all external routing. The 3CORS caddy compose file (`docker-compose.caddy.yml`) defines a separate 3CORS-only Caddy, but it is NOT currently running — only the `local-ai-packaged` caddy is active.

---

## 2. Running Container Inventory (Current State)

### 2.1 Supabase Stack (project: supabase)

| Container | Image | Host Ports | Network(s) | Role |
|-----------|-------|------------|------------|------|
| `supabase-studio` | supabase/studio:2026.01.27 | 3021→3000 | supabase_default | Supabase Dashboard UI |
| `supabase-kong` | kong:2.8.1 | 8014→8000, 8444→8443 | supabase_default | API gateway for all Supabase services |
| `supabase-auth` | supabase/gotrue:v2.185.0 | (none exposed) | supabase_default | GoTrue auth service |
| `supabase-rest` | postgrest/postgrest:v14.3 | (none exposed) | supabase_default | PostgREST REST API |
| `realtime-dev.supabase-realtime` | supabase/realtime:v2.72.0 | (none exposed) | supabase_default | WebSocket realtime |
| `supabase-storage` | supabase/storage-api:v1.33.5 | (none exposed) | supabase_default | File storage API |
| `supabase-imgproxy` | darthsim/imgproxy:v3.30.1 | (none exposed) | supabase_default | Image transformation |
| `supabase-meta` | supabase/postgres-meta:v0.95.2 | (none exposed) | supabase_default | DB meta API |
| `supabase-edge-functions` | supabase/edge-runtime:v1.70.0 | (none exposed) | supabase_default | Deno edge functions |
| `supabase-analytics` | supabase/logflare:1.30.3 | 4000→4000 | supabase_default | Log analytics |
| `supabase-db` | supabase/postgres:15.8.1.085 | 5433→5432 | supabase_default | Primary PostgreSQL |
| `supabase-vector` | timberio/vector:0.28.1 | (none exposed) | supabase_default | Log ingestion |
| `supabase-pooler` | supabase/supavisor:2.7.4 | 5434→5432, 6543→6543 | supabase_default | Connection pooler |

### 2.2 3CORS Application Services (project: supabase, overlaid via override)

| Container | Image | Host Ports | Network(s) | Role |
|-----------|-------|------------|------------|------|
| `3cors-foundation` | supabase-foundation (local build) | 8010→8010 | supabase_default | Foundation cost/billing service |
| `3cors-foundation-admin` | supabase-foundation-admin (local build) | 3003→3003 | supabase_default | Admin dashboard frontend (nginx) |
| `3cors-litellm-proxy` | ghcr.io/berriai/litellm:main-stable | (not exposed) | supabase_default | LiteLLM proxy (internal only) |
| `3cors-litellm-auth` | supabase-litellm-auth-middleware | 4001→4001 | supabase_default, local-ai-packaged_default | LiteLLM auth middleware (secure gateway) |
| `3cors-redis` | redis:7-alpine | 127.0.0.1:6379→6379 | supabase_default | Redis for Foundation |
| `3cors-auth-gateway` | 3cors-auth-gateway (local build) | 3100→3100 | supabase_default | Keycloak-backed gateway for Supabase Studio |
| `3cors-keycloak` | quay.io/keycloak/keycloak:26.1 | 8082→8080 | supabase_default, 3cors-network | Keycloak OIDC identity provider |
| `3cors-interviewer-backend` | 3cors-interviewer-interviewer-backend | 8000→8000 | supabase_default | Interviewer Python API |
| `3cors-interviewer-frontend` | 3cors-interviewer-interviewer-frontend | 3002→3002 | supabase_default | Interviewer Vite frontend |
| `3cors-corim-backend` | 3cors-corim-corim-backend | 8009→8000 | supabase_default, 3cors-network | CorIM Python API |
| `3cors-corim-frontend` | 3cors-corim-corim-frontend | 5174→5173 | supabase_default | CorIM Vite frontend |
| `3cors-n8n-backend` | supabase-n8n-backend | 8011→8000 | supabase_default, 3cors-network | N8N gateway backend API |
| `3cors-n8n-frontend` | supabase-n8n-frontend | 5176→5173 | supabase_default | N8N gateway frontend |

### 2.3 local-ai-packaged Stack

| Container | Image | Host Ports | Network(s) | Role |
|-----------|-------|------------|------------|------|
| `caddy` | caddy:2-alpine | 80, 443, 8001-8008 | local-ai-packaged_default | Main reverse proxy for ALL traffic |
| `n8n` | n8nio/n8n:latest | (internal only) | local-ai-packaged_default | Standalone N8N workflow engine |
| `open-webui` | ghcr.io/open-webui/open-webui | (internal only) | local-ai-packaged_default | Open WebUI for LLM chat |
| `local-ai-packaged-postgres-1` | postgres:17 | (internal only) | local-ai-packaged_default | Postgres for Langfuse/N8N |
| `redis` | valkey/valkey:8-alpine | (internal only) | local-ai-packaged_default | Redis for Langfuse |

### 2.4 Other Active Containers

| Container | Ports | Role |
|-----------|-------|------|
| `remote-coding-agent-app-with-db-1` | 5175→5175 | Remote coding agent chat UI |

---

## 3. Docker Network Topology

### Networks

| Network Name | Driver | Subnet | Who Uses It |
|-------------|--------|--------|-------------|
| `supabase_default` | bridge | (default) | ALL supabase + 3cors containers |
| `local-ai-packaged_default` | bridge | (default) | caddy, n8n, open-webui, local postgres, langfuse, 3cors-litellm-auth |
| `3cors-network` | bridge | 172.20.0.0/16 | 3cors-keycloak, 3cors-corim-backend, 3cors-n8n-backend |
| `remote-coding-agent_default` | bridge | (default) | remote-coding-agent |

### Critical Cross-Stack Bridge

`3cors-litellm-auth` is the **only container in both `supabase_default` and `local-ai-packaged_default`**. This is intentional — it bridges the two stacks so Caddy (in local-ai-packaged_default) could reach the auth middleware, and the auth middleware can reach Supabase and Foundation (in supabase_default).

### Network Communication Matrix

```
supabase_default (all 3CORS services)
├── supabase-kong (kong:8000 internal)
├── supabase-auth (auth:9999)
├── supabase-rest (rest:3000)
├── supabase-db (db:5434)
├── supabase-analytics (analytics:4000)
├── 3cors-foundation (foundation:8010)
├── 3cors-litellm-proxy (litellm-proxy:4000) ← no external port
├── 3cors-litellm-auth ← BRIDGED to local-ai-packaged_default
├── 3cors-redis (redis:6379)
├── 3cors-auth-gateway (port 3100)
├── 3cors-keycloak ← ALSO in 3cors-network
├── 3cors-interviewer-backend (port 8000)
├── 3cors-interviewer-frontend (port 3002)
├── 3cors-corim-backend ← ALSO in 3cors-network
├── 3cors-corim-frontend
├── 3cors-n8n-backend ← ALSO in 3cors-network
└── 3cors-n8n-frontend

local-ai-packaged_default
├── caddy (ONLY external entry point: 80/443)
├── n8n (n8n:5678)
├── open-webui (open-webui:8080)
├── local postgres (postgres:5432)
├── 3cors-litellm-auth ← BRIDGED from supabase_default
└── [langfuse, qdrant, neo4j when running]

3cors-network (isolated, used by some services)
├── 3cors-keycloak
├── 3cors-corim-backend
└── 3cors-n8n-backend
```

---

## 4. Domain Routing Map

All public traffic arrives at Caddy via Cloudflare Tunnel. Cloudflare handles TLS termination publicly; Caddy routes internally.

### Active Caddy Configuration (from `/home/juergen/local-ai-packaged/Caddyfile` + caddy-addon/*.conf)

The main Caddyfile imports all `caddy-addon/*.conf` files. Active addon configs:

#### From `3cors-corim/caddy/Caddyfile` (used by the 3CORS-specific Caddy in `docker-compose.caddy.yml`, currently NOT running as separate container):

| Domain | Routing | Backend Service |
|--------|---------|----------------|
| `corimd.hcai42.com` | `localhost:5176` | (NOTE: port mismatch — CorIM frontend is on 5174, N8N frontend is on 5176) |
| `auth.corimd.hcai42.com` | `localhost:9999` | GoTrue (not exposed on that port — potential issue) |
| `hack.hcai42.com` | `/auth/*` → `localhost:8014`, `/rest/*` → `localhost:8014`, `/api/*` → `localhost:8000`, `/` → `localhost:3002` | Kong (8014), Interviewer backend (8000), Interviewer frontend (3002) |
| `3002.vcj.hcai42.com` | `localhost:3002` | Interviewer frontend |
| `supa.hcai42.com` | `localhost:3100` | Auth Gateway → Supabase Studio |
| `monitor.hcai42.com` | `localhost:3001` | Uptime Kuma (port 3001, not currently running) |

#### From `local-ai-packaged/caddy-addon/*.conf`:

| Domain | Caddy Addon File | Backend | Notes |
|--------|-----------------|---------|-------|
| `n8n.hcai42.com` | `n8n.conf` | `n8n:5678` | Standalone N8N (local-ai-packaged) |
| `owuid.hcai42.com` | `open-webui.conf` | `open-webui:8080` | Open WebUI |
| `sdbl.hcai42.com` | `supabase-cloud.conf` | `192.168.178.243:8044` | Supabase Kong proxy — ISSUE: Kong is on 8014, not 8044 |
| `3cors-service.hcai42.com` | `3cors-frontend.conf` | `3cors-frontend:8080` | Container `3cors-frontend` is NOT running |
| `3cors-servicel.hcai42.com` | `3cors-frontend.conf` | `3cors-frontend:8080` | Container `3cors-frontend` is NOT running |
| `3cors-serviceagentl.hcai42.com` | `3cors-agent.conf` | `192.168.178.243:8009` | Routes to CorIM backend |
| `auth.corimd.hcai42.com` | `corim-auth.conf` | `corim-auth:9999` | Container `corim-auth` is NOT running |

### Summary: Domain to Final Destination

| Public Domain | Cloudflare → Caddy → Final Destination |
|--------------|---------------------------------------|
| `hack.hcai42.com` | Port 80/443 → Auth Gateway (3100), Kong (8014), Interviewer backend (8000), Interviewer frontend (3002) |
| `supa.hcai42.com` | Port 80/443 → Auth Gateway (3100) → Supabase Studio (3021/3000) + Kong API |
| `3cps.hcai42.com` | Configured in .env as frontend URL for Interviewer — domain not found in active Caddy config |
| `corimd.hcai42.com` | Port 80/443 → `localhost:5176` (N8N frontend — likely misconfigured, should be 5174 for CorIM) |
| `n8n.hcai42.com` | Port 80/443 → `n8n:5678` (standalone N8N workflow engine) |
| `keycloak.hcai42.com` | Port 80/443 → NOT FOUND in any active Caddy config — Keycloak on 8082 has no domain route |
| `owuid.hcai42.com` | Port 80/443 → `open-webui:8080` |
| `sdbl.hcai42.com` | Port 80/443 → `192.168.178.243:8044` (Kong is on 8014 — BROKEN) |
| `admin.hcai42.com` | Configured in .env CORS — no active Caddy route found |
| `3cadmin.hcai42.com` | Configured in .env ADDITIONAL_REDIRECT_URLS — no active Caddy route found |

---

## 5. Service Architecture and Data Flow

### 5.1 Supabase Stack Internal Routing (via Kong)

Kong listens on port 8000 internally (8014 on host) and routes:

| Path Prefix | Kong Service | Upstream |
|-------------|-------------|----------|
| `/auth/v1/verify` | auth-v1-open | `auth:9999` |
| `/auth/v1/callback` | auth-v1-open-callback | `auth:9999` |
| `/auth/v1/authorize` | auth-v1-open-authorize | `auth:9999` |
| `/auth/v1/` | auth-v1 (key-auth) | `auth:9999` |
| `/rest/v1/` | rest-v1 (key-auth) | `rest:3000` |
| `/graphql/v1` | graphql-v1 (key-auth) | `rest:3000/rpc/graphql` |
| `/realtime/v1/` | realtime-v1-ws | `realtime-dev.supabase-realtime:4000` |
| `/storage/v1/` | storage-v1 | `storage:5000` |
| `/functions/v1/` | functions-v1 | `functions:9000` |
| `/analytics/v1/` | analytics-v1 | `analytics:4000` |
| `/pg/` | meta (key-auth admin) | `meta:8080` |
| `/` | dashboard (basic-auth + ip-restrict) | `studio:3000` |

### 5.2 Authentication Flow

```
User Browser
  → Cloudflare Tunnel
    → Caddy (local-ai-packaged)
      → supa.hcai42.com → 3cors-auth-gateway (3100)
        ← OIDC with Keycloak (keycloak.hcai42.com:8082)
          → Supabase Studio (studio:3000 / port 3021)

User Browser  
  → hack.hcai42.com → Caddy
    → /auth/* → Kong (8014) → GoTrue (auth:9999)
      ← Google/GitHub/Keycloak OAuth
    → / → Interviewer frontend (3002)
    → /api/* → Interviewer backend (8000)
```

### 5.3 LLM Request Flow (Secured)

```
Application (interviewer/corim/n8n-backend)
  → 3cors-litellm-auth (4001) [validates JWT + billing reservation]
    → Foundation (8010) [checks cost budget, creates reservation]
    → 3cors-litellm-proxy (4000, internal only) [routes to LLM providers]
      → Azure AI Foundry (EU) — GPT models
      → Azure AI Foundry (EU) — Claude models
      → Anthropic API (direct fallback)
```

### 5.4 Foundation Service Role

`3cors-foundation` (port 8010) is the central cost protection and orchestration hub:
- Validates Supabase JWTs for incoming requests
- Manages per-user spending budgets (Redis-backed velocity limiting)
- Exposes `/v1/billing/health`, `/foundation/v1/webhooks/stripe`
- Stripe webhook receiver at `https://hack.hcai42.com/foundation/v1/webhooks/stripe`
- Manages Docker service lifecycle via mounted docker.sock
- Cost limits: 200 cents/min, €5/day, €20/month (from .env)

### 5.5 Keycloak Identity Broker

Keycloak (`3cors-keycloak`, port 8082) serves the `3cors` realm and acts as:
- OIDC provider for `3cors-auth-gateway` (Supabase Studio access control)
- Identity broker for GoTrue (Supabase auth external provider)
- Client: `supabase-studio` (used by auth-gateway)
- Client: `supabase-gotrue` (used by GoTrue)
- Database: Postgres schema `keycloak` in the Supabase DB

---

## 6. Compose File Relationships and Active Stack

### What is Currently Running

The active stack is assembled from (run from `/home/juergen/3cors-engine/3cors/docker/`):
1. `docker-compose.yml` — base Supabase services
2. `docker-compose.override.yml` — adds Foundation, Redis, Foundation Admin, LiteLLM services, auth-gateway network bridges
3. `docker-compose.auth-gateway.yml` — adds `3cors-auth-gateway`
4. `docker-compose.keycloak.yml` — adds `3cors-keycloak`
5. `docker-compose.interviewer.yml` — adds interviewer frontend/backend
6. `docker-compose.corim.yml` — adds CorIM frontend/backend
7. `docker-compose.n8n.yml` — adds N8N gateway frontend/backend

The `local-ai-packaged` stack runs separately from `/home/juergen/local-ai-packaged/`:
- `docker-compose.yml` (main) + supabase submodule
- `docker-compose.override.private.yml` — adds localhost-bound ports

### Compose Files NOT Currently Active

| File | Purpose | Status |
|------|---------|--------|
| `docker-compose.3cors.yml` | Alternative self-contained stack | Not running |
| `docker-compose.foundation.yml` | Foundation-only stack | Not running |
| `docker-compose.secure.yml` | Secure variant with interviewer | Not running |
| `docker-compose.caddy.yml` | 3CORS-specific Caddy | Not running |
| `docker-compose.integrated.yml` | Foundation integrated into Supabase only | Not running |
| `docker-compose.production.yml` | Production overrides | Not running |
| `docker-compose.monitoring.yml` | Uptime Kuma | Not running |
| `docker-compose.vps.yml` | VPS deployment config | Not running |
| `docker-compose.corim.prod.yml` | CorIM production overlay | Not running |
| `docker-compose.interviewer.prod.yml` | Interviewer production overlay | Not running |
| `docker-compose.n8n.prod.yml` | N8N production overlay | Not running |

---

## 7. Configuration Issues and Mismatches

### Issue 1: Port Mismatch — corimd.hcai42.com
**Location:** `/home/juergen/3cors-engine/3cors/3cors-corim/caddy/Caddyfile` line 16  
**Config:** `corimd.hcai42.com { reverse_proxy localhost:5176 }`  
**Reality:** CorIM frontend runs on `5174`, N8N frontend runs on `5176`  
**Impact:** `corimd.hcai42.com` routes to N8N frontend instead of CorIM frontend  
**Fix:** Change to `reverse_proxy localhost:5174`

### Issue 2: Port Mismatch — sdbl.hcai42.com
**Location:** `/home/juergen/local-ai-packaged/caddy-addon/supabase-cloud.conf` line 38  
**Config:** `reverse_proxy 192.168.178.243:8044`  
**Reality:** Supabase Kong is on host port `8014` (KONG_HTTP_PORT=8014 in .env)  
**Impact:** `sdbl.hcai42.com` returns connection refused  
**Fix:** Change to `reverse_proxy 192.168.178.243:8014`

### Issue 3: Missing Caddy Routes
The following domains appear in `.env` CORS/redirect URLs but have NO active Caddy configuration:
- `keycloak.hcai42.com` — Keycloak runs on 8082 but no domain route
- `admin.hcai42.com` — Foundation admin runs on 3003 but no domain route
- `3cadmin.hcai42.com` — No route
- `3cps.hcai42.com` — Configured as `SITE_URL` / Interviewer frontend URL but no Caddy route

### Issue 4: Dead Caddy Routes
These addon configs reference containers that are NOT running:
- `3cors-frontend.conf`: `3cors-frontend:8080` — no such running container
- `corim-auth.conf`: `corim-auth:9999` — no such running container

### Issue 5: Dual Caddy Potential Conflict
`docker-compose.caddy.yml` defines a `3cors-caddy` container using `network_mode: host` that would conflict with the `caddy` container from `local-ai-packaged` (which also binds ports 80/443). The 3CORS caddy is not running, but starting it would break the system.

### Issue 6: Dual N8N Instances
Two N8N instances run simultaneously:
- `n8n` container (local-ai-packaged) — standalone N8N workflow engine at `n8n.hcai42.com`
- `3cors-n8n-backend` + `3cors-n8n-frontend` — the 3CORS N8N gateway (custom wrapper)
These serve different purposes but the naming can cause confusion.

### Issue 7: SUPABASE_JWT_SECRET vs JWT_SECRET
In `.env`, both `JWT_SECRET` and `SUPABASE_JWT_SECRET` exist — they have the same value. Services use different variable names:
- Supabase stack: `JWT_SECRET`
- 3CORS services: `SUPABASE_JWT_SECRET`  
Currently consistent but a single-variable rename would break auth.

### Issue 8: Keycloak DB Port
`docker-compose.keycloak.yml` uses `KC_DB_URL: jdbc:postgresql://supabase-db:${POSTGRES_PORT:-5434}/keycloak`  
`POSTGRES_PORT=5434` in .env refers to the INTERNAL port (the container listens on this). Correct. But the Supabase DB container maps `5434` internally (PGPORT=5434) and exposes `5433` externally. Keycloak connects via Docker network using internal port 5434 — this is correct.

### Issue 9: litellm-proxy DB Reference
Both `docker-compose.foundation.yml` and `docker-compose.override.yml` reference:  
`DATABASE_URL=postgresql://supabase_admin:${POSTGRES_PASSWORD}@supabase-db:5432/litellm`  
The Supabase DB container is named `supabase-db` and the internal port is 5434 (not 5432). The container name resolves correctly within the network, but the port should be 5434.  
**Impact:** LiteLLM's database connection may fail silently.

---

## 8. Domain-to-Port Quick Reference

| Domain | Expected Route | Actual Host Port | Notes |
|--------|--------------|-----------------|-------|
| `hack.hcai42.com` | Interviewer app | 3002 (frontend), 8000 (api), 8014 (kong/auth) | Active via 3CORS Caddyfile |
| `supa.hcai42.com` | Supabase Studio | 3100 (auth-gateway) → 3021 | Active |
| `3cps.hcai42.com` | Interviewer frontend | — | No Caddy route configured |
| `corimd.hcai42.com` | CorIM frontend | 5174 (should be), 5176 (is, bug) | Misconfigured — points to N8N frontend |
| `n8n.hcai42.com` | Standalone N8N | n8n:5678 | Active via addon |
| `keycloak.hcai42.com` | Keycloak | 8082 | No Caddy route |
| `owuid.hcai42.com` | Open WebUI | open-webui:8080 | Active via addon |
| `sdbl.hcai42.com` | Supabase Kong | 8044 (config), 8014 (actual) | Broken port |
| `admin.hcai42.com` | Foundation admin | 3003 | No Caddy route |
| `3cadmin.hcai42.com` | Foundation admin (alt) | 3003 | No Caddy route |
| `3cors-service.hcai42.com` | 3CORS service | 3cors-frontend:8080 | Container not running |
| `3cors-serviceagentl.hcai42.com` | Agent API | 8009 (CorIM backend) | Active |
| `monitor.hcai42.com` | Uptime Kuma | 3001 | Not running |
| `n4jl.hcai42.com` | Neo4j Browser | — | No Caddy route in active config |

---

## 9. Key File Paths

| Purpose | Path |
|---------|------|
| Main .env (SSOT for all credentials) | `/home/juergen/3cors-engine/3cors/docker/.env` |
| Base Supabase compose | `/home/juergen/3cors-engine/3cors/docker/docker-compose.yml` |
| 3CORS Foundation overlay (auto-loaded) | `/home/juergen/3cors-engine/3cors/docker/docker-compose.override.yml` |
| Auth gateway compose | `/home/juergen/3cors-engine/3cors/docker/docker-compose.auth-gateway.yml` |
| Keycloak compose | `/home/juergen/3cors-engine/3cors/docker/docker-compose.keycloak.yml` |
| Interviewer compose | `/home/juergen/3cors-engine/3cors/docker/docker-compose.interviewer.yml` |
| CorIM compose | `/home/juergen/3cors-engine/3cors/docker/docker-compose.corim.yml` |
| N8N gateway compose | `/home/juergen/3cors-engine/3cors/docker/docker-compose.n8n.yml` |
| Production overrides | `/home/juergen/3cors-engine/3cors/docker/docker-compose.production.yml` |
| Kong API routing | `/home/juergen/3cors-engine/3cors/docker/volumes/api/kong.yml` |
| 3CORS Caddyfile (active for 3CORS domains) | `/home/juergen/3cors-engine/3cors/3cors-corim/caddy/Caddyfile` |
| Caddy main config (local-ai-packaged) | `/home/juergen/local-ai-packaged/Caddyfile` |
| Caddy addon: N8N route | `/home/juergen/local-ai-packaged/caddy-addon/n8n.conf` |
| Caddy addon: Supabase proxy | `/home/juergen/local-ai-packaged/caddy-addon/supabase-cloud.conf` |
| Caddy addon: Open WebUI | `/home/juergen/local-ai-packaged/caddy-addon/open-webui.conf` |
| Caddy addon: CorIM auth | `/home/juergen/local-ai-packaged/caddy-addon/corim-auth.conf` |
| Foundation schema | `/home/juergen/3cors-engine/3cors/docker/volumes/db/foundation-schema.sql` |

---

## 10. Relationship Between Stacks

```
                    INTERNET
                       │
                  Cloudflare CDN/Tunnel
                       │
              ┌────────┴────────┐
              │  Host 192.168.178.243  │
              │  caddy (port 80/443)   │ ← local-ai-packaged stack
              └─────────────────┘
               │          │          │
    ┌──────────┘   ┌──────┘   ┌──────┘
    │              │           │
    ▼              ▼           ▼
supabase_default  local-ai   host:3002,8000,etc
network           network
    │              │
    ├─ supabase    ├─ n8n (standalone)
    ├─ 3cors-auth  ├─ open-webui
    ├─ foundation  └─ 3cors-litellm-auth ← bridges both networks
    ├─ litellm
    ├─ interviewer
    ├─ corim
    └─ n8n-gateway

          3cors-network (172.20.0.0/16)
          ├─ keycloak
          ├─ corim-backend
          └─ n8n-backend
```

The two stacks are coupled via:
1. `3cors-litellm-auth` joining `local-ai-packaged_default` — allows Caddy to reach LiteLLM auth
2. Caddy's `supabase-cloud.conf` using host IP `192.168.178.243:8014` — crosses network boundary via host
3. 3CORS Caddyfile using `localhost:*` — both Caddyfiles run under the same host

