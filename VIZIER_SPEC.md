# Vizier Technical Specification

> Province orchestrator CLI + Hermes agent wrapper for Sultanate.
> See [VIZIER_MVP_PRD.md](VIZIER_MVP_PRD.md) for product requirements,
> [SULTANATE_MVP.md](SULTANATE_MVP.md) for architecture context.

---

## 1. CLI Implementation

Python + Click. Scaffold exists at `vizier/cli.py`.

**Scaffold fixes required:**
- Add `--repo` parameter to `create` command (required argument per PRD)
- Update `pyproject.toml`: change `requires-python = ">=3.11,<3.13"` to `requires-python = ">=3.11"`

### Commands

```
vizier create <firman> --berat <berat> --repo <repo> [--name <name>] [--branch <branch>]
vizier list [--status <status>]
vizier status <province>
vizier stop <province>
vizier start <province>
vizier destroy <province>
vizier logs <province> [--follow] [--tail <n>]
```

### `vizier create`

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

### `vizier list`

```python
@main.command("list")
@click.option("--status", default=None, help="Filter by status.")
def list_provinces(status: str | None) -> None:
    """List all provinces with status."""
```

**Behavior:** `GET /provinces` (with optional `?status=` filter) from Divan. Print table:

```
ID            NAME               STATUS    FIRMAN          BERAT                REPO
prov-a1b2c3d4 backend-refactor   running   hermes-firman   hermes-coding-berat  stranma/EFM
```

### `vizier status`

```python
@main.command()
@click.argument("province")
def status(province: str) -> None:
    """Show detailed province status."""
```

**Behavior:** `GET /provinces/{province}` from Divan. Print all fields. Also runs `docker inspect sultanate-{name}` for container-level info (IP, state, uptime).

### `vizier stop`

```python
@main.command()
@click.argument("province")
def stop(province: str) -> None:
    """Stop a running province."""
```

**Behavior:**
1. `GET /provinces/{province}` — verify status is `running`
2. `docker stop sultanate-{name}`
3. `PATCH /provinces/{province}` with `{"status": "stopped"}`

### `vizier start`

```python
@main.command()
@click.argument("province")
def start(province: str) -> None:
    """Start a stopped province."""
```

**Behavior:**
1. `GET /provinces/{province}` — verify status is `stopped`
2. `docker start sultanate-{name}`
3. `docker exec` to restart Hermes gateway inside container
4. `PATCH /provinces/{province}` with `{"status": "running"}`

### `vizier destroy`

```python
@main.command()
@click.argument("province")
def destroy(province: str) -> None:
    """Destroy a province and clean up."""
```

**Behavior:**
1. `PATCH /provinces/{province}` with `{"status": "destroying"}`
2. `docker rm -f sultanate-{name}`
3. Clean up host volume at `/opt/sultanate/provinces/{id}/data`
4. `DELETE /grants?province_id={id}` (clean grants in Divan)

### `vizier logs`

```python
@main.command()
@click.argument("province")
@click.option("--follow", "-f", is_flag=True, help="Follow log output.")
@click.option("--tail", default=100, help="Number of lines from end.")
def logs(province: str, follow: bool, tail: int) -> None:
    """View province logs."""
```

**Behavior:** `docker logs sultanate-{name} --tail {tail}` (add `--follow` if `-f`). Streams stdout directly.

---

## 2. Firman Artifact Format

A firman is a directory at `/opt/sultanate/firmans/{name}/` containing a `firman.yaml` manifest.

### Directory Layout

```
/opt/sultanate/firmans/hermes-firman/
  firman.yaml
```

### firman.yaml Schema

```yaml
# firman.yaml -- container template manifest
apiVersion: firman/v1
kind: Firman
metadata:
  name: hermes-firman                        # unique identifier
  description: "Hermes Agent container template for Sultanate provinces"

image:
  repository: nousresearch/hermes-agent      # Docker image repository
  tag: "v0.6.0"                              # pinned tag for stability

bootstrap:
  # Commands run inside the container after start, before berat application.
  # Executed sequentially via docker exec. Each is a shell command string.
  commands:
    - "update-ca-certificates"               # trust Sultanate CA cert
    - "git config --global http.proxy $HTTP_PROXY"
    - "git config --global https.proxy $HTTPS_PROXY"

startup:
  # Command to start the Hermes agent runtime inside the container.
  # Executed via docker exec after berat is applied.
  command: "hermes gateway"
  args: []

defaults:
  branch: main                               # default git branch
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
| `startup.command` | string | yes | Hermes startup command |
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

A berat is a directory at `/opt/sultanate/berats/{name}/` containing a manifest and template files.

### Directory Layout

```
/opt/sultanate/berats/hermes-coding-berat/
  berat.yaml
  templates/
    SOUL.md
    AGENTS.md
    config.yaml
