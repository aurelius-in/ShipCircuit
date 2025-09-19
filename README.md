# ShipCircuit

**Agentic logistics control tower** that plans, routes, and protects service from booking to delivery with auditable decisions.

ShipCircuit coordinates specialized AI agents across forecasting, labor, yard, sort, linehaul, cross-border, last mile, claims, and account health. Every action creates a signed receipt so operations can automate safely.

---

## Why ShipCircuit

* Networks are volatile: weather, ATC holds, traffic, staffing, vendor slips
* Ops leaders need measurable lifts: on-time rate, cost per stop, overtime percent, refund leakage
* Compliance needs proof: who approved what, which policy version, what evidence

ShipCircuit gives you goal-seeking, tool-using agents with policy-as-code and immutable receipts.

---

## Network-Wide Impact (Bottom Line)

| Company   | Projected Yearly Savings (USD) if adopted |
| --------- | ----------------------------------------: |
| **UPS**   |                       **\$2.0B – \$4.7B** |
| **FedEx** |                       **\$1.4B – \$3.3B** |

### How we sized it (succinct)

* **Unit of rollout:** “Hub slice” (one hub + \~2 stations + a few lanes).
* **Per-slice run-rate (provenable in pilot):** \~\$10.5M (conservative) to \~\$25M+ (upside).
* **Network scale proxy:** Daily package volume ÷ 120k packages/slice.

  * UPS ≈ 22.4M/day → \~187 slices.
  * FedEx ≈ 16.0M/day → \~133 slices.
* **Roll-up:** `Total = Per-slice $ × Slice count`

  * UPS: 187 × (\$10.5M → \$25M) ≈ **\$2.0B → \$4.7B**.
  * FedEx: 133 × (\$10.5M → \$25M) ≈ **\$1.4B → \$3.3B**.

> Assumptions mirror pilot wins across core levers: on-time +150–400 bps, hub dwell −10–20%, misload/damage −15–30%, overtime −10–15%, cost/stop −3–7%, refund leakage −20–35%, claims TAT −30–50%. Tighten with site data after Week 2.

---

## What ShipCircuit does

* **Plan the day** from bookings and history, turn demand into staffed minutes and dock doors
* **Run the sort** while vision agents watch for jams, damage, PPE misses, and hazmat issues
* **Route linehaul** across road, rail, air, and propose partner handoffs when lanes fail
* **Clear customs** with HS classification, document packs, and denied-party checks
* **Optimize last mile** for SLA, traffic, curb rules, EV range, and opportunistic pickups
* **Communicate ETAs** and save at-risk stops with pickup point or neighbor authorization
* **Settle claims** fast with scan, GPS, and photo evidence
* **Close the loop** with account QBRs and signed automation receipts

---

## Core capabilities

* **Agents**: Forecast, Labor Orchestrator, Dock and Yard, ULD Builder, Slotting and Pick Path, Vision Exception Hunter, Dangerous Goods Guardian, Linehaul Router, Partner Evaluator, Customs Packager, Sanctions Checker, Trade and Tariff Simulator, EV Fleet Guardian, Last-Mile Route Agent, Pickup Density Builder, Driver Copilot, ETA Promise Keeper, Claims and Refund Arbiter, Returns Profitability, Contract and Accessorial Copilot, Key-Account QBR, Data Quality Sheriff, Trust and Policy Guard, API Copilot for shippers
* **Receipts**: Every consequential action produces a signed JSON receipt with inputs, policy version, approvals, and outcomes
* **Policy-as-code**: DG rules, customs, interline, spend caps, safety, and privacy enforced in code with testable policies
* **Event bus**: Agents publish and subscribe to typed topics. Everything is observable

---

## Architecture overview

