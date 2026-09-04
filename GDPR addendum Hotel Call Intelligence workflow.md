# GDPR addendum: Hotel Call Intelligence workflow

**Workflow:** Airline IRROP Hotel Call Intelligence Prototype (n8n workflow, form-triggered audio upload)
**Controller:** the Carrier (a large EU airline), referred to here as "the Carrier"
**Author:** Gordan Skopljak, AI consulting Airline XYZ, Round 2
**Assessment date:** September 2026
**Reads with:** capstone/compliance/gdpr.md (base pack)

> This workflow breaks the base pack's central assumption that the AI model never receives personal data. The audio recording, its transcript, and personal data of the ops agent and the hotel front desk staff all reach OpenAI in the United States. That inversion is what makes this workflow the most complicated part of the platform and drives everything below. The base pack still describes correctly the passenger manifest pipeline. This addendum covers the call recording pipeline.

---

## 1. What this workflow adds

The Hotel Call Intelligence workflow captures a phone call between an airline ops agent and a hotel front desk, transcribes it, extracts commercial terms, and drafts a manager approval email. Passenger manifests never enter it. Two new data subjects enter it:

* The **ops agent** (Carrier employee whose voice and speech are recorded and analysed).
* The **hotel front desk staff** (third-party business contact whose voice, name, and spoken commitments are recorded and analysed).

The privacy boundary that anchors the base pack (personal data never crosses to the AI side) is inverted on this branch. The AI side receives the audio and its text.

## 2. Roles

| Party | Role | Note |
|---|---|---|
| The Carrier | **Controller** | Determines why and how the call is recorded, transcribed and processed. |
| OpenAI (Whisper) | **Processor, receiving personal data** | Whisper receives the audio recording, which is personal data (voice of identifiable individuals). Article 28 processing agreement required. Standard Contractual Clauses module 2 (controller-to-processor) for the US transfer. Transfer impact assessment on file. |
| OpenAI (gpt-5-mini) | **Processor, receiving personal data** | The extraction LLM receives the transcript, which contains names, contact details, and business intent of identifiable individuals. Same Article 28 plus SCCs. |
| The hotel (as an entity) | Third party | Not joint controller in the GDPR sense: the hotel is not determining the purpose of the recording. Its front desk staff are the data subjects. |
| Ops agent's employer (the Carrier) | Employer, subject to worker representation law | Section 8. Not a GDPR role, but a live-deployment obligation the Carrier owns. |

## 3. What personal data the workflow processes

| Data element | Category | Where it comes from |
|---|---|---|
| Ops agent voice recording | Personal data | Recorded call, uploaded via the form |
| Hotel front desk voice recording | Personal data | Same recording |
| Transcript (paraphrase of spoken content) | Personal data | Whisper output |
| Hotel contact person name | Personal data | Extracted by the LLM into the structured summary |
| Ops manager email address | Personal data | Entered on the form |
| Incident metadata (hotel name, incident id) | Business context, not personal | Form fields |

Voice is treated as ordinary personal data (Recital 30 and consistent EDPB practice). It is not automatically special category. A voice may reveal accent or language that could support an argument under Article 9(1) about ethnic origin, but that argument is weak in a business-call context and the workflow does not process voice to draw any such inference. **Voice is treated as ordinary personal data.** If the workflow were later to derive attributes from voice characteristics (accent, gender, emotional state), Article 9 would engage and the whole basis needs revisiting.

## 4. Where the base pack's design no longer applies

The base pack's spine is: "the model never receives personal data, therefore no Chapter V transfer, no profiling by the AI, no re-identification risk from the model side." That reasoning does not carry to this workflow.

* **Whisper receives personal data** (the recording).
* **The extraction LLM receives personal data** (the transcript, which names people and paraphrases what they said).
* **Both processing steps happen in the United States** on OpenAI infrastructure.

