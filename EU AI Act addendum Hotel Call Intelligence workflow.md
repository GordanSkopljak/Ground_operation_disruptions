# EU AI Act addendum: Hotel Call Intelligence workflow

**Workflow:** Airline IRROP Hotel Call Intelligence Prototype (n8n workflow, form-triggered audio upload)
**Deployer / client:** the Carrier (a large EU airline), referred to here as "the Carrier"
**Author:** Gordan Skopljak, AI consulting Airline XYZ, Round 2
**Assessment date:** September 2026
**Reads with:** capstone/compliance/eu_ai_act.md (base pack)

> This addendum covers only what the Hotel Call Intelligence workflow adds to the base classification. Where the base pack's reasoning still holds it is referenced not repeated. Where a new component (audio recording, automatic speech recognition) changes the calculus, the change is stated plainly. The base pack's overall conclusion (not high risk, limited risk transparency, deployer of a GPAI-based system) survives this analysis, on tighter reasoning.

---

## 1. What this workflow adds

This workflow takes a phone call between an airline ops agent and a hotel front desk during an IRROP, and produces a structured commercial summary plus an approval email to the ops manager.

The pipeline:

1. Form upload of the recorded call (mp3/wav/m4a/ogg/webm/mp4).
2. Whisper transcription (OpenAI, US).
3. LLM extraction of hotel terms with per-section confidence scores (gpt-5-mini, US).
4. Deterministic cost calculation and validation gate (missing fields, low confidence, high-value flag).
5. HTML result page rendered to the browser of the ops agent who uploaded.
6. Gmail send to the ops manager for approval.

Human decisions still gate anything binding. The manager approves the arrangement, the hotel commits the rooms verbally on the call, and the workflow captures and structures what a human negotiated. The system proposes, ranks and explains. It does not book, confirm, pay, or contact a passenger.

## 2. Provider and deployer roles for the new components

Two new components sit on top of the base classification.

* **Automatic Speech Recognition (Whisper).** OpenAI is the provider of Whisper, a general-purpose AI model. Whisper is not itself a biometric categorisation system or an emotion recognition system in the meaning of the Act (see 3.1 below). The Carrier is the deployer.
* **The Hotel Call Intelligence workflow itself.** As with the base system, whoever puts it into service under their own name is the provider of the composite AI system. In this capstone the Carrier operates it internally, so the Carrier is provider and deployer of the composite system, and a deployer of the underlying OpenAI models.

No change to the base pack's role analysis.

## 3. Risk classification: retest per heading

### 3.1 Prohibited practices (Article 5): two specific checks

Whisper introduces voice processing. The Act treats voice-related use more strictly than text, so two Article 5 prohibitions need an explicit read-through.

* **Emotion recognition in the workplace (Article 5(1)(f)), in force 2 August 2026.** Prohibited. Whisper is speech-to-text; it does not infer emotional state, mood, or affect. The extraction LLM reads the transcript, not the audio waveform, and its schema captures commercial facts (rooms, rate, cancellation), not inferences about the ops agent's affect. **Not engaged.**
* **Biometric categorisation by protected characteristic (Article 5(1)(g)).** Prohibited. The workflow does not categorise the speaker by race, political opinion, religious belief, sexual orientation, or union membership. Whisper transcribes speech; the LLM extracts commercial terms. **Not engaged.**
* **Remote biometric identification.** Not engaged. The system does not identify a speaker from voice biometrics. It knows who is on the call only from what the ops agent typed in the form.

None of the Article 5 practices apply.

### 3.2 High risk (Annex III): not high risk, with one boundary that must be enforced

| Annex III area | Applies? | Why not |
|---|---|---|
| Biometrics | No | Speech-to-text is not biometric identification or categorisation. Whisper does not build or match a voiceprint. |
| Critical infrastructure | No | Back-office commercial workflow. No role in air traffic management or flight safety. |
| Employment and worker management | **Marginal, requires a documented boundary** | The ops agent is an employee. Their voice is recorded and their handling of the call is transcribed and analysed. If the Carrier used this output to evaluate individual agent performance, Annex III (4)(b) engages: "AI systems intended to be used to make decisions affecting terms of work-related relationships, to monitor and evaluate performance and behavior of persons in such relationships." The intended use here is commercial fact capture from the counterparty (the hotel), not agent evaluation. That intent must be documented in the change log and the operator briefing. Any drift toward agent scoring reclassifies the system. |
| Access to essential private and public services | No | Same reasoning as the base pack. The hotel arrangement supports Regulation (EC) 261/2004 care, but this system captures commercial terms; it does not decide whether a passenger receives care. |
| Law enforcement, migration, justice | No | Explicitly out of scope. |