```

### berat.yaml Schema

```yaml
# berat.yaml -- agent profile manifest
apiVersion: berat/v1
kind: Berat
metadata:
  name: hermes-coding-berat
  description: "Coding agent profile for software development tasks"

defaults:
  pasha_name: "Pasha"
  extra_instructions: ""
  model: "anthropic/claude-sonnet-4-20250514"
  model_provider: "openrouter"

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

templates:
  soul: templates/SOUL.md          # relative to berat directory
  agents: templates/AGENTS.md
  config: templates/config.yaml

security:
  whitelist:
    # Domains the province can access without restriction.
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
    # Grant templates. Sentinel reads these and provisions actual tokens.
    # Vizier does NOT know token values -- Sentinel fills them via Infisical.
    - domain: api.github.com
      header: Authorization
      description: "GitHub API token (scoped to repo)"
    - domain: github.com
      header: Authorization
      description: "GitHub token for git operations"

  port_declarations:
    # Non-HTTP ports that require Sultan approval.
    - host: github.com
      port: 22
      protocol: tcp
      reason: "Git SSH (optional, HTTPS default)"
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
| `templates.soul` | string | yes | Path to SOUL.md template (relative) |
| `templates.agents` | string | yes | Path to AGENTS.md template (relative) |
| `templates.config` | string | yes | Path to config.yaml template (relative) |
| `security.whitelist.domains` | list[string] | no | Domains to whitelist at creation |
| `security.grants` | list | no | Grant templates for Sentinel to provision |
| `security.grants[].domain` | string | yes | Domain this grant applies to |
| `security.grants[].header` | string | yes | HTTP header to inject |
| `security.grants[].description` | string | no | Human-readable description |
| `security.port_declarations` | list | no | Non-HTTP port requests |
| `security.port_declarations[].host` | string | yes | Target host |
| `security.port_declarations[].port` | int | yes | Port number |
| `security.port_declarations[].protocol` | string | yes | Protocol (`tcp`) |
| `security.port_declarations[].reason` | string | yes | Justification for Sultan |

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

