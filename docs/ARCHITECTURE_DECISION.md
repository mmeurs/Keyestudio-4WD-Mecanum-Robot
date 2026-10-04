# Architecture Decision — Self-Hosted Personal AI Memory

Status: **Proposed** (Phase 0 output, awaiting owner sign-off before Milestone 1)
Date: 2026-10-04
Input: [`REUSE_AUDIT.md`](REUSE_AUDIT.md)

Guiding rule: *maximum useful result / minimum custom code*. Every subsystem below names
the existing project that provides it, and only the gaps listed in REUSE_AUDIT §4 get new code.

---

## Summary

```
Galaxy Watch 7  ── fork of omi4WearOS :wear (recorder, Opus, store-and-forward, Data Layer)
      │  Wear Data Layer (BT / Wi-Fi), chunk + app-level ACK
      ▼
Samsung phone   ── fork of omi4WearOS :mobile (receiver, Room queue, WorkManager retry)
      │  HTTPS + bearer token, LAN / Tailscale; Omi client replaced by MemoryServerClient
      ▼
Home server     ── docker compose: api (FastAPI + web UI) · worker · postgres+pgvector · ollama
                   STT = faster-whisper/WhisperX (pyannote diarization)
                   AI prompts/parsers ported from Daedalus Echo
```

---

## 1. Which repository becomes the primary base?

**`neurocis/omi4WearOS`** (Apache-2.0). We fork it into our project and develop the watch and phone
apps there (`:shared`, `:wear`, `:mobile` stay as they are). It is the only candidate that already
records on a Galaxy Watch 7 in the background with VAD, Opus, local buffering and Data Layer
transport. It already has a phone companion with a queue and retry.

## 2. Which repositories provide donor components?

| Donor | What we take | Form |
|---|---|---|
| Daedalus Echo (MIT) | Analysis prompt + `SmartAnalysisParser` logic; `TODO_EXTRACTION_PROMPT` + `parseTodoLines` + fuzzy dedup; `OFFLINE_GUARDRAIL`; the "Ask" retrieval strategy (top-k plus topic-sibling expansion within a char budget, "say so if not in the notes") | Ported to Python, MIT notice kept |
| iDictate (GPL-3.0) | Ideas only: start/stop haptic patterns sent as recording-state messages; inactivity warning with Stop/Continue actions; Keystore-backed secret storage | Re-implemented from scratch, **no code copied** |
| WhisperX / faster-whisper (BSD-2 / MIT) | STT, word timestamps, alignment, pyannote diarization | Library dependency |
| Ollama (MIT) | Local LLM + embedding runtime | Docker image, OpenAI-compatible HTTP |
| PostgreSQL + pgvector | Storage, FTS, vector search | Docker image |

## 3. Which components remain unchanged?

From omi4WearOS:

- `wear/audio/AudioRecorder.kt` (16 kHz PCM capture)
- `wear/audio/OpusEncoder.kt` (MediaCodec Opus, `.bin` framing)
- `wear/audio/CircularAudioBuffer.kt`
- `wear/audio/SpeechClassifier.kt` (Silero VAD; kept for AUTO mode, idle in MANUAL)
- `wear/service/DataLayerSender.kt` (send and connectivity; ACK handling is added next to it)
- `wear/tile/Omi4wosTileService.kt`, `wear/service/WatchCommandReceiver.kt`
- `mobile/service/WatchReceiverService.kt`, `mobile/service/UploadRetryWorker.kt` (WorkManager)
- The Room plumbing (`UploadDatabase`, DAO pattern), Compose theme and screens skeleton
- `shared/DataLayerPaths.kt` and `shared/Constants.kt`, with paths and constants only *added*

## 4. Which components need modification?

