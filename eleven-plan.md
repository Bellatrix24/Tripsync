# TASK: Implement Voice Onboarding Module for TripSync

You are implementing a **voice-based preference gathering system** for a group trip planning app called **TripSync**. This replaces the boring form-based onboarding with a conversational voice experience where an AI voice agent asks users about their travel preferences.

We already built and tested a working demo. Your job is to implement this as a **production-ready feature** integrated into the existing TripSync codebase.

---

## WHAT THIS FEATURE DOES

When a user joins a trip and needs to submit their preferences, they get two options:
1. **Voice mode** — An AI voice asks questions out loud, user speaks back, preferences are extracted automatically
2. **Type mode** — Traditional text input fallback (always available as a toggle)

The voice agent asks 5 questions in sequence, waits for the user to respond, acknowledges their answer, then moves to the next question. After all 5, the extracted preferences are saved to the database — identical format to the existing form-based preferences.

---

## TECH STACK

| Component | Tool | Why |
|-----------|------|-----|
| Text-to-Speech (AI voice) | **ElevenLabs API** `/v1/text-to-speech/{voice_id}/stream` | High quality, natural voice |
| Speech-to-Text (user voice) | **Browser `webkitSpeechRecognition`** | Free, no API cost, works in Chrome/Edge |
| Intent extraction | **Google Gemini API** (already in your backend) | Parse freeform voice answers into structured preference fields |
| Fallback TTS | **Browser `speechSynthesis`** | Free fallback if ElevenLabs quota runs out |

---

## ARCHITECTURE — WHERE FILES GO

### Backend (new files)

```
backend/
├── services/
│   └── elevenlabs.py        ← NEW: TTS service wrapper
├── agents/
│   └── voice_onboarding.py  ← NEW: Conversation state + intent extraction
├── routers/
│   └── voice.py             ← NEW: API endpoints for voice flow
```

### Frontend (new files)

```
frontend/src/
├── pages/
│   └── VoiceOnboarding.jsx  ← NEW: Full voice onboarding page
├── components/
│   ├── VoiceOrb.jsx         ← NEW: Animated orb that reacts to audio
│   └── PreferenceCard.jsx   ← NEW: Shows extracted preferences in real-time
```

---

## BACKEND IMPLEMENTATION

### 1. `services/elevenlabs.py`

Thin wrapper around ElevenLabs API. This is the ONLY file that talks to ElevenLabs.

```
Responsibilities:
- Accept text input, return audio bytes (streaming)
- Handle API key from environment variable: ELEVENLABS_API_KEY
- Support voice selection (store voice_id in env or config)
- Handle errors gracefully (quota exceeded, rate limits)
- If ElevenLabs fails, return a flag so frontend can fall back to browser TTS

Endpoint used:
POST https://api.elevenlabs.io/v1/text-to-speech/{voice_id}/stream

Headers:
  Content-Type: application/json
  xi-api-key: {ELEVENLABS_API_KEY}

Body:
{
  "text": "<the question or acknowledgment text>",
  "model_id": "eleven_monolingual_v1",
  "voice_settings": {
    "stability": 0.5,
    "similarity_boost": 0.75
  }
}

Response: audio/mpeg stream → return as StreamingResponse

Recommended voices (pick one as default):
- Rachel: 21m00Tcm4TlvDq8ikWAM
- Sarah: EXAVITQu4vr4xnSDxMaL
- Daniel: onwK4e9ZLuTAKqWW03F9
- Lily: pFZP5JQG7iQjIQuC4Bku
```

### 2. `agents/voice_onboarding.py`

Manages the conversation flow and extracts structured data from freeform voice answers.

```
Responsibilities:
- Maintain a state machine tracking which questions have been answered
- Define the 5 questions (see QUESTIONS section below)
- Use Gemini to parse freeform voice responses into structured fields
- Generate contextual acknowledgments based on what the user said
- Return the next question or signal completion
- Output final preferences in the EXACT same schema the existing form uses

State machine:
  IDLE → Q1_BUDGET → Q2_INTERESTS → Q3_ACCOMMODATION → Q4_PACE → Q5_DIETARY → DONE

Gemini prompt for intent extraction:
  System: "Extract travel preferences from the user's spoken response. Return JSON only."
  User: "Question was: '{question_text}'. User said: '{transcript}'. 
         Extract the relevant preference field. Return JSON: {field_name: value}"
  
  Example:
    Question: "What's your rough daily budget?"
    User said: "I'm thinking maybe 200 bucks a day"
    → {"budget_per_day": 200}

    Question: "What are you into?"
    User said: "Love the beach, nightlife, and good food"
    → {"interests": ["beach", "nightlife", "food"]}

IMPORTANT: A single spoken response might cover multiple fields. 
Example: "200 bucks a day, love beaches, want a nice hotel"
→ This answers budget, interests, AND accommodation in one shot.
The agent should detect this and skip already-answered questions.
```

### 3. `routers/voice.py`

FastAPI endpoints for the voice flow.

