(ns hardwarepaintops.governor
  "HardwarePaintRetailGovernor -- the independent compliance layer that
  earns the HardwarePaintRetailAdvisor the right to commit. The advisor
  has no notion of whether a store is actually registered and
  license-verified, whether a named supply-order vendor is itself a
  registered/verified counterparty, whether its own proposed `:effect`
  secretly claims a direct actuation instead of a mere proposal, or
  whether it has silently drifted into a permanently out-of-scope
  decision area, so this MUST be a separate system able to *reject* a
  proposal and fall back to HOLD.

  This actor's scope is deliberately narrow -- COORDINATION ONLY
  (sales/inventory/tinting-order transaction logging, floor-staff
  scheduling, hardware/paint/glass merchandise supply-order coordination,
  hazmat-handling-safety-concern flagging). It NEVER performs or
  authorizes:
    - setting or overriding a shelf/unit price
    - directly finalizing a hazmat-handling-safety clearance (certifying
      a solvent/aerosol/flammable-storage area as code-compliant,
      clearing a spill as contained, signing off on a hazmat handling
      permit, issuing a flammable-storage compliance certificate)
    - hazmat-handling-safety authority enforcement (declaring a tinting
      room ventilation compliant, authorizing hazardous waste disposal
      as compliant)

  Four HARD checks, ALL permanent, un-overridable by any human approval:

    1. Store unverified           -- the target store record must exist
                                     AND be independently confirmed
                                     `:registered?`/`:verified?` in the
                                     store before ANY proposal for it may
                                     commit or even escalate. Never trusts
                                     a proposal's own claim about the
                                     store -- re-derived from the store's
                                     own record, the same 'ground truth,
                                     not self-report' discipline every
                                     sibling actor's governor uses.
    2. Vendor unverified          -- for `:coordinate-supply-order` ONLY,
                                     the proposal's own drafted `:value`
                                     must name a `:vendor-id` that
                                     resolves to an independently
                                     `:registered?`/`:verified?` vendor
                                     record. A missing vendor-id, or one
                                     that resolves to an unregistered or
                                     unverified vendor, is a HARD block.
    3. Effect not :propose        -- every proposal's `:effect` MUST be
                                     `:propose`. Any other effect value
                                     is, by construction, a claim to
                                     directly actuate/commit outside
                                     governance -- HARD block, not merely
                                     low-confidence.
    4. Scope exclusion            -- ANY proposal (regardless of op)
                                     whose op, summary, rationale, cites
                                     or draft value touches directly
                                     finalizing a hazmat-handling-safety
                                     clearance (certifying storage
                                     compliance, clearing a spill as
                                     contained/safe, signing off on a
                                     hazmat handling permit, issuing a
                                     flammable-storage compliance
                                     certificate) is a HARD, PERMANENT
                                     block -- this actor's charter
                                     excludes that territory structurally,
                                     not as a rollout milestone. Evaluated
                                     UNCONDITIONALLY on every proposal. An
                                     op outside the closed four-op
                                     allowlist is the SAME failure mode
                                     (an advisor proposing something it
                                     was never authorized to propose) and
                                     is folded into this same check.
                                     `:flag-safety-concern` itself is
                                     never excluded by this check --
                                     surfacing a hazmat-handling/spill/
                                     storage concern for a human is
                                     exactly this actor's job; only
                                     FINALIZING/certifying/clearing that
                                     concern is excluded (see
                                     `scope-excluded-terms` below --
                                     phrased as the finalization/
                                     execution ACTION, never a bare noun
                                     like 'spill' or 'hazmat', so the
                                     default mock advisor's own
                                     `:flag-safety-concern` rationale
                                     never self-trips this check).

  Two ESCALATE (SOFT) gates, either forces human sign-off:
    - LLM confidence below the floor.
    - The op is `:flag-safety-concern` -- ALWAYS escalates to a human,
      regardless of confidence, regardless of how clean the proposal
      otherwise is. `hardwarepaintops.phase` independently agrees:
      `:flag-safety-concern` is never a member of any phase's `:auto`
      set either -- two layers, not one. This actor NEVER directly
      finalizes a hazmat-handling-safety clearance itself -- flagging a
      concern always routes to a human, never to auto-commit.
    - A `:coordinate-supply-order` whose drafted `:value` names an
      `:estimated-cost` above `supply-cost-threshold` -- a large-value
      merchandise procurement proposal always needs a human sign-off,
      even when the governor and phase would otherwise allow
      auto-commit."
  (:require [kotoba.lang.text :as str]
            [hardwarepaintops.store :as store]))

