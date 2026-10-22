# DE-5216 research results

Ticket: https://hootsuite.atlassian.net/browse/DE-5216
Parent: DE-5175 (M4). Status: To Do. Assignee: Daria Korenieva.
Current-state section treated as confirmed truth of what exists now.

## Verdict on Path A

**confirm-with-corrections**

Path A (thin Lerna list + one-token revoke) is still the only approach that keeps the Support widget: list a member's TMA LLTs, invalidate one, never store `ory_at_…`. Stock Hydra OEL/OSS 26.3.x cannot do that. Paths B and C remain rejected.

The ticket's Path A **architecture** is correct. The ticket's Path A **identifier** is not. `hydra_oauth2_access.signature` on New Hydra 26.x is **not** the last HMAC segment of `ory_at_<key>.<hmac>`. It is `hex(SHA-384(that HMAC))`. If Lerna `SELECT signature` returns the raw column, the claim "handle = last `.` segment / leftover HMAC" is **false** for every token minted after DE-5205.

A spec that copies the ticket's identifier verbatim will ship a list that cannot invalidate from a minted token's last segment, and an invalidate that 404s if HootSupport sends the HMAC instead of the stored hash (or the reverse).

## Why A wins vs B/C

Stock hydra-oel / OSS **26.3.18** (staging) admin surface, independently checked against Ory Hydra 26.x OpenAPI + `oauth2/handler.go`:

| Primitive | What it does | Usable for the widget? |
| --- | --- | --- |
| `GET /admin/oauth2/tokens` | **Does not exist** | No list-by-subject |
| `GET /admin/oauth2/auth/sessions/consent?subject=` | Lists **consent sessions** | jwt-bearer / `member_app` has no consent row → empty |
| `DELETE /admin/oauth2/auth/sessions/consent?subject=` | Flush consent (+ tokens tied to those sessions) | **No-op** for jwt-bearer on 26.3.18 |
| `DELETE /admin/oauth2/tokens?client_id=` | Deletes **every** access token for that client | All TMA members. Unusable |
| `POST /admin/oauth2/introspect` | Needs the **raw bearer** | Cannot list; do not store `ory_at_` |
| `POST /oauth2/revoke` | Needs the **raw bearer** + client auth | Path C only |

OEL changelog through 26.3.x adds indexes and revoke-performance work, **not** a token-list API. v26.2.7 added `(nid, subject, client_id)` on `hydra_oauth2_access` / `hydra_oauth2_refresh` — that helps Path A SQL, it does not replace it.

**26.4.4** (released 2026-10-21): consent revoke now also deletes jwt-bearer and password-grant tokens for that subject (or subject+client). Confirmed. We are on **26.3.18**, so the ticket's "consent flush is a no-op" is still true. Even after 26.4.4 that API is still **flush-all** (Path B), not per-token list. Upgrading Hydra does not unlock Path A without Lerna.

**Path B — flush-all — still rejected.** Support cannot revoke one leaked LLT. Invalidating one row would kill every other TMA LLT for the member (and on 26.4.4, possibly every jwt-bearer for that subject). Does not match the widget.

**Path C — paste-revoke — still rejected.** Works only while the caller still has the plaintext. No list after the create modal is dismissed. Persisting `ory_at_…` is forbidden.

**Path A wins** because Lerna already SQL-touches `hydra_oauth2_access` (cleanup job). Exposing list + one-row delete over HTTP is the smallest primitive that preserves per-token UX without storing the bearer. Ticket originally marked "New Lerna admin CRUD" out of scope because it assumed OSS had a list. OSS does not. The HTTP exception is required.

## Path A details (corrected if needed)

These must be exact in a spec. Ticket text that disagrees is called out.

### Auth

- Caller sends **TMA HTTP Basic** (`client_id:client_secret`).
- Lerna verifies with Hydra `client_credentials` (public token host). **Not** leftover scope `hydra.oauth2.tokens`. That scope is legacy-fork only.
- `client_id` used in SQL = Basic username.
- Staging TMA: `4e3be522-b7e9-434b-bcd7-69640b15fe6a`. Prod TMA: `e303d588-…` (from DE-5204).
- Missing/invalid Basic → 401. TMA cannot list other clients; another client cannot see TMA rows.

### Routes

