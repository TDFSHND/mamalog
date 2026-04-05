# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

mamalog is a single-page PWA for parenting support. Users chat with an AI assistant about parenting concerns, and conversations are automatically summarized into diary entries. Japanese-only UI.

## Tech Stack

- **Frontend**: Single `index.html` file (vanilla JS, no build step)
- **Backend**: Netlify Functions (`netlify/functions/chat.js`) — proxies Claude API requests with server-side `ANTHROPIC_API_KEY`
- **Database/Auth**: Supabase (email/password auth, PostgreSQL)
- **AI**: Claude API (claude-opus-4-5)
- **Hosting**: Netlify (static site + serverless functions)

## Deployment

Push to `main` branch → Netlify auto-deploys. No build step required.

**Required Netlify env var**: `ANTHROPIC_API_KEY`

## Architecture

### Data Flow
1. User sends message → frontend pushes to `history` array
2. Request goes to `/.netlify/functions/chat` → proxied to Claude API
3. Response rendered in chat, `history` updated
4. Chat log upserted to Supabase `chat_logs` (one row per user per day, keyed by `user_id` + `chat_date`)
5. Full conversation sent to Claude for diary summarization → upserted to `diary_entries` (one row per day)

### Auth Flow
Login → check `user_metadata.child_birthday` → if missing, show birthday input screen → save via `sb.auth.updateUser({ data: { child_birthday } })` → show app

### Daily Lifecycle
- Same-day: chat log restored from Supabase on page load, diary updated after each exchange
- Next-day login: `loadChatLog()` finds old logs, converts them to diary via `autoSaveFromMessages()`, deletes the logs, starts fresh session

## Supabase Tables

### `chat_logs`
- `user_id` (uuid), `chat_date` (date), `messages` (jsonb array of `{role, content}`)
- Unique constraint: `(user_id, chat_date)`
- RLS: users can only access own rows

### `diary_entries`
- `user_id` (uuid), `summary` (text), `question` (text), `child_age` (int), `topics` (jsonb array), `created_at` (timestamptz)
- Daily upsert logic: query by `user_id` + date range on `created_at`, update if exists, insert if not

## Key Implementation Details

- **Viewport/keyboard handling**: Uses `interactive-widget=resizes-content` meta tag + `visualViewport` API to keep input fixed above mobile keyboard. Uses `scrollTop` (not `scrollIntoView`) to prevent page-level scroll.
- **Child age**: Calculated from `child_birthday` in user metadata via `calcMonthAge()`. Not detected from conversation text.
- **Markdown**: Only `**bold**` → `<strong>` conversion is implemented. No full markdown parser.
- **Topic detection**: Keyword-based matching in `detectContext()` — categories: 睡眠, 食事, 発達, 体調, 気持ち.
- **Diary generation prompt**: Asks Claude for JSON `{summary, question}` with character limits. The `autoSave()` function uses all conversation history, not just recent messages.
- **Service Worker** (`sw.js`): Caches static assets. Network-first for API/Supabase/Google Fonts requests.

## Git Conventions

- All changes are committed and pushed per user request (no auto-push)
- Commit messages in English, concise
