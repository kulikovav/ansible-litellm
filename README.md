# LiteLLM on Podman

Ansible playbook that installs the latest LiteLLM proxy and its dependencies on
`server1`, served at `https://fqdn`.

## Components

| Component  | Image                                            | Purpose                             |
| ---------- | ------------------------------------------------ | ----------------------------------- |
| LiteLLM    | `ghcr.io/berriai/litellm-database:v1.103.0-rc.1` | OpenAI-compatible proxy             |
| Headroom   | `ghcr.io/headroomlabs-ai/headroom:0.35.0`        | Prompt compression sidecar (opt-in) |
| PostgreSQL | `postgres:16-alpine`                             | Keys, users, spend logs             |
| Redis      | `redis:7-alpine`                                 | Response cache + auth cache (AOF)   |
| Nginx      | host `nginx` (Rocky Linux)                       | TLS termination + reverse proxy     |
| acme.sh    | `neilpang/acme.sh` (installed via script)        | TLS cert (HTTP-01 webroot)          |

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
   `sk-` master key, and provider API keys (`CLINE_API_KEY`,
   `COMMANDCODE_API_KEY`, `OPENROUTER_API_KEY`, `EXA_API_KEY`,
   `FIRECRAWL_API_KEY`, `HEADROOM_API_KEY`, `CONTEXT7_API_KEY`,
   `OLLAMA_API_KEY`, `RESEND_API_KEY`).

2. Run the playbook:

   ```bash
   ansible-playbook playbooks/site.yml --ask-vault-pass
   ```

3. Verify:

   ```bash
   curl https://fqdn/health/liveliness
   ```

## Notes

- `litellm_image` is pinned to a release tag (`litellm-database:v1.103.0-rc.1`) so
  upgrades and rollbacks are explicit. Bump that variable to move version, and
  confirm the running build with
  `podman exec litellm python -c "import importlib.metadata as m; print(m.version('litellm'))"`.
- Headroom is opt-in. Set `headroom_enabled: true` to render the sidecar service
  and LiteLLM's `headroom-compression` guardrail; the default (`false`) starts
  only postgres, redis and litellm. When enabled, the guardrail calls
  `http://headroom:8787/v1/compress` (override `litellm_headroom_api_base`) and
  Headroom's `--backend litellm-openai` forwards passthrough traffic to
  `headroom_target_api_url`.
- LiteLLM logs `This model isn't mapped yet` for each Ollama deployment, because
  its model registry holds no entry for these ids. The router catches the error,
  so the requests still succeed; the pre-call context-window check is skipped for
  those deployments. `model_info.base_model` does not help (azure only), and
  `ollama_chat/library/<model>` is accepted upstream but does not change the
  lookup. `enable_pre_call_checks: false` silences it at the cost of the check
  for every provider.
- HTTP-01 requires `fqdn` to resolve to this host and port 80
  to be publicly reachable. The playbook installs a self-signed placeholder cert,
  then acme.sh issues the real cert and reloads the host nginx.
- The `/v1` API is not IP-restricted: over HTTPS it answers from any source IP
  and the master key is the only gate. The site root stays restricted to
  `litellm_allowed_ips` (default `127.0.0.1`), where other IPs get `403`. The
  `/.well-known/acme-challenge/` path stays open for certificate validation.
