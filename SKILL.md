---
name: holeesheet-discord-video
description: Build a HoleeSheet short-form Discord-chat video. Use when asked to render/build a HoleeSheet episode, a Discord chat video, or "the server".
od:
  mode: video
  scenario: video
  design_system: discord
  inputs:
    - name: episode_id
      type: string
      required: false
---

# HoleeSheet — Discord chat video

You are generating a **vertical 9:16 short-form video of an animated Discord chat** for the
HoleeSheet content engine. The data is a **render_package_v1**. Everything you need is in the
HoleeSheet project — prefer its tools over re-inventing.

> **Project root (run all commands from here):**
> `C:\Users\adowner.FSLADS\Desktop\Projects\New folder\opendesign\HooLeeSheet\HoleeSheet_Project`

## ✍️ READ FIRST — story craft (this is what makes it good, not generic)
Open **[bible/production-bible.json](../../../bible/production-bible.json)** (or call the `get_story_bible`
MCP tool) and follow it. It's distilled from many real iterations — these lessons were re-learned every
episode, so don't re-learn them, apply them. The non-negotiables:
- **Open with chaos in the first 5–10s** — a mystery/argument/confession/reveal. Never open on setup.
- **The premise is the excuse; the interactions are the content.** Don't narrate "a confession channel" — show the chaos of characters trapped in it.
- **Write real jokes, not emoji.** Dry, deadpan, absurdist escalation with straight-man reactions. Emoji punctuate; they aren't the joke.
- **Messy, not orderly** — overlaps, interruptions, reactions before a message finishes, edited/deleted/corrected messages, multiple threads. No tidy turn-taking.
- **Vary everything** — never recycle the last premise/POV/format (see the bible's premise library). Respect canon unless it's a deliberate AU.
- **Give each character ONE comedic engine** and run it (see `character_dynamics`; e.g. Naruto = chaos without trying, Sasuke = too-serious, Gaara = dry).
- **End scenes on reactions**, plant callbacks that pay off minutes later, layer emotional beats between jokes.
- **Watch-only**: every "interactive" beat plays out as dialogue/action — no buttons.
- **Interpret, don't read the interface.** The chat is the medium; the *story is what happens between the messages*. The pipeline auto-narrates a literal fallback ("Karen deleted a message"), but YOU make it a story: add a `narration` field to any event to override it with interpretation (`{user:"Karen",type:"delete",narration:"Karen quietly takes back the thing she can't unsay."}`), and drop standalone `{type:"narration",text:"…"}` beats between messages to carry mood, momentum, and who's arriving. You supply the meaning — don't make the renderer guess.
- **Self-critique like a director** against the bible's checklist, fix it, *then* render. Use `review_episode` (or read the `listener.txt`/`review.json` a render writes) to hear what a screen-off viewer hears and catch dead air, cut-off voices, or events that pass silently.

## ⚡ THE WORKFLOW — run the CLI, do NOT hand-roll HTML
The fastest correct path to a real MP4 is **one command**. Write the story as JSON, then run
`tools/make.mjs`. It compiles your script into a valid render_package, renders the video
(HyperFrames), and adds voices + SFX + music — and prints the MP4 path.

> ⚠️ **Why this matters (40+ wasted iterations):** hand-writing a big Discord HTML composition is the
> #1 failure mode — the file gets **truncated** mid-code, the JS breaks, and the "Start" button does
> nothing. It also drags in a WebAudio player whose **BGM ducking, settings drawer, scrub bar, and TTS
> queue** get re-broken every single time (hard pump, voice 4-5 min behind text, settings stuck open).
> The pipeline kills all of that: a small JSON script can't truncate, voices are pre-rendered and
> time-aligned, music sidechain-ducks under speech in the mix, and a rendered MP4 needs **no player**.
> Don't hand-roll HTML. Write the JSON, run `make.mjs`.

```bash
cd "C:\Users\adowner.FSLADS\Desktop\Projects\New folder\opendesign\HooLeeSheet\HoleeSheet_Project"

cat > story.json <<'JSON'
{ "title": "Konoha Daily Insanity",
  "characters": [ {"name":"Naruto","color":"#ff8c00","voice":"Harry"},
                  {"name":"Sasuke","color":"#5b7fc7"} ],
  "script": [
    {"type":"act","title":"Episode 1","subtitle":"The Missing Ramen"},
    {"user":"Naruto","text":"OK TEAM MISSION TIME"},
    {"user":"Sasuke","type":"typing","delay":1000},
    {"user":"Sasuke","text":"not me","reply_to":"Naruto"},
    {"user":"Sakura","type":"reaction","emoji":"💀","react_to":"Naruto"},
    {"user":"System","type":"system","text":"Naruto created a new server"}
  ] }
JSON

node tools/make.mjs story.json            # -> prints renders\<id>.mp4
```

**Do NOT write your own `requestAnimationFrame` Discord composition.** Without a paused GSAP
timeline on `window.__timelines["main"]` it renders as a **frozen frame 0** (a broken video).
`tools/make.mjs` handles all of that for you.

### Flags
- `-o out.mp4` — choose the output path.
- `--no-voice` — skip ElevenLabs voices, **keep SFX + music**.
- `--no-audio` — fully silent (no voice, no SFX, no music).
- `--no-sfx` / `--no-music` — drop just that layer.
- `--fps 30` — frame rate (default 30).
- `-` as the input reads the story JSON from stdin.

## The story format (what you write)
```jsonc
{ "title": "Team 7 Group Chat",
  "server":  { "name": "Konoha" },                  // optional
  "characters": [
    { "name":"Naruto", "color":"#ff8c00", "voice":"Harry" },   // voice by NAME (see list below)
    { "name":"Sasuke", "avatar":"👁️" },                         // emoji avatar
    { "name":"Sakura", "avatar":"https://image.pollinations.ai/prompt/sakura%20haruno%20anime%20portrait" } // real image avatar
  ],
  "extensions": { "css": ".text{font-style:italic}" },          // raw CSS for anything not modeled
  "script": [
    {"type":"act","title":"Episode 1","subtitle":"The Missing Ramen"}, // chapter card
    {"user":"Naruto","text":"who took my ramen"},
    {"user":"Sasuke","text":"not me","reply_to":"Naruto"},             // reply by name / "last"
    {"user":"Sakura","type":"reaction","emoji":"💀","react_to":"Naruto"}, // reaction targets a message
    {"user":"Naruto","type":"voice_note","seconds":4,"text":"I KNEW IT"}, // spoken voice note
    {"user":"Naruto","type":"image","image":"https://image.pollinations.ai/prompt/empty%20ramen%20bowl","text":"evidence"}
  ] }
```
- `type` ∈ `message` | `typing` | `reaction` | `ban` | `system` | `join` | `leave` | `act` | `voice_note` | `image`. Default `message`.
- `delay` = ms before the event. `effect` = a meme effect (e.g. `"vine_boom"`). Timestamps are computed for you — never derive the clock yourself.
- `react_to` / `reply_to` resolve by **character name** or `"last"` — no event ids.
- Per-line voice: put `"voice":"<name>"` on a script entry to override that line.

**Voices:** list the available cast with `node tools/elevenlabs.mjs` (or see `list_voices` via MCP),
then set `voice:"<name>"` on a character. Names resolve to ids automatically. Voices need
`ELEVENLABS_API_KEY` in `.env`; without it the render still produces video + SFX + music.

**Real images:** a character `avatar` or a `{type:"image","image":"<url>"}` line can be any image
URL — the renderer fetches it at render time. A Pollinations URL *is* the image
(`https://image.pollinations.ai/prompt/<description>`). Prefer real generated art over emoji.

## Alternative: the MCP tools (for MCP-capable agents only)
If you are an agent that can call MCP tools (e.g. Claude Code), the same pipeline is exposed as the
**holeesheet** MCP server: `make_episode` (one-shot), `compile_script`, `render_episode` +
`render_status`, `list_voices`, `submit_feedback`. Same input shape as the story format above.
⚠ Codex/Kimi cannot use MCP in Open Design — use the CLI above instead.

## Other tools in the project
- `node tools/make.mjs <story.json>` → story → MP4 (the main path, above).
- `node tools/build-render-package.mjs <episode_id>` → print a render_package_v1 from Supabase.
- `node tools/build-composition.mjs --from-db <episode_id>` → just the HTML composition (no render).
- `node tools/validate-package.mjs <pkg.json>` → check a package is well-formed.
- Credentials/data come from `.env` (Supabase, ElevenLabs). See `AGENTS.md` for the full map.

## render_package_v1 shape (what `make.mjs` produces and renders)
`{ meta, episode{title,platform,duration_seconds}, server{name,theme}, characters[{display_name,
handle,role_color,is_bot}], channels[{name}], scenes[], timeline[{start_time,duration,event_type,
character_id,content,attention_event,effects,posted_at}], assets[], effects[], audio,
rendering{aspect_ratio,resolution:"1080x1920",fps:30} }`. Each timeline event carries a
pre-computed monotonic **`posted_at`** ("Today at 4:36 PM") — the chat clock renders from that, never
from `start_time` (animation seconds).

## Output
`tools/make.mjs` writes the MP4 to `renders\<id>.mp4` (1080x1920, 30fps, h264) and prints the path.
Hand that path back, or upload it to the Supabase `renders` bucket.
