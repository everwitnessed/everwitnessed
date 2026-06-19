# EverWitnessed Protocol Specification

**Version:** 2

EverWitnessed is an open protocol for proof-of-existence on the Hive blockchain. Users submit a digest of a file (SHA-256 by default) via a Hive operation. The blockchain records it with deterministic finality. Anyone can verify from any Hive node.

## Design Principles

1. **Minimal payload.** Five fields total (`e`, `h`, `a`, `r`, `c`); only `e` and `h` are required.
2. **One payload format everywhere.** The same JSON structure is carried in the `custom_json` body and in the transfer memo.
3. **Multiple valid submission methods.** All produce equivalent proofs.
4. **Non-custodial users control their own keys.** EverWitnessed never touches them.
5. **Custodial users receive an off-chain API receipt.** Trust-based; no on-chain user signature.
6. **Forward-compatible.** Unknown fields are ignored; the `e` field versions the protocol (see Versioning).

## Versioning

New versions are added; existing versions never change. A payload is interpreted under the rules of the version named in its `e` field.

- Verifiers MUST support every published version. An unrecognised `e` is **unverifiable**, not invalid; a verifier MUST NOT guess its meaning.
- Writers SHOULD use the latest version. Older versions stay valid; the chain enforces no version.

`e:2` is the latest. `e:1` remains valid, except its commit/reveal is superseded by `e:2` (see Commit / Reveal).

## JSON Payload

Same format in `custom_json` body and transfer memo.

### Fields

| Field                 | Key | Type    | Required         | Size        | Description |
|-----------------------|-----|---------|------------------|-------------|-------------|
| EverWitnessed version | `e` | integer | yes              | —           | Protocol version + marker. `1` or `2`. |
| Hash                  | `h` | string  | yes              | per `a`     | File digest, lowercase hex. Length set by `a` (64 for `sha256`). |
| Algorithm             | `a` | string  | optional (`e:2`) | —           | File-hash algorithm. Absent ⇒ `sha256`. A registry value (see Hash Algorithms). |
| Reveal nonce          | `r` | string  | reveal only      | 32–64 chars | CSPRNG lowercase-hex nonce (≥128 bits) used in the commitment |
| Commit reference      | `c` | string  | reveal only      | 40 chars    | Transaction id of the associated commit |

The `e` key doubles as protocol marker. Seeing `{"e":` at the start of a memo or `custom_json` body identifies an EverWitnessed payload.

### Hash Algorithms (`e:2`)

The `a` field names the file-hash algorithm. Readers use `a`, not `h`'s length, to select it.

| `a` value | `h` length (hex) |
|-----------|------------------|
| `sha256` (default; absent ⇒ this) | 64 |
| `sha512` | 128 |

`a` MUST be one of these lowercase strings; absent ⇒ `sha256`; `h`'s length MUST match the row. `a` applies to the file digest only — it never appears on a commit, whose `h` is always a 64-hex SHA-256 commitment (see Commit / Reveal). Later versions may add algorithms.

### Validation Rules

- `e` MUST be `1` or `2`. A reader that does not recognise `e` MUST treat the payload as opaque and unverifiable (see Versioning) — never invalid, never guessed.
- `a` (`e:2` only) if present MUST be a registry value; absent ⇒ `sha256`.
- `h` MUST be lowercase hex of the length set by `a` (`^[0-9a-f]{64}$` for `sha256`).
- `r` if present: lowercase hex, 32–64 chars (≥128 bits), CSPRNG output.
- `c` if present: `r` must also be present; valid transaction id (40 hex chars).
- Unknown fields: **ignored** (forward compatibility).

### Parsing

Writers MUST emit compact, lowercase-hex JSON. Readers use a standard JSON parser (RFC 8259); whitespace and field order are insignificant, and a valid payload MUST NOT be rejected over formatting. Lowercase matters for commit/reveal: `r` and `h` are hashed as strings (`abcd` ≠ `ABCD`). Duplicate keys are undefined; standard parser behaviour applies.

**Memo content.** In transfer memos, the JSON payload MUST be the entire memo content. The memo starts with `{` and ends with `}`. No leading or trailing whitespace or other characters are permitted outside the payload.

