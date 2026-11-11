**English** | [中文](./README_CN.md)

# Reso

**An AI companion for an international audience. Its memory is visible and editable, and it asks before keeping anything new. Built solo.**

→ [reso.love](https://reso.love) (invite-only beta)

Most AI companions keep their memory out of sight: users can't see what was kept, or what was got wrong. Reso writes down everything a character remembers on a memory page, in plain words, where the user can edit or delete it. When a character wants to remember something new, it asks the user first.

> Stack: Vue 3 + Express + PostgreSQL + Qdrant · Users bring their own model key (OpenAI / Anthropic / Gemini / DeepSeek / custom endpoint)
> From zero to live in under a month, with ongoing work on the memory system and evaluation since

---

## Features

| Feature | Implementation |
|---|---|
| Visible memory | The memory page lists everything a character remembers; every entry can be edited or deleted |
| Asks before keeping | New memories for user-made characters wait for confirmation; when the same thing comes up worded differently, old and new are merged into one and the user picks **Remember this / Keep both / Discard** |
| "Remember this" | Say it directly and it's saved immediately, without waiting for the background step |
| Portrait | A short "who this person is", drawn from all the memories |
| Diary | Every day, at a time the user sets, the character writes an entry in its own voice |
| Recall | Before each reply: the portrait, relevant memories, moments from past conversations, and recent summaries; now and then, an old moment surfaces on its own (flashback) |
| Characters | Make your own, or import a SillyTavern card (PNG / JSON, lorebook included); Reso's own character, Casino, has a past of his own and relationship stages |
| Bring your own model | Four providers plus custom endpoints; background work can run on a separate, cheaper model |
| Data export | Chats, memories, portraits, diaries and user-made character cards in one zip; cards use the standard format |
| Feedback | Echo, a feedback board: posts, categories, votes |

---

## Architecture

```
Browser (Vue 3, Cloudflare Pages)
        │
        ▼
Cloudflare ──▶ nginx ──▶ Express
                          ├ production / test (two separate processes)
                          ├ after each reply: summary → memory candidates → same-or-not check → portrait
                          ├ scheduled jobs: diaries
                          └ calls the user's own model (four providers behind one interface)
                                │
                ┌───────────────┼────────────────┐
           PostgreSQL         Qdrant         Jina Embeddings
  (users, chats, memories) (memory vectors)  (text → vectors)
```

---

## Memory pipeline

1. **Extraction**: after a stretch of conversation, or when the user comes back after a while, a background step picks out what's worth keeping long-term from recent messages
   - Only what the user said or confirmed; the character's jokes, guesses and reminders don't count
   - "Tomorrow" and "Friday" become real dates, in the user's time zone
   - Things that only matter for one day aren't kept; neither are passwords, verification codes, card numbers or addresses
   - Written in the language the user speaks
2. **Same or not**: vector search finds similar existing memories, then a small model checks them one by one
3. **Storage**: for user-made characters, a pending memory or a merge card; for Reso's own characters, merged directly
4. **Recall**: before a reply, memories are ranked by relevance, then filtered for cooldown, mood fit and source variety, so the character doesn't keep bringing up the same thing
5. **Upkeep**: the portrait updates as new memories come in; diaries are written daily

---

## Key decisions and trade-offs

- **Ask first instead of saving automatically**: memory belongs to the user, so a wrong memory has to be visible and stoppable. The cost is a few extra taps, which is why Reso's own characters take the other path and merge quietly
- **A small model decides "same thing or not"**: vector similarity can't tell. "Taking cephalosporin" and "wisdom tooth out on Friday" look alike but are two things; "dentist on the 7th" and "moved to the 6th" differ by a date but are one. So vectors only find candidates and a small model makes the call. If the call fails, the two are treated as different: an extra memory is better than a wrong merge
- **Users bring their own model key**: users choose their model and control their own spending, and Reso carries no model costs. The cost is handling every provider's quirks: reasoning models can spend their whole output budget on thinking and return an empty answer, which silently broke "remember this". Background calls now run with thinking turned off
- **Recordings on the test server, never in production**: a recording can replay a failure down to the exact step, but it holds the full conversation. Real users' conversations are never recorded; only simulated users on the test server are
- **Our own recorder, not just Langfuse**: recordings go into our own database and show up in the admin dashboard; Langfuse stays on the test server as a point of comparison

---

## Testing and observability

- **Memory regression tests**: on the test server, simulated users chat with characters and drop 20 details along the way; afterwards we check how many were kept. Latest run: 19/20
- **Call recordings**: on the test server, every model call in every turn (reply, extraction, merge, same-or-not) is recorded with its input and output, so a failure can be replayed to the step where it went wrong, and Reso's mistakes can be told apart from the test script's
- **Test report page**: scores and recordings for every run, in the admin dashboard

---

## Deployment and security

- Production and test are fully separate: processes, databases and vector collections. Every change runs through full conversations on the test server first
- One-command deploy: pull the code, install dependencies only if they changed, restart both servers, confirm both are healthy
- The server only accepts key-based login; security patches install automatically; traffic is encrypted all the way from Cloudflare to the server; the server, domain and code-hosting accounts all have two-factor authentication
- A daily automatic check-up emails me if anything is wrong; on quiet weeks it still sends an "all good" every Sunday, so a missing email means the server itself is down
- Users' model keys are stored with AES-256-GCM encryption; sign-in uses a short-lived and a long-lived token; sign-in, chat, import, export and admin each have their own rate limits
- Admin dashboard: users, memories, invite codes, system health, error logs, usage stats

---

## Tech stack

| Layer | Technology |
|------|------|
| Frontend | Vue 3, Vite, Pinia |
| Backend | Node.js, Express, node-cron |
| Data | PostgreSQL, Qdrant, Jina Embeddings |
| LLMs | OpenAI / Anthropic / Gemini / DeepSeek behind one interface |
| Deployment | Cloudflare Pages (frontend), Cloudflare, nginx, PM2 |

---

## Known issues

- Lorebooks match on keywords, so they don't trigger when the card's language differs from the user's
- The app doesn't support phones yet; on a phone, only the landing page is available

---

## Project status

A commercial product in invite-only beta, not open source. This repository is a portfolio piece describing its product design and engineering.

**Solo-built**: product, characters, visuals, frontend, backend, deployment and operations, all by one person. I design the architecture and data flow; the implementation is written with AI coding tools such as Claude Code.

📮 delanri1107@gmail.com · Ask me for an invite code
