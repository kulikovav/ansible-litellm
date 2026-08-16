# LiteLLM on Podman (uk1)

Ansible playbook that installs the latest LiteLLM proxy and its dependencies on
`server1`, served at `https://fqdn`.

## Components

| Component  | Image                                     | Purpose                           |
| ---------- | ----------------------------------------- | --------------------------------- |
| LiteLLM    | `ghcr.io/berriai/litellm-database:latest` | OpenAI-compatible proxy           |
| Headroom   | `ghcr.io/headroomlabs-ai/headroom:0.35.0` | Prompt compression sidecar        |
| PostgreSQL | `postgres:16-alpine`                      | Keys, users, spend logs           |
| Redis      | `redis:7-alpine`                          | Response cache + auth cache (AOF) |
| Nginx      | host `nginx` (Rocky Linux)                | TLS termination + reverse proxy   |
| acme.sh    | `neilpang/acme.sh` (installed via script) | TLS cert (HTTP-01 webroot)        |

The containers run rootless under the `opc` user via `podman-compose`, managed by
a systemd user unit (`litellm-stack.service`). The host already runs nginx
(serving other sites), so LiteLLM is added as a drop-in server block
(`/etc/nginx/conf.d/litellm.conf`) that proxies to the LiteLLM container on
`127.0.0.1:4000`.

## Layout

```
ansible.cfg
inventory/hosts.yml
inventory/group_vars/all/vars.yml            # non-secret configuration
inventory/group_vars/all/secrets.yml.example # template for the secrets file
playbooks/site.yml                           # entry point
roles/
  podman/   # install podman + podman-compose, firewall, sysctl, lingering
  acme/     # acme.sh + HTTP-01 webroot cert issuance
  litellm/  # compose stack + config.yaml + host nginx + systemd unit
```

## Setup

1. Create the secrets file and encrypt it:

   ```bash
   cp inventory/group_vars/all/secrets.yml.example inventory/group_vars/all/secrets.yml
   ansible-vault encrypt inventory/group_vars/all/secrets.yml
   ```

   Fill in real values before encrypting: PostgreSQL/Redis passwords, the
   `sk-` master key, and provider API keys (`OPENCODE_API_KEY`, `CLINE_API_KEY`,
   `NANOGPT_API_KEY`, `OPENROUTER_API_KEY`, `EXA_API_KEY`, `FIRECRAWL_API_KEY`,
   `HEADROOM_API_KEY`, `CONTEXT7_API_KEY`, `RESEND_API_KEY`).

2. Run the playbook:

   ```bash
   ansible-playbook playbooks/site.yml --ask-vault-pass
   ```

3. Verify:

   ```bash
   curl https://fqdn/health/liveliness
   ```

## Notes

- `litellm_image` defaults to the rolling `latest` tag. Pin it to a `vX.Y.Z`
  release tag (e.g. `ghcr.io/berriai/litellm-database:v1.96.2`) for
  deterministic rollbacks.
- Headroom runs as a sidecar in the compose network. LiteLLM's
  `headroom-compression` guardrail calls `http://headroom:8787/v1/compress`
  (override `litellm_headroom_api_base`). Headroom's `--backend litellm-openai`
  forwards passthrough traffic to `headroom_target_api_url`.
- HTTP-01 requires `fqdn` to resolve to this host and port 80
  to be publicly reachable. The playbook installs a self-signed placeholder cert,
  then acme.sh issues the real cert and reloads the host nginx.
- The public endpoint is IP-restricted to `litellm_allowed_ips` (default
  `127.0.0.1`); all other IPs get `403`. The `/.well-known/acme-challenge/`
  path stays open for certificate validation.
