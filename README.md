# AgentEscrow

**An escrow contract for bounties paid to autonomous agents, written in [Fe](https://fe-lang.org).**

Agent marketplaces run on a simple promise: *do the work, get paid.* That
promise needs a neutral holder for the money while the work is in flight.
AgentEscrow is that holder — a small, auditable escrow contract purpose-built
for poster → agent bounty payments:

- A **poster** locks ETH into an escrow for a job, naming the **worker**
  (payee) and an optional **arbiter**, plus a deadline.
- **Release** — the poster or the arbiter pays the worker when the work is
  accepted.
- **Refund** — the arbiter can refund the poster at any time (work
  rejected); the poster can refund themselves once the deadline has passed
  (work never delivered).
- An arbiter of `address(0)` means *no arbiter*: only the poster can
  release, and only the deadline can unlock a refund.

The full lifecycle of every escrow is queryable on-chain (`getEscrow`,
`totalLocked`) and every transition emits an event, so marketplaces,
dashboards and the agents themselves can follow escrow state without
trusting an off-chain backend.

## Why Fe

This project is built for the Fe Builders Challenge and deliberately uses
the parts of Fe that make escrow logic safer to write and review:

- **Effects, declared per handler.** Every message handler lists exactly
  which capabilities it touches — `ctx` (call context), `call` (sending
  value), `log` (events) and each individual storage field. A read-only
  getter *cannot* move funds; the effect list proves it at a glance.
- **Unit-safe amounts and times.** Balances are `Wei` and deadlines are
  `Timestamp`/`Duration`, not bare `uint256`. Mixing up wei and seconds is
  a compile error, not an audit finding.
- **Message-typed interface.** The public interface is a `msg` block with
  standard Solidity selectors (`sol("create(address,address,uint256)")`),
  so the contract drops into existing Ethereum tooling unchanged — the
  generated ABI is the familiar JSON ABI.
- **Storage structs in maps.** Each escrow is a plain struct stored in a
  `StorageMap<u256, Escrow>` — no manual slot arithmetic, no parallel
  arrays drifting apart.

## Interface

| Function | Type | Description |
| --- | --- | --- |
| `create(address payee, address arbiter, uint256 deadline)` | payable | Lock `msg.value` into a new escrow; returns the escrow id. Reverts on zero value, zero payee, or a past deadline. |
| `release(uint256 id)` | — | Pay the escrow amount to the payee. Poster or arbiter only, escrow must be open. |
| `refund(uint256 id)` | — | Return the escrow amount to the poster. Arbiter any time; poster only at/after the deadline. |
| `getEscrow(uint256 id)` | view | Return `(payer, payee, arbiter, amount, deadline, state)` for an escrow. |
| `nextId()` | view | Id the next escrow will receive (also the count of escrows created). |
| `totalLocked()` | view | Total ETH currently held across all open escrows. |

Escrow states: `0` none · `1` funded (open) · `2` released · `3` refunded.

Events: `EscrowCreated(id, payer, payee, arbiter, amount, deadline)`,
`EscrowReleased(id, payee, amount)`, `EscrowRefunded(id, payer, amount)`
(indexed fields: id, payer/payee as applicable).

## Safety notes

- **Checks–effects–interactions.** The escrow state is settled and
  `totalLocked` is updated *before* any value is sent, so a re-entrant
  call observes a closed escrow.
- **Failed sends revert.** The result of every value transfer is asserted;
  if the payment fails, the whole call reverts and the escrow stays open
  rather than being marked paid.
- **No owner, no admin, no upgrade path.** Once deployed, nobody — not
  even the deployer — can move funds outside the release/refund rules.

## Build & test

Requires the Fe toolchain (26.x). Install with
[feup](https://github.com/argotorg/fe) or download the `fe` binary from the
[official releases](https://github.com/argotorg/fe/releases):

```sh
fe build   # writes out/AgentEscrow.bin, .runtime.bin and .abi.json
fe test    # runs the #[test] suite in src/lib.fe
```

Expected test result with Fe 26.6.0:

```
PASS  create_then_release_pays_the_payee
PASS  create_with_no_value_reverts
PASS  create_with_past_deadline_reverts
PASS  double_release_reverts
PASS  escrows_get_sequential_ids
PASS  refund_before_deadline_reverts
PASS  release_unknown_escrow_reverts

test result: ok. 7 passed; 0 failed
```

### Test coverage — stated honestly

The `fe test` harness drives contracts from a single caller account with
a fixed block clock, so two branches cannot be exercised end-to-end in
this suite yet: an *arbiter-initiated* release/refund (the caller in tests
is always the poster) and a *poster refund after the deadline* (the clock
cannot be advanced). Both branches share their authorization and payout
code paths with the covered flows, the deadline gate itself is covered by
`refund_before_deadline_reverts`, and covering the remaining branches via
a Foundry/Anvil integration test is on the roadmap below.

## Project layout

```
fe.toml        ingot manifest
src/lib.fe     the contract, its events, and the test suite
out/           build artifacts (generated, git-ignored)
```

## Roadmap

- Foundry/Anvil integration tests for arbiter flows and deadline refunds.
- Testnet deployment (Sepolia) with Sourcify verification.
- Optional dispute state: freeze an escrow pending arbiter decision.
- ERC-20 escrows alongside native ETH.

## License

Apache-2.0
