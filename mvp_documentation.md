# MVP documentation

**System:** Airline disruption care platform
**Author:** Gordan Skopljak, AI consulting capstone, Round 2
**Date:** September 2026

This document is written so that someone who has never seen the project can import it, run it, and see the AI capability work, without asking me anything. Every workflow has a path that runs with no uploaded file and no external service beyond an OpenAI key.

---

## STOP: read this before importing

The exported workflow JSON contains two secrets in plain text. **Remove them before pushing anything to a public repository.**

| Where | What | Action |
|---|---|---|
| IRREG v3, node `HTTP Request to Lang Smith` | LangSmith API key, hardcoded in the header parameters | Delete the value, or move it to an n8n credential, and rotate the key |
| Crisis Accommodation, node `Post Ranked Hotels To Slack` | Slack incoming webhook URL, hardcoded | Delete the value and rotate the webhook |

A Slack webhook URL is itself a credential: anyone holding it can post into the channel. Both were rotated or removed before submission. If you are reading an export that still contains them, treat them as compromised.

---

## 1. What the MVP is

Three n8n workflows that together handle one night of flight cancellation care.

| # | Workflow | What it does | AI capability demonstrated |
|---|---|---|---|
| 1 | IRREG v3, cancellation exposure | Reads a passenger manifest, discards names, costs the night, routes the authorisation decision | LLM writes the exposure briefing from pre computed numbers only |
| 2 | Crisis Accommodation | Triages partner hotels, requests offers, extracts and ranks them | LLM drafts the request, extracts numbers from replies, writes a triage overview |
| 3 | Hotel Call Intelligence | Transcribes a hotel call, extracts commercial terms with confidence scores | Whisper transcription plus structured extraction with a confidence and review gate |

**The architectural principle, and the thing to look at when reviewing the code:** the model extracts and writes prose. Every number that decides money is calculated in deterministic JavaScript. This is why the output is explainable, and it is the direct answer to the client's stated objection about AI transparency.

---

## 2. Prerequisites

- An n8n instance, cloud or self hosted. Built and tested on the Ironhack provisioned instance.
- An **OpenAI API key** with access to `gpt-5-mini` and the audio transcription endpoint. This is the only credential required to see all three workflows run.
- Optional, for full functionality only:
  - A **Gmail OAuth2** credential, for the approval email in workflow 3. Without it the workflow still renders its result page.
  - A **LangSmith** API key, for observability logging in workflow 1. Without it that one node fails and the rest continues.
  - A **Slack incoming webhook**, for the notification in workflow 2. That node is set to continue on error, so its absence does not stop the run.

Total cost to run all three workflows once: well under one cent. See the ROI document for the measured token counts.

---

## 3. Setup, about ten minutes

1. **Import the three workflow JSON files** into n8n, one at a time.
2. **Create one OpenAI credential.** Then open each of the five model nodes and select it: `Briefing Model` in workflow 1, `Drafting Model`, `Extraction Model` and `Overview Model` in workflow 2, `OpenAI Extraction Model` and `Whisper Transcribe` in workflow 3. n8n does not map credentials automatically on import, so this must be done by hand.
3. **Fix the sub workflow link.** In workflow 1, the node `Start Accommodation Sourcing` calls workflow 2 by ID. That ID will be different in your instance. Open the node and reselect Crisis Accommodation from the list.
4. **Turn off "save successful execution data"** in each workflow's settings. This is a compliance requirement, not a preference: with it on, a raw manifest with real names would persist in the database. It is documented as a go live gate in the GDPR pack.
5. Leave the LangSmith, Gmail and Slack nodes unconfigured if you do not have those services. Each is described in section 6.

---

## 4. How to run it, no files needed

Each workflow has a built in fixture so a reviewer can see it work immediately.

### Workflow 1, the fastest way to see the whole thing

![IRREG v3 cancellation exposure workflow](images/Workflow_1.png)

*Two entry points on the left: the form trigger for a real upload, and the manual trigger for the fixture. Both converge on the same chain. The privacy boundary is the pair `Build Model Payload` and `Privacy Guard` in the middle right, immediately before the only language model call in this workflow.*