```
Endpoints:

POST /api/voice/start
  → Starts a new voice onboarding session
  → Returns: { session_id, first_question_text, first_question_audio_url }

POST /api/voice/respond
  Body: { session_id, transcript }
  → Sends user's spoken text to the agent for extraction
  → Returns: {
      acknowledgment_text: str,
      acknowledgment_audio_url: str | null,
      next_question_text: str | null,
      next_question_audio_url: str | null,
      extracted_so_far: { budget: ..., interests: ..., ... },
      is_complete: bool
    }

GET /api/voice/tts?text=...
  → Converts any text to speech via ElevenLabs
  → Returns: audio/mpeg stream
  → Used by frontend to play AI voice

POST /api/voice/complete
  Body: { session_id, trip_id, user_id }
  → Saves extracted preferences to the database
  → Uses the SAME preferences schema as the existing form
  → Returns: { success: true, preferences: {...} }
```

---

## FRONTEND IMPLEMENTATION

### 1. `VoiceOnboarding.jsx` — Main Page

This page replaces (or sits alongside) the existing Preferences form page. User can toggle between voice and typing.

```
Layout:
┌─────────────────────────────────┐
│  [Voice Mode] | [Type Mode]     │  ← toggle at top
├─────────────────────────────────┤
│                                 │
│         ┌───────────┐           │
│         │           │           │
│         │   ORB     │           │  ← animated orb (see VoiceOrb.jsx)
│         │           │           │
│         └───────────┘           │
│      "Listening to you..."      │  ← phase label
│                                 │
│  ┌─────────────────────────┐    │
│  │ You: 300 dollars a day  │    │  ← live transcript (voice mode)
│  └─────────────────────────┘    │
│                                 │
│  ● ● ○ ○ ○  Question 2 of 5    │  ← progress dots
│                                 │
│  [Done Speaking]  [Skip]        │  ← action buttons
│                                 │
│  ┌─────────────────────────┐    │
│  │ Extracted Preferences   │    │
│  │ Budget: $300/day        │    │  ← real-time extraction card
│  │ Interests: —            │    │
│  │ Accommodation: —        │    │
│  └─────────────────────────┘    │
│                                 │
│  ┌─────────────────────────┐    │
│  │ Conversation Log        │    │  ← scrollable log of Q&A
│  └─────────────────────────┘    │
└─────────────────────────────────┘

Flow:
1. Page loads → call POST /api/voice/start
2. Play the first question audio (from ElevenLabs via backend)
3. After audio ends, wait 500ms, then activate mic (webkitSpeechRecognition)
4. Show live transcript as user speaks
5. Auto-stop mic after 2 seconds of silence (CRITICAL — tested and working)
6. Send transcript to POST /api/voice/respond
7. Play acknowledgment audio → then play next question audio
8. Repeat until all 5 questions done
9. Call POST /api/voice/complete to save preferences
10. Redirect to trip dashboard

Voice → Text fallback:
- If mic permission denied → auto-switch to text input
- If ElevenLabs fails → fall back to browser speechSynthesis for TTS
- "Prefer typing?" toggle always visible
```

### 2. `VoiceOrb.jsx` — Animated Orb Component

A visual indicator that shows what the AI is doing. This is the centerpiece of the UI.

```
Props:
  phase: "idle" | "speaking" | "listening" | "processing" | "done"
  audioLevel: 0-1 float (for speaking animation)

Visual behavior:
  SPEAKING (orange/amber):
    - Orb pulses with audio level (scale 1.0 → 1.35 based on audioLevel)
    - 5 concentric ripple rings expand with audio
    - Sound bar icon in center (3 animated bars)
    - Color: #f0a878 → #7c2d12 radial gradient

  LISTENING (green):
    - Orb gently breathes (sine wave on ring radii)
    - Microphone icon in center
    - Color: #6ee7b7 → #065f46 radial gradient

  PROCESSING (gray):
    - Spinning loader icon in center
    - Subtle ring animation

  DONE (green):
    - Checkmark icon in center
    - Rings settle to static

Implementation: SVG-based with requestAnimationFrame loop.
Background: dark (#0c0a09), the orb is the focal point.
```

### 3. Speech Recognition Setup (in VoiceOnboarding.jsx)

```javascript
// CRITICAL: These settings were tested and work
const recognition = new webkitSpeechRecognition();
recognition.continuous = true;       // Don't stop on first pause
recognition.interimResults = true;   // Show words as they're spoken
recognition.lang = "en-US";

// CRITICAL: Auto-stop after 2 seconds of silence
let silenceTimer = null;
const SILENCE_TIMEOUT = 2000;

recognition.onresult = (e) => {
  // Update transcript display
  // Reset silence timer on every new result
  clearTimeout(silenceTimer);
  silenceTimer = setTimeout(() => {
    recognition.stop(); // Auto-submit after 2s silence
  }, SILENCE_TIMEOUT);
};

// Don't start listening until TTS audio fully finishes + 500ms delay
// Otherwise the mic picks up the AI's own voice
```

---

## THE 5 QUESTIONS

