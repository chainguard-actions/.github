# Chainguard Actions

Chainguard Actions is a set of hardened drop-in replacements for popular GitHub Actions to protect your CI/CD pipelines from supply chain attacks. Each action has the same inputs and outputs as the community version. What changes on your end is the organization referenced in your workflow configuration's `uses:` line.

Coverage spans GitHub first-party (`actions/*`), cloud-provider (`aws-actions/*`, `azure/*`, `google-github-actions/*`), Docker, HashiCorp, security tools (Trivy, Grype, CodeQL, Semgrep), and the high-risk long tail of community actions.

Each hardened action:

- Is built from source and run through our rule-based + AI-evaluated hardening pipeline
- Has every internal `uses:` and container image reference pinned to an immutable SHA
- Ships with a `HARDENING.md` report documenting exactly what was checked and fixed
- Is re-reviewed and re-hardened whenever upstream publishes a new version or Chainguard adds a new rule

Chainguard Actions is designed to prevent common threats like tag hijacking, dependency confusion, `pull_request_target` abuse, and secret exfiltration.

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