| Component | Change | Why |
|---|---|---|
| `AudioCaptureService` (wear) | Add a **recording-mode** switch (`MANUAL` / `AUTO` / `PRIVACY`). In `MANUAL`, every frame between start and stop is encoded with no VAD gate, tagged with a `recordingId` (UUID v4) plus monotonically increasing chunk index and sample offset. Start sends 1 haptic pulse and stop sends 2. A stop writes a final chunk plus a **manifest** (start/end wall clock, sample count, codec, SHA-256 of the concatenated payload). Add a `MediaSession` for Double Pinch (§8). | Product is manual start/stop; traceability needs a gap-free timeline |
| `ChunkRepository` (wear) | Never drop unsent audio. When the quota is reached, stop recording with a 3-pulse warning and a notification. Delete a chunk only after the phone ACKs it. | "No recording should disappear silently" |
| `AudioReceiverService` (mobile) | Write each chunk to `filesDir/recordings/<recordingId>/<index>.opusbin` **before** ACKing (`/audio/ack` with `recordingId`, `index`). Dedup on `(recordingId, index)`. When the manifest arrives and all chunks are present, assemble `<recordingId>.bin` and verify the SHA-256. | Durable receipt and exact dedup |
| `UploadRecord` / `UploadDao` (mobile) | Becomes `RecordingRecord` with lifecycle states (§11), `checksum` and `attempts`. Room migration. | Queue, retry and sync-status UI |
| `AudioUploadService` + `UploadRetryRunner` (mobile) | Delete the Omi grouping/flush logic and upload one recording per request to the self-hosted API. A missing file → state `ERROR_MISSING_FILE` and show it in the UI, **never** mark it uploaded. Retry with backoff when online and when the server is reachable. | Replace Omi Cloud |
| `OmiApiClient` / `OmiConfig` → `MemoryServerClient` / `ServerConfig` | Server base URL + device token stored in `EncryptedSharedPreferences` (Keystore). "Test connection" calls `GET /api/v1/health`. | Remove the cloud dependency |
| Settings + Home screens (mobile) | Fields for server URL and token; the per-recording sync state list replaces the Omi UI | Configure and observe |
| `applicationId`, app name, About screen | Rename (e.g. `nl.<owner>.memorywatch`) in **both** apps; show the Apache NOTICE attribution | Separate identity; license compliance |

## 5. Which pieces genuinely need to be written from scratch?

These are the gaps from REUSE_AUDIT §4. All other functionality is reused.

1. **Server ingest API**: `POST /api/v1/recordings`, idempotent, bearer-token auth.
2. **`.bin` → Ogg/Opus converter**: about 40 lines using libopus via `opuslib`, then an ffmpeg mux.
3. **Worker pipeline**: a Postgres job queue (`SELECT … FOR UPDATE SKIP LOCKED`, so no Redis) that runs STT, then segments, embeds, analyses and extracts memories.
4. **Schema plus hybrid retrieval**: FTS + pgvector with reciprocal-rank fusion.
5. **Evidence-grounded Ask**: prompt → answer → citation validation → uncertainty wording.
6. **Minimal web UI**: Today, Conversations, Actions, Search, Ask, and Conversation detail with a seekable `<audio>`.
7. **Watch additions**: session/manifest, ACK, MediaSession hook, haptics (small edits to the fork).
8. **Tests**:
   - API idempotency and auth;
   - `.bin` decoder round-trip;
   - pipeline with a stub STT/LLM;
   - citation validation;
   - Kotlin unit tests for chunk/ACK/dedup logic.

Rough estimate of new code: server ~2–3 k lines of Python/HTML; Android diff ~1 k lines.

## 6. How will licensing be handled?

- The project license is **Apache-2.0**.
- The fork keeps omi4WearOS's `LICENSE` and `NOTICE`. Modified files carry a "Modified by …" header (Apache §4b). The attribution string required by NOTICE appears in the README and in both apps' About screens.
- Ported Daedalus Echo logic sits in `server/app/ai/` with a header comment that quotes the MIT copyright line. A root `THIRD_PARTY_NOTICES.md` lists all dependencies.
- **No iDictate (GPL-3.0) code enters the repository.** Reviewers reject any PR that copies from it.
- Model weights are never committed. The user downloads them at install time (`ollama pull …`; Whisper/pyannote are fetched on first worker start into a volume). pyannote needs the user to accept its HF terms once.

