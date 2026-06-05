# Kahoot Codebase Scan

Date scanned: 2026-06-05

## Project Summary

This repository is a compact Supabase-backed live quiz app named `kahoot-alternative` / `SupaQuiz`. It already has a basic Kahoot-style flow:

- Host selects a seeded quiz set from a dashboard.
- Host starts a `games` row.
- Players join a game by UUID route and nickname.
- Host starts the quiz.
- Players answer live questions.
- Supabase Realtime updates host/player screens.
- Final host results are read from a SQL view.

The current implementation is a useful prototype, but it does not yet support short game codes, durable authenticated leaderboard/history, proper guest-token identity, quiz authoring UI, session reports, or robust host/player route separation.

## Project Structure

```txt
.
├── public/
│   ├── background.jpg
│   └── default.png
├── src/
│   ├── app/
│   │   ├── game/[id]/
│   │   │   ├── lobby.tsx
│   │   │   ├── page.tsx
│   │   │   └── quiz.tsx
│   │   ├── host/
│   │   │   ├── dashboard/
│   │   │   │   ├── how-to/page.mdx
│   │   │   │   ├── layout.tsx
│   │   │   │   └── page.tsx
│   │   │   ├── game/[id]/
│   │   │   │   ├── lobby.tsx
│   │   │   │   ├── page.tsx
│   │   │   │   ├── quiz.tsx
│   │   │   │   └── results.tsx
│   │   │   └── page.tsx
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── constants.ts
│   ├── mdx-components.tsx
│   └── types/
│       ├── supabase.ts
│       └── types.ts
├── supabase/
│   ├── config.toml
│   ├── migrations/
│   │   ├── 20231215125748_remote_schema.sql
│   │   ├── 20240410082019_result_view.sql
│   │   └── 20240412023540_rls.sql
│   └── seed.sql
├── package.json
├── next.config.mjs
├── tailwind.config.ts
└── tsconfig.json
```

There is no `src/components` directory and no `src/lib` directory yet. UI and Supabase logic currently live directly inside route files.

## Framework And Routing Style

- Framework: Next.js `14.2.15`.
- Router: App Router under `src/app`.
- Components: mostly client components using `'use client'` at page level.
- MDX is enabled via `@next/mdx`; route page extensions include `md` and `mdx`.
- Styling: Tailwind CSS utility classes.
- Redirects in `next.config.mjs`:
  - `/` redirects permanently to `/host/dashboard`.
  - `/host` redirects permanently to `/host/dashboard`.

Current routes:

```txt
/
/host
/host/dashboard
/host/dashboard/how-to
/host/game/[id]
/game/[id]
```

Current route naming uses raw UUID game IDs in `[id]`. There are no current `/live/*` routes, `/teacher/*` routes, API route handlers, or server actions.

## Dependencies

Important runtime dependencies:

- `next`, `react`, `react-dom`
- `@supabase/supabase-js`
- `next-qrcode`
- `react-countdown-circle-timer`
- `react-confetti`
- `react-use`
- `@next/mdx`, `@mdx-js/*`

There is no component library, form library, query cache, schema validation library, or dedicated auth helper package currently installed.

## Existing Database And Client Setup

Supabase client is created in `src/types/types.ts`:

```ts
export const supabase = createClient<Database>(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
)
```

This client is imported directly by UI components. The app relies on browser-side Supabase access with the anon key and RLS. There is no server-side Supabase client, middleware, or route-handler service layer.

Generated database types live in `src/types/supabase.ts`. Convenience types are exported from `src/types/types.ts`.

## Existing Auth System

Supabase Auth is enabled in `supabase/config.toml`.

Current behavior:

- Host dashboard calls `supabase.auth.getSession()`.
- If no session exists, host dashboard calls `supabase.auth.signInAnonymously()`.
- Player lobby also signs users in anonymously if no session exists.
- `participants.user_id` is required and defaults to `auth.uid()`.
- `games.host_user_id` defaults to `auth.uid()` after the RLS migration.

There is no visible login page, logout flow, profile page, role system, teacher guard, or integration with a student/user system. The current "guest" behavior is Supabase anonymous auth, not a separate `guest_token` model.

