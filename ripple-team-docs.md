# Ripple — Team Documentation
### AI Coordination Platform for Caribbean Tourism
**Future Caribbean Global Agentic AI Buildathon | Track 02**

---

## The App in One Sentence
Ripple is an AI layer that sits between tourists and every operator in their trip — the moment something goes wrong, Ripple automatically notifies, reschedules, and coordinates every affected booking without the guest making a single phone call.

---

## Critical Dates

| Milestone | Date |
|---|---|
| Today | June 20, 2026 |
| Demo Deadline | **July 2, 2026** |
| Application Closes | **July 3, 2026** |
| Top 50 Teams Announced | July 17, 2026 |
| 21-Day Builder Sprint Begins | July 17, 2026 |
| Builder Sprint Ends | August 7, 2026 |
| Final Submission Review | August 8–15, 2026 |

We have **12 days** to demo day. Every decision between now and July 2 is about making the demo work — not building the perfect product.

---

## The Team

| Name | Role | Core Responsibility |
|---|---|---|
| Michael | Full Stack, DevOps, AI, UI/UX | Architecture, integration, deployment, design oversight |
| Dayna | Frontend, UI | Guest-facing React Native mobile app |
| Bhavesh | Full Stack, DevOps | Operator portal, integration glue, Supabase Realtime |
| Vedang | Machine Learning, Data Engineering | All AI agents, LangGraph pipeline, Claude API |
| Shamar | Backend Developer | Supabase schema, APIs, webhooks, external data feeds |

**One rule:** Michael is the integration point. Any time two workstreams need to connect, that decision goes through Michael. No one builds in isolation and discovers on the last day that nothing fits together.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Guest Mobile App | React Native (iOS + Android) |
| Demo (July 2) | PWA via Next.js — ships fastest, no App Store wait |
| Operator Portal | Next.js web app |
| Backend / Database | Supabase (Postgres + Realtime + Auth) |
| AI Agents | LangGraph + Claude API (Anthropic) |
| External APIs | WhatsApp Business API, Flight data feed, Weather API |
| Hosting | Vercel (frontend) + Supabase (backend) |
| Design | Stitch (all screens done) |
| CI/CD | GitHub Actions |

---

## Phase 1 — Demo Sprint (June 20 → July 2)

This phase is about one thing: **a working end-to-end demo of the Maria scenario.**

Maria's flight is delayed 4 hours. She has 4 bookings: airport transfer (Bob's Taxis), hotel (Sandals Royal), tour (Sunset Catamaran), restaurant (Tracks & Records). Ripple detects the delay and coordinates all 4 operators automatically. Maria taps one button. Everything is fixed. Nobody calls anyone.

That is the only thing that needs to work by July 2.

---

### Michael — Full Stack, DevOps, AI, UI/UX

**Day 1–2**
- Finalize the data schema with Shamar — define every table, every field, every relationship before anyone writes code
- Draw the integration diagram: who calls what API, what data lives where, what fires what event
- Set up the GitHub repo, branch structure, and CI/CD pipeline
- Set up Vercel projects for frontend and Supabase project for backend
- Brief the full team on the schema and integration diagram before anyone starts building

**Day 3–5**
- Write the full demo script — every line of dialogue, every button tap, every screen transition for the Maria scenario
- Oversee Dayna's implementation of the guest screens — ensure designs from Stitch are being followed accurately
- Assist Vedang on agent architecture if needed
- Daily blocker check — if anyone is stuck for more than 2 hours, Michael finds a solution or rescopes

**Day 6–8**
- Run the first end-to-end test of the full Maria flow yourself
- Document every bug found and assign each one to the right person
- Help Bhavesh with any integration glue issues between frontend and backend
- Review all Supabase Realtime connections are firing correctly

**Day 9–10**
- Lock the demo script
- Run full rehearsal with the team — time it, cut anything slow
- Fix all critical bugs from Day 6–8 test
- Write and submit the Future Caribbean application before July 3

**Day 11–12 (July 1–2)**
- Demo freeze — no new features, bug fixes only
- Full rehearsal run minimum twice
- Prepare phone/device for live demo
- Deploy final build to production URLs

---

### Dayna — Frontend, React Native / Next.js PWA

**Day 1–2**
- Set up the Next.js PWA project with correct mobile viewport settings (390px wide)
- Import the Ripple design system from the Stitch files — colors, fonts, spacing tokens
- Build reusable components first: Booking Card, Status Badge, Quick Action Chips, Bottom Nav, AI Status Banner

