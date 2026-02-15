# spl-name-service: missing `parent_name_owner` account causes panic/abort in `Create`

**Repo:** https://github.com/solana-labs/solana-program-library

**Program / crate:** `name-service/program` (`spl-name-service`)

**Issue type:** input-validation bug → `unwrap()` panic (deterministic abort)

**Severity:** Low (reliability / DoS against the instruction path; no state corruption)

## Summary

In `spl-name-service`'s `Create` instruction processor, when a non-default `parent_name_account` is provided, the program assumes the optional `parent_name_owner` account meta is also provided. The code reads the next account using `next_account_info(accounts_iter).ok()` and later calls `unwrap()` on the `Option<AccountInfo>`.

A caller can craft a `Create` instruction that sets `parent_name_account != Pubkey::default()` but omits the `parent_name_owner` account from the account metas. This triggers a panic/abort rather than returning a normal `ProgramError`.

While Solana transactions are atomic (so state isn’t partially committed), this is still an undesirable failure mode: it’s ungraceful, produces confusing errors for integrators, and can be used to reliably fail this instruction path (DoS-style against the specific create-with-parent flow).

## Impact

- Any user can trigger an abort in the `Create` instruction path when using a parent name.
- The program fails ungracefully instead of returning a deterministic `ProgramError`.
- Integrations that expect consistent error codes may behave incorrectly.

## Affected code

File: `name-service/program/src/processor.rs`

Function: `Processor::process_create`

Root cause: `parent_name_owner` is parsed as `Option<AccountInfo>`, but later `unwrap()` is used when `parent_name_account` is non-default.

## Reproduction

Construct a `Create` instruction with:

- `parent_name_account` set to a non-default pubkey (e.g., a real parent name account)
- **omit** the `parent_name_owner` account meta entirely

Expected behavior: program should return `ProgramError::NotEnoughAccountKeys` (or another explicit error).

Actual behavior (before fix): program aborts due to `unwrap()` on `None`.

## Fix

- Replace the `unwrap()` calls with an explicit check.
- If `parent_name_account` is non-default but `parent_name_owner` is missing, return `ProgramError::NotEnoughAccountKeys` and log a helpful message.

## Verification / proof

A regression test was added using SBF ProgramTest:

- Attempts to create a child name with a non-default parent but **without** passing `parent_name_owner`.
- Asserts the transaction fails with `InstructionError::NotEnoughAccountKeys`.

Run:

```bash
cd name-service/program
cargo test --features test-sbf
```

## Patch

Commit: `57b1f07f69cc6f9b9d11a8c0a09f1d7b5b54b8d4`

Branch: `fix/name-service-missing-parent-owner`

---

If you maintain this program and prefer a different error code (e.g., a custom error), I’m happy to adjust, but the key security improvement is eliminating the panic/abort in favor of a deterministic `ProgramError`.
