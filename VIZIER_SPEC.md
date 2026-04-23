# Vizier Technical Specification

> Province orchestrator CLI + OpenClaw agent wrapper for Sultanate.
> See [VIZIER_MVP_PRD.md](VIZIER_MVP_PRD.md) for product requirements,
> [SULTANATE_MVP.md](SULTANATE_MVP.md) for architecture context.

---

## 1. CLI Implementation

Python + Click. Binary name: `vizier-cli`. Module: `vizier`.

### Commands

```
vizier-cli create <firman> --berat <berat> --repo <repo> [--name <name>] [--branch <branch>]
vizier-cli list [--status <status>]
vizier-cli status <province>
vizier-cli stop <province>
vizier-cli start <province>
vizier-cli destroy <province>
vizier-cli logs <province> [--follow] [--tail <n>]
```

### `vizier-cli create`

```python
@main.command()
@click.argument("firman")
@click.option("--berat", required=True, help="Berat (agent profile) name.")
@click.option("--repo", required=True, help="GitHub repo (owner/name).")
@click.option("--name", default=None, help="Province display name. Auto-generated if omitted.")
@click.option("--branch", default="main", help="Branch to clone.")
def create(firman: str, berat: str, repo: str, name: str | None, branch: str) -> None:
    """Create a province from a firman + berat."""
```

**Behavior:**
1. Validate firman exists at `/opt/sultanate/firmans/{firman}/firman.yaml`
2. Validate berat exists at `/opt/sultanate/berats/{berat}/berat.yaml`
3. Generate province ID: `prov-{8 hex chars}` (from `secrets.token_hex(4)`)
4. Auto-generate name if not provided: `{repo_shortname}-{4 hex chars}`
5. Call `province.create_province()` (see §5)
6. Print province ID and status on success, error message on failure

### `vizier-cli list`

```python
@main.command("list")
@click.option("--status", default=None, help="Filter by status.")
def list_provinces(status: str | None) -> None:
    """List all provinces with status."""
```

**Behavior:** `GET /provinces` (with optional `?status=` filter) from
Divan. Print table:

```
ID            NAME               STATUS    FIRMAN             BERAT                       REPO
prov-a1b2c3d4 backend-refactor   running   openclaw-firman    openclaw-coding-berat       stranma/EFM
```

### `vizier-cli status`

```python
@main.command()
@click.argument("province")
def status(province: str) -> None:
    """Show detailed province status."""
```

**Behavior:** `GET /provinces/{province}` from Divan. Print all fields.
Also runs `docker inspect sultanate-{name}` for container-level info
(IP, state, uptime).

### `vizier-cli stop`

```python
@main.command()
@click.argument("province")
def stop(province: str) -> None:
    """Stop a running province."""
```

**Behavior:**
1. `GET /provinces/{province}` — verify status is `running`
2. `docker stop sultanate-{name}` + `docker stop wg-client-{id}`
3. `PATCH /provinces/{province}` with `{"status": "stopped"}`

### `vizier-cli start`

```python
@main.command()
@click.argument("province")
def start(province: str) -> None:
    """Start a stopped province."""
```

**Behavior:**
1. `GET /provinces/{province}` — verify status is `stopped`
2. `docker start wg-client-{id}` (re-establish WireGuard tunnel first)
3. `docker start sultanate-{name}`
4. `docker exec -d sultanate-{name} openclaw gateway --port 18789`
5. `PATCH /provinces/{province}` with `{"status": "running"}`

### `vizier-cli destroy`

```python
@main.command()
@click.argument("province")
def destroy(province: str) -> None:
    """Destroy a province and clean up."""
```

**Behavior:**
1. `PATCH /provinces/{province}` with `{"status": "destroying"}`
2. `docker rm -f sultanate-{name}`
3. `docker rm -f wg-client-{id}`
4. Remove the WireGuard peer from Janissary's `wg0.conf`
   (HUP Janissary to reload)
5. Clean up host volume at `/opt/sultanate/provinces/{id}/data`
6. `DELETE /grants?province_id={id}` (clean grants in Divan)
7. (Aga sees status=destroying and does its own cleanup in parallel)

### `vizier-cli logs`

```python
@main.command()
@click.argument("province")
@click.option("--follow", "-f", is_flag=True, help="Follow log output.")
@click.option("--tail", default=100, help="Number of lines from end.")
def logs(province: str, follow: bool, tail: int) -> None:
    """View province logs."""
```

**Behavior:** `docker logs sultanate-{name} --tail {tail}` (add
`--follow` if `-f`). Streams stdout directly.

---

## 2. Firman Artifact Format

A firman is a directory at `/opt/sultanate/firmans/{name}/` containing a
`firman.yaml` manifest.

### Directory Layout

```
/opt/sultanate/firmans/openclaw-firman/
  firman.yaml
```

### firman.yaml Schema

