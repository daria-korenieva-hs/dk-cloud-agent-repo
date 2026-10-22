# DE-5216 implementation spec

**Ticket:** [DE-5216](https://hootsuite.atlassian.net/browse/DE-5216) — Move HootSupport LLT list and invalidate onto New Hydra  
**Parent:** DE-5175 (M4)  
**Status at spec time:** To Do  
**Approach:** Path A — thin Lerna list + one-token revoke  
**Authority:** `/cursor/stores/self/de-5216-locked-decisions.md` (Path A + identifier), `/cursor/stores/self/de-5216-review-locks.md` (round 1), `/cursor/stores/self/de-5216-review-locks-round2.md` (round 2), and `/cursor/stores/self/de-5216-review-locks-round3.md` (I-01–I-05). Ticket comment 2035207 is the architecture source. Ticket Path A **identifier** text is **wrong** and must not be copied. Where this spec and the ticket comments disagree, **this spec wins**.

Engineers with GHE access **must** start from the existing draft PRs and **adjust** them. They **must not** invent a fifth HTTP approach, a new product PR (unless the rewrite repo cannot be edited from #63 — see §5.2), or `PUT /internal/tokens` on OSS/Lerna.

| PR | Branch (ticket) | Role |
| --- | --- | --- |
| [hydra-wrapper #63](https://hootsuite.ghe.com/hootsuite/hydra-wrapper/pull/63) | `DE-5216-lerna-token-list` | Lerna GET/DELETE + rewrite confirmation/wiring in the same deploy change-set. Merge and deploy first. |
| [IDL-central #1328](https://hootsuite.ghe.com/hootsuite/IDL-central/pull/1328) | (docs) | Jenkins OpenAPI/IDL mirror only. |
| [bruno-api-collection #3](https://hootsuite.ghe.com/hootsuite/bruno-api-collection/pull/3) | `DE-5216-lerna-token-admin` | Staging identifier + revoke-oracle **subset** of #63. |
| [hootsupport2 #824](https://hootsuite.ghe.com/hootsuite/hootsupport2/pull/824) | `DE-5216-list-invalidate` | Dual-read list + Lerna DELETE then leftover PUT. |

Related, **not this ticket:** [hootsupport2 #819](https://hootsuite.ghe.com/hootsuite/hootsupport2/pull/819) (DE-5205 create). Independent git branch. **Must** ship in the **same HootSupport release** as #824 when create cutover happens. If #819 would deploy without #824, **abort** that create cutover.

This environment could not read GHE diffs. Implementers **must** open those four PRs and apply the per-PR change list in §9.

---

## 1. Goal and non-goals

### Goal

Keep HootSupport Token Management **list** and **invalidate** working after DE-5205 stops calling leftover `GET`/`PUT /internal/tokens` for **new** tokens.

Support **must** be able to:

1. List **up to 200** of a member’s TMA long-lived tokens that live on **New Hydra** (at least tokens minted after DE-5205). Overflow above 200 is **logged only** (see §5.3).
2. Invalidate **one** of those tokens so OEL introspect `POST https://hydra-oel-admin.staging.hootops.com/admin/oauth2/introspect` returns JSON that includes `"active": false`. That introspect body is the **revoke oracle**. Aperture `GET /authz/oauth` **401** is a **secondary** staging check only (see §7.3). It **must not** be the #63 pass/fail gate.
3. When **Lerna GET is 200** and leftover GET is 200, still list leftover 0.9 signatures and invalidate **leftover-sourced** (and **both-sourced**) handles, until DE-5180. If Lerna GET is not 200, the list page fails (§6.2.2); leftover rows are not shown.

Create **must** stay on DE-5205. This ticket **must not** fold create into #824 or #63.

### Non-goals (this ticket)

- Mint / extend LLTs (DE-5205 / #819), including post-mint re-GET, PHP-hash-before-relist, create-modal invalidate, persist tests, and editing #819 helpers.
- Rebuild `PUT /internal/tokens` on OSS, OEL, or Lerna.
- Store raw `ory_at_…` in HootSupport (DB, cache, logs, Jira, PR bodies).
- Write or delete `llt_map` rows; remint mapped tokens (DE-5200 / DE-5201).
- Aperture **swap flag** (DE-5203). In-scope Aperture work is only confirming or adding the `/oauth2/tokens` **prefix rewrite** in the #63 deploy change-set (see §5.2). That is not the swap.
- Tear down leftover Hydra (DE-5180 / DE-5181).
- Path B (flush-all) or Path C (paste-revoke).
- Pagination, a client-visible cap warning, refresh-token delete, or listing more than 200 access rows.
- A fifth HTTP approach.
- New list/invalidate response fields (`source`, leftover membership). Invalidate re-runs dual-read per request (§6.3).

---

## 2. Current state (confirmed)

Treat this section as what exists **now**. Source: ticket Current state + Slack thread captured on the ticket. Do not “correct” production behavior in this section.

**How:** Member page → Login settings → Access tokens. Support agents list a member’s tokens, create a long-lived token, and invalidate a token.

- **List:** `GET /internal/tokens?memberId=…`. HootSupport reads only `signature` and `expiry`.
- **Invalidate:** existing HootSupport `POST /api/member/{id}/access-tokens/{signature}/invalidate`, which today `PUT /internal/tokens` with `{signature, expiry: now}` after the leftover GET proves the signature belongs to that member. Path `{id}` is the member.
- **Create (today, leftover widget path):** mint `member_app` via `/oauth2/token`, then `PUT /internal/tokens` to set expiry to +50 years. New Hydra **mint** already goes through Lerna public `/oauth2/token` (DE-5205 / Bruno). The widget Create button still follows leftover PUT until #819 is on staging. **#824 FIT must not require widget Create** (§6.6).
- **Auth (leftover):** client_credentials, scope `hydra.oauth2.tokens`.
- **Identifier (leftover):** `explode('.', $token)[1]` — a legacy JWT/`key.signature`. Wrong for `ory_at_…`.
- **Agent gate (unchanged):** HootSupport `manage_access_tokens` **plus** member-page ownership of path `{id}`. Leftover Hydra already accepts any holder of the TMA CC token for any `memberId`. Lerna will do the same for any authenticated `client_id` + `subject` (see §5.1). **Must not** change the agent gate in this ticket.

**Who:** Support agents. Used to issue customer LLTs and revoke them.

**Still needed:** all three operations were used in production in the last 30 days. This ticket owns **list + invalidate** only. Create is DE-5205.

**Callers of leftover `GET`/`PUT /internal/tokens`:** HootSupport Token Management and helper-scripts `redacted.sh` (DE-5204). No other service owner.

**Environment:**

- Staging New Hydra is **OEL 26.3.18**.
- `hydra-oss-admin.staging.hootops.com` replicas 0 / 503 is **New Hydra leftover OSS admin**. It is **unrelated** to leftover Token Management. **Must not** be used for list, invalidate, leftover dual-read, or Bruno proof.
- Leftover `/internal/tokens` is the **legacy Hydra fork**, env `HYDRA_ENDPOINT`:
  - Prod: `https://hydra-api-auth.hootops.com`
  - Staging: existing HootSupport leftover config (same `HYDRA_ENDPOINT` class). **Not** `hydra-oss-admin`.
- New Hydra token path is **Lerna** via Aperture public rewrite:
  - Staging public: `https://staging-platform.hootsuite.com`
  - Prod public: existing HootSupport `hootsuite.hydra.token_url` (`platform.hootsuite.com` class — same host already used to mint).
- Client writes (DE-5204) go to New Hydra **admin** directly. Tokens stay on Lerna.
- Published Lerna runbook endpoints today: `/public/v1/oauth2/auth`, `/public/v1/oauth2/token`, `/v1/hooks/token`. **No** `/oauth2/tokens` list yet. Adding the runbook line is a **follow-up**, not a merge gate (§8, §10).
- Lerna cleanup already `DELETE`s `hydra_oauth2_access` / `hydra_oauth2_refresh`. That is the SQL precedent for Path A. This ticket’s DELETE is **access-row only**.
- `member_app` / `organization_app` **never** issue refresh tokens (`offline` / `offline_access` stripped).
- Staging TMA client id: `4e3be522-b7e9-434b-bcd7-69640b15fe6a` (not a secret; required for Bruno).
- Prod TMA client id (DE-5204 Done ticket, not a secret): `e303d588-c9cc-4a3e-a28c-6e375464e371`.
- TMA secret stays in Vault (`PLATFORM_TOKEN_MANAGEMENT_CLIENT_SECRET`). **Must not** appear in this spec, Jira, or PR bodies.

**Stock Hydra 26.3.18 cannot power the widget.** Independently confirmed:

| Primitive | Usable for the widget? |
| --- | --- |
| `GET /admin/oauth2/tokens` | Does not exist |
| `GET /admin/oauth2/auth/sessions/consent?subject=` | Consent sessions only. jwt-bearer / `member_app` has no consent row → empty |
| `DELETE /admin/oauth2/auth/sessions/consent?subject=` | No-op for jwt-bearer on 26.3.18 |
| `DELETE /admin/oauth2/tokens?client_id=` | Deletes **every** access token for that client (all TMA members). Unusable |
| `POST /admin/oauth2/introspect` | Needs the **raw bearer**. Cannot list. Must not store `ory_at_`. **Is** the revoke oracle after DELETE (§7.3) |
| `POST /oauth2/revoke` | Needs the **raw bearer** + client auth. Path C only |

OEL changelog through 26.3.x adds indexes and revoke-performance work, **not** a token-list API. v26.2.7 added `(nid, subject, client_id)` on `hydra_oauth2_access` / `hydra_oauth2_refresh` — that helps Path A SQL; it does not replace it. List/delete SQL in this spec **must not** add `nid` unless Lerna’s existing cleanup job already binds `nid` — then copy that exact pattern (§5.3).

Hydra **26.4.4** (2026-10-21) makes consent revoke also delete jwt-bearer tokens for that subject (or subject+client). Staging is **26.3.18**, so consent flush is still a no-op. Even after 26.4.4 that API is still **flush-all** (Path B), not per-token list. Upgrading Hydra does **not** unlock Path A without Lerna.

---

## 3. Chosen approach and why

**Path A is mandatory.** Paths B and C are rejected. Do not reopen them.

**Path A — thin Lerna list + one-token revoke**

Lerna already SQL-touches `hydra_oauth2_access` (cleanup job). Expose **list** and **one-row delete** over HTTP so the widget keeps per-token UX without storing the bearer.

Ticket originally marked “New Lerna admin CRUD” out of scope because it assumed OSS had a list. OSS does not. The HTTP exception is **required**.

**Why A wins**

- Stock admin has no list-by-subject.
- Introspect/revoke need the raw bearer. Persisting `ory_at_…` is forbidden.
- Client-wide `DELETE /admin/oauth2/tokens?client_id=` is not per-token.

**Path B — flush-all — rejected.** Support cannot revoke one leaked LLT. Invalidating one row would kill every other TMA LLT for the member (and on 26.4.4, possibly every jwt-bearer for that subject). Does not match the widget.

**Path C — paste-revoke — rejected.** Works only while the caller still has the plaintext. No list after the create modal is dismissed. Persisting the bearer is forbidden.

**Identifier correction (mandatory).** Ticket Path A comment 2035207 claims:

> handle = last `.` of `ory_at_<key>.<hmac>`; Lerna `SELECT signature` returns that HMAC

That claim is **false** for every token minted on Hydra 26.x after DE-5205. See §4. A spec or PR that copies the ticket identifier will ship a list that cannot invalidate from a minted token’s last segment, and an invalidate that 404s if HootSupport sends the HMAC instead of the stored hash (or the reverse).

Existing draft PR test plans on the ticket still say “signature equals the last `.` segment (HMAC)”. Those pass-when lines **must** be corrected when the PRs are adjusted.

---

## 4. Identifier contract (list handle vs HMAC vs create-response bearer)

Three different strings exist. They **must not** be treated as interchangeable.

Opaque TMA jwt-bearer token format on Hydra 26.x (default; TMA issues these, not JWTs):

```text
ory_at_<base64url-key>.<base64url-hmac>
```

Fosite `HMACStrategy.Signature(token)` splits on `.` and, when there are exactly two parts, returns `split[1]` = the HMAC. The `ory_at_` prefix lives in part `[0]`, so the last `.` segment **is** the HMAC. That part of the ticket is true.

Hydra then stores:

```text
CreateAccessTokenSession(signature) → INSERT signature = x.SignatureHash(signature)
```

`x.SignatureHash` (Ory `x/sighash.go`):

```text
hex( SHA-384( fosite_hmac_signature ) )
```

Go equivalent (hash **in-process**, not a SQL function):

```text
fmt.Sprintf("%x", sha512.Sum384([]byte(sig)))
```

Lookup (`GetAccessTokenSession` / `DeleteAccessTokenSession`) hashes first, then falls back to the raw value for pre-hash rows (Hydra < ~2.1). Ory hydra#3485 documents the same change.

Therefore `hydra_oauth2_access.signature` on Hydra 26.x is **not** the last token segment. It is **`hex(SHA-384(HMAC))`**: 96-character lowercase hex. Call this the **stored signature column / access-token row id**. **Must not** call it “the PK” unless a GHE review of the live schema shows that is the real key. List **must** return that column as-is, including pre-hash raw HMAC values if any such New Hydra rows exist.

| Source | Value | Invalidate handle? |
| --- | --- | --- |
| List API `tokens[].signature` | Stored `hydra_oauth2_access.signature` column. On 26.x: 96-char lowercase SHA-384 hex. On pre-hash New Hydra rows: raw HMAC. | **Yes — canonical** for **Lerna-sourced** rows. #824 **must** send this string to Lerna DELETE. |
| Minted bearer last `.` | Raw HMAC (base64url, ~43 chars) | **Lerna DELETE input only** (hash-then-fallback). **Must not** be what list returns. **Not** a #824 create-path must. |
| #819 create `signature` field | Full `ory_at_…` | **No.** Show-once copy field on DE-5205. #824 invalidate **must reject** any handle that starts with `ory_at_` (400) before DELETE/PUT. |
| Leftover list `signature` | Legacy JWT payload / leftover `key.signature` | **Leftover-sourced** → leftover `PUT /internal/tokens` after Lerna handler-404 (and after Lerna 204 if **both-sourced**, success only if PUT succeeds). |

### 4.1 Source classifier (normative for #824 — not regex)

**“Last successful merge” / “last leftover 200”** = the dual-read that #824 **must** run **on this request** (list or invalidate). Invalidate **must not** persist merge, `source`, or leftover membership across HTTP requests. **Must not** add list/invalidate fields for `source`. Classify from that in-request merge only.

| Seen on this request’s merge | Class | After Lerna DELETE |
| --- | --- | --- |
| Lerna list row only | **Lerna-sourced** | 204 → **no** leftover PUT. Next list: Lerna row gone. Handler-404 → agent 404, **no** leftover PUT. |
| Leftover list only | **leftover-sourced** | Handler-404 expected. Leftover-PUT that exact string (after extra leftover GET if required — §6.3.1). Agent success **iff** PUT succeeds. |
| Both lists (exact `signature` collision) | **both-sourced** | 204 or handler-404 → leftover-PUT that exact string. Agent success **iff** PUT succeeds. If PUT fails after Lerna 204: leftover error; New Hydra row is already gone; leftover copy **may** remain. |
| Not on this request’s merge (agent-typed) | If exactly `[0-9a-f]{96}` → **Lerna-sourced**. Otherwise → **leftover-sourced**. | Same rows as above. |

Notes:

- A listed **pre-hash New Hydra HMAC** is **Lerna-sourced** even though it is not 96-char hex. **Must not** leftover-PUT it after Lerna handler-404.
- A leftover `signature` that happens to be 96 lowercase hex and was seen **only** on this request’s leftover 200 is **leftover-sourced**. **Must** leftover-PUT after Lerna handler-404.
- List returns lowercase hex. Uppercase `[0-9A-F]{96}` is **not** the typed Lerna-sourced regex. An agent-typed uppercase hex handle is **leftover-sourced**. A typed lowercase-hex Lerna handle sent in uppercase therefore leftover-PUTs or 404s on leftover — **intentional**, not a bug to “fix” by case-folding.
- Regex is **only** for agent-typed handles absent from **this request’s** merge. UI clicks send `{signature}` only; invalidate re-classifies via dual-read.

### 4.2 Other identifier rules

1. List **must** return the **stored signature column**. **Must not** reverse-hash. **Must not** rewrite the column to the HMAC. **Must not** return the bearer, `request_id`, `session_data`, `form_data`, `active`, `client_id`, or `subject`.
2. The canonical UI handle for Lerna-sourced rows is `tokens[].signature` as returned. Product copy **may** still call it “signature”.
3. Lerna DELETE **must** accept either the stored column value **or** the raw HMAC (last `.` of `ory_at_…`) via hash-then-fallback.
4. **#824 musts stop at list + invalidate.** **Must not** require post-mint re-GET or PHP `hash('sha384', last_segment)` in DE-5216.
5. #824 invalidate **must reject** a path/body handle that **starts with** `ory_at_` with HTTP **400** (existing HootSupport error envelope) **before** dual-read DELETE/PUT. Test lives on #824 invalidate. **Must not** require editing #819 helpers.
6. Persist and create-modal tests stay on DE-5205 / #819.

---

## 5. Lerna API contract (auth, routes, SQL, request/response, errors, hashing)

Implement in **hydra-wrapper #63**.

### 5.0 Query constraints (copy into §5.6, §8, §9.4)

**No trim.** Only a missing param or a value that is empty or **all whitespace** is “empty.” Leading/trailing spaces on an otherwise valid value are kept and hashed/queried **as-is** (typically 404).

| Param | GET | DELETE |
| --- | --- | --- |
| `subject` | Required. Non-empty (not whitespace-only). Length ≤ 128. | Same. |
| `signature` | Not used. | Required. Non-empty (not whitespace-only). Length ≤ 256. |

**Validation order (after successful auth).** Auth fail is always **401** (or **502** per §5.1) **before** param checks.

1. `subject` missing / empty / whitespace-only → 400 `invalid_request` naming `subject`.
2. `subject` length > 128 → 400 `invalid_request` naming `subject`.
3. DELETE `signature` missing / empty / whitespace-only → 400 `invalid_request` naming `signature`.
4. DELETE `signature` length > 256 → 400 `invalid_request` naming `signature`.

DELETE missing both params therefore names **`subject` first** (step 1). Stacked failures **must not** be combined into one body; return the first failure in this list.

### 5.1 Auth (one gate)

These routes are a **generic per-`client_id` admin**, not TMA-only.

1. Caller **must** send HTTP Basic (`client_id:client_secret`). Missing or malformed Basic → **401** `invalid_client`.
2. Lerna **must** probe Hydra with that Basic:
   - Method: `POST`
   - URL: Lerna’s **already configured Hydra public token URL** (the OEL/Hydra public `/oauth2/token` host Lerna already uses for grants). **Must not** self-call `https://staging-platform.hootsuite.com` or the prod Aperture/Lerna public host.
   - Headers: `Content-Type: application/x-www-form-urlencoded`
   - Body: `grant_type=client_credentials` **only** (no scope field).
   - Auth: the incoming request Basic, forwarded as Basic.
3. Probe outcomes:
   - Hydra **200** whose JSON includes `access_token` → authenticated.
   - Hydra **400** or **401** → Lerna **401** `invalid_client`.
   - Any other Hydra outcome (timeout, 3xx, 403, 429, 5xx, HTTP 200 **without** `access_token`, connect fail) → Lerna **502** `temporarily_unavailable` (same error token as existing Lerna custom-grant exchange failures).
4. SQL `client_id` **must** be the Basic username.
5. Non-TMA clients are **allowed**. They see only their own rows (`200` with `[]` or their tokens). **Must not** TMA-allow-list. **Must not** 401 a valid non-TMA client.
6. **Must not** require leftover scope `hydra.oauth2.tokens`. If that scope appears on a CC token, **ignore** it.
7. Caching the probe is **optional** and **not** required.
8. **Must not** expose these routes as OSS admin no-auth.
9. HootSupport Token Management **will** send TMA Basic (staging `4e3be522-b7e9-434b-bcd7-69640b15fe6a`, prod `e303d588-c9cc-4a3e-a28c-6e375464e371`). That is a **caller** choice, not a Lerna allow-list.
10. Lerna **intentionally** lets any holder of a valid client secret list/revoke **any** `subject` for **that** `client_id`. Agent→member authorization stays in HootSupport (`manage_access_tokens` **and** member-page ownership of `{id}`). **Must not** add a Lerna-side agent ACL.

### 5.2 Routes and Aperture rewrite

| Method | Lerna path | Public path callers use |
| --- | --- | --- |
| GET | `/public/v1/oauth2/tokens?subject={memberId}` | `/oauth2/tokens?subject={memberId}` |
| DELETE | `/public/v1/oauth2/tokens?subject={memberId}&signature={handle}` | `/oauth2/tokens?subject={memberId}&signature={handle}` |

| Env | Public Lerna (Aperture) | Leftover `/internal/tokens` | Introspect (oracle) |
| --- | --- | --- | --- |
| Staging | `https://staging-platform.hootsuite.com` | HootSupport leftover `HYDRA_ENDPOINT` (not `hydra-oss-admin`) | `https://hydra-oel-admin.staging.hootops.com/admin/oauth2/introspect` |
| Prod | Existing HootSupport `hootsuite.hydra.token_url` (`platform.hootsuite.com` class) | `https://hydra-api-auth.hootops.com` (`HYDRA_ENDPOINT`) | Prod OEL admin introspect (same class as DE-5205 prod). **Not** required for staging Done-when |

**Rewrite owner (same #63 deploy change-set, not a fifth product PR):**

1. Use the **same** Aperture / IngressRoute prefix that already maps `POST /oauth2/token` → Lerna `/public/v1/oauth2/token`.
2. **Confirmation artifact (the deploy gate).** Both curls **must** return Lerna JSON 400:
   - `GET https://staging-platform.hootsuite.com/oauth2/tokens` with TMA Basic, **no** `subject` → **400** `invalid_request` naming `subject`.
   - Authenticated `DELETE https://staging-platform.hootsuite.com/oauth2/tokens?subject={validMemberId}` with **missing or empty** `signature` → **400** `invalid_request` naming `signature`.
   - HTML or vanilla **404** (no Lerna JSON) on either curl → rewrite miss. **Must** add `GET|DELETE /oauth2/tokens` in `deployments/hydra-wrapper` or the Aperture IngressRoute that already owns `/oauth2/token`, **in the same deploy change-set as #63**. Open a separate repo PR **only if** that IngressRoute cannot be edited from the hydra-wrapper PR.
3. Deploy curls are a **gate**, not #824 runtime state. Runtime **must** use the **body predicate** (§5.2.1).
4. This is **not** the DE-5203 Aperture swap.

**5.2.1 Runtime 404 kinds (#824 / Bruno — body predicate, not remembered curl)**

| HTTP 404 body | Kind | #824 |
| --- | --- | --- |
| JSON `{"error":"not_found","error_description":"token not found"}` | **Handler 404** | Classify per §4.1 / §6.3 (leftover-PUT or agent 404) |
| Non-JSON / HTML / empty / vanilla 404 | **Rewrite miss** | Fail **502**. **Must not** leftover-PUT. **Must not** treat as agent not-found |

Same predicate on Lerna **GET** 404 and Lerna **DELETE** 404. A DELETE-only rewrite miss is therefore **502**, not leftover-PUT.

- **Must not** add `PUT /internal/tokens`, PATCH-expiry, or any write other than this DELETE.
- **Must not** add `GET /admin/oauth2/tokens` on OSS.
- Query params only. **Must not** require a JSON body on GET or DELETE.

### 5.3 SQL

Hash **in-process**. **Must not** rely on a SQL function named `SHA384_HEX`.

```text
$hash = fmt.Sprintf("%x", sha512.Sum384([]byte(input)))
```

Use the **same Hydra MySQL connection** the Lerna cleanup job already uses. **Must not** add a replica-only GET path.

**Must not** add `nid` unless that cleanup job already binds `nid`. If it does, copy that exact bind. If it does not, omit `nid`.

**List**

```sql
SELECT signature, expires_at
FROM hydra_oauth2_access
WHERE client_id = ? AND subject = ?
ORDER BY expires_at DESC, signature ASC
LIMIT 201
```

Binds: `$client_id` = Basic username; `$subject` = validated `subject` query param (as-is, no trim).

- **Must** include expired and `active=0` rows (do not add those predicates).
- **Must not** paginate.
- Cap: `LIMIT 201`. If 201 rows return, **must** log cap-hit and return the **first 200** in that `ORDER BY`. The **201st** row in `expires_at DESC, signature ASC` **must** be the dropped row. **Must not** add a client warning field or header (log-only).
- NULL `expires_at` sorts **last** under `DESC` (MySQL default). That **must** match merge “missing expiry last” (§6.2.1). **Must not** add `NULLS LAST` unless the dialect requires it to get the same order.
- If ≤200 rows return, return them all. 200 exactly is **not** a cap-hit.
- **Must not** read or return `request_id`, `session_data`, `form_data`, or any bearer-bearing column.

**`expires_at` → `expiry`**

- Convert to UTC, second precision, `YYYY-MM-DDTHH:MM:SSZ` (example `2126-10-20T22:28:51Z`).
- NULL `expires_at` → **omit** the `expiry` field (field absent, not `null`).
- Driver fractional seconds: **truncate toward zero** (e.g. `…51.9` → `…51Z`, not `…52Z`).
- Present `expiry` **must** match `^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$`.

**Delete (access-row only)**

```sql
DELETE FROM hydra_oauth2_access
WHERE client_id = ? AND subject = ?
  AND signature IN (?, ?)
```

Binds: `$client_id`, `$subject`, `$hash`, `$input` (validated `signature` query param, as-is).

- **Must not** add `LIMIT` on DELETE. 204 if **≥1** row deleted. Handler-404 if 0 rows.
- **Must not** touch `hydra_oauth2_refresh`. Non-TMA callers **must not** assume this is a full Hydra revoke.
- **Must not** omit `client_id` or `subject`.

### 5.4 Hashing

Input = validated DELETE `signature` query param (no trim):

1. `$hash = hex(SHA-384(input))` (lowercase, no separators) in-process.
2. Match column `signature` = `$hash` **or** column `signature` = `$input`.
3. If the caller sent an already-hashed list handle, `$input` matches the column. `$hash` **typically** does not match another row; if it does, DELETE **may** remove ≥1 row (204 is still correct).
4. If the caller sent the raw HMAC, `$hash` matches 26.x hashed rows and `$input` matches pre-hash rows.

**Must not** invent a reverse of SHA-384.

### 5.5 Request / response

**GET success — 200** `Content-Type: application/json`

```json
{
  "tokens": [
    {
      "signature": "<stored hydra_oauth2_access.signature>",
      "expiry": "2126-10-20T22:28:51Z"
    }
  ]
}
```

- Empty result **must** be `200` with `"tokens": []`. **Must not** be 404.
- `expiry` **may** be absent when `expires_at` is NULL.
- Response **must not** include the bearer, `request_id`, `session_data`, `form_data`, `active`, `client_id`, `subject`, or a cap-warning field.

**DELETE success — 204**

- Lerna **should** return an empty body.
- #824 **must** treat **204** as Lerna-row-deleted and **ignore** any body. Agent success still depends on leftover PUT when the class requires PUT (§6.3).
- **Must** mean ≥1 matching access row was deleted.
- **Must never** return 204 on 0 rows, or on missing/empty/whitespace/overlong params.

### 5.6 Errors

All 400 / 401 / 404 / 502 bodies: `Content-Type: application/json`.

| Condition | Status | Body |
| --- | --- | --- |
| Missing or invalid Basic; Hydra CC 400/401 | 401 | `{"error":"invalid_client","error_description":"Client authentication failed"}` |
| Probe timeout / 3xx / 403 / 429 / 5xx / 200 without `access_token` / connect fail | 502 | `{"error":"temporarily_unavailable","error_description":"Hydra client_credentials probe failed"}` |
| `subject` missing, empty, or whitespace-only | 400 | `{"error":"invalid_request","error_description":"missing or empty subject"}` |
| `subject` longer than 128 | 400 | `{"error":"invalid_request","error_description":"subject too long"}` |
| DELETE `signature` missing, empty, or whitespace-only | 400 | `{"error":"invalid_request","error_description":"missing or empty signature"}` |
| DELETE `signature` longer than 256 | 400 | `{"error":"invalid_request","error_description":"signature too long"}` |
| Unknown handle, or handle not owned by this `client_id` + `subject`; 0 rows deleted | 404 | `{"error":"not_found","error_description":"token not found"}` |

Query constraints: **copy of §5.0**. Validation order: **copy of §5.0**. Auth (401/502) always before param checks.

- Cross-client or cross-subject miss **must** be handler-404 with the body above, not 403.
- **Must not** echo the bearer, client secret, or member PII.
- Empty `signature=` **must** 400 **before** SQL.

### 5.7 What Lerna must not do

- **Must not** invent `PUT /internal/tokens` or PATCH-expiry on OSS/Lerna.
- **Must not** call `DELETE /admin/oauth2/tokens?client_id=` (client-wide flush).
- **Must not** call consent-session flush as a substitute for one-row delete.
- **Must not** delete `hydra_oauth2_refresh` rows.
- **Must not** return or log `ory_at_…`.
- **Must not** require leftover scope `hydra.oauth2.tokens`.
- **Must not** TMA-allow-list.
- **Must not** self-call Aperture public Lerna to verify Basic.

### 5.8 Automated tests required on #63

#63 CI **must** prove isolation, cap (including **which** row is dropped), empty/whitespace/overlong params, stacked validation order, probe 502, expiry regex **and** fractional truncation, and hygiene. Bruno is **not** that proof (§7). Bruno **may** still call other-client and empty-`signature` requests; those calls **must not** be claimed as the isolation/empty-params proof.

| Case | Must assert |
| --- | --- |
| Auth 401 | Missing/invalid Basic → 401 body exactly §5.6; `Content-Type: application/json` |
| Probe 502 | Hydra timeout / 5xx / 200 without `access_token` → 502 `temporarily_unavailable` |
| Auth before params | No Basic + missing `subject` → **401**, not 400 |
| Non-TMA client | Valid other-client Basic → 200; rows are that `client_id` only |
| TMA isolation | TMA Basic lists only TMA rows |
| Cross-client | Client B GET excludes TMA signatures; Client B DELETE of a TMA handle → handler-404 and **0** rows deleted |
| Subject mismatch | TMA DELETE with another member’s `subject` → handler-404 and 0 rows deleted |
| GET empty | Unknown subject, valid auth → 200 `{"tokens":[]}` |
| DELETE by list handle | 204; row gone on next GET |
| DELETE by HMAC | 204; row gone |
| Repeat DELETE | Second DELETE → handler-404 JSON `not_found` |
| Cap 200 | 201 matching rows → 200 returned; **201st** in `expires_at DESC, signature ASC` absent; cap-hit logged; no warning field |
| Expiry shape | Every present `expiry` matches `^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$`; NULL `expires_at` omits `expiry`; fractional `expires_at` truncated toward zero |
| Empty / whitespace `subject` | 400 naming `subject`; no SQL |
| Empty / whitespace `signature` | 400 naming `signature`; **0** rows deleted |
| Both missing on DELETE | 400 naming `subject` |
| Stacked params | Whitespace `subject` + missing `signature` → 400 **subject** (not signature) |
| Overlong | `subject` >128 (even if `signature` missing/too long) → 400 subject; valid subject + `signature` >256 → 400 signature |
| No trim | `subject=%20valid-id%20` is **not** 400-empty; queried as-is. `signature=%20hmac%20` is **not** 400-empty; hashed/queried as-is |
| Response hygiene | GET never contains `ory_at_`, `request_id`, `active`, `client_id`, `subject`, `session_data`, `form_data`, or bearer columns |

---

## 6. HootSupport behavior (dual-read, invalidate fallback, handles, tests)

Implement in **hootsupport2 #824**. Git branch **must** stay unstacked on DE-5205 `#819`. **Must** ship in the **same HootSupport release** as #819 when create cutover happens (§10).

#824 owns **list + invalidate APIs only**. Invalidate route stays the existing `POST /api/member/{id}/access-tokens/{signature}/invalidate`. Path `{id}` **is** the member. Dual-read `memberId` / Lerna `subject` **must** be that `{id}`.

### 6.1 Auth and hosts from HootSupport

- **Lerna:** HTTP Basic with the **TMA** `client_id:client_secret`. Public path `/oauth2/tokens` on `hootsuite.hydra.token_url` (staging `https://staging-platform.hootsuite.com`).
- **Leftover:** existing client_credentials with scope `hydra.oauth2.tokens` against leftover `HYDRA_ENDPOINT` `GET`/`PUT /internal/tokens`. Prod: `https://hydra-api-auth.hootops.com`. Staging: existing leftover config. **Must not** call `hydra-oss-admin`.
- **Agent gate (unchanged):** `manage_access_tokens` **and** existing member-page ownership of `{id}`. If either fails, existing deny. **Stop.**
- Lerna GET and leftover GET **may** be sequential or parallel. The leftover result is **ignored unless Lerna GET is 200** (same agent outcome as “leftover only if Lerna 200”).

### 6.2 Dual-read list (until DE-5180)

Until DE-5180, **list** and **each invalidate** **must** run this same dual-read. **Must not** persist the merge across requests.

1. Call Lerna `GET /oauth2/tokens?subject={id}`.
2. Call leftover `GET {HYDRA_ENDPOINT}/internal/tokens?memberId={id}` (result used only if Lerna GET is 200).
3. Apply §6.2.2, then merge per §6.2.1 when Lerna is 200 and leftover is usable.
4. **Must not** require a leftover `key.signature` for Lerna-sourced rows.
5. **Must not** show the full bearer in the table.
6. **Must not** add `source` (or leftover-membership) fields to the list API.
7. After DE-5180, leftover GET **may** be removed. That removal is **not** this ticket.

`EnsureTokenManagementApp` **may** stay if product still requires the app install for the tab.

#### 6.2.1 Merge

- Dedup key = **exact** `signature` string. **Must not** transform identifiers.
- Collision → keep the Lerna row’s `expiry` (after Lerna mapping). Mark the handle **both-sourced** **in memory for this request only**.
- Leftover expiry: unix seconds **or** RFC3339 (any offset) → UTC `Z`. Missing or unparseable → **omit** `expiry`, **keep the row**. Already-`Z` stays. `+00:00` → `Z`.
- Sort: expiry **desc**; rows with missing expiry **last** (same as Lerna NULL `expires_at` last under `DESC`); tie-break `signature ASC`.

#### 6.2.2 Side-failure behavior

| Lerna GET | Outcome |
| --- | --- |
| 200 | Use body. Then apply leftover row. |
| 400 | Fail list **400** (bad subject). Existing HootSupport error envelope. **No token table.** Ignore leftover. |
| 401 | Fail list **401**. Existing envelope. **No token table.** Ignore leftover. |
| 404 **rewrite miss** (non-JSON / HTML / empty) | Fail list **502**. Existing envelope. **No token table.** Ignore leftover. |
| 404 JSON `not_found` | Unexpected on GET (unknown subject is 200 `[]`). Fail list **502**. Ignore leftover. |
| 429 / 5xx / timeout / 3xx / other (including 502/503) | Fail list **502**. Existing envelope. **No token table.** Ignore leftover. |

| Leftover GET (used only if Lerna GET 200) | Outcome |
| --- | --- |
| 200 (including `[]`) | Merge §6.2.1. This leftover 200 **is** “this request’s leftover 200.” |
| 404 / empty body / non-JSON / 400 / 403 / 429 / 3xx / 5xx / 401 / timeout | **Leftover-down:** HTTP **200** with Lerna tokens + `leftover_list_error: true` (or the **existing** HootSupport list warning field if one already exists — reuse, do not invent a second). UI **still renders** Lerna rows. Do **not** fail the page. This request has **no** leftover 200. |

Staging FIT leftover assertions: **only if leftover GET is 200** with at least one leftover-sourced row. Leftover-down or leftover `[]` → leftover-list / leftover-invalidate FIT is **skip**, not fail.

### 6.3 Invalidate

Route: existing `POST /api/member/{id}/access-tokens/{signature}/invalidate`.

**Order (normative):**

1. Require `manage_access_tokens` **and** existing member-page ownership of `{id}`. If either fails, existing deny. **Stop.**
2. If `{signature}` **starts with** `ory_at_` → HTTP **400** existing HootSupport error envelope. **Stop.** No dual-read. No Lerna DELETE. No leftover PUT.
3. **Re-run the same dual-read as list** (§6.2) with `subject` / `memberId` = `{id}`. **Must not** persist a prior list merge. This in-request merge **is** “last successful merge” / “this request’s leftover 200.”
4. Classify `{signature}` per §4.1 from **that** merge (regex only if absent from it).
5. **Always** Lerna `DELETE /oauth2/tokens?subject={id}&signature={signature}` next. Merged-list membership **must not** block this DELETE.
6. Branch Lerna DELETE on **status + body** (§5.2.1), then leftover PUT per the table. Extra leftover GET only per §6.3.1.

| Lerna DELETE | Class | Agent-visible | Leftover PUT? |
| --- | --- | --- | --- |
| 204 (body ignored) | Lerna-sourced | Existing success. Next list: Lerna row **gone**. | **No** |
| 204 | both-sourced | Existing success **iff** leftover PUT succeeds. If PUT fails: existing leftover error (Lerna row already gone; leftover copy **may** remain). “Next list: leftover copy gone” **only** when PUT succeeded. | **Yes** — PUT that exact string |
| 204 | leftover-sourced | Existing success **iff** leftover PUT succeeds; else existing leftover error. | **Yes** |
| Handler-404 (JSON `not_found`) | leftover-sourced or both-sourced | Existing success **iff** leftover PUT succeeds; else existing leftover error. | **Yes** |
| Handler-404 (JSON `not_found`) | Lerna-sourced | HTTP **404** existing not-found envelope. | **No** |
| Rewrite-miss 404 (non-JSON / HTML / empty) | any | Fail **502**. Existing envelope. | **No** |
| 400 | any | Fail **400**. Existing envelope. | **No** |
| 401 | any | Fail **401**. Existing envelope. | **No** |
| 403 / 429 / 5xx / timeout / 3xx / other | any | Fail **502**. Existing envelope. | **No** |

Leftover PUT body remains `{signature, expiry: now}` using the **exact** handle string.

An agent-typed uppercase `[0-9A-F]{96}` handle is leftover-sourced (§4.1). Handler-404 then leftover-PUT (or leftover 404) is **intentional**. **Must not** lowercase the handle to force Lerna-sourced.

**Must not** remint. **Must not** write or delete `llt_map` rows.

#### 6.3.1 Extra leftover ownership GET

Run this GET **only when** leftover PUT is required (leftover-sourced or both-sourced) **and** the handle was **not** on **this request’s** leftover 200 (leftover-down, leftover `[]`, or agent-typed leftover-sourced). If the handle **was** on this request’s leftover 200, **must** PUT without a second leftover GET.

| Extra leftover GET | Next |
| --- | --- |
| 200 **and** that exact handle is present | leftover PUT |
| 200 **and** handle absent | HTTP **404** existing not-found. **No PUT.** **No** agent success. |
| 401 / 403 / 404 / 429 / 5xx / timeout / non-JSON / empty | Existing leftover error. **No PUT.** **No** agent success. |

### 6.4 Handle derivation (#824)

1. UI rows: use the row’s `signature` as-is in `POST /api/member/{id}/access-tokens/{signature}/invalidate`. **Must not** send `source`. Invalidate re-classifies via the in-request dual-read.
2. **Must not** show or store the bearer.
3. **Must not** use `explode('.', $token)[1]` for Lerna-sourced rows.
4. Post-mint re-GET and PHP-hash-before-relist are **not** #824 musts.

### 6.5 Create path

- **Unchanged on DE-5216.**
- **Must not** call `internal/tokens` on create (create is #819).
- **Must not** fold #819 into #824.
- Persist / create-modal / “minted bearer not written to DB” tests stay on **DE-5205 / #819**.
- Shared-helper / create-path `ory_at_` coverage is **#819 or a same-release merge note**, not a #824 must that requires editing create.

### 6.6 Tests (HootSupport #824)

#824 **must** add or adjust automated tests for all of the following. Mock Lerna and leftover.

| Case | Must assert |
| --- | --- |
| Dual-read merge | Leftover + Lerna rows; exact-string dedup; both-sourced in-memory; sort expiry desc, missing expiry last, `signature` ASC |
| Leftover expiry map | Unix and offset RFC3339 → UTC `Z`; missing/unparseable → omit `expiry`, row kept |
| New Hydra-only row | List works with **no** leftover `key.signature` |
| Leftover-down | Leftover 401/404/5xx/timeout/non-JSON → HTTP 200 + Lerna tokens + `leftover_list_error` (or existing warning); page renders |
| Lerna GET 400 | List 400, no table; leftover ignored |
| Lerna GET 401 | List 401, no table |
| Lerna GET rewrite-miss 404 | Non-JSON 404 → list 502, no table |
| Lerna GET 5xx/429/timeout | List 502, no table |
| Empty both 200 | Empty list, no error |
| Invalidate re-runs dual-read | Each invalidate calls Lerna GET + leftover GET; no stored merge from the prior list call |
| Invalidate Lerna-sourced 204 | Lerna DELETE; leftover PUT **not** called; next list **gone** |
| Invalidate both-sourced 204, PUT ok | Lerna DELETE then leftover PUT; next list gone on **both** sides; agent success |
| Invalidate both-sourced 204, PUT fail | Agent leftover error (not success); Lerna row gone; leftover copy may remain |
| Invalidate both-sourced handler-404 | Leftover PUT; success iff PUT ok |
| Invalidate leftover-sourced handler-404 | Leftover PUT; extra leftover GET **only if** handle was not on **this request’s** leftover 200 |
| Invalidate leftover-sourced Lerna 204 | Leftover PUT; success iff PUT ok |
| Handle on this leftover 200 | leftover-sourced/both-sourced → PUT **without** a second leftover GET |
| Extra GET 200 handle absent | 404 not-found; **no PUT** |
| Extra GET fail | leftover error; **no PUT** |
| Invalidate Lerna-sourced handler-404 | No leftover PUT; agent 404 |
| Rewrite-miss DELETE 404 | Non-JSON 404 → 502; **no PUT** (even leftover-sourced) |
| Pre-hash Lerna HMAC row | Class Lerna-sourced; handler-404 → no leftover PUT |
| Typed uppercase hex | Class leftover-sourced (intentional) |
| `ory_at_…` handle | 400 **before** dual-read DELETE/PUT |
| Lerna DELETE 400/401 | Fail 400/401; no leftover PUT |
| Lerna DELETE 403/429/5xx/timeout | Fail 502; no leftover PUT |
| Ownership | `manage_access_tokens` **and** member-page `{id}` first; dual-read/`subject` use `{id}`; merged-list membership does **not** block Lerna DELETE |
| Logs | No `ory_at_`, no TMA secret, no leftover CC token |

**Must not** add tests that require `PUT /internal/tokens` on Lerna/OSS. **Must not** add persist, create-modal, or #819-helper tests on #824.

**UI / FIT (staging, after #63 + both confirmation curls 400 + #824 deploy):**

- **Mint for FIT:** Bruno `member_app` as TMA on the **same** `{id}` after #63 (keep bearer local). **Or** widget Create **only if** #819 is already on staging. **Must not** require widget Create for #824 FIT.
- Member → Login settings → Access tokens lists that Lerna-minted **stored** signature (96-char hex on 26.x). **No** full `ory_at_` in the table.
- Leftover rows / leftover-invalidate: assert **only if leftover GET is 200** with a leftover-sourced row. Otherwise **skip**.
- Invalidate the Lerna-sourced row: it is **gone** from the next list; leftover-only rows stay (when leftover GET 200).
- Introspect oracle: `"active": false`. Aperture 401 is secondary; record DE-5203 follow-up if Aperture still allows.
- **Fail when:** UI empty after the Bruno (or #819) Lerna mint; invalidate one Lerna-sourced row clears every token; UI shows the full bearer.
- **Create-path regression:** **not** a FIT case. Prove by code review that #824 adds no create PUT. FIT **must not** require widget Create.

---

## 7. Bruno proof contract

Implement in **bruno-api-collection #3**. This PR does **not** deploy a service.

**Bruno is the staging identifier + revoke-oracle subset.** #63 CI proves isolation, cap, empty-params, stacked validation, probe 502, expiry regex/truncation, and hygiene. Bruno **may** include other-client and empty-`signature` calls; it **must not** claim those as the isolation/empty-params proof. Ticket Done-when staging comment is the Bruno subset plus a pointer that #63 CI is green.

### 7.1 Environment

- Collection env: **Lerna / Staging**.
- `BaseURL` **must** be `https://staging-platform.hootsuite.com`.
- Requests **must** hit `GET`/`DELETE {{BaseURL}}/oauth2/tokens` **after** both §5.2 confirmation curls return **400**.
- **Must not** point list/delete at leftover hosts or `/internal/tokens`.
- Tester **must** override to staging TMA `4e3be522-b7e9-434b-bcd7-69640b15fe6a` plus Vault secret. **Must not** commit the secret.
- Isolation: a **second** non-TMA client Basic. **Must not** commit that secret.
- `Member_ID` **must** be a staging member the tester may manage. This is the same `{id}` #824 FIT uses after this mint.
- Proofs **must** run after **shared** hydra-wrapper #63 deploy. Jenkins: `https://jenkins.build.hootops.com/job/Platform/job/hydra-wrapper/job/hydra-wrapper/job/DE-5216-lerna-token-list/`.
- **Must not** require HootSupport #824 for the Lerna proof.
- Consent-flush **must not** be used as this proof.
- Leftover `hydra-oss-admin.staging.hootops.com` **must not** be a proof host.

### 7.2 Identifier observation (mandatory)

After mint + list, observe **without** pasting bearers into Jira:

1. `list.signature` **≠** last `.` segment of the minted `ory_at_…`.
2. `list.signature` **=** lowercase `hex(SHA-384(last HMAC))` (96 hex chars).

**Stop** if they are equal (not Hydra 26.x hashing) or if DELETE-by-HMAC 404s (missing fallback).

Ticket comment 2035304 “signature equals the last `.` segment (HMAC)” is **wrong**. Update Bruno #3 and the #63 comment.

### 7.3 Required sequence

Keep bearers local. **Must not** paste `ory_at_…`, TMA secrets, or member PII into Jira, PR bodies, or committed Bruno defaults.

1. Mint via Lerna `member_app` as TMA (`POST {{BaseURL}}/oauth2/token`). Keep the bearer local. This mint is the **#824 FIT setup** for that `{id}`.
2. `GET {{BaseURL}}/oauth2/tokens?subject={{Member_ID}}` with TMA Basic → **200** `{tokens:[{signature,expiry}]}`.
3. Perform §7.2.
4. `DELETE` with `{{list.signature}}` → **204**.
5. **Revoke oracle:**

```http
POST https://hydra-oel-admin.staging.hootops.com/admin/oauth2/introspect
Content-Type: application/x-www-form-urlencoded

token={bearer}
```

No `Authorization` header, no Basic. Response **200** JSON **must** include `"active": false`. **Must not** paste `{bearer}` in Jira.

6. **Secondary:** `GET https://aperture-authz.staging.hootops.com/authz/oauth` with that bearer. Expect **401**. If introspect is inactive and Aperture still **allows**, record DE-5203 follow-up. **Must not** fail #63.
7. Mint a **second** token for the same member. List **must** include the new row. Other listed signatures **must** remain.
8. Repeat DELETE of the already-revoked list handle → **404** `{"error":"not_found","error_description":"token not found"}`.
9. Mint another token; `DELETE` with the **last HMAC** → **204**. **Then repeat step 5** on that bearer → `"active": false`.
10. Negatives (keep the calls; **do not** claim them as #63 isolation/empty-params proof):
    - Missing Basic → **401** `invalid_client`.
    - Missing `subject` → **400** naming `subject`.
    - Empty `signature=` (valid `subject`) → **400** naming `signature`; **0** rows deleted.
    - Unknown signature → handler-404 JSON `not_found`.
    - Other-client Basic `GET` → **200** excluding TMA signatures.
    - Other-client Basic `DELETE` of the TMA list handle → handler-404; TMA re-GET still has the row.

**Fail when:** response contains `ory_at_`; list empty after a known mint; DELETE removes every TMA token; `DELETE /admin/oauth2/tokens?client_id=` was used; empty `Token_Signature` deletes any row; list/delete pointed at leftover hosts; introspect after list-handle DELETE **or** HMAC-DELETE lacks `"active": false`.

Aperture-still-allows is a **follow-up**, not a #63 fail.

### 7.4 Jira proof comment

- Hosts: `https://staging-platform.hootsuite.com`, `https://hydra-oel-admin.staging.hootops.com/admin/oauth2/introspect`, Aperture secondary.
- Both confirmation curls → 400 (GET no subject **and** DELETE empty signature).
- HTTP statuses only. Identifier observation (SHA-384 ≠ HMAC). HMAC-DELETE + introspect `active: false`.
- Pointer that #63 CI covers isolation/cap/empty-params.
- Aperture result or “DE-5203 follow-up”.
- **No** tokens, secrets, or member PII.

---

## 8. IDL / OpenAPI contract

Implement in **IDL-central #1328**. Docs / Jenkins spec mirror **only**.

#1328 **must** describe:

| Item | Contract |
| --- | --- |
| Paths | `GET` and `DELETE` `/public/v1/oauth2/tokens`; public rewrite `/oauth2/tokens` |
| Auth | HTTP Basic; probe §5.1; any client that passes CC; leftover scope **ignored**; **not** TMA-allow-listed; **not** OSS admin no-auth |
| Query | **Copy of §5.0** (GET `subject` required, non-empty, ≤128; DELETE `subject` same; DELETE `signature` required, non-empty, ≤256; no trim; validation order) |
| `tokens[].signature` | Stored signature column / access-token row id. On Hydra 26.x: **96-char lowercase hex SHA-384 of the Fosite HMAC**. Pre-hash rows may be raw HMAC. **Not** “HMAC last segment”. **Not** the bearer. **Not** “the PK” |
| DELETE `signature` | Stored column **or** raw HMAC (`IN ($hash, $input)`). Access-row only; **does not** delete refresh tokens |
| `tokens[].expiry` | UTC `YYYY-MM-DDTHH:MM:SSZ` when present; omit if `expires_at` NULL |
| Success | GET 200 `{tokens:[…]}`; empty `[]`; DELETE 204 (empty body preferred) |
| Errors | `Content-Type: application/json`. 401 example §5.6. 400 `invalid_request` as §5.6. Handler-404 `not_found` JSON as §5.6. 502 `temporarily_unavailable`. Never 204 on 0 rows or empty params |
| Cap | `LIMIT 201`; return 200; log-only; no client warning |
| Filter | All access rows for `client_id + subject`, including expired / inactive |
| Out of this API | `PUT /internal/tokens`; refresh-table ops; client-wide admin delete; HootSupport list `source` fields |

If the current #1328 draft says “HMAC / last segment” or “PK”, **must** correct before merge.

Lerna runbook update for `/public/v1/oauth2/tokens` is an explicit **follow-up**, **not** a merge gate and **not** in Done-when. **Must not** invent a new runbook page unless a human asks. Prefer the existing [Hydra-Wrapper (Lerna) Runbook](https://hootsuite.atlassian.net/wiki/spaces/PLATFORM/pages/13773177838).

---

## 9. Per-PR change list (delta only — §5–§8 remain the API)

Start from the four drafts. Adjust. **Must not** open a fifth HTTP approach.

### 9.1 hydra-wrapper #63 — `DE-5216-lerna-token-list`

- Implement §5 (including §5.0, probe 502, expiry mapping/truncation, access-row-only DELETE, handler-404 JSON).
- Same change-set: **both** confirmation curls + IngressRoute / `deployments/hydra-wrapper` wiring if either curl is vanilla 404.
- Automated tests §5.8, including **which** 201st row is dropped, signature no-trim, and fractional truncation.
- **Must not** rebuild `PUT /internal/tokens`. **Must not** touch `hydra_oauth2_refresh`.
- Fix ticket/PR pass-when: list `signature` **≠** last HMAC; introspect is the oracle.

### 9.2 hootsupport2 #824 — `DE-5216-list-invalidate`

- Existing `POST /api/member/{id}/access-tokens/{signature}/invalidate`. `{id}` = member = Lerna `subject`.
- Each invalidate **re-runs** §6.2 dual-read first. No persisted merge. No new `source` fields.
- Classify §4.1; invalidate §6.3 / §6.3.1. Runtime 404 via **body predicate** (§5.2.1).
- both-sourced 204 + PUT fail → leftover error, not success.
- Reject `ory_at_` on the invalidate API (400) with a #824-owned test. **Must not** edit #819 helpers.
- Agent gate: `manage_access_tokens` **and** member-page ownership.
- Tests §6.6. FIT mint = Bruno on the same `{id}`. Create regression = code review, not FIT.

### 9.3 bruno-api-collection #3 — `DE-5216-lerna-token-admin`

- Staging identifier + revoke-oracle subset (§7).
- Full introspect URL. HMAC-DELETE **then** introspect `active: false`.
- Keep other-client / empty-`signature` calls; **must not** claim them as isolation/empty-params proof (#63 CI).
- Both confirmation curls documented.

### 9.4 IDL-central #1328

- Mirror §5 and **copy §5.0**.
- Access-row only; handler-404 JSON; no HootSupport `source` fields.

### 9.5 hootsupport2 #819 (DE-5205 — do not change in this ticket)

- Create-response `ory_at_…` stays a copy field. Persist tests stay here.
- Same HootSupport release as #824 at create cutover, or **abort**.

---

## 10. Merge / deploy / prove order

**Must not** merge until asked.

### Staging

```text
1. Adjust hydra-wrapper #63 (including §5.8 tests)
2. Deploy #63 to shared staging Lerna
3. Confirmation curls:
   GET /oauth2/tokens (no subject) → 400 naming subject
   DELETE /oauth2/tokens?subject={id} (empty/missing signature) → 400 naming signature
   If either is vanilla 404: add IngressRoute / deployments/hydra-wrapper in the same change-set; redeploy; re-curl
4. Run Bruno #3 (identifier + revoke oracle, including HMAC-DELETE introspect)
5. Comment proof on DE-5216 (no secrets). Note #63 CI green for isolation/cap/params
6. Adjust and deploy #824 after #63 + both curls 400
7. #824 FIT using the Bruno mint on the same {id} (widget Create only if #819 is on staging)
```

- IDL #1328 **may** merge anytime. Correct identifier language **before** merge.
- Lerna runbook line is a **follow-up**, not a step in this list.
- Bruno **cannot** pass until both curls are 400.
- #824 **must not** deploy before #63.

### Prod

Staging proof **only** is required for ticket Done-when (no prod introspect gate).

1. Deploy #63 to prod Lerna. Confirm the same rewrite (prod equivalent of **both** curls).
2. **Must** ship #819 and #824 in the **same** HootSupport prod release when create cutover happens.
3. If #819 would deploy without #824, **abort**.
4. Rollback: revert #819 and #824 **together**. #63 **may** stay. Dual-read leftover stays until DE-5180.

Git branches stay **unstacked**.

---

## 11. Acceptance criteria mapped to ticket Done when

| Ticket Done when | Spec proof |
| --- | --- |
| Support can **list** a member’s TMA LLTs that live on **New Hydra** | After #63 + #824: tab shows **up to 200** Lerna rows for a Bruno `member_app` mint (or #819 widget Create if that PR is on staging). Overflow logged, no agent signal. Handle is stored `signature`. No leftover `key.signature` required. Bruno list 200 after that mint. Leftover dual-read extra: leftover rows when **Lerna GET 200 and** leftover GET 200; leftover-down is 200 + flag, not fail. Lerna-down fails the page. |
| Support can **invalidate** one of those tokens so introspect is `"active": false` | Each invalidate re-runs dual-read, then Lerna DELETE of **one** Lerna-sourced handle → 204; next list **gone**. both-sourced success **iff** leftover PUT succeeds. `POST https://hydra-oel-admin.staging.hootops.com/admin/oauth2/introspect` includes `"active": false`. HMAC-DELETE Bruno path also introspects `active: false`. Leftover-sourced invalidate when leftover GET 200. Rewrite-miss 404 → 502, no PUT. Aperture 401 secondary; does not fail #63. |
| Create path still does **not** call `internal/tokens` | Code review: #824 adds no create PUT. #63 **must not** add PUT. Not a FIT case. |
| UI/API no longer requires a legacy `key.signature` for new tokens | New rows use list `tokens[].signature`. #824 invalidate rejects `ory_at_`. |
| Staging proof commented on ticket. No tokens/secrets in Jira | Bruno comment §7.4 + #63 CI pointer. No bearers/secrets/PII. |

**Additional acceptance:**

- Dual-read until DE-5180 with §6.2.2 buckets, §5.2.1 404 kinds, and §6.3 / §6.3.1 envelopes.
- Hash-then-fallback: #63 CI + Bruno HMAC-DELETE + introspect.
- Isolation / cap (201st dropped) / empty-params / stacked validation / probe 502: **#63 CI**, not Bruno.
- #819 and #824 same HootSupport release at create cutover, or abort.

---

## 12. Explicitly out of scope

- DE-5205 create retarget / #819 `signature` field / post-mint re-GET / PHP-hash / persist tests / create-modal invalidate / editing #819 helpers.
- `llt_map` writes/deletes; reminting mapped tokens (DE-5200 / DE-5201).
- Aperture **swap flag** (DE-5203). (Prefix rewrite of `/oauth2/tokens` is in the #63 change-set — §5.2.)
- Using Aperture 401 as the #63 pass/fail oracle.
- Prod introspect `"active": false` as a Done-when gate.
- Lerna runbook page update as a merge gate.
- DE-5180 / DE-5181 leftover teardown.
- Rebuilding `PUT /internal/tokens` on OSS, OEL, or Lerna.
- Path B / Path C.
- TMA-only allow-list on Lerna.
- Refresh-table deletes (IDL: access-row only).
- Pagination or a client-visible cap warning; listing the 201st+ row.
- Filtering expired/inactive out of GET.
- Adding `nid` unless cleanup already binds it.
- Persisting merge/`source` across list and invalidate, or adding those fields to the HootSupport list API.
- A fifth HTTP approach or fifth product PR except the §5.2 IngressRoute-only exception.
- Secrets, bearers, or extra member PII in Jira / PR bodies.

---

## 13. Open items that require a human

**None blocked for implementers.** Rounds 1–3 are closed. Do not reopen Path B/C.

| Topic | Lock |
| --- | --- |
| Last merge (R3 I-01) | Each invalidate **re-runs** list dual-read first. That merge is last leftover 200. No extra fields. No persist. |
| 404 kinds (R3 I-02) | Handler-404 = JSON `not_found`. Rewrite miss = non-JSON 404 → 502, no PUT. Runtime uses body, not curl memory. Deploy curls: GET no-subject **and** DELETE empty signature → 400. |
| both-sourced PUT fail (R3 I-03) | Leftover error, not success. Lerna row already gone; leftover copy may remain. |
| Member ownership (R3 I-04) | `{id}` is the member. Dual-read and Lerna `subject` use `{id}`. `manage_access_tokens` **and** member-page gate unchanged. |
| Extra leftover GET (R3 I-05) | Only if PUT required and handle not on this request’s leftover 200. 200+present → PUT; 200+absent → 404 no PUT; leftover fail → leftover error, no PUT. |
| Classifier | **List source** (§4.1) from **this request’s** merge. Regex only for agent-typed absent handles. Uppercase hex typed → leftover-sourced (intentional). |
| Probe / validation / expiry / cap / `ory_at_` / runbook / FIT mint / Bruno vs CI | Unchanged from round 2; see prior §13 plus §5.0–§5.8, §7. |

**Implementation check:** a human with GHE **must** open #63, #824, #3, and #1328 and apply §9.

If a draft PR is structurally broken (Path B, or `PUT /internal/tokens` on Lerna), **stop**. **Must not** invent a fifth HTTP approach. Adjust the existing Path A drafts to this spec.
