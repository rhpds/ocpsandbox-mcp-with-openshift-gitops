# Converting a Lab to the Sandbox API Shared Cluster Pattern

**Full documentation:** https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/

This is a step-by-step conversion guide based on how we converted the MCP with OpenShift lab.
Start by opening your existing `common.yaml` alongside this guide and follow each step.

---

## Step 0 — Open Your Existing AgV and Identify What You Have

Before changing anything, look at your existing `common.yaml` and answer these questions:

**1. What is your `config:` value?**
- `config: openshift-workloads` or `config: openshift-cluster` → you are provisioning a dedicated cluster per order. This is what we are replacing.
- `config: namespace` → you are already on the right config. Skip Step 1.

**2. Do you have a `components:` block?**
```yaml
components:
  - name: openshift
    item: agd-v2/ocp-cluster-cnv-pools
```
If yes → your catalog item orders a full CNV cluster. This is the expensive pattern we are moving away from.

**3. Do you have a `clusters:` block?**
```yaml
clusters:
  - default:
      api_url: "{{ openshift_api_url }}"
      api_token: "{{ openshift_api_key }}"
```
If yes → this goes away. Sandbox API injects these vars automatically.

**4. What are your `workloads:`?**
Write them down. Each one will need to be re-evaluated — some move to the cluster provisioner (infra layer), some stay as tenant workloads, some get replaced by new tenant roles.

---

## Step 1 — Change `config:` and Remove `clusters:`

This is the most fundamental change.

```yaml
# BEFORE
config: openshift-workloads
clusters:
  - default:
      api_url: "{{ openshift_api_url }}"
      api_token: "{{ openshift_api_key }}"

# AFTER
config: namespace
```

That is it. No `clusters:` block needed. Sandbox API injects the cluster connection
automatically via environment variables that `config: namespace` picks up.

---

## Step 2 — Replace `components:` with a `sandboxes:` Entry

Remove the `components:` block entirely. Add a `sandboxes:` entry under `__meta__`:

```yaml
# BEFORE — orders a full CNV cluster
components:
  - name: openshift
    item: agd-v2/ocp-cluster-cnv-pools
    parameter_values:
      cluster_size: sno
    propagate_provision_data:
      - name: openshift_api_url
        var: openshift_api_url
      - name: openshift_cluster_admin_token
        var: openshift_api_key

# AFTER — picks a pre-provisioned shared cluster from the pool
__meta__:
  sandbox_api:
    actions:
      destroy:
        catch_all: false    # required — prevents deleting other tenants' LiteMaaS keys
  sandboxes:
  - kind: OcpSandbox
    alias: cluster
    namespace_suffix: user
    cloud_selector:
      cloud: cnv-dedicated-shared     # required — cluster type
      # The additional tag key and value depend on how the cluster is registered
      # in the Sandbox API pool. Ask your admin what tags your cluster has.
      # Example: demo: mcp-with-openshift  OR  lab: my-lab  etc.
      purpose: prod                   # required
    quota:
      limits.cpu: "4"
      requests.cpu: "4"
      limits.memory: 8Gi
      requests.memory: 8Gi
      requests.storage: 50Gi
```

**How does `cloud_selector` work?**
Every cluster in the pool has annotation tags. Your `cloud_selector` is a filter — the cluster
must have ALL your tags. Ask your admin which tags your cluster pool uses for the `demo:` field.

---

## Step 3 — Add `scm_ref:` to the Deployer

Without this, Babylon does not know which AgnosticD branch to run. Order fails immediately.

```yaml
__meta__:
  deployer:
    scm_url: https://github.com/agnosticd/agnosticd-v2
    scm_ref: main                                        # ← add this
    execution_environment:
      image: quay.io/agnosticd/ee-multicloud:chained-2025-12-17
```

---

## Step 4 — Variables You Now Have From Sandbox API

After the cluster is scheduled, these are injected automatically into all your roles.
You do not declare them. Just use them:

