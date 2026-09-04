# Strategic deployment plan

**System:** Airline disruption care platform (IRREG cancellation exposure, Crisis Accommodation sourcing, Hotel Call Intelligence)
**Client:** the Carrier (a large EU airline)
**Author:** Gordan Skopljak, AI consulting capstone, Round 2
**Date:** September 2026

> This plan exists to answer one question: what would a carrier actually sign, and in what order. It deliberately proposes a small first step, because the whole business case rests on a single unproven assumption and the honest thing to do is test that assumption before asking for the build budget.

---

## 1. The one thing that decides everything

The ROI model shows a three year return of roughly twenty times. That number is not the point of this plan. The point is that the entire return moves with **one variable: how much of the Carrier's booking base carries usable trip origin data, and at which stations.**

Move capture rate and the whole case moves. Move model quality, licence cost, or hosting, and it barely twitches. Compute for the full year across all three workflows is around 140 US dollars.

So the deployment sequence is built backwards from that: prove the feed, then scale, then extend. Not the reverse.

---

## 2. The three phases

### Phase 0. Proof of concept. Complete.

**Status:** done, and it is what the MVP demonstrates.

Three workflows run end to end on synthetic and public data. The cancellation exposure model reads a manifest, destroys the names, and produces a costed envelope with an authorisation routing decision. The accommodation workflow triages partner hotels and ranks offers in deterministic code. The call intelligence workflow transcribes a hotel call and extracts commercial terms with per section confidence.

**What Phase 0 proved:** the arithmetic is explainable, the model role can be confined to extraction and prose, and the privacy boundary holds under test (the privacy guard has fired in development, which is the point of it).

**What Phase 0 did not prove:** anything about a real carrier's data, real hotel behaviour, or real duty manager adoption. Every number in the business case is still modelled.

### Phase 1. Paid pilot. 8 to 12 weeks. The decision gate.

**Scope:** two or three stations where a trip origin feed already exists. One German hub plus one outstation, so the plan is tested at both ends of the operational spectrum rather than only where conditions are easiest.

**One job:** measure the real capture rate against the 25 percent year one assumption.

Everything else in the pilot is instrumentation to support that measurement. The pilot is not a soft launch and it is not a demo. It exists to produce a number that either justifies the integration spend or kills it cheaply.

**Explicit pilot success criteria.** The pilot passes only if all of the following hold. Partial success is treated as failure of the phase and triggers a re scope, not a rollout.

| # | Criterion | Threshold | How it is measured |
|---|---|---|---|
| 1 | Over provisioning recovered | at least 25 percent of the modelled ceiling at pilot stations | rooms not booked for passengers who went home, against the pre system baseline |
| 2 | Time to a ranked, costed decision | under 40 minutes, from a baseline of about 2h40 | timestamps in the workflow execution record |
| 3 | Passenger identities reaching the language model | zero | audit of model payloads plus privacy guard firing on any attempt |
| 4 | Evidence trail completeness | 100 percent of pilot incidents | every offer, rejection, authorisation and deadline recorded under one incident reference |
| 5 | Crew rest | no crew placed below EASA minimum rest by an accommodation the system ranked usable | crew scheduling reconciliation |
| 6 | Duty manager adoption | at least 80 percent of decisions accepted without overriding the room requirement | override log, plus a structured usability interview with every pilot manager |
| 7 | Care estimate accuracy | within a defined tolerance of actual invoiced cost | reconciliation against the accommodation invoices for pilot incidents |

Criterion 1 is the gate. Criteria 3 and 5 are hard stops: a failure on either halts the pilot rather than adjusting the target.

**What must be true before the pilot starts.** These come directly from the compliance packs and are not negotiable.

- Execution data saving turned off on every workflow, and audio file cleanup verified.
- The Carrier's DPO signs the DPIA for the manifest pipeline and the separate DPIA for the call recording pipeline, including the transfer impact assessment for the US processing.
- Works council agreement in Germany under section 87 BetrVG, scoped to IRROP commercial capture and explicitly excluding employee evaluation.
- Call recording notification script deployed, and the disable recording branch built so a hotel objection can be honoured on the call.
- Zero retention OpenAI endpoint contracted, or an EU hosted alternative deployed.
- Operator briefing delivered to every pilot duty manager, satisfying the Article 4 AI literacy duty.
- One incident reference issued by the exposure workflow and carried into both downstream workflows.

**Cost of Phase 1:** a small fraction of the full build. If capture holds, the rollout is an easy decision with the full case behind it. If it does not, the Carrier has learned the one thing that decides the project for a fraction of the money.

### Phase 2. Scaled rollout. Months 4 to 18.

Triggered only by a passing Phase 1.

- **Integration.** The reservation and departure control feed for trip origin, which is the dominant one time cost and the hard part of the engineering. Nothing else in the build is expensive.
- **Station by station rollout,** sequenced by disruption exposure. The German hubs carry the most delay and the most cancellation cost, so they go first and pay for the rest.
- **Capture path.** 25 percent in year one, 55 in year two, 70 at steady state. These are targets to be measured against, not forecasts to be defended.
- **Vendor and contract work.** Article 28 processing agreements with the accommodation vendor and ground handlers, and a Chapter V transfer mechanism where a vendor processes outside the EEA.
- **Sourcing branch built out.** The accommodation workflow currently suggests and does not send. Outbound dispatch to hotels is Phase 2 work and carries its own review, because a system that contacts a supplier is a different risk posture from one that produces a call list.

