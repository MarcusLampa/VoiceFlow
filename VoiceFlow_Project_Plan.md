# VoiceFlow — Project Plan

## 1. Purpose and first release

VoiceFlow is a native Windows dictation app inspired by Handy. It serves both short dictation into another application and long brainstorming sessions; preserving long sessions takes priority over minimum latency. The first user-facing release is a packaged app with a small recording popup, settings window, system tray, global hotkeys, continuous local audio recording, continuously written local transcription, and delivery of finished text.

English is the first-release language. Swedish support follows in a later release; mixed English/Swedish speech is evaluated after both languages work independently. Ollama cleanup, summarization, and in-app retranscription are later features.

Recording and transcription work without a network connection once the speech model has been downloaded during setup. The app may contact GitHub at startup solely to check for published updates; an offline or failed check never blocks dictation. Audio and transcripts stay local.

## 2. Session experience

1. The user places the cursor in a text field if they might want to paste there, then presses the configurable start/stop hotkey (`Ctrl+Space` by default). VoiceFlow remembers the target **before** showing its popup, whether or not automatic paste is currently enabled.
2. Audio capture starts immediately, including while the popup is displayed. The first spoken word must not depend on choosing a mode or on Whisper loading at hotkey time. The popup shows recording state, elapsed time, transcription state, a delivery-mode switch, and **Stop**.
3. The popup starts in the last-used delivery mode, or **Clipboard** on a fresh install. It stays available as a compact recording indicator. The user may switch between **File** and **Clipboard** there or press `Ctrl+M` to toggle between them. VoiceFlow claims `Ctrl+M` globally only while recording; its normal application behavior is unaffected at other times.
4. Pressing the start/stop hotkey again or clicking Stop ends capture. The selected delivery mode and the automatic-paste setting are locked at that moment. VoiceFlow finishes outstanding transcription before delivering text; another session cannot start during finalization. A growing backlog is shown and allowed to finish while it makes progress, without a fixed time cutoff.
5. If healthy speech detection finds **no voice**, discard the entire session folder (audio, transcript, metadata) and briefly show “No speech detected; session discarded.” If detection failed or is uncertain, preserve the audio and mark the session for attention instead of discarding it.
6. Otherwise, retain the original audio and raw transcript. The delivery mode affects only what happens **after** finalization:

   | Mode | Finished-text behavior |
   | --- | --- |
   | File | Leave text in the session files; do not change the clipboard or paste. |
   | Clipboard | Copy the spoken text to the clipboard and leave it there. If **Automatically paste after copying to clipboard** is enabled in settings at Stop, also paste into the field selected at the start **only when that target can be identified reliably**. If it cannot, keep the text on the clipboard and alert the user; never paste into an uncertain field. |

   Automatic paste is off by default and applies only to Clipboard delivery; File delivery never copies or pastes. Copy/paste content contains the spoken text only: no Markdown title, timestamps, or session metadata. `transcript.md` retains that context. The clipboard is not automatically restored.

If transcription fails or is incomplete, preserve the audio and any raw transcript, label the session clearly, and **do not automatically deliver partial text**. An audio-only session remains accessible as audio with “transcription unavailable”; manual access to its files is part of the first release, in-app retranscription is not. Losing the microphone or the ability to save audio stops capture and alerts the user. A writable output folder is a prerequisite to starting a session; a failed model load is not.

## 3. Non-negotiable data rules

- Every non-silent session retains original audio. There is no “Save original audio” checkbox. Only a confirmed silent session is automatically discarded.
- Capture and durable audio writing must not wait for speech detection, transcription, UI rendering, or optional LLM work. If transcription falls behind, continue capturing and finish transcription from retained audio after Stop; do not silently drop speech to preserve low latency.
- Append finalized recognition results to `transcript.md` while recording. Never overwrite the raw transcript with edited output. Optional future outputs use separate files such as `cleaned.md` and `summary.md`.
- File-mode delivery still produces **both** audio and transcript. “File” and “Clipboard” are delivery modes, not recording or retention modes; automatic paste is a setting for Clipboard delivery, not a third mode.
- Prefer preserving uncertain material to silently deleting or delivering it. A detected failure must be visible to the user.

## 4. Session files and lifecycle

Each session has a unique folder beneath a configurable output directory, e.g.:

```text
D:\Voice Notes\
  2026-09-29_2015_<short-id>\
    audio.wav
    transcript.md
    session.json
```