```yaml
# firman.yaml -- container template manifest
apiVersion: firman/v1
kind: Firman
metadata:
  name: openclaw-firman
  description: "OpenClaw agent container template for Sultanate provinces"

image:
  repository: openclaw/openclaw
  tag: "v2026.4.15"                          # pinned by digest in deploy

bootstrap:
  # Commands run inside the container after start, before berat application.
  # Executed sequentially via docker exec. Each is a shell command string.
  commands:
    - "update-ca-certificates"               # trust Sultanate CA cert

startup:
  # Command to start the OpenClaw daemon inside the container.
  # Executed via docker exec -d after berat is applied.
  command: "openclaw gateway"
  args: [ "--port", "18789" ]

defaults:
  branch: main
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiVersion` | string | yes | Always `firman/v1` |
| `kind` | string | yes | Always `Firman` |
| `metadata.name` | string | yes | Firman identifier, matches directory name |
| `metadata.description` | string | no | Human-readable description |
| `image.repository` | string | yes | Docker image repository |
| `image.tag` | string | yes | Docker image tag |
| `bootstrap.commands` | list[string] | no | Bootstrap commands run via `docker exec` |
| `startup.command` | string | yes | OpenClaw startup command |
| `startup.args` | list[string] | no | Arguments to startup command |
| `defaults.branch` | string | no | Default branch (default: `main`) |

### Vizier Resolution

```python
def load_firman(name: str) -> dict:
    path = Path(f"/opt/sultanate/firmans/{name}/firman.yaml")
    if not path.exists():
        raise click.ClickException(f"Firman not found: {name}")
    with open(path) as f:
        data = yaml.safe_load(f)
    # Validate required fields
    assert data["apiVersion"] == "firman/v1"
    assert data["image"]["repository"]
    assert data["image"]["tag"]
    assert data["startup"]["command"]
    return data
```

---

## 3. Berat Artifact Format

A berat is a directory at `/opt/sultanate/berats/{name}/` containing a
manifest and template files. OpenClaw auto-loads several workspace-root
files (`SOUL.md`, `AGENTS.md`, `IDENTITY.md`, `USER.md`, `TOOLS.md`,
`BOOTSTRAP.md`); the berat provides templates for the ones we want.

### Directory Layout

```
/opt/sultanate/berats/openclaw-coding-berat/
  berat.yaml
  templates/
    SOUL.md
    AGENTS.md
    IDENTITY.md
    openclaw.json
```

### berat.yaml Schema

```yaml
# berat.yaml -- agent profile manifest
apiVersion: berat/v1
kind: Berat
metadata:
  name: openclaw-coding-berat
  description: "Coding agent profile for software development tasks"

defaults:
  pasha_name: "Pasha"
  extra_instructions: ""
  model: "anthropic/claude-sonnet-4"

# Variables declared here. Vizier validates required ones are provided.
variables:
  - name: province_id
    required: true
    source: auto               # auto-generated by Vizier
  - name: province_name
    required: true
    source: auto               # auto-generated or --name
  - name: pasha_name
    required: false
    source: default            # from defaults.pasha_name
  - name: repo_name
    required: true
    source: cli                # from --repo
  - name: extra_instructions
    required: false
    source: default            # from defaults.extra_instructions
  - name: model
    required: false
    source: default            # from defaults.model
  - name: pasha_telegram_bot_token
    required: true
    source: vizier             # Vizier provisions a bot via BotFather at create
  - name: janissary_api
    required: true
    source: vizier             # injected by Vizier (http://10.13.13.1:8081)

templates:
  soul:     templates/SOUL.md                # relative to berat directory
  agents:   templates/AGENTS.md
  identity: templates/IDENTITY.md
  config:   templates/openclaw.json

security:
  whitelist:
    domains:
      - github.com
      - api.github.com
      - pypi.org
      - files.pythonhosted.org
      - registry.npmjs.org
      - cdn.jsdelivr.net
      - docs.python.org
      - stackoverflow.com

  grants:
    # Grant templates. Aga reads these and mints actual tokens via
    # OpenBao (GitHub App primary) or asks Sultan (KV fallback).
    # Vizier does NOT know token values.
    - service: github
      domains:
        - api.github.com
        - github.com
      header: Authorization
      description: "GitHub API + git operations via GitHub App token"
      kind: dynamic             # "dynamic" for GitHub App, "kv" for Sultan-pasted

  port_declarations:
    # Non-HTTP ports that require Sultan approval.
    - host: github.com
      port: 22
      protocol: tcp
      reason: "Git SSH (optional; HTTPS default)"
```

### Field Reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `apiVersion` | string | yes | Always `berat/v1` |
| `kind` | string | yes | Always `Berat` |
| `metadata.name` | string | yes | Berat identifier, matches directory name |
| `metadata.description` | string | no | Human-readable description |
| `defaults.*` | map | no | Default values for template variables |
| `variables` | list | yes | Variable declarations with name, required, source |
| `templates.soul` | string | yes | Path to SOUL.md template |
| `templates.agents` | string | yes | Path to AGENTS.md template |
| `templates.identity` | string | no | Path to IDENTITY.md template (optional) |
| `templates.config` | string | yes | Path to openclaw.json template |
| `security.whitelist.domains` | list[string] | no | Domains to whitelist at creation |
| `security.grants` | list | no | Grant templates (service, domains, header, kind) |
| `security.grants[].service` | string | yes | Logical service name (e.g. `github`) |
| `security.grants[].domains` | list[string] | yes | Domains this grant applies to |
| `security.grants[].header` | string | yes | HTTP header to inject |
| `security.grants[].kind` | string | no | `dynamic` (default for github) or `kv` (Sultan-pasted) |
| `security.port_declarations` | list | no | Non-HTTP port requests |

