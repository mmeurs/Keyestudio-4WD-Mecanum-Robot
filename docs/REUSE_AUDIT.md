# Reuse Audit — Self-Hosted Personal AI Memory

Phase 0 / Milestone 0 deliverable. Date: 2026-10-04.

This audit inspects the candidate open-source projects **before** any implementation,
maps their capabilities against the user story, and records what can be reused.
The companion decision document is [`ARCHITECTURE_DECISION.md`](ARCHITECTURE_DECISION.md).

## 0. Method and limits of this audit

| Step | What was done | Result |
|---|---|---|
| 1. Inspect repositories | Shallow-cloned all three repos and read every source file of the watch/phone/AI paths (not just READMEs). | Done |
| 2. Test current build | Attempted. The audit environment's network policy blocks `dl.google.com` (Android SDK / Gradle Android plugin), so **no Android module could be compiled here**. Hugging Face is blocked too, so Whisper/pyannote models could not be downloaded for an STT benchmark. | **Not verified — must be repeated on a developer machine (see §8)** |
| 3. Map capabilities | Done from source, see §2 matrix. | Done |
| 4. Missing capabilities | Done, see §4. | Done |
| 5. Licenses | Read `LICENSE` / `NOTICE` files directly. | Done |
| 6. Select components | See §5 and `ARCHITECTURE_DECISION.md`. | Done |
| 7. Document decision | `ARCHITECTURE_DECISION.md`. | Done |

Statements marked *(upstream claim)* come from the project's own docs and were not
reproduced here. Statements marked *(source)* were verified by reading the code.

Legend used in the matrix:

- **as-is**: works as-is
- **modify**: reusable with modification
- **ref**: architecture/reference only
- **unsuitable**: unsuitable
- **missing**: missing / not present

---

## 1. Project overview

| | omi4WearOS | Daedalus Echo | iDictate |
|---|---|---|---|
| Repo | `neurocis/omi4WearOS` (attribution in NOTICE points to `cipioh/omi4wos-cipioh`) | `DaedalusApps/daedalus-echo` | `petepiet/iDictate` |
| License | **Apache-2.0** + NOTICE with attribution requirement *(source)* | **MIT** (added 2026-08-25) *(source)* | **GPL-3.0** *(source)* |
| Last commit | 2026-04-12 | 2026-08-25 | 2026-07-21 |
| History | 57 commits, 2026-04-07 → 2026-04-12 (6 days of very intensive work, then quiet ~6 months) | 117 commits, 2026-07-16 → 2026-08-25, PR-based, has CI workflow (`.github/workflows/ci.yml`) | 4 commits, 2026-06-21 → 2026-07-21 |
| Activity assessment | Young and dormant. Usable as a code base we own via fork, not as a maintained upstream. | Most active and best-engineered (tests, CI). | Small, effectively a single-author snapshot. |
| Size (Kotlin) | ~5.3 k lines (`:wear`, `:mobile`, `:shared`) | ~9.8 k lines + 41 test files | ~3.0 k lines |
| Language / stack | Kotlin, Compose, Wear Compose, Room, WorkManager, OkHttp | Kotlin, Compose, Room (FTS4), MediaPipe LLM + Text Embedder, sherpa-onnx | Kotlin, Views, Room + SQLCipher, OkHttp |
| Modules | watch + phone + shared | phone only | watch + phone |
| Min/target SDK | wear 30/34, mobile 28/34, compileSdk 34, JDK 17 | 26/35, compileSdk 35, JDK 21, AGP 8.7.3, Kotlin 2.0.21 | watch 30/34, phone 26/34, JDK 17 |
| Build status | Not compiled here (SDK download blocked). Prebuilt APKs are committed in `releases/` (`Omi4wOS_Wear_v2.0.apk`, `Omi4wOS_Mobile_v2.0.apk`). | Not compiled here. Upstream CI badge exists; first build downloads a ~56 MB sherpa-onnx AAR from GitHub Releases. | Not compiled here. |
| Tested hardware | **Galaxy Watch 7** (battery measurements in changelog) *(upstream claim)* | Galaxy S24 Ultra *(upstream claim)* | Galaxy Watch 6 Classic *(upstream claim)* |

---

## 2. Capability matrix