| Variable | Description |
|---|---|
| `sandbox_openshift_api_url` | Cluster API URL |
| **`sandbox_openshift_ingress_domain`** | **Wildcard apps domain — use this for all service URLs** |
| `sandbox_openshift_console_url` | Web console URL |
| `cluster_admin_agnosticd_sa_token` | Cluster-admin SA token |
| `sandbox_openshift_namespace` | Primary namespace (`sandbox-{guid}-{suffix}`) |

**If any role uses `openshift_api_key` (common in older roles), add this mapping:**
```yaml
# Required if any role expects openshift_api_key (old config: openshift-workloads name)
openshift_api_key: "{{ cluster_admin_agnosticd_sa_token }}"
openshift_api_url: "{{ sandbox_openshift_api_url }}"
```

**Build all service URLs from `sandbox_openshift_ingress_domain`:**
```yaml
my_app_url: "https://myapp-{{ sandbox_openshift_namespace }}.{{ sandbox_openshift_ingress_domain }}"
```

---

## Step 5 — Sort Your Workloads Into Two Buckets

This is the key architectural decision. Look at each workload in your existing list and decide:

**Bucket A — Cluster Provisioner (runs once per cluster, NOT in your catalog item AgV)**
These are cluster-wide operators and shared services. They belong in the cluster provisioner
playbook, not in your per-order `workloads:`. If the cluster already has these installed, remove
them from your `workloads:` list entirely.

Common ones to move out:
- `ocp4_workload_openshift_gitops` — ArgoCD
- `ocp4_workload_pipelines` — Tekton
- `ocp4_workload_gitea_operator` — Gitea operator
- `ocp4_workload_authentication` / `ocp4_workload_authentication_htpasswd` — cluster auth
- `ocp4_workload_toolhive` — ToolHive

**Bucket B — Tenant Workloads (per order, stays in your AgV)**
These create per-user resources. They stay in your `workloads:` list.

In our MCP migration, we replaced the old monolithic `ocp4_workload_mcp_user` role
(which did everything for all users) with individual single-responsibility roles:

| Old role | New replacement | What it does |
|---|---|---|
| `ocp4_workload_authentication_htpasswd` | `ocp4_workload_tenant_keycloak_user` | Creates one RHBK user per order |
| (namespace creation was in `ocp4_workload_mcp_user`) | `ocp4_workload_tenant_namespace` | Creates per-user OCP namespaces |
| `ocp4_workload_gitea_operator` (per order) | `ocp4_workload_tenant_gitea` | Deploys per-tenant Gitea instance |
| `ocp4_workload_litellm_virtual_keys` | same role | Creates LiteMaaS virtual key |
| (hardcoded ArgoCD ApplicationSets in mcp_user) | `ocp4_workload_gitops_bootstrap` | Creates ArgoCD app-of-apps |
| `ocp4_workload_showroom` | same role | Showroom tabs |

---

## Step 6 — Write the New `workloads:` List

For a typical shared cluster lab converting from the old pattern:

```yaml
# Add these collections
requirements_content:
  collections:
  - name: https://github.com/agnosticd/namespaced_workloads.git
    type: git
    version: tenant-roles              # use main once PR #12 merges
  - name: https://github.com/agnosticd/core_workloads.git
    type: git
    version: gitops-bootstrap-userinfo # use main once PR #62 merges
  - name: https://github.com/rhpds/rhpds.litellm_virtual_keys.git
    type: git
    version: main

# New workloads list
workloads:
- agnosticd.namespaced_workloads.ocp4_workload_tenant_keycloak_user    # 1. Create RHBK user
- agnosticd.namespaced_workloads.ocp4_workload_tenant_namespace         # 2. Create namespaces
- agnosticd.namespaced_workloads.ocp4_workload_tenant_gitea             # 3. Gitea per tenant
- rhpds.litellm_virtual_keys.ocp4_workload_litellm_virtual_keys         # 4. LiteMaaS key
- agnosticd.core_workloads.ocp4_workload_gitops_bootstrap               # 5. ArgoCD app-of-apps
- agnosticd.showroom.ocp4_workload_showroom                             # 6. Showroom
# NOTE: ocp4_workload_ocp_console_embed belongs in the cluster provisioner (runs once).
# Do NOT add it to tenant workloads — it triggers a router rollout on every order.

# Explicit destroy order — runs in THIS order (not automatic reverse)
remove_workloads:
- rhpds.litellm_virtual_keys.ocp4_workload_litellm_virtual_keys
- agnosticd.core_workloads.ocp4_workload_gitops_bootstrap   # cascade deletes all ArgoCD apps
- agnosticd.namespaced_workloads.ocp4_workload_tenant_gitea
- agnosticd.namespaced_workloads.ocp4_workload_tenant_namespace
- agnosticd.namespaced_workloads.ocp4_workload_tenant_keycloak_user
```

