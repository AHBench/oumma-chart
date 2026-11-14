# Oumma Helm chart

Deploys the Oumma agentic workflow platform:

- **engine** — the Go workflow engine (HUMAN + SYSTEM + tool steps, LLM
  routing, Keycloak-authenticated API). Exposed **only in-cluster**.
- **web** — the React workspace. The **only public surface**: its nginx
  reverse-proxies every engine API path (`/api`, `/processes`, `/oper-api`,
  `/dms-api`, `/data-api`) to the engine service, so the browser talks to a
  single origin and the backend is never directly reachable.

Identity is delegated to an **existing Keycloak** (realm + two clients),
persistence to an **existing PostgreSQL**. The engine auto-creates its
tables at startup.

## Prerequisites

| Dependency | What to prepare |
|---|---|
| Keycloak | realm, a password-grant client (direct access grants on) + a confidential admin client with service account |
| PostgreSQL | a database the engine can connect to (any reachable instance) |
| LLM | any OpenAI-compatible endpoint (OpenAI, Groq, vLLM, Ollama, ...) |
| Ingress | nginx ingress class (+ TLS of your choice; cert-manager supported) |
| Process definitions | a ConfigMap of `<process_id>.yaml` files (optional; engine also loads designs stored in the database) |

## Install

```bash
helm install oumma ./oumma-chart -f my-values.yaml -n oumma --create-namespace
```

Minimal required values:

```yaml
database:
  host: <pg-host>
  port: 5432
  username: <user>
  password: <password>
  database: <db>

keycloak:
  url: http://<keycloak-host>:8080   # reachable from the engine pods
  realm: oumma
  clientId: oumma-app
  adminClientId: oumma-admin
  adminClientSecret: <secret>

llm:
  baseUrl: https://<llm-host>/v1
  model: <model>
  apiKey: <key>

ingress:
  host: oumma.example.com
  tls:
    secretName: oumma-example-com-tls
    hosts: [oumma.example.com]
```

## Keycloak setup (once, per realm)

1. **Realm** `oumma` (name is a value, not fixed).
2. **Client `oumma-app`** — the login client. *Client authentication* off,
   *Authentication* → **Direct access grants: on**.
3. **Client `oumma-admin`** — *Client authentication: on*, **Service account
   roles: realm-admin** (or a scoped admin role). Copy its client secret into
   `keycloak.adminClientSecret`.
4. **Groups** your processes reference (e.g. `Operateur`, `cashier`,
   `management`), users in them, and a **`department` user attribute**
   (comma-separated codes) where your processes gate on departments.

## Notes

- The web app runs in *proxy mode* (`OUMMA_ENGINE_UPSTREAM` is set by the
  chart); all `*_BASE_URL` app settings are intentionally empty (same origin).
- Engine process definitions: create a ConfigMap and point
  `engine.processesConfigMap` at it:
  ```bash
  kubectl create configmap oumma-processes \
    --from-file=order-processing.yaml=./workflows/order-processing.yaml
  ```
- Keep `tools.dryRun: true` until you deliberately wire real
  WhatsApp/Telegram/email credentials.
