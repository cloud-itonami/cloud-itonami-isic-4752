# Business Model: Hardware, Paint and Glass Specialty Retail Operations Coordination

## Classification
- Repository: `cloud-itonami-isic-4752`
- ISIC Rev.5: `4752` -- retail sale of hardware, paints and glass in
  specialized stores (hardware stores, paint/coatings specialty
  retailers, glass retailers; distinct from ISIC 4719's general-
  merchandise retail)
- Social impact: local economy, consumer protection, transparency

## Customer
- independent hardware/paint/glass specialty stores needing an
  auditable operations-coordination platform
- multi-store operators needing consistent staffing/supply-order/
  hazmat-handling-safety governance across sites
- programs that cannot accept closed, unauditable back-office platforms

## Offer
- sales/inventory/tinting-order transaction logging
- floor-staff scheduling coordination
- merchandise supply-order coordination with registered, verified vendors
- hazmat-handling-safety-concern flagging (solvent/aerosol/flammable-
  storage, spill, ventilation observations) for human triage
- role-based access and immutable audit ledger

## Revenue
- self-host setup fee
- managed hosting subscription per store
- support retainer with SLA

## Trust Controls
- `:hardware-paint-retail-governor` never lets a proposal for an
  unregistered/unverified store, or a supply order naming an
  unregistered/unverified vendor, commit or even escalate
- every proposal's `:effect` must be `:propose` -- a claim to directly
  actuate is a HARD, un-overridable block
- directly finalizing a hazmat-handling-safety clearance (certifying
  storage compliance, clearing a spill as contained, signing off on a
  handling permit) is permanently out of scope, not a rollout milestone
  -- the actor may only flag a concern for a human
- a `:flag-safety-concern` proposal, and a high-cost `:coordinate-
  supply-order`, always require human sign-off
- sensitive customer, employee and supplier data stays outside Git