```
Bookings/API  Cameras  Telematics  WMS/TMS  Weather/ATC  HR Rosters
     |           |        |          |           |          |
     v           v        v          v           v          v
                 [ Connectors + Ingest ]  ->  [ Event Bus ]
                                   |                  |
                             [ Data Contracts ]       |
                                   |                  |
       ---------------------------------------------------------------
       |                Orchestrated Agent Mesh                      |
       | Forecast | Labor | Yard | ULD | Slotting | Vision | Router  |
       | Customs | Sanctions | Tariff | Last Mile | ETA | Claims     |
       ---------------------------------------------------------------
                                   |
                           [ Trust and Policy Guard ]
                                   |
                            [ Signed Receipts Store ]
                                   |
                         Dashboards, QBRs, Audits, Exports
```

* **Event bus**: Kafka, Pulsar, or NATS
* **Receipts store**: append-only database or object storage with hash chain
* **Policy engine**: OPA or custom rules VM
* **Feature store**: optional for model features
* **Serving**: gRPC and REST for synchronous asks

---

## Key event topics

* `forecast.plan.v1`, `labor.plan.v1`, `yard.assign.v1`, `uld.plan.v1`
* `sort.wave.v1`, `vision.incident.v1`, `dg.check.v1`
* `linehaul.plan.v1`, `partner.proposal.v1`
* `customs.packet.v1`, `sanctions.check.v1`, `tariff.sim.v1`
* `route.plan.v1`, `eta.update.v1`, `pickup.opportunity.v1`
* `claim.case.v1`, `returns.plan.v1`, `contract.audit.v1`
* `policy.receipt.v1`, `dq.alert.v1`

---

## Receipt schema (core)

```json
{
  "receipt_id": "rcpt_2025_09_19_ABC123",
  "timestamp": "2025-09-19T02:30:12Z",
  "actor": {
    "type": "agent",
    "name": "linehaul_router",
    "version": "1.8.2"
  },
  "action": "propose_linehaul_change",
  "inputs_ref": ["evt_linehaul_plan_...","weather_alert_...","capacity_snapshot_..."],
  "policy": {
    "bundle": "interline_rules_v5",
    "decision": "requires_approval",
    "constraints": ["spend_delta_lt_15pct","safety_ok","dg_not_present"]
  },
  "proposal": {
    "before": {"mode":"air","eta":"2025-09-19T16:10Z","cost_usd": 18400},
    "after": {"mode":"road","eta":"2025-09-19T19:30Z","cost_usd": 8200},
    "kpis": {"sla_risk_delta_bp": 22, "refund_exposure_usd": -5200}
  },
  "approvals": [
    {"role":"duty_manager","user_id":"u_456","status":"approved","at":"2025-09-19T02:31:04Z"}
  ],
  "hash": "sha256:...",
  "prev_hash": "sha256:..."
}
```

---

## Minimal APIs

### Plan of day

```
POST /api/plan
Body: { date, stations[], constraints[] }
Resp: { plan_id, labor_plan_ref, yard_assign_ref, risks[] }
```

### Route plan

```
POST /api/route/plan
Body: { station_id, vehicle_set, stops[], constraints }
Resp: { plan_id, eta_baseline, risk_hotspots[], receipt_ref }
```

### Customs packet

```
POST /api/customs/packet
Body: { shipment_id, items[], parties, incoterms }
Resp: { packet_id, hs_details[], sanctions_ok, docs[], receipt_ref }
```

### Claims triage

```
POST /api/claims/triage
Body: { claim_id, scans[], gps[], photos[] }
Resp: { recommended_outcome, confidence, precedent_refs[], receipt_ref }
```

---

## Data contracts (examples)

```json
// event: vision.incident.v1
{
  "incident_id": "inc_7f9",
  "type": "damage|jam|ppe_miss|misload",
  "source": "camera_23",
  "station_id": "STN-DFW-03",
  "proof_uri": "s3://.../clip.mp4",
  "severity": "low|med|high",
  "detected_at": "2025-09-19T02:05:12Z"
}
```

```json
// event: route.plan.v1
{
  "plan_id": "rte_9012",
  "station_id": "STN-DFW-08",
  "vehicle_ids": ["V_144","V_145"],
  "stops": [{"id":"S123","sla":"2025-09-19T17:00Z","lat":32.9,"lon":-96.9}],
  "constraints": {"school_zone": true,"curb_rules": true,"ev_min_soc": 0.2}
}
```

