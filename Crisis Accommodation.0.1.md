{
  "nodes": [
    {
      "parameters": {},
      "id": "86b0e356-1b24-4d70-9bf5-e33ed4114f44",
      "name": "Run With Fixture Requirement",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        368,
        240
      ]
    },
    {
      "parameters": {
        "jsCode": "const src = $input.first() ? ($input.first().json || {}) : {};\nconst fromForm = Object.prototype.hasOwnProperty.call(src, 'roomsRequired');\nconst calledBy = src.calledFrom ? String(src.calledFrom) : null;\n\nconst num = (v, d) => {\n  const n = Number(v);\n  return Number.isFinite(n) && n >= 0 ? n : d;\n};\n\nconst req = fromForm\n  ? {\n      incidentId: String(src.incidentId || 'INC-UNKNOWN').trim(),\n      roomsRequired: num(src.roomsRequired, 0),\n      accessibleRoomsRequired: num(src.accessibleRoomsRequired, 0),\n      guests: num(src.guests, 0),\n      latestCheckIn: String(src.latestCheckIn || '23:30').trim(),\n      careCeilingEur: num(src.careCeilingEur, 0),\n      stationCommitmentLimitEur: num(src.stationCommitmentLimitEur, 25000),\n      nextFlightDeparture: String(src.nextFlightDeparture || '07:00').trim(),\n      source: calledBy ? 'SUB_WORKFLOW' : 'HANDOVER_FORM',\n      calledFrom: calledBy\n    }\n  : {\n      incidentId: 'INC-LH1234-2026-08-29',\n      roomsRequired: 89,\n      accessibleRoomsRequired: 4,\n      guests: 121,\n      latestCheckIn: '23:30',\n      careCeilingEur: 23498,\n      stationCommitmentLimitEur: 25000,\n      nextFlightDeparture: '07:00',\n      source: 'BUILT_IN_FIXTURE'\n    };\n\nconst problems = [];\nif (req.roomsRequired <= 0) problems.push('Rooms required is zero or missing. There is nothing to source.');\nif (req.guests < req.roomsRequired) problems.push('Fewer guests than rooms. Check the handover, one of the two numbers is wrong.');\nif (req.accessibleRoomsRequired > req.roomsRequired) problems.push('More accessible rooms than rooms in total. Check the handover.');\nif (problems.length > 0) throw new Error(problems.join(' '));\n\nconst toMinutes = (hhmm) => {\n  const m = String(hhmm).match(/^(\\d{1,2}):(\\d{2})$/);\n  return m ? Number(m[1]) * 60 + Number(m[2]) : null;\n};\n\nreq.latestCheckInMinutes = toMinutes(req.latestCheckIn);\nif (req.latestCheckInMinutes === null) throw new Error('Latest check in must be in HH:MM form, for example 23:30.');\n\nreq.boundaries = [\n  'This workflow requests offers, reads them and ranks them. It does not book, does not confirm, and does not give any hotel a payment guarantee.',\n  'No passenger data crosses into this workflow. It receives room counts and times only.',\n  'Nothing is sent to a real address. The dispatch step is simulated and labelled as such.'\n];\nif (calledBy) req.boundaries.push('This run was started automatically by ' + calledBy + ' as soon as the cancellation was costed. Starting the enquiry needs no authorisation because an enquiry commits nothing.');\n\nreturn [{ json: { req: req } }];\n"
      },
      "id": "2a49ffbc-773a-4aef-9ad0-dbd9b1e70a1d",
      "name": "Normalise Requirement",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        624,
        400
      ]
    },
    {
      "parameters": {
        "jsCode": "const req = $input.first().json.req;\n\n// A carrier owned list of airport partners and known nearby hotels.\n// In production this is the contracted supplier list plus a static nearby ring.\n// distanceKm is road distance from the terminal, not straight line.\nconst HOTELS = [\n  { hotelId: 'H1', name: 'Airport Inn',        partner: true,  distanceKm: 3,  nominalRooms: 160, accessibleRooms: 6, coachParking: true,  nominalRateEur: 119 },\n  { hotelId: 'H2', name: 'Terminal Lodge',     partner: true,  distanceKm: 5,  nominalRooms: 70,  accessibleRooms: 3, coachParking: true,  nominalRateEur: 129 },\n  { hotelId: 'H3', name: 'Skyport Hotel',      partner: true,  distanceKm: 9,  nominalRooms: 120, accessibleRooms: 4, coachParking: true,  nominalRateEur: 109 },\n  { hotelId: 'H4', name: 'City Park Hotel',    partner: false, distanceKm: 14, nominalRooms: 130, accessibleRooms: 5, coachParking: true,  nominalRateEur: 99  },\n  { hotelId: 'H5', name: 'Grand Central',      partner: false, distanceKm: 20, nominalRooms: 150, accessibleRooms: 5, coachParking: true,  nominalRateEur: 95  },\n  { hotelId: 'H6', name: 'Lakeside Hotel',     partner: false, distanceKm: 30, nominalRooms: 95,  accessibleRooms: 2, coachParking: false, nominalRateEur: 89  }\n];\n\n// Assumptions, all unverified and all in one place.\nconst COACH_SPEED_KMH = 50;      // airport road at night\nconst COACH_TURN_MIN = 20;       // load plus unload buffer per leg\nconst COACH_SEATS = 50;\nconst RECHECKIN_LEAD_MIN = 90;   // back at terminal this long before the next departure\nconst MIN_REST_MIN = 240;        // below four hours of room rest is flagged\nconst GOOD_REST_MIN = 300;       // five hours or more is comfortable\n\nconst toMin = (hhmm) => { const m = String(hhmm || '').match(/^(\\d{1,2}):(\\d{2})$/); return m ? Number(m[1]) * 60 + Number(m[2]) : null; };\nconst fmt = (mins) => { const h = Math.floor(mins / 60); const m = mins % 60; return h + 'h' + (m < 10 ? '0' : '') + m; };\n\nconst allInRooms = req.latestCheckInMinutes; // everyone in a room by this time tonight\nlet nextFlight = toMin(req.nextFlightDeparture);\nif (nextFlight === null) nextFlight = 7 * 60; // default 07:00\n// The next flight is tomorrow morning, tonight is after 20:00, so it is on the far side of midnight.\nconst nextFlightAbs = nextFlight + 1440;\nconst backAtTerminal = nextFlightAbs - RECHECKIN_LEAD_MIN;\n\nconst coaches = Math.ceil(req.guests / COACH_SEATS);\n\nconst assessed = HOTELS.map((h) => {\n  const transit = Math.round((h.distanceKm / COACH_SPEED_KMH) * 60) + COACH_TURN_MIN;\n  const departHotel = backAtTerminal - transit;      // must leave the hotel this early\n  const restMin = departHotel - allInRooms;          // actual room rest before the morning\n  const coversRooms = h.nominalRooms >= req.roomsRequired;\n  const coversAccessible = h.accessibleRooms >= req.accessibleRoomsRequired;\n\n  const blockers = [];\n  if (!coversRooms) blockers.push('holds about ' + h.nominalRooms + ' rooms, ' + req.roomsRequired + ' needed');\n  if (!coversAccessible) blockers.push(h.accessibleRooms + ' accessible, ' + req.accessibleRoomsRequired + ' needed');\n  if (!h.coachParking) blockers.push('no coach parking, ' + coaches + ' coaches must unload on the street');\n  if (restMin < MIN_REST_MIN) blockers.push('only ' + fmt(Math.max(0, restMin)) + ' rest before the ' + req.nextFlightDeparture + ' departure');\n\n  let tier;\n  if (blockers.length === 0) tier = restMin >= GOOD_REST_MIN ? 'CALL_FIRST' : 'USABLE';\n  else if (blockers.length === 1 && restMin >= MIN_REST_MIN && coversAccessible && h.coachParking) tier = 'PARTIAL';\n  else tier = 'AVOID';\n\n  return {\n    hotelId: h.hotelId, name: h.name, partner: h.partner, distanceKm: h.distanceKm,\n    nominalRooms: h.nominalRooms, accessibleRooms: h.accessibleRooms, coachParking: h.coachParking,\n    nominalRateEur: h.nominalRateEur,\n    transitMin: transit, restMin: restMin, restLabel: fmt(Math.max(0, restMin)),\n    coversRooms: coversRooms, coversAccessible: coversAccessible,\n    tier: tier, blockers: blockers,\n    estRoomsCostEur: coversRooms ? req.roomsRequired * h.nominalRateEur : h.nominalRooms * h.nominalRateEur,\n    estCoachCostEur: coaches * (300 + 8 * h.distanceKm)\n  };\n});\n\n// Order: call first, then usable, then partial, then avoid.\n// Inside a tier, partners before non partners, then more rest, then cheaper.\nconst tierRank = { CALL_FIRST: 0, USABLE: 1, PARTIAL: 2, AVOID: 3 };\nassessed.sort((a, b) =>\n  (tierRank[a.tier] - tierRank[b.tier]) ||\n  ((b.partner ? 1 : 0) - (a.partner ? 1 : 0)) ||\n  (b.restMin - a.restMin) ||\n  (a.nominalRateEur - b.nominalRateEur)\n);\n\nconst callList = assessed.filter((h) => h.tier === 'CALL_FIRST' || h.tier === 'USABLE' || h.tier === 'PARTIAL');\nconst singleHotelPossible = assessed.some((h) => h.tier === 'CALL_FIRST' || h.tier === 'USABLE');\n\nconst notes = [\n  'This is triage before anyone is contacted. Room counts are nominal, from the partner list, not a live availability check. The workflow still has to ask the hotel to confirm.',\n  'Rest is the room time a passenger gets before the ' + req.nextFlightDeparture + ' departure, after coaches carry them out and back with ' + RECHECKIN_LEAD_MIN + ' minutes to re-check-in. A closer hotel is worth more than a cheaper one on a short night.',\n  'Coach time assumes ' + COACH_SPEED_KMH + ' km/h and ' + COACH_TURN_MIN + ' minutes to load and unload each leg. ' + coaches + ' coaches move ' + req.guests + ' guests.',\n  singleHotelPossible ? '' : 'No single partner covers the requirement on its own. The station has to split across two hotels or pull in a non partner, which is a decision for a person.'\n].filter((n) => n.length > 0);\n\nconst timing = {\n  allInRooms: fmt(allInRooms),\n  nextFlight: req.nextFlightDeparture || '07:00',\n  backAtTerminal: fmt(backAtTerminal - 1440),\n  windowLabel: fmt(backAtTerminal - allInRooms) + ' between all in rooms and back at the terminal'\n};\n\nreturn [{\n  json: {\n    req: req,\n    coaches: coaches,\n    timing: timing,\n    hotels: assessed,\n    callList: callList,\n    singleHotelPossible: singleHotelPossible,\n    triageNotes: notes\n  }\n}];\n"
      },
      "id": "5545f344-c93f-488b-ac85-9c0ddbfb08fe",
      "name": "Triage Nearby Hotels",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        752,
        400
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ JSON.stringify($json.req) }}",
        "messages": {
          "messageValues": [
            {
              "message": "You draft the message a station duty manager sends to airport hotels in the first minutes after a flight cancellation. You are given room counts and times only, never any passenger information. Write one short message, at most eight lines of plain text. State the rooms needed, how many must be accessible, the time by which every guest must be in a room, and that coaches will bring the guests. Then ask, as a numbered list, for exactly these six things: rooms available tonight, rate per room per night in EUR, whether breakfast is included and if not the per person charge, number of accessible rooms, earliest check in time, and whether coaches can park on site. Ask for a reply by the stated deadline. Never promise payment, never mention a guarantee, never state a budget, never invent a number that is not given to you. No greeting placeholders, no sign off, no markdown."
            }
          ]
        },
        "batching": {}
      },
      "id": "cc0efe86-86ae-423b-b365-5d0d70b400e0",
      "name": "Draft The Request To Hotels",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1.9,
      "position": [
        864,
        400
      ]
    },
    {
      "parameters": {
        "model": {
          "__rl": true,
          "mode": "list",
          "value": "gpt-5-mini",
          "cachedResultName": "gpt-5-mini"
        },
        "builtInTools": {},
        "options": {}
      },
      "id": "f0ca05fc-6acf-4045-8498-82dd2db9927b",
      "name": "Drafting Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1.3,
      "position": [
        864,
        640
      ],
      "credentials": {}
    },
    {
      "parameters": {
        "jsCode": "const triage = $('Triage Nearby Hotels').first().json;\nconst req = triage.req;\nconst draft = String(($input.first().json || {}).text || '').trim();\n\n// Only the hotels triage marked worth calling are contacted, in the order it set.\nconst toCall = triage.callList;\n\nif (toCall.length === 0) {\n  return [{\n    json: {\n      hotelId: 'NONE', name: 'No hotel passed triage', distanceKm: null,\n      incidentId: req.incidentId, requestText: draft, dispatched: false,\n      triageEmpty: true\n    }\n  }];\n}\n\nreturn toCall.map((h) => ({\n  json: {\n    hotelId: h.hotelId,\n    name: h.name,\n    distanceKm: h.distanceKm,\n    coachParking: h.coachParking,\n    partner: h.partner,\n    tier: h.tier,\n    restLabel: h.restLabel,\n    incidentId: req.incidentId,\n    requestText: draft,\n    dispatched: false\n  }\n}));\n"
      },
      "id": "7d691f13-b319-4d8b-8d61-960ca7ceb758",
      "name": "Supplier List",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1104,
        400
      ]
    },
    {
      "parameters": {
        "assignments": {
          "assignments": [
            {
              "id": "a1",
              "name": "dispatched",
              "type": "boolean",
              "value": true
            },
            {
              "id": "a2",
              "name": "dispatchChannel",
              "type": "string",
              "value": "SIMULATED. Nothing was sent to a real address."
            },
            {
              "id": "a3",
              "name": "sentAt",
              "type": "string",
              "value": "={{ $now.toISO() }}"
            }
          ]
        },
        "includeOtherFields": true,
        "options": {}
      },
      "id": "50845176-c4dd-46a6-962b-303f169d0f0c",
      "name": "Dispatch Requests (Simulated)",
      "type": "n8n-nodes-base.set",
      "typeVersion": 3.4,
      "position": [
        1360,
        400
      ]
    },
    {
      "parameters": {},
      "id": "d6a7c144-9aa1-4da8-a3bf-b20a9936846d",
      "name": "Wait For The Reply Deadline",
      "type": "n8n-nodes-base.wait",
      "typeVersion": 1.1,
      "position": [
        1600,
        400
      ],
      "webhookId": "9e97b3b9-f40e-4dee-8d55-a7ad170cc000"
    },
    {
      "parameters": {
        "jsCode": "const sent = $input.all().map((i) => i.json);\nconst byId = {};\nfor (const s of sent) byId[s.hotelId] = s;\n\nconst inbox = {\n  H1: 'Dear colleagues, we confirm 160 rooms for tonight including 6 barrier free rooms. Rate 119 EUR per room, breakfast included. Check in from 21:00. Coach parking confirmed, we are 3 km from the terminal. Please confirm by 01:00. Kind regards, Airport Inn.',\n  H2: 'Good evening. We can hold 70 rooms tonight, 3 accessible, at 129 EUR per room, breakfast included. Check in any time, coach access at the front. 5 km out. Reply by 00:30 please. Terminal Lodge reception.',\n  H3: 'Guten Abend, wir haben heute Abend noch Zimmer frei. Bitte rufen Sie uns kurz an, dann besprechen wir Preis und Anzahl. Mit freundlichen Gruessen, Skyport Hotel.',\n  H4: 'Hi, we have about 130 rooms free, 5 accessible, 99 EUR with breakfast. Check in from 23:00, coach parking yes, 14 km from you. Let me know by 00:00. City Park Hotel.'\n}\n\nconst replied = [];\nconst silent = [];\nfor (const s of sent) {\n  if (Object.prototype.hasOwnProperty.call(inbox, s.hotelId)) {\n    replied.push({ hotelId: s.hotelId, name: s.name, distanceKm: s.distanceKm, coachParkingOnFile: s.coachParking, replyText: inbox[s.hotelId] });\n  } else {\n    silent.push({ hotelId: s.hotelId, name: s.name });\n  }\n}\n\nif (replied.length === 0) throw new Error('The deadline passed and no hotel replied. Escalate by phone. The workflow cannot help further.');\n\nreturn replied.map((r, i) => ({\n  json: i === 0 ? Object.assign({}, r, { _silent: silent, _repliedCount: replied.length, _sentCount: sent.length }) : r\n}));\n"
      },
      "id": "917b6bcf-5603-4d39-9588-7955031f6cc0",
      "name": "Collect Replies",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        1840,
        400
      ]
    },
    {
      "parameters": {
        "text": "={{ $json.replyText }}",
        "schemaType": "manual",
        "inputSchema": "{\n  \"type\": \"object\",\n  \"properties\": {\n    \"roomsAvailable\": {\n      \"type\": [\n        \"number\",\n        \"null\"\n      ],\n      \"description\": \"Number of guest rooms the hotel says it can hold tonight. Null if not stated. Never use the capacity of a conference or meeting room.\"\n    },\n    \"accessibleRooms\": {\n      \"type\": [\n        \"number\",\n        \"null\"\n      ],\n      \"description\": \"Number of barrier free or accessible rooms. Null if the reply does not mention accessible rooms at all. Never assume zero and never assume enough.\"\n    },\n    \"ratePerRoomEur\": {\n      \"type\": [\n        \"number\",\n        \"null\"\n      ],\n      \"description\": \"Rate per room per night in EUR. Null if the rate is quoted in any other currency. Never convert.\"\n    },\n    \"breakfastIncluded\": {\n      \"type\": [\n        \"boolean\",\n        \"null\"\n      ],\n      \"description\": \"True if breakfast is included in the room rate, false if charged separately, null if not mentioned.\"\n    },\n    \"breakfastPerPersonEur\": {\n      \"type\": [\n        \"number\",\n        \"null\"\n      ],\n      \"description\": \"Extra breakfast charge per person in EUR when breakfast is not included. Null otherwise or if quoted in another currency.\"\n    },\n    \"checkInFrom\": {\n      \"type\": [\n        \"string\",\n        \"null\"\n      ],\n      \"description\": \"Earliest check in time in HH:MM 24 hour form. Null if not stated.\"\n    },\n    \"coachAccess\": {\n      \"type\": [\n        \"boolean\",\n        \"null\"\n      ],\n      \"description\": \"True if coaches can park or unload on site, false if explicitly not, null if not mentioned.\"\n    },\n    \"distanceKm\": {\n      \"type\": [\n        \"number\",\n        \"null\"\n      ],\n      \"description\": \"Distance from the airport in kilometres. Null if not stated.\"\n    },\n    \"offerDeadline\": {\n      \"type\": [\n        \"string\",\n        \"null\"\n      ],\n      \"description\": \"Time by which the hotel needs an answer, in HH:MM 24 hour form. Null if not stated.\"\n    },\n    \"guaranteeRequested\": {\n      \"type\": [\n        \"boolean\",\n        \"null\"\n      ],\n      \"description\": \"True if the hotel asks for a payment guarantee or prepayment before holding the rooms.\"\n    }\n  },\n  \"required\": [\n    \"roomsAvailable\",\n    \"accessibleRooms\",\n    \"ratePerRoomEur\",\n    \"breakfastIncluded\",\n    \"checkInFrom\",\n    \"coachAccess\"\n  ]\n}",
        "options": {
          "batching": {
            "batchSize": 1,
            "delayBetweenBatches": 0
          }
        }
      },
      "id": "98ce07a4-0420-4d09-9d8b-9a8c55d18be6",
      "name": "Extract Hotel Offer",
      "type": "@n8n/n8n-nodes-langchain.informationExtractor",
      "typeVersion": 1.2,
      "position": [
        2080,
        400
      ]
    },
    {
      "parameters": {
        "model": {
          "__rl": true,
          "mode": "list",
          "value": "gpt-5-mini",
          "cachedResultName": "gpt-5-mini"
        },
        "builtInTools": {},
        "options": {}
      },
      "id": "8a4da4a1-986f-43d8-890d-98b0d67fede1",
      "name": "Extraction Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1.3,
      "position": [
        2080,
        640
      ],
      "credentials": {}
    },
    {
      "parameters": {
        "jsCode": "const req = $('Normalise Requirement').first().json.req;\nconst replies = $('Collect Replies').all().map((i) => i.json);\nconst head = replies[0] || {};\nconst silent = head._silent || [];\n\nconst extractions = $input.all().map((i) => {\n  const j = i.json || {};\n  return j.output && typeof j.output === 'object' ? j.output : j;\n});\n\nconst COACH_SEATS = 50;\nconst COACH_BASE_EUR = 300;\nconst COACH_PER_KM_EUR = 8;\n\nconst toMinutes = (hhmm) => {\n  const m = String(hhmm || '').match(/^(\\d{1,2}):(\\d{2})$/);\n  return m ? Number(m[1]) * 60 + Number(m[2]) : null;\n};\n\nconst coaches = Math.ceil(req.guests / COACH_SEATS);\n\nconst grounded = (value, text) => {\n  if (value === null || value === undefined) return true;\n  const t = String(text || '');\n  if (typeof value === 'number') {\n    return new RegExp('(^|[^0-9.,])' + String(value) + '([^0-9]|$)').test(t);\n  }\n  if (typeof value === 'string') {\n    return t.indexOf(value) !== -1;\n  }\n  return true;\n};\n\nconst assessed = replies.map((r, idx) => {\n  const e = extractions[idx] || {};\n  const rooms = (typeof e.roomsAvailable === 'number') ? e.roomsAvailable : null;\n  const accessible = (typeof e.accessibleRooms === 'number') ? e.accessibleRooms : null;\n  const rate = (typeof e.ratePerRoomEur === 'number') ? e.ratePerRoomEur : null;\n  const breakfastIncluded = (typeof e.breakfastIncluded === 'boolean') ? e.breakfastIncluded : null;\n  const breakfastPer = (typeof e.breakfastPerPersonEur === 'number') ? e.breakfastPerPersonEur : null;\n  const checkInFrom = e.checkInFrom || null;\n  const coach = (typeof e.coachAccess === 'boolean') ? e.coachAccess : r.coachParkingOnFile;\n  const distance = (typeof e.distanceKm === 'number') ? e.distanceKm : r.distanceKm;\n\n  const missing = [];\n  if (rooms === null) missing.push('rooms available');\n  if (rate === null) missing.push('rate per room in EUR');\n  if (accessible === null) missing.push('accessible rooms');\n  if (breakfastIncluded === null) missing.push('whether breakfast is included');\n  if (checkInFrom === null) missing.push('earliest check in time');\n\n  const fails = [];\n  if (rooms !== null && rooms < req.roomsRequired) fails.push('Covers ' + rooms + ' of ' + req.roomsRequired + ' rooms.');\n  if (accessible !== null && accessible < req.accessibleRoomsRequired) fails.push('Offers ' + accessible + ' accessible rooms, ' + req.accessibleRoomsRequired + ' are required.');\n  if (coach === false) fails.push('No coach parking on site. ' + coaches + ' coaches have to unload.');\n  const ci = toMinutes(checkInFrom);\n  if (ci !== null && ci > req.latestCheckInMinutes) fails.push('Check in opens at ' + checkInFrom + ', after the latest usable time of ' + req.latestCheckIn + '.');\n\n  let accommodation = null;\n  let breakfast = null;\n  let transport = null;\n  let total = null;\n  if (rooms !== null && rate !== null) {\n    accommodation = req.roomsRequired * rate;\n    breakfast = breakfastIncluded === false && breakfastPer !== null ? breakfastPer * req.guests : 0;\n    transport = coaches * (COACH_BASE_EUR + COACH_PER_KM_EUR * (distance || 0));\n    total = Math.round(accommodation + breakfast + transport);\n  }\n\n  const ungrounded = [];\n  const checks = [\n    ['rooms available', rooms], ['rate per room', rate], ['accessible rooms', accessible],\n    ['breakfast charge', breakfastPer], ['check in time', checkInFrom], ['offer deadline', e.offerDeadline || null]\n  ];\n  for (const c of checks) {\n    if (c[1] !== null && c[1] !== undefined && !grounded(c[1], r.replyText)) ungrounded.push(c[0] + ' (' + c[1] + ')');\n  }\n\n  let status = 'RANKED';\n  if (ungrounded.length > 0) status = 'NOT_GROUNDED';\n  else if (missing.length > 0) status = 'INCOMPLETE';\n  else if (fails.length > 0) status = 'REJECTED';\n  else if (total === null) status = 'INCOMPLETE';\n\n  return {\n    hotelId: r.hotelId,\n    name: r.name,\n    status: status,\n    distanceKm: distance,\n    roomsOffered: rooms,\n    accessibleOffered: accessible,\n    ratePerRoomEur: rate,\n    breakfastIncluded: breakfastIncluded,\n    breakfastPerPersonEur: breakfastPer,\n    checkInFrom: checkInFrom,\n    coachAccess: coach,\n    offerDeadline: e.offerDeadline || null,\n    accommodation: accommodation,\n    breakfast: breakfast,\n    transport: transport,\n    coaches: coaches,\n    totalEur: total,\n    missing: missing,\n    fails: fails,\n    ungrounded: ungrounded,\n    followUp: ungrounded.length > 0\n      ? 'The extraction produced values that do not appear in the reply: ' + ungrounded.join(', ') + '. Read this reply by hand.'\n      : (missing.length > 0 ? 'Call back and ask for: ' + missing.join(', ') + '.' : null)\n  };\n});\n\nconst ranked = assessed.filter((a) => a.status === 'RANKED').sort((a, b) => a.totalEur - b.totalEur);\nconst rejected = assessed.filter((a) => a.status === 'REJECTED');\nconst incomplete = assessed.filter((a) => a.status === 'INCOMPLETE');\nconst notGrounded = assessed.filter((a) => a.status === 'NOT_GROUNDED');\n\nconst best = ranked.length > 0 ? ranked[0] : null;\n\nlet decision;\nlet decisionReason;\nif (!best) {\n  decision = 'NO_USABLE_OFFER';\n  decisionReason = 'No single hotel meets the requirement. Chasing the incomplete replies is the next action, and a split across two hotels has to be considered by a person, not by this workflow.';\n} else if (best.totalEur <= req.stationCommitmentLimitEur) {\n  decision = 'STATION_MAY_COMMIT';\n  decisionReason = 'The cheapest usable offer is ' + best.totalEur + ' EUR against a station commitment limit of ' + req.stationCommitmentLimitEur + ' EUR.';\n} else {\n  decision = 'ESCALATE_TO_CARRIER';\n  decisionReason = 'The cheapest usable offer is ' + best.totalEur + ' EUR, above the station commitment limit of ' + req.stationCommitmentLimitEur + ' EUR. The station cannot commit this.';\n}\n\nconst notes = [\n  'Ranking is arithmetic in code, not a model judgement. The same offers always produce the same winner.',\n  'Coach cost is an assumption: ' + COACH_BASE_EUR + ' EUR per coach plus ' + COACH_PER_KM_EUR + ' EUR per km, ' + coaches + ' coaches for ' + req.guests + ' guests.',\n  'An incomplete reply is never ranked. Missing fields become a specific question, not a guess.',\n  'Every number the model extracts is checked against the reply text it came from. A value that does not appear in the source is treated as ungrounded and the offer is pulled out of the ranking for a human to read.',\n  'Splitting the requirement across two hotels purely to keep each piece under the commitment limit is limit avoidance. This workflow will not do it.'\n];\n\nreturn [{\n  json: {\n    incidentId: req.incidentId,\n    req: req,\n    decision: decision,\n    decisionReason: decisionReason,\n    best: best,\n    ranked: ranked,\n    rejected: rejected,\n    incomplete: incomplete,\n    notGrounded: notGrounded,\n    silent: silent,\n    sentCount: head._sentCount || 0,\n    repliedCount: head._repliedCount || 0,\n    notes: notes,\n    boundaries: req.boundaries\n  }\n}];\n"
      },
      "id": "55fdf94c-40cf-42dd-a532-e858c3c48275",
      "name": "Score And Rank Offers",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2320,
        400
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "={{ JSON.stringify({ timing: $('Triage Nearby Hotels').first().json.timing, coaches: $('Triage Nearby Hotels').first().json.coaches, hotels: $('Triage Nearby Hotels').first().json.hotels, singleHotelPossible: $('Triage Nearby Hotels').first().json.singleHotelPossible }) }}",
        "messages": {
          "messageValues": [
            {
              "message": "You brief a station duty manager in the first minutes after a cancellation, before any hotel has been called. You are given a JSON triage of airport hotels: distance, nominal rooms, accessible rooms, coach access, and the room rest each one leaves before the next morning departure. You are given hotel data only, never any passenger information. Write at most six short lines of plain text. Say which hotels to call first and why, name the constraint that matters most tonight (rest before the morning flight, capacity, or accessible rooms), and if no single hotel covers the requirement say so and that a split is a decision for a person. Never invent a number that is not in the JSON, never tell the manager to book or commit, and be explicit that these room counts are nominal and still have to be confirmed with the hotel. No greeting, no sign off, no markdown."
            }
          ]
        },
        "batching": {}
      },
      "id": "e2917546-5d2a-4e20-b018-28a6dcc3953d",
      "name": "Station Manager Overview",
      "type": "@n8n/n8n-nodes-langchain.chainLlm",
      "typeVersion": 1.9,
      "position": [
        2080,
        400
      ]
    },
    {
      "parameters": {
        "model": {
          "__rl": true,
          "mode": "list",
          "value": "gpt-5-mini",
          "cachedResultName": "gpt-5-mini"
        },
        "builtInTools": {},
        "options": {}
      },
      "id": "d0bdadb5-14bc-4a22-84b2-9b13bf689eca",
      "name": "Overview Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
      "typeVersion": 1.3,
      "position": [
        2080,
        640
      ],
      "credentials": {}
    },
    {
      "parameters": {
        "jsCode": "const d = $('Score And Rank Offers').first().json;\nconst triage = $('Triage Nearby Hotels').first().json;\nconst overview = String(($('Station Manager Overview').first().json || {}).text || '').trim();\nconst req = d.req;\n\nconst eur = (n) => (n === null || n === undefined) ? 'n/a' : Number(n).toLocaleString('de-DE') + ' EUR';\nconst esc = (s) => String(s === null || s === undefined ? '' : s).replace(/&/g, '&amp;').replace(/</g, '&lt;').replace(/>/g, '&gt;');\n\nconst ok = d.decision === 'STATION_MAY_COMMIT';\nconst colour = ok ? '#0f7b4f' : '#a15c00';\nconst bg = ok ? '#e8f5ee' : '#fdf3e2';\nconst title = d.decision === 'STATION_MAY_COMMIT' ? 'Station may commit'\n  : (d.decision === 'ESCALATE_TO_CARRIER' ? 'Carrier authorisation required' : 'No usable offer');\n\nconst rankRows = d.ranked.map((r, i) => {\n  const win = i === 0;\n  return '<tr style=\"background:' + (win ? '#f2f8f4' : '#fff') + '\">'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;font-weight:' + (win ? '600' : '400') + '\">' + (i + 1) + '. ' + esc(r.name) + (win ? ' <span style=\"color:#0f7b4f\">cheapest</span>' : '') + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right\">' + r.roomsOffered + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right\">' + r.accessibleOffered + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right\">' + eur(r.ratePerRoomEur) + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right\">' + eur(r.accommodation) + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right\">' + eur(r.breakfast) + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right\">' + eur(r.transport) + '</td>'\n    + '<td style=\"padding:8px 10px;border-bottom:1px solid #eee;text-align:right;font-weight:600\">' + eur(r.totalEur) + '</td></tr>';\n}).join('');\n\nconst outRow = (r, reason) => '<li style=\"margin:6px 0\"><strong>' + esc(r.name) + '</strong>. ' + esc(reason) + '</li>';\n\nconst rejectedHtml = d.rejected.length\n  ? '<h2 style=\"font-size:17px;margin:22px 0 8px\">Offers that were not ranked</h2><ul style=\"font-size:14px;margin:0;padding-left:20px\">'\n    + d.rejected.map((r) => outRow(r, r.fails.join(' '))).join('') + '</ul>'\n  : '';\n\nconst incompleteHtml = d.incomplete.length\n  ? '<h2 style=\"font-size:17px;margin:22px 0 8px\">Replies that did not say enough</h2><ul style=\"font-size:14px;margin:0;padding-left:20px\">'\n    + d.incomplete.map((r) => outRow(r, r.followUp || 'Not enough information to price the offer.')).join('') + '</ul>'\n  : '';\n\nconst notGroundedHtml = (d.notGrounded && d.notGrounded.length)\n  ? '<h2 style=\"font-size:17px;margin:22px 0 8px\">Extractions that did not match the reply</h2><ul style=\"font-size:14px;margin:0;padding-left:20px\">'\n    + d.notGrounded.map((r) => outRow(r, r.followUp || 'Values not found in the source text.')).join('') + '</ul>'\n  : '';\n\nconst silentHtml = d.silent.length\n  ? '<div style=\"font-size:14px;margin-top:10px;color:#6b6b70\">No reply before the deadline from: '\n    + d.silent.map((s) => esc(s.name)).join(', ') + '. Chase by phone.</div>'\n  : '';\n\nconst bestHtml = d.best\n  ? '<div style=\"font-size:14px;margin-top:4px\">' + esc(d.best.name) + ', ' + d.best.roomsOffered + ' rooms of which '\n    + d.best.accessibleOffered + ' accessible, ' + eur(d.best.ratePerRoomEur) + ' per room, check in from '\n    + esc(d.best.checkInFrom) + ', ' + d.best.distanceKm + ' km. Offer deadline ' + esc(d.best.offerDeadline || 'not stated') + '.</div>'\n  : '';\n\nconst html = '<div style=\"max-width:900px;margin:0 auto;padding:24px;font-family:-apple-system,Segoe UI,Roboto,Helvetica,Arial,sans-serif;color:#1c1c1e;line-height:1.5;text-align:left\">'\n  + '<div style=\"font-size:13px;letter-spacing:.08em;text-transform:uppercase;color:#6b6b70\">Crisis accommodation</div>'\n  + '<h1 style=\"font-size:26px;margin:6px 0 2px\">' + esc(d.incidentId) + '</h1>'\n  + '<div style=\"color:#6b6b70;font-size:14px;margin-bottom:18px\">Requirement ' + req.roomsRequired + ' rooms of which '\n  + req.accessibleRoomsRequired + ' accessible, for ' + req.guests + ' guests, everyone in a room by ' + esc(req.latestCheckIn)\n  + '. Requested from ' + d.sentCount + ' hotels, ' + d.repliedCount + ' replied.</div>'\n  + '<div style=\"padding:14px 16px;background:' + bg + ';border-left:3px solid ' + colour + ';margin-bottom:20px\">'\n  + '<div style=\"font-weight:600;color:' + colour + '\">' + title + '</div>'\n  + '<div style=\"font-size:14px;margin-top:4px\">' + esc(d.decisionReason) + '</div>' + bestHtml + '</div>'\n  + '<div style=\"margin:18px 0;padding:12px 14px;background:#f7f7f9;border-left:3px solid #29528f\">'\n  + '<div style=\"font-weight:600;margin-bottom:6px\">Before anyone was called: the shortlist</div>'\n  + '<div style=\"font-size:14px;margin-bottom:8px\">Next departure ' + esc(triage.timing.nextFlight) + '. Everyone in a room by ' + esc(triage.timing.allInRooms)\n  + ', back at the terminal by ' + esc(triage.timing.backAtTerminal) + '. ' + triage.coaches + ' coaches for ' + req.guests + ' guests.</div>'\n  + (overview ? '<div style=\"white-space:pre-wrap;font-size:14px;margin-bottom:10px\">' + esc(overview) + '</div>' : '')\n  + '<div style=\"overflow-x:auto\"><table style=\"width:100%;border-collapse:collapse;font-size:13px\">'\n  + '<tr><th style=\"text-align:left;padding:6px 8px;border-bottom:2px solid #1c1c1e\">Hotel</th>'\n  + '<th style=\"text-align:left;padding:6px 8px;border-bottom:2px solid #1c1c1e\"></th>'\n  + '<th style=\"text-align:right;padding:6px 8px;border-bottom:2px solid #1c1c1e\">Distance</th>'\n  + '<th style=\"text-align:right;padding:6px 8px;border-bottom:2px solid #1c1c1e\">Rest before ' + esc(triage.timing.nextFlight) + '</th>'\n  + '<th style=\"text-align:right;padding:6px 8px;border-bottom:2px solid #1c1c1e\">Rooms held</th>'\n  + '<th style=\"text-align:left;padding:6px 8px;border-bottom:2px solid #1c1c1e\">Call?</th></tr>'\n  + triage.hotels.map(function(h){\n      var tierColour = h.tier === 'CALL_FIRST' ? '#0f7b4f' : (h.tier === 'USABLE' ? '#0f7b4f' : (h.tier === 'PARTIAL' ? '#a15c00' : '#999'));\n      var tierText = h.tier === 'CALL_FIRST' ? 'call first' : (h.tier === 'USABLE' ? 'usable' : (h.tier === 'PARTIAL' ? 'partial, ' + esc(h.blockers[0]) : 'skip, ' + esc(h.blockers.join('; '))));\n      return '<tr><td style=\"padding:6px 8px;border-bottom:1px solid #eee;font-weight:600\">' + esc(h.name) + '</td>'\n        + '<td style=\"padding:6px 8px;border-bottom:1px solid #eee;color:#6b6b70\">' + (h.partner ? 'partner' : 'nearby') + '</td>'\n        + '<td style=\"padding:6px 8px;border-bottom:1px solid #eee;text-align:right\">' + h.distanceKm + ' km</td>'\n        + '<td style=\"padding:6px 8px;border-bottom:1px solid #eee;text-align:right\">' + esc(h.restLabel) + '</td>'\n        + '<td style=\"padding:6px 8px;border-bottom:1px solid #eee;text-align:right\">' + h.nominalRooms + '</td>'\n        + '<td style=\"padding:6px 8px;border-bottom:1px solid #eee;color:' + tierColour + '\">' + tierText + '</td></tr>';\n    }).join('')\n  + '</table></div>'\n  + '<div style=\"font-size:12px;color:#6b6b70;margin-top:6px\">Room counts are nominal from the partner list, not a live check. Rest is room time before the morning re-check-in, after coaches carry passengers out and back. The hotels marked call first were the ones contacted.</div></div>'\n  + '<h2 style=\"font-size:17px;margin:22px 0 8px\">Usable offers, cheapest first</h2>'\n  + (d.ranked.length\n      ? '<div style=\"overflow-x:auto\"><table style=\"width:100%;border-collapse:collapse;font-size:14px\">'\n        + '<tr><th style=\"text-align:left;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Hotel</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Rooms</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Accessible</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Rate</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Rooms cost</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Breakfast</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Coaches</th>'\n        + '<th style=\"text-align:right;padding:8px 10px;border-bottom:2px solid #1c1c1e\">Total</th></tr>'\n        + rankRows + '</table></div>'\n      : '<div style=\"font-size:14px\">Nothing was rankable.</div>')\n  + rejectedHtml + incompleteHtml + notGroundedHtml + silentHtml\n  + '<div style=\"margin:20px 0;padding:12px 14px;background:#eef3fb;border-left:3px solid #29528f\">'\n  + '<div style=\"font-weight:600;margin-bottom:6px\">How this number was reached</div>'\n  + '<ul style=\"margin:0;padding-left:18px;font-size:14px\">'\n  + d.notes.map((n) => '<li style=\"margin:4px 0\">' + esc(n) + '</li>').join('') + '</ul></div>'\n  + '<div style=\"margin:18px 0;padding:12px 14px;background:#f7f7f9;border-left:3px solid #6b6b70\">'\n  + '<div style=\"font-weight:600;margin-bottom:6px\">What this workflow is not allowed to do</div>'\n  + '<ul style=\"margin:0;padding-left:18px;font-size:14px\">'\n  + d.boundaries.map((b) => '<li style=\"margin:4px 0\">' + esc(b) + '</li>').join('') + '</ul></div>'\n  + '<div style=\"margin-top:26px;padding-top:14px;border-top:1px solid #eee;font-size:12px;color:#6b6b70\">'\n  + 'Simulated hotel replies. No message was sent to any real address. Unit assumptions for coach transport are unverified.</div></div>';\n\nreturn [{ json: { html: html, incidentId: d.incidentId, decision: d.decision, bestHotel: d.best ? d.best.name : null, bestTotalEur: d.best ? d.best.totalEur : null } }];\n"
      },
      "id": "621999d6-b833-4cbd-808a-54304ec3ebf2",
      "name": "Render Comparison",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        2560,
        400
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "combinator": "and",
          "conditions": [
            {
              "id": "c1",
              "operator": {
                "type": "string",
                "operation": "equals"
              },
              "leftValue": "={{ $('Normalise Requirement').first().json.req.source }}",
              "rightValue": "HANDOVER_FORM"
            }
          ]
        },
        "options": {}
      },
      "id": "a336aca6-0774-47ac-af8d-b2135c5f3a84",
      "name": "Came From The Form",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.3,
      "position": [
        2800,
        400
      ]
    },
    {
      "parameters": {
        "operation": "completion",
        "respondWith": "showText",
        "responseText": "={{ $json.html }}"
      },
      "id": "8b708852-d381-49ce-86fe-2e0747a4a46d",
      "name": "Show Comparison",
      "type": "n8n-nodes-base.form",
      "typeVersion": 2.5,
      "position": [
        3040,
        272
      ],
      "webhookId": "eca47d5e-5a84-4b45-b463-f55559f7e274"
    },
    {
      "parameters": {},
      "id": "ab0184e1-61e6-44e8-888e-b451cc1570c2",
      "name": "No Page On Manual Run",
      "type": "n8n-nodes-base.noOp",
      "typeVersion": 1,
      "position": [
        3040,
        496
      ]
    },
    {
      "parameters": {
        "method": "POST",
        "url": "REDACTED_SLACK_WEBHOOK_URL",
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={{ (() => { const d = $('Score And Rank Offers').first().json; const r = d.req || {}; const eur = (n) => (n===null||n===undefined) ? 'n/a' : Number(n).toLocaleString('de-DE') + ' EUR'; const decTxt = d.decision === 'STATION_MAY_COMMIT' ? ':white_check_mark: Station may commit'   : (d.decision === 'ESCALATE_TO_CARRIER' ? ':warning: Carrier authorisation needed' : ':grey_question: No usable single offer'); const b = d.best; const L = []; L.push(':hotel: *Accommodation options ready*  ' + d.incidentId); L.push('*Need:* ' + r.roomsRequired + ' rooms (' + r.accessibleRoomsRequired + ' accessible) for ' + r.guests + ' guests, all in by ' + r.latestCheckIn); L.push('*Decision:* ' + decTxt + '   (station limit ' + eur(r.stationCommitmentLimitEur) + ')'); const b = d.best; if (b) {   L.push('*Cheapest usable:* ' + b.name + '  —  ' + eur(b.totalEur));   L.push('    ' + b.roomsOffered + ' rooms, ' + b.accessibleOffered + ' accessible  ·  ' + eur(b.ratePerRoomEur) + '/room  ·  check-in ' + b.checkInFrom + '  ·  ' + b.distanceKm + ' km');   if (d.ranked && d.ranked[1]) L.push('*Runner-up:* ' + d.ranked[1].name + '  —  ' + eur(d.ranked[1].totalEur)); } else {   L.push('_No single hotel covered the requirement. Chase the incomplete replies or split across two, a person decides._'); } const extras = []; if (d.incomplete && d.incomplete.length) extras.push(d.incomplete.length + ' incomplete'); if (d.silent && d.silent.length) extras.push(d.silent.length + ' no reply'); L.push('*Sourcing:* asked ' + d.sentCount + ' hotels, ' + d.repliedCount + ' replied' + (extras.length ? ' (' + extras.join(', ') + ')' : '') + '.'); L.push('_Ranked shortlist, not a booking. A person commits._'); return JSON.stringify({ text: L.join('\\n') }); })() }}",
        "options": {}
      },
      "id": "806cbf12-66b1-4a0e-a96a-7bb1af0e6849",
      "name": "Post Ranked Hotels To Slack",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.4,
      "position": [
        2560,
        672
      ],
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "formTitle": "Crisis accommodation",
        "formDescription": "Handover from the cancellation exposure model. Room counts and times only. No passenger data is accepted here and none is needed.",
        "formFields": {
          "values": [
            {
              "fieldLabel": "Incident reference",
              "fieldName": "incidentId",
              "placeholder": "INC-LH1234-2026-08-29",
              "requiredField": true
            },
            {
              "fieldLabel": "Rooms required including crew",
              "fieldType": "number",
              "fieldName": "roomsRequired",
              "defaultValue": "89",
              "requiredField": true
            },
            {
              "fieldLabel": "Of which accessible",
              "fieldType": "number",
              "fieldName": "accessibleRoomsRequired",
              "defaultValue": "4",
              "requiredField": true
            },
            {
              "fieldLabel": "Guests to move and feed",
              "fieldType": "number",
              "fieldName": "guests",
              "defaultValue": "121",
              "requiredField": true
            },
            {
              "fieldLabel": "Everyone in a room by (HH:MM)",
              "fieldName": "latestCheckIn",
              "defaultValue": "23:30",
              "requiredField": true
            },
            {
              "fieldLabel": "Next departure these passengers are on (HH:MM next morning)",
              "fieldName": "nextFlightDeparture",
              "defaultValue": "07:00",
              "requiredField": true
            },
            {
              "fieldLabel": "Care ceiling from the exposure model in EUR",
              "fieldType": "number",
              "fieldName": "careCeilingEur",
              "defaultValue": "23498"
            },
            {
              "fieldLabel": "Station commitment limit in EUR",
              "fieldType": "number",
              "fieldName": "stationCommitmentLimitEur",
              "defaultValue": "25000",
              "requiredField": true
            }
          ]
        },
        "responseMode": "lastNode",
        "options": {
          "buttonLabel": "Source accommodation"
        }
      },
      "id": "f40dbd75-77a2-4952-b910-acb8510624e7",
      "name": "Accommodation Request",
      "type": "n8n-nodes-base.formTrigger",
      "typeVersion": 2.5,
      "position": [
        368,
        560
      ],
      "webhookId": "996df65f-fe58-45f7-83af-8322efaf7c7c"
    },
    {
      "parameters": {
        "inputSource": "passthrough"
      },
      "id": "e9ac81f7-e588-468a-9727-bff4ed4a6090",
      "name": "Called By IRREG",
      "type": "n8n-nodes-base.executeWorkflowTrigger",
      "typeVersion": 1.1,
      "position": [
        368,
        896
      ]
    }
  ],
  "connections": {
    "Run With Fixture Requirement": {
      "main": [
        [
          {
            "node": "Normalise Requirement",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Normalise Requirement": {
      "main": [
        [
          {
            "node": "Triage Nearby Hotels",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Triage Nearby Hotels": {
      "main": [
        [
          {
            "node": "Draft The Request To Hotels",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Draft The Request To Hotels": {
      "main": [
        [
          {
            "node": "Supplier List",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Drafting Model": {
      "ai_languageModel": [
        [
          {
            "node": "Draft The Request To Hotels",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Supplier List": {
      "main": [
        [
          {
            "node": "Dispatch Requests (Simulated)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Dispatch Requests (Simulated)": {
      "main": [
        [
          {
            "node": "Wait For The Reply Deadline",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Wait For The Reply Deadline": {
      "main": [
        [
          {
            "node": "Collect Replies",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Collect Replies": {
      "main": [
        [
          {
            "node": "Extract Hotel Offer",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Extract Hotel Offer": {
      "main": [
        [
          {
            "node": "Score And Rank Offers",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Extraction Model": {
      "ai_languageModel": [
        [
          {
            "node": "Extract Hotel Offer",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Score And Rank Offers": {
      "main": [
        [
          {
            "node": "Station Manager Overview",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Station Manager Overview": {
      "main": [
        [
          {
            "node": "Render Comparison",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Overview Model": {
      "ai_languageModel": [
        [
          {
            "node": "Station Manager Overview",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "Render Comparison": {
      "main": [
        [
          {
            "node": "Came From The Form",
            "type": "main",
            "index": 0
          },
          {
            "node": "Post Ranked Hotels To Slack",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Came From The Form": {
      "main": [
        [
          {
            "node": "Show Comparison",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "No Page On Manual Run",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Accommodation Request": {
      "main": [
        [
          {
            "node": "Normalise Requirement",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Called By IRREG": {
      "main": [
        [
          {
            "node": "Normalise Requirement",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "meta": {
    "aiBuilderAssisted": true,
    "instanceId": "00cafa1eb8a383ac595b6bbf54c5f8acd05d9d1462911bd4a0edac031509b829"
  }
}