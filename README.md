# Airline disruption care platform

**AI consulting capstone, Ironhack. Rounds 1 and 2.**  
Author: Gordan Skopljak  
Client persona: Chleo, CEO of a large EU airline. Her stated objection to AI is that it is not transparent.

---

## What this is, in three sentences

When a flight cancels at night, one station duty manager has minutes to work out how many hotel rooms are needed, what they cost, whether the station is allowed to commit that spend, and which hotels can take the passengers before check in closes. This project is three working n8n workflows that cost the night, rank the accommodation options, and capture what was agreed on the phone. The model extracts and writes prose; every number that decides money is calculated in deterministic code, which is what makes the output explainable and is the direct answer to the client's transparency objection.

**The system proposes, ranks and explains. A human commits.** Nothing is booked, confirmed or paid automatically.

![Workflow 1, IRREG v3 cancellation exposure, in n8n](Workflow%201.png)
*Workflow 1 in n8n: manifest in, names pseudonymised, the night costed and the authorisation routed. Workflows 2 and 3 are shown under [The three workflows](#the-three-workflows).*

---

## Read this first: the sector changed after Round 1

Round 1 was researched and presented on **German grantmaking foundations**. Round 2 is built on **airline disruption care**. Two changes happened and both are documented in `round1_decision.md`, which is the file to read before judging why the Round 1 research does not match the Round 2 build.

What Round 1 found, and what carried across into Round 2, is kept in `round1_decision.md` rather than deleted, because a decision record that removes the evidence of what was decided is not a decision record.

---

## Start here, five minutes

1. Read `use_case_definition.md` to understand the problem.
2. Import the three workflow JSON exports into n8n (being added, see Known limitations), add one OpenAI credential, and click the manual trigger on IRREG v3. It runs on a built in 195 passenger fixture with no upload required.
3. Follow `mvp_documentation.md`, section 4, which tells you exactly which nodes to open to see the output.

**Before importing anything, read the warning at the top of `mvp_documentation.md` about credentials in the exported JSON.**

---

## Repository map

```
README.md                                           this file
use_case_definition.md                              problem, stakeholders, success criteria, scope limits
round1_decision.md                                  both sector changes, and what carried across
roi_risk_assessment.md                              costs, value, 12 and 36 month return, nine risks
strategic_deployment_plan.md                        POC to pilot to rollout, KPIs, commercialisation
mvp_documentation.md                                setup, run instructions, node by node walkthrough

Compliance
  eu_ai_act.md                                      classification and obligations
  EU AI Act addendum Hotel Call Intelligence workflow.md
  gdpr.md                                           data flows, legal bases, DPIA
  GDPR addendum Hotel Call Intelligence workflow.md

Workflows (n8n)
  Workflow 1.png / 2.png / 3.png                    canvas screenshots
  JSON exports                                      not yet uploaded, see Known limitations

Communication layer (HTML pages, PDF copies)
  Cancellation exposure.html                        page for workflow 1
  Crisis accommodation.html                         page for workflow 2
  IRROP Hotel Call Intake.html                      page for workflow 3
  OCC Recovery Map.html                             how the recovery process fits together

Data and samples
  RP4_Flights_2025_Jan_Dec.xlsx, RP4_Flights_2026_Jan_Jul.xlsx, State_ATFM_Delay.xlsx
  manifest-sample.csv, pnl-sample.txt, requirements.txt

disruption_care_capstone_final.pptx                 final presentation
```

**Why there are two compliance addenda.** The third workflow records a phone call, so personal data reaches the model. That breaks the design defence the first two workflows rest on. Rather than quietly weakening the main packs, the change is documented separately and argued on a different footing. See section 4 of the GDPR addendum.

---

## The three workflows

| # | Workflow | What it does | Where the model is used |
|---|---|---|---|
| 1 | IRREG v3, cancellation exposure | Reads the manifest, destroys the names, costs the night, routes the authorisation decision | Writes one briefing from numbers already computed |
| 2 | Crisis Accommodation | Triages partner hotels on rest before the morning departure, ranks offers | Drafts the request, extracts numbers from replies, writes a triage overview |
| 3 | Hotel Call Intelligence | Transcribes the closing call, extracts commercial terms with confidence scores | Whisper transcription plus structured extraction behind a review gate |

**Workflow 2, Crisis Accommodation.** Drafts the hotel request, waits for replies, extracts and ranks the offers, posts the ranking to Slack.

![Workflow 2, Crisis Accommodation, in n8n](Workflow%202.png)

**Workflow 3, Hotel Call Intelligence.** Transcribes the hotel call, extracts the agreed terms, recalculates the cost in code and drafts the manager email. The broken production webhook is labelled in the canvas itself.

![Workflow 3, Hotel Call Intelligence, in n8n](Workflow%203.png)

Full setup, run instructions, error handling and known limitations are in `mvp_documentation.md`.

---

## Verifying the claims

Every claim below can be checked in the code rather than taken on trust. `mvp_documentation.md` section 8 names the exact node for each.

| Claim | Where to check |
|---|---|
| Passenger names are destroyed at ingestion | `Read And Pseudonymise Manifest`, workflow 1 |
| No personal data reaches the model in workflows 1 and 2 | `Build Model Payload` and `Privacy Guard`, workflow 1 |
| The model never calculates money | `Build Cost Envelope`, workflow 1, and `Score And Rank Offers`, workflow 2 |
| Extracted values are checked against the source text | the `grounded` function in `Score And Rank Offers`, workflow 2 |
| A human gates every commitment | `Who Must Authorise The Care Spend`, workflow 1, and `Calculate Costs & Validate`, workflow 3 |

---

## Data and provenance

**Real:** cancellation counts, causes, dates, airports and distances. American Airlines 2015, US Department of Transportation, evening departures that strand passengers overnight. 3,305 overnight care events drawn from 10,919 real cancellations.

**Modelled:** passengers per flight, room, meal and transport unit costs, and EU261 compensation applied from the regulation's own bands.

**Synthetic:** all manifests, all hotel replies, all trip origin data.

**No real passenger data has been processed at any point.** Euro figures on US operational data are deliberate: the client is a European carrier operating under Regulation (EC) 261/2004, and no EU carrier publishes cancellation microdata at this granularity.

---

## Known limitations

Stated here rather than left to be discovered.

- The n8n JSON exports of the three workflows are not yet in this repository. They are being added with credentials and webhook URLs removed. Until then the canvas screenshots above show the node structure.
- The production webhook path in workflow 3 is broken. The demo runs through an upload form.
- Workflow 2 does not contact any hotel. Dispatch is simulated and replies come from a fixture.
- There is no shared incident reference across the three workflows. The design is in `strategic_deployment_plan.md`; it is not wired. This is the most significant gap.
- Hotel room counts are nominal from a static partner list, not live availability.
- Unit costs are unverified assumptions, to be re baselined on a carrier's real invoices.
- Trip origin is synthetic. The entire business case depends on that feed existing, which is precisely what the proposed pilot measures.

---

## The recommendation to the client

Do not fund the integration. Fund a three station pilot for eight to twelve weeks with one job: measure the real capture rate against the 25 percent assumption. The software is cheap, around 140 US dollars a year for all model calls across 3,305 events, and the return is large, but both depend entirely on one feed of data the carrier already owns and does not yet use at the station.

Full reasoning, phased milestones, KPIs and the commercialisation model are in `strategic_deployment_plan.md`.
