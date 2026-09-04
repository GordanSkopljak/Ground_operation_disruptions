# GDPR data protection pack

**System:** Airline disruption care platform (IRREG cancellation exposure model, plus the Crisis Accommodation sourcing workflow)
**Controller:** the Carrier (a large EU airline), referred to here as "the Carrier"
**Author:** [your name], AI consulting capstone, Round 2
**Assessment date:** September 2026
**Legal instrument:** Regulation (EU) 2016/679 (GDPR)

> The prototype runs on synthetic data only. This pack describes the data protection position for a production deployment on real passenger data, which is what the Carrier would need before going live. Where the design already prevents a risk, that is stated rather than dressed up as a control that still needs building.

---

## 1. Roles

| Party | Role | Note |
|---|---|---|
| The Carrier | **Controller** | Determines why and how passenger data is processed for disruption care. |
| Ground handler / station | Processor, or joint controller | Acts on the Carrier's behalf under the ground handling agreement. Handling passenger care can make the station a joint controller for that purpose; settle this in the contract. |
| Accommodation vendor (e.g. an incumbent portal) | Processor | Books rooms on the Carrier's instructions. Needs an Article 28 processing agreement. |
| OpenAI (gpt-5-mini) | Processor | **Receives no personal data** (section 4). Processing agreement and acceptable use still on file. |
| LangSmith (EU region) | Processor | Receives the computed briefing only, **no passenger identities**. Article 28 agreement on file. |

## 2. What personal data the system touches

The passenger manifest is the only source of personal data. Everything downstream is derived from it or is non personal.

| Data element | Category | Where it comes from |
|---|---|---|
| Passenger name | Personal data | PNL name element / CSV |
| Booking reference (PNR / record locator) | Personal data (an identifier) | PNL / CSV |
| Party grouping | Personal data | Derived from the booking |
| Trip origin (airport where the booking began) | Personal data | PNR extract |
| Reduced mobility / special assistance (WCHR, WCHS, PRM) | **Special category (health), Article 9** | SSR service codes in the manifest |
| Unaccompaniedminor flag (UMNR) | Personal data about a **child** | SSR service codes |
| Crew count | Not personal (an aggregate) | Separate crew list |

Two sensitive points are called out because they drive the DPIA: PRM/WCHR codes are health data, and UMNR concerns a child. Both attract the higher protection of Articles 9 and of the child specific duties.

## 3. Data flow

```
Passenger manifest (PNL or CSV)      <- personal data, incl. special category
        |
        v
  Parse and pseudonymise             <- names read and DISCARDED here
        |                               PNR replaced by salted hash
        |                               PRM reduced to "needsAccessibleRoom" operational flag
        v
  Segment parties, model behaviour   <- works on counts and pseudonymous party keys only
        |
        v
  Build model payload                <- allow-list: counts and money only
        |
   [ PRIVACY GUARD ]                 <- throws if any identifier reaches this point
        |
        v
  Language model (OpenAI)            <- receives NO personal data
        |
        v
  Result page / briefing            <- counts, money, and the model's prose
        |
        v
  LangSmith (EU) log                <- briefing text only, no identities
```

The vertical line after "Build model payload" is the privacy boundary. Personal data lives above it and inside n8n. Below it, only non personal aggregates exist. This is the single most important fact in the whole pack, because it collapses several risks that would otherwise be serious (international transfer, model training exposure, profiling by the AI).

## 4. The point that resolves most of the risk

The language model, and the US based processing behind it, **never receive personal data**. The payload is built field by field by an allow list and contains counts, room numbers and money. A fail closed "privacy guard" node inspects that payload and throws an error if any identifier pattern is present, so a future coding mistake cannot quietly leak data to the model. Because of this:

* There is **no international transfer of personal data** to OpenAI in the United States. Only non personal aggregates leave the EEA. Chapter V is therefore not engaged for the model call.
* The model cannot profile a passenger, because it never sees one.
* LangSmith is on the **EU region endpoint** and logs the computed briefing only, so even the observability trail holds no identities.

This is privacy by design under Article 25, and it is the spine of the compliance argument.

## 5. Legal bases

### 5.1 Ordinary personal data (Article 6)

* **Article 6(1)(b), performance of the contract of carriage.** Re accommodating a passenger whose flight the Carrier cancelled is part of performing, and remedying, the transport contract.
* **Article 6(1)(c), legal obligation.** Regulation (EC) 261/2004 Article 9 obliges the Carrier to provide care (accommodation, meals, transport). Processing the manifest to discharge that duty rests on a legal obligation.
* **Article 6(1)(f), legitimate interests**, as a secondary basis for the cost and logistics optimisation, balanced against passengers who benefit from being accommodated faster.

### 5.2 Special category data: PRM/WCHR health data (Article 9)

Article 9 prohibits processing health data unless an exception applies. Two moves handle this.

