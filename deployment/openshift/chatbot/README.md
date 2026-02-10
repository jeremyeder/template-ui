# Amber Chatbot - OpenShift Deployment

Interactive chat UI for the Amber codebase intelligence agent, deployed to OpenShift.

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  chatbot namespace                                      │
│                                                         │
│  ┌──────────────┐    ┌───────────────┐    ┌──────────┐  │
│  │ template-ui  │───▶│ amber-agent   │───▶│ postgres │  │
│  │ (Route:8080) │    │ (ClusterIP)   │    │(ClusterIP)│  │
│  └──────────────┘    └───────────────┘    └──────────┘  │
│   Edge TLS Route      :8000 internal      :5432         │
│   (public)            /v1/stream          checkpoints   │
│                       /v1/history                       │
│                       /v1/threads                       │
└─────────────────────────────────────────────────────────┘
```

- **template-ui** — The only public-facing component. Proxies `/v1/stream`, `/v1/history`, `/v1/threads` to the agent server-side, keeping the agent internal.
- **amber-agent** — LangGraph-based agent with streaming chat, tool execution, and conversation persistence via PostgreSQL checkpointing.
- **postgresql** — Stores LangGraph checkpoints for multi-turn conversation persistence.

## Prerequisites

- OpenShift cluster with `oc` CLI authenticated
- GCP service account JSON key with access to Vertex AI Model Garden (Anthropic models)
- GitHub personal access token for Amber's GitHub tools

## Deploy

### 1. Generate secrets

All secret values in the manifests are placeholders (`REPLACE_ME`). Generate and set them before deploying.

```bash
# Generate a random PostgreSQL password
PG_PASS=$(openssl rand -base64 18)

# Create the namespace
oc create namespace chatbot

# PostgreSQL credentials
oc create secret generic postgresql \
  --from-literal=POSTGRESQL_USER=amber \
  --from-literal=POSTGRESQL_PASSWORD="$PG_PASS" \
  --from-literal=POSTGRESQL_DATABASE=amber \
  -n chatbot

# Amber agent secrets
oc create secret generic amber-agent-secrets \
  --from-literal=GITHUB_TOKEN="ghp_your_token_here" \
  --from-literal=POSTGRES_URL="postgresql://amber:${PG_PASS}@postgresql:5432/amber" \
  -n chatbot

# GCP service account key (for Vertex AI Model Garden)
oc create secret generic gcp-sa-key \
  --from-file=sa-key.json=/path/to/your/gcp-sa-key.json \
  -n chatbot
```

### 2. Update configuration

Edit `amber-agent-configmap.yaml`:
- Set `GCP_PROJECT_ID` to your GCP project ID
- Set `GCP_REGION` to the region where Claude is available (default: `us-east5`)

### 3. Apply with Kustomize

Since you created the secrets via `oc create` above, skip the placeholder secret manifests:

```bash
# Remove placeholder secrets from kustomization.yaml before applying,
# or just apply and let the existing secrets take precedence
```

```bash
oc apply -k deployment/openshift/chatbot/
```

### 4. Build the template-ui image

```bash
oc start-build template-ui -n chatbot
```

### 5. Verify

```bash
# Check pods
oc get pods -n chatbot

# Check route
oc get route template-ui -n chatbot
```

Open the route URL in a browser to access the chat UI.

## Configuration

### Amber Agent

| Variable | Source | Description |
|----------|--------|-------------|
| `GCP_PROJECT_ID` | ConfigMap | GCP project for Vertex AI |
| `GCP_REGION` | ConfigMap | Vertex AI region (default: `us-east5`) |
| `LLM_MODEL` | ConfigMap | Anthropic model name |
| `LOG_LEVEL` | ConfigMap | Python log level |
| `GITHUB_TOKEN` | Secret | GitHub API token for agent tools |
| `POSTGRES_URL` | Secret | PostgreSQL connection string |
| `GOOGLE_APPLICATION_CREDENTIALS` | Env | Path to mounted GCP SA key (auto-set) |

### Template UI

| Variable | Source | Description |
|----------|--------|-------------|
| `AGENT_HOST` | ConfigMap | Internal agent URL (`http://amber-agent:8000`) |
| `AUTH_ENABLED` | ConfigMap | SSO authentication (`false` for now) |

## API Endpoints

All endpoints are proxied through template-ui. The agent is internal-only (ClusterIP).

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/v1/stream` | Stream a chat response (SSE) |
| `GET` | `/v1/history/:threadId` | Get conversation history |
| `GET` | `/v1/threads/:userId` | List thread IDs for a user |
| `GET` | `/health` | Health check |

## Troubleshooting

```bash
# Agent logs
oc logs -f deployment/amber-agent -n chatbot

# UI logs
oc logs -f deployment/template-ui -n chatbot

# PostgreSQL logs
oc logs -f deployment/postgresql -n chatbot

# Test agent directly (from within cluster)
oc exec deployment/template-ui -n chatbot -- \
  curl -s http://amber-agent:8000/health
```
