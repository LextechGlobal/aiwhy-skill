# aiwhyskill — maintainer runbook

Operator-facing procedures for cutting new releases, rolling back bad releases, and rotating credentials. End-user install instructions are in [../README.md](../README.md).

## The two-repo model

Launcher code lives in the private repo `LextechGlobal/aiwhy-secure-skill-launcher`. This public repo (`LextechGlobal/aiwhy-skill`) is a **distribution destination** — it does not mirror source code. The only things that land here are:

- `README.md` / `docs/MAINTAINER.md` — user-facing docs (committed manually).
- GitHub Releases with `aiwhyskill.zip` + `SHA256SUMS` assets (created by the `release-launcher` GitHub Action in the private repo).

All code changes happen in the private repo. The release pipeline publishes here via a `contents:write`-scoped fine-grained PAT.

## Release audiences — customer vs engineering

The release workflow classifies tags by shape:

| Tag shape | Audience | Published as | Surfaces in `/releases/latest`? |
|-----------|----------|--------------|----------------------------------|
| `vX.Y.Z` | Customer | Full release with **Friendly Summary** + Changelog body | Yes — end users see it in the update prompt |
| `vX.Y.Z-eng.N` | Engineering | GitHub pre-release, auto-notes only | No — hidden from `/releases/latest` |

Use engineering pre-releases (`-eng.N`) for internal testing, smoke tests of CI changes, or any build that should not prompt customers to upgrade.

## Prerequisites

