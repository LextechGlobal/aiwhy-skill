# aiwhyskill

Secure launcher for the AIWhy marketplace of skills in Claude Cowork. Install once, get the full catalog of licensed skills delivered on-demand.

## What it does

aiwhyskill is a Claude Cowork [custom skill](https://docs.claude.com/en/docs/claude-code/settings) that authenticates you against the AIWhy marketplace, lists the skills your organization is entitled to, and fetches any of them into your current Cowork session on demand. Downloaded skills execute from ephemeral storage and are cleaned up after the session ends — nothing persists on your laptop.

## Install

1. Download the latest `aiwhyskill.zip` from the [Releases page](https://github.com/LextechGlobal/aiwhy-skill/releases/latest).
2. In Claude Cowork: **Customize → Skills → Upload a skill**. Pick the `aiwhyskill.zip` file.
3. Start a new Cowork task and say `aiwhyskill` to list available skills, or `aiwhyskill <skill-id>` to launch one.

## How updates work

Once installed, aiwhyskill checks for a newer version on every Cowork session (first command only — subsequent commands in the same session skip the check). If a newer release is on the [Releases page](https://github.com/LextechGlobal/aiwhy-skill/releases), Claude surfaces the release's **Friendly Summary** and offers to install the update for you.

Installing happens in-chat. When you accept, Claude presents a **Save skill** card in the same Cowork conversation — click it and Cowork writes the new version into your personal skill library in the same turn. You do not need to leave the session, download the zip, or re-upload anything. Your next `aiwhyskill` command runs on the new version.

The check is rate-limit-tolerant: if GitHub's API is unreachable or throttled, the launcher fails silently and tries again the next Cowork session. You can force an on-demand check with `aiwhyskill check`. Operators can suppress the check entirely during an incident by setting `AIWHYSKILL_DISABLE_UPDATE_CHECK=1` in the Cowork environment — `list` and `launch` keep working.

Certain releases are flagged as **required** — if your installed version is below the required minimum, aiwhyskill's `list` / `launch` commands will refuse to run until you upgrade. Required updates exist specifically for security-affecting fixes; we avoid them otherwise.

## Integrity

Each release attaches:
- `aiwhyskill.zip` — the launcher code.
- `SHA256SUMS` — SHA-256 digest of the zip, for verification.

To verify before installing:

```bash
shasum -a 256 -c SHA256SUMS
```

Expected output: `aiwhyskill.zip: OK`.

Cryptographic signing of releases (RSA sig + verification against an embedded public key) is tracked as a future enhancement. Until then, releases are trusted via GitHub's HTTPS serving plus the SHA256SUMS integrity check above.

## Privacy

On each Cowork session, aiwhyskill performs an unauthenticated `GET https://api.github.com/repos/LextechGlobal/aiwhy-skill/releases/latest` to check for updates. GitHub receives the Cowork sandbox's egress IP and a `urllib` User-Agent. No Circle.so identity, no organization ID, no skill usage is included in this request. A future enhancement may route this check through `skills.aiwhy.app` for rate-limit smoothing and optional adoption telemetry (opt-in).

## Support

- General help, licensing, new access requests: contact your AIWhy marketplace administrator.
- Bug reports or feature requests: open a [GitHub issue](https://github.com/LextechGlobal/aiwhy-skill/issues).
- Release history: the [Releases page](https://github.com/LextechGlobal/aiwhy-skill/releases).
