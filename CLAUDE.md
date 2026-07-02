# CLAUDE.md — clusters-provision

This repo holds the Helm charts for shared, cluster-wide infrastructure — ingress, secret
sync, tunnel ingress, and the OIDC identity provider. It's the "what to deploy" half of the
`clusters-provision`/`clusters-definition` pair — the infra-layer counterpart to
`devtools-provision`/`devtools-definition` (see that pair's `CLAUDE.md` files, and the
`add-devtool` skill, for end-user tools like Jira/Bitbucket instead).

## Role in the Architecture

ArgoCD's `ApplicationSet` (in `clusters-definition/applicationset.yaml`) auto-discovers
every directory under `clusters/*` here and deploys it with two Helm value sources merged
in order:

1. `clusters/<tool>/values.yaml` (this repo — values that don't change per environment)
2. `clusters-definition/clusters/<tool>/values.yaml` (overrides on top — see that repo's
   `CLAUDE.md`)

A tool directory must exist in both repos under the identical name, or the
`ApplicationSet`'s unconditional `$definition` reference fails.

## Repository Structure

```
.
└── clusters/
    └── <tool>/
        ├── Chart.yaml       # umbrella (dependencies: block) or plain
        ├── values.yaml      # env-invariant defaults
        ├── charts/          # only for umbrella charts — vendored upstream chart, unpacked
        ├── crds/            # only if the tool ships CRDs Helm can't manage via templates/
        └── templates/       # own templates (plain charts) or extra cluster-scoped
                              # resources (umbrella charts)
```

## Two chart patterns — pick based on whether a solid upstream chart exists

| Pattern | When | Examples |
|---|---|---|
| **A — Umbrella**, vendoring an unpacked upstream chart under `charts/<subchart>/` via a `file://` dependency | A well-maintained upstream Helm chart exists | `ingress-nginx`, `external-secrets-operator` |
| **B — Plain, self-authored** templates | No upstream chart exists, or the tool is simple enough to template directly | `cloudflared`, `rhbk` |

Full onboarding steps (fetching/unpacking a chart, SSM+ExternalSecret secrets,
bootstrap-ordering caveats, Cloudflare DNS exposure) are in the `add-cluster-provision`
skill — use it rather than freehanding a new tool's structure.

## Why `rhbk` is Pattern B, not an umbrella chart

This comes up because `rhbk` looks like it should be an umbrella chart — it deploys a real
Kubernetes Operator (`keycloak-operator`), the same shape as `external-secrets-operator`,
which *is* an umbrella chart. The difference: `external-secrets-operator`'s upstream
project publishes an actual Helm chart to vendor. RHBK's upstream operator project
(`keycloak/keycloak-k8s-resources`) does not — checking its tagged release tree (e.g.
`26.6.4`) shows only a raw `kubernetes/kubernetes.yml` install manifest and two CRD YAML
files, no `Chart.yaml` anywhere. Unofficial community charts exist on Artifact Hub that
just repackage that same manifest, but taking a dependency on one of those would trade a
version-pinned, auditable copy of upstream's own file for an unrelated third party's
re-wrapping of it, for no functional gain. So `rhbk` follows the `cloudflared` pattern
(Pattern B) instead: the manifest is vendored directly under `rhbk/templates/operator.yaml`
with a header comment pointing at the exact upstream source URL and tag, same idea as
`cloudflared` having no upstream chart to wrap either.

**Don't re-attempt converting `rhbk` to Pattern A** on the assumption an official chart
must exist somewhere — it doesn't, as of the check above. If the upstream
`keycloak-k8s-resources` project ever starts publishing an actual chart, re-verify by
checking that repo's tree at the target version tag before changing anything.

## Currently tracked tools

| Tool | Pattern | Notes |
|---|---|---|
| `ingress-nginx` | A | Ingress controller, `ClusterIP` only — `cloudflared` handles external routing |
| `external-secrets-operator` | A | Syncs AWS SSM `SecureString`s into cluster `Secret`s via a `ClusterSecretStore` |
| `cloudflared` | B | Dials out to Cloudflare, routes inbound traffic by Host header — no AWS load balancer needed |
| `rhbk` | B | RHBK/Keycloak OIDC provider, deployed via the upstream Keycloak Operator |

## Adding a New Tool

Use the `add-cluster-provision` skill — don't freehand the chart structure or secrets
pattern. Only use this repo for infrastructure the *cluster itself* needs to function
(ingress, secrets, tunnel, identity provider); end-user devtools belong in
`devtools-provision`/`devtools-definition` instead (`add-devtool` skill).
