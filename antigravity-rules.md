These are the rules for you:
# TripSync — Antigravity Project Rules

## Git Rules (MANDATORY)
- NEVER auto-commit or auto-push to GitHub
- After completing any working feature or phase, stop and ask me:
  - Show a brief summary of what changed
  - Ask "Ready to push? What commit message would you like?"
  - Wait for my explicit reply with the commit message
  - Only then run git add, commit, and push
- Every push must have my approval and my commit message

## Code Style
- TypeScript throughout, strict mode
- Next.js App Router conventions (app/ directory)
- Tailwind CSS for all styling, no custom CSS files unless necessary
- Components go in /components, organized by feature
- API routes go in /app/api/
- Reusable types go in /types/
- Utility functions go in /lib/
- Supabase client setup goes in /lib/supabase.ts
- Environment variables prefixed with NEXT_PUBLIC_ for client-side access

## AI Integration
- All Gemini API calls go through server-side API routes (never expose keys to client)
- ElevenLabs calls also server-side or via their SDK with proper key handling
- AI responses must always include reasoning/explainability
- Structure AI prompts as system + user messages, expect JSON responses

## Development Approach
- Build incrementally, phase by phase
- Each phase should produce a working, demoable state
- Mobile-first responsive design
- Keep the UI clean and accessible — no unnecessary complexity
- Test with the 4 fake user personas (Ron, Jon, Tom, Mike) defined in the project brief

## Error Handling
- Always handle loading states with visual indicators
- Handle API failures gracefully with user-friendly error messages
- Supabase queries should have error handling

## Dependencies — Only Add What We Need
- next, react, react-dom
- @supabase/supabase-js, @supabase/ssr
- @google/generative-ai (Gemini SDK)
- elevenlabs (ElevenLabs SDK)
- tailwindcss
- recharts or chart.js (for expense dashboard)
- framer-motion (only if time permits, for polish)
- Do not add unnecessary packages