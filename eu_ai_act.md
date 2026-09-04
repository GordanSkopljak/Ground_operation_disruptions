# EU AI Act classification and obligations

**System:** Airline disruption care platform (IRREG cancellation exposure model, plus the Crisis Accommodation sourcing workflow)
**Deployer / client:** the Carrier (a large EU airline), referred to here as "the Carrier"
**Author:** [your name], AI consulting capstone, Round 2
**Assessment date:** September 2026
**Legal instrument:** Regulation (EU) 2024/1689 (the AI Act), as amended by the Digital Omnibus

> This document is a capstone compliance assessment on a prototype built with synthetic data. It is written to be honest rather than impressive: the system is deliberately classified at the level the law actually supports, not inflated to sound serious.

---

## 1. What the system does

The platform has two connected parts.

**IRREG** takes the passenger list of a cancelled flight (a PNL teletype message or a CSV export) and estimates the total cost of caring for the stranded passengers overnight: rooms, meals, transport, and the airline's exposure under Regulation (EC) 261/2004 (care under Article 9 and compensation under Article 7). It never sends passenger identities to any AI model. A language model (OpenAI gpt-5-mini) writes one plain language briefing for the station duty manager from already computed numbers, and separately drafts the text of a request to hotels.

**Crisis Accommodation** takes room counts and times only (no passenger data crosses the boundary), triages a static partner hotel list by distance, capacity and overnight rest before the next departure, has the model write a short overview for the duty manager, requests offers, extracts the numbers from replies, and ranks them in code against the station's contractual commitment limit.

The output is decision support for a human. The station duty manager and, above the commitment limit, the carrier operations centre make every decision that commits money or affects a passenger. The system proposes, ranks and explains; it does not book, confirm, pay, or contact a passenger.

## 2. Who we are under the Act: provider or deployer

The Act assigns obligations by role, so the role has to be settled first.

* **OpenAI** is the provider of the general purpose AI model (gpt-5-mini). Its GPAI obligations are its own and are already in force (since 2 August 2025). We rely on them but do not carry them.
* **The Carrier** is the **deployer** of the AI system: it uses the system under its own authority in the course of its operations.
* Whoever puts the workflow into service under their own name and trademark (the Carrier, or a vendor selling it to the Carrier) is the **provider of the AI system** built on top of the model. In this capstone the Carrier operates it internally, so the Carrier is both provider and deployer of the system, while remaining a deployer of the underlying model.

This matters because provider obligations are heavier than deployer obligations, and both are light for a system that is not high risk. See section 4.

## 3. Risk classification

The Act sorts systems into prohibited, high risk, limited risk (transparency) and minimal risk. Each tier is tested in turn.

### 3.1 Prohibited practices (Article 5) — not applicable

The system does none of the prohibited things: no social scoring, no biometric categorisation, no emotion recognition, no exploitation of vulnerability, no untargeted scraping of faces, no predictive policing. It processes an operational manifest to estimate a hotel bill. **Not prohibited.**

### 3.2 High risk (Annex III) — not high risk, with reasoning per category

This is the load bearing judgement, so every Annex III heading is checked rather than waved past.

| Annex III area | Applies? | Why not |
|---|---|---|
| Biometrics | No | No biometric data is processed at all. |
| Critical infrastructure | No | The system is not a safety component in the management or operation of critical infrastructure. It is back office cost and logistics. It has no role in air traffic management or flight safety. A wrong number produces a wrong hotel estimate, not an unsafe flight. |
| Education and vocational training | No | Irrelevant. |
| Employment and worker management | No | It touches crew only as a room count for rest, never to evaluate, hire, allocate or monitor a worker. |
| Access to essential private and public services | No | This is the closest call and still no. The heading covers systems that decide a data subject's access to a service, benefit, creditworthiness, or emergency dispatch. Here the passenger's right to care is guaranteed by Regulation (EC) 261/2004 regardless of what the system outputs; the system estimates the airline's cost of providing that care. It does not decide whether a passenger receives care, and it does not evaluate the passenger. |
| Law enforcement, migration, justice | No | Explicitly out of scope. The design forbids using APIS country of residence data for cost modelling, precisely to stay clear of the migration heading and of purpose creep. |

**Conclusion: the system is not high risk under Annex III.** Even if a future version were argued into the "essential services" heading, Article 6(3) provides an exemption where the system performs a narrow procedural task or improves the result of a completed human activity without replacing human judgement, which describes this system well: a human makes every consequential decision.

### 3.3 Limited risk / transparency (Article 50) — partially engaged

Article 50 transparency duties came into force on 2 August 2026 (with a grace period to 2 December 2026). They apply to systems that interact with people, and to AI generated content.

* The **briefing and the triage overview** are written by the model for an internal professional user (the duty manager). This is not a public facing chatbot, so the direct interaction duty is a light touch: the manager should simply know these texts are AI drafted, which the result page states plainly.
* The **hotel request text** is AI drafted and sent outward to a business. Marking machine generated text as such is good practice here even though a short operational message to a hotel is a marginal case.

**Position: treat the system as limited risk. Apply the transparency practices in section 5 as a matter of course rather than waiting to be compelled.**

