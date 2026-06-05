# Kahoot Implementation Plan

Date planned: 2026-06-05

## Goals

Build a maintainable Kahoot-style live quiz system with:

- Public guest play without login.
- Optional authenticated play with long-term score/history.
- Teacher/host quiz authoring and live session controls.
- Supabase Realtime-powered classroom gameplay.
- Mobile-first player screens and projector-friendly host screens.

The repo should not be rewritten. The existing Next App Router, Tailwind, Supabase client, Realtime subscriptions, and current game pages should be evolved in small steps.

## Recommended Implementation Order

1. Database foundation
2. Supabase types and reusable live-quiz utilities
3. Public join flow with game code and guest token
4. Host session flow improvements
5. Player live gameplay improvements
6. Quiz authoring
7. Reports and long-term leaderboard/history
8. Route guards and RLS hardening

This order keeps the core identity/session model stable before UI expansion.

## Core Data Model

Use hybrid player identity:

```sql
user_id uuid null
guest_token text null
nickname text not null
session_id uuid not null
score int default 0
```

Rules:

- Authenticated player: `user_id` is set, `guest_token` may be null.
- Guest player: `guest_token` is set, `user_id` is null.
- Exactly one of `user_id` or `guest_token` should be present.
- Guest token is generated client-side and stored in `localStorage`.
- Guest scores live only inside the live session.
- Authenticated scores can be copied into long-term history/leaderboards when the session ends.
- Guest rejoin should find the same `quiz_players` row by `(session_id, guest_token)`.
- Authenticated rejoin should find the same row by `(session_id, user_id)`.

## Database Plan

The current schema has `quiz_sets`, `questions`, `choices`, `games`, `participants`, and `answers`. The clean target can either rename tables or add new tables. To minimize disruption, add new tables and migrate UI gradually. Later, old `games`/`participants` can be retired.

### `quiz_sets`

Purpose: Teacher-owned quiz definition.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `owner_user_id uuid not null references auth.users(id)`
- `title text not null`
- `description text null`
- `cover_image_url text null`
- `visibility text not null default 'private'`
- `is_archived boolean not null default false`

Relationships:

- Has many `quiz_questions`.
- Has many `quiz_sessions`.

Indexes:

- `(owner_user_id, created_at desc)`
- `(visibility)`

Security/RLS:

- Public can select only public/published quiz metadata if needed.
- Teacher can select/insert/update/delete their own quiz sets.
- Service role can manage all.

### `quiz_questions`

Purpose: Ordered questions inside a quiz set.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `quiz_set_id uuid not null references quiz_sets(id) on delete cascade`
- `body text not null`
- `image_url text null`
- `order_index int not null`
- `time_limit_ms int not null default 20000`
- `choice_reveal_delay_ms int not null default 5000`
- `points int not null default 1000`
- `type text not null default 'multiple_choice'`

Relationships:

- Belongs to `quiz_sets`.
- Has many `quiz_choices`.
- Has many `quiz_answers`.

Indexes:

- `(quiz_set_id, order_index)` unique.

Security/RLS:

- Teacher can manage questions for quiz sets they own.
- Players can read questions only for active sessions they joined; avoid exposing correct choices until needed.

### `quiz_choices`

Purpose: Answer options for each question.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `question_id uuid not null references quiz_questions(id) on delete cascade`
- `body text not null`
- `image_url text null`
- `order_index int not null`
- `is_correct boolean not null default false`

Relationships:

- Belongs to `quiz_questions`.
- Referenced by `quiz_answers`.

Indexes:

- `(question_id, order_index)` unique.
- `(question_id, is_correct)`

Security/RLS:

- Teacher can manage choices for their questions.
- Players can read choice bodies for active current questions.
- Correctness should be hidden from players until answer reveal, preferably via a view/RPC instead of direct table select.

### `quiz_sessions`

Purpose: One live run of a quiz.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `quiz_set_id uuid not null references quiz_sets(id)`
- `host_user_id uuid not null references auth.users(id)`
- `game_code text not null unique`
- `phase text not null default 'lobby'`
- `current_question_index int not null default 0`
- `current_question_started_at timestamptz null`
- `answer_revealed_at timestamptz null`
- `ended_at timestamptz null`
- `settings jsonb not null default '{}'::jsonb`

