# cloud-itonami-isic-4752

Open Business Blueprint for **ISIC Rev.5 4752**: retail sale of hardware,
paints and glass in specialized stores -- hardware stores, paint/coatings
specialty retailers and glass retailers, distinct from sibling ISIC 4719's
general-merchandise retail.

This repository publishes a hardware/paint/glass-specialty-retail
operations-COORDINATION actor -- sales/inventory/tinting-order transaction
logging, floor-staff scheduling, merchandise supply-order coordination
with registered vendors, and hazmat-handling-safety-concern flagging -- as
an OSS business that any qualified operator can fork, deploy, run, improve
and sell, so an independent hardware/paint/glass store never surrenders
its operations data to a closed back-office SaaS.

Built on this workspace's
[`langgraph`](https://github.com/kotoba-lang/langgraph)
StateGraph runtime (portable `.cljc`, supervised superstep loop,
interrupts, in-mem/Datomic checkpoints) -- the same actor pattern as
every prior actor in this fleet -- here it is **HardwarePaintRetailAdvisor
⊣ HardwarePaintRetailGovernor**. This blueprint's own
`:itonami.blueprint/governor` keyword, `:hardware-paint-retail-governor`,
is a distinct, independent build (no naming-collision precedent question
-- distinct from ISIC 4719's own `:merchandise-retail-governor`).

> **Why an actor layer at all?** An LLM is great at drafting a sales-
> record summary, a staffing proposal, or a supply-order request -- but
> it has no license to actually finalize a hazmat-handling-safety
> clearance for a store handling solvents, aerosols and other flammable
> merchandise, no way to independently confirm a store or a supply-order
> vendor is actually a registered/verified counterparty, and no notion of
> when a "flag this concern" op quietly turns into a claim to have
> already certified a storage area as compliant. Letting it act directly
> invites an unverified store's data entering the ledger, an unverified
> vendor receiving a merchandise order, or -- worst of all -- a
> fabricated claim to have already cleared a solvent spill or certified
> flammable storage as code-compliant, exposing the shop and its staff to
> real liability. This project seals the HardwarePaintRetailAdvisor into
> a single node and wraps it with an independent
> **HardwarePaintRetailGovernor**, a human **approval workflow**, and an
> immutable **audit ledger**.

## Scope: coordination only, never a hazmat-handling-safety authority

This actor is **operations coordination only**. It never performs or
authorizes:

- setting or overriding a shelf/unit price
- directly finalizing a hazmat-handling-safety clearance (certifying a
  solvent/aerosol/flammable-storage area as code-compliant, clearing a
  spill as contained, signing off on a hazmat handling permit, issuing a
  flammable-storage compliance certificate)
- hazmat-handling-safety-authority enforcement (declaring tinting-room
  ventilation compliant, authorizing hazardous waste disposal as
  compliant)

The governor's `scope-exclusion-violations` check re-scans every
proposal for this failure mode independently of the advisor's own
framing, and treats it as a HARD, permanent block regardless of
confidence or how clean everything else is. Flagging a hazmat-handling-
safety concern for a human to triage is exactly this actor's job --
`:flag-safety-concern` is never excluded by this check, only
FINALIZING/certifying/clearing that concern is. **The closed proposal-op
allowlist structurally never includes any op that directly finalizes a
hazmat-handling-safety clearance -- there is no such op to gate, only one
to permanently exclude.**

### Actuation

**Every proposal this actor generates is `:effect :propose`, never a
direct actuation.** Two independent layers enforce this
(`hardwarepaintops.governor`'s `effect-not-propose-violations` HARD check
and `hardwarepaintops.phase`'s phase table, which never puts
`:flag-safety-concern` in any phase's `:auto` set). A human store
operator/hazmat-safety coordinator is always the one who actually acts on
a flagged concern or confirms a high-cost supply order.

## The core contract

```
store/vendor registration + operations-coordination request
        |
        v
   ┌───────────────────────┐   proposal      ┌────────────────────────────┐
   │ HardwarePaintRetail-  │ ─────────────▶ │ HardwarePaintRetailGovernor  │  (independent system)
   │ Advisor (sealed)      │  + citations    │ store-unverified ·          │
   └───────────────────────┘                 │ vendor-unverified ·         │
          │                 commit ◀┼ effect-not-propose ·               │
          │                         │ scope-excluded (hazmat-handling-    │
    record + ledger        escalate ┼ safety-clearance finalization) ·    │
          │              (ALWAYS for│ op-not-allowed                      │
          │       :flag-safety-     │                                      │
          │       concern/high-cost └────────────────────────────┘
          │       supply-order)
          ▼
      human approval
```

**The HardwarePaintRetailAdvisor never commits a proposal the
HardwarePaintRetailGovernor would reject, and a hazmat-handling-safety-
concern flag or a high-cost supply order never commits without a human
sign-off.** Hard violations (an unregistered/unverified store; an
unregistered/unverified supply-order vendor; a non-`:propose` effect;
content touching hazmat-handling-safety-clearance finalization; an op
outside the closed allowlist) force **hold** and *cannot* be approved
past.

## Robotics premise

All cloud-itonami verticals are designed on the premise that a **robot
may perform physical domain work** (here: shelfing, picking, paint
tinting-machine operation, restocking, point-of-sale handling) under
human/robot floor operations gated by store policy. This actor itself
does not dispatch robot/hardware actions -- it is strictly the
operations-coordination layer (sales-record logging, staffing scheduling,
supply-order coordination, hazmat-handling-safety-concern flagging) any
physical-dispatch layer could eventually feed proposals into, always
gated the same way by the independent HardwarePaintRetailGovernor.

## Features

- **Closed proposal-op allowlist**: `log-sales-record`,
  `schedule-staffing-operation`, `coordinate-supply-order`,
  `flag-safety-concern` (all `:effect :propose`). No op in this
  allowlist finalizes a hazmat-handling-safety clearance.
- **Four HARD governor checks** (permanent, un-overridable):
  1. **Store unverified** -- the target store's business registration
     must exist AND be independently registered/verified in the store.
  2. **Vendor unverified** -- for `:coordinate-supply-order` only, the
     named vendor must exist AND be independently registered/verified --
     a supply-chain counterparty-verification gate.
  3. **Effect is :propose** -- any other `:effect` value is rejected.
  4. **Scope exclusion** -- directly finalizing a hazmat-handling-safety
     clearance (certifying storage compliance, clearing a spill as
     contained, signing off on a handling permit, issuing a compliance
     certificate) and an op outside the closed allowlist are both
     permanently blocked.
- **Two ESCALATE (SOFT) gates**, either forces human sign-off:
  - `:flag-safety-concern` -- ALWAYS escalates, regardless of confidence
    or phase. A "flag a concern" op is never auto-commit eligible and
    never finalizes a hazmat-handling-safety-clearance decision itself --
    it only surfaces the concern for a human.
  - `:coordinate-supply-order` above a cost threshold -- a large-value
    procurement proposal always needs a human sign-off.
  - (LLM confidence below the floor also escalates, as with every
    sibling actor.)
- **Staged rollout** (Phase 0→3):
  - Phase 0: read-only
  - Phase 1: sales-record logging only (approval-gated)
  - Phase 2: + staffing-operation scheduling, supply-order proposals
    (approval-gated)
  - Phase 3: auto-commits clean, high-confidence, low-cost proposals
    (safety concerns and high-cost supply orders always escalate)
- **Append-only audit ledger** -- every decision is an immutable log
  entry.
- **langgraph-clj StateGraph** -- one request = one supervised run;
  human-in-the-loop via `interrupt-before`.

### Development

```bash
# Install dependencies (if inside the superproject, use :dev alias for local overrides)
clojure -M:dev -P

# Run tests
clojure -M:test

# Run linter
clojure -M:lint

# Run demo
clojure -M:run
```

### Test suite

- `test/hardwarepaintops/governor_test.cljk` -- unit tests of governor hard
  checks, scope exclusion, and the self-trip regression test
- `test/hardwarepaintops/advisor_test.cljk` -- advisor proposal shape and
  consistency
- `test/hardwarepaintops/phase_test.cljk` -- rollout phase logic
- `test/hardwarepaintops/governor_contract_test.cljk` -- full graph
  integration, audit trail
- `test/hardwarepaintops/store_contract_test.cljk` -- Store protocol and
  MemStore implementation

### Modules

- `hardwarepaintops.store` -- SSoT (MemStore, String-keyed store/vendor
  directories, append-only ledger)
- `hardwarepaintops.advisor` -- contained intelligence node (mock +
  real-LLM seam)
- `hardwarepaintops.governor` -- independent compliance layer
- `hardwarepaintops.phase` -- staged rollout (0→3)
- `hardwarepaintops.operation` -- langgraph-clj StateGraph
- `hardwarepaintops.sim` -- demo driver

## Capability layer

This blueprint resolves its technology stack via
[`kotoba-lang/industry`](https://github.com/kotoba-lang/industry) (ISIC
`4752`).

## Business-process coverage (honest)

| Covered | Not covered (out of scope for this R0) |
|---|---|
| Sales/inventory/tinting-order transaction logging (`:log-sales-record`) | Real POS/inventory-system integration |
| Floor-staff scheduling coordination (`:schedule-staffing-operation`) | Direct staff time-clock/payroll integration |
| Merchandise supply-order coordination with a registered, verified vendor, HARD-gated on vendor verification and a double-actuation-free single-proposal shape (`:coordinate-supply-order`) | Real supplier-ordering-system integration |
| Hazmat-handling-safety-concern flagging, ALWAYS human-gated (`:flag-safety-concern`) | Directly finalizing any hazmat-handling-safety clearance -- permanently out of scope, not a gap |
| Immutable audit ledger for every log/schedule/order/flag decision | Daily reconciliation/cash-up -- a follow-up slice, not in this R0 |

Extending coverage is additive: add the next op (e.g. a return-
authorization or a cash-discrepancy-escalation check) as its own
governed op with its own HARD checks and tests, following the SAME "an
independent governor re-verifies against the actor's own records before
any real-world act" pattern this repo's flagship checks already
establish.

## Maturity

`:implemented` -- `HardwarePaintRetailAdvisor` +
`HardwarePaintRetailGovernor` run as real, tested code (see `Development`
above), following the SAME governed-actor architecture as every prior
actor across this fleet, with its own distinct, independently-named
governor and its own hazmat-handling-safety-clearance scope-exclusion
check.

## License

Code and implementation templates are AGPL-3.0-or-later.
