# Ripple — Integration Diagram

This document is the contract between all workstreams. If you need to change anything in here, tell Michael first.

---

## System Flow

```
Flight API
    │
    ▼
Agent 1: Flight Monitor ──writes──► disruptions table
                                          │
                                          ▼
                                 Agent 2: Booking Scanner ──reads──► bookings table
                                                                           │
                                                                           ▼
                                                                  Agent 3: Operator Notifier ──writes──► operator_alerts table
                                                                                                              │
                                                                           ┌──────────────────────────────────┘
                                                                           ▼
                                                              Operator confirms / declines via portal
                                                                           │
                                                                           ▼
                                                                  Agent 4: Availability Checker (if declined)
                                                                           │
                                                                           ▼
                                                                  Agent 5: Guest Communicator ──writes──► chat_messages table
                                                                                              ──updates──► bookings.status
```

---

## Agent I/O Contracts

### Agent 1 — Flight Monitor (Vedang)
- **Trigger:** Polls AviationStack API every 60s for flight `AA 204`
- **Input:** `{ flight_number: "AA 204", date: "2026-07-02" }`
- **Writes to Supabase:**
  ```
  disruptions: {
    trip_id,
    type: "flight_delay",
    delay_minutes: <int>,
    detected_at: <now>,
    raw_data: <full API response>,
    resolved: false
  }
  ```
- **Also updates:** `trips.status = "disrupted"` for the affected trip

---

### Agent 2 — Booking Scanner (Vedang)
- **Trigger:** New row inserted in `disruptions`
- **Input:** `disruption.id`, `disruption.trip_id`
- **Reads from Supabase:** All `bookings` where `trip_id` matches and `type != "flight"`
- **Output (passed to Agent 3):**
  ```
  [
    { booking_id, operator_id, type, scheduled_time, operator_name, operator_whatsapp }
  ]
  ```
- **Does not write to Supabase** — passes data directly to Agent 3 in the pipeline

---

### Agent 3 — Operator Notifier (Vedang)
- **Trigger:** Agent 2 output
- **Input:** Array of affected bookings from Agent 2 + `delay_minutes` from disruption
- **Writes to Supabase:**
  ```
  operator_alerts: {
    disruption_id,
    operator_id,
    booking_id,
    message: <Claude-generated text>,
    status: "pending"
  }
  ```
- **Also sends:** WhatsApp message to `operator.whatsapp_number` via WhatsApp Business API
- **Message tone:** Professional. Example: "Hi Bob's Taxis, your guest Maria Johnson's flight AA 204 is delayed by 4 hours. New arrival time is 18:00. Can you accommodate a pickup at 18:30?"

---

### Agent 4 — Availability Checker (Vedang)
- **Trigger:** `operator_alerts.status` updated to `"declined"`
- **Input:** `{ booking_id, operator_type, original_time, delay_minutes }`
- **Logic:** Finds alternative operator of same type with `is_available = true`, proposes new time
- **Writes to Supabase:**
  ```
  operator_alerts: { <new row for alternative operator, status: "pending"> }
  bookings: { status: "pending" }  // while seeking alternative
  ```

---

### Agent 5 — Guest Communicator (Vedang)
- **Trigger:** `operator_alerts.status` updated to `"confirmed"` or `"declined"`
- **Input:** `{ trip_id, booking_id, operator_name, operator_type, new_status, updated_time }`
- **Writes to Supabase:**
  ```
  chat_messages: {
    trip_id,
    sender: "ai",
    body: <Claude-generated message>
  }
  bookings: {
    status: "rescheduled" | "cancelled",
    updated_time: <new time if rescheduled>
  }
  ```
- **Message tone:** Warm, reassuring. Example: "Great news! Bob's Taxis has confirmed your pickup at 18:30. You're all set."
- **Final message when all resolved:** Sets `disruptions.resolved = true`

---

## API Endpoints (Shamar builds these)

| Method | Path | Description | Who calls it |
|---|---|---|---|
| `GET` | `/api/trip/:id` | Full trip with all bookings | Dayna (guest app) |
| `GET` | `/api/trip/:id/chat` | All chat messages for a trip | Dayna (guest app) |
| `POST` | `/api/bookings/:id/cancel` | Guest cancels a booking | Dayna (guest app) |
| `POST` | `/api/bookings/:id/reschedule` | Guest reschedules with new time | Dayna (guest app) |
| `GET` | `/api/operator/:id/bookings` | Operator's active bookings | Bhavesh (operator portal) |
| `GET` | `/api/operator/:id/alerts` | Pending alerts for an operator | Bhavesh (operator portal) |
| `POST` | `/api/operator-alerts/:id/confirm` | Operator confirms a change | Bhavesh (operator portal) |
| `POST` | `/api/operator-alerts/:id/decline` | Operator can't do it | Bhavesh (operator portal) |
| `POST` | `/api/demo/reset` | Reset all demo data to clean state | Bhavesh (demo reset script) |
| `POST` | `/api/trigger/disruption` | Manually fire the Maria scenario | Michael (demo trigger) |

---

## Realtime Channels (Bhavesh wires these, Dayna subscribes)

Use these **exact channel names** — both sides must match.

| Channel | Table | Event | Who subscribes | Who triggers |
|---|---|---|---|---|
| `trip:{trip_id}:bookings` | `bookings` | `UPDATE` | Dayna (guest app) | Vedang (agents) / Shamar (API) |
| `trip:{trip_id}:chat` | `chat_messages` | `INSERT` | Dayna (guest app) | Vedang (Agent 5) |
| `trip:{trip_id}:disruption` | `disruptions` | `INSERT` | Dayna (guest app) | Vedang (Agent 1) |
| `operator:{operator_id}:alerts` | `operator_alerts` | `INSERT` | Bhavesh (operator portal) | Vedang (Agent 3) |

### How to subscribe (Dayna / Bhavesh)
```ts
const channel = supabase
  .channel(`trip:${tripId}:bookings`)
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'bookings',
    filter: `trip_id=eq.${tripId}`
  }, (payload) => {
    // update UI with payload.new
  })
  .subscribe()
```

---

## External APIs (Shamar integrates)

| API | Purpose | Key env var |
|---|---|---|
| AviationStack | Flight status polling | `AVIATIONSTACK_KEY` |
| WhatsApp Business API | Operator notifications | `WHATSAPP_TOKEN`, `WHATSAPP_PHONE_ID` |

---

## Who Owns What

| Area | Owner |
|---|---|
| Supabase schema + migrations | Shamar |
| API routes (`/api/*`) | Shamar |
| Realtime channel setup in Supabase | Shamar |
| LangGraph agent pipeline | Vedang |
| Guest app (`apps/frontend`) | Dayna |
| Operator portal (`apps/operator`) | Bhavesh |
| Integration glue (frontend ↔ backend) | Bhavesh |
| Architecture + review | Michael |

Any change to this document must go through Michael.