---

## Step 7 — Configure Each Tenant Role

### RHBK User
```yaml
ocp4_workload_tenant_keycloak_username: "mylab-{{ guid }}"
common_password: "Lab{{ (guid | hash('sha256'))[:8] }}!"
ocp4_workload_tenant_keycloak_user_password: "{{ common_password }}"
# RHBK host is already on the cluster — just point to it:
ocp4_workload_tenant_keycloak_user_rhbk_host: "keycloak-keycloak.{{ sandbox_openshift_ingress_domain }}"
ocp4_workload_tenant_keycloak_user_realm: sso
```

### Namespaces
```yaml
ocp4_workload_tenant_namespace_prefix: "{{ ocp4_workload_tenant_keycloak_username }}"
ocp4_workload_tenant_namespace_username: "{{ ocp4_workload_tenant_keycloak_username }}"
ocp4_workload_tenant_namespace_suffixes:
- app          # creates app-mylab-{guid} namespace
- db           # creates db-mylab-{guid} namespace
```

### Gitea (per-tenant instance — operator already installed on cluster)
```yaml
ocp4_workload_tenant_gitea_namespace: "gitea-{{ ocp4_workload_tenant_keycloak_username }}"
ocp4_workload_tenant_gitea_username: "{{ ocp4_workload_tenant_keycloak_username }}"
ocp4_workload_tenant_gitea_password: "{{ common_password }}"
ocp4_workload_tenant_gitea_admin_password: "{{ common_password }}"
ocp4_workload_tenant_gitea_repositories:
- name: my-repo
  repo: https://github.com/myorg/my-gitops-repo
  private: false
```

### LiteMaaS Virtual Key
```yaml
ocp4_workload_litellm_virtual_keys_duration: "7d"
ocp4_workload_litellm_virtual_keys_models:
- qwen3-14b
- llama-scout-17b
ocp4_workload_litellm_virtual_keys_catch_all: false
# NOTE: catch_all ALSO must be set in __meta__.sandbox_api.actions.destroy
# (the role var alone is not enough — see Step 7b below)
```

### GitOps Bootstrap
```yaml
# repo_url is set automatically by ocp4_workload_tenant_gitea role
ocp4_workload_gitops_bootstrap_application_name: "bootstrap-tenant"
ocp4_workload_gitops_bootstrap_repo_path: "tenant/bootstrap"
ocp4_workload_gitops_bootstrap_repo_revision: main
ocp4_workload_gitops_bootstrap_helm_values:
  tenant:
    username: "{{ ocp4_workload_tenant_keycloak_username }}"
    password: "{{ common_password }}"
  litemaas:
    url: "{{ litellm_api_endpoint | default('') }}/v1"
    key: "{{ litellm_virtual_key | default('') }}"
    models: "{{ litellm_available_models | default([]) | join(',') }}"
```

---

## Step 7b — dev.yaml: Keep It Minimal

`dev.yaml` should only contain what differs from `common.yaml`. Do NOT duplicate collections,
workloads, role vars, or catalog metadata — any change would then need to be made in two places.

```yaml
---
purpose: development

ocp4_workload_showroom_antora_enable_dev_mode: "true"

__meta__:
  deployer:
    scm_ref: main
  sandbox_api:
    actions:
      destroy:
        catch_all: false
  sandboxes:
  - kind: OcpSandbox
    alias: cluster
    namespace_suffix: user
    cloud_selector:
      cloud: cnv-dedicated-shared
      demo: mcp-with-openshift
      purpose: development
    quota:
      limits.cpu: "4"
      requests.cpu: "4"
      limits.memory: 8Gi
      requests.memory: 8Gi
      requests.storage: 50Gi
```