### 3.4 Minimal risk — the residual tier

Everything the system does that is not caught above sits in the minimal risk tier, which carries no mandatory obligations beyond AI literacy. The bulk of the platform lives here.

## 4. Obligations that actually apply, given the classification and the current timeline

Because the system is not high risk, the heavy Annex III obligations (risk management system, data governance file, technical documentation to Annex IV, logging, human oversight design, accuracy and robustness, conformity assessment, CE marking, EU database registration) are **not legally required**. They are also, usefully, mostly already satisfied voluntarily (section 5).

What does apply now:

1. **AI literacy (Article 4), in force.** The Carrier must ensure the staff who operate and rely on the system have sufficient understanding of it. Deliverable: a one page operator briefing for duty managers explaining what the system does, what it does not do, and that its numbers are estimates.
2. **Transparency (Article 50), in force since August 2026.** Label AI drafted text as AI drafted. Already done on the result page; extend the same line to the outbound hotel request.
3. **GPAI provider obligations (Article 53 onward), in force since August 2025, carried by OpenAI.** The Carrier's duty is only to use the model within the provider's acceptable use terms and to keep the provider's documentation on file.

What does not yet apply, but should be watched:

* **High risk obligations are postponed, not cancelled.** The Digital Omnibus moved the Annex III standalone date to **2 December 2027** and the Annex I product embedded date to **2 August 2028**. If a future version of the system crosses into a high risk use (see section 7), these obligations attach from those dates.

## 5. Voluntary conformity: controls already in place

The strongest part of this assessment is that the system already meets, by design, several controls the Act reserves for high risk systems. This is documented here both as good practice and as evidence of readiness if the classification ever changes.

| High risk style control | How the system already does it |
|---|---|
| Human oversight (Article 14) | Every money committing or passenger affecting decision is taken by a human. The authorisation gate routes anything above the station commitment limit to the carrier operations centre. The AI proposes; it never acts. |
| Data governance and minimisation (Article 10) | Passenger names are read to count people and then discarded. Booking references are replaced by a per execution salted hash. The model receives counts and money only. |
| A guardrail against unsafe output | A "privacy guard" node inspects the model payload and throws if any identifier pattern is present, so a coding mistake cannot leak personal data to the model. It has fired in testing, which is the point: a guard that never fires is decoration. |
| Accuracy and robustness (Article 15) | Model extractions of hotel replies are checked against the source text; any value that does not appear in the reply is pulled out of the ranking for a human to read. Ranking and arithmetic are deterministic code, not model judgement. |
| Record keeping and logging (Article 12) | LLM calls are logged to LangSmith (EU region) for observability, carrying the computed briefing only, never passenger identities. |
| Transparency (Article 13, Article 50) | The result page shows the exact payload the model received and separates what was derived from data from what a human judged. |

## 6. Technical documentation outline (Annex IV shape)

Not required at the current classification, sketched here to show conformity readiness and because the rubric asks for it.

1. General description: purpose, deployer, intended users (station duty managers, carrier ops), what it is not for.
2. System architecture: the node chain of each workflow, the privacy boundary, where the model is and is not used.
3. Data: input formats (PNL, CSV), the fields used, the fields deliberately discarded, synthetic data policy for development.
4. Model: gpt-5-mini via OpenAI, the three narrow places it is used (briefing, overview, reply extraction), the system prompts, the grounding and privacy guards around it.
5. Human oversight: the authorisation gate, the manager set inputs, the "this is an estimate" framing.
6. Accuracy and limitations: known weaknesses stated plainly (nominal room counts are not live availability; trip origin misreads split bookings; unit costs are unverified assumptions).
7. Risk management: the residual risks and the controls in section 5.
8. Change log and re assessment triggers (section 7).

## 7. Re assessment triggers

The classification holds only for the system as built. Re run this assessment if any of the following change:

* The system begins to **decide** rather than propose (for example, auto confirming a hotel booking or auto issuing compensation). That removes the human in the loop and pushes toward high risk.
* It starts processing data that reaches into an Annex III heading, especially migration or law enforcement data.
* It is turned on real passenger data (the prototype is synthetic only).
* It becomes public or passenger facing rather than an internal tool.
* A new EU standard is published in the Official Journal that changes the presumption of conformity.

## 8. One honest caveat for the panel

The most defensible classification here is "not high risk", and it is defensible precisely because of design choices, human oversight and no automated decisions with legal effect, not because aviation is a low stakes field. If a reviewer pushes on the "essential services" heading, the answer is not to deny it exists, it is to point at Article 6(3) and at the human authorisation gate. The moment the system is allowed to commit spend or issue compensation on its own, this document has to be rewritten, and the honest thing is to say so before someone asks.

---

*Sources: Regulation (EU) 2024/1689 (EU AI Act); Digital Omnibus postponement of high risk obligation dates (Annex III to 2 December 2027, Annex I to 2 August 2028); Article 50 transparency obligations in force 2 August 2026; GPAI obligations in force 2 August 2025. Regulation (EC) 261/2004 for the passenger care and compensation framework the system estimates.*