This is a real Chapter V transfer of personal data. Standard Contractual Clauses and OpenAI's Data Processing Addendum apply. A transfer impact assessment must be on file. This is not a design-neutralised transfer, and it must be recorded as an active transfer in the Article 30 record of processing.

## 5. Legal bases

### 5.1 The ops agent's voice (Carrier employee)

* **Article 6(1)(f), legitimate interests.** The Carrier has a legitimate interest in accurate records of commercial commitments made by phone on its behalf during an IRROP, and in AI-supported summary and approval flows for the ops manager. The balancing test weighs this against the employee's expectation of not being recorded.
* The balance is defensible only if three conditions hold: the ops agent is informed clearly in advance that IRROP calls are recorded and transcribed by AI (Articles 13 and 14); the recording is limited to IRROP calls, not general work; the output is not used to evaluate the agent (see section 8 on works council).
* Article 6(1)(a) consent is disfavoured by the EDPB in employment contexts because the power imbalance makes it non-free. Do not rely on it.

### 5.2 The hotel front desk staff (external business contact)

* **Article 6(1)(f), legitimate interests.** The Carrier has a legitimate interest in accurately capturing what a supplier verbally committed to. This is standard business call recording practice.
* The balance requires **notification at the start of the call**, not after. Suggested script for the ops agent: "For accuracy this call is being recorded and later transcribed by our systems. If you would rather it was not, please tell me and I will disable it now."
* **Two-party consent rules vary by member state.** Germany requires all-party consent for private telephone conversations under §201 StGB. France requires consent. Most other member states are one-party for business calls. Treat all-party as the ceiling and standardise on it.
* If the hotel objects, the recording must be disabled for that call. The workflow currently has no explicit "disable recording" branch. That is a go-live gap flagged in section 10.

### 5.3 Ops manager email address

* **Article 6(1)(b), performance of the employment relationship.** Sending an internal work email to a colleague.

### 5.4 Special category data

None engaged in the workflow as designed. Voice is ordinary personal data. If the extraction ever produced inferences about a person's characteristics (accent, gender, emotional state, stress), Article 9 engages and the whole legal basis needs rewriting.

## 6. International transfer (Chapter V): now engaged

Unlike the base pack, personal data does leave the EEA on this branch.

* **OpenAI (US).** Standard Contractual Clauses module 2 (controller-to-processor). OpenAI Data Processing Addendum on file. A transfer impact assessment must consider US surveillance law (FISA 702, Executive Order 12333). OpenAI publishes a page on this and it is adequate as a starting point; the Carrier's DPO signs the final assessment.
* **Retention on OpenAI's side.** Default 30 days for abuse monitoring on the standard API tier. A zero-retention endpoint is available for enterprise customers. If the Carrier goes to production, contract the zero-retention endpoint or an enterprise-tier equivalent.
* **Alternatives if the transfer is blocked.** Self-hosted Whisper on EU infrastructure (the model weights are open). EU-hosted LLM providers (Mistral, or Anthropic when their EU region is generally available). Adds cost and operational overhead. Named here so the DPO has a fallback to point at, not necessarily to build now.

## 7. DPIA delta (Article 35)

The base pack DPIA covered the passenger manifest pipeline. This workflow triggers a new DPIA on its own facts, because it introduces three of the regulator's high-risk indicators:

* **Systematic monitoring of communications** (recording every IRROP hotel call).
* **Innovative use of new technology on personal data** (LLM transcription and extraction on voice content).
* **International transfer of newly captured personal data.**

### 7.1 Risks and mitigations

