# Nines Bond

This project contains a GenLayer smart contract for an SLA outage credit policy. The deployed contract is:

- Explorer: https://explorer-studio.genlayer.com/address/0x340289bE2e8F218b64A90916f3B0c1918EbafBe9
- Contract address: 0x340289bE2e8F218b64A90916f3B0c1918EbafBe9

The contract source is in src/Ninesbond.py and the active direct contract tests are in test/direct/test_sla_outage_credit.py.

## What the contract does

The contract lets a customer buy outage protection for a provider-backed service. A policy stores the provider, the service name, billing period, incident date, resolution date, credit amount, and two public status URLs to inspect.

The policy is active from purchase until it is resolved or refunded. When it becomes resolvable, the contract reads both public status pages and checks whether they show a relevant outage for the named service on the specified incident date.

## How it works

1. A customer calls buy_cover with payment attached.
2. The contract validates dates, credit, provider identity, and source URLs.
3. The policy is stored as ACTIVE with funds marked as RESERVED.
4. When resolve_after is reached, resolve(policy_id) evaluates the claim.
5. The contract uses a nondeterministic adjudication flow to compare the verdict against the two status sources.
6. The result can be:
   - YES: the outage is confirmed, and the customer receives the configured credit
   - NO: no qualifying outage is found, and the provider keeps the premium
   - UNKNOWN or DISAGREE: the outcome is inconclusive, and the policy remains active without paying or keeping funds
7. If the policy is still unresolved after resolve_after, timeout_refund(policy_id) returns the premium to the customer.

## Important validation rules

- customer and provider must be different addresses
- premium must be greater than zero
- credit must be greater than zero and cannot exceed the premium
- incident_date must fall inside the covered period
- resolve_after must be on or after incident_date
- status_url_a and status_url_b must be HTTPS and must come from different hosts
- source hosts are restricted to a known allowlist of public service status pages

## Policy data and views

Each policy stores:

- customer and provider addresses
- service and date range
- premium and credit
- status values such as ACTIVE, CREDITED, RETAINED, and REFUNDED
- verdict values such as YES, NO, UNKNOWN, DISAGREE, and TIMEOUT
- funds_disposition showing how funds were handled

The contract exposes read-only methods including:

- get_policy(policy_id)
- get_policy_count()
- get_reserved_premiums()
- can_resolve(policy_id)

## Current test state

The current direct tests in test/direct/test_sla_outage_credit.py verify the main behavior of the contract, including:

- rejecting a customer who also sets themselves as the provider
- requiring two distinct status sources
- preventing a credit above the premium
- validating that the incident date falls within the covered period
- blocking early resolution or timeout refund before resolve_after
- refunding the premium on timeout
- paying the credit on a confirmed outage
- leaving funds reserved when the adjudication verdict is UNKNOWN