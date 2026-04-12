# Technical Spec: hermes-firman -- Hermes Container Template

> Implements [HERMES_FIRMAN_MVP_PRD.md](HERMES_FIRMAN_MVP_PRD.md).
> For architecture context see [SULTANATE_MVP.md](SULTANATE_MVP.md).

## 1. Artifact Structure

A firman is a directory on the host at a convention path. Vizier resolves
`--firman hermes-firman` to this path.

```
/opt/sultanate/firmans/hermes-firman/
└── firman.yaml
```

No other files are needed for MVP. Everything is configuration on the
upstream image -- hermes-firman does not build a custom Docker image.

## 2. firman.yaml Schema

```yaml
name: hermes-firman
version: "0.1.0"
description: "Hermes Agent container template"
image: "nousresearch/hermes-agent:v2026.3.30"
workspace_dir: "/opt/data/workspace"
hermes_home: "/opt/data"
bootstrap:
  - command: "git clone https://github.com/{{repo_name}}.git {{workspace_dir}}"
    description: "Clone target repository"
  - command: "cd {{workspace_dir}} && git checkout {{branch}}"
    description: "Checkout branch"
startup:
  command: "hermes gateway"
  args: []
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique firman identifier. Used by Vizier to resolve the artifact directory and recorded in Divan province records (`firman` field). |
| `version` | string | yes | Semver. Vizier logs the version at province creation for traceability. |
| `description` | string | yes | Human-readable purpose. Not used at runtime. |
| `image` | string | yes | Docker image reference (registry/repo:tag). Vizier passes this to `docker create`. Pinned to a specific tag -- no `latest`. |
| `workspace_dir` | string | yes | Absolute path inside the container where the target repo is cloned. Vizier uses this to set the terminal `cwd` in berat templates. |
| `hermes_home` | string | yes | Absolute path for Hermes state. Maps to the `HERMES_HOME` env var. The upstream image expects this at `/opt/data`. |
| `bootstrap` | list | yes | Ordered list of commands run via `docker exec` after container start. Each entry has `command` (shell string) and `description` (human-readable log label). Commands support `{{variable}}` substitution (same engine as berat templates). |
| `bootstrap[].command` | string | yes | Shell command. Executed as `docker exec <container> sh -c "<command>"`. Supports template variables. |
| `bootstrap[].description` | string | yes | Log label. Vizier prints this during province creation for operator visibility. |
| `startup` | object | yes | The main process. Vizier runs this via `docker exec -d` (background) after bootstrap completes. |
| `startup.command` | string | yes | Executable name. |
| `startup.args` | list | yes | Arguments array (may be empty). Vizier concatenates: `docker exec -d <container> <command> <args...>`. |

### Template Variables in Bootstrap

Bootstrap commands support the same `{{variable}}` syntax as berat templates.
Variables available at firman level:

| Variable | Source | Required | Default |
|----------|--------|----------|---------|
| `{{repo_name}}` | Sultan's create command | yes | -- |
| `{{branch}}` | Sultan's create command | no | `main` |
| `{{workspace_dir}}` | firman.yaml `workspace_dir` | -- | `/opt/data/workspace` |

Vizier resolves these before executing each bootstrap command. Missing
required variables cause province creation to fail with a clear error.

## 3. How Vizier Uses hermes-firman

Vizier reads `firman.yaml` and executes the province creation sequence.
Each step maps to a specific field in the manifest.

### Step-by-Step Execution

```
1. READ firman.yaml
   -> parse YAML, validate required fields

2. DOCKER CREATE
   -> docker create \
        --name sultanate-prov-<id> \
        --network sultanate-internal \
        --env HERMES_HOME=/opt/data \
        --env HTTP_PROXY=http://<janissary-ip>:8080 \
        --env HTTPS_PROXY=http://<janissary-ip>:8080 \
        --env NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/sultanate-ca.crt \
        --volume prov-<id>-data:/opt/data \
        nousresearch/hermes-agent:v2026.3.30
   Uses: image, hermes_home

3. INSTALL CA CERT (see §4)

4. DOCKER START
   -> docker start sultanate-prov-<id>
   -> entrypoint.sh runs, creates default dirs under /opt/data

5. BOOTSTRAP (ordered)
   -> for each entry in bootstrap:
        docker exec sultanate-prov-<id> sh -c "<rendered command>"
   Uses: bootstrap[].command with variable substitution

6. APPLY BERAT (see HERMES_CODING_BERAT_SPEC.md §5)
   -> write rendered templates into the container volume

7. STARTUP
   -> docker exec -d sultanate-prov-<id> hermes gateway
   Uses: startup.command, startup.args
```

### Container Configuration (Vizier-Provided, Not in firman.yaml)

These are set by Vizier during `docker create`, not declared in the firman:

| Config | Value | Source |
|--------|-------|--------|
| `--network` | `sultanate-internal` | Vizier config (internal-only Docker network) |
| `HTTP_PROXY` | `http://<janissary-ip>:8080` | Vizier config |
| `HTTPS_PROXY` | `http://<janissary-ip>:8080` | Vizier config |
| `HERMES_HOME` | Value from `firman.yaml hermes_home` | firman.yaml |
| `NODE_EXTRA_CA_CERTS` | `/usr/local/share/ca-certificates/sultanate-ca.crt` | Vizier hardcoded |
| `--volume` | `prov-<id>-data:/opt/data` | Vizier-generated per-province volume |

