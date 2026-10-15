# aiwhy-skill — Transition Plan

**Status:** Handover in progress · **Prepared:** 2026-10-15 ·\
**Last verified against code:** 2026-10-15 at `ab9644d` (this repo) and `56448f1` (`aiwhy-secure-skill-launcher`) · **Repository:** `LextechGlobal/aiwhy-skill`\
**Plan of record:** `LextechGlobal/aiwhy-secure-skill-launcher` › `docs/transition-plan/` (private repo)

This repository is handed over **as part of the AIWhy Secure Skill Launcher**, not on its own. It
holds no code: only `README.md`, `docs/MAINTAINER.md`, `.gitignore`, and the GitHub Releases
(`aiwhyskill.zip` + `SHA256SUMS`) that the launcher's release workflow publishes here. The
launcher's transition plan lists this repo as in scope, so that plan is the single source of
truth. This page only points to it, so the handover facts live in one place and do not drift
apart.

The plan of record is in a **private** repository. You need write access to it (granted in its
checklist, Phase 4) to open the links below.

## Where this repo is covered

| What you need | Where in the plan of record |
|---------------|-----------------------------|
| What this repo is and what the launcher assumes about it (release body format, tag shape, asset names) | [05 › Sibling repositories and integration contracts](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/05-dependencies-and-cloud-resources.md#sibling-repositories-and-integration-contracts) |
| Write access to this repo (K2) and the release token `PUBLIC_REPO_DEPLOY_PAT` (K3) | [06 › Access inventory](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/06-access-and-credentials.md#access-inventory-k-items) |
| How the release token is handed over (S11, re-issue) | [06 › Secure Handover Package](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/06-access-and-credentials.md#secure-handover-package) |
| Cutting, verifying and withdrawing a launcher release | [08 › Releasing a launcher version](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/08-operations-runbook.md#releasing-a-launcher-version) |
| Rotating the release token | [08 › Rotating a secret](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/08-operations-runbook.md#rotating-a-secret) |
| Risk that the next release fails (R4: token likely expired, workflow actions on retired Node 20) | [09 › Risks](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/09-known-issues-and-risks.md#risks-r-items) |
| Where this repo's `docs/MAINTAINER.md` disagrees with the code or other docs (D26 required-release command, D27 deleting a release) | [09 › Documentation drift](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/09-known-issues-and-risks.md#documentation-drift-d-items) |
| Handover steps for this repo (access check, token ownership, release rehearsal) | [12 › Handover checklist](https://github.com/LextechGlobal/aiwhy-secure-skill-launcher/blob/main/docs/transition-plan/12-handover-checklist.md) — Phases 4, 5 and 6 |

## Rules for this repo during and after the handover

- Do not add a separate plan here. A change to how this repo is operated belongs in the plan of
  record, and this table gets updated only if a section moves.
- This repo is **public**. Keep account IDs, instance IDs and internal hostnames out of it;
  they live in the private plan.
- Until D26 and D27 are resolved, where `docs/MAINTAINER.md` and the plan of record disagree,
  the plan of record wins.