Simple `str.replace()` for `{{variable}}` placeholders. No Jinja2, no expression evaluation.

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
    berat_data: dict,
) -> dict[str, str]:
    """Build template context dict from CLI args + berat defaults.

    :param province_id: Generated province ID.
    :param province_name: Province name (user-provided or auto-generated).
    :param repo: Repository name from --repo.
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
    }
```

### Validation

1. **Required variables:** After building context, verify all variables marked `required: true` in `berat.yaml` have non-empty values. `repo_name` is always required.
2. **Unreplaced variables:** After rendering, scan for remaining `{{...}}` patterns. Warn (do not fail) for any leftovers — they indicate a berat/template mismatch.
3. **YAML safety:** After rendering `config.yaml`, validate it parses as valid YAML:

```python
def validate_rendered_yaml(content: str) -> dict:
    """Parse rendered config.yaml to validate it's valid YAML.

    :raises click.ClickException: If YAML parsing fails.
    """
    try:
        return yaml.safe_load(content)
    except yaml.YAMLError as e:
        raise click.ClickException(f"Rendered config.yaml is invalid YAML: {e}")
```

---

## 5. Province Creation Flow

Triggered by `vizier create <firman> --berat <berat> --repo <repo> [--name <name>] [--branch <branch>]`.

### Step-by-step

#### Step (a): Generate province ID

```python
import secrets
province_id = f"prov-{secrets.token_hex(4)}"  # e.g. "prov-a1b2c3d4"
province_name = name or f"{repo.split('/')[-1].lower()}-{secrets.token_hex(2)}"
```

#### Step (b): Load firman and berat

```python
firman_data = load_firman(firman)       # /opt/sultanate/firmans/{firman}/firman.yaml
berat_data = load_berat(berat)          # /opt/sultanate/berats/{berat}/berat.yaml
berat_dir = Path(f"/opt/sultanate/berats/{berat}")
```

#### Step (c): Create Docker container

```python
image = f"{firman_data['image']['repository']}:{firman_data['image']['tag']}"
container_name = f"sultanate-{province_name}"
host_volume = f"/opt/sultanate/provinces/{province_id}/data"
ca_cert_host = "/opt/sultanate/ca/sultanate-ca.crt"

# Ensure host volume directory exists
os.makedirs(host_volume, exist_ok=True)

subprocess.run([
    "docker", "create",
    "--name", container_name,
    "--network", "sultanate-internal",
    "--env", f"HTTP_PROXY=http://janissary:8080",
    "--env", f"HTTPS_PROXY=http://janissary:8080",
    "--env", f"NO_PROXY=divan",
    "--env", f"HERMES_HOME=/opt/data",
    "--volume", f"{host_volume}:/opt/data",
    "--volume", f"{ca_cert_host}:/usr/local/share/ca-certificates/sultanate-ca.crt:ro",
    image,
], check=True)
```

#### Step (d): Register province in Divan

```python
divan.post("/provinces", json={
    "id": province_id,
    "name": province_name,
    "ip": "",                  # filled after container starts
    "status": "creating",
    "firman": firman,
    "berat": berat,
    "repo": repo,
    "branch": branch,
})
```

#### Step (e): Post port declarations to Divan

```python
for port_decl in berat_data.get("security", {}).get("port_declarations", []):
    divan.post("/port_requests", json={
        "province_id": province_id,
        "host": port_decl["host"],
        "port": port_decl["port"],
        "protocol": port_decl["protocol"],
        "reason": port_decl["reason"],
    })
```

#### Step (f): Set default whitelist in Divan

```python
whitelist_domains = berat_data.get("security", {}).get("whitelist", {}).get("domains", [])
divan.put(f"/whitelists/{province_id}", json={
    "domains": whitelist_domains,
})
```

#### Step (g): Start container

```python
subprocess.run(["docker", "start", container_name], check=True)

# Get container IP for Divan update
result = subprocess.run(
    ["docker", "inspect", "-f", "{{.NetworkSettings.Networks.sultanate-internal.IPAddress}}", container_name],
    capture_output=True, text=True, check=True,
)
container_ip = result.stdout.strip()

# Update Divan with IP
divan.patch(f"/provinces/{province_id}", json={"ip": container_ip})
```

#### Step (h): Run bootstrap commands + clone repo

```python
# Run firman bootstrap commands
for cmd in firman_data.get("bootstrap", {}).get("commands", []):
    subprocess.run(
        ["docker", "exec", container_name, "bash", "-c", cmd],
        check=True,
    )

# Clone repo into workspace
clone_cmd = f"git clone https://github.com/{repo}.git /opt/data/workspace --branch {branch}"
subprocess.run(
    ["docker", "exec", container_name, "bash", "-c", clone_cmd],
    check=True,
)
```

#### Step (i): Render and write berat templates

```python
context = build_template_context(province_id, province_name, repo, berat_data)

# Render templates
soul_content = render_template(berat_dir / berat_data["templates"]["soul"], context)
agents_content = render_template(berat_dir / berat_data["templates"]["agents"], context)
config_content = render_template(berat_dir / berat_data["templates"]["config"], context)

# Validate config.yaml
validate_rendered_yaml(config_content)

# Write into container via docker exec (stdin pipe)
for dest, content in [
    ("/opt/data/SOUL.md", soul_content),
    ("/opt/data/workspace/AGENTS.md", agents_content),
    ("/opt/data/config.yaml", config_content),
]:
    subprocess.run(
        ["docker", "exec", "-i", container_name, "tee", dest],
        input=content.encode(),
        stdout=subprocess.DEVNULL,
        check=True,
    )
```

#### Step (j): Start Hermes gateway

```python
startup_cmd = firman_data["startup"]["command"]
startup_args = firman_data["startup"].get("args", [])
full_cmd = f"{startup_cmd} {' '.join(startup_args)}".strip()

# Start in background (detached exec)
subprocess.run(
    ["docker", "exec", "-d", container_name, "bash", "-c", full_cmd],
    check=True,
)
```

#### Step (k): Mark province as running

```python
divan.patch(f"/provinces/{province_id}", json={"status": "running"})
click.echo(f"Province {province_id} ({province_name}) is running.")
```

### Error Handling

If any step after (d) fails:
1. `PATCH /provinces/{province_id}` with `{"status": "failed"}`
2. Print error details to stderr
3. Leave the container for debugging (do not auto-destroy on failure)

---

## 6. Docker Integration Details

### Network

- **Network name:** `sultanate-internal`
- **Driver:** bridge, `internal: true` (no external route)
- **Created by:** deploy script (once, before any component starts)

```bash
docker network create --driver bridge --internal sultanate-internal
```

All Sultanate containers (Vizier, provinces) attach to this network. Janissary bridges between `sultanate-internal` and the host network.

### Container Naming

Pattern: `sultanate-{province_name}`

Examples:
- `sultanate-backend-refactor`
- `sultanate-efm-a1b2`

Province name is sanitized: lowercase, alphanumeric + hyphens only, max 48 chars.

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
| `/opt/sultanate/provinces/{id}/data` | `/opt/data` | rw | HERMES_HOME — sessions, workspace, config |
| `/opt/sultanate/ca/sultanate-ca.crt` | `/usr/local/share/ca-certificates/sultanate-ca.crt` | ro | Sultanate CA cert for HTTPS MITM trust |

### Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `HTTP_PROXY` | `http://janissary:8080` | Route HTTP through Janissary |
| `HTTPS_PROXY` | `http://janissary:8080` | Route HTTPS through Janissary |
| `NO_PROXY` | `divan` | Bypass proxy for Divan (if province needs direct access — reserved for future use) |
| `HERMES_HOME` | `/opt/data` | Hermes data directory |

### Docker Commands Reference

**Create:**
```bash
docker create \
  --name sultanate-{name} \
  --network sultanate-internal \
  -e HTTP_PROXY=http://janissary:8080 \
  -e HTTPS_PROXY=http://janissary:8080 \
  -e NO_PROXY=divan \
  -e HERMES_HOME=/opt/data \
  -v /opt/sultanate/provinces/{id}/data:/opt/data \
  -v /opt/sultanate/ca/sultanate-ca.crt:/usr/local/share/ca-certificates/sultanate-ca.crt:ro \
  nousresearch/hermes-agent:{tag}
```

**Start:**
```bash
docker start sultanate-{name}
```

**Exec (bootstrap):**
```bash
docker exec sultanate-{name} bash -c "update-ca-certificates"
docker exec sultanate-{name} bash -c "git config --global http.proxy \$HTTP_PROXY"
docker exec sultanate-{name} bash -c "git config --global https.proxy \$HTTPS_PROXY"
```

**Exec (clone):**
```bash
docker exec sultanate-{name} bash -c \
  "git clone https://github.com/{owner}/{repo}.git /opt/data/workspace --branch {branch}"
```

**Exec (write files):**
```bash
echo '<rendered content>' | docker exec -i sultanate-{name} tee /opt/data/SOUL.md > /dev/null
echo '<rendered content>' | docker exec -i sultanate-{name} tee /opt/data/workspace/AGENTS.md > /dev/null
echo '<rendered content>' | docker exec -i sultanate-{name} tee /opt/data/config.yaml > /dev/null
```

**Exec (start Hermes):**
```bash
docker exec -d sultanate-{name} bash -c "hermes gateway"
```

**Stop:**
```bash
docker stop sultanate-{name}
```

**Destroy:**
```bash
docker rm -f sultanate-{name}
```

**Logs:**
```bash
docker logs sultanate-{name} --tail 100
docker logs sultanate-{name} --follow
```

**Inspect IP:**
```bash
docker inspect -f '{{.NetworkSettings.Networks.sultanate-internal.IPAddress}}' sultanate-{name}
```

---

## 7. Appeal Relay

Vizier polls Divan for pending appeals and relays them to Sultan via Telegram.

### Polling Loop

```python
import time

POLL_INTERVAL = 10  # seconds

def appeal_relay_loop(divan: DivanClient) -> None:
    """Poll Divan for pending appeals, relay to Sultan."""
    while True:
        try:
            response = divan.get("/appeals", params={"status": "pending"})
            appeals = response["data"]
            for appeal in appeals:
                relay_appeal_to_sultan(appeal)
        except Exception as e:
            # Log error, continue polling
            click.echo(f"Appeal poll error: {e}", err=True)
        time.sleep(POLL_INTERVAL)
```

### Message Format

For each pending appeal, Vizier sends Sultan a Telegram message:

```
🔒 Appeal from province {province_name} ({province_id})

URL: {method} {url}
Reason: {justification}

Reply: approve / deny / whitelist
```

### Sultan Response Handling

| Sultan replies | Vizier action |
|---------------|---------------|
| `approve` | `PATCH /appeals/{id}` with `{"status": "approved", "decision": "one-time"}` |
| `deny` | `PATCH /appeals/{id}` with `{"status": "denied"}` |
| `whitelist` | Vizier tells Sultan: "Please tell Sentinel to whitelist {domain} for {province_name}." Vizier cannot write grants or modify whitelists post-creation — that is Sentinel's role. |

### API Calls

**Poll:**
```
GET /appeals?status=pending
Authorization: Bearer {DIVAN_KEY_VIZIER}
```

**Resolve (approve one-time):**
```
PATCH /appeals/{id}
Authorization: Bearer {DIVAN_KEY_VIZIER}
Content-Type: application/json

{"status": "approved", "decision": "one-time"}
```

**Resolve (deny):**
```
PATCH /appeals/{id}
Authorization: Bearer {DIVAN_KEY_VIZIER}
Content-Type: application/json

{"status": "denied"}
```

---

## 8. Divan Client Module

File: `vizier/divan.py`

HTTP client using `httpx` for all Divan API communication.

### Configuration

| Env Var | Required | Description |
|---------|----------|-------------|
| `DIVAN_URL` | yes | Divan base URL (e.g. `http://divan:8600`) |
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

    def __init__(self, base_url: str | None = None, api_key: str | None = None) -> None:
        self.base_url = base_url or os.environ["DIVAN_URL"]
        self.api_key = api_key or os.environ["DIVAN_KEY_VIZIER"]
        self._client = httpx.Client(
            base_url=self.base_url,
            headers={"Authorization": f"Bearer {self.api_key}"},
            timeout=30.0,
        )

    def _request(self, method: str, path: str, **kwargs: Any) -> dict:
        """Make an HTTP request with retry logic.

        :raises httpx.HTTPStatusError: On 4xx/5xx responses (after retries for connection errors).
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
        """HTTP GET."""
        return self._request("GET", path, params=params)

    def post(self, path: str, json: dict | None = None) -> dict:
        """HTTP POST."""
        return self._request("POST", path, json=json)

    def put(self, path: str, json: dict | None = None) -> dict:
        """HTTP PUT."""
        return self._request("PUT", path, json=json)

    def patch(self, path: str, json: dict | None = None) -> dict:
        """HTTP PATCH."""
        return self._request("PATCH", path, json=json)

    def delete(self, path: str, params: dict | None = None) -> dict:
        """HTTP DELETE."""
        return self._request("DELETE", path, params=params)

    def health_check(self) -> bool:
        """Check if Divan is reachable."""
        try:
            resp = self._client.get("/health")
            return resp.status_code == 200
        except httpx.ConnectError:
            return False

    def wait_for_divan(self, timeout: float = 60.0) -> None:
        """Block until Divan is reachable or timeout expires.

        :raises TimeoutError: If Divan is not reachable within timeout.
        """
        start = time.monotonic()
        while time.monotonic() - start < timeout:
            if self.health_check():
                return
            time.sleep(2.0)
        raise TimeoutError(f"Divan not reachable at {self.base_url} after {timeout}s")
