# Nines Bond

GenLayer SLA outage credit. A customer prepays a month. Two public status pages must agree that an incident hit the named service on `incident_date`.

- Explorer: https://explorer-studio.genlayer.com/address/0x0ce9eF532FF4572c954232FC2b1e2930A2f25847
- Contract address: 0x0ce9eF532FF4572c954232FC2b1e2930A2f25847

Source: `src/Ninesbond.py`. Direct tests: `test/direct/test_sla_outage_credit.py`.

## What the contract does

A customer buys cover against a provider. The policy stores the provider, service, covered period, incident date, `resolve_after`, `refund_after`, credit, premium, and two status URLs.

`resolve_after` is the first day consensus may adjudicate. `refund_after` is the next UTC day. `timeout_refund` is not callable on the day resolution first opens.

## How it works

1. The customer calls `buy_cover` with the monthly premium attached.
2. Dates are checked as real calendar dates, then ordered.
3. The policy is stored `ACTIVE` with the premium `RESERVED`.
4. On or after `resolve_after`, anyone may call `resolve(policy_id)`.
5. Both pages are read. A stable verdict is `YES`, `NO`, `UNKNOWN`, or `DISAGREE`.
6. Outcomes:
   - `YES`: customer receives the credit. Any leftover premium goes to the provider.
   - `NO`: provider keeps the premium.
   - `UNKNOWN` or `DISAGREE`: policy stays `ACTIVE`. Funds stay reserved.
7. On or after `refund_after`, if the policy is still `ACTIVE`, anyone may call `timeout_refund(policy_id)`. The premium returns to the customer once.

## Validation

- `period_start`, `period_end`, `incident_date`, and `resolve_after` must be real `YYYY-MM-DD` dates before they are compared. `2026-02-31` is rejected.
- `period_end` is on or after `period_start`.
- `incident_date` falls inside the covered period.
- `resolve_after` is on or after `incident_date`.
- `refund_after` is `resolve_after` plus one UTC day. It is stored, not supplied.
- Customer and provider are different addresses.
- Premium is greater than zero. Credit is greater than zero and not above the premium.
- Both status URLs are HTTPS, on the allowlist, and on different hosts.

## Views

- `get_policy(policy_id)` includes `resolve_after` and `refund_after`
- `get_policy_count()`
- `get_reserved_premiums()`
- `can_resolve(policy_id)` returns `allowed` and `timeout_refund_allowed` separately

Statuses: `ACTIVE`, `CREDITED`, `RETAINED`, `REFUNDED`.
Verdicts: `YES`, `NO`, `UNKNOWN`, `DISAGREE`, `TIMEOUT`.

## Tests

`test/direct/test_sla_outage_credit.py` covers:

- customer cannot be the provider
- sources must differ
- credit cannot exceed premium
- incident date must sit inside the period
- impossible calendar dates are rejected
- resolve and timeout are blocked before their dates
- timeout is blocked on the day resolve first opens
- timeout after `refund_after` returns the premium once
- `UNKNOWN` does not pay or keep the premium
