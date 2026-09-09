# b08x.llmops dify Role

Deploy self-hosted [Dify](https://dify.ai) — an open-source LLM application
platform — via Docker Compose or Podman Compose.

## Requirements

- Docker or Podman installed on the target host
- `community.docker` collection (for Docker Compose deployments)
- `ansible.posix` collection (for firewalld management)

## Role Variables

All variables are defined in `defaults/main.yml` and use the `dify_` prefix
(Tier 2 role scope). The `user.home` variable is consumed from `group_vars`
(Tier 1 host scope).

### Core

| Variable | Default | Description |
|---|---|---|
| `dify_container_runtime` | `docker` | Container runtime: `docker` or `podman` |
| `dify_deploy_dir` | `{{ user.home }}/dify` | Deployment directory for compose files and data |
| `dify_project_name` | `dify` | Docker Compose project name |
| `dify_image_tag` | `1.17.0` | Dify API, web, and agent image tag |
| `dify_bind_localhost` | `true` | Bind ports to 127.0.0.1 only |
| `dify_force_recreate` | `false` | Force container recreation on every run |

### Ports and Firewall

| Variable | Default | Description |
|---|---|---|
| `dify_nginx_port` | `80` | Host port for Nginx HTTP |
| `dify_nginx_ssl_port` | `443` | Host port for Nginx HTTPS |
| `dify_plugin_debugging_port` | `5003` | Host port for plugin debugging |
| `dify_firewall_ports` | `[nginx ports]` | Firewall ports to open when `dify_bind_localhost` is false |

### Database and Redis

| Variable | Default | Description |
|---|---|---|
| `dify_db_username` | `postgres` | PostgreSQL user |
| `dify_db_password` | `difyai123456` | PostgreSQL password |
| `dify_db_database` | `dify` | PostgreSQL database name |
| `dify_db_plugin_database` | `dify_plugin` | Plugin daemon database name |
| `dify_redis_password` | `difyai123456` | Redis password |

### Security

| Variable | Default | Description |
|---|---|---|
| `dify_secret_key` | `""` | Leave empty to auto-generate |
| `dify_init_password` | `""` | Initial admin password |
| `dify_sandbox_api_key` | `dify-sandbox` | Sandbox API key |
| `dify_plugin_daemon_key` | (see defaults) | Plugin daemon server key |
| `dify_agent_api_token` | (see defaults) | Agent backend API token |

### Vector Store

| Variable | Default | Description |
|---|---|---|
| `dify_vector_store` | `weaviate` | Vector store backend |
| `dify_weaviate_api_key` | (see defaults) | Weaviate API key |

### Backup/Restore

| Variable | Default | Description |
|---|---|---|
| `dify_backup_action` | `none` | `none`, `backup`, or `restore` |
| `dify_backup_dir` | `{{ dify_deploy_dir }}/backups` | Backup directory |
| `dify_backup_file` | `""` | Restore file path (for restore) |
| `dify_backup_format` | `custom` | `custom` (-Fc) or `plain` (-Fp) |
| `dify_backup_retention_days` | `7` | Days to retain backups |

## Dependencies

No role dependencies.

## Example Playbook

### Docker Compose

```yaml
- name: Deploy Dify
  hosts: workstations
  become: false
  vars:
    dify_deploy_dir: "{{ user.home }}/LLMOS/dify"
  roles:
    - role: b08x.llmops.dify
      tags: ["dify", "llmops"]
```

### Podman

```yaml
- name: Deploy Dify (Podman) on tinybot
  hosts: tinybot
  become: false
  gather_facts: true
  vars:
    dify_container_runtime: podman
    dify_deploy_dir: "{{ user.home }}/LLMOS/dify"
    dify_bind_localhost: false
  roles:
    - role: b08x.llmops.dify
      tags: ["dify", "llmops"]
```

## Architecture

```
dify role
  tasks/main.yml
    debug entry → verify runtime → create dirs → render .env
    include docker.yml  (when runtime == docker)
      render docker-compose.yml.j2 → docker compose pull → docker_compose_v2
    include podman.yml   (when runtime == podman)
      render docker-compose.yml.j2 → podman compose down → pull → up -d
    firewalld (when bind_localhost == false)
    include backup.yml   (when backup_action != none)
  handlers/main.yml
    Restart Dify stack (docker_compose_v2 recreate, docker-only)
```

Services deployed:
- nginx (entry point, ports 80/443)
- api (Dify API server)
- api_websocket (workflow collaboration, optional via profile)
- worker (Celery worker)
- worker_beat (Celery beat scheduler)
- web (frontend)
- db_postgres (PostgreSQL 15)
- redis (Redis 6)
- sandbox (code execution)
- plugin_daemon (plugin management)
- agent_backend (agent runtime backend)
- local_sandbox (agent shell workspaces)
- ssrf_proxy (Squid SSRF proxy for sandbox)
- agent_ssrf_proxy (Squid SSRF proxy for agent sandbox)
- weaviate (vector store, default)

## Role Idempotency

True — the role is idempotent. Docker Compose `state: present` and
`podman compose up -d` are no-ops when containers are already running
with the current configuration.

## Role Atomicity

True — the role deploys the complete Dify stack in a single pass.

## Roll-back capabilities

Set `dify_force_recreate: false` (default) and revert the variable
changes. The compose stack will converge to the previous state on next
run. For full teardown, run `docker compose down` or `podman compose down`
in the deploy directory.

## License

GPL-2.0-or-later

## Author Information

b08x