1. **Minimise it out of the AI path entirely.** PRM/WCHR is reduced at the earliest step to a single operational flag, `needsAccessibleRoom`. The flag records an operational requirement (an accessible room), not a diagnosis, and it never reaches the model. This is data minimisation, not a legal basis, but it shrinks the Article 9 processing to the smallest possible footprint.
2. **For the unavoidable processing of the raw code in the manifest**, the applicable exception is **Article 9(2)(g), substantial public interest** (safe carriage and the statutory duty of care to disabled and reduced mobility passengers, which sits alongside Regulation (EC) 1107/2006 on the rights of disabled air passengers), supported by the Carrier's existing obligation to accommodate assistance needs. Explicit consent under 9(2)(a) is not workable on a cancellation night and is not relied on.

### 5.3 Child data (UMNR)

An unaccompanied minor flag concerns a child, so the processing follows the same bases as above but with heightened care: the flag is used only to guarantee the child a single room and an escort room, is pseudonymised like every other party, and never reaches the model.

## 6. Data Protection Impact Assessment (short form, Article 35)

### 6.1 Is a DPIA required?

Yes, and doing one is the safe call. A DPIA is required where processing is likely to result in a high risk to individuals. Here two of the regulator's triggers are present: **special category (health) data**, and processing that could be seen as **large scale** and using **new technology** (an AI system on passenger manifests). Even though the design reduces the actual risk sharply, the trigger is about the nature of the data, so the DPIA is warranted.

### 6.2 Necessity and proportionality

The processing is necessary to discharge a statutory duty of care and to do it at the speed a disruption demands. It is proportionate because it uses the minimum data: names are discarded, identifiers are hashed, health data is reduced to an operational flag, and the AI component sees none of it.

### 6.3 Risks and mitigations

| Risk | Likelihood after controls | Mitigation already in place |
|---|---|---|
| Passenger identities leak to a third party AI or to the US | Low | Allow list payload plus fail closed privacy guard; only aggregates leave the EEA |
| Health data (PRM) processed beyond what is needed | Low | Reduced to an operational flag at ingestion; never modelled |
| Raw manifest persists in the workflow database | Medium until configured | "Do not save successful execution data" must be enabled; documented as a go live gate |
| A child's data mishandled | Low | UMNR pseudonymised, used only for room allocation |
| Re identification from the pseudonymous key | Low | Key is a per execution salted hash, meaningless outside one run |
| Wrong estimate harms a passenger | Low | Human authorises every decision; outputs are labelled estimates |

### 6.4 Residual risk and conclusion

Residual risk is **low** provided the two go live gates are met: execution data saving is turned off, and real deployment keeps the privacy boundary intact. The DPIA conclusion is that the processing may proceed with those controls, and should be reviewed if the system starts to decide rather than propose.

## 7. Data subject rights

In the prototype there are no real data subjects (synthetic data), so this section describes the production commitment.

* **Access, rectification, erasure, restriction, portability, objection:** served by the Carrier's existing passenger data processes, since the manifest originates in the reservation and departure control systems. The disruption platform holds no durable personal record of its own once execution data saving is off; it processes a manifest transiently and keeps only non personal aggregates.
* **Automated decision making (Article 22):** **not engaged.** There is no solely automated decision producing legal or similarly significant effects on a passenger. A human (the station duty manager, and the carrier operations centre above the commitment limit) makes every consequential decision. The system is decision support, and the design makes that structural, not a policy promise.
* **Children:** the UMNR handling above reflects the heightened duty to a child data subject.

## 8. International transfers (Chapter V)

* **OpenAI (US):** no personal data is transferred, by design (section 4), so Chapter V is not engaged for the model call. Keep the processing agreement and the provider's transfer safeguards on file regardless.
* **LangSmith:** the EU region endpoint is used, so the observability data stays in the EEA, and it carries no identities in any case.
* **Accommodation vendor:** if a vendor processes passenger data outside the EEA, a Chapter V transfer mechanism (adequacy decision or standard contractual clauses) is required in that contract. Flagged as a procurement gate.

## 9. Retention and security

* **Retention:** the platform is designed to hold no durable personal data. With execution data saving off, a manifest is processed and discarded within the run. Any retention of aggregates for reporting is non personal.
* **Security:** pseudonymisation of booking references, discarding of names, the privacy boundary, EU region logging, and access control on the n8n instance. Standard transport security (TLS) on all calls.

## 10. Go live gates (the honest checklist)

Before this touches one real passenger, three things are non negotiable, and all three are already understood from building it:

1. **Turn off "save successful execution data"** on every workflow, or the raw manifest sits in the database with names in it.
2. **Never load a real manifest into the prototype environment;** production runs on a controlled instance under the Carrier's controllership.
3. **Keep the privacy boundary intact:** any change that would put personal data into the model payload must be caught by the privacy guard and reviewed.

---

*Sources: Regulation (EU) 2016/679 (GDPR), Articles 5, 6, 9, 22, 25, 28, 35 and Chapter V; Regulation (EC) 261/2004 (passenger care duty); Regulation (EC) 1107/2006 (rights of disabled and reduced mobility air passengers). Regulator guidance on when a DPIA is required (special category data, large scale processing, innovative technology).*
