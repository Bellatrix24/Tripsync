# TripSync — Project Handoff

Welcome to the TripSync project! This document serves as a comprehensive reference guide to get you up to speed quickly on the architecture, current state, and future roadmap of the application.

## Project Overview
TripSync is an AI-powered group travel planning application that takes the headache out of coordinating trips with friends. It collects individual preferences (vacation vibes, budget, dietary restrictions, dealbreakers) either via a web form or a conversational voice assistant. Using Gemini AI, it synthesizes the group's preferences into an optimal destination recommendation and generates a detailed, conflict-free, budget-aware day-by-day itinerary. It also features ElevenLabs text-to-speech to narrate the final itinerary to the group.

## Tech Stack
- **Framework:** Next.js 15.0.3 (App Router)
- **UI Library:** React 19.0.0
- **Styling:** Tailwind CSS 4 (via `@tailwindcss/postcss`)
- **Database & Auth:** Supabase (Auth, Postgres DB)
- **AI / LLM:** Google Gemini 3.1 Pro Preview (via `@google/generative-ai`)
- **Voice / Audio:** ElevenLabs streaming TTS API, Web Speech API (for Speech-to-Text)
- **Icons & Visuals:** Vanilla CSS gradients, SVG components, native HTML elements

## Project Structure
```
src/
├── app/
│   ├── api/
│   │   ├── trips/
│   │   │   ├── [tripId]/
│   │   │   │   ├── generate-itinerary/route.ts  # Generates day-by-day itinerary via Gemini
│   │   │   │   ├── preferences/route.ts         # Saves individual preferences to DB
│   │   │   │   ├── recommend-locations/route.ts # Scores destinations based on synthesis
│   │   │   │   └── synthesize/route.ts          # Synthesizes all group preferences into one profile
│   │   │   └── route.ts                         # Creates new trips in the DB
│   │   └── voice/
│   │       ├── extract-preferences/route.ts     # Extracts JSON prefs from a raw voice transcript
│   │       ├── narrate/route.ts                 # Writes a radio-style script and fetches TTS audio
│   │       └── tts/route.ts                     # Wrapper for ElevenLabs TTS API
│   ├── auth/callback/route.ts                   # Supabase OAuth callback route
│   ├── dashboard/page.tsx                       # User dashboard listing active/past trips
│   ├── join/[shareCode]/
│   │   ├── JoinTripClient.tsx                   # Client component for joining a trip
│   │   └── page.tsx                             # Server page for Magic Link / join process
│   ├── trip/[tripId]/
│   │   ├── itinerary/
│   │   │   ├── ItineraryView.tsx                # Client component displaying the final plan
│   │   │   └── page.tsx                         # Server page wrapping the itinerary
│   │   ├── locations/
│   │   │   ├── LocationsClient.tsx              # Renders destination options and budget warnings
│   │   │   └── page.tsx                         # Server page for location recommendation
│   │   ├── preferences/
│   │   │   ├── PreferenceForm.tsx               # Form + Voice Assistant entry for preferences
│   │   │   └── page.tsx                         # Server page wrapper
│   │   └── status/
│   │       ├── StatusDashboard.tsx              # Leader dashboard tracking who has filled forms
│   │       └── page.tsx                         # Server page wrapper
│   ├── trip/new/page.tsx                        # Form to create a new trip
│   ├── globals.css                              # Tailwind entrypoint + custom gradients/dark mode vars
│   ├── layout.tsx                               # Root layout with ThemeProvider and Navbar
│   └── page.tsx                                 # Landing page
├── components/
│   ├── NarrationPlayer.tsx                      # Modal player for ElevenLabs TTS narration
│   ├── Navbar.tsx                               # Top navigation with dark mode toggle + Auth
│   ├── ThemeProvider.tsx                        # Handles light/dark mode persistence
│   ├── VoiceConversation.tsx                    # Full-screen conversational AI mic interface
│   └── VoiceOrb.tsx                             # Animated canvas responding to mic/speaker amplitude
├── lib/
│   ├── destinations.ts                          # Hardcoded JSON dataset of US locations
│   ├── gemini.ts                                # Gemini API wrapper with JSON parsing + retries
│   ├── supabase/
│   │   ├── client.ts                            # Supabase browser client
│   │   └── server.ts                            # Supabase server client (cookies implementation)
│   └── types.ts                                 # Typescript interfaces for DB tables
└── middleware.ts                                # Next.js middleware protecting routes via Supabase
```

