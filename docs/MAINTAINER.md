# aiwhyskill — maintainer runbook

Operator-facing procedures for cutting new releases, rolling back bad releases, and rotating credentials. End-user install instructions are in [../README.md](../README.md).

## The two-repo model

Launcher code lives in the private repo `LextechGlobal/aiwhy-secure-skill-launcher`. This public repo (`LextechGlobal/aiwhy-skill`) is a **distribution destination** — it does not mirror source code. The only things that land here are:

- `README.md` / `docs/MAINTAINER.md` — user-facing docs (committed manually).
- GitHub Releases with `aiwhyskill.zip` + `SHA256SUMS` assets (created by the `release-launcher` GitHub Action in the private repo).

All code changes happen in the private repo. The release pipeline publishes here via a `contents:write`-scoped fine-grained PAT.

## Prerequisites

- Push access to `LextechGlobal/aiwhy-secure-skill-launcher` (the private repo).
- No AWS access needed.
- The `PUBLIC_REPO_DEPLOY_PAT` GitHub Actions secret must be valid (see PAT rotation below).

## Cutting a normal release

1. From the private repo, on `main`, bump `metadata.version` in `aiwhyskill/SKILL.md`:

   ```bash
   # From aiwhy-secure-skill-launcher on main:
   vim aiwhyskill/SKILL.md  # change metadata.version to the new SemVer
   git add aiwhyskill/SKILL.md
   git commit -m "chore(aiwhyskill): bump version to vX.Y.Z"
   git push origin main
   ```

2. Tag the commit and push:

   ```bash
   git tag vX.Y.Z
   git push origin vX.Y.Z
   ```

3. The `release-launcher` workflow fires on the tag push. Watch it:

   ```bash
   gh run watch --repo LextechGlobal/aiwhy-secure-skill-launcher
   ```

   On success, the release appears at `https://github.com/LextechGlobal/aiwhy-skill/releases/tag/vX.Y.Z` with `aiwhyskill.zip` + `SHA256SUMS` attached.

4. Verify the published release:

   ```bash
   curl -L -o /tmp/aiwhyskill.zip https://github.com/LextechGlobal/aiwhy-skill/releases/download/vX.Y.Z/aiwhyskill.zip
   curl -L -o /tmp/SHA256SUMS    https://github.com/LextechGlobal/aiwhy-skill/releases/download/vX.Y.Z/SHA256SUMS
   cd /tmp && shasum -a 256 -c SHA256SUMS
   # Expected: aiwhyskill.zip: OK
   ```

## Cutting a required (Minimum-Required) release

Use this ONLY for security-affecting fixes. It forces every installed launcher that predates the release to stop working on `list` / `launch` until the user upgrades — real cost to customers.

```bash
gh workflow run release-launcher.yml \
  --repo LextechGlobal/aiwhy-secure-skill-launcher \
  --ref main \
  -f tag=vX.Y.Z \
  -f minimum_required=X.Y.Z
```

The `minimum_required` input gets prepended to the release body as a `Minimum-Required: X.Y.Z` line, which the launcher's `_parse_release_body` strips out and uses to decide whether to block.

Requirements:
- The tag (`vX.Y.Z`) must already exist on the private repo with matching SKILL.md `metadata.version` — do the normal bump/tag steps first, but don't push the tag. Then use `workflow_dispatch` to add the `minimum_required` line. (Or push the tag, let the auto-release run, and then dispatch a separate workflow for a subsequent tag with `minimum_required` — cleaner, no race with the push trigger.)

## Dry-running a release (no publish)

Validates the pipeline without creating a real release. Use this after any change to `release-launcher.yml`.

1. Create a smoketest branch with SKILL.md version temporarily set to something distinct (e.g. `0.0.1`):

   ```bash
   git checkout -b smoketest/dry-run
   sed -E -i.bak 's/(version: )"[0-9.]+"/\1"0.0.1"/' aiwhyskill/SKILL.md && rm aiwhyskill/SKILL.md.bak
   git add aiwhyskill/SKILL.md
   git commit -m "smoketest: bump SKILL.md version to 0.0.1"
   git push -u origin smoketest/dry-run
   ```

2. Dispatch the workflow with `dry_run=true`:

   ```bash
   gh workflow run release-launcher.yml \
     --repo LextechGlobal/aiwhy-secure-skill-launcher \
     --ref smoketest/dry-run \
     -f tag=v0.0.1 \
     -f dry_run=true
   ```

3. Download the artifact bundle and verify:

   ```bash
   gh run download --repo LextechGlobal/aiwhy-secure-skill-launcher --name release-bundle-v0.0.1
   cd release-bundle-v0.0.1
   shasum -a 256 -c SHA256SUMS
   # Expected: aiwhyskill.zip: OK
   ```

4. Clean up:

   ```bash
   git checkout main
   git push origin --delete smoketest/dry-run
   git branch -D smoketest/dry-run
   ```

No tags were created or pushed, so nothing else needs cleanup.

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

**Workflow fails at "Version-from-tag match":** the pushed tag (or `workflow_dispatch` input) doesn't equal SKILL.md's `metadata.version`. Bump one or the other.

**Workflow fails at "Publish to public repo" with 403:** the PAT is expired, revoked, or pending admin approval. Rotate (above).

**Workflow fails at "Publish to public repo" with 422 "already_exists":** you're trying to create a release for a tag that already has one. Use `gh release delete` on the existing release, or bump the tag.

**Published release has 0-byte `SHA256SUMS`:** should be impossible (step-fails halt the workflow), but if you see it, delete the release and re-run the workflow.

**Launcher in Cowork says "Update available" but the user just re-uploaded:** the session marker at `$CLAUDE_CODE_TMPDIR/.aiwhy-launcher-update-checked` is stale. The user can run `aiwhyskill check` (bypasses marker) or start a fresh Cowork task.