```

---

## 9. Hermes Agent Wrapper

Vizier itself runs as a Hermes agent on the upstream `nousresearch/hermes-agent` image. It uses the terminal tool to execute `vizier` CLI commands and polls Divan for appeals.

### Deployment Path

Vizier's Hermes configuration lives at `/opt/sultanate/vizier/` (mounted as `HERMES_HOME`).

```
/opt/sultanate/vizier/
  SOUL.md
  config.yaml
  workspace/          # contains the vizier Python package (pip-installed)
```

### SOUL.md

```markdown
You are Vizier, the Grand Vizier of the Sultanate. You manage the realm on
behalf of the Sultan.

Your responsibilities:
- Create, start, stop, and destroy provinces on Sultan's command
- Report province status and health
- Relay appeals from provinces to Sultan for decision
- Execute realm management tasks via the vizier CLI

You are precise and efficient. You execute Sultan's commands using the
terminal tool to run vizier CLI commands. You do not improvise security
policy -- if something is blocked, you relay the appeal to Sultan and wait
for a decision.

You never attempt to read secrets, modify network rules, or bypass security.
If an agent needs access to something, you tell Sultan to instruct Sentinel.

Province management commands:
- vizier create <firman> --berat <berat> --repo <repo> [--name <name>]
- vizier list [--status <status>]
- vizier status <province>
- vizier stop <province>
- vizier start <province>
- vizier destroy <province>
- vizier logs <province> [--follow] [--tail <n>]
```

### config.yaml

```yaml
model:
  default: "anthropic/claude-sonnet-4-20250514"
  provider: openrouter

