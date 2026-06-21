# Ripple — Integration Diagram
> Single source of truth for how every workstream connects. If it's not in this doc, it hasn't been agreed. Nothing changes here without Michael's sign-off.

---

## System Flow

```
Flight API
    │
    ▼
Agent 1: Flight Monitor ──writes──► disruptions table
                                          │
                          Realtime fires  ▼
                                 Agent 2: Booking Scanner ──reads──► bookings table
                                                                           │
                                                                           ▼
                                                                  Agent 3: Operator Notifier ──writes──► operator_alerts table
                                                                                             ──sends───► WhatsApp Business API
                                                                                                              │
                                                                           ┌──────────────────────────────────┘
                                                                           │  Realtime fires → Operator portal
                                                                           ▼
                                                              Operator confirms / declines via portal
                                                                           │
                                                          ┌────────────────┴──────────────────┐
                                                          ▼                                   ▼
                                               Agent 5: Guest Communicator        Agent 4: Availability Checker
                                               ──writes──► chat_messages                  (if declined)
                                               ──updates──► bookings.status
```

---

## 1. What Each Agent Reads From / Writes To Supabase

Vedang owns all 5 agents. Every agent has a defined input (what it reads) and output (what it writes). No agent writes to a table not listed here.

---

### Agent 1 — Flight Monitor

**Trigger:** Scheduled poll every 2 minutes against the flight status API (or manual trigger for demo)

**Reads:**
- `bookings` — all rows where `type = 'flight'` and `status = 'confirmed'`
- `bookings.metadata` — extracts `flight_number` to query the external flight API

**Writes:**
- `disruptions` — inserts a new row when a delay is detected:
```json
{
  "trip_id": "<from booking>",
  "booking_id": "<flight booking id>",
  "type": "flight_delay",
  "severity": "high",
  "raw_data": {
    "flight_number": "AA204",
    "original_departure": "2026-07-02T14:30:00Z",
    "updated_departure": "2026-07-02T18:30:00Z",
    "delay_minutes": 240,
    "reason": "Late incoming aircraft"
  },
  "resolved": false
}
```
- `bookings` — updates the flight booking row: `status = 'ai_handling'`

**What happens next:** Inserting into `disruptions` fires Supabase Realtime → Agent 2 picks it up.

---

### Agent 2 — Booking Scanner

**Trigger:** Supabase Realtime event on `disruptions` table — new row inserted

**Reads:**
- `disruptions` — the newly inserted disruption row
- `bookings` — all bookings for the same `trip_id` where `type != 'flight'` and `status != 'cancelled'`
- `operators` — reads `whatsapp_number` and `is_available` for each affected operator

**Writes:**
- `bookings` — updates all affected bookings: `status = 'ai_handling'`
- `chat_messages` — inserts first AI message to guest:
```json
{
  "trip_id": "<trip_id>",
  "sender": "ai",
  "body": "Hi Maria, I've detected that flight AA204 is delayed by 4 hours. I'm now coordinating your transfer, hotel, tour, and dinner — you don't need to do anything.",
  "message_type": "update"
}
```

**What happens next:** Agent 3 runs immediately after Agent 2 completes.

---

### Agent 3 — Operator Notifier

**Trigger:** Called directly by Agent 2 after it scans affected bookings

**Reads:**
- `bookings` — the list of affected bookings (passed from Agent 2)
- `operators` — name, type, whatsapp_number for each operator
- `disruptions` — the disruption details for message generation

**Writes:**
- `operator_alerts` — one row per affected operator:
```json
{
  "disruption_id": "<disruption_id>",
  "operator_id": "<operator_id>",
  "booking_id": "<booking_id>",
  "message": "Hi Bob's Taxis, your guest Maria Johnson's flight AA204 has been delayed by 4 hours. Her new estimated arrival at MBJ is 6:45pm. Can you accommodate a new pickup time of 7:30pm?",
  "proposed_time": "2026-07-02T23:30:00Z",
  "status": "pending"
}
```
- Also sends WhatsApp message via Shamar's `/api/whatsapp/send` endpoint (see API section)

**What happens next:** Supabase Realtime on `operator_alerts` fires → Bhavesh's operator portal shows the alert to the operator.

