# Has To Be A Cat — Multiplayer

Real-time 2-player version of the cat-region puzzle. Both players get the
identical board; first to place all 8 cats correctly wins.

## 1. Create a Supabase project

1. Go to https://supabase.com, sign in, and create a new project.
2. Open **SQL Editor** in the dashboard, paste the contents of
   `supabase/schema.sql`, and run it. This creates the `rooms` table.
3. Go to **Project Settings → API** and copy:
   - **Project URL**
   - **anon public** key

No extra Realtime setup is needed — this app uses Broadcast + Presence,
which work out of the box and don't require enabling table replication.

## 2. Configure environment variables

```bash
cp .env.local.example .env.local
```

Edit `.env.local` and paste in your Project URL and anon key:

```
NEXT_PUBLIC_SUPABASE_URL=https://your-project-ref.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-public-key
```

## 3. Install & run

```bash
npm install
npm run dev
```

Open http://localhost:3000 in two different browser tabs (or share the
room link with a friend on another device) to test a match.

## How it works

- **`lib/cat-puzzle.js`** — pure puzzle logic (region generation, conflict
  checking), ported from the single-player game. No DOM/React dependency.
- **`app/page.js`** — lobby: "Create Match" generates a 6-character room
  code + a puzzle, inserts one row into `rooms`, then navigates to
  `/room/[code]`. "Join Match" looks up an existing code.
- **`app/room/[code]/page.js`** — the game itself:
  - Fetches the room's puzzle from Postgres so both players render the
    exact same board.
  - Opens a Supabase Realtime channel scoped to the room code, using
    **Presence** to detect when both players are connected.
  - Once 2 presences are seen, the host computes a shared start
    timestamp (`Date.now() + 3000ms`), writes it to the room row, and
    **Broadcasts** a `start` event. Both clients derive the same 3-2-1
    countdown and timer start from that one timestamp.
  - The instant a player completes the board validly, it **Broadcasts**
    a `win` event with their player id. Both clients freeze their board
    and timer immediately on receipt — the winner sees "You Won! 🏆",
    the other sees "You Lost 😢".

## Known limitations (by design, given scope)

- No authentication — anyone with a room code can join. Fine for a
  casual link-sharing game between friends.
- No rematch-in-place; "Back to Lobby" starts a fresh room/code.
- Countdown/timer sync assumes both players' system clocks are roughly
  correct (no NTP-style offset correction). Good enough for a casual
  match; a few hundred ms of skew won't affect who wins, since winning
  is decided by whoever's board completes first, not by clock reads.
- Running out of lives (3 wrong guesses) doesn't end the match — it's
  kept as visual feedback only, since the multiplayer win condition is
  strictly "first to solve," not "survive mistakes." Easy to change if
  you'd rather it end the match too.
- Rooms aren't automatically deleted. The commented-out cleanup query
  in `supabase/schema.sql` can be run manually or wired to a cron job.