## Existing Tables, View, And RLS

Current tables:

- `quiz_sets`
- `questions`
- `choices`
- `games`
- `participants`
- `answers`

Current view:

- `game_results`

Current stored function:

- `add_question(quiz_set_id uuid, body text, "order" int, choices json[])`

Current realtime publication includes:

- `games`
- `participants`
- `answers`

Current RLS summary:

- `quiz_sets`, `questions`, `choices`: public select.
- `games`: public select; insert/update only by `host_user_id`.
- `participants`: public select; insert only when `auth.uid() = user_id`.
- `answers`: public select; insert policy currently allows `with check (true)`.

Important current schema notes:

- `participants.user_id` is `not null`, so true non-auth guests are not supported.
- `answers.participant_id` has a default of `auth.uid()`, but it references `participants.id`; this default is likely incorrect because auth user IDs and participant IDs are different values.
- `answers.choice_id` was added after initial table creation.
- `game_results` calculates totals by joining answers to participants and games.
- There are no long-term score/history tables.
- There are no explicit indexes except primary keys, foreign keys, and unique constraints created by the schema.

## Existing API Routes And Server Actions

No API route handlers or server actions were found.

All mutations currently happen from client components using Supabase directly:

- Start game: insert into `games`.
- Start quiz/advance question: update `games`.
- Join lobby: insert into `participants`.
- Answer question: insert into `answers`.
- Results: select from `game_results`.

## Existing UI Components

There are no shared component files. UI is route-local.

Host side:

- `src/app/host/dashboard/layout.tsx`: header/sidebar layout.
- `src/app/host/dashboard/page.tsx`: quiz set list and "Start Game".
- `src/app/host/game/[id]/page.tsx`: host game state coordinator.
- `src/app/host/game/[id]/lobby.tsx`: waiting room, participant list, QR code, start button.
- `src/app/host/game/[id]/quiz.tsx`: question display, answer count, timer, answer distribution, next button.
- `src/app/host/game/[id]/results.tsx`: final leaderboard with confetti.

Player side:

- `src/app/game/[id]/page.tsx`: player game state coordinator.
- `src/app/game/[id]/lobby.tsx`: nickname registration and waiting state.
- `src/app/game/[id]/quiz.tsx`: mobile answer screen and feedback.

UI style is direct Tailwind classes, simple colored buttons/cards, and inline SVG icons.

## State Management Style

State management is local React state:

- `useState` for current screen, questions, participants, selected answer, timers, and results.
- `useEffect` for initial fetches and Realtime subscriptions.
- `useRef` for avoiding stale participant/answer state inside Realtime callbacks.

There is no global store, context provider, reducer, URL-state helper, or query cache.

## Reusable Utilities

Current reusable files:

- `src/constants.ts`
  - `TIME_TIL_CHOICE_REVEAL = 5000`
  - `QUESTION_ANSWER_TIME = 20000`
- `src/types/types.ts`
  - Supabase client
  - Type aliases for current tables/views
- `src/mdx-components.tsx`
  - MDX component mapping

The app would benefit from adding reusable modules later, such as:

- `src/lib/supabase/client.ts`
- `src/lib/live-quiz/game-code.ts`
- `src/lib/live-quiz/scoring.ts`
- `src/lib/live-quiz/guest-token.ts`
- `src/components/live-quiz/*`

## Current Quiz/Game Flow

Host flow:

1. Host opens `/host/dashboard`.
2. Dashboard fetches all `quiz_sets` with nested questions and choices.
3. Host clicks "Start Game".
4. App ensures an auth session via anonymous sign-in.
5. App inserts a row into `games`.
6. App opens `/host/game/{gameId}` in a new tab.
7. Host game page fetches game and quiz set.
8. Host subscribes to participants inserts and games updates.
9. Host lobby shows participant nicknames and a QR code to `/game/{gameId}`.
10. Host updates game phase to `quiz`.
11. Host quiz subscribes to answer inserts for the current question.
12. Host reveals answers when time expires or all participants answer.
13. Host advances question or sets phase to `result`.
14. Host results select from `game_results`.