- Push access to `LextechGlobal/aiwhy-secure-skill-launcher` (the private repo).
- No AWS access needed.
- The `PUBLIC_REPO_DEPLOY_PAT` GitHub Actions secret must be valid (see [PAT rotation](#deploy-pat-rotation-every-90-days)).

## Cutting a customer release

Customer releases require a **version-bump PR** that promotes the `CHANGELOG.md` `[Unreleased]` section. The promoted text ships verbatim to end users as the Friendly Summary in the update prompt — author it for a non-technical reader.

### 1. Open the version-bump PR

From the private repo (`aiwhy-secure-skill-launcher`):

```bash
git checkout -b mcp/bump-vX.Y.Z
```

Make three changes in one commit:

1. `CHANGELOG.md`: rename the top `## [Unreleased]` heading to `## [X.Y.Z] - YYYY-MM-DD`, then insert a fresh `## [Unreleased]` heading above it.
2. `aiwhyskill/SKILL.md`: bump `metadata.version` to `X.Y.Z`.
3. Commit with `chore(aiwhyskill): bump version to X.Y.Z`.

Open the PR. The reviewer's job: read the promoted `## [X.Y.Z]` section as if you were an end user and confirm it reads well. That text is what the launcher will surface in the update prompt.

**Rules enforced by the release workflow:**

- **Customer MAJOR or MINOR (`X.Y.0`)**: the `## [X.Y.Z]` section must be non-empty. The build fails otherwise.
- **Customer PATCH (`X.Y.Z`, Z > 0)**: the section may be empty. The workflow substitutes the canned Friendly Summary `"Bug fixes and improvements."` and proceeds.

### 2. Tag after merge

```bash
git checkout main && git pull
git tag vX.Y.Z
git push origin vX.Y.Z
```

Last-minute copy edits to the promoted `## [X.Y.Z]` Friendly Summary are fine on `main` between merge and tag — the release body is extracted at tag-push time, not at PR-merge time.

### 3. Watch the release

```bash
gh run watch --repo LextechGlobal/aiwhy-secure-skill-launcher
```

On success, the release appears at `https://github.com/LextechGlobal/aiwhy-skill/releases/tag/vX.Y.Z` with `aiwhyskill.zip` + `SHA256SUMS` attached. The "Build release body" step log prints the rendered Friendly Summary preview — eyeball it before the run completes.

### 4. Verify the published release

```bash
curl -L -o /tmp/aiwhyskill.zip https://github.com/LextechGlobal/aiwhy-skill/releases/download/vX.Y.Z/aiwhyskill.zip
curl -L -o /tmp/SHA256SUMS    https://github.com/LextechGlobal/aiwhy-skill/releases/download/vX.Y.Z/SHA256SUMS
cd /tmp && shasum -a 256 -c SHA256SUMS
# Expected: aiwhyskill.zip: OK
```

## Cutting an engineering pre-release

Use these for internal testing builds that should **not** prompt customers to upgrade.

```bash
# From main on the private repo. CHANGELOG.md is NOT modified.
# metadata.version in SKILL.md must match the BASE of the tag
# (e.g., tag v1.3.1-eng.1 -> metadata.version: "1.3.1").
git tag v1.3.1-eng.1
git push origin v1.3.1-eng.1
```

The workflow:
- Publishes with `--prerelease`, so the release is invisible to `/releases/latest`.
- Skips Friendly Summary extraction — engineering releases carry the auto-generated notes plus a `<!-- engineering-release -->` marker.
- Runs the same zip + SHA256SUMS pipeline so internal testers get the full artifact bundle.

## Cutting a required (Minimum-Required) release

Use this ONLY for security-affecting fixes. It forces every installed launcher that predates the release to stop working on `list` / `launch` until the user upgrades — real cost to customers.

A required release still needs the full CHANGELOG dance: open the same version-bump PR, promote `[Unreleased]`, merge, and tag. The only difference is how you cut the release: don't rely on the push trigger; use `workflow_dispatch` so you can pass `minimum_required`.

```bash
gh workflow run release-launcher.yml \
  --repo LextechGlobal/aiwhy-secure-skill-launcher \
  --ref main \
  -f tag=vX.Y.Z \
  -f minimum_required=X.Y.Z
```

The `minimum_required` input prepends a `Minimum-Required: X.Y.Z` line to the release body, which the launcher's `_parse_release_body` strips out and uses to decide whether to block.

**Sequencing tip:** if you've already pushed the tag and the auto-release ran, you can't retroactively add `Minimum-Required`. Either delete the just-published release and dispatch with `minimum_required`, or plan the required-tag flow up front and skip `git push origin <tag>` — go straight to `gh workflow run` with the tag as an input.

## Dry-running a release (no publish)

Validates the pipeline without creating a real release. Use this after any change to `release-launcher.yml`.

**Pre-merge (against the version-bump PR branch):**

```bash
gh workflow run release-launcher.yml \
  --repo LextechGlobal/aiwhy-secure-skill-launcher \
  --ref mcp/bump-vX.Y.Z \
  -f tag=vX.Y.Z \
  -f dry_run=true
```

**Post-merge (against main):**

```bash
gh workflow run release-launcher.yml \
  --repo LextechGlobal/aiwhy-secure-skill-launcher \
  --ref main \
  -f tag=vX.Y.Z \
  -f dry_run=true
```

Both variants require SKILL.md `metadata.version` to match the `tag` base. No throwaway branches, no SKILL.md tampering.

Inspect the artifact:

```bash
gh run download --repo LextechGlobal/aiwhy-secure-skill-launcher --name release-bundle-vX.Y.Z
cd release-bundle-vX.Y.Z
shasum -a 256 -c SHA256SUMS
# Expected: aiwhyskill.zip: OK
```

The workflow run's "Build release body" step log renders the final body (with `## Friendly Summary`) for preview. Check it before cutting the real tag.

Dispatches from a branch never publish, regardless of the `dry_run` value — branches are publish-blocked at the job's `if` gate. `dry_run=true` simply keeps the audit trail honest and skips the `generate-notes` API call.

## Rolling back a bad release

If a published release is broken (build issue, bug in the bundled code):

```bash
gh release delete vX.Y.Z --repo LextechGlobal/aiwhy-skill --yes
git push --delete origin vX.Y.Z   # from the private repo
git tag -d vX.Y.Z                  # local cleanup
```

Effects:
- Customers on older versions: unaffected. The launcher's `--check-update` stops offering vX.Y.Z as soon as the release is gone.
- Customers already on vX.Y.Z: stay on it until the next good release lands. If the bad version is security-affecting, cut the next good release as a **required** update (`minimum_required` > `X.Y.Z`) so the bad version is blocked.

If you also want to undo the CHANGELOG promotion locally before cutting the next version, revert the `chore(aiwhyskill): bump version to X.Y.Z` commit (or roll forward with a new version-bump PR — usually cleaner).

## Incident response: disable update check

If a customer release ships with a broken `--check-update` that's flooding users with bad prompts, the kill-switch is an environment variable:

```bash
# In the Cowork session / operator's shell:
export AIWHYSKILL_DISABLE_UPDATE_CHECK=1
```

Behavior with the flag set:
- `--check-update` short-circuits to `status=disabled` and SKILL.md treats that silently (no prompt).
- `--list`, `--skill`, `--cleanup`, and `--install-update` keep working.

Remove the env var after a fix ships. Communicate the flag in your user advisory for any `/check-update` incident.

## PR CI checks

`pr-changelog-check.yml` runs on every PR that touches `aiwhyskill/scripts/**` or `aiwhyskill/SKILL.md`. It enforces that user-visible changes get a CHANGELOG entry:

- **Default behavior**: the PR must diff `CHANGELOG.md` and the diff must touch (or add) the `## [Unreleased]` section. The check passes if either is true.
- **Test-only bypass**: diffs scoped entirely under `aiwhyskill/tests/` auto-bypass.
- **Label bypass**: add the `no-user-changelog` label to a PR to skip the check entirely. Reserve this for refactors, dev-tooling changes, or code with no observable behavior change for end users.

If a PR author forgot the CHANGELOG bump, fix it by either (a) adding an entry under `## [Unreleased]`, or (b) applying the `no-user-changelog` label if the change truly isn't user-visible.

## Deploy PAT rotation (every 90 days)

The `PUBLIC_REPO_DEPLOY_PAT` secret expires every 90 days. Rotation:

1. GitHub → Settings → Developer settings → Fine-grained tokens → Generate new token.
   - Resource owner: `LextechGlobal`
   - Expiration: 90 days
   - Repository access: only `LextechGlobal/aiwhy-skill`
   - Repository permissions: `Contents: Read and write`, `Metadata: Read-only`.

2. Copy the token.

3. Update the secret on the private repo:

   ```bash
   gh secret set PUBLIC_REPO_DEPLOY_PAT \
     --repo LextechGlobal/aiwhy-secure-skill-launcher \
     --body '<paste new token>'
   ```

4. Verify the next release succeeds. If the workflow fails with 403 on the publish step, the new token is in "pending admin approval" — check org-level PAT policy.

5. Put a new calendar reminder 83 days out.

## Deferred: cryptographic signing

The original spec included RSA signature verification for release integrity. v1 defers that to make the initial pipeline simpler. Tracked as [issue #19](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/issues/19) in the private repo. When it re-lands, this document will pick up a new "Key rotation" section and the `Integrity` section in `../README.md` will expand to describe `openssl dgst -verify` steps.

## Troubleshooting

**Workflow fails at "Version-from-tag match":** the pushed tag (or `workflow_dispatch` input) doesn't equal SKILL.md's `metadata.version`. Bump one or the other. For engineering tags, the base version must match (tag `v1.3.1-eng.1` requires `metadata.version: "1.3.1"`).

**Workflow fails at "Extract Friendly Summary" with "CHANGELOG.md is missing or empty for [X.Y.Z]":** a customer MAJOR or MINOR release was tagged without promoting `## [Unreleased]`. Open a CHANGELOG-only PR to add the section, merge, and re-tag. PATCH releases (Z > 0) never hit this — they fall back to the canned summary.

**Workflow fails at "Publish to public repo" with 403:** the PAT is expired, revoked, or pending admin approval. Rotate (above).

**Workflow fails at "Publish to public repo" with 422 "already_exists":** you're trying to create a release for a tag that already has one. Use `gh release delete` on the existing release, or bump the tag.

**Published release has 0-byte `SHA256SUMS`:** should be impossible (step-fails halt the workflow), but if you see it, delete the release and re-run the workflow.

**Launcher in Cowork says "Update available" but the user just re-uploaded:** the session marker at `$CLAUDE_CODE_TMPDIR/.aiwhy-launcher-update-checked` is stale. The user can run `aiwhyskill check` (bypasses marker) or start a fresh Cowork task.

**PR blocked by `pr-changelog-check` but the change isn't user-visible:** apply the `no-user-changelog` label. If the change is test-only, the check should have auto-bypassed; if it didn't, confirm all diffed paths are under `aiwhyskill/tests/`.

**Engineering pre-release shows up in `/releases/latest`:** the tag wasn't shaped `vX.Y.Z-eng.N`, or the `--prerelease` flag wasn't applied. Check the "Publish to public repo" step log for `Marked as pre-release (engineering audience).` — if absent, the classifier saw the tag as a customer release. Delete the release and re-tag with the correct `-eng.N` suffix.