---

### Agent 4 — Availability Checker

**Trigger:** Supabase Realtime event on `operator_alerts` where `status` changes to `'declined'`

**Reads:**
- `operator_alerts` — the declined alert row
- `bookings` — the affected booking
- `operators` — finds alternative operators of the same `type` on the same `island` where `is_available = true`

**Writes:**
- `operator_alerts` — inserts a new alert row for the alternative operator (same structure as Agent 3 output)
- `chat_messages` — inserts an update to the guest:
```json
{
  "sender": "ai",
  "body": "Bob's Taxis wasn't available for the new time. I've found an alternative transfer — Island Express. They've been notified.",
  "message_type": "update"
}
```

---

### Agent 5 — Guest Communicator

**Trigger:** Supabase Realtime event on `operator_alerts` where `status` changes to `'confirmed'`

**Reads:**
- `operator_alerts` — the confirmed alert row
- `bookings` — the rescheduled booking details
- `operators` — operator name for the message

**Writes:**
- `bookings` — updates the confirmed booking:
  - `status = 'rescheduled'`
  - `updated_time = <proposed_time from operator_alert>`
- `chat_messages` — inserts a confirmation message to the guest:
```json
{
  "sender": "ai",
  "body": "✓ Bob's Taxis confirmed your new pickup time: 7:30pm at MBJ Airport.",
  "message_type": "confirmation",
  "metadata": { "booking_id": "<booking_id>", "operator_name": "Bob's Taxis" }
}
```
- When ALL operator_alerts for a disruption are `confirmed` or `declined+resolved`:
  - Updates `disruptions.resolved = true`, `disruptions.resolved_at = now()`
  - Inserts final chat message: `"All sorted! Your trip is back on track. No calls needed. 🎉"`

---

## 2. Supabase Realtime Channels

Bhavesh sets these up exactly as named. Dayna and Vedang subscribe to these exact channel names — no variation.

| Channel Name | Table | Event | Filter | Who Subscribes |
|---|---|---|---|---|
| `bookings:trip:{trip_id}` | `bookings` | `UPDATE` | `trip_id=eq.{trip_id}` | Dayna (guest itinerary UI) |
| `operator_alerts:operator:{operator_id}` | `operator_alerts` | `INSERT` | `operator_id=eq.{operator_id}` | Bhavesh (operator alert screen) |
| `operator_alerts:disruption:{disruption_id}` | `operator_alerts` | `UPDATE` | `disruption_id=eq.{disruption_id}` | Dayna (AI coordination progress screen) + Vedang (Agent 4 & 5 trigger) |
| `chat_messages:trip:{trip_id}` | `chat_messages` | `INSERT` | `trip_id=eq.{trip_id}` | Dayna (chat feed) |
| `disruptions:trip:{trip_id}` | `disruptions` | `INSERT` | `trip_id=eq.{trip_id}` | Dayna (disruption alert screen trigger) + Vedang (Agent 2 trigger) |

**Setup in Supabase:**
Enable Realtime on these tables in the Supabase dashboard (Database → Replication):
- `bookings` ✓
- `operator_alerts` ✓
- `chat_messages` ✓
- `disruptions` ✓

---

## 3. API Endpoints — Shamar Builds These

All endpoints return JSON. All errors return `{ "error": "message" }` with appropriate HTTP status code. All authenticated routes require a valid Supabase JWT in the `Authorization: Bearer <token>` header.

---

### Auth
Handled by Supabase Auth — no custom auth endpoints needed. Frontend uses Supabase client `signInWithPassword()` and `signUp()`.

---

### Guest Endpoints

**Get full trip with all bookings**
```
GET /api/trips/:trip_id
Auth: required (guest)
Returns: trip row + all bookings array + active disruption if any
```

**Get all trips for current user**
```
GET /api/trips
Auth: required (guest)
Returns: array of trips for the authenticated user
```

**Cancel a booking**
```
POST /api/bookings/:booking_id/cancel
Auth: required (guest)
Body: none
Returns: { "success": true, "booking": <updated row> }
Side effect: Updates booking.status = 'cancelled'. Sends notification to operator via WhatsApp.
```