### Vizier Resolution

```python
def load_berat(name: str) -> dict:
    path = Path(f"/opt/sultanate/berats/{name}/berat.yaml")
    if not path.exists():
        raise click.ClickException(f"Berat not found: {name}")
    with open(path) as f:
        data = yaml.safe_load(f)
    assert data["apiVersion"] == "berat/v1"
    assert data["templates"]["soul"]
    assert data["templates"]["agents"]
    assert data["templates"]["config"]
    return data
```

---

## 4. Template Rendering

Simple `str.replace()` for `{{variable}}` placeholders. No Jinja2, no
expression evaluation.

### Rendering Logic

```python
def render_template(template_path: Path, context: dict[str, str]) -> str:
    """Render a berat template with variable substitution.

    :param template_path: Absolute path to the template file.
    :param context: Dict of variable_name -> value.
    :returns: Rendered string.
    :raises ValueError: If a required variable is missing from context.
    """
    content = template_path.read_text()
    for key, value in context.items():
        content = content.replace(f"{{{{{key}}}}}", value)
    return content


def build_template_context(
    province_id: str,
    province_name: str,
    repo: str,
    pasha_bot_token: str,
    janissary_api: str,
    berat_data: dict,
) -> dict[str, str]:
    """Build template context dict from CLI args + berat defaults + runtime.

    :param province_id: Generated province ID.
    :param province_name: Province name (user-provided or auto-generated).
    :param repo: Repository name from --repo.
    :param pasha_bot_token: Telegram bot token provisioned by Vizier.
    :param janissary_api: Janissary appeal API URL (http://10.13.13.1:8081).
    :param berat_data: Parsed berat.yaml dict.
    :returns: Context dict for template rendering.
    """
    defaults = berat_data.get("defaults", {})
    return {
        "province_id": province_id,
        "province_name": province_name,
        "pasha_name": defaults.get("pasha_name", "Pasha"),
        "repo_name": repo,
        "extra_instructions": defaults.get("extra_instructions", ""),
        "model": defaults.get("model", "anthropic/claude-sonnet-4"),
        "pasha_telegram_bot_token": pasha_bot_token,
        "janissary_api": janissary_api,
    }
```

### Validation

1. **Required variables:** After building context, verify all variables
   marked `required: true` in `berat.yaml` have non-empty values.
   `repo_name` is always required.
2. **Unreplaced variables:** After rendering, scan for remaining `{{...}}`
   patterns. Warn (do not fail) for any leftovers -- they indicate a
   berat/template mismatch.
3. **JSON safety:** After rendering `openclaw.json`, validate it parses
   as valid JSON:

```python
def validate_rendered_json(content: str) -> dict:
    """Parse rendered openclaw.json to validate it's valid JSON.

    :raises click.ClickException: If JSON parsing fails.
    """
    try:
        return json.loads(content)
    except json.JSONDecodeError as e:
        raise click.ClickException(
            f"Rendered openclaw.json is invalid JSON: {e}"
        )
```

---

## 5. Province Creation Flow

Triggered by `vizier-cli create <firman> --berat <berat> --repo <repo>
[--name <name>] [--branch <branch>]`.

### Step-by-step

#### Step (a): Generate province ID and name

```python
import secrets
province_id = f"prov-{secrets.token_hex(4)}"  # e.g. "prov-a1b2c3d4"
province_name = name or f"{repo.split('/')[-1].lower()}-{secrets.token_hex(2)}"
```

#### Step (b): Load firman and berat

```python
firman_data = load_firman(firman)
berat_data = load_berat(berat)
berat_dir = Path(f"/opt/sultanate/berats/{berat}")
```

#### Step (c): Generate WireGuard peer config

```python
# Allocate the next available IP in 10.13.13.0/24
wg_peer_ip = allocate_wg_peer_ip(divan)        # e.g., "10.13.13.5"
wg_conf_dir = Path(f"/opt/sultanate/provinces/{province_id}")
wg_conf_dir.mkdir(parents=True, exist_ok=True)

# Generate keypair
private_key, public_key = generate_wg_keypair()
write_wg_conf(wg_conf_dir / "wg0.conf",
              wg_peer_ip, private_key, janissary_pub_key, janissary_endpoint)

# Add peer to Janissary server config and HUP
add_peer_to_janissary_config(public_key, wg_peer_ip)
subprocess.run(["docker", "kill", "-s", "HUP", "janissary"], check=True)
```

#### Step (d): Provision Pasha Telegram bot

Vizier owns the per-province bot lifecycle. For MVP, Vizier keeps a pool
of pre-created bots (tokens provisioned once at deploy time) and assigns
one per province. On destroy, the bot returns to the pool. Phase 2 may
auto-create via BotFather API.

```python
pasha_bot_token = bot_pool.acquire(province_id)
```

