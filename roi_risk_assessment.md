# ROI and risk assessment

**System:** Airline disruption care platform (IRREG cancellation exposure model plus Crisis Accommodation sourcing)
**Client:** the Carrier (a large airline)
**Author:** Gordan Skopljak, AI consulting capstone, Round 2
**Basis:** real American Airlines 2015 cancellations (US DOT), overnight care subset, costs modelled

> All euro figures are built on the American Airlines 2015 analog used for the dashboard: 3,305 overnight care events from 10,919 real cancellations. A real engagement re-runs this model on the carrier's own invoices and booking data. The point of this document is the structure of the return and where the risk actually sits, not the second decimal place.

---

## 1. The one honest line before any number

This system does not make the care bill smaller and it does not make the compensation smaller. Both are legal obligations: Article 9 care has no ceiling, and Article 7 compensation is owed when it is owed. Anyone who pitches an AI that "cuts your EU261 costs" is either wrong or planning to under-provide care, which is unlawful. The value here is narrower and real: it recovers the money the airline **over-spends** by booking rooms for passengers who were going home anyway, it collapses the time to secure those rooms, and it produces the evidence trail a disputed claim needs. Everything below is measured against that, and only that.

## 2. Where the value comes from

| Value lever | Annual size (AA 2015 analog) | How the system captures it |
|---|---|---|
| Over-provisioning recovered | up to **9.2M EUR** | Reads trip origin from the reservation system, so passengers who begin their journey at the station and go home are not costed a room they never use |
| Faster sourcing | time to secure rooms falls from about 2h40 to under 40 minutes | Triage plus parallel requests plus deterministic ranking, before the last check-in |
| Audit trail | avoids leakage on contested EU261 care claims | Every offer, rejection, timestamp and authorisation recorded |
| Crew-rest protection | not quantified, potentially large | Flags hotels too far out to keep crew inside minimum rest, avoiding a next-morning cancellation |

Only the first lever is put into the ROI. The other three are real but conservatively left out, so the return is understated rather than inflated. The 9.2M is the **ceiling** of the first lever, not what gets captured on day one (section 4).

## 3. Costs

| Cost | One-time | Annual | Note |
|---|---|---|---|
| Discovery and design | included below | | The workflow logic is already built as a working prototype, which removes most of the usual build cost |
| Integration to the reservation / DCS trip-origin feed | **dominant line** | | This is the hard part and the real spend. Everything else is cheap |
| Workflow build and configuration | low | | Largely done |
| LLM API (gpt-5-mini) | | under 3,000 EUR / yr | About 3,305 events times a few short calls at fractions of a cent. Genuinely trivial |
| Observability (LangSmith), hosting (n8n), monitoring | | | Small subscriptions |
| Operations, 0.5 FTE owner plus support | | | The ongoing human cost |
| **Assumed totals** | **350,000 EUR** | **100,000 EUR / yr** | Integration-dominated one-time; lean run cost |

The striking fact: the technology is cheap. The cost is integration and change management, not compute. A skeptical CFO should be told the software is a rounding error and the real question is the data feed.

## 4. The capture assumption, and why it is the whole ballgame

The system cannot recover the full 9.2M in year one, for honest reasons: trip origin is not available for every booking (split tickets, stations that only receive a PNL), the share who actually go home is a human judgement that will sometimes miss, and rollout is station by station. So value is a growing fraction of the ceiling.

| | Year 1 | Year 2 | Year 3 |
|---|---|---|---|
| Capture of the 9.2M ceiling | 25% | 55% | 70% |
| Rationale | pilot at a few stations, feed coverage partial | scaled rollout, coverage grows, managers calibrated | steady state, most bookings carry usable trip origin |

## 5. Return

Base case, using the capture path above.

| EUR millions | Year 1 | Year 2 | Year 3 | 3-year |
|---|---:|---:|---:|---:|
| Value captured | 2.30 | 5.06 | 6.44 | 13.80 |
| Cost (build + run) | 0.45 | 0.10 | 0.10 | 0.65 |
| Net | 1.85 | 4.96 | 6.34 | 13.15 |
| Cumulative net | 1.85 | 6.81 | 13.15 | |