terminal:
  backend: local
  cwd: "/opt/sultanate/vizier/workspace"
  timeout: 120

tools:
  - terminal

mcp_servers:
  janissary_security:
    command: "npx"
    args: ["-y", "@sultanate/janissary-mcp"]
    env:
      JANISSARY_URL: "http://janissary:8081"
```

**Notes:**
- Only `terminal` tool enabled — Vizier interacts with the system exclusively through `vizier` CLI commands.
- Janissary MCP server is included so Vizier can appeal blocks on its own traffic.
- No `web`, `browser`, `file`, or other tools — Vizier does not browse or edit files directly.

---

## 10. Deployment

### Docker Run (Vizier Container)

```bash
docker run -d \
  --name sultanate-vizier \
  --network sultanate-internal \
  --restart unless-stopped \
  -e HTTP_PROXY=http://janissary:8080 \
  -e HTTPS_PROXY=http://janissary:8080 \
  -e NO_PROXY=divan \
  -e HERMES_HOME=/opt/sultanate/vizier \
  -e DIVAN_URL=http://divan:8600 \
  -e DIVAN_KEY_VIZIER={vizier_api_key} \
  -v /opt/sultanate/vizier:/opt/sultanate/vizier \
  -v /opt/sultanate/firmans:/opt/sultanate/firmans:ro \
  -v /opt/sultanate/berats:/opt/sultanate/berats:ro \
  -v /opt/sultanate/provinces:/opt/sultanate/provinces \
  -v /opt/sultanate/ca/sultanate-ca.crt:/usr/local/share/ca-certificates/sultanate-ca.crt:ro \
  -v /var/run/docker.sock:/var/run/docker.sock \
  nousresearch/hermes-agent:v0.6.0