## 7. Where will AI processing run?

**Only on the home server**: STT, diarization, embeddings, LLM analysis and Q&A. The watch does capture
and optional VAD. The phone runs **no AI**. Daedalus Echo's on-phone Gemma/Whisper was rejected
because it is English-only, has no timestamps and no diarization. It is also too weak for Dutch
extraction, and it would make the phone the system of record.

All AI endpoints are configured through environment variables (`STT_MODEL`, `STT_DEVICE`, `LLM_BASE_URL`,
`LLM_MODEL`, `EMBEDDING_MODEL`). They default to the in-compose Ollama, and no external API is
configured by default.

## 8. What remains on the Watch?

- Audio capture, Opus encoding, the recording state machine (authoritative in the foreground service) and haptics.
- Store-and-forward queue with app-level ACK.
- **Controls**:
  1. Double Pinch via `MediaSession` play/pause → `toggle()`;
  2. fallbacks: Ongoing Activity action, Universal Gestures, Tile, large on-screen button.
- **Recording modes** as an enum in `:shared`: `MANUAL` (MVP), `AUTO` (existing omi4WearOS VAD behaviour, kept and disabled by default), `PRIVACY` (mic hard-off), `BOOKMARK` (later: sends a timestamp marker event inside the current recording).
- **VAD in MANUAL mode**: off for the MVP, so audio offset = wall-clock offset (exact traceability). As an optional later optimisation, VAD may mark silence and the encoder can emit DTX/low-bitrate frames *without* removing time. Silence is never cut out of the timeline.

## 9. What remains on the phone?

The phone is a transport and resilience layer only:

- receive chunks, persist them durably, ACK, assemble and verify each recording;
- queue and upload with retry/backoff, idempotently;
- configure the server URL and token (Keystore-encrypted);
- show sync state per recording;
- delete local audio **only after** the server confirms `SERVER_RECEIVED` with a matching checksum (configurable retention, default 7 days, as a safety net).

## 10. What runs on the home server?

`docker compose up -d` starts four services (no separate frontend service: the API serves the HTML):

| Service | Image | Role |
|---|---|---|
| `api` | own (python:3.12-slim + FastAPI + Jinja/htmx) | Ingest API, search/ask API, web UI, audio streaming with HTTP Range |
| `worker` | own (same image, `[stt]` extra; optional CUDA variant) | Job loop: convert → transcribe → diarize → segment → embed → analyse → extract |
| `postgres` | `pgvector/pgvector:pg16` | Single system of record (relational + FTS + vectors + job queue) |
| `ollama` | `ollama/ollama` | Local LLM and embeddings |

Volumes: `pgdata`, `audio` (originals plus derived Ogg), `models` (Whisper/pyannote cache), `ollama`.
Configuration comes from `.env`, and `.env.example` is committed while `.env` is git-ignored.
TLS options: put Caddy in front on the LAN, or use `tailscale serve` (HTTPS with automatic certs). Both are
documented, and neither is mandatory inside compose.

### Data model (PostgreSQL)

```
recording(id uuid pk, device_id, start_time timestamptz, end_time, duration_ms,
          codec, checksum_sha256, file_path, playback_path, status, error, attempts,
          created_at, updated_at)                      -- unique(id); checksum checked on re-upload
transcript_segment(id bigserial pk, recording_id fk, idx, start_ms, end_ms,
          speaker_label, language, text, words jsonb,
          tsv tsvector generated, embedding vector(N))
conversation(id pk, recording_id fk, title, summary, topics text[], start_time, end_time)
memory(id pk, conversation_id fk, type enum(action, agreement, decision, promise,
          appointment, deadline, fact, topic, person), text, owner_speaker,
          due_text, due_at, confidence real, model, created_at)
memory_source(memory_id fk, transcript_segment_id fk, pk(memory_id, segment_id))
job(id pk, recording_id fk, stage, status, run_after, attempts, last_error)
```