**Conclusion: the workflow is not high risk, on the same reasoning as the base pack, subject to one explicit boundary. The system must not be used to evaluate the ops agent.** If that boundary is crossed the workflow reclassifies to high risk under Annex III (4)(b) and the heavy obligations attach. Article 6(3) exemption (narrow procedural task, does not replace human judgement) continues to cover the residual risk in the "essential services" heading.

### 3.3 Limited risk / transparency (Article 50): extended

Article 50 duties in the base pack apply here too, extended to two new outputs.

* **The extracted terms and cost summary shown on the result page** are LLM output. The page labels the extraction as AI generated.
* **The approval email sent to the ops manager** is AI drafted. The subject line starts with the status flag and the body identifies itself as generated from the call.
* **The transcript itself** is a machine-generated artifact, but is not "AI generated content" in the Article 50(4) sense (deepfake / synthetic media). It is a factual record of what was said. No transparency obligation attaches.

### 3.4 Minimal risk: unchanged

Everything else sits in the minimal risk tier.

## 4. Obligations that apply now

Same three that applied in the base pack:

1. **AI literacy (Article 4).** Operator briefing extended: ops agents must understand that the recording is transcribed by AI, that the extracted terms are estimates that need human verification before the manager signs off, and that the ops agent (not the AI) is on the hook for what the hotel actually agreed to.
2. **Transparency (Article 50).** Already applied to the result page, extended to the manager email.
3. **GPAI provider obligations (Articles 53 onward).** Carried by OpenAI. The Carrier keeps the provider's documentation on file.

## 5. Voluntary conformity: what the workflow already does

Same structural controls as the base pack, plus one specific to this workflow.

| High-risk style control | How the workflow already does it |
|---|---|
| Human oversight (Article 14) | Manager approval required for any commercial commitment. The validation node flags REVIEW_REQUIRED on missing critical fields, per-section confidence below 0.75, or total above the high-value threshold. Any flag routes to human review before the workflow proceeds. |
| Data governance (Article 10) | The extraction schema is a strict allow-list of fields with types. The LLM cannot invent fields; per-section confidence is required output. |
| Accuracy (Article 15) | Deterministic re-check after LLM extraction: critical fields present, section confidences meet the floor, total under threshold, LLM's own review_required flag honoured. |
| Transparency of what the model saw | The result page shows the transcript (first 800 chars) alongside the extraction, so a human can verify without opening n8n. |
| Guardrail against wrong-call-type input | The system prompt instructs the LLM to set review_required if the transcript does not look like a hotel booking, and to empty out numeric fields. |

## 6. Re-assessment triggers specific to this workflow

Rerun this addendum if any of the following change:

* The system starts using the transcript or extraction to evaluate individual ops agents (crosses Annex III (4)(b)).
* Whisper output is used to infer emotional state, stress, competence, or any speaker attribute (crosses Article 5(1)(f) and potentially (g)).
* The workflow begins to auto-confirm bookings with the hotel without a human step (removes the Article 6(3) narrow-task defence).
* Recording scope widens beyond IRROP hotel calls to routine or general ops calls (increases systematic-monitoring character; may trigger Article 27 fundamental rights impact assessment for a public-authority-adjacent deployer).
* The system starts processing calls with actual passengers rather than business-to-business hotel calls (adds consumer-facing risk and Article 50(1) direct-interaction transparency).

## 7. One honest caveat

The sharpest reviewer question is: "if you are recording your own employee's voice and using AI to extract structured intelligence from what they say, how is that not employment monitoring under Annex III?" The defensible answer is that the purpose and use are commercial fact capture from the counterparty, not agent evaluation, and this purpose is documented and enforced by the operator briefing, the change log, and the absence of agent-level dashboards. If a reviewer pushes, the honest response is not to deny the risk exists. It is to point at the intended-use restriction and at the manager approval gate. The moment agent performance dashboards are built on top of this data, the workflow is high risk under Annex III (4)(b) and this addendum has to be rewritten.

---

*Sources as in the base pack, plus Article 5(1)(f) (emotion recognition in the workplace, in force 2 August 2026); Article 5(1)(g) (biometric categorisation); Annex III (4)(b) (worker performance monitoring); Article 27 (fundamental rights impact assessment).*