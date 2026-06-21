# Ripple

**AI coordination platform for Caribbean tourism.**

Autonomous system that detects travel disruptions, checks operator availability, reschedules bookings, and notifies all stakeholders via WhatsApp — without a single phone call.

Built for the [Future Caribbean Global Agentic AI Buildathon 2026](https://futurecaribbean.com) - Track 02: AI for Tourism and Transportation.

## The Problem
A tourist lands in Jamaica. Flight delayed 4 hours. Their driver, hotel, and excursion operator have no idea. Everyone loses money.

## Our Solution
Ripple sits between tourists and every operator in their trip. The moment something goes wrong, a five-agent pipeline detects the disruption, checks every operator's availability, reschedules what can be moved, finds alternatives for what cannot, and sends one WhatsApp message to the guest — all in under 60 seconds.

## Tech Stack
- **Guest App**: Next.js PWA (mobile-first, 390px)
- **Operator Portal**: Next.js web app
- **Backend**: Supabase (PostgreSQL + Realtime + Auth)
- **AI Agents**: LangGraph + Claude API (Anthropic)
- **Messaging**: WhatsApp Business API
- **Hosting**: Vercel (frontend) + Supabase (backend)

## Team
| Name | Role |
|------|------|
| Michael | Full Stack, DevOps, AI, UI/UX |
| Dayna | Frontend |
| Bhavesh | Full Stack, Operator Portal |
| Vedang | AI Agents, ML |
| Shamar | Backend, APIs |

## Project Structure
- `apps/frontend` — Guest itinerary dashboard (PWA)
- `apps/operator` — Operator alert and confirmation portal
- `apps/backend` — API routes, Supabase schema, WhatsApp integration
- `agents/` — Five-agent LangGraph pipeline

## The Demo Scenario
Maria's flight is delayed 4 hours. She has 4 bookings: airport transfer, hotel, tour, and restaurant. Ripple detects the delay and coordinates all 4 operators automatically. Maria taps one button. Everything is fixed. Nobody calls anyone.
