# SOP — Building a Discord AI Community Bot (Custom Build)

**Stack:** `discord.js` (the bot) + the **Vercel AI SDK** (`ai`) for the model + tool-calling, running on **Kimi** (Moonshot) — swappable to Claude/GPT in one line. This is the "own the codebase" approach: reliable tool-calling, deploys anywhere, no third-party agent harness. This repo (`larry-bot/`) is the reference implementation — every file referenced below is real and runnable.

> **Mental model:** the bot is three layers. **discord.js is the body** (connects to Discord, hears messages, replies). **The Vercel AI SDK is the nervous system** (calls the model, runs the tool loop). **The model (Kimi) is the brain** (produces the answer). You build the body + nervous system once; you swap the brain in one line.

---

## 0. Prerequisites
- **Node 20+** and npm.
- **Admin on the target Discord server** (you need *Manage Server* to invite the bot).
- **A model API key** — Kimi/Moonshot (https://platform.moonshot.ai). Any OpenAI-compatible or Anthropic key also works.
- **Your content** — docs/FAQs/notes the bot should answer from (becomes the knowledge base).
- **The bot's identity** — name, purpose, and voice (becomes the persona).

---

## PHASE 1 — Create the bot in the Discord Developer Portal (once)
1. https://discord.com/developers/applications → **New Application** → name it.
2. **Bot** tab → set username + avatar → **Reset Token** → copy it **straight into `.env`** (never a message, never git).
3. **⚠️ Privileged Gateway Intents** (the #1 gotcha) → turn **ON**:
   - **Message Content Intent** — without it the bot can't read message text and will sit silent.
   - **Server Members Intent** — needed to resolve members and gate privileged actions.
4. **OAuth2 → URL Generator** → scope **`bot`** → permissions: View Channels, Send Messages, Read Message History, Send Messages in Threads, Create Public Threads, Embed Links, Attach Files, Add Reactions **+ Manage Channels** (only if the bot creates channels, as the `create_forum` tool does). **Never grant Administrator.**
5. Open the generated URL → pick the server → **Authorize**.

---

## PHASE 2 — Scaffold the project
```
larry-bot/
├─ package.json         # deps: ai, @ai-sdk/openai, discord.js, zod, dotenv, tsx
├─ .env.example         # token, guild, model key, channel/role IDs  (copy → .env)
├─ kb/                  # your knowledge base (.md files)
└─ src/
   ├─ index.ts          # discord.js client: gating, @mention → agent → reply
   ├─ agent.ts          # THE loop: generateText(system + history + tools, maxSteps)
   ├─ model.ts          # the model, via createOpenAI({ baseURL })
   ├─ persona.ts        # system prompt = who the bot IS (biggest lever)
   ├─ memory.ts         # per-channel history (swap for a DB in prod)
   ├─ gating.ts         # requireMention + role-based auth (in code, not prompt)
   ├─ discord-rest.ts   # raw Discord REST helper (channel/thread creation)
   ├─ forum-utils.ts    # shared helpers (find a forum, slugify, timestamps)
   └─ tools/            # one file per capability + a registry (index.ts)
```
```bash
npm install
cp .env.example .env    # fill everything in
```

---

## PHASE 3 — Wire the model (the brain), via the SDK
`src/model.ts` — Kimi is OpenAI-compatible, so use the SDK's OpenAI provider with a custom baseURL:
```ts
const provider = createOpenAI({ baseURL: config.moonshotBaseUrl, apiKey: config.moonshotKey, compatibility: "compatible" });
export const model = provider(config.modelId); // e.g. "kimi-k2.6"
```
**Swap to Claude/GPT later = change these two lines** (import a different provider, return its model). Nothing else in the codebase changes.

---

## PHASE 4 — The agent loop (the part a harness hides — here you own it)
`src/agent.ts`:
```ts
const { text } = await generateText({
  model,
  system: SYSTEM_PROMPT,                 // persona.ts
  messages: [...history, userMsg],       // memory.ts
  tools: buildTools(ctx),                // tools/index.ts (built per-request, for auth)
  maxSteps: 6,                           // think → call tool → read result → answer
});
```
`maxSteps` is the trick: the SDK runs the tool-call loop for you. When the model asks for a tool, the SDK executes it, feeds the result back, and loops until the model gives a final answer. Because **you own this loop**, the "model won't call the tool" problem you get from prompt-only bots mostly disappears — the tool is right in the schema, and you can force or validate it.

---

## PHASE 5 — Tools = what the bot can DO (add one = one file)
A tool is a name + a Zod schema + an `execute` function.

**Read-only example — `kb_search`** (answer from your material, never generic):
```ts
export const kbSearchTool = tool({
  description: "Search the knowledge base. ALWAYS use before answering; answer only from results.",
  parameters: z.object({ query: z.string() }),
  execute: async ({ query }) => ({ results: /* grep/score kb/*.md */ }),
});
```

**Action example — `create_forum`** (make a private forum). Key points:
- **Auth is enforced in code** (`isAuthorizedCoach(ctx)`) — a normal member can ask all day and it refuses, because the gate checks their **roles**, not the prompt.
- **Dedup** before creating so it never makes duplicates.
- Creates a private channel (deny `@everyone` view; allow the right roles + the person), seeds standard posts.

**To add a capability:** drop a file in `src/tools/`, register it in `src/tools/index.ts`. No harness, no allowlist, no restart ritual.

### Tools shipped in this reference build
| Tool | Who can use | What it does |
|---|---|---|
| `kb_search` | anyone | Answer from the knowledge base only |
| `community_pulse` | anyone | Recent activity in the public channels |
| `request_invoice` | anyone | Member request → pings the admin/finance channel |
| `create_forum` | admins/coaches | Create a member's private forum + standard posts |
| `add_member_to_forum` | admins/coaches | Grant someone access to a forum |
| `remove_member_from_forum` | admins/coaches | Revoke access |
| `find_student_forum` | admins/coaches | Locate a member's forum |
| `find_inactive_members` | admins/coaches | "Going quiet" report |
| `post_announcement` | admins/coaches | Post to a channel |

Every privileged tool enforces auth **in code** (`isAuthorizedCoach`), so the guard is never just the prompt.

> **Per-request context:** tools that need "who's asking" are built by a factory `buildTools(ctx)` that closes over the message author's id + roles. That's how the privileged tools know whether the requester is authorized.

---

## PHASE 6 — Persona + Knowledge = making it *your* bot
- **`src/persona.ts`** — the system prompt: name, purpose, **voice**, and hard rules ("only answer from the KB", "never leak private channels", "if you have a tool for it, call it — don't say you can't"). Edit this the most.
- **`kb/`** — drop your `.md` files (method, FAQs, notes). `kb_search` reads them. Start with keyword search; upgrade to embeddings later without touching anything else.

---

## PHASE 7 — Gating + guardrails
- **`REQUIRE_MENTION=true`** — only reply when @mentioned (`src/index.ts`).
- **Role auth** — privileged tools gated in `gating.ts` against real roles.
- **Server scope** — ignores any guild that isn't `GUILD_ID`.
- **Prompt guardrails** — the "hard rules" block in `persona.ts`.

---

## PHASE 8 — Run + test
```bash
npm run dev            # hot reload  (npm start for prod, npm run typecheck to check types)
```
In a test channel: `@Bot hello` → it replies. Then test each path:
1. **KB answer** — ask something in the KB → cites the material; ask something *not* in it → it says "I don't have that" (no hallucinating).
2. **Privileged action** — as an authorized user, trigger `create_forum` → the forum appears with standard posts.
3. **Auth** — as a normal member, ask the same → it **refuses**.

---

## PHASE 9 — Deploy
It's a plain Node process — no special host:
- **VPS:** `pm2 start` or a systemd unit. One process per bot.
- **Container/serverless:** works too.
- Before running multiple instances, point `src/memory.ts` at Postgres/Redis so history survives restarts.

---

## Gotchas cheat-sheet
| Symptom | Cause | Fix |
|---|---|---|
| Bot online but silent | Message Content Intent off | Dev Portal → Bot → enable it |
| `Used disallowed intents` on boot | Server Members Intent off | Enable it |
| Tool never fires | Model didn't call it | Sharpen the tool `description`; raise `maxSteps`; tell it in the persona to use the tool |
| Answers from generic knowledge | Skipped `kb_search` | Persona rule: "ALWAYS kb_search first, answer only from results" |
| Normal member ran a privileged tool | Auth in prompt only | Gate in **code** (`isAuthorizedCoach`) — this build does |
| Reply cut off | Discord 2000-char cap | Chunk the reply (index.ts already does) |
| Token leaked | Pasted in chat / committed | Reset in Dev Portal; keep it in `.env` only |

---

## Why build it (vs. a config-only agent harness)
- **Reliable tool-calling** — you own the loop and gate actions in code, so "the model won't run the tool" goes away.
- **Direct model calls** — swap models in one line; nothing to wedge.
- **Portable + ownable** — plain TypeScript you can deploy anywhere and hand to anyone. A real product.
- **Cost:** you build + maintain the app. The trade is control and reliability — which is what a sellable product needs.
