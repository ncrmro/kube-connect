# Contributing

`kube-connect` is currently a design repository. The first review establishes
the product boundary, threat model, requirements, and implementation order.
Code contributions should wait until that contract is accepted.

## Source of truth

The project uses four linked documents:

1. `REQUIREMENTS.md` defines normative behavior with stable `KC-*` identifiers.
2. `PLAN.md` shows the intended commit history, dependencies, and release
   boundary.
3. `TASKS.md` breaks planned commits into verifiable work.
4. `README.md` explains the supported user experience without replacing the
   normative requirements.

When behavior changes, update all affected documents in the same pull request.
Do not silently weaken a `MUST` into an implementation detail.

## Proposing a change

Before implementation:

1. identify the affected requirement IDs;
2. describe the threat or user need;
3. update the normative text and acceptance criteria;
4. place the work in the projected git graph;
5. add or adjust the corresponding tasks; and
6. call out provider-specific behavior for both GitHub and Forgejo.

If a proposal changes a trust boundary, include a negative test showing how
the previous or unsafe behavior is rejected.

## Development workflow

The implementation will use a reproducible devenv v2 environment once the
foundation milestone begins. Until then, documentation changes can be checked
with:

```sh
git diff --check
```

Future project commands must be exposed as named devenv tasks. Contributors
should not require globally installed language runtimes beyond Nix and devenv.

Use Conventional Commit subjects. A planned commit in `PLAN.md` should become
one real commit with the same subject unless the plan is deliberately amended
first. Do not squash a multi-commit planned span.

## Pull requests

A pull request should:

- stay within one planned milestone or explain why the graph changed;
- reference each implemented `KC-*` requirement;
- include positive and negative tests in the same commit as behavior;
- update public examples when inputs or outputs change;
- avoid environment-specific hostnames, account IDs, repositories, or roles;
- pin third-party actions and downloaded binaries; and
- document any compatibility difference between GitHub and Forgejo.

Reviewers should evaluate the security boundary before implementation style.
In particular, verify issuer and audience checks, immutable identity claims,
event and ref restrictions, target-to-tag mapping, replay prevention, secret
redaction, cleanup, and the absence of action-created RBAC.

## Security and secrets

Never commit or paste:

- OIDC JWTs;
- Headscale or Tailscale keys;
- broker API keys;
- kubeconfigs containing credentials;
- runner registration tokens; or
- real internal hostnames and identity mappings.

Use synthetic examples such as `example.com`. Tests should use signed fixtures
created for the test suite, never captured production tokens.

Do not print bearer tokens, pre-auth keys, authorization headers, or generated
kubeconfig contents. Failure output should identify the rejected policy
condition without echoing credentials.

## Compatibility

GitHub Actions and Forgejo Actions are similar but not interchangeable.
Provider adapters must be tested on their native runners. A successful GitHub
test is not evidence that Forgejo supports the same action runtime, workflow
syntax, context field, or cross-host action reference.

The first release targets Linux. Changes for macOS or Windows require an
explicit requirements and plan update.