Open **IRREG v3** and click the manual trigger **Run Built In Scenario**, then Execute Workflow.

This runs the entire chain on a synthetic 195 passenger manifest without any upload. Nothing renders in the browser on a manual run by design, so **click through the nodes to read the output**. The two worth opening are:

- `Privacy Guard`, which shows the exact payload sent to the model and confirms zero identifiers were found.
- `Render Result Page`, whose `html` field contains the full result page. Copy it into a file and open it in a browser to see what a duty manager would see.

**To see the browser version instead:** open the `Upload Passenger Manifest` form trigger, take its URL, and submit a manifest CSV. The form runs in two steps, upload then confirm, and the result page renders at the end.

### Workflow 2

![Crisis Accommodation sourcing workflow](images/Workflow_2.png)

*Three entry points: manual fixture, a handover form, and a sub workflow trigger called by IRREG. Note `Dispatch Requests (Simulated)`, which marks records as sent without contacting anyone, and `Wait For The Reply Deadline`, which pauses the execution until you resume it. The two model nodes overlap visually at the same canvas coordinates; the connections are correct.*

Open **Crisis Accommodation** and click **Run With Fixture Requirement**. It runs on a hardcoded requirement of 89 rooms, 4 accessible, 121 guests. Hotel replies are simulated from a hardcoded inbox inside the `Collect Replies` node, so no email account is needed. Read the output at `Render Comparison`.

The `Wait For The Reply Deadline` node is set to manual resume, so the execution will pause. Resume it from the executions list to continue.

### Workflow 3

![Hotel Call Intelligence workflow](images/Workflow_3.png)

*The green box top left is the production webhook path, which is broken and unused. The demo runs along the blue path below it. The red box is the extraction and validation core: the model returns structured JSON with per section confidence, then `Calculate Costs & Validate` recomputes the total in code and decides whether a human is required.*

Open the **Hotel Call Upload Form** trigger, take its URL, and upload any audio recording of a conversation about a hotel booking, plus an email address. Whisper transcribes it, the extractor pulls out the terms, and the result page renders in the browser.

There is no fixture path for this workflow because it needs real audio. If you have no recording, record thirty seconds on a phone stating a hotel name, a number of rooms, a rate and a cancellation policy. That is enough to exercise the full chain.

---

## 5. What "working" looks like

If the run succeeded you should see, in workflow 1:

- A `Privacy Guard` output where `guard.passed` is true and the payload contains only counts, money and codes. No names, no booking references.
- A cost envelope with three columns, low, expected and high, and an `authorisation` value of either `STATION` or `CARRIER`.
- A briefing of at most nine lines written by the model, containing no number that was not already in the payload.

In workflow 2:

- Hotels sorted into tiers `CALL_FIRST`, `USABLE`, `PARTIAL` and `AVOID`, with the blockers stated for each.
- Offers ranked cheapest first, with a `decision` of `STATION_MAY_COMMIT`, `ESCALATE_TO_CARRIER` or `NO_USABLE_OFFER`.
- At least one reply not ranked. The Skyport Hotel fixture reply deliberately states no numbers, so it should land in the incomplete list with a specific follow up question rather than being guessed at.

In workflow 3:

- A `validation.status` of either `READY_FOR_MANAGER` or `REVIEW_REQUIRED`, with reasons listed when it is the latter.
- Per section confidence scores between 0 and 1.
- A cost total recalculated in code, not taken from the model.

---

## 6. Error handling

Errors are deliberate and specific. The system fails loudly rather than producing a confident wrong answer.

**Hard stops, which throw and halt the execution:**