**`custom_json` `json` field.** The same rule applies. The payload MUST be the entire value of the `json` field.

**Hashing non-file inputs.** When hashing a string or document, the exact bytes matter — encoding, whitespace, and newlines change `h`.

### Payload Sizes

| Payload | Size |
|---------|------|
| Regular timestamp / commit: `{"e":2,"h":"<64>"}` | 78 bytes |
| Reveal (32-char nonce): `{"e":2,"h":"<64>","r":"<32>","c":"<40>"}` | 164 bytes |
| Reveal (64-char nonce): `{"e":2,"h":"<64>","r":"<64>","c":"<40>"}` | 196 bytes |
| SHA-512 timestamp: `{"e":2,"h":"<128>","a":"sha512"}` | 155 bytes |

All fit in `custom_json` (8,192 bytes) and transfer memo (2,047 bytes).

### Examples

**Regular timestamp (also the shape of a commit):**
```json
{"e":2,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
```

**SHA-512 timestamp (algorithm declared with `a`):**
```json
{"e":2,"h":"<128-hex SHA-512 digest>","a":"sha512"}
```

**Reveal:**
```json
{"e":2,"h":"409c44e6ae488ba6696c7d165afb01f0aa5ae72ed40dab4b95f914984fef61cc","r":"1b2903784675a6670547bdacb8a1e94f9ba2b5518b3263ac9725ae48a9caf7ad","c":"174cf6f46fb5052f74b86d3d539bfe781630c801"}
```

