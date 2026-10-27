# Larry — Discord AI Community Bot (custom build)

A Discord AI coach/community bot built the **custom** way: `discord.js` for Discord +
the **Vercel AI SDK** (`ai`) for the model and tool-calling, running on **Kimi**
(Moonshot) — no OpenClaw, no cli-bridge. We own the whole agent loop, so tool-calling
is reliable and the bot deploys anywhere.

> 📘 **New here / want to build your own?** Read **[`SOP.md`](./SOP.md)** — the full
> step-by-step: Discord Developer Portal setup, wiring the bot, the model, the tools,
> the persona, running it, deploying it, and the gotchas. This README is the quick start;
> `SOP.md` is the complete guide.

## What it does
Full tool set for a coaching/community server. Privileged tools are gated **in code**, not the prompt.

| Tool | Who | What |
|---|---|---|
| `kb_search` | anyone | Answer from the client's KB only |
| `community_pulse` | anyone | Recent activity in the public channels |
| `request_invoice` | anyone | Ping the finance coach for an invoice/receipt |
| `create_forum` | coaches | New student's private 1:1 forum + standard posts |
| `add_member_to_forum` | coaches | Grant forum access |
| `remove_member_from_forum` | coaches | Revoke forum access |
| `find_student_forum` | coaches | Locate a student's forum |
| `find_inactive_members` | coaches | Going-quiet report |
| `post_announcement` | coaches | Post to a channel |

## Architecture (one process)
```
Discord ──▶ index.ts (discord.js: gating, @mention)
                 │
                 ▼
            agent.ts (Vercel AI SDK: generateText + tools + maxSteps)
                 │            │
     model.ts (Kimi)   tools/ (create_forum, kb_search)  ──▶ Discord REST / KB files
```

## Run it
```bash
npm install
cp .env.example .env      # fill in the token, guild, Kimi key, category/role IDs
npm run dev               # hot-reload; use `npm start` for prod
```
`npm run typecheck` to sanity-check types without running.

## Make it *this* client's bot
1. **Persona** → edit `src/persona.ts` (voice, scope, hard rules). Biggest lever.
2. **Knowledge** → drop `.md` files in `kb/`.
3. **Server structure** → set `GUILD_ID`, `ONE_TO_ONE_CATEGORY_IDS`, `COACHES_ROLE_ID`, `ADMIN_ROLE_ID` in `.env`.
4. **New capabilities** → add a tool in `src/tools/`, register it in `src/tools/index.ts`. That's it.

## Discord Developer Portal (once)
Enable **Message Content Intent** + **Server Members Intent** (Bot page), and invite with
`bot` scope + View Channels / Send Messages / Read History / Send-in-Threads / Create Public
Threads / Embed Links / Attach Files / Add Reactions (+ Manage Channels **only** because
`create_forum` makes channels). **Never grant Administrator.**

## Deploy
It's a plain Node process — run it under `pm2`/systemd on a VPS, or containerize it. It is
not tied to any one host. (Model history is in-memory; wire `src/memory.ts` to a DB before
you run multiple instances.)