| Workflow | Condition | Message intent |
|---|---|---|
| 1 | Uploaded file is empty | Tells the user the file is empty |
| 1 | No passenger name elements recognised | States the two formats accepted, CSV or PNL name lines |
| 1 | Any identifier pattern found in the model payload | Privacy guard blocks the call and names what it found and where |
| 2 | Rooms required is zero, or guests fewer than rooms, or accessible rooms exceed total rooms | Names which two numbers contradict each other |
| 2 | Check in time not in HH:MM form | States the expected format |
| 2 | No hotel replied before the deadline | Says to escalate by phone and that the workflow cannot help further |
| 3 | Transcript missing or under 20 characters | "Refusing to fabricate a booking" |
| 3 | Manager email missing or malformed | Names the field |

**Soft handling, which continues with a warning rather than stopping:**

- **Missing manifest columns.** If there is no PNR column, every passenger is treated as travelling alone, which overstates the room requirement. If there is no trip origin column, nobody can be identified as based at the station and everyone is costed a room. Both are the conservative direction, and both raise a warning that appears on the result page.
- **Implausible derivations.** If more than 90 percent or fewer than 5 percent of passengers derive as based at the station, the workflow warns that the station code may be wrong rather than silently accepting it.
- **Manager input clamped.** If the manager says more passengers will go home than are eligible, the number is reduced to the eligible count and the discrepancy is reported.
- **Ungrounded extraction.** Every value the model extracts from a hotel reply is checked against the source text. A value that does not appear there is marked ungrounded and the offer is pulled out of the ranking for a human to read, rather than being ranked on an invented number.
- **Incomplete replies are never ranked.** A missing field becomes a specific question to ask the hotel, not a guess.
- **Confidence and value gates.** Workflow 3 flags REVIEW_REQUIRED on any missing critical field, any section confidence below 0.75, or any total above the high value threshold, and honours the model's own review flag on top of those checks.
- **Slack node** is set to continue on error, so a missing webhook does not break the run.

---

## 7. Known limitations

Stated here rather than discovered by the reader.

- **The production webhook path in workflow 3 is broken.** It expects JSON but routes to Whisper, which needs binary audio. The demo runs through the upload form. This is marked with a sticky note on the canvas and will be fixed when a telephony platform is chosen.
- **Workflow 2 does not send anything.** The `Dispatch Requests (Simulated)` node marks records as dispatched without contacting any hotel. Replies come from a hardcoded inbox. Outbound dispatch is Phase 2 work.
- **There is no shared incident reference across the three workflows.** Each generates its own. The design is written up in the strategic plan but it is not wired, which means the audit trail claim is currently true within a workflow and not across them. This is the most important known gap.
- **Hotel room counts are nominal,** taken from a static partner list, not a live availability check. The workflow says so on its own output.
- **Unit costs are unverified assumptions.** Room rates, meal vouchers, coach costs, accept and claim rates. A real engagement re baselines these on the carrier's invoices.
- **Trip origin is synthetic.** There is no reservation system integration, and the entire business case depends on that feed existing. This is the subject of the proposed pilot.
- **Two node pairs overlap on the workflow 2 canvas** at the same coordinates, which makes the diagram harder to read than it should be. Cosmetic only, the connections are correct.
- **Synthetic and public data only.** No real passenger data has been processed at any point.

---

## 8. Where to look in the code

For a reviewer who wants to check the claims rather than take them on trust:

| Claim | Node to open |
|---|---|
| Names are destroyed at ingestion | `Read And Pseudonymise Manifest`, workflow 1 |
| The model receives no personal data | `Build Model Payload` and `Privacy Guard`, workflow 1 |
| The model never calculates money | `Build Cost Envelope`, workflow 1, and `Score And Rank Offers`, workflow 2 |
| Extracted values are checked against the source | the `grounded` function in `Score And Rank Offers`, workflow 2 |
| The model cannot invent fields | the extraction schemas in `Extract Hotel Offer`, workflow 2, and `LLM Fact Extraction`, workflow 3 |
| A human gates every commitment | `Who Must Authorise The Care Spend`, workflow 1, and `Calculate Costs & Validate`, workflow 3 |

---

*Companion documents: use case definition, EU AI Act classification and hotel call addendum, GDPR pack and hotel call addendum, ROI and risk assessment, strategic deployment plan, dashboard specification.*