MVP rule: one recording = one conversation. Splitting long recordings into several conversations is a
later refinement.

### Ingest API

`POST /api/v1/recordings` (multipart: `metadata` JSON + `audio` file), with `Authorization: Bearer <device token>`.

- Metadata fields: `recording_id`, `device_id`, `start_time`, `end_time`, `codec` (`opus-bin-v1`), `duration_ms`, `checksum` (sha256).
- The server recomputes the checksum and rejects a mismatch with `422`.
- **Idempotency**:
  - same `recording_id` + same checksum → `200` with the existing status (no duplicate);
  - same `recording_id` + different checksum → `409`;
  - new → `201`.
- The file is written to a temp path, fsynced, atomically renamed, and only then is the DB row committed. The response is sent last.

Other endpoints:

- `GET /api/v1/recordings/{id}` returns the status, so the phone can show the server-side state.
- `GET /api/v1/health`.

Device tokens are stored hashed (argon2/sha256) and created with a CLI command (`python -m app.cli create-token`).

## 11. How does audio travel through the system?

```
WATCH_RECORDING        mic → PCM → Opus frames → chunk files (recordingId, idx) on watch disk
      │ stop: manifest(start,end,samples,sha256)
WATCH_PENDING_TRANSFER chunks + manifest sent via MessageClient; deleted only on /audio/ack
      ▼
PHONE_RECEIVED         chunks fsynced in filesDir, assembled, sha256 verified vs manifest
SERVER_PENDING         Room row queued; WorkManager uploads with backoff (network constraint)
      ▼
SERVER_RECEIVED        original .bin stored immutable; checksum verified; job enqueued
TRANSCRIBING           .bin → .ogg (playback + STT input); WhisperX: nl/en auto-detect, word timestamps,
                       pyannote → Speaker 1..n
INDEXING               segments → tsvector + embeddings (Ollama EMBEDDING_MODEL)
ANALYSING              title/summary/topics + typed memories with source segment ids
READY
(any stage) ERROR_<stage> → retryable; job.attempts++; exponential backoff; visible in UI
```

Each failure mode maps to a defined behaviour:

| Failure | Behaviour |
|---|---|
| Watch↔phone link down | Chunks stay on the watch |
| Phone or server offline | The Room queue keeps the recording |
| Server restart | Jobs are rows in Postgres and resume |
| STT crash | The job is retried; the original audio is untouched |
| LLM down | The recording reaches `INDEXED` (searchable via FTS) and analysis retries later |

## 12. How will source evidence remain traceable?

- **The evidence is immutable**. The original `.bin` is never modified, and its checksum is recorded. The playback `.ogg` is derived from it and can be regenerated.
- **Timestamps** come from the STT on the gap-free MANUAL audio. A segment's `start_ms`/`end_ms` is the offset into the recording, and `recording.start_time + start_ms` gives the wall-clock time. Example: `2026-10-04_143017 · 00:17:31.200–00:17:38.900 · Speaker 2`.
- **Interpretation is kept separate from evidence**: `conversation.summary` and `memory.*` sit in separate tables flagged as AI-generated, and every memory has ≥1 `memory_source` row. The extractor must return segment IDs drawn from the prompt's own segment list, and the server **drops any memory whose cited IDs were not in the provided context**.
- **Ask Memory** works as follows:
  1. Interpret the query; extract any date range (e.g. "vandaag", "deze week") deterministically in Python.
  2. Run FTS and vector top-k over segments, fuse them, and expand each hit by ±2 neighbouring segments.
  3. Give the LLM a numbered context, e.g. `[S7812] 2026-09-24 14:32 · Rec 812 · Speaker 2: "…"`.
  4. Require the LLM to cite `[S…]` for every claim, and validate the citations on the server.
  5. The UI renders each citation as a link to `/conversations/{rec}#t=1051.2`, which seeks the `<audio>` to that point and plays it.
