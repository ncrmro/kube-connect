# AGENTS.md

@CONTRIBUTING.md

## Repository contract

- `REQUIREMENTS.md` is normative. Preserve stable `KC-*` identifiers and use
  RFC 2119 keywords deliberately.
- `PLAN.md` is a projected git history. Time flows upward, planned commit
  subjects become real commit subjects, and planned spans are not squashed.
- `TASKS.md` is the execution checklist. Every implementation task references
  one or more requirements and names a verification step.
- `README.md` is user-facing and must not promise unreleased behavior.

## Architecture essentials

- Keep the root interface provider-neutral. Put GitHub, Forgejo, Tailscale, and
  Headscale differences behind typed adapters.
- Fetch separate OIDC tokens for Kubernetes and network enrollment. Never
  reuse a token across audiences.
- Kubernetes authenticates CI tokens directly. The action must not create or
  modify RBAC.
- Headscale enrollment goes through a broker that validates the CI token and
  returns a one-use, short-lived pre-auth key.
- The Headscale broker owns target-to-tag policy. The caller cannot submit raw
  Headscale tags.
- Tailscale mode uses workload identity federation. Static auth keys are not a
  supported production path.
- Run `tailscaled` in isolated userspace mode under the runner temporary
  directory by default, expose a loopback SOCKS5 proxy, and clean up in the
  action post phase.
- Treat kernel networking as an explicit Linux-only privileged mode.
- Prefer immutable repository and owner IDs in trust decisions. Also validate
  issuer, audience, subject, workflow, event, and ref.
- Reject untrusted pull-request events by default.
- Keep the full local Forgejo and Headscale lane split into management and
  target clusters. Install Headscale from the pinned `gabe565/headscale` OCI
  chart in the management cluster; treat cert-manager as an external
  prerequisite, never as a project subchart.
- Treat any privileged Forgejo Actions runner chart as disposable test
  infrastructure. Disable service-account token mounting, grant it no
  Kubernetes RBAC, and prevent direct target API access.

## Implementation rules

- Use TypeScript for the action and broker unless an accepted requirements
  change says otherwise.
- Use command or service objects with explicit dependencies for non-trivial
  behavior; keep action entrypoints thin.
- Validate all external input at the boundary and fail closed on unknown
  providers, targets, claims, or configuration fields.
- Store temporary files only below the runner temporary directory with
  owner-only permissions.
- Never log OIDC tokens, authorization headers, pre-auth keys, or kubeconfig
  credentials. Register every received secret with the runner masking API
  before further processing.
- Do not expose credentials as action outputs.
- Pin downloaded binaries by version and checksum. Pin third-party actions by
  full commit SHA in tests and examples intended for production.
- Use the repository's devenv v2 tasks for formatting, linting, type checking,
  unit tests, integration tests, and builds once those tasks exist.
- Keep the JavaScript action runtime compatible with GitHub's current Node
  runtime and gate Forgejo runner compatibility independently.

## Testing rules

- Cover each normative requirement with a traceable test or documented
  inspection.
- Include negative tests for wrong issuer, audience, repository ID, workflow,
  event, ref, target, expired token, future token, and replayed token identity.
- Test cleanup after success, action failure, and cancellation where the
  runner permits it.
- Run provider contract tests against GitHub and Forgejo OIDC fixtures.
- Run the native Forgejo and Headscale lane with real OIDC, broker exchange,
  chart-installed Headscale, overlay-only target access, exact RBAC, and
  cleanup assertions.
- Do not claim a provider matrix cell is supported until an end-to-end native
  runner test passes.

## Repository hygiene

- Keep this public repository provider-neutral. Consumer-specific cluster
  names, hostnames, identity mappings, and RBAC belong in private deployment
  configuration.
- Update architecture and user documentation in the same change as directory
  layout or public interface changes.
- Preserve unrelated work in a dirty worktree and stage files explicitly.
