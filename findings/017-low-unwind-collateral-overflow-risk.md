# Unwind Collateral Formula Overflow for Large Amounts

## Description

* The collateral withdrawal formula multiplies `_amount * debtTokenPrice * (10 ** collateralDecimals) * LTV_PRECISION` before dividing.

* For large `_amount` and 18-decimal tokens, this product can exceed `uint256` and cause an overflow revert.

```solidity
@>          uint256 collateralToWithdraw = (
                _amount * debtTokenPrice * (10 ** IERC20(unwindParams.collateralToken).decimals()) * LTV_PRECISION
            ) / (collateralTokenPrice * (10 ** IERC20(_asset).decimals()) * liqThreshold);
```

## Risk

**Likelihood (low)**:

* Requires very large positions (e.g. billions of units with 18 decimals).

* Typical DeFi amounts stay below overflow range.

**Impact (medium)**:

* Unwind reverts; large positions cannot be unwound.

**Severity (low)**:

## Proof of Concept

The numerator is computed as a single product before division. With 18-decimal tokens and typical price precision (8 decimals), the product scales as `_amount * 1e8 * 1e18 * 1e4 = _amount * 1e30`. For overflow, we need `_amount * 1e30 > uint256.max ≈ 1.15e77`, so `_amount > ~1e47`. In human units (18 decimals), that is ~10^29 tokens—far beyond any realistic position. Typical amounts (e.g. 1e18–1e24) remain safe. The risk is theoretical but worth noting for protocols that may support very large positions, custom tokens with high decimals, or future scaling.

```solidity
// _amount = 1e27, debtTokenPrice = 1e8, 10^18, LTV_PRECISION = 1e4
// numerator ≈ 1e57; uint256 max ≈ 1.15e77
// For _amount = 1e30: numerator ≈ 1e60, still safe
// Overflow threshold: _amount ≈ 1e47 (raw units) → ~10^29 tokens with 18 decimals
```

## Recommended Mitigation

**Option 1: Use a checked-math library** — Replace the inline multiplication with OpenZeppelin's `Math.mulDiv` (or similar) so overflow reverts with a clear error instead of an opaque arithmetic failure. The library performs multiplication and division in a way that avoids intermediate overflow when the final result fits in `uint256`.

**Option 2: Enforce a maximum `_amount`** — Add a cap in `unwindPosition` (e.g. `require(_debtAmount <= MAX_UNWIND_AMOUNT, "Amount too large")`) so the formula stays within safe bounds. Choose `MAX_UNWIND_AMOUNT` conservatively (e.g. well below 1e47 for 18-decimal tokens) to leave headroom for price and decimal variations.

**Option 3: Split the calculation** — Break the numerator into smaller multiplications and divide early where possible (e.g. divide by the denominator term before multiplying by the next factor) to keep intermediate values below `uint256.max`. This requires careful ordering to preserve precision.
