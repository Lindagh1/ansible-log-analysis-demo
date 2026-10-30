# Sandbox deployment automation

Idempotent Ansible automation to deploy the **Ansible Log Monitor** AI
quickstart (https://docs.redhat.com/en/learn/ai-quickstarts/rh-ansible-log-analysis)
onto any temporary OpenShift sandbox, so the demo can be rebuilt reliably on
a brand new sandbox later without redoing manual steps.

## What it does

1. Logs into the OpenShift cluster (or reuses your current `oc login` session).
2. Creates/labels the target namespace (no-op if it already exists).
3. Auto-detects a Ready vLLM model served via OpenShift AI / KServe on the
   cluster (so you don't have to hardcode a model name/namespace that will
   differ between sandboxes) — or uses the LLM settings you pass explicitly.
4. Deploys the Helm chart with `helm upgrade --install` (idempotent: works
   the same on a fresh sandbox or to reconcile an existing one).
5. Waits for the core workloads (`ui`, `backend`, `aap-mock`) to roll out.
6. Lists all the application Routes.
7. (Re)starts the bundled mock-log replay so the demo has fresh alerts
   flowing when you present it.

## Usage

### First time on a new sandbox

```bash
ansible-playbook sandbox-deploy/deploy.yml \
  -e ocp_api_url=https://api.<CLUSTER_DOMAIN>:6443 \
  -e ocp_token=sha256~xxxxxxxxxxxx \
  -e target_namespace=ansible-logs-monitor
```

### If you're already `oc login`-ed in this shell

```bash
ansible-playbook sandbox-deploy/deploy.yml -e target_namespace=ansible-logs-monitor
```

### Re-running (idempotent)

Running either command again — on the same sandbox or a brand new one — is
safe. It will reconcile the existing release instead of failing.

### Overriding the LLM explicitly

If auto-detection doesn't find the right model (e.g. several are deployed,
or it lives outside OpenShift AI), set it explicitly:

```bash
ansible-playbook sandbox-deploy/deploy.yml \
  -e llm_api_endpoint=https://your-endpoint/v1 \
  -e llm_model=your-model-name \
  -e llm_api_token=your-token
```

### Accessing the deployed apps

After a successful run, the playbook prints all application Routes. Grafana
credentials are `admin` / `alm_password`. Phoenix needs no login at all —
auth is intentionally disabled via the single-node overrides below (the
upstream chart otherwise pulls `arizephoenix/phoenix:latest`, which enables
auth by default with no secret configured, silently dropping all trace
exports from the backend). See `docs/DEMO_GUIDE.md` for a full walkthrough.

### Tearing down

```bash
ansible-playbook sandbox-deploy/teardown.yml -e target_namespace=ansible-logs-monitor
```

## Files

- `deploy.yml` — main idempotent deployment playbook.
- `teardown.yml` — removes the Helm release and PVCs.
- `group_vars/all.yml` — all tunable defaults (non-secret).
- `templates/llm-values.yaml.j2` — renders the LLM secret values passed to Helm.

No secrets are ever written into this repo: the LLM token is passed as an
extra-var and only ever lands in a `0600` temp file under `/tmp`, removed
right after the Helm call.