(def confidence-floor 0.6)

(def supply-cost-threshold
  "Example single-store hardware/paint/glass merchandise-procurement
  threshold (USD-equivalent units, domain-illustrative -- not a universal
  cross-domain constant). A `:coordinate-supply-order` proposal citing an
  `:estimated-cost` above this value ALWAYS escalates to human sign-off,
  regardless of confidence or rollout phase."
  1000.0)

(def allowed-ops
  "The closed proposal-op allowlist -- an op outside this set is a scope
  violation by construction (see `scope-exclusion-violations`). NOTE: no
  op in this allowlist finalizes a hazmat-handling-safety clearance --
  that action is structurally excluded from the actor's vocabulary, not
  merely gated."
  #{:log-sales-record :schedule-staffing-operation
    :coordinate-supply-order :flag-safety-concern})

(def always-escalate-ops
  "Ops that ALWAYS require human sign-off, clean or not."
  #{:flag-safety-concern})

(def scope-excluded-terms
  "Case-insensitive substrings that mark a proposal as touching a
  permanently out-of-scope decision area -- directly finalizing a
  hazmat-handling-safety clearance (certifying storage as compliant,
  clearing a spill as contained, signing off on a handling permit,
  issuing a compliance certificate) or otherwise finalizing/certifying a
  hazmat-safety matter rather than merely flagging it for a human.
  Scanned across the proposal's op/summary/rationale/cites/value, never
  trusting the advisor's own framing of its intent.

  CRITICAL: every term here is phrased as the finalization/execution
  ACTION (e.g. 'certified the storage area as compliant', 'cleared the
  spill as contained'), never a bare noun like 'hazmat', 'spill',
  'solvent' or 'flammable' -- a bare noun would accidentally match inside
  this actor's own legitimate `:flag-safety-concern` default proposal
  text (whose whole job is to talk about solvent/aerosol/flammable-
  storage/spill/ventilation concerns, and whose own printed `:op` keyword
  literally contains the substring 'safety') and self-block the happy
  path. See
  `hardwarepaintops.governor-test/default-mock-advisor-proposals-never-self-trip-scope-exclusion`
  for the regression test."
  ["finalize the hazmat clearance" "finalized the hazmat clearance" "finalizing the hazmat clearance"
   "finalize the hazmat handling clearance" "finalized the hazmat handling clearance"
   "certify the storage area as compliant" "certified the storage area as compliant" "certifying the storage area as compliant"
   "certify the storage as code-compliant" "certified the storage as code-compliant"
   "clear the spill as contained" "cleared the spill as contained" "clearing the spill as contained"
   "clear the spill as safe" "cleared the spill as safe" "clearing the spill as safe"
   "sign off on the hazmat clearance" "signed off on the hazmat clearance" "signing off on the hazmat clearance"
   "sign off on the hazmat handling permit" "signed off on the hazmat handling permit"
   "approve the hazardous materials storage permit" "approved the hazardous materials storage permit" "approving the hazardous materials storage permit"
   "issue the flammable-storage compliance certificate" "issued the flammable-storage compliance certificate" "issuing the flammable-storage compliance certificate"
   "declare the tinting room ventilation compliant" "declared the tinting room ventilation compliant" "declaring the tinting room ventilation compliant"
   "authorize the solvent disposal as compliant" "authorized the solvent disposal as compliant" "authorizing the solvent disposal as compliant"
   "finalize the spill containment clearance" "finalized the spill containment clearance" "finalizing the spill containment clearance"
   "confirm the aerosol storage as code-compliant" "confirmed the aerosol storage as code-compliant" "confirming the aerosol storage as code-compliant"
   "certify the site as hazmat-compliant" "certified the site as hazmat-compliant"
   "grant the hazmat handling clearance" "granted the hazmat handling clearance" "granting the hazmat handling clearance"
   "危険物取扱いの許可を確定" "危険物取扱いの許可を確定した" "危険物取扱いの許可を確定する"
   "保管区画の適合証明を発行" "保管区画の適合証明を発行した" "保管区画の適合証明を発行する"
   "漏出物の安全確認を完了" "漏出物の安全確認を完了した" "漏出物の安全確認を完了する"
   "消防法適合証明を発行" "消防法適合証明を発行した" "消防法適合証明を発行する"]
  )

;; ----------------------------- checks -----------------------------

(defn- store-unverified-violations
  "The target store must exist AND be independently
  `:registered?`/`:verified?` in the store -- never trust the proposal's
  own `:store-id` claim without a store lookup."
  [{:keys [store-id]} st]
  (let [s (store/store-record st store-id)]
    (when-not (and s (:registered? s) (:verified? s))
      [{:rule :store-unverified
        :detail (str store-id " は未登録または未検証の店舗 -- いかなる提案も進められない")}])))

(defn- vendor-unverified-violations
  "For `:coordinate-supply-order` ONLY, the proposal's own drafted
  `:value` must name a `:vendor-id` that resolves to an independently
  `:registered?`/`:verified?` vendor record. A missing vendor-id, or one
  that resolves to an unregistered/unverified vendor, is a HARD block --
  never trust the proposal's own vendor claim without a store lookup, the
  SAME 'ground truth, not self-report' discipline as
  `store-unverified-violations`, reapplied to the supply-chain
  counterparty."
  [proposal st]
  (when (= :coordinate-supply-order (:op proposal))
    (let [vendor-id (get-in proposal [:value :vendor-id])
          v (and vendor-id (store/vendor-record st vendor-id))]
      (when-not (and v (:registered? v) (:verified? v))
        [{:rule :vendor-unverified
          :detail (str (or vendor-id "(vendor-id missing)")
                        " は未登録または未検証の仕入先 -- 発注調整提案を進められない")}]))))

(defn- effect-not-propose-violations
  "`:effect` must ALWAYS be `:propose` -- any other value is a claim to
  directly actuate/commit outside governance."
  [proposal]
  (when (not= :propose (:effect proposal))
    [{:rule :effect-not-propose
      :detail (str ":effect は :propose のみ許可されるが " (pr-str (:effect proposal)) " が提案された")}]))

(defn- text-blob
  "Flatten every advisor-authored field on a proposal into one lower-cased
  blob the scope-exclusion scan checks."
  [proposal]
  (str/lower (pr-str (select-keys proposal [:op :summary :rationale :cites :value]))))

(defn- scope-exclusion-violations
  "HARD, PERMANENT block: a proposal outside the closed op allowlist, or
  one whose content touches directly finalizing a hazmat-handling-safety
  clearance (certifying storage compliance, clearing a spill as
  contained, signing off on a handling permit, issuing a compliance
  certificate), regardless of confidence or how clean every other check
  is. Evaluated UNCONDITIONALLY on every proposal."
  [proposal]
  (let [op (:op proposal)
        blob (text-blob proposal)]
    (cond
      (not (contains? allowed-ops op))
      [{:rule :op-not-allowed
        :detail (str (pr-str op) " は許可された操作(closed allowlist)に含まれない")}]

      (some #(str/includes? blob %) scope-excluded-terms)
      [{:rule :scope-excluded
        :detail "危険物取扱い安全許可の確定行為(hazmat-handling-safety-clearance finalization)に触れる提案は永久に禁止"}])))

(defn- high-cost-supply-order?
  "A `:coordinate-supply-order` proposal citing an `:estimated-cost` above
  `supply-cost-threshold` -- always needs human sign-off (SOFT escalate,
  not a hard block: the order itself is in scope, only its size requires
  a human)."
  [proposal]
  (and (= :coordinate-supply-order (:op proposal))
       (some-> proposal :value :estimated-cost (> supply-cost-threshold))))

(defn check
  "Censors a HardwarePaintRetailAdvisor proposal against the governor
  rules. Returns {:ok? bool :violations [..] :confidence c :escalate?
  bool :high-stakes? bool :hard? bool}."
  [request _context proposal store]
  (let [store-id (or (:store-id proposal) (:store-id request))
        hard (into []
                   (concat (store-unverified-violations {:store-id store-id} store)
                           (vendor-unverified-violations proposal store)
                           (effect-not-propose-violations proposal)
                           (scope-exclusion-violations proposal)))
        conf (:confidence proposal 0.0)
        low? (< conf confidence-floor)
        stakes? (boolean (or (always-escalate-ops (:op proposal))
                              (high-cost-supply-order? proposal)))
        hard? (boolean (seq hard))]
    {:ok?          (and (not hard?) (not low?) (not stakes?))
     :violations   hard
     :confidence   conf
     :hard?        hard?
     :escalate?    (and (not hard?) (or low? stakes?))
     :high-stakes? stakes?}))

(defn hold-fact
  "The audit fact written when a proposal is rejected (HOLD)."
  [request context verdict]
  {:t          :governor-hold
   :op         (:op request)
   :actor      (:actor-id context)
   :store-id   (:store-id request)
   :disposition :hold
   :basis      (mapv :rule (:violations verdict))
   :violations (:violations verdict)
   :confidence (:confidence verdict)})
