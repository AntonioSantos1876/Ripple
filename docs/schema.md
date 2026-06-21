# Ripple — Database Schema
**Supabase / PostgreSQL**
> This is the single source of truth for all tables. Shamar runs migrations from this. Vedang writes agent outputs against this. Dayna and Bhavesh build API calls against this. Nothing changes here without Michael's sign-off.

Supabase project: `https://fhzqfmseiflzdqtbaogm.supabase.co`

---

## Tables Overview

```
users
trips
bookings
operators
disruptions
operator_alerts
chat_messages
```

---

## Table: `users`

Stores both guests and operators. Role field determines which portal they access.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique user ID |
| `name` | `text` | NOT NULL | Full name |
| `email` | `text` | UNIQUE, NOT NULL | Login email |
| `phone` | `text` | NULLABLE | WhatsApp-capable phone number (E.164 format e.g. +18761234567) |
| `role` | `text` | NOT NULL, CHECK IN ('guest', 'operator') | Determines which portal the user sees |
| `created_at` | `timestamptz` | NOT NULL, default `now()` | Account creation timestamp |

**Relationships:**
- `users.id` ← referenced by `trips.guest_id`
- `users.id` ← referenced by `operators.user_id` (for operator login)

**Notes:**
- Guests have `role = 'guest'`
- Operators have `role = 'operator'` and a linked row in the `operators` table
- Phone must be in E.164 format for WhatsApp Business API to work

**SQL:**
```sql
create table users (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  email text unique not null,
  phone text,
  role text not null check (role in ('guest', 'operator')),
  created_at timestamptz not null default now()
);
```

---

## Table: `trips`

One trip per guest visit. A trip contains multiple bookings.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique trip ID |
| `guest_id` | `uuid` | NOT NULL, FK → `users.id` | The guest this trip belongs to |
| `name` | `text` | NOT NULL | Trip display name e.g. "Jamaica · July 2026" |
| `destination` | `text` | NOT NULL | Island/location name |
| `start_date` | `date` | NOT NULL | Trip start date |
| `end_date` | `date` | NOT NULL | Trip end date |
| `status` | `text` | NOT NULL, CHECK IN ('active', 'completed', 'cancelled'), default `'active'` | Current trip status |
| `created_at` | `timestamptz` | NOT NULL, default `now()` | Record creation timestamp |

**Relationships:**
- `trips.guest_id` → `users.id`
- `trips.id` ← referenced by `bookings.trip_id`
- `trips.id` ← referenced by `disruptions.trip_id`
- `trips.id` ← referenced by `chat_messages.trip_id`

**SQL:**
```sql
create table trips (
  id uuid primary key default gen_random_uuid(),
  guest_id uuid not null references users(id) on delete cascade,
  name text not null,
  destination text not null,
  start_date date not null,
  end_date date not null,
  status text not null default 'active' check (status in ('active', 'completed', 'cancelled')),
  created_at timestamptz not null default now()
);
```

---

## Table: `operators`

All service providers on the platform — transfers, hotels, tours, restaurants.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique operator ID |
| `user_id` | `uuid` | NULLABLE, FK → `users.id` | Linked user account for portal login |
| `name` | `text` | NOT NULL | Business name e.g. "Bob's Taxis" |
| `type` | `text` | NOT NULL, CHECK IN ('transfer', 'hotel', 'tour', 'restaurant') | Operator category |
| `whatsapp_number` | `text` | NULLABLE | WhatsApp number for AI notifications (E.164) |
| `email` | `text` | NULLABLE | Email for fallback notifications |
| `island` | `text` | NOT NULL | Island they operate on e.g. "Jamaica" |
| `is_available` | `boolean` | NOT NULL, default `true` | Whether they're currently accepting bookings |
| `is_online` | `boolean` | NOT NULL, default `false` | Whether they're active in the operator portal right now |
| `created_at` | `timestamptz` | NOT NULL, default `now()` | Record creation timestamp |

**Relationships:**
- `operators.user_id` → `users.id`
- `operators.id` ← referenced by `bookings.operator_id`
- `operators.id` ← referenced by `operator_alerts.operator_id`

**Notes:**
- `is_available` = long-term availability (toggle in settings)
- `is_online` = real-time portal presence (toggled from operator dashboard)
- An operator can exist without a `user_id` if they're only reachable via WhatsApp

**SQL:**
```sql
create table operators (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references users(id) on delete set null,
  name text not null,
  type text not null check (type in ('transfer', 'hotel', 'tour', 'restaurant')),
  whatsapp_number text,
  email text,
  island text not null,
  is_available boolean not null default true,
  is_online boolean not null default false,
  created_at timestamptz not null default now()
);
```

