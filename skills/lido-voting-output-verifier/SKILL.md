---
name: lido-voting-output-verifier
description: Perform a full independent verification and repair pass on voting_items.md generated from Lido DAO voting calldata, including strict byte-for-byte checks of every address, hash, and calldata-derived value. Use when checking or fixing a human-readable Lido governance action list against source calldata, parsed Aragon execution traces, nested EVM scripts, registry names, numbering, formatting, calls with calldata parsed as empty, or role and pauser semantics.
---

# Lido Voting Output Verifier

## Objective

Audit `voting_items.md` against the original Lido voting calldata and fix the file in place. Treat the generated action list as untrusted until every effective governance action has been reconciled with source calldata, nested scripts, and the Lido Docs registry, and every address, hash, and calldata-derived value has passed the strict byte-for-byte checks below.

Use this registry for canonical contract-name checks:

`https://raw.githubusercontent.com/lidofinance/docs/refs/heads/feat/srv3-proposed-contracts/docs/deployed-contracts/index.md`

## Verification Workflow

1. Read the source voting calldata, parsed proposal text, and any linked parsed EVM scripts.
2. Independently reconstruct the effective call list. Expand `AragonAgent.forward(bytes _evmScript)` and similar wrappers; ignore only pure wrappers; keep wrapper calls only when the wrapper itself is the meaningful action.
3. Count effective non-wrapper calls, including empty-calldata calls.
4. Read `voting_items.md` and map each action line to exactly one effective call.
5. Run the strict byte-for-byte checks below for every mapped action. Use a mechanical comparison for every raw hex value; visual inspection is not verification. Treat one mismatched, omitted, truncated, rounded, or unverified byte sequence as a verification failure.
6. Fetch or read the Lido Docs registry and verify every contract name/address pair used in `voting_items.md`.
7. Fix missing, extra, duplicated, mislabeled, misordered, misformatted, or byte-inaccurate actions directly in `voting_items.md`.
8. Re-run the checklist below after edits until no issues remain.

## Strict Byte-For-Byte Checks

Verify every address, hash, and value independently against the original calldata bytes. Do not accept a parsed label, proposal description, decoded trace summary, registry entry, or the existing `voting_items.md` as proof of byte equality when raw source bytes are available.

For each effective call, retain and check the exact target address bytes, call value, function selector, and ABI-encoded argument bytes. For nested EVM scripts, perform the same checks against the exact inner call bytes after locating each call boundary.

For every address, compare the complete 20-byte value. For every hash, role identifier, selector, `bytesN`, and dynamic `bytes` value, compare the complete byte sequence and its exact length. Never use prefix, suffix, case-insensitive text, visually similar text, or shortened-value matching as a substitute for byte equality. Preserve the source spelling and checksum casing in the action list when available after byte equality is established.

### Mandatory Mechanical Hex Comparison

For every raw hex literal that is retained in an action—including addresses, hashes, role identifiers, permissions, setup calldata, nested calldata, selectors, and arbitrary `bytes`—extract the complete source literal and the complete output literal and compare them mechanically. Never approve a long hex literal by reading it on screen.

Apply all of these checks to each source/output pair:

1. Strip only the syntactic `0x` prefix. Do not trim, pad, regroup, decode, or otherwise normalize the value before the raw comparison.
2. Assert that both values contain only hex digits and have an even number of hex digits. An odd-length value cannot represent a complete byte sequence and is an immediate failure.
3. Compare the exact hex-digit counts before comparing content. Report any length delta explicitly; a length mismatch fails even when decoding appears to produce the intended values.
4. Compare the complete values byte for byte with a deterministic tool or script. Hex letter case may be normalized only for this equality test; restore and verify the source spelling or checksum casing in the final action.
5. When values differ, locate the first differing byte offset and inspect fixed-width, start-aligned chunks around it. Do not use independently wrapped visual text because a missing nibble or byte shifts all later apparent groups.
6. After a repair, re-extract the literal from `voting_items.md` and repeat the same length and full-content assertions. Do not treat editing the line as proof that the repair succeeded.

For a raw `bytes` value that itself contains standard ABI function calldata, perform an additional structural sanity check: it must contain an 8-hex-digit selector, and the ABI payload after that selector must have a length consistent with the function ABI. For ordinary ABI encoding, each static word occupies exactly 64 hex digits and offsets, lengths, padding, arrays, and tuples must resolve within the literal. This structural check supplements the direct source/output comparison; it never replaces it.

If only a parsed trace is available, its displayed raw hex literal is the comparison source. Compare that literal directly with the action-list literal even when the parsed arguments or a fresh ABI decode look equivalent. In particular, decoded numeric equality does not excuse missing leading zero bytes, truncation, concatenation shifts, or a different raw encoding.

For every calldata-derived value—including integers, booleans, strings, token amounts, durations, IDs, arrays, tuples, enum values, and call ETH values—decode it with the exact ABI type and verify it against its complete source byte range. If the action renders a decoded or human-readable form, ABI-encode or otherwise losslessly reverse that rendered value and require the result to equal the original bytes exactly. Unit conversion is allowed only when it is exact and reversible; never round, approximate, silently rescale, normalize, or drop precision.

For strings, compare the exact encoded byte content and length; do not silently alter Unicode normalization, whitespace, punctuation, or letter case. For arrays and tuples, verify every element in order as well as lengths and nesting. For signed and fixed-width integers, verify width, sign, and two's-complement encoding where applicable.