**Reschedule a booking**
```
POST /api/bookings/:booking_id/reschedule
Auth: required (guest)
Body: { "new_time": "2026-07-03T16:00:00Z" }
Returns: { "success": true, "booking": <updated row> }
Side effect: Sets booking.updated_time, creates operator_alert with proposed_time.
```

**Get chat messages for a trip**
```
GET /api/trips/:trip_id/messages
Auth: required (guest)
Returns: array of chat_messages ordered by created_at ASC
```

**Send a chat message (guest)**
```
POST /api/trips/:trip_id/messages
Auth: required (guest)
Body: { "body": "Can you check if the 4pm slot is available?" }
Returns: { "success": true, "message": <inserted row> }
Side effect: Inserts with sender='guest', message_type='text'. Triggers Agent 5 if action-relevant.
```

---

### Operator Endpoints

**Get today's bookings for operator**
```
GET /api/operator/bookings
Auth: required (operator)
Query params: ?date=2026-07-02 (defaults to today)
Returns: array of bookings for this operator on the given date, with guest name included
```

**Get pending alerts for operator**
```
GET /api/operator/alerts
Auth: required (operator)
Returns: array of operator_alerts where status='pending' for this operator
```

**Confirm an alert (operator accepts new time)**
```
POST /api/operator/alerts/:alert_id/confirm
Auth: required (operator)
Body: none
Returns: { "success": true }
Side effect: Sets operator_alert.status='confirmed', responded_at=now(). Triggers Agent 5.
```

