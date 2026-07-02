# clusters-provision

Helm charts for the cluster-wide infrastructure this GitOps platform runs on — ingress,
secrets sync, DNS tunneling, and the OIDC identity provider. This is the "what to deploy"
half of a provision/definition pair; environment-specific values live in the sibling
[`clusters-definition`](https://github.com/devops-tashtiot/clusters-definition) repo, and
both are auto-discovered by an ArgoCD `ApplicationSet`. The devtools equivalent of this
pair is [`devtools-provision`](https://github.com/devops-tashtiot/devtools-provision)/
[`devtools-definition`](https://github.com/devops-tashtiot/devtools-definition).

## What's here

Each directory under `clusters/` is a Helm chart, using one of two patterns depending on
whether a solid upstream chart exists to vendor:

| Tool | Pattern | Why |
|---|---|---|
| `ingress-nginx` | Umbrella — vendors the upstream `ingress-nginx/ingress-nginx` chart | Solid upstream chart exists |
| `external-secrets-operator` | Umbrella — vendors the upstream `external-secrets/external-secrets` chart | Solid upstream chart exists |
| `cloudflared` | Plain, self-authored | No official Helm chart — `cloudflared` ships as a bare binary/image, not a chart |
| `rhbk` | Plain, self-authored — vendors the upstream Operator's raw install manifest directly | No official Helm chart — see note below |

### Why `rhbk` isn't an umbrella chart

It looks like it should be — RHBK (Red Hat Build of Keycloak) runs behind a real Kubernetes
Operator, the same shape as `external-secrets-operator`. But the upstream project that
Operator ships from (`keycloak/keycloak-k8s-resources`) doesn't publish a Helm chart at
all — only a plain `kubernetes.yml` manifest and two CRD files (confirmed directly against
that repo's tagged release tree). Community/third-party Helm charts exist that repackage
that same manifest, but depending on one of those would just swap a version-pinned copy of
upstream's own file for an unrelated party's re-wrapping of it, with no real gain. So
`rhbk` follows the `cloudflared` pattern instead: the manifest is vendored straight into
`rhbk/templates/operator.yaml`, with a header comment pointing at its exact upstream
source and tag.

## Adding a new tool

See the `add-cluster-provision` skill (`.claude/skills/add-cluster-provision/SKILL.md` at
the project root) for the full onboarding checklist — chart pattern selection, secrets via
SSM + `ExternalSecret`, bootstrap ordering, and Cloudflare DNS exposure.

See [`CLAUDE.md`](./CLAUDE.md) for the fuller architecture writeup.