Valid phases:

- `lobby`
- `question_intro`
- `answering`
- `question_result`
- `leaderboard`
- `ended`

Relationships:

- Belongs to `quiz_sets`.
- Has many `quiz_players`.
- Has many `quiz_answers`.
- Has many `quiz_scores`.

Indexes:

- unique `(game_code)`
- `(host_user_id, created_at desc)`
- `(quiz_set_id, created_at desc)`
- `(phase)`

Security/RLS:

- Public can select active session by `game_code` with limited columns.
- Host can insert/update their own sessions.
- Players can select sessions they joined.
- Phase transitions should be restricted to host or a security-definer RPC.

### `quiz_players`

Purpose: A participant in one live session.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `session_id uuid not null references quiz_sessions(id) on delete cascade`
- `user_id uuid null references auth.users(id) on delete set null`
- `guest_token text null`
- `nickname text not null`
- `score int not null default 0`
- `answered_count int not null default 0`
- `last_seen_at timestamptz not null default now()`
- `is_connected boolean not null default true`

Constraints:

- check exactly one of `user_id`, `guest_token` is not null.
- unique `(session_id, user_id)` where `user_id is not null`.
- unique `(session_id, guest_token)` where `guest_token is not null`.

Relationships:

- Belongs to `quiz_sessions`.
- Has many `quiz_answers`.
- Has many `quiz_scores`.

Indexes:

- `(session_id, score desc, created_at asc)`
- `(session_id, nickname)`
- `(user_id, created_at desc)` where `user_id is not null`.
- `(guest_token)` where `guest_token is not null`.

Security/RLS:

- Public/anon can insert guest player only with a valid active `session_id` and `guest_token`.
- Auth user can insert themselves with `auth.uid() = user_id`.
- Player can update their own `last_seen_at`/nickname before game start.
- Host can read players for sessions they own.
- Avoid public global reads.

### `quiz_answers`

Purpose: Player answer for one question in one session.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `session_id uuid not null references quiz_sessions(id) on delete cascade`
- `player_id uuid not null references quiz_players(id) on delete cascade`
- `question_id uuid not null references quiz_questions(id) on delete cascade`
- `choice_id uuid null references quiz_choices(id) on delete set null`
- `is_correct boolean not null default false`
- `score_delta int not null default 0`
- `answered_at timestamptz not null default now()`
- `response_time_ms int null`

Constraints:

- unique `(session_id, player_id, question_id)`.

Relationships:

- Belongs to `quiz_sessions`.
- Belongs to `quiz_players`.
- Belongs to `quiz_questions`.
- Optionally belongs to `quiz_choices`.

Indexes:

- `(session_id, question_id, created_at)`
- `(session_id, player_id)`
- `(choice_id)`

Security/RLS:

- Player can insert one answer only for their own player row and the current active question.
- Host can read answers for sessions they own.
- Do not let clients submit `is_correct` or `score_delta` directly. Compute in an RPC or server action.

### `quiz_scores`

Purpose: Per-question scoring ledger for live sessions.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `session_id uuid not null references quiz_sessions(id) on delete cascade`
- `player_id uuid not null references quiz_players(id) on delete cascade`
- `question_id uuid not null references quiz_questions(id) on delete cascade`
- `answer_id uuid null references quiz_answers(id) on delete set null`
- `score_delta int not null`
- `total_score int not null`
- `reason text not null default 'answer'`

Constraints:

- unique `(session_id, player_id, question_id, reason)`.

Relationships:

- Mirrors scoring changes and supports reports.

Indexes:

- `(session_id, player_id, created_at)`
- `(session_id, total_score desc)`

Security/RLS:

- Host can read for owned sessions.
- Player can read their own score rows.
- Inserts should be through trusted RPC/server logic only.

### `leaderboard_entries`

Purpose: Long-term leaderboard/history for authenticated players after a session ends.

Columns:

