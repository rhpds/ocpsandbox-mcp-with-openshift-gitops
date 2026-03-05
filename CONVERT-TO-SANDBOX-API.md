# Converting a Lab to the Sandbox API Shared Cluster Pattern

**Full documentation:** https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/

This document is a quick reference for converting an existing lab to use the Sandbox API
scheduler-only pattern for shared cluster workshops (Summit 2026, Lightning Labs, booth demos).

---

## What You Are Doing

Instead of provisioning a dedicated CNV cluster per order (45–60 min), you run your lab on a
pre-provisioned shared cluster from the Sandbox API pool (2–3 min). The Sandbox API picks a
cluster and gives you credentials. Your Ansible roles do the rest.

**Three layers — understand this first:**

| Layer | Runs | What it does |
|---|---|---|
| **Infra** | Once per cluster | Installs operators (RHBK, Gitea, ArgoCD, Tekton, ToolHive). Done by cluster operator. |
| **Platform** | Once per cluster | Creates operator instances, enables monitoring. Done via ArgoCD (bootstrap-platform). |
| **Tenant** | Every order | Creates per-user resources. This is what you configure in AgV. |

You only write the **Tenant layer** in your `common.yaml`.

---

## Step 1 — Fix Your `common.yaml` (Mandatory Changes)

### Change `config:`

```yaml
# WRONG
config: openshift-workloads
clusters:
  - default:
      api_url: "{{ sandbox_user.sandbox_openshift_api_url }}"
      api_token: "{{ sandbox_user.cluster_admin_agnosticd_sa_token }}"

# CORRECT
config: namespace
```

### Remove `var:` from sandbox entry

```yaml
# WRONG
sandboxes:
  - kind: OcpSandbox
    namespace_suffix: user
    var: sandbox_user       # ← remove this
    cloud_selector: ...

# CORRECT
sandboxes:
  - kind: OcpSandbox
    namespace_suffix: user
    cloud_selector: ...
```

### Add `scm_ref:` to deployer

```yaml
# WRONG — Babylon doesn't know which branch to run
deployer:
  scm_url: https://github.com/agnosticd/agnosticd-v2
  execution_environment:
    image: ...

# CORRECT
deployer:
  scm_url: https://github.com/agnosticd/agnosticd-v2
  scm_ref: main             # ← required
  execution_environment:
    image: quay.io/agnosticd/ee-multicloud:chained-2025-12-17
```

---

## Step 2 — The `__meta__.sandboxes` Entry

The sandbox entry tells Sandbox API which cluster to pick and what to create.

```yaml
__meta__:
  sandbox_api:
    actions:
      destroy: {}
  sandboxes:
  - kind: OcpSandbox
    alias: cluster              # name this entry — used in cluster_condition
    namespace_suffix: user      # creates sandbox-{guid}-user namespace
    cloud_selector:
      cloud: cnv-dedicated-shared   # cluster type
      demo: my-lab-name             # tag specific to your lab's clusters
      purpose: prod                 # prod or dev pool
      # keycloak: "yes"             # add ONLY if you need an RHSSO user
    quota:
      limits.cpu: "4"
      requests.cpu: "4"
      limits.memory: 8Gi
      requests.memory: 8Gi
      requests.storage: 50Gi
```

**Multiple namespaces on the same cluster:**

```yaml
sandboxes:
- kind: OcpSandbox
  alias: primary
  namespace_suffix: user
  cloud_selector: { cloud: cnv-dedicated-shared, demo: my-lab, purpose: prod }
  quota: ...
- kind: OcpSandbox
  alias: tools
  var: sandbox_tools              # vars become: sandbox_tools_namespace etc.
  namespace_suffix: tools
  cluster_condition: same('primary')  # REQUIRED — pins to same cluster
  cloud_selector: { cloud: cnv-dedicated-shared, demo: my-lab, purpose: prod }
  quota: ...
```

---

## Step 3 — Variables You Get From Sandbox API

After cluster selection, these are available in every role automatically:

| Variable | What it is |
|---|---|
| `sandbox_openshift_api_url` | Cluster API server URL |
| **`sandbox_openshift_ingress_domain`** | **Wildcard apps domain — use for ALL service URLs** |
| `sandbox_openshift_console_url` | Web console URL |
| `cluster_admin_agnosticd_sa_token` | Cluster-admin SA token — never log or commit |
| `sandbox_openshift_namespace` | Primary namespace (`sandbox-{guid}-{suffix}`) |
| `sandbox_username` / `sandbox_password` | RHSSO user (only when `keycloak: "yes"`) |

**Build URLs like this:**
```yaml
my_role_app_url: "https://myapp-{{ sandbox_openshift_namespace }}.{{ sandbox_openshift_ingress_domain }}"
```

---

## Step 4 — Choose Your Approach

### Option A: Simple (Full Namespace API)
Sandbox API creates the namespace and RHSSO user. Your roles just deploy into them.

```yaml
config: namespace
workloads:
  - my_collection.ocp4_workload_my_lab
  - agnosticd.showroom.ocp4_workload_showroom

my_lab_namespace: "{{ sandbox_openshift_namespace }}"
my_lab_username: "{{ sandbox_username }}"      # only with keycloak: "yes"
my_lab_ingress: "{{ sandbox_openshift_ingress_domain }}"
```

Good for: simple labs, single namespace, no need for RHBK/Gitea customisation.
Full guide: https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/namespace-api.html

