---
name: lido-voting-output-verifier
description: Perform a full independent verification and repair pass on voting_items.md generated from Lido DAO voting calldata. Use when checking or fixing a human-readable Lido governance action list against source calldata, parsed Aragon execution traces, nested EVM scripts, registry names, numbering, formatting, unknown empty-calldata calls, or role and pauser semantics.
---

# Lido Voting Output Verifier

## Objective

Audit `voting_items.md` against the original Lido voting calldata and fix the file in place. Treat the generated action list as untrusted until every effective governance action has been reconciled with source calldata, nested scripts, and the Lido Docs registry.

Use this registry for canonical contract-name checks:

`https://raw.githubusercontent.com/lidofinance/docs/refs/heads/feat/srv3-proposed-contracts/docs/deployed-contracts/index.md`

## Verification Workflow

1. Read the source voting calldata, parsed proposal text, and any linked parsed EVM scripts.
2. Independently reconstruct the effective call list. Expand `AragonAgent.forward(bytes _evmScript)` and similar wrappers; ignore only pure wrappers; keep wrapper calls only when the wrapper itself is the meaningful action.
3. Count effective non-wrapper calls, including empty-calldata calls.
4. Read `voting_items.md` and map each action line to exactly one effective call.
5. Fetch or read the Lido Docs registry and verify every contract name/address pair used in `voting_items.md`.
6. Fix missing, extra, duplicated, mislabeled, misordered, or misformatted actions directly in `voting_items.md`.
7. Re-run the checklist below after edits until no issues remain.

## Coverage Checks

Every effective non-wrapper call from the calldata must appear exactly once.

Every empty-calldata call must appear exactly once, must be marked `[UNKNOWN]`, and must keep the called target address. Include the target contract name only if the address is verified in the registry.

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

## Registry And Label Checks

Verify every parsed contract address against the Lido Docs protocol contracts registry when possible.

Use the registry's canonical contract name only when the address matches. Preserve checksum address casing from the parsed input when available; otherwise use the registry checksum casing if available.

If an address is absent from the registry, mark it as `[UNKNOWN]` and keep the raw address. A parsed label or proposal-context label may be included only after `[UNKNOWN]`.

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

If a contract name does not match the registry address, replace the name with the canonical registry name when matched, or mark the address `[UNKNOWN]` when unmatched.

If numbering is invalid, renumber the smallest necessary range while preserving the relationship between top-level items and decimal subactions.

After every repair pass, recount effective calls, remap all action lines, and re-check formatting until the file is clean.
