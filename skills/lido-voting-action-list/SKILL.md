---
name: lido-voting-action-list
description: Convert provided Lido DAO voting calldata or parsed vote execution traces into a concise human-readable governance action list in voting_items.md. Use when the user supplies Lido voting calldata, Aragon vote call data, parsed EVM scripts, or proposal execution items and asks for voting items, action lists, or human-readable Lido governance actions.
---

# Lido Voting Action List

## Objective

Read the voting calldata supplied by the user and write only the effective governance actions to `voting_items.md`.

Expand nested execution wrappers, especially `AragonAgent.forward(bytes _evmScript)` entries with `See parsed evm script at ...`. Ignore wrapper calls in the final action list unless the wrapper itself is the meaningful action. Do not ignore calls whose calldata is parsed as `[empty]`, and do not mark those calls as `[UNKNOWN]`.

## Required Inputs

Use the voting calldata, parsed execution trace, proposal metadata, and any linked parsed EVM scripts supplied by the user. Fetch and use the Lido Docs protocol contracts registry before writing final output:

`https://raw.githubusercontent.com/lidofinance/docs/refs/heads/feat/srv3-proposed-contracts/docs/deployed-contracts/index.md`

Use the registry to verify parsed contract addresses and obtain canonical contract names. Do not use a registry name unless the address matches. For the matching address entry, render a contract marked `[proposed to remove]` as `old <canonical contract name>` and a contract marked `[proposed]` as `new <canonical contract name>`. Treat these markers as address-specific; do not prefix an unmarked entry merely because another entry has the same canonical name. If an address is absent from the registry, mark it as `[UNKNOWN]` and keep the raw address; include any label supplied by parsed calldata or proposal context only after `[UNKNOWN]`. Calls whose calldata is parsed as `[empty]` are an exception: never add `[UNKNOWN]` to those calls, even when the target is absent from the registry.

## Workflow

1. Parse all top-level voting items and all nested scripts referenced by wrappers.
2. Build the effective call list by removing pure wrappers and retaining every non-wrapper action exactly once.
3. Preserve all material values from calldata: contract names, addresses, role hashes, IDs, limits, durations, names, and accounts.
4. Verify every target address against the Lido Docs registry when possible.
5. Convert action semantics into concise one-sentence action lines.
6. Create or overwrite `voting_items.md` with only the final action list.
7. Run the checks in `Verification` before finishing.

## Output Format

The final output artifact is `voting_items.md`; it must contain only the action list.

Group related actions under Roman-numeral section headings:

```text
I. <short objective>
II. <short objective>
```

Do not put the Dual Governance submission item into any Roman-numeral section. Output it first as standalone item `1.`, then start section `I.` immediately after it.

Use integer labels for top-level voting items: `1.`, `2.`, `3.`, etc. Use decimal labels only for expanded subactions inside a top-level voting item: `1.1.`, `1.2.`, `1.3.`, etc. Treat the Dual Governance submission as top-level voting item `1.`; label other top-level voting items as `2.`, `3.`, etc., not as further `1.x.` subactions.

Each action line must be one sentence with this shape:

```text
<number>. <verb> <object/value> <preposition phrase with target contract name and address>
```

Do not indent generated output; every section heading and action line must start at column 1. Do not put a period at the end of section headings or action lines; keep the period after the action number only. Keep headings and action lines concise. Do not include raw ABI blocks, calldata arrays, or examples. Use blank lines between logical groups inside a large section when it improves scanability.

Use checksum address casing when available from the parsed input. Do not lowercase addresses in the final output.

Wrap every address, hash, numeric ID, limit, duration, token amount, role value, account value, and string value from calldata in backticks. Put hashes and addresses verbatim inside backticks. Do not shorten them.

## Grouping Rules

Group repeated role migrations into one section when they share the same objective.

For migration triplets, keep this order:

1. revoke old permission
2. grant new permission
3. register or activate the replacement authority

Create separate sections for unrelated policy changes, node-operator changes, Easy Track limit changes, registry updates, or deactivations.

Derive section titles from proposal metadata when it is clear; otherwise summarize the dominant action in plain English.