- `id uuid primary key default gen_random_uuid()`
- `created_at timestamptz not null default now()`
- `quiz_set_id uuid not null references quiz_sets(id)`
- `session_id uuid not null references quiz_sessions(id) on delete cascade`
- `user_id uuid not null references auth.users(id) on delete cascade`
- `nickname text not null`
- `score int not null`
- `rank int not null`
- `answered_count int not null default 0`
- `correct_count int not null default 0`

Constraints:

- unique `(session_id, user_id)`.

Relationships:

- Belongs to authenticated user.
- Belongs to quiz set and session.

Indexes:

- `(quiz_set_id, score desc, created_at desc)`
- `(user_id, created_at desc)`
- `(session_id, rank)`

Security/RLS:

- Authenticated users can read their own history.
- Teachers can read entries for sessions they hosted.
- Public leaderboard should be opt-in and limited to display fields.
- Guests are not inserted here.

## RPC / Server Logic Plan

Prefer Supabase RPC functions or Next route handlers for operations that need validation:

- `create_quiz_session(quiz_set_id)`:
  - Requires authenticated host.
  - Generates unique short game code.
  - Inserts `quiz_sessions`.

- `join_quiz_session(game_code, nickname, guest_token)`:
  - Public.
  - Resolves session by game code.
  - Creates or reuses `quiz_players`.
  - Supports authenticated `auth.uid()` or guest token.

- `submit_quiz_answer(session_id, player_id, question_id, choice_id)`:
  - Validates current phase/question.
  - Rejects duplicate answer.
  - Calculates correctness and score from server time.
  - Inserts `quiz_answers`.
  - Inserts `quiz_scores`.
  - Updates `quiz_players.score`.

- `advance_quiz_session(session_id, next_phase, next_question_index)`:
  - Requires host ownership.
  - Controls phase transitions and timestamps.

- `finalize_quiz_session(session_id)`:
  - Requires host ownership.
  - Marks ended.
  - Writes authenticated players into `leaderboard_entries`.

## Suggested Routes

Use the current App Router style.

Teacher routes should require auth:

```txt
/teacher/live-quiz
/teacher/live-quiz/new
/teacher/live-quiz/[quizId]
/teacher/live-quiz/[quizId]/edit
/teacher/live-quiz/session/[sessionId]
/teacher/live-quiz/session/[sessionId]/report
```

Player routes should be public:

```txt
/live/join
/live/[gameCode]
/live/[gameCode]/play
/live/[gameCode]/leaderboard
```

Migration compatibility:

- Keep current `/host/dashboard` and `/host/game/[id]` working initially.
- Add redirects or links from `/host/dashboard` to `/teacher/live-quiz`.
- Keep old `/game/[id]` until `/live/[gameCode]` is stable.
- Do not remove old routes until replacement flows are complete.

## Realtime Plan

Supabase Realtime is available and already enabled in `supabase/config.toml`. Current realtime tables are `games`, `participants`, and `answers`; the new implementation should add:

```sql
alter publication supabase_realtime add table quiz_sessions;
alter publication supabase_realtime add table quiz_players;
alter publication supabase_realtime add table quiz_answers;
alter publication supabase_realtime add table quiz_scores;
```

Live events needed:

- Player joined: `INSERT` on `quiz_players` filtered by `session_id`.
- Player presence/last seen: `UPDATE` on `quiz_players`.
- Host started game: `UPDATE` on `quiz_sessions.phase`.
- Question changed: `UPDATE` on `quiz_sessions.current_question_index`.
- Answer submitted: `INSERT` on `quiz_answers` filtered by `session_id` and `question_id`.
- Answer count updated: derive from `quiz_answers` inserts, or subscribe to aggregate state if added.
- Score updated: `UPDATE` on `quiz_players.score` or `INSERT` on `quiz_scores`.
- Leaderboard updated: `UPDATE` on `quiz_players` filtered by session.
- Game ended: `UPDATE` on `quiz_sessions.phase = 'ended'`.

Recommended channel naming:

- `session:{sessionId}:host`
- `session:{sessionId}:player:{playerId}`
- `session:{sessionId}:leaderboard`

Keep filters specific to `session_id`. The current host answer listener filters only by `question_id`, which can mix answers if multiple sessions use the same quiz.

