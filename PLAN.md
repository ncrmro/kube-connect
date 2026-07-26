---
updated: 2026-07-26
landing: merge-commit
---

# kube-connect plan

The first release establishes a portable, least-privilege path from GitHub or
Forgejo Actions to a private Kubernetes API. Work proceeds from reproducible
tooling and provider contracts, through direct Kubernetes OIDC and ephemeral
networking, to the Headscale broker and native provider verification. The
order keeps every security boundary testable before it is composed into the
public action.

```text
◇  v0.1.0 (next)
│
○  docs: publish operator runbooks
○  test(e2e): verify Forgejo Headscale path
○  feat(action): verify scoped cluster access
○  feat(action): compose the portable action
│  ── milestone: portable Kubernetes connection ──
○  feat(network): add Tailscale federation
○  feat(network): connect through Headscale
○  feat(broker): issue constrained Headscale keys
│  ── milestone: ephemeral private networking ──
○  feat(kube): generate OIDC exec kubeconfig
○  feat(oidc): normalize GitHub and Forgejo
│  ── milestone: direct workload identity ──
○  test(ci): add native provider matrices
○  chore(dev): add reproducible toolchain
│
│ ◉  docs: tighten Headscale E2E contract    PR #1 draft  agent/initial-design → main
│ ○  docs: add Headscale Helm E2E            004a066
│ ○  docs: establish project contract        e6d5aa3
├─╯
●  chore: initialize repository              3a68033  ← main
```

## Landing agreement

Every `○` above lands on `main` as its own commit. Pull requests use merge
commits. A multi-commit planned span must retain one mainline commit per
planned line and must not be squashed.

The graph is dependency order, not an estimate. Time flows upward: the lowest
unshipped commit is next.

## Milestones

### Project foundation

`chore(dev)` creates the devenv v2 task surface. `test(ci)` adds fixture-based
checks on both CI systems before production credentials exist.

Exit criteria:

- local and CI commands use the same devenv tasks;
- third-party dependencies are pinned;
- GitHub and Forgejo can run the provider contract suite; and
- the compatibility matrix distinguishes verified support from targets.

Requirements: KC-002, KC-070 through KC-074, KC-080, KC-085.

### Direct workload identity

`feat(oidc)` implements the provider-neutral token contract and separate
GitHub and Forgejo adapters. `feat(kube)` creates a temporary kubeconfig whose
exec helper requests a fresh Kubernetes-audience token.

Exit criteria:

- both providers request explicit audiences;
- no bearer token is persisted;
- the kubeconfig uses `ExecCredential` v1;
- provider and claim failures are actionable and redacted; and
- the action creates no RBAC.

Requirements: KC-001, KC-004, KC-005, KC-010 through KC-027, KC-060 through
KC-064.

### Ephemeral private networking

`feat(broker)` implements the constrained Headscale key issuer.
`feat(network)` then adds the Headscale client path and Tailscale workload
identity federation as separate adapters.

Exit criteria:

- Headscale keys are one-use, ephemeral, target-mapped, and replay-protected;
- Tailscale uses workload identity federation;
- userspace networking is isolated below the runner temporary directory;
- the Kubernetes API is reached through the loopback SOCKS5 proxy; and
- cleanup succeeds after both success and failure.

Requirements: KC-030 through KC-058, KC-063, KC-080, KC-081, KC-083.

### Portable Kubernetes connection

`feat(action)` composes the adapters behind the public interface and adds an
optional, read-only RBAC capability check. `test(e2e)` exercises the native
Forgejo/Headscale lane in a disposable two-cluster lab. `docs` finishes
operator and consumer runbooks.

Exit criteria:

- all four CI/network matrix cells pass native end-to-end tests;
- the local Forgejo/Headscale lane uses the locked `gabe565/headscale` chart
  and verifies that the target API has no direct runner route;
- cert-manager remains an external prerequisite with explicit CA
  distribution;
- unsupported or unsafe combinations fail before mutation;
- operator documentation covers setup, rotation, audit, and revocation; and
- a release candidate meets the supply-chain requirements.

Requirements: KC-001 through KC-096.

## Planned pull-request slices

| Branch | Planned commits | Primary requirements |
| --- | --- | --- |
| `chore/foundation` | `chore(dev)`, `test(ci)` | KC-070–KC-074, KC-080, KC-085 |
| `feat/workload-identity` | `feat(oidc)`, `feat(kube)` | KC-010–KC-027 |
| `feat/headscale-broker` | `feat(broker)` | KC-050–KC-058 |
| `feat/network-adapters` | two `feat(network)` commits | KC-030–KC-043, KC-050 |
| `feat/portable-action` | two `feat(action)` commits | KC-001–KC-005, KC-060–KC-064 |
| `test/forgejo-headscale-e2e` | `test(e2e): verify Forgejo Headscale path` | KC-090–KC-096 |
| `docs/operator-runbooks` | `docs: publish operator runbooks` | KC-084–KC-085 |

Branches may be stacked when a later slice depends on an unmerged earlier
slice. They land bottom-up and are retargeted to `main` after their base lands.

## Decisions

- Kubernetes trusts CI OIDC directly; there is no Kubernetes token broker.
- Headscale requires a narrow credential broker; Tailscale uses its native
  workload identity federation.
- The common action selects a symbolic target. Authorization remains in
  cluster RBAC and network control-plane policy.
- Userspace networking is the unprivileged default. Kernel networking is an
  explicit Linux-only exception.
- The public repository remains environment-neutral. Concrete issuers,
  repository IDs, target mappings, hostnames, and RBAC live in deployment
  configuration.
- The local Forgejo/Headscale lane uses separate management and target
  clusters. The management cluster installs the pinned `gabe565/headscale`
  chart and starts with cert-manager already installed.

## Risks

- Forgejo follows GitHub Actions concepts but differs in OIDC enablement,
  runner runtime, reusable workflow behavior, and cross-host action lookup.
- Userspace SOCKS5 may not support every Kubernetes streaming operation.
- Headscale's administrative API has a broader credential boundary than the
  desired broker endpoint and must be isolated.
- The selected Headscale chart is community-maintained and includes
  third-party chart dependencies, so upgrades require review of the rendered
  manifests and end-to-end test results.
- A container-backed Forgejo runner may require privileged test
  infrastructure. The lab must isolate its namespace, service-account token,
  RBAC, and network so the runner cannot use those privileges against the
  target cluster.
- OIDC claim shapes and runner runtimes evolve, so compatibility must be
  versioned and tested rather than assumed.
