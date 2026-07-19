# Signal Vault — Master Prompt

This is the master spec for **Signal Vault**, a personal tool that turns raw
voice/audio recordings into a written essence and flags business opportunities
hidden inside everyday conversations. Use this document to regenerate,
extend, or hand this project to another AI coding assistant.

---

## 1. The problem this solves

The user records a large volume of real-life audio on their phone (meetings,
conversations, ideas spoken aloud) but never has time to re-listen to it.
Two things get lost as a result:

1. **The content itself** — decisions, follow-ups, context.
2. **Business signal** — pain points and opportunities that surface in casual
   conversation but are never captured or acted on.

The tool must close that gap without requiring the user to fully re-listen to
or manually transcribe every recording.

## 2. Hard technical constraint (do not violate)

This app runs as a **single self-contained HTML file in a sandboxed browser
artifact** with no custom backend. That means:

- There is **no real speech-to-text available**. The Anthropic Claude API used
  from the browser (`POST /v1/messages`) accepts **text, image, and PDF**
  input only — not audio.
- The Web Speech API (`SpeechRecognition` / `webkitSpeechRecognition`) only
  works against a **live microphone**, not an uploaded audio file, and is not
  reliably available inside a sandboxed iframe.
- Therefore: **do not attempt to auto-transcribe uploaded audio.** Do not fake
  it, do not silently degrade, and do not imply transcription is happening.
  Be explicit with the user about this constraint if asked.

The workaround that actually works, and is the core mechanic of this app:

- Let the user play the file at high speed (2–3x) in-browser.
- Let them drop short timestamped text fragments ("tags") the moment
  something matters, instead of writing full sentences.
- Use real signal-processing (via the Web Audio API, client-side, no
  network call) to **auto-suggest candidate timestamps** by detecting
  amplitude peaks in the actual decoded waveform — reducing how much of the
  file the user has to actively hunt through.
- Send only the *short text fragments* (not audio) to the Claude API, and ask
  it to reconstruct a plausible narrative essence, action items, and business
  opportunities from those fragments.

## 3. The three-layer output (core product requirement)

Every processed recording must produce exactly three layers:

1. **The Story** — a 2–4 sentence, engagingly written (not clinical) essence
   of what the recording was likely about, reconstructed from the fragments.
2. **Action Items** — concrete follow-ups, max 4.
3. **Business Signal** — an array of `{ title, pain_point, opportunity,
   confidence }` objects. `confidence` is `high | medium | low`. If there is
   genuinely no business angle, return an **empty array** — never force one.

Additionally return `tags`: up to 4 short lowercase category tags for search.

## 4. Required JSON contract for the Claude API call

The client sends a single `/v1/messages` call per recording with
`model: "claude-sonnet-4-6"`, `max_tokens: 1000`, and a single user message.
The prompt must instruct Claude to return **only** raw JSON, no markdown
fences, matching exactly:

```json
{
  "summary": "2-4 sentence story-like essence...",
  "action_items": ["...", "..."],
  "business_ideas": [
    {
      "title": "...",
      "pain_point": "...",
      "opportunity": "...",
      "confidence": "high | medium | low"
    }
  ],
  "tags": ["...", "..."]
}
```

