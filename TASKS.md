# kube-connect tasks

This checklist executes the projected commits in `PLAN.md`. Requirement IDs
refer to `REQUIREMENTS.md`. A task is complete only when its verification item
passes.

## Initial design review

- [x] **T-001 — Define the public boundary.** Describe direct Kubernetes OIDC,
  ephemeral private networking, supported providers, and non-goals.
  Requirements: KC-001 through KC-005.
  Verification: README, requirements, plan, and tasks agree.
- [x] **T-002 — Record the threat-driven requirements.** Give every normative
  behavior a stable RFC 2119 requirement ID.
  Requirements: KC-001 through KC-085.
  Verification: each planned milestone cites its governing requirements.
- [x] **T-003 — Draw the implementation history.** Order foundation, identity,
  networking, composition, and operations as conventional commits.
  Verification: the graph is dependency-ordered and predicts v0.1.0.

## `chore(dev): add reproducible toolchain`

- [ ] **T-010 — Add devenv v2 configuration.** Pin Node.js 24, package tooling,
  formatting, linting, type checking, testing, and build dependencies.
  Requirements: KC-070, KC-072.
  Verification: a clean shell can enter the environment without global Node.
- [ ] **T-011 — Define named tasks.** Add format, lint, typecheck, unit,
  integration, build, and aggregate check tasks with explicit dependencies.
  Requirements: KC-070.
  Verification: the aggregate task runs the same commands locally and in CI.
- [ ] **T-012 — Add dependency policy.** Configure lockfile enforcement,
  license checks, vulnerability scanning, and update automation.
  Requirements: KC-071, KC-074.
  Verification: unpinned or unlocked production dependencies fail CI.

## `test(ci): add native provider matrices`

- [ ] **T-020 — Add GitHub fixture workflow.** Run static checks and the OIDC
  provider contract suite without external credential exchange.
  Requirements: KC-002, KC-012, KC-080.
  Verification: the workflow passes on the minimum supported GitHub runner.
- [ ] **T-021 — Add Forgejo fixture workflow.** Use
  `enable-openid-connect: true` and run the same contract suite on a native
  Forgejo runner.
  Requirements: KC-002, KC-013, KC-072, KC-080.
  Verification: the workflow passes on the declared Forgejo/runner versions.
- [ ] **T-022 — Publish compatibility data.** Record tested CI, runner,
  Kubernetes, Tailscale, and Headscale versions in a machine-readable file and
  rendered document.
  Requirements: KC-085.
  Verification: CI rejects an advertised matrix cell without a matching test.

## `feat(oidc): normalize GitHub and Forgejo`

- [ ] **T-030 — Define the provider contract.** Model explicit-audience token
  requests and normalized claims as typed inputs and outputs.
  Requirements: KC-010, KC-011, KC-014.
  Verification: contract tests run against GitHub and Forgejo fixtures.
- [ ] **T-031 — Implement GitHub OIDC.** Request tokens through the native
  runner endpoint, detect missing permission, and normalize immutable claims.
  Requirements: KC-012, KC-014, KC-015.
  Verification: positive and missing-permission tests pass.
- [ ] **T-032 — Implement Forgejo OIDC.** Request tokens through the native
  runner endpoint, detect missing enablement, and normalize Forgejo claims.
  Requirements: KC-013, KC-014, KC-015.
  Verification: positive and missing-enablement tests pass.
- [ ] **T-033 — Enforce event policy.** Reject pull-request events by default
  before requesting downstream credentials.
  Requirements: KC-016.
  Verification: fork and same-repository pull-request fixtures are rejected.
- [ ] **T-034 — Add credential redaction.** Mask OIDC values immediately and
  sanitize errors, traces, snapshots, and debug output.
  Requirements: KC-063, KC-064.
  Verification: seeded canary credentials never appear in captured logs.

## `feat(kube): generate OIDC exec kubeconfig`

- [ ] **T-040 — Validate cluster inputs.** Validate HTTPS API server, CA data,
  audience, context, namespace, and safe filesystem names.
  Requirements: KC-060, KC-061, KC-064.
  Verification: malformed and conflicting inputs fail before writing files.
- [ ] **T-041 — Implement the exec helper.** Request a fresh
  Kubernetes-audience token and print `ExecCredential` v1 with expiry.
  Requirements: KC-011, KC-023, KC-024.
  Verification: no token exists in files and sequential invocations refresh.
- [ ] **T-042 — Generate isolated kubeconfig.** Write owner-only temporary
  files, set `interactiveMode: Never`, context, namespace, CA, and proxy
  placeholder.
  Requirements: KC-023 through KC-026.
  Verification: schema inspection and a fake API-server authentication test
  pass.
- [ ] **T-043 — Document cluster trust.** Provide generic structured JWT
  authenticator and namespace RBAC examples with immutable claim policy.
  Requirements: KC-020 through KC-022.
  Verification: examples validate against supported Kubernetes APIs and create
  no cluster-wide binding by default.

## `feat(broker): issue constrained Headscale keys`

- [ ] **T-050 — Define broker API and policy schema.** Accept only an OIDC
  bearer token and symbolic target; validate all policy fields.
  Requirements: KC-050, KC-052, KC-055.
  Verification: raw tags, users, and expiry overrides are rejected.
- [ ] **T-051 — Verify provider tokens.** Discover issuers, validate signatures
  and time claims, and apply immutable repository/workflow/event/ref policy.
  Requirements: KC-051, KC-058.
  Verification: the complete negative claim matrix passes.
- [ ] **T-052 — Prevent replay.** Atomically reserve issuer, audience, and
  token identity (`jti` or signed-token digest) until token expiry.
  Requirements: KC-054.
  Verification: concurrent duplicate requests yield exactly one issuance.