Commits are byte-identical in shape to a plain SHA-256 timestamp; no tag, no marker. Only reveals are identifiable, by the presence of `r` and `c`. (A live, verifiable `e:2` commit/reveal pair is in the repository's `examples/`.)

---

## Submission Methods

Each EverWitnessed operation carries exactly one payload. Each transaction MUST contain at most one EverWitnessed operation, so that the transaction id unambiguously identifies a single payload (required by the commit/reveal flow). Other non-EverWitnessed operations may share the same transaction.

The block time of the transaction in which the operation is included is the moment of proof.

Methods are grouped by custody. All produce equivalent proofs.

### Method 1 · Non-Custodial `custom_json` (recommended)

User signs with their own posting key.

```json
{
  "required_auths": [],
  "required_posting_auths": ["gandalf"],
  "id": "everwitnessed",
  "json": "{\"e\":2,\"h\":\"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70\"}"
}
```

- **Key:** Posting.
- **Cost:** Free. Resource Credits only.
- **Identity:** Blockchain-verified (account in `required_posting_auths`).
- **Per-block cap:** 5 `custom_json` operations per account per block.
- **Indexing:** Appears in the signer's account history. Discovering all Method 1 submissions across the network requires a global scan of `custom_json` with `id: "everwitnessed"`, typically via HAF.

### Method 2 · Non-Custodial Transfer

User signs a transfer with the payload in the memo. The recipient may be any account.

```
From:   gandalf
To:     <any account>
Amount: 0.001 HBD
Memo:   {"e":2,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
```

- **Key:** Active (required by the transfer operation).
- **Cost:** 0.001 HIVE or HBD. Requires a non-zero balance.
- **Identity:** Blockchain-verified (sender's signature).
- **Per-block cap:** No per-account cap; bounded only by overall block size.
- **Indexing:** A transfer appears in the account history of both the sender and the recipient. Sending to `@everwitnessed` centralises timestamps in one well-known history (convenient for single-account scanning). Sending to self keeps the record in the user's own history only. Sending to a third party places it in that account's history as well as the sender's.

### Method 3 · Custodial `custom_json`

`@everwitnessed` broadcasts on behalf of a non-Hive user.

```json
{
  "required_auths": [],
  "required_posting_auths": ["everwitnessed"],
  "id": "everwitnessed",
  "json": "{\"e\":2,\"h\":\"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70\"}"
}
```

- **Key:** Posting (of `@everwitnessed`, managed server-side).
- **Cost:** Free to the user.
- **Identity:** Trust-based. The user relies on EverWitnessed's API receipt. No on-chain user identity.
- **Per-block cap:** 5 `custom_json` operations from `@everwitnessed` per block.
- **Indexing:** `@everwitnessed`'s account history.

### Method 4 · Custodial Transfer

`@everwitnessed` broadcasts a transfer on behalf of a non-Hive user. By convention the transfer is to itself, so that the HBD does not leave the service account.

```
From:   everwitnessed
To:     everwitnessed
Amount: 0.001 HBD
Memo:   {"e":2,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
```

- **Key:** Active (of `@everwitnessed`, managed server-side).
- **Cost:** Free to the user. 0.001 HBD is paid by EverWitnessed.
- **Identity:** Trust-based. Same as Method 3.
- **Per-block cap:** No per-account cap.
- **Indexing:** `@everwitnessed`'s account history.

### Method Comparison

| | Method 1 | Method 2 | Method 3 | Method 4 |
|---|---|---|---|---|
| Custody | non-custodial | non-custodial | custodial | custodial |
| Operation | `custom_json` | transfer | `custom_json` | transfer |
| Signer | user | user | `@everwitnessed` | `@everwitnessed` |
| Key | posting | active | posting | active |
| Cost to user | free | 0.001 HIVE/HBD | free | free |
| Balance required | no | yes | n/a | n/a |
| Identity | blockchain auth | blockchain auth | trust-based | trust-based |
| Per-account block cap | 5 | none | 5 | none |
| Hive account required | yes | yes | no | no |

---

## Custodial Model

Custodial timestamps (Methods 3 and 4) are trust-based. The on-chain record says `@everwitnessed` timestamped hash H at time T. There is no on-chain proof of which end-user requested it.

Flow:

1. User sends hash to EverWitnessed's API.
2. EverWitnessed validates, constructs the Hive operation, signs with the appropriate key, and broadcasts.
3. User receives an API receipt: tx id, block number, block time, hash.

The receipt is verifiable by anyone. A lookup on any Hive node confirms the hash is on-chain at the recorded block time. It does not cryptographically prove which end-user requested the timestamp; that is an off-chain claim by EverWitnessed.

**Upgrade path.** A custodial user who wants self-sovereign, cryptographically bound proof creates a Hive account and switches to Method 1. Future timestamps are signed with their own posting key.

---

## Commit / Reveal (Anti-Frontrunning)

An observer of the P2P network — including a block producer — can see a pending transaction, extract the hash, and race to submit it under their own account before yours lands. Commit/reveal is a two-phase flow that prevents this: it hides the hash until reveal, and (in `e:2`) binds the commitment to the committer so no one else can claim it. Use `e:2` for commit/reveal; see *Improvement over `e:1`* below.

### Phase 1. Commit (`e:2`)

Choose a nonce `r` — a CSPRNG output of ≥128 bits, 32–64 lowercase hex chars (e.g. `openssl rand -hex 32`). Compute the commitment:

```
commitment = SHA-256( ASCII( account "," r "," h ) )
```

- `account` — the account that signs the commit: `required_posting_auths[0]` for a `custom_json` commit, or `from` for a transfer commit.
- `r` — the lowercase-hex nonce.
- `h` — the lowercase-hex file digest being committed.
- The three values are joined by a literal comma `,` (which cannot occur in a Hive account name or in hex), as ASCII bytes, with no whitespace. The commitment's own hash is always SHA-256, regardless of the file's `a`.

Reproduce in shell:

```sh
printf '%s,%s,%s' "$account" "$r" "$h" | sha256sum
```

Then broadcast the commitment as a normal timestamp:

```json
{"e":2,"h":"<commitment>"}
```

The commitment is opaque without the nonce, and byte-identical in shape to a plain SHA-256 timestamp — no observer can tell which `h` values are commits.

### Phase 2. Reveal (`e:2`)

Once the commit is irreversible (OBI typically advances finality within the same block), broadcast:

```json
{"e":2,"h":"<file_hash>","r":"<nonce>","c":"<commit_tx_id>"}
```

Include `a` if the file digest is not SHA-256, exactly as in a plain timestamp.

### Verification (`e:2`)

1. Read `h`, `r`, `c` (and `a`, default `sha256`) from the reveal.
2. Look up the commit transaction `c`; read its committed value, its block time, and its **signer account** (`required_posting_auths[0]` for `custom_json`, or `from` for a transfer).
3. Recompute `SHA-256( ASCII( <commit_signer> "," r "," h ) )` and check it equals the commit's `h`.
4. If it matches, the file digest `h` (under algorithm `a`) was committed by the commit's signer at the commit's block time.

The block time is the proof. Binding the account makes the commitment non-transferable: recomputing with a different signer yields a different value, so a copied commitment cannot be opened.

### Nonce quality

`r` MUST be a CSPRNG output of at least 128 bits. A guessable nonce defeats hiding when `h` is guessable: an observer can brute-force `SHA-256(account,r,h)` over candidate nonces.

### Improvement over `e:1`

`e:2` binds the committer's account into the commitment (`SHA-256("account,nonce,h")`, vs `e:1`'s `SHA-256(nonce‖h)`). Because the `e:1` commitment is independent of the committer, a copied commitment could be opened by another account after the reveal; `e:2` prevents this. Use `e:2` for new commit/reveal; `e:1` plain timestamps are unaffected.

### Applies to Both Tiers

Frontrunning is an on-chain concern regardless of who broadcasts. Commit/reveal works the same way for custodial and non-custodial flows; for custodial, EverWitnessed performs both phases (the bound account is `@everwitnessed`). A custodial receipt may include the nonce so the user can recompute the binding themselves.

---

## Security Considerations

### Duplicate Timestamps

The same hash can be timestamped more than once. The earliest occurrence (lowest block number) wins for priority; later submissions are harmless duplicates.

### Replay

A payload copied from one operation and resubmitted in another just creates a duplicate. Earlier wins. No harm.

### Frontrunning

A hash observed pending can be resubmitted under another account. Commit/reveal mitigates this by hiding the hash until reveal; `e:2` additionally binds the commitment to the committer's account, so a copied commitment cannot be opened by anyone else. Use `e:2` for commit/reveal.

### Display Safety

All on-chain content is user-controlled. An EverWitnessed client MUST:

- Validate `e` is a known version (`1` or `2`); treat an unknown version as unverifiable, not invalid.
- Validate `h` strictly against the length set by `a` (`^[0-9a-f]{64}$` for `sha256`); reject or hide invalid values.
- Validate `a`, `r`, and `c` if present (registry value; hex format; expected lengths).
- Parse malformed JSON gracefully (skip, never crash).
- Ignore unknown fields.

---

## Indexing

### By Transaction Id

The user retains the tx id from when the operation was broadcast. Any Hive node can look it up; no EverWitnessed infrastructure is required.

### By Account History

Every Hive operation appears in the account history of its affected accounts. For `custom_json`, that is the signing account. For a transfer, both the sender and the recipient see the operation in their history.

- Method 1 (non-custodial `custom_json`): the user's own history.
- Method 2 (non-custodial transfer): both sender (user) and recipient (any chosen account). Sending to `@everwitnessed` centralises all such timestamps in one account.
- Methods 3 and 4 (custodial): `@everwitnessed`'s history.

### By Global Scan

Finding all EverWitnessed timestamps across Method 1 submissions requires a global scan of `custom_json` operations with `id: "everwitnessed"`, typically via HAF.

---

## Accounts

### @everwitnessed

Serves multiple roles:

- Receive target for non-custodial transfers (Method 2).
- Sender and receiver for custodial transfers (Method 4).
- Broadcaster for custodial `custom_json` (Method 3).

The posting and active keys are operational (managed server-side) when custodial methods are enabled.

---

## Limits

- `custom_json` body: 8,192 bytes.
- `custom_json` per account per block: 5.
- Transfer memo: 2,047 bytes (strict `<` check).
- Transfer per account per block: no per-account cap; limited only by overall block size.

## Resource Credits

Hive transactions consume Resource Credits (RC): a continuously regenerating allowance on each account, sized by the account's Hive Power (staked HIVE). A `custom_json` operation's RC cost is comparable to posting a comment.
