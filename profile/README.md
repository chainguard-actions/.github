# Chainguard Actions

Chainguard Actions is a set of hardened drop-in replacements for popular GitHub Actions to protect your CI/CD pipelines from supply chain attacks. Each action has the same inputs and outputs as the community version. What changes on your end is the organization referenced in your workflow configuration's `uses:` line.

Coverage spans GitHub first-party (`actions/*`), cloud-provider (`aws-actions/*`, `azure/*`, `google-github-actions/*`), Docker, HashiCorp, security tools (Trivy, Grype, CodeQL, Semgrep), and the high-risk long tail of community actions.

Each hardened action:

- Is built from source and run through our rule-based + AI-evaluated hardening pipeline
- Has every internal `uses:` and container image reference pinned to an immutable SHA
- Ships with a `HARDENING.md` report documenting exactly what was checked and fixed
- Is re-reviewed and re-hardened whenever upstream publishes a new version or Chainguard adds a new rule

Chainguard Actions is designed to prevent common threats like tag hijacking, dependency confusion, `pull_request_target` abuse, and secret exfiltration.

## Prerequisites

Requires:

- `chainctl` v0.2.261+
- An organization in the Chainguard platform that you own or can administer

## Preliminary steps

You must do these things before anything else:

- Log in to Chainguard
- Enable Chainguard Actions for your organization

### Log in to Chainguard

Authenticate using `chainctl`:

```shell
chainctl auth login
```

Confirm you are logged in and view your organization details.

```shell
chainctl auth status
```

### Enable Chainguard Actions for your organization

Create the Chainguard Actions entitlement to enable access to the hardened actions hosted at `github.com/chainguard-actions`.

```shell
chainctl actions entitlements create --parent $ORGANIZATION
```

This will return a message like this ($ENTITLEMENT_ID will show your actual ID, unlike in the example).

```output
Enabled Actions product for org chainguard.edu ($ENTITLEMENT_ID) [entitlement id: $ENTITLEMENT_ID]
```

Confirm your entitlement with:

```shell
chainctl actions entitlements list --parent $ORGANIZATION
```

This will return a table like this:

```output
                    ID                    |         CREATED
------------------------------------------|-------------------------
 $ENTITLEMENT_ID                          | 2026-05-19 17:33:24 UTC
```

## Basic usage

Edit your workflow's YAML configuration file, replacing the current source of an action with `chainguard-actions`, like this.

```yaml
- uses: chainguard-actions/<action-name>@main
```

## Configure your workflows to use Chainguard Actions

Take the following steps to switch your workflow over to using Chainguard Actions.

1. Inventory the community actions you currently use.

    Run this from the root of your repo to get a deduplicated list of every `uses:` line across every workflow.

    ```shell
    grep -rhE "uses:\s*[^@]+@" .github/workflows/ | sort -u
    ```