| Capability | omi4WearOS | Daedalus Echo | iDictate | Reuse decision |
|---|---|---|---|---|
| Watch recording | **as-is** — `AudioRecorder` (16 kHz PCM16 mono, `AudioSource.MIC`) + `AudioCaptureService` *(source)* | missing (phone app) | ref — uses `SpeechRecognizer` on the watch; it streams no raw audio and keeps no audio file *(source)* | **Reuse omi4WearOS** |
| Background recording | **as-is** — foreground service `microphone\|dataSync`, `PARTIAL_WAKE_LOCK`, `START_STICKY` with persisted `recording_enabled` resume *(source)* | ref (phone FGS) | ref (phone `RecordingService`, 3-min wake-lock cap on watch) | **Reuse omi4WearOS** |
| Galaxy Watch 7 support | **as-is** *(upstream claim; battery ≈ +18–24 mAh/h, ~1 %/5 min while recording)*. Notes 32-bit `armeabi-v7a` userspace and pins OnnxRuntime 1.14.0 to avoid SIGBUS. | unsuitable (needs ARM64 + 4 GB RAM) | ref (GW6 Classic) | **Reuse omi4WearOS**; re-verify on the user's watch (Milestone 1) |
| VAD | **as-is** — loudness gate + Silero LSTM via `gkonovalov/android-vad`, pre-roll 3.5 s, hysteresis *(source)* | missing | missing | **Reuse, but turn off for MANUAL mode in the MVP** (see ADR §8) |
| Opus | **modify** — MediaCodec Opus 24 kbps, 20 ms frames, written as **4-byte LE length-prefixed raw Opus frames** (Limitless/Omi `.bin` layout), no Ogg container; silent PCM fallback when the encoder fails *(source)* | missing | missing (phone uses AAC/M4A for OpenAI) | **Reuse encoder**; server decodes `.bin` → Ogg/Opus with libopus (small converter) |
| Local buffering | **modify** — `ChunkRepository` file queue, 500 MB cap. **Gap:** when the quota is exceeded it **deletes the oldest unsent audio** *(source)* — contradicts "no recording disappears silently". | ref | ref | **Reuse, change quota policy** (stop + warn instead of dropping) |
| Watch-phone transport | **modify** — `MessageClient` chunks ≤100 KB on `/audio/speech`, control on `/audio/control`, capability discovery. **Gaps:** the watch deletes a chunk as soon as `sendMessage()` resolves, which only means the message was handed to the transport, not that the phone persisted it. The phone assembles segments **in memory** (`activeSegments`) until `isFinal`, so a phone process kill mid-segment loses audio. Dedup key is a hash of the first 48 bytes. *(source)* | missing | ref — clean Data Layer command pattern (`/recording-state`, start/stop from watch) *(source)* | **Reuse omi4WearOS + add an app-level ACK**: delete on the watch only after the phone confirms it persisted the chunk |
| Phone storage | **modify** — Room `uploads` table (unique `segment_id`) + `.bin` files in **`cacheDir`** (the OS may evict these; the retry runner then *marks records as uploaded* and gives up) *(source)* | ref (Room, WAL) | ref — SQLCipher + Keystore-backed `EncryptedSharedPreferences` *(source)* | **Reuse Room queue, move files to `filesDir`**, never auto-dismiss missing files silently |
| Server upload | **modify** — OkHttp multipart POST to Omi `/v2/sync-local-files`, WorkManager retry every 15 min *(source)*. Hard-wired to Omi Cloud + Firebase token refresh. | missing (local-only) | unsuitable (OpenAI/Anthropic APIs) | **Reuse upload/retry plumbing, replace endpoint + auth** (Milestone 3) |
| Whisper | missing (removed; Omi Cloud did STT) | ref — sherpa-onnx + **Whisper `base.en` int8** (English-only), fixed 30 s chunks, **no timestamps kept**, on-device *(source)* | unsuitable (OpenAI cloud) | **Server-side faster-whisper / WhisperX** (multilingual large-v3 class models); Daedalus as reference only |
| Speaker diarization | missing | missing | missing | **Missing everywhere → reuse WhisperX's pyannote integration** on the server |
| Summaries | missing | **modify** — `SmartAnalysis` prompt + tolerant JSON/markdown parser (title, short summary, topics, mind map) *(source)* | unsuitable (Claude API) | **Port the prompt and parser concepts** to the server (Python); MIT allows code reuse |
| Action extraction | missing | **modify** — `TodoExtraction` prompt + bullet parser + fuzzy dedup *(source)*. It produces text-only todos with no source segment. | unsuitable (Claude API) | **Port, extend** to typed memories that link to source segment IDs |
| Semantic search | missing | ref — one embedding per *recording* (USE via MediaPipe), brute-force cosine, top-k 3–5, topic-sibling expansion *(source)* | missing | **Reimplement the concept with pgvector at segment granularity** (Kotlin/MediaPipe code cannot run in a Python/Postgres server) |
| Full-text search | missing | ref — SQLite FTS4 over transcript + summary *(source)* | missing | **PostgreSQL FTS** (`tsvector`, `simple`/`dutch`/`english` configs) |
| Source citations | missing | ref — "Ask Your Notes" cites **note titles** only; prompt says "if the answer is not in the notes, say so" *(source)* | missing | **Concept reuse**; extend to segment + timestamp citations |
| Audio timestamp playback | missing | ref — ExoPlayer playback of a recording, no transcript↔time mapping *(source)* | missing | **Missing → small web UI with `<audio>` + `currentTime` seek** |
| Self-hosted backend | **unsuitable** — the target is Omi Cloud (Firebase auth). The Omi backend itself is open source but depends on Firebase, Deepgram, OpenAI, Pinecone, Redis and GCS (see §6). | missing (on-device only) | missing | **Missing → minimal FastAPI + Postgres + worker** |
| Double Pinch | missing — Tile + on-screen button + phone command only *(source)* | missing | missing | **Missing**. There is no public Samsung API, so use native mechanisms (see §7). |

