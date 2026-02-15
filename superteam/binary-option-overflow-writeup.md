# SPL `binary-option` — integer overflow in `process_trade` payment calculations (escrow under/over-payment)

Repo: https://github.com/solana-labs/solana-program-library

Component:
- `binary-option/program/src/processor.rs`

Patch:
- Commit (local): `28001a9` (branch: `fix/binary-option-overflow`)
- Patch file: `superteam/binary-option-overflow-fix.patch`

## Summary
The on-chain `binary-option` program computes escrow payment amounts using unchecked `u64` arithmetic like `n * buy_price` and `(n - n_b) * buy_price`.

In Solana programs compiled in release mode, integer overflow **wraps** (mod 2^64) instead of reverting, which can make required escrow transfers **much smaller than intended** (or otherwise incorrect).

This can enable a trader to:
- receive/mint/burn option tokens for a large `size` while
- paying a wrapped-around (tiny) amount of escrow tokens,

breaking the intended economic invariants of the betting pool.

## Impact
- **Economic integrity failure**: A malicious user can craft `size` values that cause overflow in the amount transferred to/from escrow, resulting in underpayment/overpayment.
- **Potential loss of funds**: If the program transfers out of escrow based on overflowed amounts, escrow token balances can be drained or accounting can be corrupted.

Severity depends on whether the program is deployed and used with meaningful liquidity.

## Vulnerable code paths
In `process_trade`, the following patterns were present (examples):
- `n * sell_price`
- `n * buy_price`
- `(n - n_b) * buy_price`
- `(n - n_s) * sell_price`
- `n_b * sell_price`
- `n_s * buy_price`

Additionally, the price normalization check used `u64::pow(10, decimals)` which can overflow for large `decimals` values.

## Reproduction (conceptual, minimal)
The invariant intended by the program is:

- `buy_price + sell_price == 10^decimals`
- For a trade of size `n`, escrow transfers should scale linearly with `n`.

However, if `n * buy_price` overflows:

```text
amount = (n * buy_price) mod 2^64
```

An attacker can choose `n` such that the multiplication wraps.

Example (illustrative):
- `decimals = 9` (so `10^decimals = 1_000_000_000`)
- `buy_price = 1_000_000_000`, `sell_price = 0`
- Choose `n = floor(u64::MAX / buy_price) + 1`

Then `n * buy_price` wraps to a small value. The program would attempt to transfer only that wrapped value while treating the trade size as `n`.

Whether the trade is feasible depends on balances (SPL amounts are `u64`, so very large supplies are representable), but the core bug is the unchecked arithmetic.

## Fix
The patch:
1. Introduces local helpers in `process_trade`:
   - `checked_mul(qty, price)`
   - `checked_sub(a, b)`
   returning `BinaryOptionError::AmountOverflow` on failure.
2. Replaces all price/size arithmetic used for token transfers with checked operations.
3. Replaces `u64::pow(10, decimals)` with `10u64.checked_pow(decimals)`.
4. Adds a guard in `process_initialize_binary_option` rejecting `decimals > 19` (since `10^20` exceeds `u64::MAX`).

## Verification
- Manual audit confirms all `* buy_price` / `* sell_price` multiplications and `(n - x)` subtractions in `process_trade` now use checked arithmetic.
- Any overflow now returns `BinaryOptionError::AmountOverflow` instead of silently wrapping.

Notes:
- Building this workspace locally required Rust >= 1.79 due to `solana-program = 2.1.0`; the current environment had Rust 1.75, so full compilation wasn’t possible here.
- The patch is small, deterministic, and should compile on a modern toolchain.

## Patch / diff
- Patch file: `superteam/binary-option-overflow-fix.patch`
- Commit: `28001a9`