- [ ] **T-053 — Mint constrained keys.** Use the fixed target mapping to issue
  one-use ephemeral keys with the minimum computed expiry.
  Requirements: KC-052, KC-053, KC-056.
  Verification: a fake Headscale API observes no caller-controlled policy.
- [ ] **T-054 — Add safe operations.** Implement rate limits, redacted audit
  events, cache-prevention headers, bounded JWKS caching, health checks, and
  fail-closed dependencies.
  Requirements: KC-055 through KC-058, KC-063.
  Verification: dependency and canary-secret fault tests pass.

## `feat(network): connect through Headscale`

- [ ] **T-060 — Exchange OIDC at the broker.** Request a network-audience
  token, call the broker, mask the returned key, and validate expiry.
  Requirements: KC-011, KC-050, KC-055, KC-063.
  Verification: request and response credentials do not appear in logs.
- [ ] **T-061 — Start isolated userspace networking.** Launch pinned
  `tailscaled` below the runner temporary directory with unique sockets, state,
  hostname, and loopback SOCKS5 port.
  Requirements: KC-030, KC-031, KC-035, KC-071.
  Verification: two concurrent fake jobs do not share state or ports.
- [ ] **T-062 — Enroll and wait.** Join the configured Headscale server with
  the one-use key and block until the configured target is reachable.
  Requirements: KC-030, KC-032, KC-050.
  Verification: a disposable Headscale integration test reaches a private
  endpoint.
- [ ] **T-063 — Register cleanup first.** Install post cleanup before
  enrollment, then logout, terminate, and remove temporary state.
  Requirements: KC-034.
  Verification: success, failure, and supported cancellation tests leave no
  daemon or state.

## `feat(network): add Tailscale federation`

- [ ] **T-070 — Implement federated enrollment.** Exchange the provider OIDC
  identity through Tailscale workload identity federation without an OAuth
  secret or auth key input.
  Requirements: KC-040, KC-041.
  Verification: tests reject static credential inputs in production mode.
- [ ] **T-071 — Map symbolic targets.** Derive only configured tags and rely on
  Tailscale tag ownership as the authorization boundary.
  Requirements: KC-041, KC-042.
  Verification: a target cannot request a tag outside the federated identity's
  ownership.
- [ ] **T-072 — Verify both CI providers.** Run native GitHub/Tailscale and
  Forgejo/Tailscale enrollment tests.
  Requirements: KC-043, KC-082.
  Verification: both matrix cells publish passing evidence and version data.

## `feat(action): compose the portable action`

- [ ] **T-080 — Define action metadata.** Publish the validated common and
  namespaced inputs, safe outputs, Node.js 24 runtime, main entrypoint, and
  post entrypoint.
  Requirements: KC-001 through KC-003, KC-060 through KC-064, KC-072.
  Verification: metadata tests cover defaults, conflicts, and output safety.
- [ ] **T-081 — Compose adapters.** Detect or select the CI provider, select
  the network provider, start connectivity, and finish kubeconfig proxy
  configuration.
  Requirements: KC-001 through KC-003, KC-011, KC-032.
  Verification: all four provider combinations pass orchestration tests.
- [ ] **T-082 — Add kernel-mode escape hatch.** Detect Linux privileges and
  isolate privileged setup behind an explicit input.
  Requirements: KC-033.
  Verification: default mode is unprivileged and missing privileges fail
  before mutation.
- [ ] **T-083 — Harden packaging.** Bundle action code, pin binary manifests,
  generate checksums and SBOM, and verify no development-only files are needed
  at runtime.
  Requirements: KC-071, KC-072, KC-074.
  Verification: a source-free action fixture runs from the committed bundle.

## `feat(action): verify scoped cluster access`

- [ ] **T-090 — Add optional capability check.** Accept explicit verb,
  resource, namespace, and optional resource name, then invoke
  `kubectl auth can-i`.
  Requirements: KC-027.
  Verification: allowed and denied checks report clearly without changing
  RBAC.
- [ ] **T-091 — Run the native end-to-end matrix.** Verify GitHub/Forgejo
  across Tailscale/Headscale against a dedicated Kubernetes test cluster.
  Requirements: KC-082, KC-083.
  Verification: every advertised cell authenticates, exercises scoped RBAC,
  and cleans up.

## `docs: publish operator runbooks`

- [ ] **T-100 — Write cluster setup.** Cover structured JWT authenticators,
  claim mappings, namespace RBAC, audience rotation, and revocation.
  Requirements: KC-021, KC-022, KC-084.
  Verification: a clean test cluster can follow the runbook.
- [ ] **T-101 — Write Tailscale setup.** Cover federated identity, immutable
  claims, tag ownership, target configuration, and troubleshooting.
  Requirements: KC-041, KC-084.
  Verification: no static production credential is required.
- [ ] **T-102 — Write Headscale setup.** Cover broker deployment, target
  policy, API secret rotation, replay storage, audit review, rate limits, and
  emergency revocation.
  Requirements: KC-050 through KC-058, KC-084.
  Verification: an operator can rotate the Headscale credential without
  changing consumer workflows.
- [ ] **T-103 — Finalize consumer examples.** Publish SHA-pinned GitHub and
  Forgejo workflows for all supported matrix cells.
  Requirements: KC-073, KC-082, KC-085.
  Verification: every example is exercised by a native end-to-end test.
- [ ] **T-104 — Prepare v0.1.0.** Confirm requirement traceability,
  compatibility evidence, checksums, SBOM, provenance, and release notes.
  Requirements: KC-074, KC-080 through KC-085.
  Verification: the release checklist has no waived security requirement.
