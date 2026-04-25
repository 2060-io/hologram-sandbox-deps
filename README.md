# Hologram Sandbox Agent — Dependencies

Shared infrastructure for the [Hologram Sandbox Agent Example](https://github.com/2060-io/hologram-sandbox-agent-example). This repo deploys two Verifiable Services:

- **Organization** — `Hologram Demo SA`, the trust anchor. Obtains the ECS Organization + Service credentials from Verana testnet, creates its own Trust Registry, registers an Avatar credential schema, and issues Service credentials to child agents.
- **Avatar Issuer** — a DIDComm chatbot that issues an **Avatar credential** (AnonCreds) to end users so they can authenticate against the example agent.

These two services are what a documentation reader depends on to try out [`hologram-sandbox-agent-example`](https://github.com/2060-io/hologram-sandbox-agent-example) without having to stand up their own Trust Registry first.

```
┌─────────────────────────┐
│   Hologram Demo SA      │   organization.sandbox.hologram.zone
│   (Organization VS)     │
│   - ECS credentials     │
│   - Trust Registry      │
│   - Avatar schema       │
└──────────┬──────────────┘
           │ Service credential
           ▼
┌─────────────────────────┐
│   Demo Avatar Issuer    │   avatar.sandbox.hologram.zone
│   (Avatar VS)           │
│   - AnonCreds credDef   │
│   - Issuer chatbot      │
└─────────────────────────┘
           │ Avatar credential (issued to users)
           ▼
┌─────────────────────────┐
│   hologram-sandbox-    │   example-agent.sandbox.hologram.zone
│     agent-example       │
└─────────────────────────┘
```

## Services

| Service | Role | Public URL | Chart |
|---|---|---|---|
| `organization` | Trust anchor | `organization.sandbox.hologram.zone` | `vs-agent-chart` |
| `avatar` | Avatar credential issuer (chatbot) | `avatar.sandbox.hologram.zone` | `vs-agent-chart` |

## Repository layout

```
hologram-sandbox-deps/
├── organization/
│   ├── config.env          # Hologram Demo SA identity + TR configuration
│   ├── deployment.yaml     # Helm values for the vs-agent-chart
│   └── avatar-schema.json  # Hologram Demo Avatar Credential JSON schema
├── avatar/
│   ├── config.env          # Child service identity, chatbot port, MinIO config
│   └── deployment.yaml     # Helm values for the vs-agent-chart + chatbot
├── common/
│   └── common.sh           # Shared shell helpers (veranad account, schema discovery, credential flow)
└── .github/workflows/
    ├── 1_deploy-organization.yml   # Deploy org, get ECS creds, create Trust Registry
    └── 2_deploy-avatar.yml         # Deploy avatar issuer, get Service cred, deploy chatbot
```

## GitHub Actions workflows

Run them **in order** from the Actions tab (manual dispatch on the `main` branch):

| # | Workflow | Steps |
|---|---|---|
| 1 | Deploy Organization | `deploy` · `get-ecs-credentials` · `create-trust-registry` · `all` |
| 2 | Deploy Avatar | `deploy` · `get-credentials` · `deploy-chatbot` · `all` |

On first setup, run each one with step `all`.

### Required GitHub Secrets

| Secret | Used by | Description |
|---|---|---|
| `OVH_KUBECONFIG` | both | Kubeconfig for the target K8s cluster |
| `K8S_NAMESPACE` | both | Target namespace (e.g. `hologram-demo`) |
| `ORG_WALLET_KEY` | 1 | VS Agent wallet encryption key (`openssl rand -base64 32`) |
| `ORG_VSAGENT_DB_PASSWORD` | 1 | Organization VS Agent internal DB password |
| `AVATAR_WALLET_KEY` | 2 | VS Agent wallet encryption key for avatar |
| `AVATAR_VSAGENT_DB_PASSWORD` | 2 | Avatar VS Agent internal DB password |
| `CHATBOT_DB_PASSWORD` | 2 | PostgreSQL password for the issuer chatbot |
| `MINIO_ACCESS_KEY` / `MINIO_SECRET_KEY` | 2 | MinIO credentials for avatar previews |

## Local development

### Prerequisites

- Docker and Docker Compose
- [ngrok](https://ngrok.com/) (authenticated)
- `curl`, `jq`

### 1. Start the organization locally

```bash
set -a
source organization/config.env
set +a
./common/common.sh   # sources helper functions (or rely on scripts/setup.sh if you add one)
```

The GHA workflows contain the full deploy + credential logic. For local iteration, read `common/common.sh` for the helper functions you can use directly (`setup_veranad_account`, `discover_ecs_vtjsc`, `issue_remote_and_link`, etc.).

### 2. Start the avatar issuer

Point the avatar at the running organization and use the same helpers to obtain its Service credential + create the AnonCreds credential definition.

> **Note:** Only one ngrok tunnel runs at a time on the free plan. For multi-service local testing, deploy the organization to the cluster first and point the avatar's `ORG_VS_PUBLIC_URL` / `ORG_VS_ADMIN_URL` at the deployed instance.

## End-user flow

Once both services are deployed on `sandbox.hologram.zone`, an end user:

1. Opens the [Hologram Messaging](https://hologram.zone) app on their phone.
2. Navigates to `https://avatar.sandbox.hologram.zone/` (or scans its QR code) to obtain a **Hologram Demo Avatar credential**.
3. Connects to `example-agent.sandbox.hologram.zone` (from [`hologram-sandbox-agent-example`](https://github.com/2060-io/hologram-sandbox-agent-example)) and authenticates with that credential.

## Related repos

- **[`hologram-sandbox-agent-example`](https://github.com/2060-io/hologram-sandbox-agent-example)** — the starter agent that consumes these services. Fork it to build your own.
- **[`hologram-generic-ai-agent-vs`](https://github.com/2060-io/hologram-generic-ai-agent-vs)** — the underlying AI agent container used by the example.
- **[`vs-agent`](https://github.com/2060-io/vs-agent)** — the DIDComm / credential primitives.

## License

Apache-2.0