## Teacher / Host Side Features

### Create Quiz Set

Build under `/teacher/live-quiz/new`.

Minimum fields:

- Title
- Description
- Cover image URL optional
- Questions with choices

Implementation notes:

- Use existing Tailwind style first.
- Add shared form components only if repetition becomes meaningful.
- Save to `quiz_sets`, `quiz_questions`, `quiz_choices`.

### Add/Edit/Delete Questions

Build in `/teacher/live-quiz/[quizId]/edit`.

Support:

- Add question
- Edit question text/image/time limit
- Add/edit/delete choices
- Mark correct choices
- Reorder questions
- Delete question

Implementation notes:

- Start with simple form sections.
- Enforce at least two choices and at least one correct choice.
- Keep `order_index` stable after deletes/reorders.

### Start Live Quiz Session

From `/teacher/live-quiz` or `/teacher/live-quiz/[quizId]`.

Behavior:

- Requires authenticated host.
- Calls `create_quiz_session`.
- Opens `/teacher/live-quiz/session/[sessionId]`.
- Shows short `game_code` and QR code to `/live/[gameCode]`.

### Waiting Room

Host screen:

- Large game code
- QR code
- Joined player list
- Player count
- Start button

Realtime:

- Subscribe to `quiz_players` inserts/updates for the session.

### Start Question

Host controls:

- Start first question from lobby.
- Advance through phases:
  - `question_intro`
  - `answering`
  - `question_result`
  - `leaderboard`
  - next question or `ended`

### Live Answers

Host screen:

- Question text
- Answer countdown
- Answer count
- Per-choice answer distribution after reveal

Security:

- Do not expose correct answer to players before reveal.

### Question Result

Host screen:

- Correct choices highlighted
- Answer distribution
- Fastest/correct players optional later

### Live Leaderboard

Host screen:

- Top players by `quiz_players.score`.
- Update from score changes.
- Projector-friendly typography.

### End Game

Host action:

- Calls `finalize_quiz_session`.
- Sets phase to `ended`.
- Writes authenticated players to `leaderboard_entries`.

### Game Report

Route:

```txt
/teacher/live-quiz/session/[sessionId]/report
```

Report sections:

- Final leaderboard
- Per-question answer distribution
- Per-player answers and score changes
- Authenticated entries saved to history
- Guest entries marked session-only

## Player Side Features

### Join By Game Code

Route:

```txt
/live/join
```

Flow:

1. Enter game code.
2. Resolve active session.
3. Redirect to `/live/[gameCode]`.

Also support QR/deep link directly to `/live/[gameCode]`.

### Choose Guest Or Login Mode

Route:

```txt
/live/[gameCode]
```

If already authenticated:

- Show "Continue as profile" and optional nickname display.

If not authenticated:

- Show "Play as guest" immediately.
- Show optional "Log in for saved history".

Guest must not be blocked by login.

### Guest Nickname

Behavior:

- Generate `guest_token` if missing:
  - key: `supaquiz_guest_token`
  - value: crypto-random string
- Store nickname per session if useful:
  - key: `supaquiz_nickname:{sessionId}`
- Call `join_quiz_session`.

### Answer Live Questions

Route:

```txt
/live/[gameCode]/play
```

Behavior:

- Subscribe to `quiz_sessions`.
- Show waiting state until answering starts.
- Fetch current question payload.
- Submit answer through trusted RPC/server route.
- Disable answer buttons after submission.

### Result Feedback

Show:

- Correct/incorrect
- Points earned
- Current total score
- Waiting for next question

### Leaderboard

Route:

```txt
/live/[gameCode]/leaderboard
```

Player view:

- Top players
- Player's own rank
- Final state after game ends

### Rejoin If Disconnected

Behavior:

- Guest rejoin uses localStorage guest token.
- Auth rejoin uses `auth.uid()`.
- Reuse existing `quiz_players` row.
- Fetch latest session state and route to correct screen.

## Auth Plan

Short term:

- Add explicit login route only if needed by current auth provider setup.
- Keep Supabase Auth as the identity provider.
- Stop using anonymous sign-in as the only guest mechanism.

Teacher access:

