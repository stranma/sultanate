# Technical Spec: openclaw-firman -- OpenClaw Container Template

> Implements [OPENCLAW_FIRMAN_MVP_PRD.md](OPENCLAW_FIRMAN_MVP_PRD.md).
> For architecture context see [SULTANATE_MVP.md](SULTANATE_MVP.md).
> For Vizier's province creation flow see [VIZIER_SPEC.md](VIZIER_SPEC.md) §5.

## 1. Artifact Structure

A firman is a directory on the host at a convention path. Vizier resolves
`--firman openclaw-firman` to this path.

```
/opt/sultanate/firmans/openclaw-firman/
└── firman.yaml
```

No other files are needed for MVP. Everything is configuration on the
upstream image -- openclaw-firman does not build a custom Docker image.

## 2. firman.yaml Schema

```yaml
name: openclaw-firman
version: "0.1.0"
description: "OpenClaw agent container template"
image: "openclaw/openclaw:v2026.4.15"
workspace_dir: "/opt/data/workspace"
openclaw_home: "/opt/data"
bootstrap:
  - command: "update-ca-certificates"
    description: "Trust Sultanate CA in the system trust store"
  - command: "git clone https://github.com/{{repo_name}}.git {{workspace_dir}}"
    description: "Clone target repository"
  - command: "cd {{workspace_dir}} && git checkout {{branch}}"
    description: "Checkout branch"
startup:
  command: "openclaw"
  args: [ "gateway", "--port", "18789" ]
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique firman identifier. Used by Vizier to resolve the artifact directory and recorded in Divan province records (`firman` field). |
| `version` | string | yes | Semver. Vizier logs the version at province creation for traceability. |
| `description` | string | yes | Human-readable purpose. Not used at runtime. |
| `image` | string | yes | Docker image reference (registry/repo:tag). Vizier passes this to `docker create`. Pinned to a specific tag -- no `latest`. |
| `workspace_dir` | string | yes | Absolute path inside the container where the target repo is cloned. Vizier uses this both in bootstrap commands (`{{workspace_dir}}`) and when selecting the workspace for OpenClaw's auto-loaded files. |
| `openclaw_home` | string | yes | Absolute path for OpenClaw state. Maps to the `OPENCLAW_HOME` env var. OpenClaw looks for its config at `$OPENCLAW_HOME/.openclaw/openclaw.json`. |
| `bootstrap` | list | yes | Ordered list of commands run via `docker exec` after container start. Each entry has `command` (shell string) and `description` (human-readable log label). Commands support `{{variable}}` substitution (same engine as berat templates). |
| `bootstrap[].command` | string | yes | Shell command. Executed as `docker exec <container> sh -c "<command>"`. Supports template variables. |
| `bootstrap[].description` | string | yes | Log label. Vizier prints this during province creation for operator visibility. |
| `startup` | object | yes | The main process. Vizier runs this via `docker exec -d` (background) after bootstrap completes. |
| `startup.command` | string | yes | Executable name. |
| `startup.args` | list | yes | Arguments array. For openclaw-firman, always starts with `gateway --port 18789`. |

### Template Variables in Bootstrap

Bootstrap commands support the same `{{variable}}` syntax as berat
templates. Variables available at firman level:

| Variable | Source | Required | Default |
|----------|--------|----------|---------|
| `{{repo_name}}` | Sultan's create command | yes | -- |
| `{{branch}}` | Sultan's create command | no | `main` |
| `{{workspace_dir}}` | firman.yaml `workspace_dir` | -- | `/opt/data/workspace` |

Vizier resolves these before executing each bootstrap command. Missing
required variables cause province creation to fail with a clear error.

## 3. How Vizier Uses openclaw-firman

Vizier reads `firman.yaml` and executes the province creation sequence.
Each step maps to a specific field in the manifest.

### Step-by-Step Execution

```
1. READ firman.yaml
   -> parse YAML, validate required fields

2. CREATE WG-CLIENT SIDECAR
   -> docker create \
        --name wg-client-prov-<id> \
        --cap-add NET_ADMIN \
        -v /opt/sultanate/provinces/<id>/wg0.conf:/etc/wireguard/wg0.conf:ro \
        -e MITMPROXY_HOST=10.13.13.1 \
        sultanate/wg-client:latest

3. DOCKER CREATE (province, shares sidecar network)
   -> docker create \
        --name sultanate-prov-<id> \
        --network-mode container:wg-client-prov-<id> \
        --env OPENCLAW_HOME=/opt/data \
        --env NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/sultanate-ca.crt \
        --env JANISSARY_API=http://10.13.13.1:8081 \
        --volume /opt/sultanate/provinces/<id>/data:/opt/data \
        openclaw/openclaw:v2026.4.15
   Uses: image, openclaw_home