- Lerna: `GET /public/v1/oauth2/tokens?subject={memberId}`
- Lerna: `DELETE /public/v1/oauth2/tokens?subject={memberId}&signature={handle}`
- Aperture rewrite: `/oauth2/tokens` (ticket also writes `{LERNA}/oauth2/tokens` — same public path after rewrite).
- Missing `subject` → 400. Unknown / not-owned handle → 404. Success DELETE → 204.
- Body on GET only: `{tokens:[{signature,expiry}]}`. Never return the bearer, request_id, session_data, or form_data.

These routes are **not** in the published Lerna runbook yet (only `/public/v1/oauth2/auth`, `/public/v1/oauth2/token`, `/v1/hooks/token`).

### SQL

List (exact):

```sql
SELECT signature, expires_at
FROM hydra_oauth2_access
WHERE client_id = ? AND subject = ?
```

`client_id` = authenticated TMA. `subject` = member id (jwt-bearer assertion subject; DE-5205 introspect proof uses `sub` = member id).

Delete (corrected — must match Hydra lookup, not exact-HMAC-only):

```sql
-- try hashed first, then raw (Hydra Get/DeleteAccessTokenSession)
DELETE FROM hydra_oauth2_access
WHERE client_id = ? AND subject = ?
  AND signature IN (SHA384_HEX(?), ?)
```

Go equivalent of Hydra:

```go
x.SignatureHash(sig) == fmt.Sprintf("%x", sha512.Sum384([]byte(sig)))
```

Then, if 0 rows, also try `signature = ?` as provided (already included in `IN` above).

Do **not** invent `PUT /internal/tokens` or PATCH-expiry on OSS/Lerna.

Refresh rows: Lerna runbook states `member_app` / `organization_app` **never issue refresh tokens** (`offline` / `offline_access` stripped). TMA LLT delete is access-table-only. Do not also flush `hydra_oauth2_refresh` unless a later grant change issues refresh tokens.

OEL 26.2.7 `(nid, subject, client_id)` index makes this query cheap.

### Response shape

```json
{ "tokens": [ { "signature": "<stored hydra_oauth2_access.signature>", "expiry": "<expires_at>" } ] }
```

`signature` here is the **DB column**, not the last HMAC, not the full `ory_at_`. For New Hydra 26.x rows it is a 96-char lowercase hex SHA-384. For any leftover raw-signature rows (Hydra < ~2.1) it is the raw HMAC.

### Identifier / hashing (the correction)

Token format (opaque, default on 26.x; TMA jwt-bearer issues these, not JWTs):

```
ory_at_<base64url-key>.<base64url-hmac>
```

Fosite `HMACStrategy.Signature(token)` splits on `.` and, when there are exactly two parts, returns `split[1]` = the HMAC. The `ory_at_` prefix lives in part[0], so last `.` segment **is** the HMAC. That part is correct.

Hydra then stores:

```
CreateAccessTokenSession(signature) → INSERT signature = x.SignatureHash(signature)
```

`x.SignatureHash` = `hex(SHA-384(hmac_string))` (`x/sighash.go`). Lookup hashes first, then falls back to raw for pre-hash rows (`GetAccessTokenSession` / `DeleteAccessTokenSession`).

Implication: ticket Path A "handle = last `.` of `ory_at_<key>.<hmac>` and Lerna `SELECT signature` returns that HMAC" is **false** for every token Hydra 26.3.18 mints.

| Source | Value | Use as invalidate handle? |
| --- | --- | --- |
| Minted bearer last `.` | raw HMAC (base64url, ~43 chars) | Only if DELETE hashes first |
| List API `tokens[].signature` | stored column (SHA-384 hex, 96 chars) | **Yes — canonical** |
| `#819` create `signature` field | full `ory_at_…` | **No.** Show-once copy field only. Do not persist. Do not send to DELETE/legacy PUT |
| Leftover JWT | `explode('.', $token)[1]` = payload | Legacy `PUT /internal/tokens` only |

### HootSupport dual-read / invalidate

Until **DE-5180** (Backlog — tear down Legacy Hydra):