### Additional capabilities in the user story

| Capability | omi4WearOS | Daedalus Echo | iDictate | Reuse decision |
|---|---|---|---|---|
| Haptics start=1 / stop=2 | `VIBRATE` permission only, no pattern used | — | **ref** — exactly `[0,150]` for start and `[0,150,120,150]` for stop *(source)* | Re-implement in about 10 lines (pattern idea from iDictate, no code copied) |
| Watch Tile start/stop | **as-is** — `Omi4wosTileService` | — | — | Reuse as fallback control |
| Remote start/stop from phone | **as-is** — `WatchCommandReceiver` (`start`/`stop`/`FORCE_SYNC`) | — | ref | Reuse |
| Crash-safe drafts / encrypted storage | — | — | **ref** (SQLCipher, EncryptedSharedPreferences) | Concept: store the server token in Keystore-backed prefs. SQLCipher is not needed because the phone is not the system of record. |
| Ask Memory UI | — | ref (Compose `AskHomeScreen`) | — | Web UI on the server |
| Mind map / knowledge graph | — | as-is (on phone) | — | **Out of scope** |

---

## 3. Per-project technical notes

### 3.1 omi4WearOS: primary base

**Strengths**

- Proven on the target watch model, with a battery profile *(upstream claim)*.
- Clean module split: `:shared` (protocol constants, `AudioChunk`), `:wear`, `:mobile`.
- The service owns the recording state: `AudioCaptureService.isRecording` is a `StateFlow` and the Activity only observes it. This already meets "UI Activity must not own recording state".
- Store-and-forward on the watch, dedup on the phone, WorkManager retry.
- Solid battery engineering: async encoder, idle classification throttling, and no UI animations.

**Technical debt / defects relevant to this project**

1. **Cloud lock-in**: `OmiApiClient`, `OmiConfig` (Firebase refresh token + Web API key that the user copies out of browser DevTools), `Constants.OMI_*`. All of this must be replaced.
2. **Silent data loss paths** (each conflicts with the reliability requirements):
   - watch `enforceStorageQuota()` drops the oldest unsent chunks;
   - the watch deletes a chunk after a transport-level send rather than after an application ACK;
   - the phone keeps partial segments in RAM;
   - phone `.bin` files live in `cacheDir`, and the retry runner marks missing files as "uploaded".
