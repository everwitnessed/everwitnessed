# EverWitnessed Protocol — Examples

Concrete JSON payloads for each supported submission method.

All examples share the same EverWitnessed payload format — the inner JSON — regardless of whether it travels inside a `custom_json` `json` field or a `transfer` `memo`.

| File | Method | Signer | Key | Cost | Identity |
|------|--------|--------|-----|------|----------|
| [`01-custom-json-timestamp.json`](./01-custom-json-timestamp.json) | `custom_json` — non-custodial | user | posting | free (RC) | blockchain auth |
| [`02-custom-json-custodial.json`](./02-custom-json-custodial.json) | `custom_json` — custodial | `@everwitnessed` | posting | free to user | trust-based |
| [`03-transfer-to-everwitnessed.json`](./03-transfer-to-everwitnessed.json) | Transfer to `@everwitnessed` | user | active | 0.001 HBD | blockchain auth |
| [`04-transfer-to-self.json`](./04-transfer-to-self.json) | Transfer to self | user | active | 0.001 HBD | blockchain auth |
| [`05-commit.json`](./05-commit.json) | Commit (phase 1 of commit/reveal) | user | posting | free (RC) | blockchain auth |
| [`06-reveal.json`](./06-reveal.json) | Reveal (phase 2 of commit/reveal) | user | posting | free (RC) | blockchain auth |

## Values used in the examples

- **File hash `h`** = `6c1adcafe1cf72602fc9fff64d35304a649c27bff5ef16b2034fbe673c8b4c70` — the SHA-256 of the ASCII string `everwitnessed.net`. Reproduce with `printf "everwitnessed.net" | sha256sum`.
- **Reveal nonce `r`** = `beeab0de00000000000000000000000000000000000000000000000000000000` — the full 64-hex-char (256-bit) Hive chain id, used here as a recognizable constant at the maximum nonce length. Any 32–64 hex-character random value is valid in real use.
- **Commit reference `c`** = `027e1a81b357817388dea1e84e0d388809adbefb` — the id of the first block produced after Hive's fork from the legacy chain; used here only because it's a recognizable 40-char hex value.
- **Commit payload `h`** (in `05-commit.json`) = `55203a2bd7eff1b9a95711bc2f0c3f4d38868ce87a55d8ea24d8ff96ddab6e15` — computed as `sha256(nonce + h)` over the string concatenation of the reveal nonce and the file hash shown above.
- **Account** = `gandalf`.

## See also

- [PROTOCOL.md](../PROTOCOL.md) — full specification
- [README.md](../README.md) — protocol overview
