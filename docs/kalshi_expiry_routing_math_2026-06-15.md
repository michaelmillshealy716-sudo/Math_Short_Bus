# Kalshi Expiry Routing Math

Source implementation: `michaelmillshealy716-sudo/hvl-workstation`

Source commit: `6994de1 fix(kalshi): guard expiry routing math`

## Problem

The Kalshi bankroll engine entered a paper trade on a contract whose API
`close_time` implied multiple hours remained, while the ticker hour implied the
contract was about to expire. The engine was using the API close-time value for
its time-to-expiry gate, so a stale or inconsistent market payload could make a
dying contract look tradeable.

## Invariant

New entries must use the strictest available expiry clock:

```text
contract_entry_seconds_to_close =
  min(api_seconds_to_close, ticker_hour_seconds_to_close)
```

If either source says the contract is inside the hard cutoff, the engine must
not open a new position.

```text
blocked =
  ticker_hour_already_swept
  OR contract_entry_seconds_to_close <= max(
       hard_no_entry_final_seconds,
       no_entry_final_seconds,
       min_hold_seconds
     )
```

The hard floor is pinned to at least `120` seconds.

## Regression Case

This is the failure pattern that must stay blocked:

```text
ticker:             KXBTCD-26JUN1417-T63499.99
now:                2026-06-14T16:59:19Z
api close_time:     2026-06-14T21:00:00Z
ticker-hour expiry: 2026-06-14T17:00:00Z
effective TTE:      41 seconds
hard cutoff:        120 seconds
decision:           contract_expiry_cutoff_active
```

## Runtime Fields Added

The engine now records both raw clocks and the effective clock in telemetry:

- `api_seconds_to_close`
- `ticker_seconds_to_close`
- `contract_entry_seconds_to_close`
- `contract_entry_cutoff_seconds`
- `hard_no_entry_final_seconds`
- `contract_entry_cutoff_blocked`

For any future `BUY`, the entry record stores:

- `entry_api_seconds_to_close`
- `entry_ticker_seconds_to_close`
- `entry_contract_seconds_to_close`
- `entry_contract_cutoff_seconds`

## Validation

The HVL regression suite was run after the patch:

```text
.venv/bin/python3 test_engine_pipes.py
```

Result:

```text
[PASS] Cauchy stabilizer integration pipes verified.
```
