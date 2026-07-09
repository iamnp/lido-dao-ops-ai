Task: Work with Lido voting calldata by using the local skills in `skills/`.

Use `skills/lido-voting-action-list` when converting provided Lido voting calldata or parsed execution traces into the final `voting_items.md` action list.

Use `skills/lido-voting-output-verifier` after generation, or whenever asked to review/fix `voting_items.md`, to verify the output against the source calldata and repair issues in place.

Use `skills/lido-raw-evm-script-verifier` when comparing or verifying raw Aragon/Lido EVM script bytes against parsed voting calldata, decoded execution traces, or parsed call lists.

For normal end-to-end work, run the action-list skill first and the verifier skill second. If raw EVM script bytes must first be reconciled with parsed calldata, run the raw EVM script verifier before the action-list skill.
