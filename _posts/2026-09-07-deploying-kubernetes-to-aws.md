---
layout: post
title: "Deploying Kubernetes to AWS"
date: 2026-09-07 12:00:00 +0000
categories: platform kubernetes argocd
---

The most consequential line in my platform is five characters long:

```yaml
clusterResourceWhitelist: []
```

It sits in [`projects/apps.yaml`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/projects/apps.yaml), an ArgoCD `AppProject` that
governs one repository of tenant workloads. An empty list means no cluster-scoped resource of any kind may be
created: no `ClusterRole`, no `CustomResourceDefinition`, no
`ValidatingWebhookConfiguration`, no `Namespace`, no `PriorityClass`.

I did not expect it to change anything. It has now twice forced a design better
than the one I would have shipped, and both times it did so by refusing
something I had already written.

## The shape it sits in

Three repositories, split by change rate and blast radius rather than by tidiness:

| Repo | Contents | Changes | Blast radius |
|---|---|---|---|
| [`platform-infra`](https://github.com/Wasee3/platform-infra) | VPC, EKS, IAM, Karpenter prerequisites | quarterly | the account |
| [`platform-system`](https://github.com/Wasee3/platform-system) | ArgoCD, Cilium, External Secrets, telemetry | monthly | the cluster |
| [`platform-apps`](https://github.com/Wasee3/platform-apps) | tenant workloads | hourly | one namespace |

The argument for splitting is ordinary enough. A VPC changes quarterly, an image
tag changes hourly, and one repository forces a single review process onto both —
which will be wrong for one of them. Either the CNI gets rubber-stamped along
with deploy traffic, or every deploy waits on somebody qualified to review a CNI
change.

That argument is real but it is not the interesting one, because it is satisfied
by directories. You can get different CODEOWNERS on different paths in a single
repository and capture most of it.

## Why a convention was not enough

What a single repository cannot reproduce is enforcement.

CODEOWNERS is advisory. It requests a review; it does not prevent a merge with an
approval from somebody who did not read carefully. A directory convention is a
sentence in a README that a person has to have read, remembered, and chosen to
follow at 5pm on a Friday.

Two repositories mean two `AppProject`s, and an `AppProject` is not advisory:

| | [`system`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/projects/system.yaml) | [`apps`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/projects/apps.yaml) |
|---|---|---|
| `sourceRepos` | this repo + pinned chart repos | `platform-apps` + one named upstream |
| `destinations` | all namespaces | `demo-otel`, `greenlight` |
| `clusterResourceWhitelist` | `*` | **empty** |
| prune | off | on |

The two projects invert each other ([ADR-0001](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/docs/adr/0001-system-apps-repo-split.md)
has the full argument and what it costs). `system` has broad permissions and a narrow
source: it can create anything, but those permissions are reachable only from one
repository that sits behind review. `apps` has the opposite — it may deploy from
a tenant repo into two named namespaces, and may not create a single
cluster-scoped object.

A manifest from `platform-apps` carrying a `ClusterRole` is **rejected by the
ArgoCD API server at sync time**. Not flagged in review. Refused.

That difference — flagged versus refused — is the whole post.

## The first time it bit: namespaces

A `Namespace` is a cluster-scoped resource.

Which means `platform-apps` cannot create the namespace its own workloads run
in. I hit this immediately and my first instinct was that I had misconfigured
something.

I had not. Tenant onboarding moved to [`platform-system/system/tenants/`](https://github.com/Wasee3/platform-system/tree/d11d60ab082d7d1a60b70bcfc265f15166a7e497/system/tenants), where a
namespace arrives as a reviewed unit: the `Namespace` itself, its `ResourceQuota`,
its `LimitRange`, and its Pod Security labels, together in one file.

That is better than what I was going to do, which was let each app bring its own
namespace and add quotas later, once something had misbehaved. The constraint
converted "later" into "now."

The `LimitRange` turned out to be load-bearing rather than a safety net. Not one
of the 26 containers in the upstream demo chart sets a CPU request, and a
`ResourceQuota` on `requests.*` makes requests mandatory. Without defaults, every
pod is rejected at admission with an error naming the quota rather than the
omission — a message that sends you looking in exactly the wrong place.

## The second time: the telemetry pipeline

`platform-apps` deploys the [OpenTelemetry Astronomy Shop](https://github.com/Wasee3/platform-apps/blob/0fdec45173f85031afe19cb3a78f1f30dbb3f823/apps/otel-demo/app.yaml): 25 instrumented
microservices in a dozen languages, the reference demo for distributed tracing.

Deploying it unchanged fails. Rendered with defaults, the chart produces three
`ClusterRole`s and three `ClusterRoleBinding`s:

```
grafana-clusterrole    dashboard sidecar, discovers ConfigMaps cluster-wide
otel-collector         k8sattributes processor, reads pods and namespaces
prometheus             Kubernetes service discovery
```

ArgoCD refused all six, and the Application failed.

The tempting move here is to add three entries to `clusterResourceWhitelist` and
carry on. It is one line of YAML and the demo works again.

The rejection was correct, though, and worth sitting with. The `k8sattributes`
processor exists to turn an IP address into a workload name, and it does that by
reading pods, namespaces and nodes **across every namespace**. A tenant that can
create that `ClusterRole` can read the metadata of every other tenant's
workloads. The boundary caught precisely the thing it exists to catch.

So the pipeline moved instead ([ADR-0005](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/docs/adr/0005-telemetry-pipeline-is-platform.md)). The collector and Jaeger became platform
components in `platform-system` — a shared OTLP gateway at sync wave -30, a trace
backend at -35 — and the tenant app disables every bundled backend and ships its
spans to the gateway.

The result is checkable, which is the part I like:

```
chart defaults          → 3 ClusterRoles  (grafana, otel-collector, prometheus)
collector enabled only  → 1 ClusterRole   (otel-collector)
this repo's values      → 0
```

52 objects, zero cluster-scoped resources.

And a shared gateway is the better shape regardless of the constraint. Letting
every tenant ship its own collector means N collectors, N configurations, N sets
of cluster RBAC, and no single place to add sampling or redaction or a new
backend. I knew that. I would not have done it under deadline. The empty list
made the good version the only version that synced.

## The failure mode is silence

Here is the property that makes this fragile, and the reason it needs more than a
comment.

If somebody widens `clusterResourceWhitelist` — adds a `*`, adds three entries to
unblock a chart, standardises it against the `system` project — **nothing breaks**.
Every deployment keeps working. Every test passes. The cluster is fine. The only
thing that changed is that the boundary is gone, and there is no signal.

A comment saying "do not widen this" is read once, by the person who writes it.
Six months later somebody tidying up sync policies flips it, everything keeps
working, and the protection is gone.

So the invariants are executable. [`scripts/check-invariants.py`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/scripts/check-invariants.py) fails the build
on:

- `clusterResourceWhitelist` on the `apps` project being anything but empty
- any Application under `registrations/` using a project other than `apps`
- any Application missing an explicit sync-wave annotation
- `prune: true` on Cilium, whose deletion would sever ArgoCD's own networking

The script's own docstring puts it better than I can paraphrase: these are not
style checks, each one guards a failure mode that is silent, and a comment cannot
catch a regression.

## Running it yourself

The deploy sequence is worth including because its shape is the same argument.
Every step below is ordered by a hard dependency, and there is exactly one value
that has to be moved by hand.

**0. State backend — once per AWS account.** This root module creates the bucket
that every other root module stores state in, so it cannot use that bucket on its
own first apply.

```bash
cd platform-infra/bootstrap
terraform init && terraform apply -var-file=account.tfvars
# add the backend block the `backend_config` output prints, then:
terraform init -migrate-state
rm terraform.tfstate terraform.tfstate.backup
```

**1. The cloud foundation.** VPC, EKS control plane, OIDC provider, IRSA roles,
Karpenter's IAM and interruption queue.

```bash
$EDITOR envs/dev/backend.hcl        # bucket, region, kms_key_id
$EDITOR envs/dev/terraform.tfvars   # region, AZs, admin principals

terraform -chdir=envs/dev init -backend-config=backend.hcl
make plan ENV=dev
make apply ENV=dev

aws eks update-kubeconfig --name sandbox-dev --region me-central-1
```

Leave `enable_vpc_cni_addon` and `enable_kube_proxy_addon` at their defaults of
`true`. That looks wrong in a cluster whose target CNI is Cilium, and it is not:
a node with no CNI never reaches `Ready`, so the managed node group creation
times out. EKS has to bootstrap on the VPC CNI and Cilium chains behind it. The
mesh is a later, deliberate cutover.

**2. The one value that cannot be automated.** External Secrets projects every
other AWS identity into the cluster, so its own identity cannot arrive that way —
it is the component doing the reading.

```bash
terraform -chdir=../platform-infra/envs/dev output -raw external_secrets_role_arn
```

That goes into [`platform-system/system/external-secrets/values/base.yaml`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/system/external-secrets/values/base.yaml),
replacing the placeholder, and gets committed. It is not a secret; it is an
identifier that is useless without the OIDC trust relationship behind it. Every
bootstrap has an irreducible seed like this. The goal is to have exactly one and
to name it, rather than to pretend it does not exist.

**3. Seed ArgoCD.**

```bash
cd platform-system
./bootstrap/bootstrap.sh
```

[`bootstrap.sh`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/bootstrap/bootstrap.sh) creates the `argocd` namespace, renders ArgoCD from **the same chart version
and values file** that [`system/argocd/app.yaml`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/system/argocd/app.yaml) uses, applies it twice, waits for
the rollout, then applies `projects/` and the two root Applications.

Three details in that sentence are load-bearing. It renders from the same source
as the self-managing Application, because if the seed and the Application produce
different manifests, ArgoCD's first self-reconcile rewrites its own Deployment
and restarts mid-sync. It applies twice because the first pass installs the
`Application` and `AppProject` CRDs and any custom resource in the same manifest
set fails on that pass. And it applies projects before roots because
[`root-app.yaml`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/bootstrap/root-app.yaml) declares `project: system`, and an Application naming a project
that does not exist is rejected outright.

Then it stops. Everything after this point is a git commit.

**4. Watch it converge.**

```bash
kubectl -n argocd get applications -w
```

```
-200  root-system, root-registrations
-100  gateway-api-crds
 -90  cilium
 -60  argocd            adopts the install bootstrap.sh just made
 -50  external-secrets
 -40  handoff
  10  platform-apps
```

There is no step 5 for the tenant repo. `platform-apps` is registered from
`platform-system` and arrives at wave 10 on its own — under the restricted
project, which is where this post started.

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

No ingress, deliberately: until the mesh is up there is no Gateway to put it
behind, and an unauthenticated ArgoCD on the internet is worse than a
port-forward.

What you have at this point is a working cluster with L3/L4 policy and Hubble
flow observability, and **no service mesh**. Gateway API needs
`kubeProxyReplacement`, which needs kube-proxy gone, which is the phase-2
cutover — two values files, a paired change in `platform-infra`, and nodes
recycled one at a time. Failure modes for each wave above, and the cutover
procedure, are in [`platform-system/docs/bootstrap.md`](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/docs/bootstrap.md) and
[ADR-0004](https://github.com/Wasee3/platform-system/blob/d11d60ab082d7d1a60b70bcfc265f15166a7e497/docs/adr/0004-cilium-two-phase-cutover.md).

## What it costs

I would rather be honest about this than sell the pattern.

**Cross-cutting changes need coordinated pull requests.** Adding a controller that
needs a new IAM role touches all three repositories: `platform-infra` for the
role, `platform-system` for the Helm release, `platform-apps` for anything
consuming it. That is the real price and it is paid on every genuinely new
capability.

**Bootstrap takes longer to explain.** Somebody new has to understand three
repositories before they understand the platform.

**The demo is no longer self-contained.** Anyone reading the upstream chart's
documentation and expecting `helm install` behaviour finds five components
missing. The demo's `frontend-proxy` still carries `GRAFANA_HOST` and
`JAEGER_HOST` pointing at now-absent in-namespace services, so its `/grafana` and
`/jaeger` shortcuts return 502. Repointing them needs a cross-namespace
NetworkPolicy allowance I have not written. The honest position is that the
shortcut is broken and Jaeger is reached directly.

## When I would give it up

When the `apps` project needs so many exceptions that the list stops being empty.

That would mean tenants legitimately need cluster-scoped resources, and the right
answer then is not to widen the project — that keeps the shape while losing the
property. It is a mechanism that grants them narrowly: Crossplane compositions,
or an operator that owns the cluster-scoped object on the tenant's behalf.

Until then the list stays empty, and every so often it refuses something I wrote
and turns out to be right.
