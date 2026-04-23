# PRD: hermes-firman v1 -- Hermes Container Template

> For shared glossary, deployment model, and component overview see
> [SULTANATE.md](SULTANATE.md).

## Vision

hermes-firman is the default container template (firman) for Sultanate. It
defines the infrastructure needed to run a Hermes-based province (isolated
container): Docker image, workspace bootstrap, and runtime startup.

A firman is the office. Who works there -- personality, tools, permissions --
is defined by the berat (agent profile) (see SULTANATE.md).

## What hermes-firman Defines

1. **Docker image** -- OS, Hermes installation, development tools
2. **Workspace bootstrap** -- repo clone, directory structure
3. **Runtime startup** -- how to launch Hermes inside the container
4. **Networking** -- proxy configuration pointing to Janissary

## Province Bootstrap Sequence

When Vizier (deployment orchestrator) creates a province from hermes-firman + a berat:

```text
1. Vizier creates container from hermes-firman Docker image
   --> internal Docker network only (no external route)
   --> HTTP_PROXY / HTTPS_PROXY pointing to Janissary

2. Vizier runs workspace bootstrap inside the container:
   --> clones target repo (via Janissary proxy, credentials injected)

3. Vizier applies berat inside the container:
   --> writes AGENTS.md to workspace root (from berat instructions template)
   --> writes ~/.hermes/config.yaml (from berat tool/model selection)
   --> writes ~/.hermes/SOUL.md (from berat soul)
   --> configures security policy in Divan (from berat whitelist/grants)

4. Vizier starts `hermes gateway` inside the container
   --> Hermes loads AGENTS.md from workspace
   --> Hermes connects to Telegram via bot token
   --> Pasha (agent inside province) is now reachable by Sultan

5. Vizier writes province state to Divan (shared state store)
   --> Aga (security advisor) reads new province, provisions grants per berat
   --> Vizier updates status to running
```

## Docker Image

The hermes-firman Docker image includes:

- **Base**: Ubuntu LTS or similar
- **Hermes**: pre-installed with CLI and gateway
- **Dev tools**: git, Python, Node.js, common build tools
- **No secrets**: no API keys, tokens, or credentials baked into the image

The image is generic. All province-specific configuration comes from the
berat, applied by Vizier during bootstrap.

## Workspace Bootstrap

The firman handles repo cloning and workspace setup. Parameters provided
by Sultan at province creation time:

- **repo** -- GitHub repository URL (e.g., `github.com/stranma/EFM`)
- **branch** -- branch to check out (default: `main`)
- **workspace_path** -- path inside container (default: `/workspace`)

Vizier clones the repo through Janissary (egress proxy, credentials injected
transparently via grant table). The clone happens before Hermes starts -- Hermes finds a
ready workspace.

## Hermes Runtime Startup

After workspace bootstrap and berat application, Vizier starts Hermes:

1. `hermes gateway` launched as the container's main process
2. Hermes reads `~/.hermes/config.yaml` (written by berat application)
3. Hermes loads `AGENTS.md` from workspace root (written by berat application)
4. Hermes connects to Telegram (bot token from berat/Aga provisioning)

The firman defines HOW Hermes starts. WHAT it's configured with comes from
the berat.

## Telegram Configuration

Each province gets its own Telegram bot for Sultan (human operator)
communication:

- Bot token provisioned by Aga from a pool or created on demand
- `TELEGRAM_ALLOWED_USERS` set to Sultan's Telegram user ID
- Hermes gateway started with Telegram channel enabled
- Sultan sees the province as a separate Telegram thread

Phase 2 will add shared Telegram channels for multi-agent coordination
(see SULTANATE.md Communication Model).

## Province Parameters

When Sultan creates a province, these parameters are firman-level
(infrastructure). Berat-level parameters (tools, whitelist, model) are
defined in the berat PRD.

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `repo` | yes | -- | GitHub repo to clone |
| `branch` | no | `main` | Branch to check out |
| `name` | no | auto-generated | Province display name |

## Phase 1 Scope

**In scope:**
- Docker image with Hermes, git, Python, Node.js
- Workspace bootstrap (repo clone into /workspace)
- Hermes gateway startup with Telegram
- HTTP_PROXY/HTTPS_PROXY configuration pointing to Janissary
- Province parameters (repo, branch, name)

**Deferred:**
- Multiple firman variants (openhands-firman, crewai-firman)
- Firman versioning and migration
- Multi-stage bootstrap (clone + additional setup scripts)
- Custom base images per province
- Post-task cleanup and artifact collection
