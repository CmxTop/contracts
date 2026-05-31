# Test suite: multicall E2E (#135), property tests (#136), ADL E2E (#134), differential tests (#137)

## Summary

- closes #135 — Router multicall E2E (atomic success + atomic revert); fixes double `require_auth` contract bug
- closes #136 — Property tests for math, pricing, and position utilities (monotonicity, bounds, overflow safety)
- closes #134 — ADL E2E through deployed-style handler clients (trigger, no-trigger, keeper gate)
- closes #137 — Differential tests against hand-derived GMX reference values (formula drift detection)

---

## Issue #135 — Router multicall E2E (`contracts/exchange_router/src/lib.rs`)

**Bug fixed:** `multicall` delegated to `Self::create_order`, `Self::create_deposit`, etc. — each of which calls `caller.require_auth()`. Soroban rejects a second `require_auth()` for the same address in the same invocation frame with `Error(Auth, ExistingValue)`. Fixed by having `multicall` call handler clients directly, so `require_auth()` fires exactly once per multicall.

**New tests (2):**

| Test | Assertion |
|---|---|
| `e2e_multicall_send_tokens_then_create_order_succeeds` | SendTokens + CreateOrder completes atomically; order key is non-zero and executable; trader collateral moves to vault. |
| `e2e_multicall_failed_step_reverts_preceding_steps` | A failing CancelOrder (unknown key) as Step 2 rolls back the SendTokens from Step 1; trader balance and vault balance are both unchanged. |

---

## Issue #136 — Property tests for math utilities

**Bug fixed in `integer_sqrt`:** used `(x + 1) / 2` as Newton initialisation, which overflows for `n = i128::MAX` in debug mode. Changed to `(x >> 1) + (x & 1)`.

**New tests (22 across 3 files):**

`libs/math/src/lib.rs` — 11 properties:
- `ceil >= floor` for all positive inputs to `mul_div_wide` / `mul_div_wide_up`
- `mul_div_wide` monotone in `a`; `apply_factor` identity / zero-factor
- `integer_sqrt` monotone, correct on perfect squares, never negative (including `i128::MAX`)
- `bound_above_zero` clamps negatives; `abs_safe` never negative; `pow_factor` x⁰ = 1, x¹ = x
- Zero denominator returns 0 without panic

`libs/pricing_utils/src/lib.rs` — 6 properties:
- Swap impact zero / non-positive on balanced pool; negative impact monotone in `amount_in`
- Position impact non-positive on balanced OI
- `get_execution_price` unchanged with zero impact; raised by negative impact for longs
- `apply_swap_impact_value` with zero impact is a no-op

`libs/position_utils/src/lib.rs` — 8 properties:
- PnL zero for zero-size position, at entry price, negative when price falls, positive when price rises
- Partial-close PnL proportional to size delta (within 1 unit rounding)
- All fee components non-negative; `total_cost == sum of components`
- `is_liquidatable` returns false for a well-collateralised position at entry price

---

## Issue #134 — ADL E2E through deployed-style clients (`contracts/adl_handler/src/lib.rs`)

Added dev-dependencies to `Cargo.toml` so the full protocol stack can be spun up in tests.

**New tests (4):**

| Test | Assertion |
|---|---|
| `e2e_adl_executes_when_pnl_factor_exceeds_threshold` | Opens profitable long, sets low ADL cap, confirms `is_adl_required` true, calls `execute_adl` via client, verifies position size decreases. |
| `e2e_adl_reverts_when_pnl_factor_below_threshold` | High threshold → `execute_adl` panics with `AdlNotRequired`. |
| `e2e_adl_requires_adl_keeper_role` | No `ADL_KEEPER` role → panics with `Unauthorized`. |
| `e2e_adl_not_required_when_position_unprofitable` | `is_adl_required` returns false when PnL ≤ 0, even with strictest threshold. |

---

## Issue #137 — Differential tests against GMX reference formulas

Pins known numeric results derived by hand. Unintentional deviations fail the assertion; intentional ones must be documented.

**New tests (13 across 3 files):**

`libs/market_utils/src/lib.rs` — 5 tests:
- `get_pnl` long: `5 ETH × $3000 − $10000 = $5000`
- `get_pnl` short: `$8000 − 4 ETH × $1500 = $2000`
- `get_pool_value` (no positions): `5×$2000 + 4000×$1 = $14000`
- `get_borrowing_fees`: `10% × 2 tokens = 0.2 tokens = 2_000_000 units`
- Long PnL exactly 0 at entry price (break-even identity)

`libs/position_utils/src/lib.rs` — 4 tests:
- `get_position_pnl_usd` long profit: `1 token $2000→$3000 = +$1000`
- `get_position_pnl_usd` long loss: `2 tokens $2000→$1000 = −$2000`
- Position fee: `30 bps × $1000 / $2000 = 15_000 units`
- Borrowing fee: `20% × 3 tokens = 6_000_000 units`

`libs/pricing_utils/src/lib.rs` — 4 tests:
- `get_execution_price` with zero impact = index price
- Negative impact `−$100` on `$10000` trade raises execution price above index
- `get_swap_output_amount` no-fee no-impact: `100 tokens × $2000/$1 = 200_000 tokens`
- Swap fee: `0.3% × 1000 tokens = 3 tokens` (ceiling-rounded)

---

## Test results