---

## Table: `bookings`

Every individual service booking within a trip — one row per service.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique booking ID |
| `trip_id` | `uuid` | NOT NULL, FK → `trips.id` | The trip this booking belongs to |
| `operator_id` | `uuid` | NULLABLE, FK → `operators.id` | The operator fulfilling this booking |
| `type` | `text` | NOT NULL, CHECK IN ('flight', 'transfer', 'hotel', 'tour', 'restaurant') | Booking category |
| `status` | `text` | NOT NULL, CHECK IN ('confirmed', 'rescheduled', 'cancelled', 'pending', 'ai_handling'), default `'confirmed'` | Current booking status |
| `reference` | `text` | NULLABLE | External booking reference number |
| `scheduled_time` | `timestamptz` | NOT NULL | Original scheduled date and time |
| `updated_time` | `timestamptz` | NULLABLE | New time after rescheduling (set by AI agent) |
| `notes` | `text` | NULLABLE | Any additional info e.g. flight number, room type |
| `metadata` | `jsonb` | NULLABLE | Flexible field for type-specific data (e.g. flight number, terminal, seat) |
| `created_at` | `timestamptz` | NOT NULL, default `now()` | Record creation timestamp |

**Relationships:**
- `bookings.trip_id` → `trips.id`
- `bookings.operator_id` → `operators.id`
- `bookings.id` ← referenced by `operator_alerts.booking_id`

**Notes:**
- `status = 'ai_handling'` means an agent is actively working on this booking — guest UI shows the gold "AI Handling" badge
- `metadata` jsonb examples:
  - Flight: `{ "flight_number": "AA204", "origin": "JFK", "destination": "MBJ", "terminal": "T4" }`
  - Transfer: `{ "pickup_location": "MBJ Airport", "vehicle": "SUV" }`
  - Hotel: `{ "room_type": "Beachfront Suite", "check_in": "3:00 PM", "confirmation": "SDR-99291" }`

**SQL:**
```sql
create table bookings (
  id uuid primary key default gen_random_uuid(),
  trip_id uuid not null references trips(id) on delete cascade,
  operator_id uuid references operators(id) on delete set null,
  type text not null check (type in ('flight', 'transfer', 'hotel', 'tour', 'restaurant')),
  status text not null default 'confirmed' check (status in ('confirmed', 'rescheduled', 'cancelled', 'pending', 'ai_handling')),
  reference text,
  scheduled_time timestamptz not null,
  updated_time timestamptz,
  notes text,
  metadata jsonb,
  created_at timestamptz not null default now()
);
```

---

## Table: `disruptions`

Logged every time the AI detects a disruption event affecting a trip.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique disruption ID |
| `trip_id` | `uuid` | NOT NULL, FK → `trips.id` | The trip affected by this disruption |
| `booking_id` | `uuid` | NULLABLE, FK → `bookings.id` | The specific booking that triggered the disruption (e.g. the flight) |
| `type` | `text` | NOT NULL, CHECK IN ('flight_delay', 'flight_cancellation', 'weather', 'operator_cancellation') | Disruption category |
| `severity` | `text` | NOT NULL, CHECK IN ('low', 'medium', 'high'), default `'medium'` | How impactful the disruption is |
| `detected_at` | `timestamptz` | NOT NULL, default `now()` | When the AI detected the disruption |
| `raw_data` | `jsonb` | NOT NULL | Raw data from the external source (flight API response, weather data etc.) |
| `resolved` | `boolean` | NOT NULL, default `false` | Whether all affected bookings have been handled |
| `resolved_at` | `timestamptz` | NULLABLE | When the disruption was fully resolved |

**Relationships:**
- `disruptions.trip_id` → `trips.id`
- `disruptions.booking_id` → `bookings.id`
- `disruptions.id` ← referenced by `operator_alerts.disruption_id`

**Notes:**
- `raw_data` for a flight delay example:
```json
{
  "flight_number": "AA204",
  "original_departure": "2026-07-02T14:30:00Z",
  "updated_departure": "2026-07-02T18:30:00Z",
  "delay_minutes": 240,
  "reason": "Late incoming aircraft",
  "source": "flightaware"
}
```
- When a disruption is inserted, it triggers Vedang's agent pipeline via Supabase Realtime

