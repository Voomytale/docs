# Voomytale — Mobile Client Architecture (for review & estimation)

> **Purpose of this doc.** A concise, review-ready overview of the Voomytale mobile
> client so an Android lead can sanity-check the approach, poke holes in it, and give a
> rough effort estimate. It is intentionally *not* a deep implementation spec — it's a
> shared starting point for a conversation. Anywhere you disagree, that's exactly the
> feedback I'm looking for.

---

## 1. What Voomytale is (30-second version)

A **voice-first AI companion for kids (~3-7; 0+ for calming music)** — think an offline-capable,
kid-safe, on-device story/quiz/game/"Why?"/calm-music companion. Used anywhere, including in
the car (phone audio through Bluetooth/aux). **Voice-first**: every action is doable by voice;
the on-screen buttons are a convenience layer, never required.

The interesting part for us: **most of the experience runs on-device and offline**. Stories are
pre-generated, human-reviewed skeletons cached on the phone; audio is synthesized on-device;
speech recognition runs on-device; the cloud is used opportunistically (novel "Why?" answers,
sync) but is never required for the core loop.

This is closer to building an **on-device voice assistant** than a typical CRUD app.

---

## 2. Team context & what I'm asking you

- **Me:** architect + backend (.NET). I own the **backend / content factory / cloud sync**.
- **You:** Android lead (if you're up for it). Android is the **first** client platform;
  iOS follows later.
- **What I need from you:** a gut-check on the architecture below, especially the on-device
  ML + audio + Bluetooth parts, and a **rough effort estimate / risk list**. Tell me where I'm
  being naive.

---

## 3. High-level shape: shared core + native shells

```
        ┌─────────────────────────────────────────────┐
        │        Shared Core (Kotlin Multiplatform)    │
        │  orchestrator · continuity · memory/profile  │
        │  content-skeleton engine · cache/bundle mgr  │
        │  sync client · SAFETY POLICY · entity extract│
        └───────────────▲───────────────────▲──────────┘
                        │                   │
         ┌──────────────┴──────┐   ┌────────┴───────────┐
         │  Android Shell      │   │  iOS Shell (later) │
         │  (native, FIRST)    │   │  (native, second)  │
         │  Compose UI         │   │  SwiftUI           │
         │  audio session      │   │  AVAudioSession    │
         │  on-device ML       │   │  CoreML / MLX      │
         │  Bluetooth          │   │  CoreBluetooth     │
         └─────────────────────┘   └────────────────────┘
                        │
                        ▼  (opportunistic)
             .NET Backend / Content Factory / Sync
```

**The idea:** put all the *device-independent logic* in a shared core (Kotlin Multiplatform),
and keep each platform's *I/O* (audio, ML runtimes, Bluetooth, background playback, UI) in a
thin native shell.

- **Shared core = ~60-70% of client code**, pure logic, no platform APIs.
- **Native shell = the hard platform-specific stuff** where you can't afford an abstraction
  layer in the way.

**Why this split (and not one cross-platform framework like Flutter / RN / MAUI):** the client
is dominated by audio-session behavior, on-device ML runtimes, Bluetooth/HFP handling, and
background audio — the exact areas cross-platform abstractions leak. We'd end up writing native
bindings for all of it anyway. So: share the *logic*, go native for the *I/O*.

**Open question for you:**
> Do you agree with KMP for the shared core, or given Android-first would you rather start
> pure-Kotlin Android and extract a shared module later? I lean KMP up front so iOS is cheaper
> later, but you're the one who'd live with the KMP build tooling — your call carries weight here.

---

## 4. Android client — component breakdown

### 4.1 UI layer (small but kid-safe)
- **Jetpack Compose.** A minimal, kid-safe UI: large mode buttons (Story / Quiz / Game / Why? /
  Calm), listening/speaking indicators, a "who's riding?" profile picker.
- Voice-sufficiency is a hard rule: the UI is never *required* — but it exists as a convenience.

### 4.2 Audio session & routing (this is where the dragons are)
- `AudioManager` + `AudioFocusRequest` + `AudioRecord`.
- **Bluetooth topology matters.** In the car the phone pairs with the head unit. We deliberately
  keep audio on **A2DP (stereo media)** and **never activate HFP** (the hands-free profile),
  because activating HFP to grab the car mic downgrades *all* audio to mono ~8-16kHz phone-call
  quality. So: **A2DP for output, never HFP.**
- **Mic input priority:** optional child-worn **BT lapel mic** (best) > **phone built-in mic**.
  The car's built-in mic is deliberately **not** used. **One active input at a time**, with
  hot-reselection when a mic connects/disconnects mid-trip.
- Future: a **companion peripheral toy** (BT speaker + mic) is just another BT peer on this same
  topology — no core changes.

> **This is my #1 risk area and where I most want your read.** How painful is robust A2DP-only /
> HFP-avoidance + multi-BT-peer input selection on real Android devices + real car head units?

### 4.3 On-device ML (three models, running on-device)
| Role | Candidate(s) | Notes |
|---|---|---|
| **STT** (speech-to-text) | whisper.cpp via NNAPI / ONNX Runtime Mobile, or a pediatric STT engine (e.g. SoapBox Labs) | Child speech in car noise is hard; emits a confidence score used for the parent "Co-Pilot" fallback |
| **TTS** (text-to-speech) | Android `TextToSpeech`, or Piper via ONNX Runtime Mobile | Renders the mascot's voice per language, on-device, offline |
| **SLM** (small language model) | MediaPipe LLM Inference, or llama.cpp (Gemma 2 2B / Qwen 2.5 1.5-3B) | Simple offline conversational responses; NOT story generation (stories are pre-authored) |

- Audio pipeline before STT: a **~250Hz high-pass filter** to strip road/engine noise, plus an
  **extended VAD (3-4s)** so young kids aren't cut off, plus an always-on **voice-interrupt
  listener** for barge-in during narration.

> **Open questions for you:**
> - Which on-device SLM runtime would you trust in production on Android — MediaPipe LLM Inference
>   or llama.cpp? Memory/latency footprint concerns on mid-tier devices?
> - Any strong opinion on whisper.cpp vs a commercial pediatric STT SDK for kids' voices?

### 4.4 Background audio & media controls
- Story/music must keep playing with the screen locked → **foreground service + MediaSessionCompat**,
  lock-screen controls, and Bluetooth AVRCP (steering-wheel / head-unit buttons).

### 4.5 Local storage, cache & sync
- **Offline-first.** Content "bundles" (per language × age bracket, all themes) are downloaded
  over Wi-Fi and cached locally (a few MB each — text skeletons; audio is synthesized on-device).
- **Opportunistic sync** of the child profile, interests, and story-progress to the backend
  (keyed by `profile_id`, last-write-wins on conflict).
- Local store: likely SQLite/Room or SQLDelight (SQLDelight if we want it in the KMP shared core).

### 4.6 What lives in the shared core (not the Android shell)
- Runtime orchestration (what to play, for whom), continuity/resume logic, memory/profile model,
  content-skeleton engine + variable injection, cache/bundle management, sync client, entity
  extraction rules, and — importantly — **the safety policy**.
- **Why safety policy is in the shared core:** it's the single source of truth shared with the
  backend content factory. Duplicating it per-platform risks Android and iOS drifting apart on a
  child-safety-critical path. One module, no drift.

---

## 5. Complexity flags (so the estimate is honest)

These are the things that make this more than a weekend app:

1. **Three concurrent on-device ML models** with a real memory/latency budget on mid-tier phones.
2. **Real-time barge-in audio pipeline** (VAD, high-pass filter, interrupt-during-playback).
3. **Bluetooth topology** — A2DP-only, HFP avoidance, multi-peer input selection, hot-reselect.
4. **Background audio** + lock-screen/head-unit controls.
5. **Offline-first + opportunistic sync** with conflict resolution.
6. **Multi-language** — STT/TTS/SLM and content all per-language (incl. lower-resource langs).
7. **Multi-profile data isolation** — one child's data must never leak into a sibling's.
8. **Compliance-by-design** (kids' data) — consent gate, no ad SDKs, deletion propagation.

---

## 6. Suggested phasing (open to your reshaping)

A possible build order for the Android client — this is a strawman, please reshape it:

- **Phase A — Skeleton + playback (offline core).** Bundle download/cache, content-skeleton
  engine, variable injection, on-device TTS playback, minimal Compose UI, background audio.
  *Goal: a cached story plays end-to-end offline with the mascot voice.*
- **Phase B — Voice in.** Audio pipeline (HPF + VAD), on-device STT, voice-interrupt listener,
  barge-in, mode/theme switching by voice. *Goal: fully voice-driven.*
- **Phase C — Bluetooth & car.** A2DP-only routing, HFP avoidance, mic input selection
  (phone/lapel), media controls. *Goal: works well in the car.*
- **Phase D — Memory, profiles, sync.** Entity extraction, profile store, multi-profile
  isolation, cross-session resume, opportunistic cloud sync.
- **Phase E — Cloud "Why?" + SLM.** Local SLM for simple responses; online novel-"Why?" path
  with moderation + graceful deferral.
- **Phase F — Compliance, polish, hardening.** Consent gate, deletion, store submission
  (Google Play "Designed for Families").

> **What I'd love from you:** rough effort per phase (t-shirt sizes are fine — S/M/L/XL), which
> phases you'd sequence differently, and where you smell the biggest risk or the biggest unknown.

---

## 7. Key decisions still open (your input wanted)

1. **Shared core: KMP now, or pure-Kotlin Android first and extract later?**
2. **On-device SLM runtime:** MediaPipe LLM Inference vs llama.cpp?
3. **STT:** whisper.cpp vs a commercial pediatric STT SDK?
4. **Local store:** Room vs SQLDelight (SQLDelight = usable inside the KMP core)?
5. **How hard is the Bluetooth A2DP-only / HFP-avoidance story on real devices + cars,** really?
6. **Minimum Android version / device tier** we should target (drives ML feasibility)?
7. **Rough team + timeline** for an Android MVP given the phasing above?

---

## 8. Reference

The full product/system design (content factory, data models, safety, compliance, correctness
properties, future roadmap) lives in the spec design document:
`voomytale/.kiro/specs/kids-voice-companion/design.md`. This mobile doc is a distilled client
view of it — happy to walk you through the full design if useful.