- **List:** merge Lerna `GET …/oauth2/tokens?subject=` **and** leftover `GET /internal/tokens?memberId=`. Dedup by handle. Do not require a leftover `key.signature` for new rows.
- **Invalidate:** Lerna `DELETE` first. On **404**, leftover `PUT /internal/tokens` `{signature, expiry: now}` after the existing ownership GET. Do **not** send a 96-char SHA-384 handle to leftover PUT. Do **not** send a leftover JWT payload to Lerna and treat 404 as fatal without the PUT fallback.
- Ownership: still re-check the handle belongs to the member (list or DELETE scoped by subject).
- Auth to Lerna: TMA Basic. Auth to leftover: existing CC scope `hydra.oauth2.tokens`.

Create stays on **DE-5205 / hootsupport2 #819**. Do not fold create into DE-5216. Do not call `internal/tokens` on create.

### Handle derivation (must be in the spec)

1. **From the list API (UI rows):** use `tokens[].signature` as-is. That is the invalidate handle. Never show or store the bearer.
2. **From a just-minted token:** last `.` is the HMAC, **not** the list handle. After mint, either:
   - re-GET list and take the new row (preferred; no hash in PHP), or
   - compute `hash('sha384', last_segment)` (PHP) if a handle is needed before re-list, or
   - pass last-segment to DELETE **only if** Lerna implements hash-then-fallback.
3. **Do not store raw `ory_at_…` in HootSupport DB.** Create modal may display the bearer once for the customer. Drop it when the modal closes.
4. **Leftover JWTs:** keep `explode('.', $token)[1]` **only** for the old PUT path.

## Identifier finding

**What `hydra_oauth2_access.signature` actually is (Hydra 26.x / OEL 26.3.18):**

```
hex( SHA-384( fosite_hmac_signature ) )
```

where `fosite_hmac_signature` = last `.` segment of `ory_at_<key>.<hmac>`.

Source: `CreateAccessTokenSession` in `persistence/sql/persister_oauth2.go` writes `x.SignatureHash(signature)`. `x/sighash.go`:

```go
func SignatureHash(signature string) string {
    return fmt.Sprintf("%x", sha512.Sum384([]byte(signature)))
}
```

Ory issue #3485 documents the same: previously the PK was `Y3JSKXSGRIsoxMEfHdAEbweH-rE5p-qajxSAcoAdmxI`; after the hash change it is `54dcf7602fc6e6abf7df20f49f5cd445ee0ac431497638d69cdb013d924df5da9dff9b8040c45d40a48b3b2c8102b30d`. Lookup still falls back to raw for very old rows.

**Implication for list / delete / UI:**

- List must return the **column**. Do not invent a reverse-hash (impossible). Do not rewrite it to the HMAC.
- DELETE must accept the **column value** (what the UI has after list). Spec should also accept the raw HMAC via hash-then-fallback so create-response last-segment and Hydra-style callers work.
- UI handle for new tokens is a 96-char hex string, not a JWT-looking payload and not the HMAC. Product copy may still call it "signature".
- `#819` returning `signature == full ory_at_…` is a **create-response convenience**, not the list/invalidate identifier. If #824 treats that field as the handle, invalidate is wrong.

## Company knowledge deltas

| Source | Claim | Research |
| --- | --- | --- |
| Ticket Path A comment 2035207 | Handle = last `.` of `ory_at_<key>.<hmac>`; Lerna `SELECT signature` returns that HMAC | **False** on Hydra 26.x. Column is SHA-384 hex of the HMAC |
| Ticket original description | `explode('.', $token)[1]` is a **legacy** `key.signature` and is **wrong** for `ory_at_…` | **True.** Path A comment later re-introduced last-segment as if it were the PK |
| Ticket / brief | Consent flush is a no-op for jwt-bearer | **True on 26.3.18.** False as "never" — 26.4.4 flushes jwt-bearer too, but still flush-all (Path B) |
| Ticket original How | Default is **direct OSS admin**, Lerna only if we add a thin proxy | Path A chose Lerna HTTP. Correct: OSS admin has no list. DE-5204 client writes go to New Hydra **admin** directly; tokens stay Lerna |
| Lerna runbook (page 13773177838, v5, 2026-08-05) | Published endpoints are auth / token / token_hook only; custom grants translated against **hydra-oss** | No `/oauth2/tokens` list yet (expected). Staging New Hydra is **OEL 26.3.18**, hydra-oss replicas 0 (DE-5183). Runbook "hydra-oss" wording is stale |
| Lerna runbook | Cleanup job already DELETEs `hydra_oauth2_access` / `hydra_oauth2_refresh` | Confirmed precedent for Path A SQL. `member_app` issues **no refresh tokens** |
| DE-5205 #819 comments | Create `signature` = full `ory_at_…` | True of that draft. Must not become DE-5216's handle |
| DE-5205 | Recommend same HootSupport **release** as DE-5216 | Still correct. Merge order of **this** ticket is #63 then #824; #819 is independent code but should ship together |
| Brief / ticket | Dual-read until DE-5180 | Confirmed. DE-5180 and DE-5181 are Backlog / To Do |
| Brief | IDL #1328 is spec mirror only; Bruno #3 is staging proof | Confirmed from comment 2035248 |