## What's Complete
### Phase 1: Setup & Core Auth
- Supabase integration & Next.js middleware
- Landing page, Dashboard, Create Trip flow
- Join trip via share code link

### Phase 2: Preference Engine
- Multi-step preference form (Vibes, Budget, Dietary, Dealbreakers)
- Real-time status dashboard for the trip leader
- AI Group Synthesis (identifying common ground, conflicts, and constraints)
- AI Location Recommendation (scoring destinations against the group profile)

### Phase 3: AI Itinerary & Voice Integration
- Day-by-day itinerary generation with strict budget enforcement and validation
- Budget warning banners and color-coded progress bars
- Voice Preference Collection (Web Speech API mic + ElevenLabs TTS assistant)
- AI Itinerary Narration (Gemini scripts a radio segment, ElevenLabs speaks it)

## What's Working
- ✅ All AI functions (JSON extraction, synthesis, location scoring, itinerary creation)
- ✅ Budget validation (locations over budget are blocked, individual overages are flagged)
- ✅ The Voice Assistant (mic connects, AI responds, UI auto-scrolls, data populates form)
- ✅ Itinerary Narration (Script generates, audio plays, pause/play/seek works)
- ✅ Dark mode / Light mode switching across all components

## Known Issues
- **Stale Closures in Voice:** `VoiceConversation.tsx` uses `[]` dependency arrays in some `useCallback` hooks (`speakAndListen`, `handleUserResponse`). If dynamic state is added to these later, it could cause stale closure bugs.
- **Double TTS Call in Narration:** `narrate/route.ts` requests TTS from ElevenLabs, but if it fails silently (returning `audioUrl: null`), `NarrationPlayer.tsx` immediately fires a second retry fetch to `/api/voice/tts`.
- **AudioContext Reuse:** `VoiceOrb.tsx` does not disconnect the `AudioContext` from the source node when a track ends. This might prevent the visualizer from working on subsequent questions if the `audioElement` reference fundamentally changes.
- **Supabase Placeholder Keys:** `server.ts` falls back to "placeholder-key" if env vars are missing, which can cause opaque connection errors locally if `.env.local` is misconfigured.
- **Current Voice Status:** Audio is playing correctly, microphone correctly captures speech, and the 2-second silence auto-stop works exactly as expected. The scrolling physics in the voice modal are confirmed fixed.

## What's Left to Build
### Phase 4
- **Expense Dashboards:** A "Splitwise-lite" feature to log actual expenses during the trip.
- **Individual/Group Views:** Toggles on the itinerary to see only "My Schedule" vs "Group Schedule".

### Phase 5 & Stretch Goals
- **Polish & Responsive Design:** Final mobile breakpoint checks, animation tuning.
- **Stretch:** Flight/Hotel live API integration (currently handled by Gemini Google Grounding).

## Deployment
The app is deployed directly (no Docker) on a Vultr VPS (Ubuntu 24.04) at **http://144.202.69.28.nip.io:3000**.

To deploy:
```bash
ssh user@144.202.69.28
cd ~/HackCU-TripAi && git pull origin main && npm run build && pm2 restart tripsync
```

## Database Schema
- **`users`**
  - `id` (uuid, PK), `email`, `name`, `avatar_url`, `created_at`
- **`trips`**
  - `id` (uuid, PK), `share_code`, `tagline`, `destination`, `origin_city`, `start_date`, `end_date`, `trip_duration_days`, `group_size`, `leader_id` (FK), `status` (collecting, synthesis, complete)
- **`trip_members`**
  - `id`, `trip_id` (FK), `user_id` (FK), `role`
- **`preferences`**
  - `id`, `trip_id`, `user_id`, `budget_min`, `budget_max`, `vibe_beach`, `vibe_city`, `vibe_nature`, `vibe_culture`, `vibe_relaxation`, `vibe_nightlife`, `vibe_adventure`, `dietary_restrictions`, `activity_level`, `accommodation_pref`, `must_haves`, `dealbreakers`, `additional_notes`
- **`trip_synthesis`**
  - `id`, `trip_id`, `synthesis_data` (jsonb)
- **`location_recommendations`**
  - `id`, `trip_id`, `destination`, `score`, `reasoning`, `pros`, `cons`, `estimated_budget_pp`
- **`itineraries`**
  - `id`, `trip_id`, `destination`, `itinerary_data` (jsonb), `budget_breakdown` (jsonb)