Everything else (collections, workloads, role vars, catalog) is inherited from `common.yaml`.

---

## Step 8 — Showroom

Replace old component-based Showroom vars with sandbox API vars:

```yaml
# These two are required for showroom to connect to the cluster
openshift_api_url: "{{ sandbox_openshift_api_url }}"
openshift_cluster_admin_token: "{{ cluster_admin_agnosticd_sa_token }}"

ocp4_workload_showroom_namespace: "showroom-{{ ocp4_workload_tenant_keycloak_username }}"
ocp4_workload_showroom_content_git_repo: https://github.com/myorg/my-showroom
ocp4_workload_showroom_content_git_repo_ref: main
ocp4_workload_showroom_openshift_api_url: "{{ sandbox_openshift_api_url }}"
ocp4_workload_showroom_openshift_api_token: "{{ cluster_admin_agnosticd_sa_token }}"
```

---

## Step 9 — Test Destroy as Early as Possible

Every role must have `remove_workload.yml`. Test destroy early — do not wait until the end.

```bash
# Order, verify it works, then destroy
# Check ArgoCD shows all apps Synced/Healthy
# Then destroy and verify all namespaces, users, Gitea org, LiteMaaS key are gone
```

---

## Common Mistakes (Real Ones From Our Migration)

| Mistake | Symptom | Fix |
|---|---|---|
| `config: openshift-workloads` | Fails before provisioning starts | `config: namespace` |
| Missing `scm_ref:` | Fails before provisioning starts | Add `scm_ref: main` |
| `var: sandbox_user` on sandbox entry | Wrong variable names | Remove `var:` |
| `clusters:` block with `config: namespace` | Invalid config | Remove `clusters:` block |
| Showroom fails with "invalid value bearer" | OCP console integration role fails in `config: namespace` | Use `agnosticd.showroom.ocp4_workload_ocp_console_embed` and add `openshift_cluster_admin_token: "{{ cluster_admin_agnosticd_sa_token }}"` |
| Duplicate YAML key | yamllint error | Search and remove the duplicate |
| No `cluster_condition: same('alias')` on second namespace | Two namespaces on different clusters | Add `cluster_condition` |
| `catch_all: true` for LiteMaaS | Destroys all tenants' AI keys | Always `false` |
| `ocp4_workload_litellm_virtual_keys_catch_all: false` as a role var | Does not work — Sandbox API does not read it | Put `catch_all: false` under `__meta__.sandbox_api.actions.destroy` instead |
| Missing `cloud: cnv-dedicated-shared` in `cloud_selector` | Sandbox API finds no cluster | All three tags required: `cloud:`, `demo:`, `purpose:` |
| Catalog item in `summit-2026/` with shared cluster | Order fails — `summit-2026/account.yaml` sets `reservation: pgpu-event` which excludes shared clusters. Empty string override is not supported by AgV. | Move the catalog item to `agd_v2/` instead. The `summit-2026/` directory is for Summit event-specific clusters only. |
| `dev.yaml` duplicating collections and workloads from `common.yaml` | Maintenance nightmare — changes must be made in two places | `dev.yaml` should only contain `purpose: development` and `__meta__` overrides |

---

## References

| Resource | Link |
|---|---|
| Full docs | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/ |
| Scheduler-Only complete AgV | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/scheduler-only.html |
| Full Namespace API guide | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/namespace-api.html |
| Migration story (MCP lab) | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/migration-guide.html |
| GitOps architecture | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/gitops-pattern.html |
| tenant_keycloak_user role | https://github.com/agnosticd/namespaced_workloads/tree/tenant-roles/roles/ocp4_workload_tenant_keycloak_user |
| tenant_namespace role | https://github.com/agnosticd/namespaced_workloads/tree/tenant-roles/roles/ocp4_workload_tenant_namespace |
| tenant_gitea role | https://github.com/agnosticd/namespaced_workloads/tree/tenant-roles/roles/ocp4_workload_tenant_gitea |
| namespaced_workloads PR #12 | https://github.com/agnosticd/namespaced_workloads/pull/12 |
| core_workloads PR #62 | https://github.com/agnosticd/core_workloads/pull/62 |