| Risk | Likelihood after controls | Mitigation |
|---|---|---|
| Hotel staff recorded without their knowledge or consent | Medium until the disable-recording branch is built and the notification script is standardised | Notification script at call start; documented ops procedure; disable-recording flow in the workflow |
| Ops agent recording used as employee performance evaluation | Low if intent is enforced | Documented purpose limitation; no agent-level KPIs or dashboards built on the extraction data; works council agreement on the recording purpose |
| Transcript exposes hotel commercial terms externally | Low | Transcript lives only in the n8n execution data (turn off), the manager email, and the OpenAI temporary API buffer |
| OpenAI subpoenaed for a recording under US surveillance law | Low, non-zero | Zero-retention endpoint in production; DPO's transfer impact assessment covers the residual |
| Audio file persists in n8n binary storage after the run | Medium until configured | Enable file cleanup on the workflow; execution data saving off |
| Voice recording accidentally used to infer speaker attributes | Low | Not part of the extraction schema; enabling it requires a code change, which is a re-assessment trigger under section 6 of the AI Act addendum |
| Wrong extraction leads to a bad commercial commitment | Low | Per-section confidence scores; REVIEW_REQUIRED gate; human manager approves; hotel staff verbally confirmed on the call |

### 7.2 Residual risk

Residual risk is **medium** until three gates are met: the two-party recording notification is standardised and enforced in the ops procedure, execution data saving is off and audio cleanup verified, and the OpenAI zero-retention endpoint is used or an EU-hosted alternative is deployed. These are additive to the base pack's gates, not replacements.

## 8. Employment law overlay (outside GDPR, in scope for a live deployment)

Not GDPR strictly, but a Carrier will ask. In Germany, employee call recording requires works council (Betriebsrat) consultation and a Betriebsvereinbarung under §87 BetrVG. Other member states have analogous worker-representation requirements. HR and works council need to sign off the recording purpose, tightly scoped to IRROP commercial capture, explicitly not employee evaluation. Named here so it does not surprise anyone at the pitch stage.

## 9. Data subject rights

* **Access, rectification, erasure.** The two data subject categories are the ops agent (Carrier employee) and the hotel front desk staff (external business contact). Both need a route to request their recording or its deletion. The ops agent is served by the Carrier's employee data process. The hotel staff route is a documented business-contact procedure that the Carrier will need to publish before go-live.
* **Automated decision-making (Article 22).** Not engaged. The manager approves. The AI's review_required flag is decision support, not a decision with legal or similarly significant effect on the hotel staff. The commercial decision affects the hotel as an entity, not the front desk person as an individual data subject.
* **Right to object (Article 21).** Hotel staff can object to being recorded on the call. The workflow must support disabling the recording, per section 5.2.

## 10. Go-live gates (added to the base pack list)

Before this workflow touches a real IRROP call:

1. **Notification script deployed and enforced** for ops agents ("this call is recorded and transcribed by AI for internal record-keeping").
2. **Disable-recording branch built into the workflow** so a hotel objection can be honoured on the same call.
3. **Zero-retention OpenAI endpoint** contracted, or an EU-hosted alternative deployed.
4. **Works council agreement** signed in Germany (and analogous in other member states), scoped to IRROP commercial capture, explicitly excluding employee evaluation.
5. **Execution data saving off** and audio file cleanup verified on the workflow.
6. **DPIA sign-off** by the DPO on this specific workflow, including the transfer impact assessment for the US processing.

## 11. One honest caveat

The base pack is defensibly strong because the design keeps personal data away from the model. This workflow does not have that defence. It is defensible on a different footing: clear notification, tight purpose limitation, human oversight on any commercial commitment, and standard SCCs for the US transfer. It stops being defensible if the Carrier reuses the transcripts for agent evaluation, records hotel staff silently, or builds a voice-attribute inference layer on top. Any of those and the residual risk moves from medium to high, the DPIA has to be rewritten, and the AI Act classification (in the parallel addendum) may push into Annex III.

---

*Sources: Regulation (EU) 2016/679 (GDPR), Articles 6, 9, 13, 14, 21, 22, 28, 30, 35 and Chapter V; EDPB guidelines on consent in employment; §87 BetrVG and §201 StGB (works council consultation and private-conversation recording law in Germany); OpenAI Data Processing Addendum and enterprise zero-retention terms.*