**SQL:**
```sql
create table disruptions (
  id uuid primary key default gen_random_uuid(),
  trip_id uuid not null references trips(id) on delete cascade,
  booking_id uuid references bookings(id) on delete set null,
  type text not null check (type in ('flight_delay', 'flight_cancellation', 'weather', 'operator_cancellation')),
  severity text not null default 'medium' check (severity in ('low', 'medium', 'high')),
  detected_at timestamptz not null default now(),
  raw_data jsonb not null,
  resolved boolean not null default false,
  resolved_at timestamptz
);
```

---

## Table: `operator_alerts`

One row per operator that needs to be notified for a disruption. Tracks their response.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique alert ID |
| `disruption_id` | `uuid` | NOT NULL, FK → `disruptions.id` | The disruption that triggered this alert |
| `operator_id` | `uuid` | NOT NULL, FK → `operators.id` | The operator being notified |
| `booking_id` | `uuid` | NOT NULL, FK → `bookings.id` | The specific booking affected |
| `message` | `text` | NOT NULL | The AI-generated message sent to the operator |
| `proposed_time` | `timestamptz` | NULLABLE | New time proposed to the operator |
| `status` | `text` | NOT NULL, CHECK IN ('pending', 'confirmed', 'declined', 'alternative_requested'), default `'pending'` | Operator's response status |
| `responded_at` | `timestamptz` | NULLABLE | When the operator responded |
| `created_at` | `timestamptz` | NOT NULL, default `now()` | When the alert was created |

**Relationships:**
- `operator_alerts.disruption_id` → `disruptions.id`
- `operator_alerts.operator_id` → `operators.id`
- `operator_alerts.booking_id` → `bookings.id`

**Notes:**
- When `status` changes to `'confirmed'` → Vedang's agent updates the linked `booking.status` to `'rescheduled'` and sets `booking.updated_time`
- When `status` changes to `'declined'` → Vedang's agent triggers availability check for alternatives
- Supabase Realtime broadcasts on this table — Dayna's guest UI listens for status changes to animate the coordination progress screen
- Bhavesh's operator portal listens for new `pending` rows to show the alert screen

**SQL:**
```sql
create table operator_alerts (
  id uuid primary key default gen_random_uuid(),
  disruption_id uuid not null references disruptions(id) on delete cascade,
  operator_id uuid not null references operators(id) on delete cascade,
  booking_id uuid not null references bookings(id) on delete cascade,
  message text not null,
  proposed_time timestamptz,
  status text not null default 'pending' check (status in ('pending', 'confirmed', 'declined', 'alternative_requested')),
  responded_at timestamptz,
  created_at timestamptz not null default now()
);
```

---

## Table: `chat_messages`

The WhatsApp-style message thread between the guest and the Ripple AI.

| Field | Type | Constraints | Description |
|---|---|---|---|
| `id` | `uuid` | PRIMARY KEY, default `gen_random_uuid()` | Unique message ID |
| `trip_id` | `uuid` | NOT NULL, FK → `trips.id` | The trip this message belongs to |
| `sender` | `text` | NOT NULL, CHECK IN ('ai', 'guest') | Who sent the message |
| `body` | `text` | NOT NULL | The message content |
| `message_type` | `text` | NOT NULL, CHECK IN ('text', 'update', 'confirmation', 'action'), default `'text'` | Type of message for UI rendering |
| `metadata` | `jsonb` | NULLABLE | Optional data for rich messages (e.g. booking ID for a confirmation message) |
| `created_at` | `timestamptz` | NOT NULL, default `now()` | When message was sent |

**Relationships:**
- `chat_messages.trip_id` → `trips.id`

**Notes:**
- `message_type` controls how Dayna renders the bubble:
  - `'text'` — standard chat bubble
  - `'update'` — AI status update (blue, slightly different style)
  - `'confirmation'` — booking confirmed (shows booking name + green tick)
  - `'action'` — guest tapped a quick reply chip (logged here)
- Supabase Realtime broadcasts new rows to Dayna's chat feed in real time
- Vedang's agent inserts rows here as each operator confirms

**SQL:**
```sql
create table chat_messages (
  id uuid primary key default gen_random_uuid(),
  trip_id uuid not null references trips(id) on delete cascade,
  sender text not null check (sender in ('ai', 'guest')),
  body text not null,
  message_type text not null default 'text' check (message_type in ('text', 'update', 'confirmation', 'action')),
  metadata jsonb,
  created_at timestamptz not null default now()
);
```

---

## Relationships Diagram

