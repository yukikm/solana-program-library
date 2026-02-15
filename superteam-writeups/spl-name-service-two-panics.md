# SPL Name Service: two panic-level DoS issues fixed (missing parent owner + OOB write)

Repo audited:
- https://github.com/solana-labs/solana-program-library
- Program: `name-service/program` (`spl-name-service` / `spl_name_service`)

Patch:
- Commit: `8c54b11901a01273aa1cdf7cd1beb19386c7c5b8`
- Patch file (for offline review): `superteam/spl-name-service-fix.patch`

## Summary
I found two separate panic-level bugs in the SPL Name Service program that allow an attacker (or just a malformed client) to trigger a **program abort** instead of a clean `ProgramError`.

While these do not allow theft, they are **denial-of-service (DoS) vectors per-instruction**, and can break UX / tooling that expects deterministic error codes, and can cause unnecessary compute consumption.

This PR hardens the program by converting both cases into explicit error returns:
- missing required account → `NotEnoughAccountKeys`
- out-of-bounds write attempt → `InvalidArgument`

## Issue 1 — `process_create`: missing `parent_name_owner` account causes `.unwrap()` panic

### Where
`name-service/program/src/processor.rs` (Create flow)

The code reads an optional account:
- `let parent_name_owner = next_account_info(accounts_iter).ok();`

…but later does:
- `parent_name_owner.unwrap()` when `parent_name_account` is non-default.

### Impact
A transaction can pass a non-default `parent_name_account` but omit the final `parent_name_owner` account. That makes `parent_name_owner = None`, and the subsequent `.unwrap()` causes a panic/abort.

This is:
- an easy footgun for integrators
- a trivial per-instruction DoS vector (program abort)

### Reproduction (pre-fix)
Construct a `NameRegistryInstruction::Create` instruction where:
- `parent_name_account != Pubkey::default()`
- *do not include* `parent_name_owner` in the accounts list

Expected behavior should be a clean error like `NotEnoughAccountKeys`, but pre-fix it aborts due to `.unwrap()`.

### Fix
If `parent_name_account` is non-default, require the `parent_name_owner` account:
- `let parent_name_owner = parent_name_owner.ok_or(ProgramError::NotEnoughAccountKeys)?;`

This prevents the panic and yields a deterministic error.

## Issue 2 — `write_data`: out-of-bounds slice write causes panic (abort)

### Where
`name-service/program/src/state.rs`

The helper:
```rust
pub fn write_data(account: &AccountInfo, input: &[u8], offset: usize) {
    let mut account_data = account.data.borrow_mut();
    account_data[offset..offset.saturating_add(input.len())].copy_from_slice(input);
}
```

There is **no bounds check**. If `offset + input.len()` exceeds `account_data.len()`, Rust panics with an out-of-range slice.

### Impact
Any caller of the Update instruction can supply an oversized offset (or any combination of `offset` and `data.len()` that exceeds the account’s data length) to trigger a panic/abort.

This is a clean example of untrusted input reaching a panic path.

### Reproduction (pre-fix)
Call `NameRegistryInstruction::Update` with:
- `offset` set so that `NameRecordHeader::LEN + offset + data.len() > name_account.data_len()`

In the upstream tests, the line:
- `update(program_id, space as u32, data, ...)`

already triggers a panic today (it was previously only asserted as “error”, but the failure mode is an abort, not a clean error).

### Fix
Change `write_data` to return `Result<(), ProgramError>` and explicitly check bounds:
- if `end > account_data.len()` return `ProgramError::InvalidArgument` and log a message.

Update callsites (`process_update`, `process_delete`) to propagate the error with `?`.

## Verification / Proof
I added a regression test to demonstrate Issue #1 and to ensure Issue #2 fails cleanly.

Run:
```bash
cargo test -p spl-name-service --features test-sbf -- --nocapture
```

Expected outputs after fix:
- Missing parent owner create attempt fails with `InstructionError::NotEnoughAccountKeys`
- OOB update attempt fails with `InstructionError::InvalidArgument`

(See `name-service/program/tests/functional.rs` for the exact test cases.)

## Notes
- These are minimal, backward-compatible hardening changes. They do not change any valid success path.
- The new errors are deterministic and more friendly to client integrations.