The firman declares _what_ image and _what_ commands. Vizier provides the
networking, proxy, and volume configuration that integrates the container
into the Sultanate network.

## 4. CA Certificate Installation

Provinces route all HTTPS through Janissary, which performs TLS MITM for
credential injection. Containers must trust the Sultanate CA.

### What Vizier Does

```
1. docker cp /opt/sultanate/ca/sultanate-ca.crt \
     sultanate-prov-<id>:/usr/local/share/ca-certificates/sultanate-ca.crt

2. docker exec sultanate-prov-<id> update-ca-certificates

3. NODE_EXTRA_CA_CERTS env var already set at docker create (§3 step 2)
```

### Why Three Steps

| Step | Covers |
|------|--------|
| `docker cp` + `update-ca-certificates` | System-wide trust: Python `requests`, `curl`, `git`, `apt`, Playwright/Chromium, and all programs using the OS trust store. |
| `NODE_EXTRA_CA_CERTS` env var | Node.js processes (npm, npx, Node scripts) which use their own bundled CA store and ignore the OS store unless this env var is set. |

The CA cert file lives on the host at `/opt/sultanate/ca/sultanate-ca.crt`.
It is generated once during Sultanate deployment and shared across all
provinces. The private key (`sultanate-ca.key`) is only readable by
Janissary's process -- never copied into containers.

### Timing

CA cert installation happens after `docker create` but before `docker start`.
This ensures the trust store is ready before the entrypoint or any bootstrap
commands make HTTPS requests through Janissary.

## 5. Upstream Hermes Agent Image

hermes-firman uses `nousresearch/hermes-agent:v2026.3.30` without
modification. Understanding what the image provides is essential for correct
firman and berat design.

### What the Image Provides

| Component | Detail |
|-----------|--------|
| **Base OS** | Debian 13.4 (Trixie) |
| **Python** | System Python 3.x with pip |
| **Node.js** | LTS via NodeSource |
| **Tools** | ripgrep, git, curl, Playwright + Chromium |
| **User** | Non-root `hermes` user. Entrypoint uses `gosu` to drop privileges from root. |
| **Entrypoint** | `/opt/hermes/docker/entrypoint.sh` -- runs as root, creates default directories under `$HERMES_HOME`, then drops to `hermes` user via `gosu`. |
| **HERMES_HOME** | `/opt/data` (Docker VOLUME). All Hermes state lives here: `config.yaml`, `SOUL.md`, sessions, memories, skills, workspace. |
| **Gateway mode** | `hermes gateway` starts Telegram polling + webhook listener. |
| **Profiles** | Hermes supports profile-based isolation, but Sultanate uses one profile per container (the default). |

### Volume Mount at /opt/data

The image declares `/opt/data` as a VOLUME. Vizier creates a named Docker
volume per province (`prov-<id>-data`) and mounts it here. This volume
persists across container restarts.

Contents after entrypoint runs (defaults created by entrypoint.sh):

```
/opt/data/
├── config.yaml      (default Hermes config -- overwritten by berat)
├── SOUL.md          (default soul -- overwritten by berat)
├── sessions/        (Hermes session storage)
├── memories/        (persistent agent memory)
├── skills/          (learned skills)
└── workspace/       (created by bootstrap repo clone)
```

### Entrypoint Behavior

The entrypoint (`/opt/hermes/docker/entrypoint.sh`):

1. Runs as root (container starts as root)
2. Creates default directories under `$HERMES_HOME` if they don't exist
3. Generates default `config.yaml` and `SOUL.md` if not present
4. Drops privileges to `hermes` user via `gosu`
5. Executes the CMD (or falls through if no CMD)

**Critical ordering**: Berat files (SOUL.md, config.yaml, AGENTS.md) must
be written _after_ the entrypoint runs, otherwise the entrypoint may
overwrite them with defaults. Vizier's sequence (§3) handles this: berat
application (step 6) happens after `docker start` (step 4), which triggers
the entrypoint.

### How hermes-firman Leverages This

| Upstream Feature | hermes-firman Usage |
|-----------------|---------------------|
| `/opt/data` volume | Vizier mounts a per-province named volume here. All state persists. |
| Entrypoint creates defaults | Vizier lets the entrypoint run first, then overwrites with berat files. |
| `hermes` user + gosu | Bootstrap and berat commands run via `docker exec` (as root by default in exec). Files written to `/opt/data` are accessible to the `hermes` user because the volume is owned by `hermes`. |
| `hermes gateway` | Used as the startup command. Hermes reads config.yaml and SOUL.md from `$HERMES_HOME`, AGENTS.md from `workspace_dir`. |
| Playwright/Chromium | Available for the `browser` tool if enabled in berat. No additional installation needed. |
| Python + Node.js | Available for `code_execution` tool and for agent workflows. |

## 6. Image Versioning

hermes-firman pins `nousresearch/hermes-agent:v2026.3.30`. This tag is
stored in `firman.yaml` and used by Vizier for all province creations.

- **Upgrades**: Update the `image` field in firman.yaml. Existing provinces
  continue on the old image. New provinces use the new image.
- **Override**: Sultan can pass `--image <tag>` to Vizier to override the
  firman's pinned tag for a specific province (not in MVP, but the schema
  supports it).
- **No `latest`**: Always pin a specific tag for reproducibility.