#### Step (e): Create Docker containers

```python
image = f"{firman_data['image']['repository']}:{firman_data['image']['tag']}"
container_name = f"sultanate-{sanitize_name(province_name)}"
sidecar_name = f"wg-client-{province_id}"
host_volume = f"/opt/sultanate/provinces/{province_id}/data"
ca_cert_host = "/opt/sultanate/certs/sultanate-ca.pem"

os.makedirs(host_volume, exist_ok=True)

# Create wg-client sidecar (WireGuard + iptables kill-switch)
subprocess.run([
    "docker", "create",
    "--name", sidecar_name,
    "--cap-add", "NET_ADMIN",
    "--volume", f"{wg_conf_dir}/wg0.conf:/etc/wireguard/wg0.conf:ro",
    "--env", "MITMPROXY_HOST=10.13.13.1",
    "--restart", "unless-stopped",
    "sultanate/wg-client:latest",
], check=True)

# Create province container sharing sidecar's network namespace
subprocess.run([
    "docker", "create",
    "--name", container_name,
    "--network-mode", f"container:{sidecar_name}",
    "--env", f"JANISSARY_API=http://10.13.13.1:8081",
    "--env", f"OPENCLAW_HOME=/opt/data",
    "--volume", f"{host_volume}:/opt/data",
    "--volume", f"{ca_cert_host}:/usr/local/share/ca-certificates/sultanate-ca.crt:ro",
    image,
], check=True)
```

#### Step (f): Register province in Divan

```python
divan.post("/provinces", json={
    "id":     province_id,
    "name":   province_name,
    "ip":     wg_peer_ip,
    "status": "creating",
    "firman": firman,
    "berat":  berat,
    "repo":   repo,
    "branch": branch,
})
```

#### Step (g): Post port declarations to Divan

```python
for port_decl in berat_data.get("security", {}).get("port_declarations", []):
    divan.post("/port_requests", json={
        "province_id": province_id,
        "host":        port_decl["host"],
        "port":        port_decl["port"],
        "protocol":    port_decl["protocol"],
        "reason":      port_decl["reason"],
    })
```

#### Step (h): Set default whitelist in Divan

```python
whitelist_domains = berat_data.get("security", {}) \
                              .get("whitelist", {}).get("domains", [])
divan.put(f"/whitelists/{province_id}", json={
    "domains": whitelist_domains,
})
```

#### Step (i): Start containers

Aga polls Divan in parallel; by the time Step (j) runs the berat
application, Aga has probably already written the GitHub App token to
Divan's grants table (for provinces whose repo is already covered by
the installed App). Janissary will then inject on the first HTTPS call.

```python
# Start wg-client sidecar first (establishes WireGuard tunnel)
subprocess.run(["docker", "start", sidecar_name], check=True)

# Wait for WireGuard ping to Janissary
wait_for_wg_tunnel(wg_peer_ip, timeout=30)

# Start province container (shares sidecar's network namespace)
subprocess.run(["docker", "start", container_name], check=True)
```

#### Step (j): Run bootstrap commands + clone repo

```python
# Run firman bootstrap commands
for cmd in firman_data.get("bootstrap", {}).get("commands", []):
    subprocess.run(
        ["docker", "exec", container_name, "bash", "-c", cmd],
        check=True,
    )

# Clone repo into workspace. Janissary injects the GitHub token on
# api.github.com and github.com transparently (provided Aga has
# written the grant by now).
clone_cmd = (
    f"git clone https://github.com/{repo}.git "
    f"/opt/data/workspace --branch {branch}"
)
subprocess.run(
    ["docker", "exec", container_name, "bash", "-c", clone_cmd],
    check=True,
)
```