The timestamp makes folders readable; the unique ID prevents collisions. `audio.wav` is the continuous original recording. `transcript.md` is append-only during capture and contains elapsed timestamps with direct faster-whisper text (not LLM output). `session.json` records a stable ID, start/end times, microphone, model, language, delivery mode and automatic-paste setting at Stop, file names, progress and outcome. Store timestamps in an unambiguous format with timezone information; transcript timestamps are offsets into the session audio.

Example transcript:

```markdown
# VoiceFlow Session

Started: 2026-09-29 20:15

## Transcript

[00:00:07] I have been thinking about the architecture.

[00:00:19] Another option is to reference it by ID.
```

Distinguish **recording**, **finalizing**, **completed**, **interrupted**, and **incomplete transcription** in session metadata and in the UI. “Interrupted” means the session did not finish normally; “incomplete transcription” means the captured audio is available but text could not be completed. Do not mark a session completed merely because the audio file was closed. On startup, identify recording/finalizing sessions left by a previous run, retain their files, mark them interrupted, and give the user a way to open their folder. Repair or expose a playable version of a crash-truncated WAV when possible without overwriting the only surviving audio. Automatic retranscription can come later.

Persist audio and transcript as work proceeds, not only on Stop. Flush text after finalized segments and periodically sync as appropriate; ensure audio frames and a recoverable file layout survive ordinary failures as far as the OS/storage allows. A microphone callback must never perform slow disk writes, Whisper calls, or blocking queue waits. Audio-writing failures are critical and stop capture; speech-processing failures are non-critical to capture. If a transcription queue grows, use the retained recording as the source for catch-up rather than an unbounded memory-only backlog. Process segments in audio order and avoid duplicate/missing text at split boundaries.

## 5. Architecture and suggested stack

Native Windows, Python, and ordinary worker threads with bounded queues are sufficient for the first implementation:

```text
                                    ┌─→ continuous audio writer ─→ audio.wav
Microphone → short nonblocking handoff
                                    └─→ VAD/chunker → transcription worker
                                                           │
                                                    ordered finalized segments
                                                           ↓
                                                   transcript.md

UI / tray / hotkeys → session controller → state + delivery + recovery
```

The controller coordinates start, stop, finalization, silent-session discard, and interrupted-session discovery. It does not perform microphone or model work on the UI thread. Keep the recording path independent from transcription and optional post-processing. If processing cannot keep pace, recover unprocessed audio ranges from `audio.wav` while preserving ordering; design the callback-to-writer handoff so a full queue is reported as a critical capture failure rather than silently losing frames.

| Responsibility | Initial choice |
| --- | --- |
| GUI, popup and tray | PySide6 |
| Audio capture and writing | sounddevice, soundfile, NumPy |
| Recognition | faster-whisper / CTranslate2 |
| Global shortcut and target handling | Windows APIs / pywin32; choose a hotkey mechanism that can register/unregister `Ctrl+M` while recording and report conflicts |
| Settings and metadata | JSON in the user's application-data directory; no database |
| Packaging | PyInstaller or an equivalent Windows packaging approach |
| Later optional LLM | local Ollama API, separate from capture and raw transcription |

Begin with `large-v3` on an RTX 5070 Ti using CUDA/float16 as a **candidate**, and verify native Windows compatibility, model loading, memory use, and real-time throughput before committing to it. Offer a smaller/faster supported model if measurements justify it. Model loading should start before normal use when possible, without blocking audio-only recording if the model is unavailable. Pin tested dependency versions once the working CUDA/audio combination is known.

Measure rather than assume latency: aim for hotkey-to-capture under 250 ms when the microphone is available and for a speech pause to appear in the transcript within roughly 2–8 seconds under normal load. Neither target overrides completeness or recovery. A 20-minute session is required; longer sessions and changing foreground applications are useful stress tests.

Use a speech detector to end phrases after a pause and split uninterrupted speech at a maximum duration; starting points to test are a 500–1000 ms pause and a 20–30 s upper segment bound. Keep enough context at boundaries to avoid cut words, then deduplicate overlap. Never use “no transcript text” as proof of a silent session: silence discard requires a healthy, trustworthy speech-detection result. Model performance and detection quality must be measured with real English samples before fixing settings.

## 6. Windows interaction and UI