Player flow:

1. Player opens `/game/{gameId}`.
2. Player lobby ensures anonymous auth.
3. If an existing `participants` row exists for `(game_id, user_id)`, it is reused.
4. Otherwise player enters nickname and inserts `participants`.
5. Player page subscribes to `games` updates.
6. On quiz phase, player fetches all questions and choices.
7. Player waits for the choice reveal delay.
8. Player answers by inserting into `answers`.
9. Player sees correct/incorrect feedback when host reveals answers.
10. On result phase, player sees a simple thank-you screen.

## Incomplete Or Broken Code

Items found during scan:

- No true guest-token mode. Current guest behavior creates a Supabase anonymous auth user.
- No optional login mode UI. Users are always silently anonymous unless an existing session is present.
- No short game code. URLs and QR codes expose raw UUID game IDs.
- No quiz authoring UI for create/edit/delete quiz sets, questions, or choices.
- No session report route beyond final host results.
- No long-term leaderboard/history tables.
- No authenticated player profile linkage except raw `auth.users.id`.
- No route guards for teacher routes. Anonymous users can become hosts.
- No server-side validation for phase transitions or scoring.
- `answers` insert RLS uses `with check (true)`, which is too permissive.
- `participants` public select exposes all nicknames and user IDs to everyone.
- `answers` public select exposes all answers.
- `answers.participant_id default auth.uid()` is inconsistent with `participants.id`.
- Player result screen contains mojibake/corrupted text: `๏ผ`, `๐`.
- Seed data contains mojibake/corrupted non-English text.
- `src/app/game/[id]/lobby.tsx` imports `on` from Node `events` but does not use it.
- Some fetch error handlers recursively retry without backoff.
- `quizSet!` and `participant!` non-null assertions can crash if realtime updates arrive before data loads.
- Host answer subscription filters only by `question_id`, not `game_id`; if the same quiz question is used in multiple live games, answer counts can mix across sessions.
- Player-side timer uses local client time, so scoring can vary by device timing and reconnect behavior.
- Final player screen does not show leaderboard or personal rank.
- `README.md` says `/` joins as a player, but `next.config.mjs` redirects `/` to `/host/dashboard`.

## Naming Conventions

Observed conventions:

- Database table names use plural snake_case: `quiz_sets`, `questions`, `choices`, `games`, `participants`, `answers`.
- Database columns use snake_case: `created_at`, `quiz_set_id`, `current_question_sequence`.
- React components use PascalCase: `Lobby`, `Quiz`, `Results`.
- Route folders use lower-case names and App Router dynamic segments: `[id]`.
- Type aliases use PascalCase: `Participant`, `QuizSet`, `GameResult`.
- CSS is inline Tailwind utility classes.
- Current host terminology uses `host`; requested future routes use `teacher`.

## Key Files For Future Implementation

Most important existing files:

- `supabase/migrations/20231215125748_remote_schema.sql`
- `supabase/migrations/20240412023540_rls.sql`
- `supabase/migrations/20240410082019_result_view.sql`
- `src/types/types.ts`
- `src/types/supabase.ts`
- `src/app/host/dashboard/page.tsx`
- `src/app/host/game/[id]/page.tsx`
- `src/app/host/game/[id]/lobby.tsx`
- `src/app/host/game/[id]/quiz.tsx`
- `src/app/host/game/[id]/results.tsx`
- `src/app/game/[id]/page.tsx`
- `src/app/game/[id]/lobby.tsx`
- `src/app/game/[id]/quiz.tsx`

## Recommended Direction

Keep the existing App Router + Supabase Realtime architecture, but evolve the data model and route structure carefully:

- Add true `quiz_sessions` and `quiz_players` semantics rather than overloading `games` and `participants`.
- Add short `game_code`.
- Add nullable `user_id` and nullable `guest_token` identity.
- Move scoring and phase validation into database functions or server route handlers.
- Keep player routes public.
- Add proper teacher auth before allowing quiz authoring/session control.
- Add shared live quiz utilities/components incrementally instead of rewriting the app.