---

## KPIs and targets

* **On-time performance**: +150 to +400 basis points
* **Hub dwell**: −10 to −20 percent
* **Misload and damage rate**: −20 percent
* **Overtime percent**: −10 to −15 percent
* **Refund leakage**: −25 percent
* **Customs clearance time**: −15 percent on pilot lanes
* **Claim cycle time**: −30 percent to first decision
* **EV utilization**: +10 percent without SLA loss

(Exact targets set per lane and station during pilot.)

---

## 30-day pilot plan

**Scope**

* One hub, two delivery stations, one cross-border lane

**Week 1**

* Connect WMS/TMS feeds, scans, weather/ATC, roster export
* Enable Forecast, Labor, Dock and Yard, Trust and Policy Guard
* Shadow receipts only, no auto actions

**Week 2**

* Turn on Vision incidents and Slotting and Pick Path in assisted mode
* Start Customs Packager on the single lane
* Begin ETA Promise Keeper with opt-in customers

**Week 3**

* Enable Last-Mile Route Agent and EV Fleet Guardian for two routes
* Turn on Claims triage for a subset of claim types
* Daily receipts review, adjust policies

**Week 4**

* Graduate stable actions to auto within spend and safety caps
* Exec readout: KPI deltas, receipts log, next-site rollout plan

---

## Security and compliance

* **PII and PHI**: field-level encryption, tokenization where possible
* **Least privilege**: per-agent service accounts, scoped topics
* **Immutable receipts**: append-only with hash chain and off-site replication
* **Policy governance**: code-reviewed bundles with version pins, unit tests
* **Audit**: exportable receipts pack with evidence links and policy states

---

## Integrations

* **Ops**: Manhattan, Blue Yonder, Körber, custom WMS and TMS adapters
* **Vision**: ONVIF RTSP, edge inference on Jetson or CPU
* **Maps and routing**: OSRM, Valhalla, commercial SDKs
* **Messaging**: Twilio, SendGrid, WhatsApp, push providers
* **Customs**: AES/EEI, tariff databases, denied-party lists
* **Data**: Snowflake, BigQuery, Redshift, S3, ADLS

---

## Use Cases

* Protect on-time performance during weather and ATC events
* Reduce overtime with smarter shift and wave planning
* Prevent misloads and reduce damage with vision tickets and proof clips
* Clear customs faster on pilot lanes and reduce holds
* Save at-risk deliveries with live re-promises and pickup point redirects
* Cut refund leakage by standardizing evidence and decisions

---

## User Stories

* **Hub supervisor**: I want a single Plan of Day with the top risks and an approve button so I can focus my team on the few things that matter.
* **Yard lead**: I want door and yard assignments with alerts when a re-spot will protect a downstream SLA.
* **Customs analyst**: I want HS codes and documents prebuilt with flags for missing data so I can clear shipments faster.
* **Route manager**: I want routes that respect school zones and curb rules and can accept opportunistic pickups without breaking promises.
* **Claims specialist**: I want evidence assembled automatically so I can settle fairly in minutes.
* **Account manager**: I want QBR packs that show wins, misses, and next experiments with receipts.

---

## Operating model

* **Three modes**: shadow, assisted, auto with spend and safety caps
* **Graduation rule**: action graduates to auto after N accepted assisted decisions with KPI improvement and zero safety incidents
* **Change control**: policies and models ship with receipts and rollback plans

---

## Roadmap

* Dynamic interline auctions with real-time rate proofs
* Return consolidation across merchants to shared nodes
* Work instruction copilot that adapts to station-level takt
* Active learning loops on vision incidents with human labeling shifts
* Fine-grained carbon accounting per leg and customer

---

## Branding

**Name**: ShipCircuit
**One-liner**: Control tower agents that keep freight on time and customers informed, with receipts for every decision.

---

## License

TBD. Recommend Apache-2.0 for SDKs, commercial license for orchestrator.
