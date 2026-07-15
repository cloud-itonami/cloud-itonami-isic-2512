# ADR-0001: MetalTankAdvisor ⊣ Metal Tank Plant Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-2512` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-2512` publishes an OSS blueprint for a
metal-tank/reservoir/container fabrication shop's **plant operations
coordination** (production-batch product-category/weight/defect-rate
data logging, welding-line/pressure-test-rig/forming-line maintenance
scheduling, safety-concern flagging, and outbound tank/reservoir/
container shipment coordination). Like every actor in this fleet, the
blueprint alone is not an implementation: this ADR records the
governed-actor architecture that promotes it to real, tested code,
following the same langgraph StateGraph + independent Governor + Phase
0->3 rollout pattern established across the cloud-itonami fleet.

The closest architectural analog is `cloud-itonami-isic-2599`
(Manufacture of other fabricated metal products n.e.c.): both are
back-office coordination actors for a fixed processing PLANT with heavy
manufacturing equipment and a real physical safety dimension, and both
share the same four-op shape (`:log-production-batch`/`:schedule-
maintenance`/`:flag-safety-concern`/`:coordinate-shipment`) and the
same two-entity verified/registered gate structure (equipment for
maintenance scheduling, batch for shipment coordination). The two
verticals are, however, distinct plants with distinct hazard profiles:
2599's central physical hazards are sharp-edge/burr laceration risk
from freshly stamped/pressed sheet metal, stamping-press pinch-point/
crush hazard, and wire-forming-machine entanglement hazard, while
2512's are weld-fume exposure, pressure-test rupture risk at the
pressure-testing line, heavy-plate crush hazard during forming/
handling, confined-space entry hazard (tank/vessel interior inspection
and welding), and hot-work fire risk. This build mirrors 2599's
architecture closely but adapts the hazard profile and equipment/
product vocabulary to the metal-tank/reservoir/container shop: 2512's
permanent equipment-actuation block guards a welding line/pressure-
testing line (`:actuate-welding-line?`) rather than a stamping press
(`:actuate-press-line?`); 2512's production-batch record declares a
`:product-category` (spanning storage tanks, reservoirs, metal
containers, gas cylinders, process vessels, and central-heating
boilers, per ISIC 2512's own scope) rather than 2599's stamped/pressed/
wire-product family; and 2512 adds an ELEVENTH governor check with no
2599 analog: a permanent, unconditional block on any
`:log-production-batch` proposal that declares `:issue-code-stamp?
true` -- a pressure-vessel-certification-authority decision (e.g. an
ASME code stamp) this actor has NO authority to make, since some of
2512's own product family (process vessels) are pressure-bearing
equipment subject to third-party certification this actor must never
claim to grant.

`cloud-itonami-isic-2512` is also distinct from three sibling classes
in the same ISIC 251-259 metal-products group: `cloud-itonami-isic-2511`
(Manufacture of structural metal products -- `:implemented`, a distinct
structural-steel product family), `cloud-itonami-isic-2513`
(Manufacture of steam generators, except central heating hot water
boilers -- `:spec` in this fleet as of this ADR, a distinct boiler/
steam-generator product family upstream of this shop's own tank/
reservoir/container scope), and `cloud-itonami-isic-2591` (Forging,
pressing, stamping and roll-forming of metal -- `:implemented`, a
distinct heavy metal-forming process, not welded-tank fabrication).
ISIC 2512 is deliberately the WELDED tank/reservoir/container class: a
metal-fabrication shop that welds and forms sheet/plate metal into
storage tanks, reservoirs, metal containers, gas cylinders, process
vessels, and central-heating boilers of a kind used for storage or
manufacturing purposes -- a distinct plant, distinct process shape
(welding -> pressure-testing -> forming, not stamping/pressing/wire-
forming, structural fabrication, or steam-generator manufacture), and
this build follows the 2599-style four-op propose-only pattern
specified for this class.

This vertical has NO pre-existing `kotoba-lang/metaltankmfg`-style
capability library to wrap (verified: no such repo exists). This build
therefore uses self-contained domain logic — pure functions in
`metaltankmfg.registry` (equipment/batch verification, shipment-weight
recompute, product-category validation, defect-rate plausibility
validation) are re-verified independently by the governor, the same
"ground truth, not self-report" discipline established across prior
actors (most directly `cloud-itonami-isic-2599`'s
`metalfabmfg.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:metal-tank-plant-operations-governor`, is grep-verified UNIQUE
fleet-wide (`gh search code "metal-tank-plant-operations-governor"
--owner cloud-itonami`, zero hits before this repo was created).

## Decision

### Decision 1: Self-contained domain logic (no external metal-tank/reservoir/container-manufacturing capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
"tanks, reservoirs and containers of metal" vertical has NO pre-existing
capability library to wrap. The equipment/batch-verification /
shipment-weight / product-category / defect-rate validation functions
live as pure functions in `metaltankmfg.registry` and are re-verified
independently by `metaltankmfg.governor` — the same "ground truth, not
self-report" discipline established across prior actors (most directly
`cloud-itonami-isic-2599`'s `metalfabmfg.registry`).

### Decision 2: Coordination, not control or certification — scope boundary at the back-office

This actor is **strictly back-office coordination** of metal-tank/
reservoir/container fabrication shop plant operations. It does NOT:
- Control the welding line, pressure-testing line, or forming line directly
- Make shop-safety or materials-safety decisions (exclusive to the human shop supervisor)
- Actuate the welding line or pressure-testing/forming line
- Act as a pressure-vessel-certification authority (e.g. issue or claim an ASME code stamp) — certification is exclusively a qualified third-party inspector/certification body's authority, never this actor's

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human shop-supervisor
approval. This is not a replacement for the supervisor's authority or
a third-party certifier's authority — it is a proposal-screening and
documentation layer.

**CRITICAL SAFETY BOUNDARY**: metal-tank/reservoir/container fabrication
has real physical hazards (weld-fume exposure, pressure-test rupture
risk, heavy-plate crush hazard, confined-space entry hazard, hot-work
fire risk). Safety-concern flagging NEVER auto-commits. All safety
concerns escalate immediately to human review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (weld-fume exposure, pressure-test rupture risk,
heavy-plate crush hazard, confined-space entry hazard, hot-work fire
risk, equipment-safety concern) ALWAYS escalates, never auto-commits.
This is not a "low-stakes proposal" — it is a circuit-breaker that must
reach human authority.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Like `cloud-itonami-isic-2599`, this vertical has TWO entity kinds each
gating a different op: `:schedule-maintenance` independently verifies
the referenced **equipment** unit's own `:verified?`/`:registered?`
fields; `:coordinate-shipment` independently verifies the referenced
**batch**'s own `:verified?`/`:registered?` fields. Both are the same
"shop/batch record must be independently verified/registered before
any action" HARD invariant applied to the two distinct record kinds
this domain actually has. `:coordinate-shipment` additionally
independently recomputes whether a batch's own recorded shipped-to-date
weight plus the proposal's own claimed weight would exceed the batch's
own recorded production weight — never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override) — two independent permanent scope-boundary blocks, not one

Four HARD governor invariants (elaborated into eleven concrete checks
in `metaltankmfg.governor`, one more than `cloud-itonami-isic-2599`'s
own ten, mirroring 2599's elaboration but adding a second, independent
permanent block) block proposals and cannot be overridden by human
approval:
1. Shop/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's weight must independently recompute within the batch's own logged production weight
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Any proposal touching welding-line-equipment control (`:actuate-welding-line? true`), OR a pressure-vessel-certification-authority decision such as an ASME code stamp (`:issue-code-stamp? true`), is permanently blocked — these are TWO independently-checked scope boundaries (`welding-line-actuate-blocked-violations` and `code-stamp-authority-blocked-violations`), deliberately not folded into one check, because they guard two different kinds of authority this actor must never exercise (equipment control vs. certification authority) and a future change to one check's logic must not silently weaken the other
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Metal-tank/reservoir/container fabrication shop plant operations
back-office now has a documented, governed, auditable coordination
layer that funnels all decisions through independent validation before
human approval.

(+) The "coordination, not control or certification" boundary is
explicit in code: all `:effect :propose`, all real-world actuation
requires human shop-supervisor sign-off, and no code-path can ever
produce a signed/authoritative pressure-vessel certification.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into eleven concrete governor checks) protect against scope creep into
unauthorized equipment operation, welding-line actuation, or
pressure-vessel-certification-authority claims. Safety concerns are a
circuit-breaker, not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.
Human review is mandatory.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation, pressure-testing-line operation,
and pressure-vessel certification remain human/third-party-controlled
via external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch, third-party certification
registries) — this is a standalone coordinator blueprint.

## Verification

- `cloud-itonami-isic-2512`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-weight-exceeded, welding-line-actuate-
  blocked, code-stamp-authority-blocked, already-scheduled, invalid-
  product-category, invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