```

### Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `HTTP_PROXY` | `http://janissary:8080` | All Vizier egress routes through Janissary |
| `HTTPS_PROXY` | `http://janissary:8080` | HTTPS through Janissary (MITM with Sultanate CA) |
| `NO_PROXY` | `divan` | Bypass proxy for Divan (internal service) |
| `HERMES_HOME` | `/opt/sultanate/vizier` | Hermes agent data directory |
| `DIVAN_URL` | `http://divan:8600` | Divan API base URL |
| `DIVAN_KEY_VIZIER` | `{generated at deploy}` | API key for Vizier role |

### Volumes

| Host Path | Container Path | Mode | Purpose |
|-----------|---------------|------|---------|
| `/opt/sultanate/vizier` | `/opt/sultanate/vizier` | rw | Vizier's HERMES_HOME (SOUL.md, config.yaml) |
| `/opt/sultanate/firmans` | `/opt/sultanate/firmans` | ro | Firman manifests |
| `/opt/sultanate/berats` | `/opt/sultanate/berats` | ro | Berat manifests + templates |
| `/opt/sultanate/provinces` | `/opt/sultanate/provinces` | rw | Province data volumes |
| `/opt/sultanate/ca/sultanate-ca.crt` | `/usr/local/share/ca-certificates/sultanate-ca.crt` | ro | CA cert for HTTPS trust |
| `/var/run/docker.sock` | `/var/run/docker.sock` | rw | Docker socket for container management |

### Network

Vizier is on `sultanate-internal`. All egress (including Telegram API calls from Hermes) routes through Janissary via `HTTP_PROXY`. The Docker socket is mounted so Vizier can create/manage province containers on the host.

### Startup Order

Vizier starts after Divan and Janissary are ready (see SULTANATE_MVP.md §Startup Order):

```
1. Divan        (shared state)
2. Janissary    (proxy)
3. Sentinel     (secrets)
4. Vizier       (this component)
5. Provinces    (on demand)
```

Vizier's `DivanClient.wait_for_divan()` blocks until Divan returns `200` on `/health`.

---

## Appendix: Module Map

```
vizier/
  __init__.py       Package init
  __main__.py       Entry point for python -m vizier
  cli.py            Click CLI commands
  config.py         Configuration (env vars, paths)
  divan.py          DivanClient (httpx)
  docker.py         Docker subprocess wrapper
  models.py         Pydantic models (Province, Firman, Berat)
  province.py       Province lifecycle (create, start, stop, destroy)
  templates.py      Template rendering (str.replace, YAML validation)
  appeals.py        Appeal relay polling loop
```