The client must defensively strip a leading/trailing ` ```json ` fence before
`JSON.parse`, in case the model doesn't follow instructions exactly.

## 5. Functional requirements

### Capture
- Drag-and-drop or tap-to-browse file upload.
- Accept common formats explicitly: `audio/*, video/mp4, audio/mp4,
  audio/x-m4a, audio/mpeg, .mp3, .mp4, .m4a, .wav, .aac, .ogg, .webm, .mov`.
- Play/pause, ±10s skip, seek-by-click-on-waveform.
- Speed control: 1x / 1.5x / 2x / 3x, default 2x.
- **Duration must be read reliably.** Some MP3/MP4 files report
  `Infinity` for `audio.duration` until the browser is forced to seek through
  the file once. Implement the standard workaround: on `loadedmetadata` /
  `durationchange`, if `duration` isn't a finite positive number, set
  `currentTime = 1e101`, wait for the resulting `timeupdate` event, reset
  `currentTime = 0`, and re-read `duration`. Also fall back to the duration
  reported by `AudioContext.decodeAudioData()` if the `<audio>` element still
  can't resolve it.
- **Real waveform, not decorative/random bars.** Use
  `AudioContext.decodeAudioData` on the file's `ArrayBuffer`, downsample the
  first channel into ~70 buckets of average absolute amplitude, normalize,
  and render as bars. This is also the input to moment-detection below.
- **"Suggest moments to tag"**: run simple peak detection over the waveform
  buckets (local maxima above ~1.08x the mean, spaced at least
  `duration / (count * 1.4)` apart) and pre-populate ~7 tag markers on the
  waveform at those timestamps with empty, editable text, so the user only
  has to fill in a word or two per suggested moment instead of finding
  moments themselves.
- Manual tag capture: text input + "+Tag" button that stamps the current
  playback timestamp. Tags render as chips below the waveform and as dots on
  the waveform itself; tag text is inline-editable (`contenteditable`).
- Optional free-text "notes" field as a fallback if the user doesn't want to
  tag at all.
- "Extract the essence" button — disabled/spinner while the API call is in
  flight; requires at least one filled tag or a note before allowing submit.
- Clear inline error messaging on failure that reassures the user their tags
  aren't lost (no destructive resets on error).

### Vault
- Every processed recording is saved **permanently** via the artifact
  `window.storage` API (`shared: false` — personal data only), keyed under a
  single `'recordings'` key holding a JSON array (avoid many small keys).
- Searchable by summary text, tags, and filename.
- List view shows date, duration, a business-idea-count badge, a 2-line
  summary preview, and category tags.
- Tapping a card opens a detail sheet with the full story, action items,
  business signal cards (color-coded by confidence), tags, and a delete
  action.

### Idea Bank
- Aggregates every `business_ideas` entry across every recording into one
  reverse-chronological feed, each card linking back to its source recording.
- This is the compounding value of the tool — patterns across weeks/months
  of recordings should be visible in one place.

### Data policy
- **Audio files themselves are never persisted** — only the derived text
  (summary, action items, business ideas, tags, and the raw tag fragments)
  is saved. This should be communicated to the user: once a recording is
  processed, it's safe to delete the original file from their device.

## 6. Design direction

- Personal, moody, "private notebook" feel — not corporate SaaS.
- Palette: warm sunset gradient, pink + orange as the primary identity
  (avoid generic dark-mode teal/cyan or the common AI-generated cream/terracotta
  or pure-black/acid-green defaults).
  - Background: deep aubergine-black with warm undertone (`#170D14`)
  - Surfaces: `#241621`, `#331D2C`
  - Text: warm off-white `#FDEDE7`; muted: `#C09AA5`
  - Orange: `#FF7A45` — primary actions, record/process CTA
  - Pink: `#FF3E7F` — secondary accent, medium-confidence signal
  - Peach: `#FFB877` — tertiary accent, high-confidence signal, tags
  - Signature gradient: `linear-gradient(135deg, orange, pink)` used on the
    brand mark, primary buttons, waveform "played" fill, and section
    eyebrows.
- Typography:
  - Display/narrative: **Fraunces** (serif, italic for storytelling moments)
  - Body/UI: **Sora**
  - Data/mono (timestamps, tags, labels): **IBM Plex Mono**
  - Generous type scale — section titles ~32px, story text ~19–21px italic
    serif, not cramped default sizing.
- Mobile-first, single-column, bottom tab bar (Capture / Vault / Idea Bank),
  max-width ~520px centered layout, safe-area padding for iOS.
- Respect `prefers-reduced-motion`.

## 7. Non-goals / explicit exclusions

- No login system, no multi-user accounts (personal `window.storage` only).
- No audio storage/hosting.
- No claim of real transcription — never say "transcript" in the product
  copy in a way that implies word-for-word accuracy.
- No forced business idea when none genuinely exists in the material.

## 8. Tech stack

- Single self-contained `.html` file — inline `<style>` and `<script>`, no
  build step, no external JS framework.
- Web Audio API for decoding + waveform + peak detection.
- `window.storage` (artifact persistence API) for the Vault / Idea Bank data.
- `fetch` directly to `https://api.anthropic.com/v1/messages` with
  `model: "claude-sonnet-4-6"`, `max_tokens: 1000`, no API key handling
  (handled by the host environment).



  # Signal Vault

A personal tool for turning raw audio recordings into a written essence,
action items, and flagged business opportunities — without needing full
transcription.

## Repo structure

```
signal-vault/
├── README.md
├── prompts/
│   └── master-prompt.md     ← the full spec/prompt this app was built from
└── src/
    └── signal-vault.html    ← the app itself (single self-contained file)
```

## Using this repo

- **`prompts/master-prompt.md`** is the living spec. Update it whenever the
  product direction changes, so the prompt and the code never drift apart.
  It's written so it can be pasted into a fresh chat with any AI coding
  assistant to regenerate or extend the app from scratch.
- **`src/signal-vault.html`** is the current, working build. Open it directly
  in a browser — no build step, no dependencies to install.

## Suggested workflow going forward

1. Make a change → update `master-prompt.md` first if it's a product-level
   change (new feature, changed constraint, changed design direction).
2. Update `src/signal-vault.html` to match.
3. Commit both together so the prompt and the code stay in sync as history.
