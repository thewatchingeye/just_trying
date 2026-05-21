# JustTrying — Project Handoff for Claude Code

A browser-based AI-guided musical performance app by James Saunders (Bath Spa University),
for collaborator Zubin Kanga. Single-file HTML app deployed to a cPanel server.

## How to continue in Claude Code (VS Code)

1. Install the Claude Code extension in VS Code (Extensions panel → search "Claude Code").
2. Create a project folder locally, e.g. `justtrying/`.
3. Place `justtrying.html` in it (download the latest from the chat).
4. Open the folder in VS Code, open the Claude Code panel.
5. Paste this whole document as your first message so Claude has full context.

## What the app is

A single-file HTML app (`justtrying.html`, ~2960 lines, no build step, no dependencies).
It runs structured musical-performance exercises: a timed "score" fires category cues
(spoken words/notes/chords), plays background music, runs an AI voice guide, and listens
to the performer via the microphone for voice commands and conversation.

Deployed structure on the server (`public_html/justtrying/`):
- `index.html` (this is `justtrying.html` renamed)
- `manifest.json` — lists categories, scores, music, gpt_texts, defaults
- `categories/` — CSV files: object, pitch, sound, action, extract, chord_major, chord_minor
- `scores/` — txt score files
- `music/focus|relaxing|motivational|field|hold/` — mp3/wav files
- `gpt_texts/` — gpt_text.csv

## Core architecture (all inside the one HTML file)

- `JustTryingApp` class holds everything. `app` is the global instance.
- `state` object holds all runtime state.
- Score parser: reads timed events, `gpt say "literal"`, `gpt say file.csv`,
  `GPT: instruction`, category start/stop, music commands.
- Audio: OpenAI TTS (preferred) routed through Web Audio (ConvolverNode reverb,
  StereoPannerNode); browser speechSynthesis as fallback.
- AI voice queue: priority-based (0=acks, 1=conversation, 2=score GPT, 3=global).
- Voice recognition: mic → VAD (volume threshold) → MediaRecorder → OpenAI Whisper API.
- Voice Level Calibration: two-step (measure speaker bleed, measure player voice),
  sets a volume threshold so only the player's voice triggers recording.

## Current known-good state (Build v14)

Working well:
- Score parsing, category cues, music, pitch sequence generator with spoken intros.
- OpenAI TTS with reverb + per-category panning.
- Voice Level Calibration (mic auto-enables; AI voice guides the steps; AGC disabled
  for stable readings).
- Whisper voice recognition with noise-word + hallucination filtering.
- Auto-enable mic on Play.
- CSV repair: `repairExtractItems` / `repairChordItems` normalize malformed CSVs.
- `stripExtractPrefix` collapses any doubled "extract" prefixes.
- IPA support: `/eɪ/ flat major` → "ay flat major".
- Diagnostic logging: `[CSV <cat>]` lines show loaded items; cues log
  `source → spoken: "..."`.

## Open issues to tackle next

1. **Category cue skipping with OpenAI TTS.** When cues fire faster than a voice can
   speak (e.g. extract every 1-2s), the new cue is dropped — log shows
   `Skipping (<voice> still speaking)`. Browser TTS queues instead. Consider adding a
   small per-voice queue (size 1, drop-oldest) so near-miss cues still play.
   Code: `speakWithTTS`, the `voiceAudioPlaying[voice]` check.

2. **Speaker-bleed into mic.** Calibration helps but isn't perfect with omnidirectional
   mics. User now has a Shure MV7 which is much better. Browser TTS bleeds far more than
   OpenAI TTS because browser speechSynthesis bypasses the browser's echo cancellation
   (it goes to the OS audio engine). Recommendation baked into UI: use OpenAI TTS for
   everything.

3. **Short-utterance TTS reliability.** OpenAI `tts-1` sometimes returned truncated audio
   for very short phrases. Mitigations applied: switched to `tts-1-hd`; there's a
   `[TTS WARN]` log if returned audio is suspiciously short. Could add: retry-on-short,
   or pre-fetch/pre-cache all category items at session start.

4. **Pronunciation.** "a" → "Ay" at start of chord/pitch cues. Comma inserted before
   major/minor when there's an accidental ("E flat, minor"). IPA notation supported as
   a manual override in CSVs. No "number"/"chord" padding words (user rejected those).

## Key functions / where things live

- `init()` — startup, loads manifest, applies defaults.
- `loadServerContent()` — fetches manifest + CSVs + scores; runs CSV repair.
- `parseCSV()`, `repairExtractItems()`, `repairChordItems()`, `stripExtractPrefix()`.
- `parseScore()` / `parseCueInstruction()` — score parsing.
- `fireCue(category)` — picks a cue item, transforms text for TTS, calls `speak()`.
- `speak()` / `speakWithTTS()` / `speakGPTBrowser()` — audio output.
- `speakGPT()` / `processAISpeechQueue()` — priority AI voice queue.
- `runGPTInstruction()` / `handleConversationalInput()` — OpenAI Chat calls.
- `startVoiceActivityDetection()` — mic VAD loop; uses `calibratedThreshold`.
- `transcribeWithWhisper()` — sends recorded audio to Whisper, filters result.
- `calibrateVoice()` — two-step volume calibration.
- `getTTSAudio()` — OpenAI TTS fetch + cache (keyed `voice_text`).
- `convertIPA()` — IPA-to-phonetic conversion.
- `numberToWords()` — digit → word for extract cues.

## Conventions

- No build step. Edit the HTML directly. All JS is in one `<script>` block.
- After edits, sanity-check JS parses (e.g. `node -e` wrapping the script body in
  `new Function(...)`).
- There's a visible build banner near the top of the Settings tab — bump its version
  string when shipping a change so the user can confirm cache-busting.
- Memory/localStorage keys are prefixed `jt_`.
- TTS model is `tts-1-hd`. Whisper endpoint is OpenAI `/v1/audio/transcriptions`.

## Testing tips

- Hard refresh (Cmd+Shift+R) after each deploy — the build banner confirms the version.
- The log panel is the main diagnostic surface. `[CSV <cat>]` lines show loaded items.
- For pronunciation tests, space cues 3-4s apart so they don't skip.
