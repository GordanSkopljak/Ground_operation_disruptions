# Round 1 decision

**Author:** Gordan Skopljak, AI consulting capstone
**Date:** September 2026
**Campus identifiers:** Round 1 `project-5`, Round 2 `final-project`

> This file records what changed after the Round 1 presentation to teaching staff, and why. Two changes happened, not one. The second is a change of industry, which is why this document exists rather than a single line in the use case definition.

---

## 1. What changed, in one table

| | Round 1 as presented | After the change | Round 2 as built |
|---|---|---|---|
| Sector | German grantmaking foundations | German grantmaking foundations | Airline disruption care |
| Use case | Grant application triage and structured summary | Mittelverwendungskontrolle, verifying awarded funds were spent as agreed | Overnight care for stranded passengers after a cancellation |
| Client persona | Chleo, foundation CEO, objection is transparency | unchanged | Chleo, airline CEO, objection is transparency |
| Where the model is used | Extraction from unstructured applications | Reading free text against the stated purpose | Extraction and prose only, never arithmetic |

The client persona and her stated objection were held constant on purpose. The transparency problem is the thing being solved. The sector is the setting.

---

## 2. First change: inside the foundation sector

**What prompted it.** An interview with a founding board member of a German grantmaking foundation, who also serves as its board secretary. She was asked where AI would actually help in the grant lifecycle. Her answer was not the part I had proposed. She pointed at the end of the process: the reporting, and checking whether money that was given was spent the way the approved project said it would be.

**Why I accepted that rather than defending my proposal.** Two reasons, and the first is the one that matters.

Grant decisions at that foundation are a governance process. The board proposes, a separate supervisory body decides, and by policy no reason is given for acceptance or rejection. Putting software near that process is a conversation about legitimacy, and it is the wrong first conversation to have with a cautious client. Mittelverwendungskontrolle happens after the decision, runs against rules that are already written down, and nobody is refused funding because of it.

Second, the rules there are genuinely defined rather than vague. Grantees submit cost accounting as a table with date, currency and amount in their own columns and a flag for Eigenbelege. Deadlines are fixed: interim report on 31 December, final accounting at 18 months, a six week clock on a material finding. That is a schema and a rule set, which means most of the work is arithmetic in code and only one narrow task, judging free text against the Zweckbindung, needs a model at all.

**What this taught me, and it is the more durable lesson:** I had picked the most visible step in the process rather than the one where automation was welcome. A practitioner corrected that in twenty minutes of conversation. That is what discovery is for, and it is why the Round 2 recommendation is a paid pilot rather than a build.

---

## 3. Second change: leaving the sector entirely

**I changed my mind.** Nobody redirected me on this one. Having settled the foundation use case, I decided the capstone was better served in a different industry, and moved from the NGO sector to aviation.

**What I moved to.** Airline disruption care: costing the overnight accommodation of passengers stranded by a cancellation, sourcing the rooms, and capturing what was agreed on the phone.

**Why, stated plainly.**

**The number.** The Mittelverwendung case is real and defensible, but its value is measured in Controlling staff hours, and I had no way to size that without access to the foundation's own data. The airline case rests on published US Department of Transportation cancellation records: 10,919 real American Airlines cancellations in 2015, of which 3,305 strand passengers overnight. That produces a business case built on real operational counts rather than assumed ones. A consulting proposal that cannot put a defensible number in front of a CFO is an interesting idea, not a proposal.

**The data I would have needed, and the time I had.** My own pitch document names what the foundation case required to become real: the Fördervereinbarung template, a set of past cost accountings with their tables, and time with whoever in Controlling actually does the checking. That last one was the important one, and none of the three could be obtained inside the remaining weeks. Rather than build an MVP on invented foundation data and describe it as if it were grounded, I moved to a domain where the underlying operational data is public.

**Interest, which I am not going to dress up as something else.** I was more engaged by the aviation problem. Over a capstone timeline that is not a trivial factor. The work is better because I wanted to keep doing it at eleven at night, and pretending otherwise in a document about honest decision making would be an odd choice.

**What I am not claiming.** I am not claiming the foundation case was wrong, or that the steer I was given was mistaken. It was correct, and if I were continuing past this capstone that is the case I would take back to a foundation, with the three things above requested up front. The move was about what could be evidenced and built in the time available, not about the quality of the use case.

---

## 4. What carried across

This is the part that makes the second change a continuation rather than a restart. Four things survived intact.

**The architectural principle.** The model extracts and writes prose. Every number that decides money is calculated in deterministic code. In the foundation case, sums, dates, schema checks, sampling and thresholds were rules, so they were code, and the model read only free text against the Zweckbindung. In the airline case the model writes the briefing and pulls values out of a hotel reply, and every euro is arithmetic in JavaScript. Identical discipline, different subject.

**Extraction never evaluation.** Round 1 excluded scoring or ranking applicants for award, and named it as risk R1. Round 2 excludes booking, paying and contacting passengers, and names the same drift risk as an EU AI Act re assessment trigger. Both systems propose. A human commits.

**Honest provenance.** The Round 1 research labelled every figure SOURCED or ASSUMPTION. Round 2 carries the same discipline: real cancellation counts, causes, dates and airports; modelled passengers and unit costs; and the distinction stated on the dashboard itself rather than hidden in a footnote.

**Transparency as the thing being demonstrated, not claimed.** Round 1 argued that a use case where nothing visibly goes wrong cannot answer a transparency objection. Round 2 acts on that: the privacy guard is shown firing, extractions that do not match the source text are pulled out of the ranking and displayed as failures, and incomplete hotel replies become specific questions rather than guesses.

---

## 5. What was lost, honestly

The sector research, opportunity and risk mapping, and use case comparison produced for Round 1 apply to German foundations and do not transfer. That is a genuine cost of the change, roughly two weeks of research that informs no Round 2 deliverable directly. The risk register method transferred; its contents did not.

The Round 1 artefacts remain in the repository under `project-5` rather than being deleted, because a decision document that quietly removes the evidence of what was decided is not a decision document.

---

## 6. Round 1 to Round 2 mapping

| Round 1 artefact | Status in Round 2 |
|---|---|
| `sector_research.md` (foundations) | Superseded. Retained as Round 1 evidence |
| `use_cases.md` (three foundation use cases) | Superseded by `use_case_definition.md` |
| `opportunities_risks.md` | Method carried into the ROI and risk assessment; contents replaced |
| `mittelverwendung-pitch.md` | The output of the first change. Retained as the record of what the practitioner interview produced |
| POC | Rebuilt as three n8n workflows in the new domain |
| LangSmith monitoring | Carried forward. Observability logging remains in the exposure workflow |
| Dashboard | Rebuilt on the American Airlines 2015 data |

---

## 7. What I would do differently

Ask for the data before choosing the use case, not after designing it. Both changes trace to the same root: I designed first and discovered second. The interview that redirected the foundation work should have happened in week one, and the question of what data actually existed should have been answered before either use case was written up.

That is also why the Round 2 recommendation to the client is a three station pilot rather than a full integration build. The lesson is in the deliverable, not just in this file.
