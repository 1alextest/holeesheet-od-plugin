---
name: holeesheet-discord-video
description: Build a HoleeSheet short-form Discord-chat video from a render_package. Use when asked to render/build a HoleeSheet episode, a Discord chat video, or "the server".
od:
  mode: video
  scenario: video
  design_system: discord
  preview: ../../../render/discord-template.html
  inputs:
    - name: episode_id
      type: string
      required: false
---

# HoleeSheet — Discord chat video

You are generating a **vertical 9:16 short-form video of an animated Discord chat** for the
HoleeSheet content engine. The data is a **render_package_v1** (the single source of truth).
Everything you need is in this project (`HoleeSheet_Project/`). Prefer the project's own tools
over re-inventing.

## ⚡ THE WORKFLOW — use the HoleeSheet MCP, do NOT hand-roll HTML
The fastest correct path to a real MP4 is three MCP tool calls. **Do not write your own
`requestAnimationFrame` Discord composition** — without a paused GSAP timeline on
`window.__timelines["main"]` it renders as a **frozen frame 0** (a broken video). If you must
hand-author, run **`lint_composition`** on it first; it will tell you if it's un-renderable.

1. **`compile_script`** — write the story in the simple format and we return a valid, renderable
   `render_package_v1` (timestamps, timing, structure all correct):
   ```
   compile_script({
     title: "Konoha Daily Insanity",
     characters: [{name:"Naruto", color:"#ff8c00"}, {name:"Sasuke", color:"#5b7fc7"}],
     script: [
       {user:"Naruto", text:"OK TEAM MISSION TIME"},
       {user:"Sasuke", type:"typing", delay:1000},
       {user:"Sasuke", text:"fine"},
       {user:"Sakura", type:"reaction", emoji:"💀"},        // targets the previous message
       {user:"System", type:"system", text:"Naruto created a new server"}
     ],
     feedback: "what was hard / what capability you wished existed"   // REQUIRED — we read this
   })
   ```
   `type` ∈ message | typing | reaction | ban | system | join | leave. `delay` = ms before the
   event. `effect` = a meme effect name (e.g. "vine_boom"). Timestamps are computed for you — never
   derive the clock yourself.
2. **`render_episode({ package })`** — pass the package from step 1; returns a `job_id`.
3. **`render_status({ job_id })`** — poll until `completed`, then use the MP4 path.

**`submit_feedback`** anytime: friction, missing event types/effects/character features — anything
that would help you write better stories. We use it to improve the tools.

## Easiest path: write a simple script, let `compile_script` do the rest
Don't hand-build a render_package or a raw HTML composition (a hand-rolled rAF page renders as a
frozen frame 0 — `lint_composition` will tell you). Instead author the natural way and compile:

```json
{ "title":"Team 7 Group Chat",
  "characters":[ {"name":"Naruto","avatar":"🦊","color":"#ff8c00"}, {"name":"Sasuke","avatar":"👁️"} ],
  "extensions": { "css": ".text{font-style:italic}" },
  "script":[
    {"type":"act","title":"Episode 1","subtitle":"The Missing Ramen"},
    {"user":"Naruto","text":"who took my ramen"},
    {"user":"Sasuke","text":"not me","reply_to":"Naruto"},
    {"user":"Sakura","type":"reaction","emoji":"💀","react_to":"Naruto"},
    {"user":"Naruto","type":"voice_note","seconds":4,"text":"I KNEW IT"} ] }
```
`compile_script` → a valid, renderable render_package. Then `render_episode {package}` → MP4.
**Flexibility (use it):** `react_to`/`reply_to` by **character name** or `"last"` (no event ids);
emoji `avatar` per character; `type:"act"` chapter cards; `type:"voice_note"`; and **`extensions.css`**
for anything we don't model yet. If something's still too rigid, `submit_feedback` — that's how the
contract grows.

**Voices:** call **`list_voices`** to see the cast you can use, then set `voice: "<name>"` on a
character (e.g. `{"name":"Naruto","voice":"Harry"}`) or override a single line with `voice` on that
script entry. Names resolve automatically (e.g. "Charlie" → its id).

**Real images:** a character's `avatar` can be an image URL (rendered as a real round avatar), and a
script line `{type:"image", image:"<url>", text?}` shows an image attachment. The renderer fetches
the URL at render time. You can generate images on the fly — e.g. a Pollinations URL is itself the
image: `avatar:"https://image.pollinations.ai/prompt/naruto%20uzumaki%20anime%20portrait"`. Prefer
real generated character art over emoji when you can.

## Tools available in this project (run them)
- `node tools/build-render-package.mjs <episode_id>` → prints a render_package_v1 from Supabase.
  (Default episode: `70fb8b16-018d-4439-b6f0-cea9e91adb99`, "Kanye Discord".)
- `node tools/build-composition.mjs --from-db <episode_id>` → writes a standalone animated HTML
  composition to `out/<id>.html`. Or `node tools/build-composition.mjs` to use the fixture.
- `node tools/validate-package.mjs <pkg.json>` → check a package is well-formed.
- Credentials/data come from `.env` (Supabase). See `AGENTS.md` for the full map.

## render_package_v1 shape (what you're rendering)
`{ meta, episode{title,platform,duration_seconds}, server{name,theme}, characters[{display_name,
handle,role_color,is_bot}], channels[{name}], scenes[], timeline[{start_time,duration,event_type,
character_id,content,attention_event,effects,posted_at}], assets[], effects[], rendering{aspect_ratio,
resolution:"1080x1920",fps:30} }`. Timeline `event_type` ∈ message | typing | reaction | ban |
system | join | leave. Reactions point at `content.target_event_id`.

⚠ **Timestamps:** render the chat clock from each event's **`posted_at`** string (e.g. "Today at 4:36 PM")
— it's pre-computed monotonic. Do NOT derive the time from `start_time` (that's animation seconds;
mapping it to a clock wraps past 12h and produces out-of-order timestamps).

## Visual contract — make it look like real Discord (use the `discord` design system)
Pull tokens from the `discord` design-system. Non-negotiables:
- Dark 3-step depth: server rail `#1e1f22`, channel column `#2b2d31`, chat surface `#313338`.
- Blurple `#5865f2` ONLY for accents/mentions/"you". gg-sans. 16px+ body (scaled up for 1080w).
- Left **server rail** with rounded-square server icons (active = white pill). A channel header
  (`# general`, server name, member hint). Tight message rows, grouped by consecutive author.
- Avatars = solid role-colored circles w/ white initial + status dot. `@mentions` render as
  blurple pills. Reactions = Discord reaction pills. ban/system = centered muted/red rows.
- No dead space: open on a "Welcome to #general" marker, let the chat fill from the bottom.

## Animation — FRAME-DRIVEN (critical for rendering)
Do NOT animate with bare `requestAnimationFrame` real-time playback — it does not run in a
headless render context. Expose a pure `render(t)` (or use HyperFrames/Remotion frame model):
each element's enter progress = clamp((t - event.start_time)/0.3, 0, 1). The renderer sets the
clock per frame. `attention_event` / `vine_boom` effects = brief shake/pulse.

## Output
1. Build/refresh the composition for the requested `episode_id` (default Kanye).
2. Render it to **MP4 (1080x1920, 30fps, h264)** using the **video-hyperframes** skill.
3. Upload to the Supabase `renders` bucket (or leave the MP4 path for the worker).

Start `render/discord-template.html` is a working baseline — improve its design, keep it
frame-driven, and feed it the render_package.