3. **VAD-only segmentation**: there is no notion of a user-initiated *recording session*. Audio is a stream of ≤60 s speech segments with silence removed, and segments are grouped later by a 30 s debounce plus a 5-min split on the phone. Mapping a transcript time back to wall-clock time therefore needs per-chunk timestamps, which the phone discards when it concatenates.
4. Chunk timestamps are approximate (`System.currentTimeMillis()` at extraction, fixed 960 ms duration).
5. The Opus PCM fallback silently writes raw PCM inside the same framing, so the codec is ambiguous.
6. The dedup key hashes only the first 48 bytes of each message.
7. Both apps use `applicationId = "com.omi4wos"`, which is required for the Data Layer. We rename it in our fork for both apps together.
8. No unit tests.
9. Release builds are signed with the debug key.
10. The watch requests `BODY_SENSORS` without using it.

**Android / Wear OS requirements**

Wear OS 3+ (API 30+), tested on API 34. It needs the `RECORD_AUDIO`, `FOREGROUND_SERVICE_MICROPHONE`, `WAKE_LOCK`, `POST_NOTIFICATIONS` and `REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` permissions. From Android 14, a microphone FGS **cannot be started while the app is in the background** unless an exemption applies. This is critical for gesture-triggered starts (see §7 and ADR risk R1).

**Hardware assumptions**

- 16 kHz mono mic.
- Hardware/MediaCodec Opus encoder (API 29+).
- 32-bit ARM userspace on GW7 *(upstream claim)*.

**Cloud dependencies**

- Omi Cloud (`api.omi.me`).
- Firebase Secure Token API (`securetoken.googleapis.com`).
- Google Play Services Wearable. Required for the Data Layer, and acceptable because no user data passes through a Google server for BT/Wi-Fi Data Layer messages between paired nodes.

### 3.2 Daedalus Echo: donor of AI-pipeline concepts and prompts

**Strengths**

- MIT license.
- Fully local architecture.
- Good tests.
- Tolerant LLM-output parsers.
- A sensible "Ask" flow: embed the question, run cosine top-k, add topic siblings within a char budget, then send a prompt with guardrails.

**Limits for this project**

- **English-only STT** (`base.en`, `language = "en"`). It cannot meet the Dutch requirement.
- **No timestamps**: the transcript is a single string, and its 30 s chunk boundaries are thrown away.
- **No diarization**.
- **Recording-level embeddings only**, so retrieval cannot point at a segment.
- Citations are note titles only.
- Phone-only. The models are sized for a phone (Gemma 3 1B, USE embeddings). That is too weak for reliable Dutch extraction, and it would make the phone the system of record, against the architecture principle.
- The code is Kotlin/Android (MediaPipe, sherpa-onnx AAR), so it **cannot be lifted into a server process unchanged**.

**Verdict**: *architecture/reference plus prompt/parser donor*. Port these pieces into the Python server:

- prompts: `SmartAnalysis`, `TODO_EXTRACTION_PROMPT`, `OFFLINE_GUARDRAIL`, `buildLibraryQuestionPrompt`;
- the parsing and dedup heuristics: `parseTodoLines`, `isDuplicateTodoNormalized`, `SmartAnalysisParser`;
- the retrieval strategy: seed top-k plus topic-sibling expansion with a budget.

We keep the MIT notice with each ported file.

**Requirements**: ARM64 phone, API 26+, 4–6 GB RAM, ~700 MB of models.

**Cloud deps**: none at runtime. Model downloads come from Kaggle/HF/GitHub.

### 3.3 iDictate: architecture reference only (GPL-3.0)

- The watch does **not record audio files**. It uses the platform `SpeechRecognizer`, which on Samsung typically sends audio to a cloud recognizer, or it uses the phone mic via Remote Input. Transcription and analysis use OpenAI and Anthropic APIs.
- Useful ideas:
  - watch↔phone recording-state messages with distinct haptic patterns (1 pulse start, 2 pulses stop);
  - an inactivity warning with Stop/Continue notification actions;
  - SQLCipher with a Keystore-protected passphrase;
  - continuously persisted drafts;
  - a Whisper hallucination filter list.
- **GPL-3.0**: copying its code would force the combined Android app to be GPL-3.0. Our base (Apache-2.0) is GPL-3-compatible in one direction only. **Decision: no code is copied. We learn from the ideas only and re-implement from scratch** (haptic patterns and message names are not copyrightable expression).

---

## 4. Gaps: capabilities nobody provides