```
1. BUDGET
   AI says: "Hey there! Let's plan your perfect trip. First up — what's your rough daily budget in dollars?"
   Extracts: budget_per_day (number)

2. INTERESTS  
   AI says: "Nice! Now tell me — what are you into? Beaches, mountains, nightlife, culture, food, adventure? List as many as you want."
   Extracts: interests (array of strings)

3. ACCOMMODATION
   AI says: "Got it. Where do you like to stay? Hostel, Airbnb, mid-range hotel, or go all out luxury?"
   Extracts: accommodation_style (string)

4. PACE
   AI says: "Almost done. Do you like a relaxed trip with lots of free time, a moderate pace, or do you want a packed schedule with tons of activities?"
   Extracts: pace (string: "relaxed" | "moderate" | "packed")

5. DIETARY / ACCESSIBILITY
   AI says: "Last one — any dietary restrictions or accessibility needs I should know about? Say none if you're all good."
   Extracts: dietary_restrictions (string), accessibility_needs (string)
```

## CONTEXTUAL ACKNOWLEDGMENTS

After each answer, the AI gives a short acknowledgment before the next question. These should feel natural, not robotic.

```
Budget answered with a number → "Sweet, got your budget locked in."
Budget answered vaguely → "Alright, I'll work with that."
Interests include "beach" → "Beach vibes, love it. Next question."
Interests include "mountain" → "Mountains, nice choice. Moving on."
Interests general → "Great picks! Let's keep going."
Accommodation luxury/hotel → "Living the good life! Almost done."
Accommodation other → "Noted. Almost there."
Pace answered → "Perfect, that helps a lot."
No speech detected → "I didn't catch that, but let's keep going."
Final completion → "Awesome, I've got everything! Your preferences are locked in. The agents will start planning your trip now. Have a great one!"
```

---

## ENVIRONMENT VARIABLES NEEDED

```
ELEVENLABS_API_KEY=xi-...         # Get from elevenlabs.io → Profile → API Keys
ELEVENLABS_VOICE_ID=21m00Tcm4TlvDq8ikWAM   # Rachel (default), or any voice ID
```

---

## CRITICAL IMPLEMENTATION NOTES

1. **ElevenLabs key goes in backend .env ONLY** — frontend never sees it. Frontend calls your FastAPI backend, backend calls ElevenLabs.

2. **500ms delay between TTS ending and mic starting** — without this, the mic picks up the tail end of the AI's voice and transcribes it as user input.

3. **2-second silence auto-stop is essential** — user says "300 dollars", stops talking, and after 2s the mic auto-submits. Without this, the user has to manually press a button every time which kills the conversational feel.

4. **Browser speech recognition only works in Chrome/Edge** — show a message if user is on Safari/Firefox telling them to switch browsers, and fall back to text input.

5. **The extracted preferences MUST match the existing form schema exactly** — the rest of the pipeline (agent planning, itinerary generation, budget analysis) expects a specific format. Voice onboarding is just a different input method for the same data.

6. **Always keep the text input fallback** — some users won't have a mic, some will be in public, some browsers won't support it. The toggle between voice and typing should be seamless.

7. **ElevenLabs free tier gives ~10K characters/month** — each full onboarding run uses ~800-900 characters. For production, consider caching common question audio files so you only call ElevenLabs for dynamic acknowledgments.

8. **Audio analysis for the orb** — when playing ElevenLabs audio, create an AudioContext → MediaElementSource → AnalyserNode to get frequency data. Feed the average amplitude (0-1) to the orb component for the pulsing animation.

---

## INTEGRATION CHECKLIST

- [ ] Backend: `elevenlabs.py` service with streaming TTS
- [ ] Backend: `voice_onboarding.py` agent with state machine + Gemini extraction
- [ ] Backend: `voice.py` router with /start, /respond, /tts, /complete endpoints
- [ ] Frontend: `VoiceOnboarding.jsx` page with full conversation loop
- [ ] Frontend: `VoiceOrb.jsx` animated component
- [ ] Frontend: Voice ↔️ Text mode toggle
- [ ] Frontend: 2-second silence auto-stop on mic
- [ ] Frontend: 500ms delay between TTS end and mic start
- [ ] Frontend: Browser TTS fallback when ElevenLabs fails
- [ ] Frontend: Text input fallback when mic unavailable
- [ ] Frontend: Real-time extracted preferences card
- [ ] Frontend: Progress dots showing question progress
- [ ] Frontend: Conversation log showing full Q&A history
- [ ] Database: Save voice-gathered preferences in same schema as form
- [ ] Routing: Add voice onboarding as option on the join/preferences flow
- [ ] ENV: ELEVENLABS_API_KEY and ELEVENLABS_VOICE_ID in .env

---

## DESIGN TOKENS (match existing TripSync theme)

```
Background: #0c0a09
Text primary: #fafaf9
Text secondary: #a8a29e
Text dim: #78716c
Accent (speaking/AI): #f0a878
Accent (listening/user): #6ee7b7
Card bg: rgba(255,255,255,0.04)
Card border: rgba(255,255,255,0.08)
Error: #fca5a5
Font: DM Sans (already in the project)
```