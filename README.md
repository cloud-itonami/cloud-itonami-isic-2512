# cloud-itonami-isic-2512: Manufacture of tanks, reservoirs and containers of metal

Open Business Blueprint for **ISIC Rev.5 2512**: manufacture of tanks, reservoirs and containers of metal — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office **metal-tank/reservoir/container fabrication shop plant operations**: production-batch data logging (product-category/weight/defect-rate), welding-line/pressure-test-rig/forming-line maintenance scheduling, safety-concern flagging, and outbound tank/reservoir/container shipment coordination.

This repository designs a forkable OSS business for metal-tank/
reservoir/container fabrication shop plant operations: run by a
qualified operator so a shop keeps its own operating records instead of
renting a closed SaaS.

## Scope: the welded-tank/reservoir/container shop, not steam generators, forging, or structural steel

ISIC 2512 covers the metal-fabrication shop that welds and forms sheet/
plate metal into storage tanks, reservoirs, metal containers, gas
cylinders, process vessels, and central-heating boilers of a kind used
for storage or manufacturing purposes. This is distinct from
`cloud-itonami-isic-2513` (Manufacture of steam generators, except
central heating hot water boilers), a distinct boiler/steam-generator
product family; from `cloud-itonami-isic-2591` (Forging, pressing,
stamping and roll-forming of metal), a distinct heavy metal-forming
process, not welded-tank fabrication; and from `cloud-itonami-isic-2511`
(Manufacture of structural metal products), a distinct structural-steel
product family. This actor's own hazard profile centers on weld-fume
exposure, pressure-test rupture risk at the pressure-testing line,
heavy-plate crush hazard during forming/handling, confined-space entry
hazard (tank/vessel interior inspection and welding), and hot-work fire
risk.

## What this actor does

Proposes **plant operations coordination**, not equipment operation or
certification:
- `:log-production-batch` — product-category/weight/defect-rate data logging (administrative, not an operational decision)
- `:schedule-maintenance` — welding-line/pressure-test-rig/forming-line maintenance scheduling proposal
- `:flag-safety-concern` — surface a weld-fume-exposure/pressure-test-rupture-risk/heavy-plate-crush-hazard/confined-space-entry/hot-work-fire-risk/equipment-safety concern (always escalates)
- `:coordinate-shipment` — outbound tank/reservoir/container shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY — this is a safety-relevant domain**
(pressure-test rupture risk, weld-fume exposure, heavy-plate crush
hazard, confined-space entry hazard, hot-work fire risk):

- Does NOT control the welding line, pressure-testing line, or forming line directly
- Does NOT make shop-safety or materials-safety decisions (that's the shop supervisor's exclusive human authority)
- Does NOT actuate the welding line or pressure-testing/forming line (human shop supervisor decides)
- Is NOT a pressure-vessel-certification authority — it never issues, claims, or proposes an ASME code stamp or any other pressure-vessel certification decision. Certification is exclusively a qualified third-party inspector/certification body's authority, never this actor's.
- ONLY proposes/coordinates operations back-office; all actuation and certification require explicit human/third-party authority outside this actor
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`metaltankmfg.operation/build`, a langgraph-clj StateGraph):
1. **`metaltankmfg.advisor`** (sealed intelligence node, `MetalTankAdvisor`): proposes decisions only, never commits
2. **`metaltankmfg.governor`** (independent, `Metal Tank Plant Operations Governor`): validates against domain rules, re-derived from `metaltankmfg.registry`'s pure functions and `metaltankmfg.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Shop/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct welding-line-equipment control)
     - Any proposal touching welding-line-equipment control is a hard, permanent block (`:actuate-welding-line? true` on a `:schedule-maintenance` proposal)
     - Any proposal touching a pressure-vessel-certification-authority decision (e.g. an ASME code stamp) is a hard, permanent block (`:issue-code-stamp? true` on a `:log-production-batch` proposal)
     - A shipment may not push a batch's own recorded shipped weight past its own logged production weight (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:product-category` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`metaltankmfg.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`metaltankmfg.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