---

### Option B: Full Control (Scheduler-Only — Summit 2026 pattern)
Sandbox API just picks a cluster. Your roles create everything: RHBK user, namespaces,
Gitea instance, ArgoCD app-of-apps, LiteMaaS key.

```yaml
config: namespace

# Central username — ALL roles reference this one var
ocp4_workload_tenant_keycloak_username: "mylab-{{ guid }}"
common_password: "Lab{{ (guid | hash('sha256'))[:8] }}!"

workloads:
- agnosticd.namespaced_workloads.ocp4_workload_tenant_keycloak_user
- agnosticd.namespaced_workloads.ocp4_workload_tenant_namespace
- agnosticd.namespaced_workloads.ocp4_workload_tenant_gitea
- rhpds.litellm_virtual_keys.ocp4_workload_litellm_virtual_keys
- agnosticd.core_workloads.ocp4_workload_gitops_bootstrap
- agnosticd.showroom.ocp4_workload_showroom

# Destroy order (explicit — NOT automatic reverse)
remove_workloads:
- rhpds.litellm_virtual_keys.ocp4_workload_litellm_virtual_keys
- agnosticd.core_workloads.ocp4_workload_gitops_bootstrap
- agnosticd.namespaced_workloads.ocp4_workload_tenant_gitea
- agnosticd.namespaced_workloads.ocp4_workload_tenant_namespace
- agnosticd.namespaced_workloads.ocp4_workload_tenant_keycloak_user

requirements_content:
  collections:
  - name: https://github.com/agnosticd/namespaced_workloads.git
    type: git
    version: tenant-roles    # use main once PR #12 is merged
  - name: https://github.com/agnosticd/core_workloads.git
    type: git
    version: gitops-bootstrap-userinfo   # use main once PR #62 is merged
  - name: https://github.com/rhpds/rhpds.litellm_virtual_keys.git
    type: git
    version: main
```

Good for: complex labs needing custom namespaces, Gitea, RHBK, ArgoCD GitOps.
Full guide: https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/scheduler-only.html

---

## Step 5 — Tenant Roles (Option B Only)

| Role | What it does | Key vars |
|---|---|---|
| `ocp4_workload_tenant_keycloak_user` | Creates one RHBK user | `ocp4_workload_tenant_keycloak_username`, `ocp4_workload_tenant_keycloak_user_password` |
| `ocp4_workload_tenant_namespace` | Creates OCP namespaces with LimitRange | `ocp4_workload_tenant_namespace_suffixes`, `ocp4_workload_tenant_namespace_prefix` |
| `ocp4_workload_tenant_gitea` | Deploys per-tenant Gitea instance (operator CR) + user + repo | `ocp4_workload_tenant_gitea_username`, `ocp4_workload_tenant_gitea_repositories` |
| `ocp4_workload_litellm_virtual_keys` | Creates LiteMaaS virtual API key | `ocp4_workload_litellm_virtual_keys_models`, `ocp4_workload_litellm_virtual_keys_catch_all: false` |
| `ocp4_workload_gitops_bootstrap` | Creates ArgoCD Application (app-of-apps) | `ocp4_workload_gitops_bootstrap_repo_url`, `ocp4_workload_gitops_bootstrap_helm_values` |

**Important:** `ocp4_workload_litellm_virtual_keys_catch_all` MUST be `false` on shared clusters.
Setting it to `true` will delete all other tenants' LiteMaaS keys on destroy.

---

## Step 6 — Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| `config: openshift-workloads` | Fails before provisioning starts | Change to `config: namespace` |
| Missing `scm_ref:` in deployer | Fails before provisioning starts | Add `scm_ref: main` |
| `var: sandbox_user` on sandbox entry | Variables prefixed, not injected | Remove `var:` — use `sandbox_openshift_api_url` directly |
| `clusters:` block with `config: namespace` | Invalid config combination | Remove `clusters:` block entirely |
| Duplicate YAML key | yamllint error, order rejected | Search and remove the duplicate |
| Missing `cluster_condition: same('alias')` on second namespace | Two namespaces on different clusters | Always add `cluster_condition: same('alias')` |
| `catch_all: true` for LiteMaaS | Destroys all tenants' AI keys | Always `catch_all: false` |

---

## Step 7 — PRs and Collections

These are the branches to use until PRs are merged:

| Collection | Branch | PR |
|---|---|---|
| `agnosticd/namespaced_workloads` | `tenant-roles` | [PR #12](https://github.com/agnosticd/namespaced_workloads/pull/12) |
| `agnosticd/core_workloads` | `gitops-bootstrap-userinfo` | [PR #62](https://github.com/agnosticd/core_workloads/pull/62) |
| `agnosticd/agnosticd-v2` | `main` | PR #123 merged ✓ |

---

## References

| Resource | Link |
|---|---|
| Full documentation site | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/ |
| Scheduler-Only guide (Summit 2026) | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/scheduler-only.html |
| Full Namespace API guide | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/namespace-api.html |
| Migration guide | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/migration-guide.html |
| GitOps pattern | https://rhpds.github.io/ocpsandbox-mcp-with-openshift-gitops/gitops-pattern.html |
| MCP with OpenShift AgV | `agd_v2/mcp-with-openshift-sandbox/common.yaml` in rhpds/agnosticv |
| GitOps repo | https://github.com/rhpds/ocpsandbox-mcp-with-openshift-gitops |