| Gap | Size | Planned solution |
|---|---|---|
| Self-hosted ingest API (authenticated, idempotent) | Small | FastAPI endpoint `POST /api/v1/recordings` |
| Recording *session* concept (start→stop) on watch and phone | Small–medium | Session ID generated on Double-Pinch start. Chunks are tagged with it. A manifest is sent on stop. |
| Application-level ACK watch↔phone | Small | Phone replies `/audio/ack` after a durable write; the watch deletes the chunk only then |
| Server: `.bin` → Ogg/Opus | Small | ~40-line Python using `opuslib`/libopus + ffmpeg mux |
| Multilingual STT with word timestamps + diarization | **Reuse** | WhisperX (faster-whisper + pyannote) or faster-whisper only (see §5) |
| Segment-level storage, FTS, vector search | Small | PostgreSQL + pgvector, one schema |
| Typed memory extraction with evidence links | Medium | Prompt (ported from Daedalus) + JSON schema + segment-ID validation |
| Evidence-grounded Q&A with uncertainty | Medium | Hybrid retrieval → LLM with strict citation format → server-side citation validation |
| Web UI (Today / Conversations / Actions / Search / Ask) with seekable audio | Small–medium | Server-rendered HTML (Jinja + htmx) inside the API container |
| Double Pinch start/stop | Unknown until device test | Native Samsung mechanisms, see §7 |

---

## 5. Server-side components evaluated (beyond the three candidates)

The user story requires that existing projects also be preferred on the server.
These were evaluated from documentation and source knowledge. **None were benchmarked
here** (model downloads were blocked), so benchmarking is a Milestone 4 entry task.

| Component | License | Role | Assessment |
|---|---|---|---|
| **faster-whisper** (SYSTRAN) | MIT | STT | CTranslate2 Whisper. Multilingual (nl + en, code-switching handled per 30 s window). Has segment and word timestamps, built-in Silero VAD filter, and runs on CPU (int8) or NVIDIA GPU. **Provisional default.** |
| **WhisperX** (m-bain) | BSD-2 | STT + alignment + diarization | Wraps faster-whisper, adds wav2vec2 forced alignment (a Dutch align model exists) and pyannote diarization. **This is the reuse candidate for diarization.** pyannote models need a one-time Hugging Face token to download; after that they run offline. |
| whisper.cpp | MIT | STT | Excellent on CPU / Apple Silicon. Has timestamps but no diarization (only tinydiarize, English-only), so more glue is needed than with WhisperX. This is the fallback if the server has no NVIDIA GPU and Python performance is too low. |
| sherpa-onnx (as used by Daedalus) | Apache-2.0 | STT / diarization | Viable, but Daedalus uses it with `base.en`. On a server, faster-whisper/WhisperX have the larger ecosystem. |
| **pyannote.audio** 3.x | MIT (models gated, CC-BY-4.0/MIT) | Diarization | Used through WhisperX. |
| **Ollama** | MIT | Local LLM + embeddings | OpenAI-compatible HTTP API, so the LLM layer is a replaceable `LLM_BASE_URL`. Candidates: Qwen2.5/3 7–14B, Llama 3.1 8B, Gemma 3 12B (Dutch quality must be tested). Embeddings: `bge-m3` (multilingual, strong for nl/en) or `nomic-embed-text`. |
| **PostgreSQL 16 + pgvector** | PostgreSQL / PostgreSQL | Single DB: relational, FTS, vectors | One database covers everything. No Elasticsearch, Qdrant or Redis. |
| Omi backend (BasedHardware/omi `backend/`) | MIT | Would let the existing phone code talk to a "self-hosted Omi" | **Unsuitable**: it hard-depends on Firebase Auth/Firestore, Deepgram/Soniox STT, OpenAI LLM, Pinecone, Redis and GCS. Removing those is a bigger job than a 3-endpoint FastAPI service. |
| speaches / faster-whisper-server | MIT | STT as an HTTP service | An option if STT should be a separate container later. Not needed for the MVP; the worker imports faster-whisper directly. |

---

## 6. License summary

