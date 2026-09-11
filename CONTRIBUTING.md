# Contributing

`cloud-itonami-isic-4752` accepts contributions to the OSS blueprint,
capability bindings, policy tests, documentation and operator model.

## Development

```bash
kbb -M:test
kbb -M:lint
```

## Rules
- Do not commit real customer, employee, supplier or hazmat-handling-
  safety-incident data.
- Keep sales-record logging, staffing-operation scheduling, supply-order
  coordination and hazmat-handling-safety-concern flagging behind the
  HardwarePaintRetailGovernor.
- Treat hardware/paint/glass-store-operations workflows as high-risk: add
  tests for store/vendor verification, effect discipline, scope
  exclusion, escalation and audit logging.
- Never phrase a governor scope-exclusion term as a bare noun (e.g.
  "hazmat", "spill", "solvent") -- phrase it as the finalization/
  execution ACTION (e.g. "certified the storage area as compliant"), and
  add/extend the
  `default-mock-advisor-proposals-never-self-trip-scope-exclusion`
  regression test for any new term. A bare-noun term will self-trip this
  actor's own legitimate `:flag-safety-concern` happy path -- see
  `hardwarepaintops.governor/scope-excluded-terms`'s docstring.
- Never add an op that directly finalizes a hazmat-handling-safety
  clearance to the closed proposal-op allowlist -- that action is
  structurally excluded from this actor's vocabulary, not merely gated.
- Document any new business-model or operator assumption in `docs/`.

## Pull Requests
PRs should describe: what behavior changed, which policy invariant is
affected, how it was tested, whether operator or certification docs need
updates.
