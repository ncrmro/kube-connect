# kube-connect

`kube-connect` is a proposed portable CI action for reaching a private
Kubernetes API with short-lived identity. It is designed to support GitHub
Actions and Forgejo Actions, and to connect through either Tailscale or
Headscale without storing a long-lived Kubernetes credential in the CI
platform.

> [!IMPORTANT]
> This repository is in design review. The action and broker described below
> have not been implemented or released.

## What it is

The action will perform two independent jobs:

1. establish temporary network reachability to a private Kubernetes API; and
2. configure `kubectl` to request a fresh CI OIDC token for each authentication
   attempt.

The Kubernetes API server, not the action, will authenticate the token and
apply RBAC. Network membership will not imply Kubernetes authorization.

```text
GitHub Actions or Forgejo Actions
                │
                ├── CI OIDC token (Kubernetes audience)
                │       └── kubeconfig exec credential
                │               └── Kubernetes JWT authenticator + RBAC
                │
                └── CI OIDC token (network audience)
                        ├── Tailscale workload identity federation
                        └── Headscale credential broker
                                └── one-use, short-lived pre-auth key

Temporary tailscaled process
        └── userspace SOCKS5 proxy
                └── private Kubernetes API
```

## Design goals

- One provider-neutral action interface for GitHub and Forgejo.
- Direct CI OIDC authentication to Kubernetes.
- No static kubeconfig, service-account token, or reusable Headscale key in CI.
- Tailscale workload identity federation when the control plane supports it.
- A small credential broker for Headscale, because Headscale does not consume
  CI OIDC tokens directly.
- Unprivileged userspace networking by default, with explicit cleanup.
- Narrow trust policies based on issuer, audience, repository identity,
  workflow identity, event, and protected ref.
- Provider-specific behavior contained behind adapters and covered by
  compatibility tests.

## Proposed provider matrix

| CI provider | Network provider | Identity exchange |
| --- | --- | --- |
| GitHub | Tailscale | GitHub OIDC to Tailscale workload identity federation |
| GitHub | Headscale | GitHub OIDC to the kube-connect broker |
| Forgejo | Tailscale | Forgejo OIDC to Tailscale workload identity federation |
| Forgejo | Headscale | Forgejo OIDC to the kube-connect broker |

Every combination also requests a separate OIDC token whose audience is the
target Kubernetes cluster. The cluster validates that token directly.

## Proposed usage

The examples show the intended interface, not a released action. Production
workflows will pin the action to a full commit SHA.

### GitHub Actions

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: ncrmro/kube-connect@<full-commit-sha>
        with:
          ci-provider: auto
          network-provider: headscale
          target: production
          api-server: https://kubernetes.example.internal:6443
          certificate-authority-data: <base64-ca-data>
          kubernetes-audience: kubernetes-production
          headscale-login-server: https://headscale.example.com
          headscale-broker-url: https://ci-auth.example.com
          headscale-broker-audience: headscale-production

      - run: kubectl auth can-i get deployments --namespace app
```

### Forgejo Actions

Forgejo enables OIDC with `enable-openid-connect`, rather than GitHub's
`permissions.id-token` syntax.

```yaml
jobs:
  deploy:
    runs-on: docker
    enable-openid-connect: true
    steps:
      - uses: https://github.com/ncrmro/kube-connect@<full-commit-sha>
        with:
          ci-provider: auto
          network-provider: headscale
          target: production
          api-server: https://kubernetes.example.internal:6443
          certificate-authority-data: <base64-ca-data>
          kubernetes-audience: kubernetes-production
          headscale-login-server: https://headscale.example.com
          headscale-broker-url: https://ci-auth.example.com
          headscale-broker-audience: headscale-production

      - run: kubectl auth can-i get deployments --namespace app
```

The Forgejo action reference and runner labels are placeholders until the
compatibility matrix is verified.

## Security boundary

`kube-connect` will configure connectivity and credentials only. It will not
create `Role`, `ClusterRole`, `RoleBinding`, or `ClusterRoleBinding` objects.
Cluster administrators remain responsible for:

- trusted OIDC issuers and audiences;
- claim validation and user/group mapping;
- namespace-scoped RBAC;
- Tailscale or Headscale ACL policy;
- the Headscale broker's target-to-tag mapping; and
- restricting privileged workflows to trusted refs and reviewed code.

The action will not expose raw OIDC tokens, pre-auth keys, or kubeconfig
contents as outputs.

## Scope

Version 1 targets Linux runners and ordinary Kubernetes API traffic. Userspace
networking is the default. Kernel networking for commands that require
streaming transports is an explicit, privileged escape hatch.

The initial release does not aim to:

- provision clusters, OIDC issuers, RBAC, Headscale, or Tailscale;
- replace a general-purpose VPN action;
- support untrusted pull-request deployment workflows;
- issue Kubernetes credentials from a custom broker; or
- support macOS or Windows runners.

## Project documents

- [REQUIREMENTS.md](REQUIREMENTS.md) is the normative product and security
  contract.
- [PLAN.md](PLAN.md) is the projected git history through the first release.
- [TASKS.md](TASKS.md) is the executable implementation checklist.
- [CONTRIBUTING.md](CONTRIBUTING.md) defines the contribution and review
  workflow.
- [AGENTS.md](AGENTS.md) contains repository instructions for coding agents.

## Primary references

- [GitHub Actions OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)
- [Forgejo Actions OIDC](https://forgejo.org/docs/latest/user/actions/security-openid-connect/)
- [Tailscale GitHub Action](https://tailscale.com/docs/integrations/github/github-action)
- [Headscale pre-authenticated keys](https://headscale.net/development/usage/getting-started/#pre-authenticated-key)
- [Kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes client authentication v1](https://kubernetes.io/docs/reference/config-api/client-authentication.v1/)