**Decline an alert (operator can't do it)**
```
POST /api/operator/alerts/:alert_id/decline
Auth: required (operator)
Body: none
Returns: { "success": true }
Side effect: Sets operator_alert.status='declined', responded_at=now(). Triggers Agent 4.
```

**Toggle operator online status**
```
POST /api/operator/status
Auth: required (operator)
Body: { "is_online": true }
Returns: { "success": true }
Side effect: Updates operators.is_online for the authenticated operator.
```

---

### Internal / Agent Endpoints

**Trigger disruption manually (demo use)**
```
POST /api/internal/trigger-disruption
Auth: service role key only
Body: {
  "trip_id": "...",
  "booking_id": "...",
  "type": "flight_delay",
  "delay_minutes": 240,
  "flight_number": "AA204"
}
Returns: { "success": true, "disruption_id": "..." }
Side effect: Inserts into disruptions table, which fires Agent 2 via Realtime.
```

**Send WhatsApp message**
```
POST /api/whatsapp/send
Auth: service role key only
Body: {
  "to": "+18769876543",
  "message": "Hi Bob's Taxis, your guest Maria Johnson..."
}
Returns: { "success": true, "message_id": "..." }
Side effect: Calls WhatsApp Business API.
```

**Reset demo data**
```
POST /api/internal/reset-demo
Auth: service role key only
Body: none
Returns: { "success": true }
Side effect: Runs the reset SQL script — clears disruptions, alerts, chat messages, resets booking statuses.
```

---

## 4. What Dayna's Frontend Subscribes To

Every Realtime subscription Dayna needs, with what UI change each one triggers.

---

### Dashboard / Home Screen
No Realtime subscriptions. Loads data once on mount via `GET /api/trips`.
Refresh on pull-to-refresh.

---

### Active Trip View
```javascript
supabase
  .channel(`bookings:trip:${tripId}`)
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'bookings',
    filter: `trip_id=eq.${tripId}`
  }, (payload) => {
    // Update the booking card status badge for payload.new.id
    // If status = 'ai_handling' → show gold "AI Handling" badge with pulse
    // If status = 'rescheduled' → show blue "Rescheduled" badge + updated_time
    // If status = 'cancelled' → show red "Cancelled" badge
  })
  .subscribe()
```

---

### Disruption Alert Screen (trigger)
```javascript
// Subscribe on Dashboard mount — this is what opens the alert screen
supabase
  .channel(`disruptions:trip:${tripId}`)
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'disruptions',
    filter: `trip_id=eq.${tripId}`
  }, (payload) => {
    // Navigate to / show the Disruption Alert screen
    // Pass payload.new as the disruption data
  })
  .subscribe()
```

---

### AI Coordination Progress Screen
```javascript
supabase
  .channel(`operator_alerts:disruption:${disruptionId}`)
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'operator_alerts',
    filter: `disruption_id=eq.${disruptionId}`
  }, (payload) => {
    // Find the alert row for payload.new.id
    // If payload.new.status = 'confirmed' → animate row to green checkmark
    // If payload.new.status = 'declined' → animate row to grey "Finding alternative..."
    // Update the "X of 4 resolved" counter
    // If all resolved → trigger success state animation
  })
  .subscribe()
```

---

### Chat Feed Screen
```javascript
supabase
  .channel(`chat_messages:trip:${tripId}`)
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'chat_messages',
    filter: `trip_id=eq.${tripId}`
  }, (payload) => {
    // Append new message bubble to the thread
    // If payload.new.sender = 'ai' → left-aligned blue bubble
    // If payload.new.message_type = 'confirmation' → show booking name + green tick
    // Scroll to bottom of thread
  })
  .subscribe()
```

---

## 5. External APIs

| API | Purpose | Key env var | Owner |
|---|---|---|---|
| AviationStack | Flight status polling for Agent 1 | `AVIATIONSTACK_KEY` | Shamar |
| WhatsApp Business API | Operator notifications via Agent 3 | `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_ID` | Shamar |
| Claude API (Anthropic) | Message generation in Agents 3 & 5 | `ANTHROPIC_API_KEY` | Vedang |

---

## 6. Who Owns What

| Area | Owner |
|---|---|
| Supabase schema + migrations | Shamar |
| API routes (`/api/*`) | Shamar |
| External API integrations (flight, WhatsApp) | Shamar |
| Realtime channel setup in Supabase | Shamar |
| LangGraph agent pipeline | Vedang |
| Claude prompt engineering | Vedang |
| Guest app (`apps/frontend`) | Dayna |
| Operator portal (`apps/operator`) | Bhavesh |
| Integration glue (frontend ↔ backend) | Bhavesh |
| Demo reset script | Bhavesh |
| Architecture + review | Michael |

---

## Event Flow — Full Maria Scenario

Exact sequence from flight delay detection to "All sorted":

```
1.  Agent 1 polls flight API → detects AA204 delayed 4 hours
2.  Agent 1 → INSERT into disruptions → UPDATE bookings (flight status = 'ai_handling')
3.  Realtime fires → Dayna's Dashboard receives disruption INSERT → shows Disruption Alert screen
4.  Agent 2 triggered → reads all 4 affected bookings → UPDATE bookings (all status = 'ai_handling')
5.  Realtime fires → Dayna's Active Trip View updates all 4 badge statuses to "AI Handling"
6.  Agent 2 → INSERT chat_message ("Hi Maria, I've detected your flight is delayed...")
7.  Realtime fires → Dayna's Chat Feed appends AI message
8.  Agent 3 → INSERT 4 rows into operator_alerts (status = 'pending')
9.  Agent 3 → POST /api/whatsapp/send × 4 (one per operator)
10. Realtime fires → Bhavesh's Operator Portal shows alert screen to Bob's Taxis
11. Bob's Taxis taps "Confirm New Time" → POST /api/operator/alerts/:id/confirm
12. Shamar's API → UPDATE operator_alerts (status = 'confirmed')
13. Realtime fires → Dayna's Coordination Progress screen animates Bob's Taxis to green ✓
14. Realtime fires → Vedang's Agent 5 triggered
15. Agent 5 → UPDATE bookings (transfer status = 'rescheduled', updated_time = '7:30pm')
16. Agent 5 → INSERT chat_message ("✓ Bob's Taxis confirmed your new pickup: 7:30pm")
17. Realtime fires → Dayna's Chat Feed appends confirmation message
18. Steps 10–17 repeat for remaining 3 operators
19. Agent 5 detects all 4 resolved → UPDATE disruptions (resolved = true)
20. Agent 5 → INSERT chat_message ("All sorted! Your trip is back on track 🎉")
21. Dayna's Coordination Progress screen triggers success state animation
```

Total expected time end-to-end: **under 3 minutes for demo**.