The small popup must not delay audio capture. Show the active File/Clipboard delivery mode and a clear recording/finalizing state. Provide a Stop button and non-intrusive sound or visual feedback; allow sounds to be disabled. The settings window exposes microphone, output folder, start/stop hotkey, model, language, feedback preferences, and a persistent **Automatically paste after copying to clipboard** toggle (off by default). Detect hotkey registration conflicts and let the user choose another start/stop shortcut. Closing settings minimizes to the tray, not an accidental end to recording.

Tray actions: Start/Stop, show current session or recordings folder, open settings, and Exit. Exit during recording/finalization must not silently discard or abandon a session. The app cannot guarantee arbitrary Windows text fields can be targeted after focus moves: capture the intended target before opening the popup even if automatic paste is off, verify that the target is still safe before pasting, and fall back to clipboard with an explanation. Test the behavior in browsers, editors, Notepad, and Word; do not promise field-level restore in applications where it cannot be established.

## 7. Updates and network behavior

Publish versioned packaged Windows assets through **GitHub Releases**. On every app startup, check the latest published release asynchronously with a short bounded network timeout. Compare app versions, not commit dates or branch heads. If a newer release exists, show its version and notes with an action opening its GitHub release page so the user can install it. The first packaged release includes this check and prompt; it does **not** download or execute an update itself. No connection, timeout, API error, or malformed response may prevent the tray/hotkeys/recording from becoming ready. Downloading the speech model during initial setup is separate from this check; after setup, transcription remains offline.

## 8. Implementation milestones

These are development checkpoints; the **first user-facing release** includes all the core items below.

1. **Audio prototype:** select a microphone, record to a durable single session audio file, handle device/storage failure, and verify a 20-minute recording and crash recovery behavior.
2. **Recognition prototype:** transcribe existing English WAV files locally, test CUDA/model compatibility and speed on the target PC, and establish actual segmentation settings.
3. **Live pipeline:** record and transcribe concurrently; append timestamped finalized segments while recording; test long sessions, backlog catch-up, silent detection, and audio-only fallback.
4. **Windows interaction:** configurable `Ctrl+Space` start/stop; popup and `Ctrl+M` switching between File and Clipboard only while recording; optional automatic paste into a verified original selected field after Clipboard delivery; tray and settings.
5. **Reliability and release:** session states and metadata, interrupted-session discovery, warnings and logs, packaging, fresh-install model setup, and nonblocking GitHub release check. Test the packaged app without a terminal and offline after initial setup.
6. **Later:** Swedish, then mixed-language evaluation; in-app retranscription; optional Ollama cleanup/summary in separate files; enhanced session browsing and startup-with-Windows preference. Keep these out of the first-release acceptance gate.

## 9. First-release acceptance scenarios

- Starting from a selected text field with `Ctrl+Space` begins capture before mode selection; the first words are present in `audio.wav`. Both the hotkey and Stop button end capture; a second session cannot start while the first is finalizing.
- A 20-minute English session continuously grows `audio.wav` and `transcript.md`. When transcription falls behind, audio remains complete and the transcript catches up after Stop. Delivery contains speech only, not transcript headers or timestamps.
- File mode leaves the clipboard and other apps alone even when automatic paste is enabled. Clipboard mode leaves complete text ready to paste; with automatic paste enabled at Stop, it also pastes only when the original field can be targeted safely. If automatic paste is enabled but the original field cannot be verified, it alerts and leaves complete text on the clipboard without pasting into another field. The setting is off by default and its value, like the mode, is fixed at Stop. Switching between File and Clipboard via `Ctrl+M` and the popup works during recording; `Ctrl+M` acts normally outside recording.
- Confirmed no-voice capture discards the entire folder and shows brief feedback. If speech detection is broken or uncertain, the recording remains. A model failure permits an explicitly audio-only session; a later recognition failure retains audio and partial text without automatically pasting it.
- Microphone disconnect or output-write failure stops capture with a visible error. Forced termination retains recoverable files to the extent written; relaunch flags an interrupted session and exposes its folder.
- A packaged clean install can download its model, then transcribe without internet. Each subsequent startup checks GitHub releases without blocking readiness; an available update links to the release page, while offline startup remains usable.

## 10. Explicitly later or out of scope

First release does not require Ollama, Swedish or mixed-language acceptance, retranscription inside the app, automatic installation of updates, word-by-word captions, continuous typing into the target, a web server, cloud transcription, accounts, Docker, WSL, a database, diarization, or a plugin system. When Ollama is added, it must read the raw transcript after Stop and write separate cleaned/summary outputs without replacing `transcript.md` or blocking audio capture.
