# b08x.llmops.langfuse

Deploys self-hosted [Langfuse v4](https://langfuse.com) — the open-source LLM observability platform — via Docker Compose with PostgreSQL, ClickHouse, Redis, and MinIO.

## Requirements

- Docker Engine with Docker Compose plugin installed on the target host
- The `community.docker` Ansible collection for the `docker_compose_v2` module

## Role Variables

All variables are defined in `defaults/main.yml` with the `langfuse_` prefix. Key variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `langfuse_deploy_dir` | `{{ user.home }}/langfuse` | Deployment directory for compose and env files |
| `langfuse_project_name` | `langfuse` | Docker Compose project name |
| `langfuse_image_tag` | `4` | Langfuse web/worker image tag |
| `langfuse_web_port` | `3000` | Host port for the Langfuse web UI |
| `langfuse_bind_localhost` | `true` | Bind infrastructure ports to 127.0.0.1 only |
| `langfuse_nextauth_url` | `http://localhost:3000` | Public URL for NextAuth |
| `langfuse_nextauth_secret` | `mysecret` | NextAuth JWT secret |
| `langfuse_encryption_key` | `0000...` | 32-byte hex key (generate via `openssl rand -hex 32`) |
| `langfuse_postgres_password` | `postgres` | PostgreSQL password |
| `langfuse_clickhouse_password` | `clickhouse` | ClickHouse password |
| `langfuse_redis_auth` | `myredissecret` | Redis password |
| `langfuse_minio_root_password` | `miniosecret` | MinIO root password |

Override secrets in group/host vars or via `--extra-vars`.

## Dependencies

None.

## Example Playbook

```yaml
- name: Deploy Langfuse
  hosts: workstations
  become: true
  roles:
    - role: b08x.llmops.langfuse
      vars:
        langfuse_nextauth_url: "http://langfuse.example.com:3000"
        langfuse_nextauth_secret: "{{ vault_langfuse_nextauth_secret }}"
        langfuse_encryption_key: "{{ vault_langfuse_encryption_key }}"
        langfuse_postgres_password: "{{ vault_langfuse_postgres_password }}"
        langfuse_clickhouse_password: "{{ vault_langfuse_clickhouse_password }}"
        langfuse_redis_auth: "{{ vault_langfuse_redis_auth }}"
        langfuse_minio_root_password: "{{ vault_langfuse_minio_root_password }}"
```

## Role Idempotency

True — the `docker_compose_v2` module is idempotent; re-running the role will reconcile the stack to the desired state.

## Role Atomicity

True — configuration templates and the compose stack are deployed together. If template changes trigger a handler, the entire stack is recreated.

## Roll-back capabilities

Removing the role or reverting the compose/env templates and re-running will restore the previous state. Docker volumes for PostgreSQL, ClickHouse, MinIO, and Redis persist independently under named volumes.

## Argument Specification

See `meta/argument_specs.yml` for the full argument specification.

## License

GPL-2.0-or-later

## Author Information

b08x