4. INSTALL CA CERT (see §4)

5. DOCKER START
   -> docker start wg-client-prov-<id>  (sidecar first)
   -> docker start sultanate-prov-<id>
   -> entrypoint runs (drops to non-root user, ready to exec)

6. BOOTSTRAP (ordered)
   -> for each entry in bootstrap:
        docker exec sultanate-prov-<id> sh -c "<rendered command>"
   Uses: bootstrap[].command with variable substitution

7. APPLY BERAT (see OPENCLAW_CODING_BERAT_SPEC.md §5)
   -> write rendered templates into the container volume:
      /opt/data/workspace/SOUL.md
      /opt/data/workspace/AGENTS.md
      /opt/data/workspace/IDENTITY.md        (optional)
      /opt/data/.openclaw/openclaw.json

8. STARTUP
   -> docker exec -d sultanate-prov-<id> \
        bash -c "OPENCLAW_HOME=/opt/data openclaw gateway --port 18789"
   Uses: startup.command, startup.args

9. HEALTH CHECK
   -> wait for OpenClaw's gateway HTTP endpoint on port 18789 to respond.
      Vizier polls `docker exec sultanate-prov-<id> curl -sf
      http://127.0.0.1:18789/health` until 200 or 30 s timeout.
```

### Container Configuration (Vizier-Provided, Not in firman.yaml)

These are set by Vizier during `docker create`, not declared in the
firman:

| Config | Value | Source |
|--------|-------|--------|
| `--network-mode` | `container:wg-client-prov-<id>` | Shares wg-client sidecar's network (WireGuard transparent proxy) |
| `OPENCLAW_HOME` | Value from `firman.yaml openclaw_home` | firman.yaml |
| `NODE_EXTRA_CA_CERTS` | `/usr/local/share/ca-certificates/sultanate-ca.crt` | Vizier hardcoded |
| `JANISSARY_API` | `http://10.13.13.1:8081` | Vizier hardcoded |
| `--volume` | `/opt/sultanate/provinces/<id>/data:/opt/data` | Host-mounted per-province data directory |

The firman declares _what_ image and _what_ commands. Vizier provides the
networking and volume configuration that integrates the container into
the Sultanate network. Provinces access the internet via WireGuard
transparent proxy (all traffic automatically routed through Janissary).

## 4. CA Certificate Installation

Provinces route all HTTPS through Janissary, which performs TLS MITM for
credential injection. Containers must trust the Sultanate CA.

### What Vizier Does

```
1. docker cp /opt/sultanate/certs/sultanate-ca.pem \
     sultanate-prov-<id>:/usr/local/share/ca-certificates/sultanate-ca.crt

2. docker exec sultanate-prov-<id> update-ca-certificates
   (also listed in firman bootstrap, idempotent)

3. NODE_EXTRA_CA_CERTS env var already set at docker create (§3 step 3)
```

### Why Three Steps