## Function Actions

For `revokeRole(bytes32 role, address account)`, write:

```text
Revoke <ROLE_NAME> <role_hash> from <account label/address> on <target contract label> <target address>
```

For `grantRole(bytes32 role, address account)`, write:

```text
Grant <ROLE_NAME> <role_hash> to <account label/address> on <target contract label> <target address>
```

For `registerPauser(address _pausable, address _newPauser)`, write:

```text
Register <_pausable label/address> on <called contract label/address> with pauser <_newPauser>
```

For `setLimitParameters(uint256 _limit, uint256 _periodDurationMonths)`, convert 18-decimal token amounts to token units when the registry or factory context indicates stETH, then write:

```text
Set limit to <amount> stETH per <months> months on <registry/factory label> <address>
```

For `setNodeOperatorName(uint256 _nodeOperatorId, string _name)`, write:

```text
Change name to <_name> for Node Operator <known old name if available> (id = <id>) in <module label> <address>
```

For `deactivateNodeOperator(uint256 _nodeOperatorId)`, write:

```text
Deactivate Node Operator <known name if available> (id = <id>) in <module label> <address>
```

For proxy upgrade functions such as `upgradeTo(address)`, `upgradeToAndCall(address,bytes)`, and equivalent proxy-admin or upgrade-template calls, write:

```text
Upgrade <contract name> <proxy address> to implementation <new implementation address>
```

Do not repeat the contract name after `implementation` or immediately before the new implementation address. Name the upgraded contract once, before the proxy address. Append setup calldata, force-call flags, or other material arguments after the new implementation address when present.

## Normalization Rules

Retain every call whose calldata is parsed as `[empty]`, but never mark the call as `[UNKNOWN]`. Use known semantics from the trace or proposal context when available. Otherwise describe the observable operation neutrally, such as an empty-calldata call to the target, or an ETH transfer when the call value is nonzero. Always keep the called target address and exact call value. Include a canonical contract name only when verified in the registry; otherwise keep the raw address and any supplied parsed label without inventing a label. This rule overrides the general `[UNKNOWN]` rule for addresses absent from the registry.

Do not invent missing labels. If a label is unavailable, use the raw address or hash and optionally note the uncertainty.

Normalize `[PAUSE ROLE]` to `PAUSE_ROLE`.

If a role hash is unlabeled but can be confidently identified from nearby calls or known contract docs, include the role name; otherwise output the hash without a role name.

Prefer domain names used in Lido docs or parsed labels, but keep the exact target address so the action remains verifiable.

For token amounts encoded as wei, divide by `1e18` and present the human amount without unnecessary decimals.

For node operator actions, include both ID and name when both are present or discoverable from the provided calldata or context.

## Verification

Before finishing, check that `voting_items.md` contains the final action list and no raw ABI blocks, calldata arrays, or examples.

Count effective inner calls and ensure the final numbered list covers every non-wrapper action exactly once.

Check that every call parsed with calldata `[empty]` is represented exactly once and is not marked as `[UNKNOWN]`.

Check that each `grantRole` and `revokeRole` action names the called contract as the permission target, not the forwarding agent.

Check that `registerPauser` uses the called contract as the registry or authority and the first argument as the pausable contract.

Check contract names and addresses against the Lido Docs protocol contracts registry.

Check that every registry entry marked `[proposed to remove]` uses `old` immediately before its contract name, every entry marked `[proposed]` uses `new` immediately before its contract name, and unmarked entries use neither prefix.

Check that every address absent from the registry is marked with `[UNKNOWN]`, except calls whose calldata is parsed as `[empty]`, which must never be marked `[UNKNOWN]`.

Check that the Dual Governance submission appears before section `I.` and is not included under any section heading.

Check that every generated section heading and action line starts at column 1 with no leading spaces.

Check that section headings match the actions below them and that integer top-level numbering is continuous, with decimal numbering used only for subactions of the corresponding top-level item.

Check that every address, hash, and material value is wrapped in backticks.

Check that every proxy upgrade names the upgraded contract only before the proxy address and does not repeat that contract name before the new implementation address.
