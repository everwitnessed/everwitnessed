# EverWitnessed Protocol Specification

**Version:** 1
**Status:** Active

EverWitnessed is an open protocol for proof-of-existence on the Hive blockchain. Users submit the SHA-256 digest of a file via a Hive operation. The blockchain records it with deterministic finality. Anyone can verify from any Hive node.

## Design Principles

1. **Minimal payload.** Four fields total (`e`, `h`, `r`, `c`); only `e` and `h` are required.
2. **One payload format everywhere.** The same JSON structure is carried in the `custom_json` body and in the transfer memo.
3. **Multiple valid submission methods.** All produce equivalent proofs.
4. **Non-custodial users control their own keys.** EverWitnessed never touches them.
5. **Custodial users receive an off-chain API receipt.** Trust-based; no on-chain user signature.
6. **Forward-compatible.** Unknown fields are ignored; the `e` field enables future versions.

## JSON Payload

Same format in `custom_json` body and transfer memo.

### Fields

| Field                 | Key | Type    | Required     | Size        | Description |
|-----------------------|-----|---------|--------------|-------------|-------------|
| EverWitnessed version | `e` | integer | yes          | —           | Protocol version. `1` for v1. Also serves as protocol marker. |
| Hash                  | `h` | string  | yes          | 64 chars    | SHA-256 digest, lowercase hex |
| Reveal nonce          | `r` | string  | reveal only  | 32–64 chars | Random hex string used in the commitment |
| Commit reference      | `c` | string  | reveal only  | 40 chars    | Transaction id of the associated commit |

The `e` key doubles as protocol marker. Seeing `{"e":` at the start of a memo or `custom_json` body identifies an EverWitnessed payload.

### Validation Rules

- `e` must be a positive integer (`1` for v1).
- `h` must match `^[0-9a-f]{64}$`.
- `r` if present: hex string, 32–64 chars (128–256 bits of entropy).
- `c` if present: `r` must also be present; valid transaction id (40 hex chars).
- Unknown fields: **ignored** (forward compatibility).

### Parsing

Readers MUST use a standard JSON parser (RFC 8259). Whitespace and field order carry no semantic meaning. Any well-formed JSON object that passes field-level validation is a valid EverWitnessed payload. Writers SHOULD produce compact form (no whitespace) for byte efficiency, but this is not required for correctness. Duplicate keys are not defined by this spec; standard parser behaviour applies.

**Memo content.** In transfer memos, the JSON payload MUST be the entire memo content. The memo starts with `{` and ends with `}`. No leading or trailing whitespace or other characters are permitted outside the payload.

**`custom_json` `json` field.** The same rule applies. The payload MUST be the entire value of the `json` field.

### Payload Sizes

| Payload | Size (`e:1`) |
|---------|-------------|
| Regular timestamp / commit: `{"e":1,"h":"<64>"}` | 78 bytes |
| Reveal (32-char nonce): `{"e":1,"h":"<64>","r":"<32>","c":"<40>"}` | 162 bytes |
| Reveal (64-char nonce): `{"e":1,"h":"<64>","r":"<64>","c":"<40>"}` | 194 bytes |

All fit in `custom_json` (8,192 bytes) and transfer memo (2,047 bytes).

### Examples

**Regular timestamp (also the shape of a commit):**
```json
{"e":1,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
```

**Reveal:**
```json
{"e":1,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70","r":"beeab0de00000000000000000000000000000000000000000000000000000000","c":"027e1a81b357817388dea1e84e0d388809adbefb"}
```

Commits are byte-identical in shape to regular timestamps; no tag, no marker. Only reveals are identifiable, by the presence of `r` and `c`.

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
  "json": "{\"e\":1,\"h\":\"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70\"}"
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
Memo:   {"e":1,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
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
  "json": "{\"e\":1,\"h\":\"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70\"}"
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
Memo:   {"e":1,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
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

An observer of the P2P network, including a malicious block producer, can see a pending transaction, extract the hash, and race to submit it under their own account before yours lands. Commit/reveal is a two-phase protocol that prevents this.

### Phase 1. Commit

User computes `commitment = sha256(nonce + h)` and broadcasts:

```json
{"e":1,"h":"<commitment>"}
```

The commitment is opaque without the nonce. An observer cannot recover the file hash.

A commit is byte-identical in shape to a regular timestamp; no observer can tell which `h` values are commits.

### Phase 2. Reveal

Once the commit is irreversible (OBI typically advances finality within the same block), the user broadcasts:

```json
{"e":1,"h":"<file_hash>","r":"<nonce>","c":"<commit_tx_id>"}
```

### Verification

1. Look up the reveal transaction; read `h`, `r`, `c`.
2. Look up the commit transaction referenced by `c`; read the committed hash and the block time.
3. Verify that `sha256(r + h)` equals the committed hash.
4. If it matches, the file hash `h` was committed at the commit's block time.

No timestamp field appears in the payload; the block time is the proof.

### Applies to Both Tiers

Frontrunning is an on-chain concern regardless of who broadcasts. Commit/reveal works the same way for custodial and non-custodial flows. For custodial, EverWitnessed performs both phases on the user's behalf.

---

## Security Considerations

### Duplicate Timestamps

The same hash can be timestamped more than once. The earliest occurrence (lowest block number) wins for priority; later submissions are harmless duplicates.

### Replay

A payload copied from one operation and resubmitted in another just creates a duplicate. Earlier wins. No harm.

### Frontrunning

A hash observed in a pending transaction can be submitted first by an attacker under their own account. Commit/reveal prevents this; see above.

### Display Safety

All on-chain content is user-controlled. An EverWitnessed client MUST:

- Validate `h` strictly (`^[0-9a-f]{64}$`); reject or hide invalid values.
- Validate `e` is a positive integer.
- Validate `r` and `c` if present (hex format, expected lengths).
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
