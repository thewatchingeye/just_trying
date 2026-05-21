# JustTrying — Claude Code Context

Browser-based AI-guided musical performance app by James Saunders (Bath Spa University),
for collaborator Zubin Kanga.

## What the app is

A single-file HTML app (`index.html`, ~2960 lines, no build step, no dependencies).
It runs structured musical-performance exercises: a timed "score" fires category cues
(spoken words/notes/chords), plays background music, runs an AI voice guide, and listens
to the performer via microphone for voice commands and conversation.

### Server deployment structure (`public_html/justtrying/`)
- `index.html` — the app (this file, renamed from `justtrying.html`)
- `manifest.json` — lists categories, scores, music, gpt_texts, defaults
- `categories/` — CSV files: object, pitch, sound, action, extract, chord_major, chord_minor
- `scores/` — txt score files
- `music/focus|relaxing|motivational|field|hold/` — mp3/wav files
- `gpt_texts/` — gpt_text.csv

## Core architecture

All code lives inside `index.html` in a single `<script>` block.

- `JustTryingApp` class holds everything. `app` is the global instance.
- `state` object holds all runtime state.
- Score parser reads timed events, `gpt say "literal"`, `gpt say file.csv`,
  `GPT: instruction`, category start/stop, music commands.
- Audio: OpenAI TTS (preferred) routed through Web Audio (ConvolverNode reverb,
  StereoPannerNode); browser speechSynthesis as fallback.
- AI voice queue: priority-based (0=acks, 1=conversation, 2=score GPT, 3=global).
- Voice recognition: mic → VAD (volume threshold) → MediaRecorder → OpenAI Whisper API.
- Voice Level Calibration: two-step (measure speaker bleed, measure player voice),
  sets a volume threshold so only the player's voice triggers recording.

## Current build

**Build v14** — bulletproof extract normalization (line 227 of index.html)

Bump the build banner string when shipping a change so the user can confirm cache-busting.

## Key functions

| Function | Purpose |
|---|---|
| `init()` | Startup, loads manifest, applies defaults |
| `loadServerContent()` | Fetches manifest + CSVs + scores; runs CSV repair |
| `parseCSV()`, `repairExtractItems()`, `repairChordItems()`, `stripExtractPrefix()` | CSV loading/repair |
| `parseScore()` / `parseCueInstruction()` | Score parsing |
| `fireCue(category)` | Picks a cue item, transforms text for TTS, calls `speak()` |
| `speak()` / `speakWithTTS()` / `speakGPTBrowser()` | Audio output |
| `speakGPT()` / `processAISpeechQueue()` | Priority AI voice queue |
| `runGPTInstruction()` / `handleConversationalInput()` | OpenAI Chat calls |
| `startVoiceActivityDetection()` | Mic VAD loop; uses `calibratedThreshold` |
| `transcribeWithWhisper()` | Sends recorded audio to Whisper, filters result |
| `calibrateVoice()` | Two-step volume calibration |
| `getTTSAudio()` | OpenAI TTS fetch + cache (keyed `voice_text`) |
| `convertIPA()` | IPA-to-phonetic conversion |
| `numberToWords()` | Digit → word for extract cues |

## Open issues

1. **Category cue skipping with OpenAI TTS.** When cues fire faster than a voice can
   speak (e.g. extract every 1-2s), the new cue is dropped — log shows
   `Skipping (<voice> still speaking)`. Browser TTS queues instead.
   Fix: add a small per-voice queue (size 1, drop-oldest).
   Code: `speakWithTTS`, the `voiceAudioPlaying[voice]` check.

2. **Speaker-bleed into mic.** Calibration helps but isn't perfect with omnidirectional
   mics. User has a Shure MV7. Browser TTS bleeds more than OpenAI TTS because browser
   speechSynthesis bypasses the browser's echo cancellation (goes to OS audio engine).
   Recommendation: use OpenAI TTS for everything.

3. **Short-utterance TTS reliability.** `tts-1-hd` sometimes returns truncated audio for
   very short phrases. Mitigations in place: `[TTS WARN]` log if audio suspiciously short.
   Could add: retry-on-short, or pre-fetch all category items at session start.

4. **Pronunciation edge cases.** "a" → "Ay" at start of chord/pitch cues. Comma inserted
   before major/minor when there's an accidental ("E flat, minor"). IPA notation supported
   as manual override in CSVs.

## Conventions

- No build step. Edit `index.html` directly. All JS in one `<script>` block.
- Sanity-check JS after edits: `node -e "new Function(scriptBody)"`
- Build banner is at line ~227. Bump it on every deploy.
- `localStorage` keys prefixed `jt_`.
- TTS model: `tts-1-hd`. Whisper endpoint: OpenAI `/v1/audio/transcriptions`.
- Memory/state: `state` object on the `JustTryingApp` instance.

## Testing

- Hard refresh (Cmd+Shift+R) after deploy — build banner confirms version.
- Log panel is the main diagnostic surface. `[CSV <cat>]` lines show loaded items.
- For pronunciation tests, space cues 3-4s apart so they don't skip.
- The app needs an OpenAI API key (entered in the Settings tab and stored in localStorage).
