# Airline disruption care platform

**AI consulting capstone, Ironhack. Rounds 1 and 2.**
Author: Gordan Skopljak
Client persona: Chleo, CEO of a large EU airline. Her stated objection to AI is that it is not transparent.

---

## What this is, in three sentences

When a flight cancels at night, one station duty manager has minutes to work out how many hotel rooms are needed, what they cost, whether the station is allowed to commit that spend, and which hotels can take the passengers before check in closes. This project is three working n8n workflows that cost the night, rank the accommodation options, and capture what was agreed on the phone. The model extracts and writes prose; every number that decides money is calculated in deterministic code, which is what makes the output explainable and is the direct answer to the client's transparency objection.

**The system proposes, ranks and explains. A human commits.** Nothing is booked, confirmed or paid automatically.

---

## Read this first: the sector changed after Round 1

Round 1 was researched and presented on **German grantmaking foundations**. Round 2 is built on **airline disruption care**. Two changes happened and both are documented in `feedback/round1_decision.md`, which is the file to read before judging why the Round 1 research does not match the Round 2 build.

The Round 1 materials are retained in `research/` rather than deleted, because a decision record that removes the evidence of what was decided is not a decision record.

---

## Start here, five minutes

1. Read `use_case_definition.md` to understand the problem.
2. Import `mvp/` into n8n, add one OpenAI credential, and click the manual trigger on IRREG v3. It runs on a built in 195 passenger fixture with no upload required.
3. Follow `mvp/mvp_documentation.md`, section 4, which tells you exactly which nodes to open to see the output.

**Before importing anything, read the warning at the top of `mvp/mvp_documentation.md` about credentials in the exported JSON.**

---

## Repository map

```
├── README.md                          this file
├── feedback/
│   └── round1_decision.md             both sector changes, and what carried across
├── research/                          Round 1, German foundations
│   ├── sector_research.md
│   ├── use_cases.md
│   ├── opportunities_risks.md
│   └── mittelverwendung-pitch.md
├── dashboard/                         the communication layer
├── use_case_definition.md             problem, stakeholders, success criteria, scope limits
├── poc/
│   ├── poc_workflow.json              early IRREG, the proof that the approach works
│   └── poc_documentation.md
├── roi_risk_assessment.md             costs, value, 12 and 36 month return, nine risks
├── compliance/
│   ├── eu_ai_act_compliance.md        classification and obligations
│   ├── eu_ai_act_hotel_call_addendum.md
│   ├── gdpr_documentation.md          data flows, legal bases, DPIA
│   └── gdpr_hotel_call_addendum.md
├── strategic_plan.md                  POC to pilot to rollout, KPIs, commercialisation
├── presentation.pptx                  final presentation
└── mvp/                               three working workflows
    ├── irreg_v3.json
    ├── crisis_accommodation.json
    ├── hotel_call_intelligence.json
    ├── mvp_documentation.md
    └── images/
```

**Why there are two compliance addenda.** The third workflow records a phone call, so personal data reaches the model. That breaks the design defence the first two workflows rest on. Rather than quietly weakening the main packs, the change is documented separately and argued on a different footing. See section 4 of the GDPR addendum.

---

## The three workflows

| # | Workflow | What it does | Where the model is used |
|---|---|---|---|
| 1 | IRREG v3, cancellation exposure | Reads the manifest, destroys the names, costs the night, routes the authorisation decision | Writes one briefing from numbers already computed |
| 2 | Crisis Accommodation | Triages partner hotels on rest before the morning departure, ranks offers | Drafts the request, extracts numbers from replies, writes a triage overview |
| 3 | Hotel Call Intelligence | Transcribes the closing call, extracts commercial terms with confidence scores | Whisper transcription plus structured extraction behind a review gate |

Full setup, run instructions, error handling and known limitations are in `mvp/mvp_documentation.md`.

---

## Verifying the claims

Every claim below can be checked in the code rather than taken on trust. `mvp/mvp_documentation.md` section 8 names the exact node for each.

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

- The production webhook path in workflow 3 is broken. The demo runs through an upload form.
- Workflow 2 does not contact any hotel. Dispatch is simulated and replies come from a fixture.
- There is no shared incident reference across the three workflows. The design is in `strategic_plan.md`; it is not wired. This is the most significant gap.
- Hotel room counts are nominal from a static partner list, not live availability.
- Unit costs are unverified assumptions, to be re baselined on a carrier's real invoices.
- Trip origin is synthetic. The entire business case depends on that feed existing, which is precisely what the proposed pilot measures.

---

## The recommendation to the client

Do not fund the integration. Fund a three station pilot for eight to twelve weeks with one job: measure the real capture rate against the 25 percent assumption. The software is cheap, around 140 US dollars a year for all model calls across 3,305 events, and the return is large, but both depend entirely on one feed of data the carrier already owns and does not yet use at the station.

Full reasoning, phased milestones, KPIs and the commercialisation model are in `strategic_plan.md`.