```
users
 ├── trips (guest_id → users.id)
 │    ├── bookings (trip_id → trips.id)
 │    │    └── operator_alerts (booking_id → bookings.id)
 │    ├── disruptions (trip_id → trips.id)
 │    │    └── operator_alerts (disruption_id → disruptions.id)
 │    └── chat_messages (trip_id → trips.id)
 └── operators (user_id → users.id)
      ├── bookings (operator_id → operators.id)
      └── operator_alerts (operator_id → operators.id)
```

---

## Supabase Realtime — Which Tables to Enable

Enable in the Supabase dashboard under Database → Replication.

| Table | Enable Realtime? | Who Listens | Why |
|---|---|---|---|
| `bookings` | YES | Dayna (guest UI) | Status badge updates on itinerary |
| `operator_alerts` | YES | Dayna (guest UI) + Bhavesh (operator portal) | AI progress screen + operator alert screen |
| `chat_messages` | YES | Dayna (guest UI) | New messages appear instantly |
| `disruptions` | YES | Vedang (agent trigger) | New disruption fires the agent pipeline |
| `users` | NO | — | No real-time need |
| `trips` | NO | — | No real-time need |
| `operators` | NO | — | No real-time need |

---

## Demo Seed Data

This is the exact data to seed for the Maria demo scenario. Bhavesh runs this before every rehearsal.

```sql
-- Guest
insert into users (id, name, email, phone, role)
values ('00000000-0000-0000-0000-000000000001', 'Maria Johnson', 'maria@demo.com', '+18761234567', 'guest');

-- Operators
insert into operators (id, name, type, whatsapp_number, island, is_available)
values
  ('00000000-0000-0000-0001-000000000001', 'Bob''s Taxis', 'transfer', '+18769876543', 'Jamaica', true),
  ('00000000-0000-0000-0001-000000000002', 'Sandals Royal Caribbean', 'hotel', '+18765551234', 'Jamaica', true),
  ('00000000-0000-0000-0001-000000000003', 'Sunset Catamaran Tours', 'tour', '+18765559876', 'Jamaica', true),
  ('00000000-0000-0000-0001-000000000004', 'Tracks & Records', 'restaurant', '+18765554321', 'Jamaica', true);

-- Trip
insert into trips (id, guest_id, name, destination, start_date, end_date, status)
values ('00000000-0000-0000-0002-000000000001', '00000000-0000-0000-0000-000000000001', 'Jamaica · July 2026', 'Jamaica', '2026-07-02', '2026-07-07', 'active');

-- Bookings
insert into bookings (id, trip_id, operator_id, type, status, reference, scheduled_time, metadata)
values
  ('00000000-0000-0000-0003-000000000001', '00000000-0000-0000-0002-000000000001', null, 'flight', 'confirmed', 'AA204', '2026-07-02T14:30:00Z', '{"flight_number":"AA204","origin":"JFK","destination":"MBJ","terminal":"T4"}'),
  ('00000000-0000-0000-0003-000000000002', '00000000-0000-0000-0002-000000000001', '00000000-0000-0000-0001-000000000001', 'transfer', 'confirmed', 'BOB-2026-441', '2026-07-02T16:45:00Z', '{"pickup_location":"MBJ Airport"}'),
  ('00000000-0000-0000-0003-000000000003', '00000000-0000-0000-0002-000000000001', '00000000-0000-0000-0001-000000000002', 'hotel', 'confirmed', 'SDR-99291', '2026-07-02T15:00:00Z', '{"room_type":"Beachfront Suite","check_in":"3:00 PM"}'),
  ('00000000-0000-0000-0003-000000000004', '00000000-0000-0000-0002-000000000001', '00000000-0000-0000-0001-000000000003', 'tour', 'confirmed', 'SCT-2026-8821', '2026-07-03T14:00:00Z', '{"duration":"3 hours","guests":2}'),
  ('00000000-0000-0000-0003-000000000005', '00000000-0000-0000-0002-000000000001', '00000000-0000-0000-0001-000000000004', 'restaurant', 'confirmed', 'TR-20260702-7', '2026-07-02T19:00:00Z', '{"party_size":2,"name":"Maria Johnson"}');
```

---

## Reset Script

Run before every demo rehearsal to restore clean state.

```sql
-- Reset booking statuses
update bookings set status = 'confirmed', updated_time = null
where trip_id = '00000000-0000-0000-0002-000000000001';

-- Delete disruptions and alerts
delete from operator_alerts where disruption_id in (
  select id from disruptions where trip_id = '00000000-0000-0000-0002-000000000001'
);
delete from disruptions where trip_id = '00000000-0000-0000-0002-000000000001';

-- Clear chat messages
delete from chat_messages where trip_id = '00000000-0000-0000-0002-000000000001';
```