### Phase 3. Full deployment and extension. Month 18 onward.

- Network wide coverage, with the care estimate reconciled continuously against invoiced cost so the unit assumptions stop being assumptions.
- Live availability integration to replace nominal partner list room counts.
- The production telephony path for call intelligence, replacing the current upload form.
- Reassessment of the EU AI Act classification before any extension that lets the system decide rather than propose. Annex III high risk obligations attach from 2 December 2027, so any move toward automated commitment must be assessed against that date, not after it.

---

## 3. KPIs after go live

Pilot criteria measure whether to proceed. These measure whether it is working.

**Financial:** over provisioning recovered per station per quarter; care cost per care event against baseline; variance between estimated and invoiced care cost.

**Operational:** median time to a ranked, costed accommodation decision; share of incidents resolved inside the check in window; share of incidents requiring carrier escalation; crew rest breaches attributable to accommodation placement, target zero.

**Trust and quality:** duty manager override rate on the room requirement; extraction values rejected as ungrounded against source text; REVIEW_REQUIRED rate on call extraction and how many of those flags a human confirms as correct.

**Compliance:** privacy guard firings, with any firing investigated rather than dismissed; incidents with a complete evidence trail, target 100 percent; data subject requests received and time to respond.

The trust and quality group matters more than it looks. A system whose override rate climbs is being quietly abandoned even while the financial numbers still look fine.

---

## 4. Stakeholder communication

Each group needs a different first sentence, and getting that wrong is how good systems fail internally.

| Stakeholder | What they need to hear first | What they will object to |
|---|---|---|
| Station duty managers | This does not replace your judgement, it gives you a costed answer and permission | Being measured. Address it directly: no agent level performance dashboards, and that is a documented restriction, not a promise |
| Carrier operations centre | The authorisation request arrives complete and costed instead of as a phone call | Trusting a number they did not calculate. Show them the working, which the result page already does |
| Finance | Care is a legal obligation and this does not reduce it. It recovers the over spend on top of it | Any suggestion that statutory cost can be cut. Never make that claim |
| Data protection officer | Two pipelines, two different postures, both documented, with go live gates named | The call recording branch. Bring the transfer impact assessment before being asked |
| Works council | Scoped to IRROP commercial capture, explicitly not employee evaluation | Voice recording of employees. This is settled in agreement, not by policy statement |
| IT and integration | The whole value depends on one feed you already own | Being handed an integration with no proof it is worth it. That is precisely what the pilot provides |
| Crew scheduling | Rest before the morning departure is a ranking constraint, not an afterthought | Nothing, if the constraint holds. Everything, if it does not |

Cadence during the pilot: a weekly operational review with the pilot station managers, a monthly steering update to Finance and the operations centre, and a compliance checkpoint at the midpoint and at close.

---

## 5. Commercialisation model

Three viable structures. The recommendation is the second.

**Licence plus integration.** Annual platform licence with a one time integration fee. Predictable for the Carrier, easy to budget, and it puts all the risk on the buyer before the value is proven. Appropriate only after a passing pilot.

**Paid pilot, then licence. Recommended.** A fixed fee pilot that is small relative to the build, followed by a licence and integration contract if the capture criteria are met. The pilot fee is credited against the licence if the Carrier proceeds. This aligns both sides on the only question that matters and is the standard shape for a consulting engagement where the value hypothesis is testable.

**Gainshare on recovered over provisioning.** Superficially attractive and operationally hard. It requires an agreed baseline, a shared measurement method, and trust in an attribution model, and it creates a perverse incentive to define recovery generously. Offer it only if the Carrier asks, and only with the measurement method agreed in the contract before the first euro is counted.

Why not per incident pricing: compute cost is around 140 US dollars a year for the entire platform, so usage based pricing would misrepresent where the value and the cost actually sit.

---

## 6. What would stop this

Named honestly, because a plan that lists no failure conditions is a sales document.

- **Trip origin coverage is low or absent at most stations.** This is the primary risk and the pilot exists to detect it early and cheaply.
- **The works council declines the call recording scope.** Workflows 1 and 2 proceed without it. Workflow 3 does not. The system is still valuable without the call branch, which is why the phases are separable.
- **Duty managers do not adopt it.** Overrides climb, the estimate is ignored, and the recovery never materialises even though the software works. Mitigated by keeping the human decision structural rather than advisory, and by showing the working on every output.
- **The Carrier wants automated booking.** That is a different system with a different EU AI Act classification, and this plan does not cover it. Any move in that direction reopens the assessment before it reopens the roadmap.

---

## 7. The recommendation in one line

Do not fund the integration. Fund a three station pilot for eight to twelve weeks with one job, measuring the real capture rate, because the software is cheap, the return is large, and both depend entirely on one feed of data the Carrier already owns and does not yet use at the station.

---

*All figures rest on the American Airlines 2015 analog with modelled unit costs, described in the ROI and risk assessment. Compliance prerequisites are drawn from the EU AI Act classification and the GDPR pack, including both hotel call addenda.*