## Open design decisions

**Already decided (do not reopen unless a draft PR is broken):**

- Path A, not B/C.
- Create stays on DE-5205 (#819). No `PUT /internal/tokens` on OSS/Lerna.
- Dual-read leftover `GET`/`PUT /internal/tokens` until DE-5180.
- Auth = TMA Basic + Hydra `client_credentials`, not `hydra.oauth2.tokens`, not OSS admin no-auth.
- Filter to TMA via Basic `client_id` (staging `4e3be522-…` / prod `e303d588-…`).
- Merge/deploy **hydra-wrapper #63 first**, prove list + one-token revoke, then **hootsupport2 #824**. Bruno **#3** is that proof. IDL **#1328** is docs only.
- Prefer same HootSupport release as #819 so the UI never points at a dead list/invalidate.
- Do not store raw `ory_at_…`. Do not write `llt_map`. Do not remint. Aperture swap is DE-5203.

**Still open (spec must pick):**

1. **DELETE input contract.** Exact match on the DB column only vs Hydra-style hash-then-fallback. **Recommend fallback** so list handles and last-segment HMACs both work.
2. **Post-mint handle in HootSupport.** Re-list vs `hash('sha384', last_segment)` vs rely on DELETE hashing. **Recommend re-list** as source of truth; hash only if the create modal must invalidate without a second GET.
3. **`#819` `signature` field.** Keep as full bearer for the copy dialog, or change to the stored hash. Either way, #824 must not treat full `ory_at_` as the Lerna handle.
4. **Mapped leftover LLTs (ticket Q3).** Invalidate the mapped New Hydra `ory_at_` vs leftover PUT vs tell the member to rotate. Map stays; do not delete map rows here unless Q3 says so.
5. **List expired / `active=0` rows?** Column exists. Cleanup job later deletes expired access rows (`expires_at < now - 7d`). UI can hide expired; decide whether GET filters `expires_at > now AND active=1`.
6. **Pagination.** Unlikely for TMA LLTs per member; still decide a cap.
7. **Lerna runbook + OpenAPI.** #63 / #1328 must add `/public/v1/oauth2/tokens`. Not a product fork; docs lag.

Ticket Q1 (does OSS list by subject?) is **answered: no**. Q2 (dual-read?) **answered: yes until DE-5180**. Q4 (ship order) **answered: parallel, same HootSupport release**. Q5 (who calls admin?) **answered: TMA Basic via Lerna, not OSS admin**.

## Implementation plan (high level, 4 PRs)

Already drafted. Do not merge until asked. Do not invent a fifth approach.

1. **hydra-wrapper #63** `DE-5216-lerna-token-list` — GET/DELETE `/public/v1/oauth2/tokens`. **Merge and deploy first.** Must implement the **corrected** identifier (return stored column; DELETE hash-then-fallback). Missing Basic → 401; missing subject → 400; unknown signature → 404.
2. **IDL-central #1328** — Jenkins spec mirror only. Must describe `signature` as the stored access-token PK (SHA-384 hex on 26.x), not "HMAC last segment".
3. **bruno-api-collection #3** `DE-5216-lerna-token-admin` — staging proof. Mint via Lerna `member_app` as TMA; list; confirm list `signature` **≠** last HMAC and **=** `sha384(last HMAC)`; DELETE by list handle → 204; introspect `active: false`; Aperture `/authz/oauth` 401; second mint still lists; other signatures remain. Optional: DELETE by last HMAC if fallback is specced. No bearers on the Jira ticket.
4. **hootsupport2 #824** `DE-5216-list-invalidate` — dual-read + Lerna DELETE then leftover PUT on 404. Independent of #819 in git; **same release**. Derive invalidate handle from list (or SHA-384), never persist `ory_at_`.

Order: **#63 → deploy Lerna staging → Bruno #3 proof → #824.** #1328 anytime (docs). #819 stays on DE-5205.

## Risks to check in existing PRs

GHE is not authenticated here. These are the checks a reviewer with repo access must run; do not invent file contents.

**hydra-wrapper #63**

- Does list return the raw `signature` column, or does it try to "decode" an HMAC?
- Does DELETE `WHERE signature = $query` with the HMAC (will 404 on every 26.x row) or hash-then-fallback?
- Is SQL scoped to Basic `client_id` **and** `subject`? A signature-only delete is a cross-member bug (PK is globally unique, but still require both).
- Response must not include bearer, session blob, or form_data.
- Auth is Hydra CC with TMA Basic, not leftover `hydra.oauth2.tokens`.
- No `PUT /internal/tokens` rebuild.

**hootsupport2 #824**

- Invalidate handle = list `signature`, not `explode('.', $token)[1]`, not full `ory_at_` from #819.
- Dual-read merge until DE-5180; leftover PUT only on Lerna 404; do not PUT a SHA-384 handle to leftover.
- Does not persist minted tokens.
- Independent of #819 in git; confirm it still compiles if #819's `signature == token` ships first.

**hootsupport2 #819 (DE-5205, not this ticket, but same UI)**

- `signature` is the full `ory_at_…`. Safe only as a one-shot create-response field.
- If any table/cache stores that field, that violates "do not store raw ory_at_".
- Same HootSupport release as #824 or the widget lists empty / cannot revoke new tokens.

**IDL-central #1328**

- If the spec says "HMAC / last segment", it will lock in the wrong contract. Correct to stored PK / SHA-384 hex.

**bruno-api-collection #3**

- Proof must **observe** list.signature vs last `.` vs `sha384(last)`. If they are equal, the environment is not Hydra 26.x hashing (stop and re-check). If they differ and DELETE-by-HMAC 404s, #63 is missing fallback.
- No full tokens or TMA secrets in Jira.

**Shared**

- Staging is **Hydra OEL 26.3.18**; leftover `hydra-oss-admin.staging.hootops.com` is replicas 0 / 503. Proofs go to OEL + Lerna (`staging-platform.hootsuite.com`).
- Consent-flush experiments prove nothing on 26.3.18 for jwt-bearer.
- Lerna trust `expires_at` 2037-12-31 is a create-path risk (DE-5205), not list/invalidate.

## Sources

- DE-5216: https://hootsuite.atlassian.net/browse/DE-5216 (description Current state; comment 2035207 Path A/B/C; comment 2035248 draft PRs)
- DE-5205, DE-5204, DE-5180, DE-5181 (Atlassian MCP)
- Parent brief: `/cursor/stores/self/de-5216-research-brief.md`
- Hydra-Wrapper (Lerna) Runbook: https://hootsuite.atlassian.net/wiki/spaces/PLATFORM/pages/13773177838
- Ory Hydra OAuth2 admin API (hydra-client-go v26 OAuth2API.md): no list-tokens-by-subject; `DELETE /admin/oauth2/tokens` is client-wide
- `oauth2/handler.go` `deleteOAuth2Token`: `DeleteAccessTokens(ctx, clientID)` only
- `persistence/sql/persister_oauth2.go` `CreateAccessTokenSession` / `GetAccessTokenSession` / `DeleteAccessTokenSession`
- `x/sighash.go` `SignatureHash` = SHA-384 hex
- Ory hydra#3485 (stored signature is SHA-384 of HMAC, not raw HMAC)
- fosite `token/hmac/hmacsha.go` `Signature()` = last of two `.` parts
- Ory opaque token docs: `ory_at_<key>.<hmac>`
- Ory changelog 26.4.4: consent revoke now includes jwt-bearer (flush-all). Staging is 26.3.18
- OEL changelog v26.2.7: `(nid, subject, client_id)` indexes on access/refresh
- Ory revoke-consent docs: still subject or subject+client, not per-token