Trace each backticked address, hash, and value in every action line to one exact source byte range or to an exact lossless derivation from that range. A single unexplained value or mismatch fails the entire verification pass. Repair the action and repeat the byte comparison. If the available source artifacts do not expose the bytes needed for an exact check, do not claim that verification passed; keep the exact raw value available in the source artifact and report the verification blocker to the user.

## Coverage Checks

Every effective non-wrapper call from the calldata must appear exactly once.

Every call whose calldata is parsed as `[empty]` must appear exactly once, must not be marked `[UNKNOWN]`, and must keep the called target address and exact call value. Use known semantics from the trace or proposal context when available; otherwise use neutral empty-calldata-call wording, or ETH-transfer wording when the call value is nonzero. Include a canonical target contract name only if the address is verified in the registry; otherwise retain the raw address and any supplied parsed label without inventing a label.

No pure forwarding or execution wrapper should appear as an action unless it is itself the meaningful governance action.

Expanded nested actions must retain their relationship to the top-level voting item. Use decimal labels only for expanded subactions of that top-level item.

The Dual Governance submission must be standalone top-level item `1.` before section `I.` and must not be placed under any Roman-numeral section.

Top-level integer numbering must be continuous after the Dual Governance submission. Other top-level voting items must be `2.`, `3.`, etc., not additional `1.x.` items.

## Semantic Checks

For every `grantRole(bytes32 role, address account)` or `revokeRole(bytes32 role, address account)` action, ensure the target contract in the action is the called contract receiving the permission change, not the forwarding agent. Ensure role names and role hashes are preserved. Normalize `[PAUSE ROLE]` to `PAUSE_ROLE`.

For `registerPauser(address _pausable, address _newPauser)`, ensure the action uses the called contract as the registry or authority and the first argument as the pausable contract.

For `setLimitParameters(uint256 _limit, uint256 _periodDurationMonths)`, ensure 18-decimal stETH values are divided by `1e18` when the registry or factory context indicates stETH, and preserve the period duration in months.

For `setNodeOperatorName(uint256 _nodeOperatorId, string _name)`, ensure the node operator ID and new name are present. Include the known old name when discoverable from the provided calldata or context.

For `deactivateNodeOperator(uint256 _nodeOperatorId)`, ensure the node operator ID is present and include the known name when discoverable from the provided calldata or context.

If a role hash is unlabeled but can be confidently identified from nearby calls or known contract docs, include the role name. Otherwise keep the raw hash without inventing a name.

For proxy upgrade functions such as `upgradeTo(address)`, `upgradeToAndCall(address,bytes)`, and equivalent proxy-admin or upgrade-template calls, require this wording:

```text
Upgrade <contract name> <proxy address> to implementation <new implementation address>
```

Ensure the upgraded contract name appears only before the proxy address. Do not repeat it after `implementation` or immediately before the new implementation address. Preserve setup calldata, force-call flags, and other material arguments after the new implementation address when present.

## Registry And Label Checks

Verify every parsed contract address against the Lido Docs protocol contracts registry when possible.

Use the registry's canonical contract name only when the address matches. Preserve checksum address casing from the parsed input when available; otherwise use the registry checksum casing if available.

If an address is absent from the registry, mark it as `[UNKNOWN]` and keep the raw address. A parsed label or proposal-context label may be included only after `[UNKNOWN]`. Calls whose calldata is parsed as `[empty]` are an exception: never add `[UNKNOWN]` to those calls, even when the target is absent from the registry.

Do not invent labels. If no label is available, use the raw address or hash.

Prefer domain names used in Lido docs or parsed labels, but keep the exact target address so the action remains verifiable.

## Formatting Checks

`voting_items.md` must contain only the final action list. Remove raw ABI blocks, calldata arrays, examples, notes, scratch work, and verification commentary.

Every section heading and action line must start at column 1 with no leading spaces.

Use Roman-numeral section headings exactly like `I. <short objective>` and `II. <short objective>`. Do not put a period at the end of section headings.

Each action line must be one sentence with this shape:

```text
<number>. <verb> <object/value> <preposition phrase with target contract name and address>
```

Do not put a period at the end of action lines. Keep only the period after the action number.

Wrap every address, hash, numeric ID, limit, duration, token amount, role value, account value, and string value from calldata in backticks. Put hashes and addresses verbatim inside backticks. Do not shorten them.

Keep headings and action lines concise. Use blank lines between logical groups inside a large section only when it improves scanability.

## Repair Rules

When an issue is found, edit `voting_items.md` immediately rather than only reporting it.

If an action is missing, add it under the correct section or create a new concise Roman-numeral section when it is unrelated to existing sections.

If an action is extra or represents a pure wrapper, remove it.

If a contract name does not match the registry address, replace the name with the canonical registry name when matched, or mark the address `[UNKNOWN]` when unmatched. For a call whose calldata is parsed as `[empty]`, remove the unverified canonical name if necessary but do not add `[UNKNOWN]`; retain the raw address and any supplied parsed label.

If a call whose calldata is parsed as `[empty]` is marked `[UNKNOWN]`, remove that marker while preserving the action, target address, exact call value, and any supported label or semantics.

If a proxy upgrade repeats the upgraded contract name before the new implementation address, remove only that repeated name while preserving both addresses and every other material argument.

If numbering is invalid, renumber the smallest necessary range while preserving the relationship between top-level items and decimal subactions.

After every repair pass, recount effective calls, remap all action lines, and re-check formatting until the file is clean.