## API Routes
| Method | Route | Description | Auth Required |
|--------|-------|-------------|---------------|
| POST | `/api/trips` | Creates a new trip | Yes |
| POST | `/api/trips/[id]/preferences` | Saves user preferences | Yes |
| POST | `/api/trips/[id]/synthesize` | Runs group synthesis | Yes (Leader only) |
| POST | `/api/trips/[id]/recommend-locations`| Scores US destinations | Yes (Leader only) |
| POST | `/api/trips/[id]/generate-itinerary`| Creates final day-by-day plan | Yes (Leader only) |
| POST | `/api/voice/extract-preferences` | Extracts JSON from voice transcript | Yes |
| POST | `/api/voice/narrate` | Generates script + audio for narration | Yes |
| POST | `/api/voice/tts` | Raw ElevenLabs TTS wrapper (returns stream) | Yes |

## Environment Variables
The application requires a `.env.local` file:
```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_supabase_service_role_key
GEMINI_API_KEY=your_google_ai_studio_key
ELEVENLABS_API_KEY=your_elevenlabs_api_key
ELEVENLABS_VOICE_ID=21m00Tcm4TlvDq8ikWAM # Optional (Rachel is default)
```

## How to Run Locally
1. `git clone` the repository
2. `npm install`
3. Create `.env.local` in the root and add the required variables.
4. `npm run dev`
5. Open `http://localhost:3000`

## Git Workflow Rule
**CRITICAL MUST-FOLLOW RULES FOR AI:**
1. **Always ask before pushing to GitHub.**
2. **Always ask the user for the commit message** before running `git commit`. Do not auto-generate generic commit messages.

## Design System
- **Colors:** Deep espresso/charcoal background (`#0c0a09`) with emerald greens (`#10b981`), warm ambers (`#f59e0b`), and primary blues (`bg-primary`).
- **Fonts:** Modern sans-serif stack (system defaults + Inter if provided by Next.js).
- **Dark Mode:** Implemented globally via Tailwind `dark:` classes and persistent across sessions via `ThemeProvider.tsx`. 
- **Component Patterns:**
  - Glassmorphism: `bg-card/40 backdrop-blur-xl border border-white/5`
  - Subtle gradients: `.gradient-bg` (emerald to azure)

## AI Prompt Architecture
- **Synthesis:** Analyzes individual preferences to find common ground, calculate budget overlaps (finding the tightest constraint), and highlight conflicts.
- **Location Scoring:** Maps the synthesis against `destinations.ts` (hardcoded catalog) and returns the top 4 locations. Validates that the estimated cost does not exceed the tightest budget constraint.
- **Itinerary Generation:** Hard constraints applied to budget limits. Uses Google Grounding (Search tool enabled) to verify that hotels, flights, and restaurants actually exist. Strict JSON schema extraction logic strips out markdown fences.
- **Voice Extraction:** Highly tolerant NLP prompt designed to extract structured arrays/ints from casual speech ("I'm vegetarian but I eat fish" -> `["pescatarian"]`).
- **Voice Narration:** Translates a JSON itinerary into an enthusiastic, 2-minute "radio-style" script specifically structured for TTS readability.

## ElevenLabs Setup
- **API Endpoint:** Uses `/v1/text-to-speech/{voice_id}/stream` to instantly pipe audio back to the browser.
- **Voice Setup:** Uses "Rachel" (`21m00Tcm4TlvDq8ikWAM`) by default. Settings are strictly configured (stability: 0.5, similarity_boost: 0.75).
- **Fallback:** If ElevenLabs fails (API key issue, quota limit), the backend returns a 500/failed response, and the frontend automatically catches this and falls back to the native browser window `speechSynthesis` API.

## Test Data
Four personas are currently used to test conflict resolution and budget constraints:
1. **Ron:** The frugal nature enthusiast (Budget: $500, Loves beaches/nature, Diet: None, Dealbreaker: Crowds).
2. **Jon:** The moderate explorer (Budget: $800, Loves city/culture, Diet: Dairy-Free, Must have: Walkable area).
3. **Tom:** The luxury relaxation seeker (Budget: $2500, Loves relaxation/nightlife, Diet: Vegan, Dealbreaker: Remote areas).
4. **Mike:** The chaotic thrifty traveler (Budget: $450, Loves adventure, Diet: Peanut allergy, Dealbreaker: Long drives).
