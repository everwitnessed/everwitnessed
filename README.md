# EverWitnessed Protocol

![Powered by Hive](./assets/badge-hive.svg)

Open specification for proof-of-existence on the [Hive blockchain](https://hive.io).

Write a SHA-256 hash to Hive, get a permanent, publicly verifiable timestamp. No servers, no calendars, no aggregation. The proof becomes a block in a public ledger replicated across every peer on the network.

**Reference implementation:** [everwitnessed.net](https://everwitnessed.net)

---

## What it is

A minimal JSON payload carried by a Hive `custom_json` operation or transfer memo. Two fields are required; two more are used only for the commit/reveal flow.

```json
{"e":1,"h":"6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70"}
```

| Field | Required | Purpose |
|-------|----------|---------|
| `e`   | yes | Protocol version and marker |
| `h`   | yes | SHA-256 digest, lowercase hex (64 chars) |
| `r`   | reveal only | Nonce used in the commitment |
| `c`   | reveal only | Transaction id of the commit |

That's the entire protocol. Metadata (filename, description, tag) is deliberately off-chain; it belongs to the service layer.

---

## How it's broadcast

Any of these is a valid EverWitnessed timestamp:

1. **Non-custodial `custom_json`** (recommended). User signs with their own posting key.
2. **Non-custodial transfer.** User signs a transfer with the payload in the memo. The recipient may be any account.
3. **Custodial `custom_json`.** `@everwitnessed` broadcasts on behalf of non-Hive users.
4. **Custodial transfer.** `@everwitnessed` broadcasts a transfer on behalf of non-Hive users.

All produce equivalent proofs. See [PROTOCOL.md](./PROTOCOL.md) for the full specification and [example payloads](./examples).

---

## How to verify

Given a transaction id, query any Hive node.

**TypeScript via [`@hiveio/wax`](https://www.npmjs.com/package/@hiveio/wax):**

```ts
// Any Hive node, no account, via @hiveio/wax
import { createHiveChain } from "@hiveio/wax";

const chain = await createHiveChain();
const tx = await chain.api.account_history_api.get_transaction({ id: "<tx_id>" });

// Parse the payload from the custom_json operation.
// If h matches yours, the block time is the proof.
```

**Shell via curl + jq, no dependencies:**

```bash
# Any Hive node, no account
curl -s https://api.hive.blog \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,
       "method":"account_history_api.get_transaction",
       "params":{"id":"<tx_id>"}}' | \
jq '.result | {block_num,
        id: .operations[0].value.id,
        h: (.operations[0].value.json | fromjson | .h)}'
```

No account, no API key, no pending state, no complex workflow. One call, one block, final.

---

## Why Hive

- **Three-second blocks** with deterministic finality. Typically sub-second under normal network conditions, not the probabilistic "N confirmations" model of Bitcoin.
- **No protocol fees.** Operations only consume Resource Credits.
- **Replicated** across every peer on the network.
- **Battle-tested** Layer-1, continuously producing blocks.

---

## License

MIT. See [LICENSE](./LICENSE). Build anything you want on top.