If the clone fails with a 401 (Aga hasn't written the grant yet), Vizier
waits 5 s and retries up to 3 times. If it still fails, set status=failed.

#### Step (k): Render and write berat templates

```python
context = build_template_context(
    province_id,
    province_name,
    repo,
    pasha_bot_token,
    janissary_api="http://10.13.13.1:8081",
    berat_data=berat_data,
)

soul_content     = render_template(berat_dir / berat_data["templates"]["soul"],     context)
agents_content   = render_template(berat_dir / berat_data["templates"]["agents"],   context)
identity_content = render_template(berat_dir / berat_data["templates"]["identity"], context) \
                   if "identity" in berat_data["templates"] else None
config_content   = render_template(berat_dir / berat_data["templates"]["config"],   context)

# Validate openclaw.json
validate_rendered_json(config_content)

# OpenClaw's auto-load paths:
#   workspace/SOUL.md, workspace/AGENTS.md, workspace/IDENTITY.md
#   ~/.openclaw/openclaw.json
# "workspace" for us is /opt/data/workspace; OpenClaw's home-equivalent
# is /opt/data/.openclaw/ when OPENCLAW_HOME=/opt/data.

writes = [
    ("/opt/data/workspace/SOUL.md",       soul_content),
    ("/opt/data/workspace/AGENTS.md",     agents_content),
    ("/opt/data/.openclaw/openclaw.json", config_content),
]
if identity_content is not None:
    writes.append(("/opt/data/workspace/IDENTITY.md", identity_content))

# Ensure target directories exist
subprocess.run(
    ["docker", "exec", container_name, "bash", "-c",
     "mkdir -p /opt/data/.openclaw /opt/data/workspace"],
    check=True,
)

# Write each file via docker exec (stdin pipe)
for dest, content in writes:
    subprocess.run(
        ["docker", "exec", "-i", container_name, "tee", dest],
        input=content.encode(),
        stdout=subprocess.DEVNULL,
        check=True,
    )
```

#### Step (l): Start OpenClaw gateway

```python
startup_cmd = firman_data["startup"]["command"]
startup_args = firman_data["startup"].get("args", [])
full_cmd = f"{startup_cmd} {' '.join(startup_args)}".strip()

# Start in background (detached exec)
subprocess.run(
    ["docker", "exec", "-d", container_name,
     "bash", "-c", f"OPENCLAW_HOME=/opt/data {full_cmd}"],
    check=True,
)

# Wait for OpenClaw's HTTP gateway health (port 18789 inside container)
wait_for_openclaw_health(container_name, timeout=30)
```

#### Step (m): Mark province as running

```python
divan.patch(f"/provinces/{province_id}", json={"status": "running"})
click.echo(f"Province {province_id} ({province_name}) is running.")
```

### Error Handling

If any step after (f) fails:
1. `PATCH /provinces/{province_id}` with `{"status": "failed"}`
2. Print error details to stderr
3. Leave the container for debugging (do not auto-destroy on failure)

---

## 6. Docker Integration Details

### Network

- **Network name:** `sultanate-internal` (legacy; used only for Vizier
  itself, not for provinces)
- **Driver:** bridge, `internal: true` (no external route)
- **Created by:** deploy script (once, before any component starts)

```bash
docker network create --driver bridge --internal sultanate-internal
```

Province containers do NOT attach to `sultanate-internal` directly.
Each province gets a `wg-client` sidecar container, and the province
shares the sidecar's network namespace
(`--network-mode container:wg-client-{id}`). The wg-client sidecar
establishes a WireGuard tunnel to Janissary and uses iptables to route
all traffic through mitmproxy.

Vizier itself attaches to `sultanate-internal` so it can reach Divan
(which runs on `network_mode: host`; reachable on
`http://divan-container-hostname:8600` via the default Docker DNS when
both are on `sultanate-internal`, OR `http://127.0.0.1:8600` if Vizier
also uses host networking -- deployer's choice).

### Container Naming

Pattern: `sultanate-{sanitized_province_name}`

Examples:
- `sultanate-backend-refactor`
- `sultanate-efm-a1b2`

Province name is sanitized: lowercase, alphanumeric + hyphens only,
max 48 chars.

```python
import re

def sanitize_name(name: str) -> str:
    """Sanitize province name for Docker container naming."""
    name = name.lower()
    name = re.sub(r"[^a-z0-9-]", "-", name)
    name = re.sub(r"-+", "-", name).strip("-")
    return name[:48]
```

### Volumes

| Host Path | Container Path | Mode | Purpose |
|-----------|---------------|------|---------|
| `/opt/sultanate/provinces/{id}/data` | `/opt/data` | rw | OpenClaw workspace + `.openclaw/openclaw.json` |
| `/opt/sultanate/certs/sultanate-ca.pem` | `/usr/local/share/ca-certificates/sultanate-ca.crt` | ro | Sultanate CA cert for HTTPS MITM trust |

### Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `OPENCLAW_HOME` | `/opt/data` | OpenClaw data directory (config under `.openclaw/`) |
| `JANISSARY_API` | `http://10.13.13.1:8081` | Janissary appeal HTTP API |

Province containers do not need proxy environment variables. All traffic
is transparently routed through Janissary via WireGuard.

### Docker Commands Reference

**Create wg-client sidecar:**

```bash
docker create \
  --name wg-client-{id} \
  --cap-add NET_ADMIN \
  -v /opt/sultanate/provinces/{id}/wg0.conf:/etc/wireguard/wg0.conf:ro \
  -e MITMPROXY_HOST=10.13.13.1 \
  --restart unless-stopped \
  sultanate/wg-client:latest
```

**Create province (shares sidecar network):**

```bash
docker create \
  --name sultanate-{name} \
  --network-mode container:wg-client-{id} \
  -e OPENCLAW_HOME=/opt/data \
  -e JANISSARY_API=http://10.13.13.1:8081 \
  -v /opt/sultanate/provinces/{id}/data:/opt/data \
  -v /opt/sultanate/certs/sultanate-ca.pem:/usr/local/share/ca-certificates/sultanate-ca.crt:ro \
  openclaw/openclaw:vYYYY.M.D
```

**Start:**

```bash
docker start sultanate-{name}
```

**Exec (bootstrap):**

```bash
docker exec sultanate-{name} bash -c "update-ca-certificates"
```

**Exec (clone):**

```bash
docker exec sultanate-{name} bash -c \
  "git clone https://github.com/{owner}/{repo}.git /opt/data/workspace --branch {branch}"
```

**Exec (write files):**

```bash
echo '<rendered content>' | docker exec -i sultanate-{name} tee /opt/data/workspace/SOUL.md > /dev/null
echo '<rendered content>' | docker exec -i sultanate-{name} tee /opt/data/workspace/AGENTS.md > /dev/null
echo '<rendered content>' | docker exec -i sultanate-{name} tee /opt/data/.openclaw/openclaw.json > /dev/null
```

**Exec (start OpenClaw):**

```bash
docker exec -d sultanate-{name} bash -c \
  "OPENCLAW_HOME=/opt/data openclaw gateway --port 18789"
```

**Stop:**

```bash
docker stop sultanate-{name}
docker stop wg-client-{id}
```

**Destroy:**

```bash
docker rm -f sultanate-{name}
docker rm -f wg-client-{id}
```

**Logs:**

```bash
docker logs sultanate-{name} --tail 100
docker logs sultanate-{name} --follow
```

**Inspect IP:**
Province IP is the WireGuard peer address from the generated `wg0.conf`,
not a Docker network IP. It is stored in Divan at creation time.

---

## 7. Appeal Relay

Vizier polls Divan for appeals requiring Sultan's decision and relays
them to Sultan via Telegram. Kashif auto-decides most appeals; Vizier
only forwards those requiring human decision.

### Polling Loop

```python
import time

ESCALATE_POLL = 10          # seconds
AUDIT_ALERT_POLL = 30       # seconds

def appeal_relay_loop(divan: DivanClient, telegram: TelegramClient,
                       state: RelayState) -> None:
    """Poll Divan for Kashif-escalated appeals and Kashif-block alerts."""
    while True:
        now = time.time()

        # Actionable: Kashif escalated to Sultan
        if now - state.last_escalate_poll > ESCALATE_POLL:
            try:
                resp = divan.get("/appeals", params={
                    "status": "pending",
                    "kashif_verdict": "escalate",
                })
                for appeal in resp["data"]:
                    if appeal["id"] not in state.seen_escalate:
                        relay_actionable(appeal, telegram)
                        state.seen_escalate.add(appeal["id"])
            except Exception as e:
                click.echo(f"Escalate poll error: {e}", err=True)
            state.last_escalate_poll = now

        # Informational: Kashif auto-blocked (for pattern awareness)
        if now - state.last_audit_poll > AUDIT_ALERT_POLL:
            try:
                resp = divan.get("/audit", params={
                    "severity": "alert",
                    "component": "kashif",
                    "since":     state.last_audit_since.isoformat(),
                })
                for entry in resp["data"]:
                    if entry["detail"].get("verdict") == "block":
                        relay_informational(entry, telegram)
                state.last_audit_since = datetime.now(timezone.utc)
            except Exception as e:
                click.echo(f"Audit poll error: {e}", err=True)
            state.last_audit_poll = now

        time.sleep(2)
```

### Message Formats

**Actionable (Kashif=escalate):**

```
🔒 Appeal NEEDS DECISION -- province {province_name} ({province_id})

URL: {method} {url}
Justification: {justification}

Kashif: escalate
Notes: {kashif_notes}

Reply: approve once / approve forever / deny / kill province
```

**Informational (Kashif=block):**

```
⚠️ Kashif AUTO-BLOCKED an appeal from {province_name}

URL: {method} {url}
Justification: {justification}
Kashif notes: {kashif_notes}

Decision is final. Flagging in case pattern repeats -- Aga tracks a
counter and will alert if >3 blocks in 10 min.
```

### Sultan Response Handling

| Sultan replies to actionable message | Vizier action |
|--------------------------------------|---------------|
| `approve` / `approve once` | `PATCH /appeals/{id}` with `{"status": "approved", "decision": "one-time"}` |
| `approve forever` / `whitelist` | Vizier tells Sultan: "OK, please also tell Aga to add {domain} to the whitelist." Vizier writes appeal `{"status": "approved", "decision": "whitelist"}` (Divan auto-adds domain to whitelist per the spec). |
| `deny` | `PATCH /appeals/{id}` with `{"status": "denied"}` |
| `kill` / `kill province` | Vizier runs `vizier-cli destroy {province_id}` and `PATCH /appeals/{id}` with `{"status": "denied"}`. |

### API Calls

**Poll escalated:**

```
GET /appeals?status=pending&kashif_verdict=escalate
Authorization: Bearer {DIVAN_KEY_VIZIER}
```

**Poll Kashif-alert audit:**

```
GET /audit?severity=alert&component=kashif&since={iso_timestamp}
Authorization: Bearer {DIVAN_KEY_VIZIER}
```

**Resolve (approve one-time):**

```
PATCH /appeals/{id}
Authorization: Bearer {DIVAN_KEY_VIZIER}
Content-Type: application/json

{"status": "approved", "decision": "one-time"}
```

---

## 8. Divan Client Module

File: `vizier/divan.py`

HTTP client using `httpx` for all Divan API communication.

### Configuration

| Env Var | Required | Description |
|---------|----------|-------------|
| `DIVAN_URL` | yes | Divan base URL (e.g. `http://127.0.0.1:8600`) |
| `DIVAN_KEY_VIZIER` | yes | Pre-shared API key for Vizier role |

### Implementation

```python
"""Divan HTTP client."""

import os
import time
from typing import Any

import httpx


class DivanClient:
    """HTTP client for Divan shared state API.

    :param base_url: Divan base URL. Defaults to DIVAN_URL env var.
    :param api_key: API key. Defaults to DIVAN_KEY_VIZIER env var.
    """

    MAX_RETRIES = 3
    RETRY_BACKOFF = 2.0  # seconds

    def __init__(self, base_url: str | None = None,
                 api_key: str | None = None) -> None:
        self.base_url = base_url or os.environ["DIVAN_URL"]
        self.api_key = api_key or os.environ["DIVAN_KEY_VIZIER"]
        self._client = httpx.Client(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {self.api_key}"},
            timeout=30.0,
        )

    def _request(self, method: str, path: str, **kwargs: Any) -> dict:
        """Make an HTTP request with retry logic.

        :raises httpx.HTTPStatusError: On 4xx/5xx responses (after retries
            for connection errors).
        """
        last_exc: Exception | None = None
        for attempt in range(self.MAX_RETRIES):
            try:
                response = self._client.request(method, path, **kwargs)
                response.raise_for_status()
                return response.json()
            except httpx.ConnectError as e:
                last_exc = e
                if attempt < self.MAX_RETRIES - 1:
                    time.sleep(self.RETRY_BACKOFF * (attempt + 1))
            except httpx.HTTPStatusError:
                raise  # 4xx errors are not retried
        raise last_exc  # type: ignore[misc]

    def get(self, path: str, params: dict | None = None) -> dict:
        return self._request("GET", path, params=params)

    def post(self, path: str, json: dict | None = None) -> dict:
        return self._request("POST", path, json=json)

    def put(self, path: str, json: dict | None = None) -> dict:
        return self._request("PUT", path, json=json)

    def patch(self, path: str, json: dict | None = None) -> dict:
        return self._request("PATCH", path, json=json)

    def delete(self, path: str, params: dict | None = None) -> dict:
        return self._request("DELETE", path, params=params)

    def health_check(self) -> bool:
        """Check if Divan is reachable."""
        try:
            resp = self._client.get("/health")
            return resp.status_code == 200
        except httpx.ConnectError:
            return False

    def wait_for_divan(self, timeout: float = 60.0) -> None:
        """Block until Divan is reachable or timeout expires."""
        start = time.monotonic()
        while time.monotonic() - start < timeout:
            if self.health_check():
                return
            time.sleep(2.0)
        raise TimeoutError(
            f"Divan not reachable at {self.base_url} after {timeout}s"
        )
```

---

## 9. OpenClaw Agent Wrapper

Vizier itself runs as an OpenClaw agent on the upstream
`openclaw/openclaw` image. It uses OpenClaw's `bash` tool to execute
`vizier-cli` commands and runs the appeal relay loop as a long-running
process inside the workspace.

### Deployment Path

Vizier's OpenClaw configuration and workspace live at
`/opt/sultanate/vizier/`.

```
/opt/sultanate/vizier/
  .openclaw/openclaw.json
  workspace/
    SOUL.md
    AGENTS.md
    appeal_relay.py    # the polling loop daemon
    bot_pool.json      # pre-provisioned Pasha Telegram bot tokens
```

### SOUL.md

```markdown
You are Vizier, the Grand Vizier of the Sultanate. You manage the realm
on behalf of the Sultan.

Your responsibilities:
- Create, start, stop, and destroy provinces on Sultan's command
- Report province status and health
- Relay escalated appeals from provinces to Sultan for decision
- Surface informational Kashif-auto-block events so Sultan can spot
  drift
- Execute realm management tasks via the vizier-cli CLI

You are precise and efficient. You execute Sultan's commands using the
bash tool to run vizier-cli commands. You do not improvise security
policy -- if something is blocked, Kashif triages; if Kashif escalates,
you relay to Sultan; Sultan decides.

You never attempt to read secrets, modify network rules, or bypass
security. If an agent needs new access, you tell Sultan to ask Aga
directly -- Aga has the privileges for grants, whitelists, and
iptables.

Province management commands:
- vizier-cli create <firman> --berat <berat> --repo <repo> [--name <name>]
- vizier-cli list [--status <status>]
- vizier-cli status <province>
- vizier-cli stop <province>
- vizier-cli start <province>
- vizier-cli destroy <province>
- vizier-cli logs <province> [--follow] [--tail <n>]
```

### AGENTS.md

```markdown
# Working Rules for Vizier

- Divan URL: {{DIVAN_URL}}
- Janissary API: http://janissary:8081
- You are a semi-trusted agent. You can create/manage containers but
  cannot read secrets or modify network rules.
- The appeal relay runs as a background process
  (workspace/appeal_relay.py). Do not duplicate its polling in
  foreground conversation.
- When Sultan asks a high-level question ("create a coding agent for
  repo X"), translate it into a concrete vizier-cli create command
  and run it via bash.
```

### openclaw.json

```json
{
  "agent": {
    "model": "anthropic/claude-sonnet-4",
    "workspace": "/opt/sultanate/vizier/workspace"
  },
  "agents": {
    "defaults": {
      "workspace": "/opt/sultanate/vizier/workspace",
      "sandbox": { "mode": "off" }
    }
  },
  "tools": {
    "exec": { "applyPatch": false }
  },
  "channels": {
    "telegram": {
      "botTokenEnv": "VIZIER_TELEGRAM_BOT_TOKEN",
      "allowFrom":   [ "${SULTAN_TELEGRAM_USER_ID}" ],
      "dmPolicy":    "pairing"
    }
  },
  "mcp_servers": {
    "janissary_security": {
      "transport": "http",
      "url":       "http://janissary:8081/mcp"
    }
  }
}
```

**Notes:**
- Only OpenClaw built-in `bash`/`read`/`write`/`edit` tools are used;
  Vizier interacts with the system primarily through `vizier-cli`
  commands issued via `bash`.
- Janissary security MCP is included so Vizier can appeal blocks on
  its own traffic (Vizier's container is also behind Janissary).
- Sandbox `off` because Vizier is semi-trusted (has Docker socket).

---

## 10. Deployment

### Docker Run (Vizier Container)

```bash
docker run -d \
  --name sultanate-vizier \
  --network sultanate-internal \
  --restart unless-stopped \
  -e DIVAN_URL=http://divan:8600 \
  -e DIVAN_KEY_VIZIER={vizier_api_key} \
  -e VIZIER_TELEGRAM_BOT_TOKEN={bot_token} \
  -e SULTAN_TELEGRAM_USER_ID={user_id} \
  -v /opt/sultanate/vizier:/opt/sultanate/vizier \
  -v /opt/sultanate/firmans:/opt/sultanate/firmans:ro \
  -v /opt/sultanate/berats:/opt/sultanate/berats:ro \
  -v /opt/sultanate/provinces:/opt/sultanate/provinces \
  -v /opt/sultanate/certs/sultanate-ca.pem:/usr/local/share/ca-certificates/sultanate-ca.crt:ro \
  -v /var/run/docker.sock:/var/run/docker.sock \
  --security-opt no-new-privileges:true \
  openclaw/openclaw:vYYYY.M.D
```

### Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `DIVAN_URL` | `http://divan:8600` | Divan API base URL |
| `DIVAN_KEY_VIZIER` | `{generated at deploy}` | API key for Vizier role |
| `VIZIER_TELEGRAM_BOT_TOKEN` | `{from BotFather}` | Vizier's own bot token |
| `SULTAN_TELEGRAM_USER_ID` | `{Sultan's Telegram ID}` | Access control |
| `OPENCLAW_HOME` | `/opt/sultanate/vizier` | OpenClaw data dir |

### Volumes

| Host Path | Container Path | Mode | Purpose |
|-----------|---------------|------|---------|
| `/opt/sultanate/vizier` | `/opt/sultanate/vizier` | rw | Vizier's OpenClaw workspace + `.openclaw/openclaw.json` |
| `/opt/sultanate/firmans` | `/opt/sultanate/firmans` | ro | Firman manifests |
| `/opt/sultanate/berats` | `/opt/sultanate/berats` | ro | Berat manifests + templates |
| `/opt/sultanate/provinces` | `/opt/sultanate/provinces` | rw | Province data volumes + wg0.conf files |
| `/opt/sultanate/certs/sultanate-ca.pem` | `/usr/local/share/ca-certificates/sultanate-ca.crt` | ro | CA cert for HTTPS trust |
| `/var/run/docker.sock` | `/var/run/docker.sock` | rw | Docker socket for container management |

### Network

Vizier is on `sultanate-internal`. All egress (Telegram API, any
non-Divan call) routes through Janissary via WireGuard transparent
proxy. The Docker socket is mounted so Vizier can create/manage
province containers on the host.

### Startup Order

Vizier starts after OpenBao, Divan, Janissary, Kashif, and Aga are
ready (see SULTANATE_MVP.md §Startup Order):

```
1. OpenBao    (Secret Vault; Sultan manually unseals)
2. Divan      (shared state + dashboard)
3. Janissary  (proxy)
4. Kashif     (content inspector)
5. Aga        (secrets)
6. Vizier     (this component)
7. Provinces  (on demand)
```

Vizier's `DivanClient.wait_for_divan()` blocks until Divan returns
`200` on `/health`.

---

## Appendix: Module Map

```
vizier/
  __init__.py         Package init
  __main__.py         Entry point for python -m vizier
  cli.py              Click CLI commands
  config.py           Configuration (env vars, paths)
  divan.py            DivanClient (httpx)
  docker.py           Docker subprocess wrapper
  models.py           Pydantic models (Province, Firman, Berat)
  province.py         Province lifecycle (create, start, stop, destroy)
  templates.py        Template rendering (str.replace, JSON/YAML validation)
  appeals.py          Appeal + Kashif-audit relay polling loop
  wireguard.py        WireGuard peer allocation, keygen, wg0.conf emit
  bot_pool.py         Pre-provisioned Pasha Telegram bot assignment
```
