# EverWitnessed Protocol — Examples

Concrete JSON payloads showing the shape of each EverWitnessed operation.

The same payload format (`{"e":2,"h":…}`, plus optional `a` for the hash algorithm and `r`/`c` for reveal) travels in both `custom_json` `json` fields and transfer memos. These files show the operation **body** in its readable form (legacy asset string `"0.001 HBD"` for transfers, etc.); a wax client wraps as `{custom_json_operation: …}` / `{transfer_operation: …}` and converts assets to NAI.

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

- **Reveal nonce `r`:** `1b2903784675a6670547bdacb8a1e94f9ba2b5518b3263ac9725ae48a9caf7ad` — a fresh 64-hex (256-bit) CSPRNG draw. **The nonce MUST be a CSPRNG output, never a constant.**
- **Commit reference `c`:** `174cf6f46fb5052f74b86d3d539bfe781630c801` — the transaction id of the commit the reveal points back to.

In `e:2` the commit's `h` **binds the committer**: it is the SHA-256 of the **comma-separated ASCII concatenation of the committer account, the lowercase-hex nonce `r`, and the lowercase-hex file hash `h`** (see PROTOCOL.md §Commit / Reveal):

```sh
printf '%s,%s,%s' \
  'everwitnessed' \
  '1b2903784675a6670547bdacb8a1e94f9ba2b5518b3263ac9725ae48a9caf7ad' \
  '409c44e6ae488ba6696c7d165afb01f0aa5ae72ed40dab4b95f914984fef61cc' \
  | sha256sum
# → 37f136dd2d76c336291a03b7597996efd564565796a54e393e4cdb359bd1839a
```

## See also

- [PROTOCOL.md](../PROTOCOL.md) — full specification
- [README.md](../README.md) — protocol overview
