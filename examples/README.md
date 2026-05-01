# EverWitnessed Protocol — Examples

Concrete JSON payloads showing the shape of each EverWitnessed operation.

The same payload format (`{"e":1,"h":…}`, plus optional `r` and `c` for reveal) travels in both `custom_json` `json` fields and transfer memos. These files show the operation **body** in its readable form (legacy asset string `"0.001 HBD"` for transfers, etc.); a wax client wraps as `{custom_json_operation: …}` / `{transfer_operation: …}` and converts assets to NAI.

| File | Operation |
|------|-----------|
| [`01-custom-json.json`](./01-custom-json.json) | `custom_json` carrying the EverWitnessed payload (recommended path) |
| [`02-transfer.json`](./02-transfer.json) | Transfer carrying the same payload in the memo (alternative path) |
| [`03-commit.json`](./03-commit.json) | Commit phase: a `custom_json` whose `h` is the commitment value |
| [`04-reveal.json`](./04-reveal.json) | Reveal phase: a `custom_json` carrying `{e, h, r, c}` |

For `02-transfer.json` the recipient is `everwitnessed`. Per the spec, the recipient may be any account; sending to `@everwitnessed` (or to oneself) are both common patterns.

## Values used

All examples share these:

- **File content:** the 12-byte UTF-8 sequence `f09fa799 f09fa9b6 f09fa684`.
- **File hash `h`:** `409c44e6ae488ba6696c7d165afb01f0aa5ae72ed40dab4b95f914984fef61cc`. Reproduce:
  ```sh
  printf 'f09fa799f09fa9b6f09fa684' | xxd -r -p | sha256sum
  ```
- **Account:** `everwitnessed`.

The commit/reveal pair adds:

- **Reveal nonce `r`:** `b0f200ce4f1c88c57e42e803a9ad360ab4e2a4bc95a7539f25b701d92bdb62f7` — a fresh 64-hex (256-bit) draw. **In real use the nonce MUST be a CSPRNG output, never a constant.**
- **Commit reference `c`:** `c5000470eaa07f1d7925f69d45dcd7f6d5d48bd1` — the transaction id of the commit that the reveal points back to.

The commit's `h` is `sha256(r || h)` over the byte-concatenation of the nonce and the file hash:

```sh
printf '%s%s' \
  'b0f200ce4f1c88c57e42e803a9ad360ab4e2a4bc95a7539f25b701d92bdb62f7' \
  '409c44e6ae488ba6696c7d165afb01f0aa5ae72ed40dab4b95f914984fef61cc' \
  | sha256sum
# → 0432b32c61cc4ffbba01caa35e00e56f1a3630fa3ba3491894f7e5347b20b0fe
```

## See also

- [PROTOCOL.md](../PROTOCOL.md) — full specification
- [README.md](../README.md) — protocol overview
