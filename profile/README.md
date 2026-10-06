# Chainguard Actions

Chainguard Actions are hardened drop-in replacements for popular GitHub Actions. Each action preserves the same inputs and outputs as the upstream version, but has been examined and revised to better protect your CI/CD pipelines from supply chain attacks. In most cases the only change to your workflow is the name of the action in the `uses:` line.

This organization holds the catalog. **The documentation lives at [edu.chainguard.dev](https://edu.chainguard.dev/chainguard/actions/overview/)** — start there for prerequisites, setup, migrating a whole repository, and how hardening works.

Coverage spans GitHub first-party (`actions/*`), cloud-provider (`aws-actions/*`, `azure/*`, `google-github-actions/*`), Docker, HashiCorp, and security tools actions (Trivy, Grype, CodeQL, Semgrep), as well as a growing catalog of community actions.

## What hardening does

Each hardened action:

- Is pulled from the upstream source at a pinned commit, then reviewed by a static ruleset and an AI-powered analysis pass
- Has every internal `uses:` and container image reference pinned to an immutable SHA digest
- Ships with a `HARDENING.md` report recording every finding, its location, and how it was fixed
- Ships with a signed SLSA provenance attestation naming the upstream commit and the ruleset version applied
- Is re-reviewed and re-hardened when upstream publishes a new version, and re-evaluated across the whole catalog whenever the hardening ruleset changes

Chainguard Actions protect against common threats including tag hijacking, dependency confusion, `pull_request_target` abuse, and secret exfiltration.

Two boundaries worth knowing up front. Hardening doesn't change the packages an action bundles or installs: a JavaScript action ships the same `dist/` bundle as its upstream release, with the same dependency versions, and an action built from a Dockerfile uses the same base image as upstream. Packages and binaries an action downloads while it runs are outside the review. And nested actions are swapped for hardened copies only in composite actions, and only when a hardened copy exists for that exact upstream commit, so read the `action.yml` on the version branch you plan to use to see what it references. Both are covered in full in [the documentation](https://edu.chainguard.dev/chainguard/actions/overview/#transitive-dependencies).

## Finding an action

Repository names are prefixed with the upstream organization, so `tj-actions/changed-files` becomes [`chainguard-actions/tj-actions-changed-files`](https://github.com/chainguard-actions/tj-actions-changed-files). This keeps two different sources of a `changed-files` action from clashing.

Search this organization, or list the catalog with `chainctl`:

```shell
chainctl actions catalog list --upstream-owner=tj-actions
```

To inventory what your own repository uses, including actions reached through other actions:

```shell
chainctl actions discover <owner>/<repo> --recursive
```

If an action you need isn't here, [request it](https://github.com/chainguard-actions/.github/issues/new?template=new-action.yml).

## Using an action

Reference a hardened action by commit SHA, and pair the pin with Dependabot or Renovate:

```yaml
- uses: chainguard-actions/tj-actions-changed-files@<sha> # v47
```

A commit SHA is the only immutable reference in this catalog. Tags are mutable by design: Chainguard re-hardens published versions in place and moves the tag when it does, including fully qualified patch tags. Pairing the pin with Dependabot or Renovate means you receive each re-hardening as a pull request your own CI validates, rather than having it swapped in underneath you.

Two things that will trip you up otherwise:

- **Don't reference an action with `@main`.** The main branch of each repository holds only `README.md`, `LICENSE_CHAINGUARD`, and `source.json`. The action itself lives on the version branches, so `@main` cannot resolve.
- **Read `HARDENING.md` on the version branch you plan to use**, not on `main`. The report and the attestation are per-version.

If your GitHub organization restricts which actions can run, add `chainguard-actions/*` to the allowed patterns under **Settings > Actions > General > Allow select actions**. Without it, workflows fail with a policy error on first run.

## Example migrations

Each example changes only the `uses:` line. The `with:` block, the inputs, and the outputs stay exactly as they were. Replace `<sha>` with the commit SHA of the version you want, which you can find with `gh api repos/chainguard-actions/<name>/commits/<tag> --jq '.sha'`.

### Container scanning with `aquasecurity/trivy-action`

`@master` in the community version is especially risky, because it runs whatever upstream pushed most recently. Moving to the hardened version and pinning a SHA gives you a reference that can't change under you.

```yaml
# Before
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-org/my-image:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'

# After
- uses: chainguard-actions/aquasecurity-trivy-action@<sha>
  with:
    image-ref: 'my-org/my-image:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

### Docker image builds with `docker/build-push-action`

`docker/login-action` runs with registry credentials on every build, which makes it exactly the kind of secret-adjacent action worth hardening.

```yaml
# Before
- uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

- uses: docker/build-push-action@v6
  with:
    context: .
    push: true
    tags: my-org/my-image:${{ github.sha }}

# After
- uses: chainguard-actions/docker-login-action@<sha>
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

- uses: chainguard-actions/docker-build-push-action@<sha>
  with:
    context: .
    push: true
    tags: my-org/my-image:${{ github.sha }}
```

### Cloud credentials with `aws-actions/configure-aws-credentials`

This handles the OIDC exchange for your AWS short-lived credentials. A compromise here would give an attacker direct access to your AWS environment, which makes it one of the most consequential actions to harden.

```yaml
# Before
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/ci-role
    aws-region: us-east-1

# After
- uses: chainguard-actions/aws-actions-configure-aws-credentials@<sha>
  with:
    role-to-assume: arn:aws:iam::123456789012:role/ci-role
    aws-region: us-east-1
```

### SBOM generation with `anchore/sbom-action`

This generates SPDX and CycloneDX SBOMs with Syft, widely used for SLSA and compliance attestations.

```yaml
# Before
- uses: anchore/sbom-action@v0
  with:
    image: my-org/my-image:${{ github.sha }}
    format: spdx-json

# After
- uses: chainguard-actions/anchore-sbom-action@<sha>
  with:
    image: my-org/my-image:${{ github.sha }}
    format: spdx-json
```

Inputs, outputs, and behavior are almost always identical to the upstream version, so no other workflow changes are typically needed. In rare cases hardening requires a change to inputs, outputs, or behavior, and those changes are recorded in that action's `HARDENING.md`. Read it before you migrate.

## What's in a repository

The main branch is a landing page. It holds `README.md`, `LICENSE_CHAINGUARD`, and `source.json`, the manifest naming the upstream owner and repository. Each version branch has its own `source.json` that adds the upstream version and the exact commit the hardened copy was built from.

Each version branch holds the hardened action itself: `action.yml` or `action.yaml`, the `HARDENING.md` report, `LICENSE_CHAINGUARD`, the upstream action's own files, and — for releases published since signing began — an `attestations/` directory carrying the provenance attestation.

## Request a new action

To request an action that isn't in the catalog, [open a new action issue](https://github.com/chainguard-actions/.github/issues/new?template=new-action.yml).

## Report an issue

If an action isn't working as expected, [open an action issue](https://github.com/chainguard-actions/.github/issues/new?template=action-issue.yml) with the action reference, a description of the problem, and steps to reproduce.

## Learn more

- [Chainguard Actions documentation](https://edu.chainguard.dev/chainguard/actions/overview/)
- [Telemetry and privacy](https://edu.chainguard.dev/chainguard/actions/telemetry/)
- [Chainguard Actions product page](https://www.chainguard.dev/actions)
- For other questions, reach out to interest@chainguard.dev
