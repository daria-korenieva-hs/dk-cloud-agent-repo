# DE-5216 — list and invalidate on New Hydra

Ticket: https://hootsuite.atlassian.net/browse/DE-5216

**Approach:** Path A (thin Lerna list + one-token revoke) is confirmed.
**Authority:** [`spec.md`](spec.md) (independent review: PASS). Research: [`research.md`](research.md).

## Existing draft PRs — still needed

Do **not** close these. They are the Path A starting point. Adjust them to `spec.md`; do not invent a fifth approach.

| PR | Role | Merge order |
| --- | --- | --- |
| https://hootsuite.ghe.com/hootsuite/hydra-wrapper/pull/63 | Lerna `GET`/`DELETE /public/v1/oauth2/tokens` | 1 — merge and deploy first |
| https://hootsuite.ghe.com/hootsuite/IDL-central/pull/1328 | OpenAPI/IDL mirror only | anytime (fix identifier language first) |
| https://hootsuite.ghe.com/hootsuite/bruno-api-collection/pull/3 | Staging proof of #63 | after #63 is on shared staging |
| https://hootsuite.ghe.com/hootsuite/hootsupport2/pull/824 | Dual-read + invalidate | after #63 proven |
| https://hootsuite.ghe.com/hootsuite/hootsupport2/pull/819 | DE-5205 create (not this ticket) | same HootSupport **release** as #824; abort create cutover if #824 is missing |

## Identifier correction (do not copy the ticket comment)

`hydra_oauth2_access.signature` on Hydra 26.x is `hex(SHA-384(HMAC))`, **not** the last `.` segment of `ory_at_<key>.<hmac>`.
List must return the stored column. DELETE is hash-then-fallback. Ticket pass-when lines that say “signature equals last HMAC” are wrong.

## Blocked in this Cloud Agent run

This environment has **no Hootsuite GHE SSO / PAT** (`hootsuite.ghe.com` API 401, SSH `Permission denied`). GitHub MCP Docker/Podman also failed. Product diffs were **not** reviewed or pushed. A follow-up agent with GHE access must apply `spec.md` §9 to the four drafts, then run the implementation/review loop.
