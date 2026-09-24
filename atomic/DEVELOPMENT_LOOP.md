# AtomicQMS Development Loop

Use this as the default operating guide for AtomicQMS bug fixes, UX improvements, features, and validation.

## Core Rule

Use local development for the inner loop. Do not deploy to a shared or production environment merely to check a UI change.

```
local laptop -> shared dev -> beta acceptance -> cchmc production
```

## Environment Map

| Environment | Identifier | Purpose | Normal use | Deployment posture |
|---|---|---|---|---|
| Local | localhost | Isolated development/reproduction | Default for bug fixes, UX work, narrow tests, document-editing experiments | No remote impact |
| Dev | dev | Shared rapid iteration | Team-visible testing after local verification | Real shared deployment |
| Beta | beta | Stable preview | Stakeholder acceptance, demos, release candidates | Not casual scratch space |
| Production | cchmc | Live institutional service | Approved releases only | Explicit confirmation and manual-account-approval policy |

Production is named `cchmc`, not `prod`.

## Local Stack

| Service | Address | Purpose |
|---|---|---|
| AtomicQMS web | http://localhost:3000 | Customized Gitea web app |
| WOPI | http://localhost:8000/health | Office-document editing protocol host |
| Collabora | http://localhost:9980/hosting/discovery | Browser-based DOCX/PPTX/XLSX editor |
| PostgreSQL | Internal Docker network | Application database |
| Redis | Internal Docker network | WOPI session/cache support |

### Start

```bash
open -a Docker
docker start atomicqms-local-postgres atomicqms-local-wopi-redis
docker start atomicqms-local-wopi atomicqms-local-collabora atomicqms-local
open http://localhost:3000
```

### Observe a bug

```bash
docker logs -f atomicqms-local
docker logs -f atomicqms-local-wopi
docker logs -f atomicqms-local-collabora
```

Do not restart containers during ordinary observation. Restart only when configuration or a built image changed.

## Required Loop

1. Define the user-visible problem and expected behavior.
2. Use a focused, non-protected feature branch. Never work directly on `main`.
3. Reproduce locally.
4. Identify the smallest responsible code surface.
5. Make the smallest scoped change.
6. Run the narrowest relevant automated test.
7. Verify manually in the local browser.
8. Review the diff for accidental policy, approval-chain, or unrelated changes.
9. Commit and open a pull request when ready for review.
10. Use shared `dev` only after local verification, when team-visible testing is necessary.
11. Use `beta` for acceptance/release-candidate validation.
12. Use `cchmc` only with explicit approval and the repository's production guard.

## Code Map

| Concern | Primary location |
|---|---|
| UI interactions | `gitea/gitea/public/assets/js/` |
| UI labels/wording | `gitea/gitea/options/locale/locale_en-US.ini` |
| Page templates | `gitea/gitea/templates/` |
| Approval/change requests | `gitea/gitea/public/assets/js/aqms-approval-flow.js` |
| General UI behavior | `gitea/gitea/public/assets/js/aqms-ux-reskin.js` |
| WOPI | `services/atomicqms-wopi/` |
| Collabora | `Dockerfile.collabora`, `containers/collabora/` |
| File conversion/import | `services/atomicqms-file-utils/` |
| AtomicAI | `services/atomicai/` |
| Deployment configuration | `config/environments/{dev,beta,cchmc}/` |
| Narrow tests | `scripts/prod/tests/` |

## Testing Rules

- Prefer one narrow relevant test over a broad suite.
- Pair UI tests with browser verification.
- For document editing, verify web + WOPI + Collabora together.
- Do not claim a change is verified when the local container serves stale assets; confirm the source-sync/rebuild path first.
- Record verification limits honestly. A WOPI unit test does not prove real multi-user Collabora editing.

## Approval and Audit Safety

UX simplification must not weaken governance. Do not remove or bypass approval requirements, branch protection, reviewer/sign-off behavior, publish gates, change-request/commit history, REDCap safeguards, or production deployment confirmation.

**Goal: make one-file uploads feel simple while preserving approvals, signatures, and audit requirements wherever workspace policy requires them.**

## Shared-Environment Commands

```bash
source scripts/env.sh dev
source scripts/env.sh beta
source scripts/env.sh cchmc

make check env=dev
make test-live env=dev
make status env=beta
make logs env=beta
```

`make deploy env=dev` is a real shared deployment, not a local refresh command.

## Agent and MCP Instructions

When assigning AtomicQMS work:

1. Link this guide.
2. State the problem, expected behavior, and affected environment.
3. Require local reproduction before suggesting a remote deployment.
4. Require a minimal test plan and manual verification path.
5. Identify whether approvals, audit history, permissions, external integrations, or deployment are affected.
6. Require a human gate before production, external writes, destructive work, schema changes, secrets, auth, billing, or infrastructure changes.
7. Never claim a test, reviewer, specialist, or remote environment was consulted unless it actually was.

## Current Operational Note

The local full stack exists and runs through Docker Desktop. Verify its source-sync/rebuild process before treating it as final proof for source-code UI changes. The shared dev host may require restored network/VPN access from the developer machine.