* **12-month ROI:** about **410%** (net 1.85M on 0.45M spent)
* **36-month ROI:** about **20x** (net 13.15M on 0.65M spent)
* **Payback:** the 350,000 EUR build is recovered within roughly the first two to three months of steady capture, because even 25% of 9.2M is about 190,000 EUR a month

A 20x return looks too good, so stress it. **Conservative case:** halve every capture rate (12 / 30 / 40 percent) and double every cost (700,000 build, 200,000 a year run):

| EUR millions | Year 1 | Year 2 | Year 3 | 3-year |
|---|---:|---:|---:|---:|
| Value captured | 1.10 | 2.76 | 3.68 | 7.54 |
| Cost | 0.90 | 0.20 | 0.20 | 1.30 |
| Net | 0.20 | 2.56 | 3.48 | 6.24 |

Even pessimistic on both sides, the three-year ROI is still roughly **480%** and the project is net positive from year one. The return is large because the inefficiency is large and the technology is cheap, not because the model is heroic.

## 6. The sensitivity that matters

There is only one variable that decides this, and it is not the software cost or the model quality. It is **capture rate, which is really trip-origin data coverage**. Move capture and the whole return moves; move anything else and it barely twitches. So the diligence question for the Carrier is not "how good is the AI" or "how much does it cost", it is **"how much of our booking base carries usable trip origin, and at which stations"**. That question should be answered before the integration is funded, which is exactly what the recommendation below does.

## 7. Risk matrix

Likelihood and impact are Low / Medium / High. Mitigations reference controls already built.

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| 1 | Trip-origin feed unavailable or low coverage, so capture never reaches plan | Medium | High | Pilot only at stations where the feed exists; conservative fallback costs everyone a room; treat as a go / no-go gate (section 8) |
| 2 | Capture lands below the assumed path | Medium | Medium | Conservative case is still strongly positive; phased targets, not a single bet |
| 3 | Model produces a wrong number that reaches a decision | Low | Medium | Grounding checks reject values not present in the source; arithmetic and ranking are deterministic code; a human authorises every commitment |
| 4 | Personal-data breach | Low | High | Privacy boundary: no passenger identity reaches the model or leaves the EEA; EU-region logging (see GDPR pack) |
| 5 | Duty managers do not trust or adopt it | Medium | Medium | Human stays in the loop; the result page shows its working; training and a visible "this is an estimate" framing |
| 6 | Automation bias: staff over-rely on the estimate | Low | Medium | The authorisation gate forces a human decision on spend; outputs are labelled estimates, never instructions |
| 7 | Dependence on one model provider | Low | Low | Model-agnostic design; the provider can be swapped without touching the logic |
| 8 | EU AI Act reclassification if the system starts to decide rather than propose | Low | Medium | Currently not high risk; documented re-assessment triggers (see EU AI Act doc) |
| 9 | Modelled unit costs are wrong | Medium | Low | Re-baseline on the carrier's real accommodation invoices during discovery |

**Residual risk** is concentrated almost entirely in row 1: the data. Everything else is either low or already controlled by design. That concentration is good news, because it means the project can be de-risked with one focused step rather than many.

## 8. Recommendation

Do not sign the full integration build before the data question is answered. Fund a short **paid pilot**: two or three stations where a trip-origin feed already exists, eight to twelve weeks, with one job, measure the real capture rate against the 25% year-one assumption. If capture holds, the full rollout is an easy yes with a 20x case behind it. If it does not, the Carrier has spent a small fraction of the build cost to learn the one thing that decides the project, which is exactly what a pilot is for.

The honest one-line pitch to Chleo: the software is cheap and the return is large, but both depend on one feed of data you already own and do not yet use at the station. Let us prove that feed works at three stations before you spend on the rest.

---

*Figures are illustrative, built on the American Airlines 2015 analog and the platform's modelled unit costs. A real engagement re-runs the model on the Carrier's own cancellation history, accommodation invoices and booking data. The system does not reduce statutory care or compensation, and no saving is claimed against either.*
