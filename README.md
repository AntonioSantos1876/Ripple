# Ripple

**The AI operating system for Caribbean tourism.**

Autonomous coordination platform that detects travel disruptions, checks operator availability, reschedules bookings, and notifies all stakeholders via WhatsApp - without a single phone call.

Built for the [Future Caribbean Global Agentic AI Buildathon 2026](https://futurecaribbean.com) - Track 02: AI for Tourism and Transportation.

## The Problem
A tourist lands in Jamaica. Flight delayed 4 hours. Their driver, hotel, and excursion operator have no idea. Everyone loses money.

## Our Solution
IslandFlow AI is a five-agent pipeline that detects the disruption, checks every operator's availability and business hours, reschedules what can be moved, finds alternatives for what cannot, and sends one WhatsApp message to the guest - all in under 60 seconds.

## Tech Stack
- **Frontend**: Next.js 16, Tailwind CSS
- **Backend**: Supabase (PostgreSQL + Realtime), Node.js
- **AI Agents**: LangGraph, OpenAI, Claude API
- **Messaging**: WhatsApp Business API

## Team
| Name | Role |
|------|------|
| Michael | Project Lead |
| Vedang | AI Models |
| Shamar | Backend |
| Bhavesh | Full Stack |
| Dayna | Frontend |

## Project Structure
- `apps/frontend` - Guest itinerary dashboard and operator coordination view
- `apps/backend` - API routes, Supabase schema, WhatsApp integration
- `agents/` - Five-agent LangGraph pipeline

