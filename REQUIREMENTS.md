# kube-connect requirements

## 1. Normative language

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in
[RFC 2119](https://www.rfc-editor.org/rfc/rfc2119).

Each requirement has a stable `KC-*` identifier. Implementations and tests
SHOULD cite these identifiers.

## 2. Product boundary

### KC-001: Portable action

The project MUST provide one provider-neutral action interface for configuring
temporary private-network access and Kubernetes client authentication from a
CI job.

### KC-002: Supported CI providers

The first stable release MUST support GitHub Actions and Forgejo Actions on
Linux runners. The action MAY auto-detect the provider, but it MUST accept an
explicit provider override and MUST fail closed when detection is ambiguous.

### KC-003: Supported network providers

The first stable release MUST support Tailscale and Headscale through separate
network adapters. Provider-specific inputs and behavior MUST NOT leak into the
common control flow beyond typed adapter contracts.

### KC-004: Separation of concerns

The action MUST configure connectivity and Kubernetes client authentication
only. It MUST NOT provision a Kubernetes cluster, OIDC issuer, VPN control
plane, `Role`, `ClusterRole`, `RoleBinding`, or `ClusterRoleBinding`.

### KC-005: No implicit authorization

Network enrollment MUST NOT grant Kubernetes authorization. Kubernetes RBAC
MUST remain the authority for API operations after authentication.

## 3. CI OIDC identity

### KC-010: Direct OIDC

The action MUST use the CI provider's native OIDC token endpoint. It MUST NOT
use `GITHUB_TOKEN`, `FORGEJO_TOKEN`, a personal access token, or a stored
service-account token as a substitute for workload identity.

### KC-011: Separate audiences

The action MUST request separate OIDC tokens for Kubernetes authentication and
network enrollment. A token requested for one audience MUST NOT be reused with
another relying party.

### KC-012: GitHub enablement

GitHub callers MUST grant `id-token: write`. The action MUST provide a
diagnostic that names this missing permission without printing token request
credentials.

### KC-013: Forgejo enablement

Forgejo callers MUST set `enable-openid-connect: true` at workflow or job
scope. The action MUST provide a diagnostic that names this missing setting
without printing token request credentials.

### KC-014: Provider adapter contract

Each CI provider adapter MUST return a token for an explicit audience and MUST
normalize, at minimum, issuer, audience, subject, repository, repository ID
when available, repository owner, repository owner ID when available,
workflow reference, event name, ref, ref protection state when available,
run ID, run attempt, issued-at time, not-before time, expiry time, and token
identifier when available.

### KC-015: Immutable identity

Trust policies MUST prefer immutable repository and owner IDs over mutable
names when the provider supplies those claims. Names MAY be retained for
diagnostics and policy readability but MUST NOT be the sole identity control
when stable IDs exist.

### KC-016: Untrusted events

Privileged connection attempts from pull-request events MUST be rejected by
default. Enabling any pull-request flow MUST require an explicit policy and
MUST NOT execute or inspect untrusted pull-request code while OIDC request
credentials or derived credentials are available.

## 4. Kubernetes authentication

### KC-020: Cluster validates the CI token

Kubernetes MUST validate the CI OIDC token directly with a configured JWT
authenticator. A kube-connect broker MUST NOT exchange CI identity for a
separate Kubernetes bearer token in version 1.

### KC-021: Structured authentication

Operator documentation SHOULD use Kubernetes structured authentication
configuration. Each issuer MUST have an explicit issuer URL, audience,
claim-validation rules, and claim-to-user/group mappings.

### KC-022: Narrow claim policy

Cluster-side policy MUST validate issuer, audience, repository identity,
workflow identity, event, and ref or environment as appropriate. Production
roles SHOULD require a protected ref or protected environment.

### KC-023: Fresh exec credentials

The generated kubeconfig MUST use
`client.authentication.k8s.io/v1` `ExecCredential` with
`interactiveMode: Never`. The exec helper MUST request a fresh token for the
configured Kubernetes audience and MUST return its expiration timestamp.

### KC-024: No persisted bearer token

The action MUST NOT write a raw OIDC bearer token into kubeconfig. Temporary
credential material MUST be held in memory where practical and MUST NOT
persist beyond the job.

### KC-025: Kubeconfig isolation

Generated kubeconfig and credential-helper files MUST be stored below the
runner temporary directory with owner-only permissions. The action MUST expose
only the kubeconfig path and context name as non-secret outputs.

### KC-026: Scoped context

The generated context MUST use the requested namespace when provided. The
namespace is a client default only and MUST NOT be represented as an
authorization boundary.

### KC-027: Capability verification

The action SHOULD support an optional `kubectl auth can-i` verification step.
The caller MUST specify the verb, resource, and optional namespace; the action
MUST NOT infer or create permissions from the result.

## 5. Network lifecycle

### KC-030: Ephemeral node

Every CI job MUST create a distinct ephemeral network node. Network state MUST
NOT be reused across jobs.

### KC-031: Userspace default

The default Linux mode MUST run an isolated `tailscaled` process with
`--tun=userspace-networking`, state below the runner temporary directory, and a
SOCKS5 listener bound to loopback.

### KC-032: Kubernetes proxy

In userspace mode, the generated kubeconfig MUST route Kubernetes API traffic
through the loopback SOCKS5 proxy. The action MUST wait for network readiness
before reporting success.

### KC-033: Privileged escape hatch

The action MAY provide an explicit Linux kernel-networking mode for Kubernetes
operations whose streaming transports do not work through userspace SOCKS5.
That mode MUST be disabled by default, MUST document its required privileges,
and MUST fail before mutation when the runner lacks those privileges.

### KC-034: Cleanup

The action MUST register a post phase before starting network enrollment. The
post phase MUST attempt logout, stop the isolated daemon, remove temporary
state, and avoid masking the original action failure.

### KC-035: Collision avoidance

Socket paths, state paths, proxy ports, hostnames, and context names MUST be
unique to the job and MUST NOT rely on a machine-global `tailscaled` service.

## 6. Tailscale adapter

### KC-040: Workload identity federation

The Tailscale adapter MUST use workload identity federation for production
authentication. Long-lived OAuth secrets and reusable auth keys MUST NOT be
part of the supported production interface.

### KC-041: Federated identity policy

Tailscale federated identity configuration MUST validate the CI issuer,
audience, and narrowly scoped repository and workflow claims. Its tag ownership
MUST limit which node tags the CI identity can request.

### KC-042: Target selection

The common action interface MUST accept a symbolic target. The Tailscale
adapter MAY derive a configured tag from that target, but it MUST NOT treat
caller-supplied tag text as authorization. The Tailscale control-plane policy
remains authoritative.

### KC-043: Provider compatibility

The project MUST test Tailscale workload identity federation independently
with GitHub and Forgejo before marking either matrix cell supported.

## 7. Headscale adapter and broker

### KC-050: Broker boundary

Headscale enrollment MUST use a dedicated credential broker because Headscale
does not directly validate CI OIDC for pre-auth key issuance. The CI platform
MUST NOT store a Headscale API key.

### KC-051: Token validation

Before issuing a key, the broker MUST verify the JWT signature and algorithm,
issuer, audience, subject, repository identity, workflow reference, event,
ref, issued-at time, not-before time, expiry time, and token identifier when
present.

### KC-052: Fixed target mapping

The broker MUST map a symbolic target to administrator-defined Headscale user,
tags, expiry ceiling, allowed repositories, allowed workflows, allowed events,
and allowed refs. The request MUST NOT accept raw Headscale tags, user IDs, or
arbitrary expiry values.

### KC-053: One-use short-lived keys

The broker MUST issue non-reusable, ephemeral Headscale pre-auth keys. The key
expiry MUST be the shortest of the configured target ceiling, the remaining
OIDC token lifetime, and the broker's global ceiling.

### KC-054: Replay prevention

The broker MUST reject reuse of an OIDC token at the key-issuance endpoint. It
MUST identify the token by `jti` when that claim is present and otherwise by a
collision-resistant digest of the complete signed token. Replay records MUST
live at least until the token expiry and MUST include issuer and audience in
their storage key.

### KC-055: Minimal response

The broker response MUST contain only the pre-auth key and its expiry. The
response MUST set cache-prevention headers and MUST NOT be logged by the
broker or action.

### KC-056: Broker credential

The Headscale API credential MUST be supplied to the broker through its runtime
secret mechanism, excluded from logs and traces, and scoped to the minimum
available Headscale permissions.

### KC-057: Rate limiting and audit

The broker MUST rate-limit by issuer, repository identity, workflow, and
source as appropriate. It MUST record credential-free audit events for
allow/deny result, target, immutable repository identity, workflow identity,
run identity, and policy reason.

### KC-058: Fail closed

The broker MUST fail closed when discovery, JWKS refresh, replay storage,
policy storage, or Headscale is unavailable. Cached discovery keys MAY be used
only within a documented bounded stale interval.

## 8. Inputs, outputs, and diagnostics

### KC-060: Common inputs

The action MUST define and validate common inputs for CI provider, network
provider, symbolic target, Kubernetes API server, Kubernetes CA data,
Kubernetes audience, context name, namespace, network mode, and readiness
timeout.

### KC-061: Provider inputs

Provider-only inputs MUST be namespaced and rejected when they conflict with
the selected provider. Unknown inputs SHOULD fail validation where the action
runtime permits detection.

### KC-062: Safe outputs

The action MAY output the kubeconfig path, context name, detected CI provider,
selected network provider, and non-sensitive diagnostic identifiers. It MUST
NOT output an OIDC token, pre-auth key, authorization header, Headscale API
key, or credential-bearing kubeconfig.

### KC-063: Redaction

The action MUST register each received credential with the runner's masking
mechanism before subsequent processing. The broker MUST apply secret filters
at logging and tracing boundaries. Logs, errors, traces, test snapshots, and
support bundles from either component MUST redact credentials.

### KC-064: Actionable failures

Failures MUST identify the stage, provider, and rejected policy condition
without exposing credential values. Unsupported provider combinations MUST
fail before network or filesystem mutation.

## 9. Supply chain and runtime

### KC-070: Reproducible environment

Development, validation, and build commands MUST be exposed through a devenv
v2 environment. CI MUST invoke the same named tasks used locally.

### KC-071: Pinned dependencies

Third-party actions used by project workflows MUST be pinned to full commit
SHAs. Downloaded binaries MUST be pinned by version and verified by published
checksum or signature. Installation through an unverified `curl | sh` pipeline
MUST NOT be used.

### KC-072: Action runtime

The JavaScript action MUST target GitHub's supported Node.js 24 action runtime.
Forgejo compatibility MUST be gated by a native Forgejo runner test rather
than inferred from GitHub compatibility.

### KC-073: Least workflow permissions

Published examples MUST set only the workflow permissions required by the
example. The connection action itself MUST require no repository write
permission.

### KC-074: Release integrity

Releases MUST publish immutable commit references, checksums for bundled or
downloaded artifacts, a generated software bill of materials, and provenance
when supported by the release platform.

## 10. Verification and operations

### KC-080: Unit and contract tests

Tests MUST cover input validation, provider normalization, audience separation,
kubeconfig generation, policy evaluation, expiry calculation, replay
prevention, redaction, and cleanup.

### KC-081: Negative claim matrix

Broker and cluster-policy fixtures MUST reject wrong issuer, audience,
repository identity, workflow, event, ref, target, expired token,
not-yet-valid token, and replayed token identity.

### KC-082: Native end-to-end tests

Each advertised provider matrix cell MUST pass an end-to-end test on the
corresponding native CI runner against disposable or dedicated test
infrastructure.

### KC-083: Cleanup tests

Tests MUST verify cleanup after success and action failure. Cancellation
cleanup MUST be tested where the runner exposes a deterministic cancellation
hook; any runner limitation MUST be documented.

### KC-084: Operator documentation

Before the first stable release, the project MUST document cluster JWT
authentication, RBAC ownership, Tailscale federation, Headscale broker
deployment, target policy, secret rotation, audit review, troubleshooting,
and break-glass revocation.

### KC-085: Compatibility declaration

The repository MUST publish a versioned compatibility matrix covering GitHub
runner, Forgejo, Forgejo runner, Kubernetes, Tailscale client, Tailscale
control plane, and Headscale versions tested by CI.

## 11. Local Forgejo and Headscale verification

### KC-090: Provider-native local lane

The repository MUST provide a disposable local end-to-end lane that uses a
Forgejo server, a Forgejo Actions runner with `enable-openid-connect: true`,
Headscale, the kube-connect broker, and a Kubernetes API that validates Forgejo
OIDC. In this requirement, native means that Forgejo dispatches the workflow
and Forgejo Runner executes it. Faster test layers MAY use mocks. This lane
MUST use the listed components.

The compatibility lock MUST pin the Forgejo Helm chart source, version, and
digest, plus Forgejo and Forgejo Runner image versions and digests.

### KC-091: Two-cluster isolation

The local lane MUST separate management services from the target Kubernetes
API. Forgejo, its runner, Headscale, and the broker MUST run in a management
cluster; OIDC authentication and RBAC fixtures MUST run in a distinct target
cluster. The runner MUST NOT receive a target-cluster service-account token,
target RBAC, or a direct route to the target API.

While an independent check confirms that the target API is healthy, the runner
MUST fail a TCP or TLS connection to its underlay address before enrollment.
After enrollment, the runner MUST reach that API only through its
Headscale-managed overlay address. Test evidence MUST name both attempted
endpoints and distinguish a routing or policy denial from DNS or service
failure.

### KC-092: Headscale chart lock

The management cluster MUST install Headscale from
`oci://ghcr.io/gabe565/charts/headscale`. A machine-readable compatibility lock
MUST pin the Helm chart version, resolved OCI manifest digest, enabled chart
dependency versions and digests, and all rendered container image digests. The
initial candidate baseline is chart `0.16.0` with application version
`v0.25.0`. Before accepting the baseline, the fixture MUST verify its metadata
and record its OCI digest. Changing either version MUST update the lock and
rerun the native end-to-end lane. Compatibility data MUST identify the chart
as community-maintained.

### KC-093: Headscale configuration

The Headscale release MUST persist control-plane state, use SQLite on its
persistent volume, and disable the optional PostgreSQL subchart. It MUST
configure its HTTPS server URL, a distinct MagicDNS base domain, and ACL
policy. TLS MUST terminate with a cert-manager-managed certificate.
Broker-created pre-authentication keys MUST remain one-use, ephemeral, and
constrained by administrator-owned target mappings as required by KC-052 and
KC-053. The action and broker APIs MUST NOT expose authorization-related Helm
values.

### KC-094: Cert-manager ownership

Before the lane starts, the management cluster MUST have the cert-manager
version recorded in the compatibility lock. The harness MUST verify that its
APIs and controllers are ready. If either check fails, the harness MUST stop
before mutating the cluster. Project charts and fixtures MUST NOT embed,
install, upgrade, or uninstall cert-manager. They MAY create namespaced
`Issuer` and `Certificate` resources that reference the lab trust root. The
harness MUST distribute the resulting CA trust to every client that needs it
and MUST remove only the certificate resources it owns.

### KC-095: Forgejo runner containment

The local Forgejo Actions runner MUST be ephemeral and repository-scoped. If
the test runner uses a privileged Docker-in-Docker backend, it MUST run in a
dedicated namespace with automatic Kubernetes service-account token mounting
disabled and no Kubernetes RBAC. It is disposable test infrastructure and MUST
NOT be presented as a production deployment.

### KC-096: End-to-end evidence

The local lane MUST dispatch a real protected-ref Forgejo workflow and verify
separate audiences, broker replay rejection, Headscale enrollment, successful
authentication, allowed and denied namespace RBAC operations, credential
redaction, and cleanup. Evidence MUST identify the locked Forgejo, runner,
Headscale chart, Headscale application, Kubernetes, and cert-manager versions
without recording credentials.

## 12. References

- [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119)
- [GitHub Actions OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)
- [GitHub OIDC with reusable workflows](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-with-reusable-workflows)
- [Forgejo Actions OIDC](https://forgejo.org/docs/latest/user/actions/security-openid-connect/)
- [Forgejo Actions reference](https://forgejo.org/docs/latest/user/actions/reference/)
- [Tailscale GitHub Action](https://tailscale.com/docs/integrations/github/github-action)
- [Tailscale daemon reference](https://tailscale.com/docs/reference/tailscaled)
- [gabe565 Headscale Helm chart](https://artifacthub.io/packages/helm/gabe565/headscale)
- [gabe565 chart source](https://github.com/gabe565/charts/tree/main/charts/headscale)
- [Headscale pre-authenticated keys](https://headscale.net/development/usage/getting-started/#pre-authenticated-key)
- [Kubernetes authentication](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [Kubernetes client authentication v1](https://kubernetes.io/docs/reference/config-api/client-authentication.v1/)