| Step | Covers |
|------|--------|
| `docker cp` + `update-ca-certificates` | System-wide trust: Python `requests`, `curl`, `git`, `apt`, and all programs using the OS trust store. |
| `NODE_EXTRA_CA_CERTS` env var | Node.js processes (including OpenClaw's own Node.js runtime) which use their own bundled CA store and ignore the OS store unless this env var is set. |

The CA cert file lives on the host at
`/opt/sultanate/certs/sultanate-ca.pem`. It is generated once during
Sultanate deployment and shared across all provinces. The private key
(`sultanate-ca.key`) is only readable by Janissary's process -- never
copied into containers.

### Timing

CA cert installation happens after `docker create` but before the first
bootstrap command that reaches the network (e.g., `git clone`). Vizier's
sequence (§3 step 4) handles this placement.

## 5. Upstream OpenClaw Image

openclaw-firman uses `openclaw/openclaw:v2026.4.15` without modification.
Understanding what the image provides is essential for correct firman
and berat design.

### What the Image Provides

| Component | Detail |
|-----------|--------|
| **Base OS** | Debian (slim variant) |
| **Node.js** | 22+ (OpenClaw runtime) |
| **Python** | System Python 3.x |
| **Tools** | git, curl, ripgrep |
| **User** | Non-root (gosu-based privilege drop in entrypoint) |
| **Entrypoint** | `/opt/openclaw/docker/entrypoint.sh` (or equivalent per upstream) -- starts as root, creates default directories under `$OPENCLAW_HOME`, drops to non-root user. |
| **OPENCLAW_HOME** | `/opt/data` by default (Docker VOLUME). Config lives at `$OPENCLAW_HOME/.openclaw/openclaw.json`. |
| **Gateway mode** | `openclaw gateway --port 18789` starts the long-running daemon that services configured channels (Telegram, Slack, etc.) and exposes the HTTP API on port 18789. |
| **Auto-loaded workspace files** | At first session turn, OpenClaw loads `SOUL.md`, `AGENTS.md`, `IDENTITY.md`, `USER.md`, `TOOLS.md`, and `BOOTSTRAP.md` from the workspace root (as specified by `agents.defaults.workspace` in `openclaw.json`). |
| **Multi-provider models** | Configured via `agent.model` as `"provider/model-id"` (e.g. `"anthropic/claude-sonnet-4"`). API keys read from environment or embedded config. |
| **MCP support** | `mcp_servers` section in openclaw.json registers external MCP servers; OpenClaw invokes their tools alongside built-ins. |

### Volume Mount at /opt/data

The image declares `/opt/data` as a VOLUME. Vizier creates a host
directory per province
(`/opt/sultanate/provinces/<id>/data`) and bind-mounts it here. This
directory persists across container restarts and is accessible from the
host for backup and inspection.

Contents after entrypoint + berat application:

```
/opt/data/
├── .openclaw/
│   └── openclaw.json            (agent config, written by Vizier from berat)
├── workspace/
│   ├── SOUL.md                  (agent persona, written by Vizier from berat)
│   ├── AGENTS.md                (operating instructions, written by Vizier)
│   ├── IDENTITY.md              (agent identity, optional, from berat)
│   └── <repo contents>          (cloned by bootstrap step)
├── agents/                      (OpenClaw session transcripts, per-session)
│   └── <agent-id>/sessions/<session-id>.jsonl
├── skills/                      (shared skills directory; empty in MVP)
└── logs/                        (OpenClaw process logs)
```

### Entrypoint Behavior

The entrypoint:

1. Runs as root
2. Creates default directories under `$OPENCLAW_HOME` if they don't exist
3. Ensures permissions on the `/opt/data` bind-mount are right for the
   non-root user
4. Drops privileges to the `openclaw` (or similar) non-root user via
   `gosu`
5. Executes the CMD or (more commonly) exits, leaving the container
   running in an idle state until Vizier issues `docker exec -d openclaw
   gateway --port 18789`

**Critical ordering**: Berat files (SOUL.md, AGENTS.md, openclaw.json)
must be written _after_ the entrypoint runs and _before_ `openclaw
gateway` starts. Vizier's sequence handles this: berat application
(step 7) happens after container start (step 5), and openclaw gateway
is started last (step 8).

### How openclaw-firman Leverages This

| Upstream Feature | openclaw-firman Usage |
|-----------------|-----------------------|
| `/opt/data` volume | Vizier bind-mounts a per-province host directory here. All state persists and is accessible from the host. |
| Entrypoint creates defaults | Vizier lets the entrypoint run first, then overwrites with berat files. |
| Non-root user + gosu | Bootstrap and berat commands run via `docker exec` (as root by default). Files written to `/opt/data/.openclaw` and `/opt/data/workspace` must be chowned to the non-root user so OpenClaw can read them. Vizier wraps each write with a `chown` in the same exec. |
| `openclaw gateway` | Used as the startup command. OpenClaw reads `openclaw.json` from `$OPENCLAW_HOME/.openclaw/`, SOUL.md / AGENTS.md / IDENTITY.md from the configured workspace. |
| MCP registry | Used by the berat to register the Janissary security MCP server for appeal/access-request tools. |
| Channels config | Telegram channel configured via `channels.telegram.botTokenEnv` in openclaw.json; Vizier provisions a bot token per province. |
| Sandbox mode | Set to `off` in the MVP berat; Sultanate's outer container (WireGuard + kill-switch) is the isolation boundary. Nested OpenClaw sandbox (Docker-in-Docker) is a Phase 2 option. |

## 6. Image Versioning

openclaw-firman pins `openclaw/openclaw:v2026.4.15`. This tag is stored
in `firman.yaml` and used by Vizier for all province creations.

- **Upgrades**: Update the `image` field in firman.yaml and pin by
  digest (SHA256). Existing provinces continue on the old image. New
  provinces use the new image.
- **Override**: Sultan can pass `--image <tag>` to Vizier to override
  the firman's pinned tag for a specific province (not in MVP, but the
  schema supports it).
- **No `latest`**: Always pin a specific tag for reproducibility.
- **Renovate / dependabot**: A deploy-side automation opens a PR when
  a new `openclaw/openclaw` tag appears; operator reviews and updates
  the firman manually.
