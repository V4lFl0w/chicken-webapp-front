# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project: V-Hub — Telegram Mini App

A Telegram Mini App hub with multiple HTML-based games and a FastAPI backend.

## Architecture

### Monorepo layout
- `index.html` — hub entry point (game catalog, leaderboard, user profile)
- `frontend/` — auxiliary Mini App pages (e.g. `fly_invaders.html`)
- `games/` — individual game files, each a fully self-contained single HTML file (HTML + CSS + JS inline)
- `backend/` — Python FastAPI server (`main.py`, `database.py`, `quiz_manager.py`)

### Backend (`/backend`)
- **Runtime**: FastAPI + Uvicorn, synchronous SQLAlchemy (not async), PostgreSQL
- **Env vars required**: `DATABASE_URL` (postgres DSN, asyncpg prefix is stripped automatically), `OPENAI_API_KEY`
- **DB models**: `game_players` (telegram_id, nickname, high_score, coins), `quiz_questions` (q, a JSON array, c correct index)
- **Endpoints**: `POST /get_user`, `POST /save_score`, `GET /top_players`, `GET /get_quiz_questions`
- **Quiz auto-fill**: `quiz_manager.py` uses `asyncio.to_thread` to call OpenAI GPT-4o-mini in background; maintains a 1000-question bank, refills when below 100
- **Schema migration**: done ad-hoc via `ALTER TABLE ... ADD COLUMN` in `on_startup`, errors silently swallowed

#### Run backend
```bash
cd backend
source venv/bin/activate
DATABASE_URL=postgresql://... OPENAI_API_KEY=sk-... uvicorn main:app --reload
```

### Frontend / Games
- All games are **single HTML files** — no build step, no bundler, no separate JS modules
- **GSAP 3.12.2** loaded from CDN (`cdnjs.cloudflare.com`) for all animations
- **Telegram Web App SDK** loaded from `telegram.org/js/telegram-web-app.js`
- Touch input via **Pointer Events API** (`onpointerdown/move/up/cancel`) — not Touch Events
- Cards/elements are absolutely positioned; hand layout uses CSS `left: calc(50% + Xpx)` with inline `transform: rotate`
- `localStorage` used for persistent state (facts archive, cosmetics)

## Known bugs in `games/durak.html`

1. **`isAnimating` stuck** — in `botDefend()`, if the bot cannot defend (`botNeedsToTake = true` branch), `isAnimating` is never reset to `false`. Player input becomes permanently locked.

2. **`isAnimating` not reset in `botAttack` recursion** — when `playerNeedsToTake` is true and bot has more cards to throw, `botAttack` calls itself via `setTimeout` but doesn't clear `isAnimating = false` before the recursive call; on the final iteration when no card is played it reaches `executePlayerTake` which resets it, but any exception in between leaves it stuck.

3. **`executePlayerTake` uses unscoped selector** — `gsap.to('.bout-pair', ...)` matches all `.bout-pair` elements in the document; should target `#tableArea .bout-pair`.

4. **`refillHands` fills bot first** — standard Durak rules: attacker refills first, then defender. Currently bot always refills before player regardless of who attacked.

5. **`updateSelectionVisuals()` rebuilds entire hand DOM on every state change** — called after every card play, bot action, etc. On mobile this causes layout thrashing. Cards should be updated incrementally or the hand rendered once and only transforms updated.

6. **`checkGameState` only triggers on empty deck** — if a player empties their hand mid-round (after the deck runs out), the win condition is not checked until the next `refillHands` cycle.

## Mobile performance guidelines
- Avoid `innerHTML = ''` + full re-render inside tight loops or rapid-fire callbacks
- Prefer `gsap.set()` / `gsap.to()` for transform changes over toggling inline `style.transform`
- Keep `will-change: transform` on draggable card elements; remove it after drag ends
- `touch-action: none` on the game container is correct; do not add `pointer-events: none` to parent layers that need to propagate events
- Batch DOM reads before writes; never interleave them inside animation callbacks