- Require real authenticated user for `/teacher/*`.
- Do not allow anonymous users to create sessions or quizzes.
- Later integrate teacher/student roles through a profile table or existing user system.

Suggested future `profiles` table:

```txt
profiles
- id uuid primary key references auth.users(id)
- display_name text
- role text
- created_at timestamptz
- updated_at timestamptz
```

Role checks can be added once the existing student/user system is connected.

## UX Plan

Player UX:

- Mobile-first.
- Large game code input.
- Large nickname input.
- Four large answer buttons.
- Instant join feedback.
- Clear disconnected/reconnecting state.
- No required login.

Host UX:

- Classroom/projector friendly.
- Large game code and QR code in lobby.
- Large question text.
- Big answer count and countdown.
- High-contrast leaderboard.
- Controls should be obvious but not visually dominant on projector.

Reuse:

- Keep Tailwind.
- Extract repeated quiz color/button patterns after the new flow works.
- Avoid adding a full design system prematurely.

## Files To Add Or Change First

Database:

- Add a new migration under `supabase/migrations/`.
- Regenerate `src/types/supabase.ts`.

Utilities:

- `src/lib/supabase/client.ts`
- `src/lib/live-quiz/game-code.ts`
- `src/lib/live-quiz/guest-token.ts`
- `src/lib/live-quiz/scoring.ts`

Routes:

- `src/app/live/join/page.tsx`
- `src/app/live/[gameCode]/page.tsx`
- `src/app/live/[gameCode]/play/page.tsx`
- `src/app/live/[gameCode]/leaderboard/page.tsx`
- `src/app/teacher/live-quiz/page.tsx`
- `src/app/teacher/live-quiz/new/page.tsx`
- `src/app/teacher/live-quiz/[quizId]/page.tsx`
- `src/app/teacher/live-quiz/[quizId]/edit/page.tsx`
- `src/app/teacher/live-quiz/session/[sessionId]/page.tsx`
- `src/app/teacher/live-quiz/session/[sessionId]/report/page.tsx`

Shared components, when useful:

- `src/components/live-quiz/AnswerButton.tsx`
- `src/components/live-quiz/GameCodeDisplay.tsx`
- `src/components/live-quiz/Leaderboard.tsx`
- `src/components/live-quiz/QuestionDisplay.tsx`
- `src/components/live-quiz/Timer.tsx`

## Migration Strategy From Current Schema

Phase 1:

- Keep existing `quiz_sets`, but add ownership/metadata columns if preserving data is important.
- Create new `quiz_questions` and `quiz_choices`, or adapt existing `questions` and `choices`.
- Create `quiz_sessions`, `quiz_players`, `quiz_answers`, `quiz_scores`, `leaderboard_entries`.

Phase 2:

- Build new `/live/*` player flow against new tables.
- Build new `/teacher/live-quiz/session/*` host flow against new tables.
- Keep old `/game/[id]` and `/host/game/[id]` as legacy prototype routes.

Phase 3:

- Port dashboard/quiz listing to new teacher routes.
- Add authoring.
- Add reports.

Phase 4:

- Retire or redirect old routes after parity.

## Risks And Missing Information

- Auth provider choice is not defined. Supabase Auth is enabled, but there is no login UI.
- Teacher/student role model is not defined yet.
- Existing seed data has corrupted non-English text; decide whether to repair, replace, or ignore.
- RLS needs careful design before public guest writes are enabled.
- Realtime table subscriptions with broad public select policies can leak data unless narrowed.
- Scoring should use server timestamps to prevent client timing drift or manipulation.
- Correct answers should not be sent to player clients before reveal.
- Short game code generation needs collision handling.
- LocalStorage guest token is convenient but not secure identity; it is good enough for session rejoin, not for long-term trust.

## Build First

The first implementation milestone should be:

1. Add the new session/player/answer/score tables with RLS draft.
2. Add `game_code` session creation.
3. Add public guest join by code using localStorage `guest_token`.
4. Add answer submission through RPC/server validation.
5. Add a minimal host waiting room and player answer screen on the new routes.

That gives the app its core hybrid guest/auth foundation before investing in authoring and reports.
