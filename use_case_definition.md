# Use case definition

**Use case:** AI-assisted overnight care for stranded passengers after a flight cancellation
**Client:** the Carrier (a large airline)
**Author:** Gordan Skopljak, AI consulting capstone, Round 2
**One line:** When a flight is cancelled, estimate the true cost of caring for the stranded passengers, decide fast who actually needs a room, and source those rooms before the last check-in, without ever handing passenger identities to an AI.

---

## Problem

When a flight cancels at night, a single station duty manager has minutes to work out how many rooms are needed, what it will cost, whether the station is allowed to commit that spend, and which hotels can take the passengers before check-in closes. Today this is done from memory and a phone list. Two things go wrong. The manager over-books rooms, because there is no fast way to see which passengers began their journey at the station and are simply going home, so everyone is treated as needing a bed. And the decision stalls, because the authority to commit spend sits above the station and the case for it is not in front of anyone. On the American Airlines 2015 analog, the over-booking alone is worth about 9.2 million euros a year, and the sourcing takes roughly two hours and forty minutes when the useful window is shorter than that.

There is a legal edge under all of it. Under Regulation (EC) 261/2004 the duty of care has no financial ceiling and survives even extraordinary circumstances, and on 82 percent of these overnight events the cause is extraordinary, so compensation is zero but the care is still owed. Care is the cost the airline always pays. Getting it right, fast and cheap, is the whole game.

## Company profile

A large network carrier operating across many stations, some of them its own hubs and many of them outstations where it relies on ground handlers. It already holds the data the solution needs, the passenger manifest and the reservation record, but that data does not reach the station in a usable shape on the night. It is subject to EU passenger-rights law, EASA crew-rest rules, and the EU AI Act and GDPR. Accommodation is largely sourced through contracted vendors, with a spot-market scramble when the contracted block runs out, which is the scenario this use case addresses.

## Solution

A two-part decision-support system, already built as a working prototype.

The first part reads the cancelled flight's manifest and estimates the total cost of care: rooms, meals, transport, and the EU261 exposure, distinguishing the care that is always owed from the compensation that depends on cause. It works out who is based at the station and going home, so the room requirement is right rather than inflated, and it routes the decision to the station or, above the commitment limit, to the carrier operations centre. Crucially, passenger names are read and discarded at ingestion; the AI model receives only counts and money, never an identity.

The second part triages nearby partner hotels by distance, capacity and the overnight rest they leave before the next departure, then requests, reads and ranks offers deterministically against the station's commitment limit, and hands the manager a ranked call list with the reasoning. It proposes and ranks; a human commits.

## Stakeholders

* **Station duty manager** (primary user): needs the room number, the cost, and permission, fast.
* **Carrier operations centre** (authoriser): decides anything above the station commitment limit.
* **Crew scheduling**: the crew-rest constraint that can turn one cancellation into two.
* **Accommodation vendor / ground handler**: acts on the sourcing output.
* **Finance**: owns the care budget and the over-provisioning it is losing.
* **Data protection officer and compliance**: own the GDPR and EU AI Act position.
* **IT and integration**: own the reservation feed that the whole value depends on.
* **Passengers**: the beneficiaries, accommodated faster and correctly.

## Success criteria (measurable)

Pilot, two or three stations, eight to twelve weeks:

* Recover at least **25 percent** of the modelled over-provisioning at the pilot stations, measured as rooms not booked for passengers who went home, versus the pre-system baseline.
* Cut time to a ranked, costed accommodation decision to **under 40 minutes** from the current baseline of roughly 2h40.
* **Zero** passenger identities reach the language model, verified by the privacy guard firing on any attempt and by an audit of the model payloads.
* A complete, timestamped evidence trail for **100 percent** of pilot incidents (offers, rejections, authorisation, deadline).
* No crew placed below EASA minimum rest by an accommodation the system ranked as usable.
* Duty-manager trust: at least **80 percent** of pilot decisions accepted without the manager overriding the room requirement, and a positive usability verdict from the managers.

Steady state, post-rollout: capture rising toward **70 percent** of the over-provisioning ceiling, sourcing consistently inside the check-in window, and the care estimate within a defined tolerance of actual invoiced cost.

## Out of scope

* **Booking, confirming or paying for rooms automatically.** The system sources and ranks; a human commits. It never sends a payment guarantee.
* **Contacting passengers.** No passenger-facing channel.
* **Reducing statutory care or compensation.** Both are legal obligations; no saving is claimed against either.
* **Real passenger data in the prototype.** Development and demonstration use synthetic data only.
* **APIS or migration data** for cost modelling, to respect purpose limitation.
* **Anything touching flight safety or air-traffic management.** This is back-office logistics, not an operational safety system.
* **Live hotel availability.** The triage uses nominal partner-list capacity; actual availability is always confirmed by contacting the hotel.

---

*Companion documents: EU AI Act classification, GDPR pack with DPIA, ROI and risk assessment, and the BI dashboard, all built on the American Airlines 2015 analog with synthetic costs. The working system is the set of n8n workflows (IRREG v1 to v4 and Crisis Accommodation).*
