---
name: lido-raw-evm-script-verifier
description: Verify a raw Aragon/Lido EVM script hex blob against parsed Lido voting calldata, parsed Aragon vote execution items, or decoded call lists. Use when the user supplies raw EVM script bytes and asks to compare, validate, audit, or reconcile them with parsed voting calldata; detect byte-offset, target, calldata-length, selector, argument, order, missing-call, extra-call, and nested-script parsing mismatches.
---

# Lido Raw EVM Script Verifier

## Objective

Verify that a parsed representation of Lido voting calldata exactly matches the supplied raw EVM script. Treat the raw EVM script as the source of truth and the parsed calldata as untrusted until every byte range is accounted for.

This skill is for raw-script verification only. Do not generate the final `voting_items.md` action list here; use `lido-voting-action-list` and `lido-voting-output-verifier` for action-list work.

## Inputs

Use the raw EVM script hex supplied by the user and the parsed voting calldata, decoded execution trace, or parsed call list supplied in the same request or local files. If either side is missing, state exactly which input is needed.

Accept raw script values with or without a `0x` prefix. Preserve checksum address casing from parsed input when comparing display values, but compare hex bytes case-insensitively.

## Raw CallScript Parsing

For Aragon CallScript executor ID `0x00000001`, parse the bytes as:

```text
bytes[0:4]      executor spec ID, must be 0x00000001
then repeated:
  bytes[n:n+20]     target address
  bytes[n+20:n+24]  calldata length, uint32 big-endian byte count
  bytes[n+24:n+24+length] calldata
```

Reject or flag any raw script that:

- has an odd hex length
- has a non-`0x00000001` spec ID
- ends in the middle of an address, length, or calldata segment
- has trailing bytes after the last decoded call
- has a length prefix that does not equal the actual calldata byte count

Record every decoded call with:

- call index
- start and end byte offsets within the raw script
- target address
- calldata byte length
- selector, the first 4 bytes of calldata, or `<empty>` for zero-length calldata
- full calldata hex

## Parsed-Calldata Normalization

Convert parsed voting calldata into the same ordered tuple shape:

```text
(target address, calldata length in bytes, selector or <empty>, full calldata)
```

For parsed calls that expose ABI fields but not full calldata, reconstruct calldata only when all argument values and types are present. If full calldata cannot be reconstructed confidently, compare the fields that are available and mark calldata equality as unresolved rather than passing it.

Normalize wrappers carefully:

- Keep top-level wrapper calls when they are physically present in the raw script being verified.
- If the parsed calldata expands a wrapper such as `AragonAgent.forward(bytes _evmScript)`, verify both the wrapper calldata bytes and the nested `_evmScript` bytes when the raw source includes them.
- Parse nested `_evmScript` values with the same CallScript rules when they begin with `0x00000001`.
- Do not compare expanded nested calls against the parent raw script segment unless the raw segment being verified is that nested script.

## Verification Workflow

1. Decode the raw EVM script into ordered call records and byte offsets.
2. Normalize the parsed calldata or execution trace into ordered call records.
3. Compare call counts at the same nesting level.
4. Compare each call in order by target address, calldata length, selector, and full calldata bytes.
5. For ABI-decoded parsed calls, verify that the decoded selector and arguments are consistent with the raw calldata.
6. Recursively verify nested EVM scripts found in parsed wrapper arguments.
7. Produce a concise mismatch report, or repair the parsed artifact in place if the user explicitly asked for fixes and the correct value is byte-obvious from the raw script.

## Mismatch Reporting

Report mismatches with enough detail to repair the parsed calldata:

```text
Call <index> mismatch at raw bytes <start>-<end>:
- target: raw <address>, parsed <address>
- length: raw <bytes>, parsed <bytes>
- selector: raw <selector>, parsed <selector>
- calldata: raw <0x...>, parsed <0x...>
```

For missing or extra calls, include the nearest raw byte offset and adjacent call index. For nested mismatches, include a path such as `call 3 -> forward(bytes) nested call 2`.

If verification passes, state that the parsed calldata matches the raw EVM script and include the verified call count. Do not include long raw calldata in the success response unless requested.

## Repair Rules

Only edit parsed calldata files when the user asks to fix or repair them. When repairing:

- Change only values contradicted by the raw bytes.
- Preserve existing formatting where practical.
- Do not infer contract names, labels, or semantic action text from raw bytes alone.
- Re-run the comparison after edits and report the final status.

If the parsed artifact is `voting_items.md`, use `lido-voting-output-verifier` instead; this skill verifies raw scripts against parsed calldata, not human-readable action semantics.