**Day 3–5**
- Build the guest screens in this priority order:
  1. Dashboard / Home
  2. Active Trip View (itinerary timeline)
  3. Disruption Alert Screen
  4. AI Coordination Progress Screen
  5. Chat Feed
- All screens must connect to real Supabase data — no hardcoded content
- Implement Supabase Realtime listener on the Active Trip View so booking statuses update live

**Day 6–8**
- Build remaining guest screens: Booking Detail, Cancellation bottom sheet, Reschedule screen, Profile & Settings
- Wire up all quick action buttons to actual API calls (Cancel, Reschedule, Call Operator)
- Test all screens on a real mobile device — not just browser dev tools

**Day 9–10**
- Fix all UI bugs from Michael's end-to-end test
- Polish animations: AI coordination progress sequence, success state, status badge transitions
- Ensure the Maria demo flow taps through smoothly end to end

**Day 11–12**
- No new screens — fix only
- Test on both iOS Safari and Android Chrome
- Confirm PWA is installable on home screen for demo day

---

### Bhavesh — Full Stack, Operator Portal + Integration

**Day 1–2**
- Set up the operator portal Next.js project
- Align with Shamar on which API endpoints you'll be consuming
- Align with Dayna on Supabase Realtime event names so both sides use the same channel names

**Day 3–5**
- Build operator screens in this order:
  1. Operator Login (with role selector — Hotel, Driver, Tour, Restaurant)
  2. Operator Alert Screen (Schedule Change notification with Confirm / Can't Do It)
  3. Operator Dashboard (booking list with status badges)
- Connect operator confirmation action back to Supabase so the guest's UI updates in real time when an operator confirms

**Day 6–8**
- Build the integration glue layer — the middleware that takes an AI agent output and triggers the right Supabase update, which then fires the Realtime event to both guest and operator UIs
- Seed the database with demo data: Maria's profile, her 4 bookings, the 4 operators
- Test the full loop: AI fires event → Supabase updates → operator sees alert → operator confirms → guest sees green checkmark

**Day 9–10**
- Fix any integration bugs from Michael's end-to-end test
- Ensure the operator dashboard online/offline toggle works and reflects in the system
- Help Dayna with any frontend issues if she's blocked

**Day 11–12**
- Demo data reset script — so you can cleanly re-run the Maria scenario during the pitch without leftover state
- Final integration test with all 5 team members on a call

---

### Vedang — AI Agents, LangGraph, Claude API

**Day 1–2**
- Set up the LangGraph project and agent scaffolding
- Define the 5 agent contracts with Shamar — what each agent receives as input, what it outputs, what it writes to Supabase
- Get Claude API keys and test a basic prompt response

**Day 3–5**
- Build and test Agent 1: **Flight Monitor** — polls flight status API, detects delay, writes disruption event to Supabase
- Build and test Agent 2: **Booking Scanner** — reads all bookings linked to the affected flight, identifies which ones need updating
- Both agents must be working and writing to Supabase before moving to agents 3–5

**Day 6–8**
- Build Agent 3: **Operator Notifier** — reads affected bookings, generates notification message per operator type (different message for hotel vs driver vs restaurant), writes to operator_alerts table
- Build Agent 4: **Availability Checker** — when an operator can't do it, finds alternative time slots or alternative operators
- Build Agent 5: **Guest Communicator** — generates the chat feed messages Maria sees, confirming each update in a conversational tone
- Wire all 5 agents into a single LangGraph pipeline that runs start to finish

**Day 9–10**
- Run the full Maria scenario through the pipeline end to end
- Tune Claude prompts — messages to operators should be professional, messages to guests should be warm and reassuring
- Fix any agent logic bugs from Michael's end-to-end test
- Ensure the pipeline completes in under 2 minutes for demo purposes

**Day 11–12**
- Pipeline freeze — no changes unless something is broken
- Run the pipeline 5 times with demo data to confirm it's stable
- Document what each agent does in one paragraph each for the pitch

---

### Shamar — Backend, APIs, External Integrations

**Day 1–2**
- Design the full Supabase schema with Michael — every table finalized before anyone writes code:
  - `users` (guests + operators)
  - `trips`
  - `bookings` (with type: flight / transfer / hotel / tour / restaurant)
  - `operators`
  - `disruptions`
  - `operator_alerts`
  - `chat_messages`
- Set up Supabase project, run migrations, enable Realtime on key tables

**Day 3–5**
- Build all API routes:
  - `GET /trip/:id` — full trip with all bookings
  - `POST /bookings/:id/cancel` — cancel a booking
  - `POST /bookings/:id/reschedule` — reschedule with new time
  - `GET /operator/:id/bookings` — operator's daily bookings
  - `POST /operator-alerts/:id/confirm` — operator confirms a change
  - `POST /operator-alerts/:id/decline` — operator can't do it
- Integrate flight status API — poll for AA 204 delay and write event to `disruptions` table

**Day 6–8**
- Integrate WhatsApp Business API — send operator notifications via WhatsApp when an alert is created (backup to in-app notification)
- Set up Supabase Realtime channels for:
  - Guest booking status updates
  - Operator alert arrivals
  - Chat message feed
- Test all Realtime events fire correctly when Supabase rows are updated

**Day 9–10**
- Fix any API bugs from Michael's end-to-end test
- Ensure the demo data is seeded correctly and all foreign keys are correct
- Write the data reset endpoint so Bhavesh can cleanly reset the Maria scenario

**Day 11–12**
- API freeze — no schema changes
- Monitor Supabase logs during demo rehearsals for any errors
- Confirm all external API keys (flight data, WhatsApp) are on paid/production tier — no free tier rate limits during the demo

---

## Team Workflow

### Daily Standup
- **When:** Every day, 15 minutes max, voice or video
- **Format:** Three questions only:
  1. What did you finish yesterday?
  2. What are you working on today?
  3. Is anything blocking you?
- If a blocker takes more than 2 hours to solve — escalate to Michael immediately, don't wait for the next standup

### Git Workflow
- Main branch: `main` — production only, always deployable
- Development branch: `dev` — all feature branches merge here first
- Feature branches: `feature/your-name/what-youre-building`
- Never push directly to `main`
- Pull requests require Michael's review before merging to `main`
- Commit messages: be specific — "Add operator alert confirm endpoint" not "fix stuff"

### Integration Protocol
- Any time your work needs to connect to someone else's work — tell Michael first
- Agreed API contracts are written in a shared doc before either side starts building
- If you change an API contract after it's agreed — notify Michael and the affected person immediately
- Bhavesh owns the integration layer — if Dayna's frontend and Shamar's backend aren't talking, Bhavesh fixes the bridge

### Communication
- Daily standup: voice/video call
- Real-time questions: group chat (WhatsApp or Slack — team's choice)
- Decisions that affect more than one person: Michael makes the call
- Design questions: reference the Stitch file and design.md first before asking

### Blocker Protocol
- Stuck for under 2 hours — try to solve it yourself
- Stuck for over 2 hours — post in the group chat with exactly what you're trying to do, what you've tried, and what the error is
- Michael triages and either solves it, pairs with you, or rescopes the task

### Demo Data
- All demo runs use the same fixed seed data: Maria Johnson, Flight AA 204, 4 bookings, 4 operators
- Bhavesh maintains the reset script — run it before every rehearsal so the state is clean
- No one modifies demo data without telling the team

---

## Phase 2 — Buildathon Sprint (July 17 → August 7)

*This phase only begins if we're selected as a Top 50 team on July 17.*

If selected, the sprint moves from demo to real product. The focus shifts from "make it work for one scenario" to "make it work for any scenario on any Caribbean island."

### What changes in Phase 2

**Michael** — scale the architecture for multi-tenant operators (multiple islands, multiple operators per type), build admin dashboard, harden DevOps for real traffic

**Dayna** — migrate PWA to full React Native for App Store submission (iOS + Android), add push notifications, refine animations and polish

**Bhavesh** — build operator onboarding flow (self-serve sign-up for hotels/drivers/tours), build billing and subscription layer

**Vedang** — expand AI agents to handle more disruption types (weather, cancellations, overbookings), improve agent reasoning with more context, build availability and alternatives database for the region

**Shamar** — expand to multi-island data (Jamaica, Barbados, Trinidad, Bahamas), integrate additional flight APIs, build operator availability calendar system

---

## Definition of Done — July 2 Demo

The demo is ready when all of the following are true:

- [ ] Maria's flight delay triggers automatically (or can be manually triggered for demo)
- [ ] All 4 operator alerts appear within 30 seconds of the disruption
- [ ] Guest sees live AI coordination progress screen with all 4 operators resolving
- [ ] At least 3 of 4 operators can confirm via the operator portal in real time
- [ ] Guest chat feed shows correct AI messages after each confirmation
- [ ] The full flow runs in under 3 minutes end to end
- [ ] Demo works on a real phone (not just a laptop browser)
- [ ] Demo data can be reset and re-run cleanly
- [ ] No crashes or loading errors during a rehearsal run
- [ ] Michael has rehearsed the demo script at least 3 times with the full flow working