| Project | License | How we use it | Obligations |
|---|---|---|---|
| omi4WearOS | Apache-2.0 + NOTICE | **Fork; modify in place** | Keep `LICENSE` + `NOTICE`. Mark changed files (Apache §4b). Show the attribution string *"Based on omi4wOS, Copyright 2026 cipioh + neurocis. Licensed under the Apache License, Version 2.0. Original source: https://github.com/cipioh/omi4wos-cipioh"* in the README **and** the app's About screen. |
| Silero VAD / android-vad / OnnxRuntime | MIT | Transitive, via the omi4WearOS fork | Keep their notices (already listed in NOTICE) |
| Daedalus Echo | MIT | Port prompts, parsers and retrieval heuristics to Python | Keep the MIT copyright notice in ported files / `THIRD_PARTY_NOTICES` |
| iDictate | GPL-3.0 | **Ideas only, no code** | None, as long as no code is copied |
| faster-whisper / WhisperX / pyannote / Ollama / pgvector | MIT / BSD-2 / MIT / MIT / PostgreSQL | Dependencies | Notices in `THIRD_PARTY_NOTICES` |
| Model weights | Whisper: MIT. Llama / Gemma / Qwen: own model licenses. pyannote models: gated (accept terms). | Downloaded by the user at install time, never committed | User accepts the model licenses |

**Resulting project license**: **Apache-2.0**. This is compatible with every component we
incorporate, and it matches the primary base.

---

## 7. Double Pinch investigation

Findings (Samsung docs and press, 2024–2025):

- Double Pinch was introduced on One UI 6 Watch (Galaxy Watch 7 / Ultra) and extended in One UI 8 Watch. It is **contextual**: it answers or ends calls, dismisses alarms and notifications, scrolls notifications, **plays and pauses media**, controls timer/stopwatch and runs the camera shutter. Users configure it under *Settings → Buttons and gestures → Double pinch*.
- **No public SDK or API lets a third-party app receive the Double Pinch event**, and it cannot be assigned to "open app X" (this is an open community feature request).
- **Universal Gestures** (Accessibility) is a separate, full-UI gesture system (pinch, double pinch, fist, double fist). It drives a focus cursor and actions, so a user can "tap" the focused element of the foreground app.

Native mechanisms ranked for our need (none needs custom accelerometer code):

| # | Mechanism | How it would toggle recording | Pros | Risks / must verify on device |
|---|---|---|---|---|
| A | **MediaSession on the watch** | The recorder exposes a `MediaSession`. Double Pinch's built-in "play/pause media" sends `onPlay` / `onPause`, which map to start / stop in the service. | Works with the screen off and no app in the foreground; this is exactly the "contextual media" path Samsung supports. | (a) Does Double Pinch target *our* session if another media app was used last? (b) Android 14 FGS-from-background rules: the media-button path is an allowed FGS start exemption when the app has a media session, but microphone *while-in-use* access may still be denied, so the FGS may need to be pre-started ("armed"). (c) An idle "paused" session must stay the active media session. |
| B | **Ongoing Activity / ongoing notification primary action** | While armed, show an Ongoing Activity whose primary action is Start/Stop. One UI 8 Double Pinch acts on notifications. | Standard Wear OS API. | Double Pinch's notification behaviour is "dismiss/scroll", so whether it fires a custom *action* is unknown. |
| C | **Universal Gestures** | Double pinch → tap the focused big Start/Stop button in our app. | Supported accessibility feature. | Needs the screen on and our app in the foreground; turns on a global accessibility mode. This is the fallback only. |
| D | Large on-screen button / Tile (already in omi4WearOS) | Tap | Already exists | Not hands-free |

**Decision**: build A first as a spike in Milestone 8 and fall back to B, then C, then D.
Custom accelerometer gesture recognition is **not** planned. Gesture work does not block
Milestones 1–7 (the Tile/button is the interim control).

---

## 8. Build verification still owed (to run on a dev machine)

The audit could not compile anything because the Android SDK host was blocked. Before Milestone 1:

```bash
# omi4WearOS (JDK 17, SDK 34)
./gradlew :wear:assembleDebug :mobile:assembleDebug
# Daedalus Echo (JDK 21, SDK 35) — only to confirm donor code/tests compile
cd android && ./gradlew testDebugUnitTest
```

Record the results here (date, JDK, AGP, success or failure). Then run the Milestone 1 field test on the
user's Galaxy Watch 7 using the upstream prebuilt `Omi4wOS_Wear_v2.0.apk`, before modifying anything,
to get a baseline:

- screen-off recording;
- 60-minute battery drain;
- audio validity.