- **Negative answers**: the prompt and the post-check enforce three outcomes:
  - **found supporting evidence**: answer with sources;
  - **found contradicting evidence**: say so, with sources;
  - **no evidence found**: answer *"Ik kan geen bewijs vinden dat … (doorzocht: N opnames, periode X–Y)"*, never "this was never said".

  An answer whose citations are all invalid is replaced with the "no evidence found" template.

---

## Rejected alternatives

| Alternative | Reason rejected |
|---|---|
| Daedalus Echo as the primary base (phone-centric AI) | English-only STT, no timestamps or diarization, phone becomes the system of record, no watch code |
| iDictate as the watch base | No audio recording (uses `SpeechRecognizer`), cloud APIs, GPL-3.0 |
| Self-hosting the Omi backend to keep the phone code unchanged | Hard dependencies on Firebase, Deepgram, OpenAI, Pinecone, Redis and GCS; removing them is more work than a small API |
| Streaming audio watch→server live | Fragile on BT/Wi-Fi/VPN changes; store-and-forward is already built and battery-tested |
| Elasticsearch / Qdrant / Redis | Postgres covers FTS, vectors and the job queue, so there is one database to run |
| Custom accelerometer gesture recognition | Samsung's native Double Pinch (via MediaSession/notifications) and Universal Gestures come first, per the user story |

## Top risks (verify early)

| # | Risk | Mitigation / verification point |
|---|---|---|
| R1 | Android 14+ blocks starting a **microphone** foreground service from the background, e.g. on a gesture with the screen off | M1/M8 spike. Option 1: keep the service running in an "armed" state (FGS started while the app was in the foreground; a user-visible ongoing notification), so the gesture only flips the internal state. Option 2: rely on the MediaSession media-button exemption. Measure the battery cost of the armed state (mic closed while idle). |
| R2 | Double Pinch may not route to our MediaSession | Spike A→B→C (REUSE_AUDIT §7); the Tile/button fallback keeps M1–M7 unblocked |
| R3 | Dutch/English code-switching accuracy | M4 benchmark with ≥3 real watch recordings: faster-whisper `large-v3` vs `large-v3-turbo` vs whisper.cpp. Report the WER on a hand-corrected 5-minute sample. |
| R4 | Server hardware (no GPU → slow large-v3) | Tested in M4; fall back to `medium`/turbo int8 or whisper.cpp; jobs are asynchronous anyway |
| R5 | Small local LLM hallucinating memories | Strict JSON schema, segment-ID validation, confidence threshold, and memories shown as "candidates" |
| R6 | Data Layer message size/throughput for long recordings (1 h ≈ 11 MB at 24 kbps) | Already chunked ≤100 KB; switch to `ChannelClient` streaming if needed (the path constant already exists) |

## Repository placement

This decision was produced on the branch assigned to this session, inside
`mmeurs/Keyestudio-4WD-Mecanum-Robot` (an unrelated MakeCode robot project). **Recommendation:** create
a dedicated repository (e.g. `personal-memory`) with this layout before Milestone 1, and move these two
documents there:

```
android/        fork of omi4WearOS (:shared :wear :mobile), LICENSE + NOTICE retained
server/         FastAPI api + worker, Dockerfile, tests/
docker-compose.yml  .env.example  README.md  THIRD_PARTY_NOTICES.md  docs/
```

## Next step

After the owner approves this ADR:

1. Run the build verification and baseline from REUSE_AUDIT §8 on real hardware.
2. **Milestone 1**: install the upstream `Omi4wOS_Wear_v2.0.apk` unchanged on the Galaxy Watch 7 and record with the screen off.

The server skeleton (Milestones 3–5) can be developed in parallel because it does not need the watch.