```
cargo test -p exchange-router      # 6 tests  (4 pre-existing + 2 new)  ✓
cargo test -p adl-handler          # 4 tests  (0 pre-existing + 4 new)  ✓
cargo test -p gmx-math             # 23 tests (12 pre-existing + 11 new) ✓
cargo test -p gmx-pricing-utils    # 14 tests (4 pre-existing + 10 new)  ✓
cargo test -p gmx-position-utils   # 17 tests (5 pre-existing + 12 new)  ✓
cargo test -p gmx-market-utils     # 16 tests (11 pre-existing + 5 new)  ✓
```

All 80 tests pass. No pre-existing tests were broken.

---

🤖 Generated with [Claude Code](https://claude.ai/claude-code)

---

## Issue rgb(7, 31, 31) — Testnet market bootstrap workflow

**Files changed:** `scripts/bootstrap.sh` (new), `mx/deploy.mk`, `mx/tokens.mk`, `scripts/deploy.sh`, `README.md`

Adds a repeatable, end-to-end testnet bootstrap path so a fresh deployment needs no private context beyond the Stellar key names.

`scripts/bootstrap.sh` runs four steps after `deploy-all`:
1. Grants `MARKET_KEEPER`, `ORDER_KEEPER`, `LIQUIDATION_KEEPER`, `ADL_KEEPER`, and `FEE_KEEPER` roles to the configured keeper account.
2. Calls `market_factory.create_market()` and appends `MARKET_TOKEN` to the deploy env file.
3. Writes per-market config keys (`max_pool_amount`, `min_collateral_factor`, `max_leverage`, fee/borrowing/funding factors) to `data_store`.
4. Prints manual seeding instructions (oracle must be live before `execute_deposit`).

`mx/deploy.mk` gains three new targets:
- `make bootstrap` — full post-deploy bootstrap
- `make market-init` — market creation + config only (skips role grants and seed)
- `make seed-liquidity` — prints deposit_handler invocation for seeding the pool

`mx/tokens.mk` gains:
- `make market-tokens` — creates both `TWBTC` and `TUSDC` test tokens in one step

`scripts/deploy.sh` now prints the four next-step commands at the end of a successful deployment so operators know what to run immediately after.

`README.md` gains a **Testnet Market Bootstrap** section with a quick-start checklist, a target reference table, and `SKIP_*` flag documentation for idempotent re-runs.

---

## Issue #28 — Global withdrawal list support

**Files changed:** `contracts/withdrawal_handler/src/lib.rs`

The implementation (key insertion in `create_withdrawal`, key removal in `remove_withdrawal`, reader views `get_withdrawal_count` / `get_withdrawal_keys` / `get_account_withdrawal_*`) already existed. This PR adds `withdrawal_list_reflects_full_lifecycle`, a dedicated integration test that:

- Creates three withdrawals for three distinct users.
- Verifies all three keys appear in both the global list and the per-account lists.
- Cancels the first → asserts key removed from global and account list (count drops to 2).
- Executes the second → asserts key removed from global and account list (count drops to 1).
- Asserts the third withdrawal is still present in the global list and in local storage.

This covers the done-criterion: *"Withdrawal lists remain correct after create, cancel, and execute in tests."*

---

## Issue #56 — Order lifecycle tests for limit swap

**Files changed:** `contracts/order_handler/src/lib.rs`

**Behaviour change:** `execute_order` now checks the trigger price condition for `LimitSwap` orders. When `trigger_price > 0` and `index_price.min > trigger_price`, the execution reverts with `UnsatisfiedTrigger`. When `trigger_price == 0`, there is no price gate (the existing `min_output_amount` check is the only guard, mirroring `MarketSwap`).

**New tests** (`// ── Issue #56`):
| Test | Verifies |
|---|---|
| `limit_swap_above_trigger_price_reverts` | Stale/unfavorable price (index > trigger) → reverts |
| `limit_swap_at_trigger_price_executes` | Price == trigger → executes, user receives `long_tk` |
| `limit_swap_below_trigger_price_executes` | Price < trigger (favorable) → executes |
| `limit_swap_no_trigger_always_executes` | `trigger_price = 0` → always executes |
| `limit_swap_min_output_not_met_reverts` | `min_output_amount = i128::MAX` → reverts |
| `limit_swap_min_output_met_succeeds` | Achievable `min_output_amount` → executes, output ≥ min |

---

## Issue #27 — Global deposit list support

**Files changed:** `contracts/reader/src/lib.rs`, `contracts/deposit_handler/src/lib.rs`

**`reader.rs`** — adds four deposit list query functions symmetric to the existing withdrawal list functions:
- `get_deposit_count(env, data_store) -> u32`
- `get_deposit_keys(env, data_store, start, end) -> Vec<BytesN<32>>`
- `get_account_deposit_count(env, data_store, account) -> u32`
- `get_account_deposit_keys(env, data_store, account, start, end) -> Vec<BytesN<32>>`

Also adds `deposit_list_key` and `account_deposit_list_key` to the `gmx_keys` import.

**`deposit_handler.rs`** — adds `deposit_list_reflects_full_lifecycle`, a multi-user integration test that:
- Creates three deposits for three distinct users.
- Verifies the global count is 3 and all keys appear in global and per-account lists.
- Cancels the first → global count drops to 2, key absent.
- Executes the second → global count drops to 1, key absent.
- Asserts the third deposit key is the only entry in a paginated list query.

This covers the done-criterion: *"Create, cancel, and execute each update list membership correctly. A list query after all three operations returns only the expected keys."*