1. Check the catalog of Chainguard Actions for each one.

    Browse [the Chainguard Actions repository](https://github.com/chainguard-actions) or use the GitHub UI search. Match by denoting the origin org and action. Example: if you are currently using `tj-actions/changed-files` → look for `org:chainguard-actions tj-actions/changed-files`.

    If the action isn't in the catalog, [open an issue](https://github.com/chainguard-actions/.github/issues/new?template=new-action.yml) to request it and move on — new actions are typically added within days once triaged.

1. Replace the `uses:` line in each workflow.

    In your GitHub Actions workflow YAML, change only the organization; keep the action name. Then, instead of using the mutable release, we recommend you pin to the SHA with the tag in a comment. For example:

    ```yaml
    # Before
    - uses: tj-actions/changed-files@v47
    ```

    Placing the tag in a comment after the SHA makes it possible for Dependabot and Renovate and human reviewers to handle upgrades.

    ```yaml
    # After
    - uses: chainguard-actions/changed-files@<SHA> # v47
    ```

    Leave the original version as a YAML comment so Dependabot/Renovate and human reviewers can read it for history.

    ```yaml
    - uses: chainguard-actions/changed-files@<SHA> # v47
    # originally - uses: tj-actions/changed-files@v47
    ```

1. Update your allowed-actions list.

    If your GitHub org or repo restricts which actions can run (with Settings → Actions → General → Allow select actions), add `chainguard-actions/*` to the allowed patterns. Without this, workflows will fail with a policy error on first run.

1. Commit, open a PR, and watch CI pass.

    The action's inputs, outputs, and behavior are almost always identical to upstream, so no other workflow changes should be needed.

    > **Note:** Always read the `HARDENING.md` file for the Chainguard Action before migrating in case there is a rare instance where something had to be changed for the hardening process. Any changes to inputs, outputs, or behavior will be documented in this file.

    If something does break, [file an issue](https://github.com/chainguard-actions/.github/issues/new?template=action-issue.yml) with a reproducer.

## Find the SHA for a specific release

It is a best practice to use the SHA instead of a tag. We've said that and shown where to put the SHA. But what if you don't know how to find the SHA. That's what this section is about.

Returning to our previous example, we put this in our workflow YAML file.

```yaml
- uses: chainguard-actions/changed-files@<SHA> # v47
```

When we say *SHA* we are referring to the specific commit hash of a published action.

Here's a quick way to find the SHA that correlates to the v47 release of our action.

```shell
gh api repos/chainguard-actions/changed-files/commits/v47 --jq '.sha'
```

```response
25a1eb5aa40568ec6f8c0e58f2e809ef4270ebfa
```

If you only need the short version, use:

```shell
gh api repos/chainguard-actions/changed-files/commits/v47 --jq '.sha[:7]'
```

```response
25a1eb5
```

Here's what our example looks like with the full SHA:

```yaml
- uses: chainguard-actions/changed-files@25a1eb5aa40568ec6f8c0e58f2e809ef4270ebfa # v47
```

See GitHub to learn more about the [commit hash for GitHub Actions](https://github.com/marketplace/actions/commit-hash).

## View the actions you are currently using in a repo

Direct `chainctl` to your Git repo to see what actions it currently uses. `chainctl` will scan every workflow and composite action and list what they depend on (works transitively).

```shell
chainctl actions discover $GIT_ORGANIZATION/$REPO
```

Expected output:

```output
    scanning $GIT_ORGANIZATION/$REPO for workflows and actions
               ACTION                    | REQUESTED | USED BY
    -------------------------------------|-----------|---------
     actions/checkout                    | v4        | 1
     chainguard-actions/actions-checkout | v6.0.2    | 1

    2 actions, 0 container images

```

The repo in our example has two workflows that do the same thing. One uses the upstream action, one uses the hardened equivalent.

## Example migrations

Here are a few examples of migrations to help you get started with yours. Each example is focused on one specific attribute for clarity.

### The compromise using `tj-actions/changed-files`

All that happens here is that we change the `uses:` line. Even with the same `with:` block and the same outputs, and still using the mutable version tag, if there was an upstream compromise, it would never hit the Chainguard repo and you would be protected. This is the quick and easy implementation, the compromise, that doesn't yet implement pinning by SHA and it still provides significant protection.

```yaml
# Before
- uses: tj-actions/changed-files@v45
  with:
    files: |
      src/**
      tests/**

# After
- uses: chainguard-actions/changed-files@v45>
  with:
    files: |
      src/**
      tests/**
```

### Container screening example using `aquasecurity/trivy-action`

`@master` in the community version is especially risky — it always runs whatever the upstream just pushed. The hardened version lets you pin to a SHA and still get automatic re-hardening behind the scenes.

```yaml
# Before
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-org/my-image:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'

# After
- uses: chainguard-actions/trivy-action@v0.28
  with:
    image-ref: 'my-org/my-image:${{ github.sha }}'
    severity: 'CRITICAL,HIGH'
    exit-code: '1'
```

### Docker image builds using `docker/build-push-action`

`docker/login-action` gets invoked with registry credentials on every build — exactly the kind of secret-adjacent action customers want hardened.

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
- uses: chainguard-actions/login-action@<sha>
  with:
    username: ${{ secrets.DOCKER_USERNAME }}
    password: ${{ secrets.DOCKER_PASSWORD }}

- uses: chainguard-actions/build-push-action@<sha>
  with:
    context: .
    push: true
    tags: my-org/my-image:${{ github.sha }}
```

### Cloud credentials using `aws-actions/configure-aws-credentials`

This handles OIDC exchange for your AWS short-lived credentials. Any compromise here would give an attacker direct access to your AWS environment — one of the most consequential actions to harden.

```yaml
# Before
- uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/ci-role
    aws-region: us-east-1

# After
- uses: chainguard-actions/configure-aws-credentials@<sha>
  with:
    role-to-assume: arn:aws:iam::123456789012:role/ci-role
    aws-region: us-east-1
```

### SBOM generation using `anchore/sbom-action`

This generates SPDX/CycloneDX SBOMs with Syft, which is widely used for SLSA/compliance attestations — customers asking about supply-chain provenance consistently ask for this one.

```yaml
# Before
- uses: anchore/sbom-action@v0
  with:
    image: my-org/my-image:${{ github.sha }}
    format: spdx-json

# After
- uses: chainguard-actions/sbom-action@<sha>
  with:
    image: my-org/my-image:${{ github.sha }}
    format: spdx-json
```

## Hardened action repo contents

In each repo for a hardened action, you will find:

- `HARDENING.md` — the authoritative, per-action record of what was checked, what was fixed, and how.
- `action.yml` / `action.yaml` — the hardened action definition (preserving upstream inputs/outputs, with fixes applied).
- `LICENSE_CHAINGUARD` — the Chainguard license attached to the hardened variant.
- `source.json` / `published.json` — (in most repos) manifests pointing at the upstream source and the upstream version being tracked. Not yet universal — some older hardened repos don't have them.

## The continuous re-hardening process

Chainguard Actions are continuously re-hardened like this.

- When upstream publishes a new version, the pipeline re-runs and publishes a new hardened version.
- When the hardening ruleset is updated, affected actions are re-reviewed against the new rules.
- The `HARDENING.md` report is regenerated on every hardening run, with its own Policy SHA pinning the exact set of actions that were applied.


## Request a new action

To request a new action, [open an issue](https://github.com/chainguard-actions/.github/issues/new?template=new-action.yml).

## Report an issue

If an action isn't working as expected, [open an issue](https://github.com/chainguard-actions/.github/issues/new?template=action-issue.yml) and include the action reference, a description of the problem, and steps to reproduce.

## Contact

For questions or issues, reach out to interest@chainguard.dev.

## Learn more

Learn more at [chainguard.dev/actions](https://www.chainguard.dev/actions).
