# Florilegium of Potential Inventions in ElevenX

A florilegium is a curated collection of excerpts. The following technical elements are documented in the ElevenX codebase and accompanying analysis. They are presented as candidate inventive concepts that could be framed for patent protection; they are **not** formal claims and have not been examined for novelty or non-obviousness.

---

## 1. Lazy-Loaded XTTS-v2 Synthesis Pipeline

A method for synthesizing speech using a reference recording wherein the XTTS-v2 (or compatible multi-speaker) model is loaded lazily on first synthesis request, thereby reducing startup latency and resident memory footprint when synthesis is not immediately required.

**Location in codebase:** `src/elevenx/engine.py` — `_tts` attribute initialized to `None`; `_load_tts()` instantiates and caches the model only when needed.

---

## 2. Optional ECAPA-TDNN Speaker Embedding Analysis

A modular feature in which SpeechBrain’s ECAPA-TDNN embeddings are provided as a separate analysis capability that is not required for normal synthesis. This permits speaker comparison or verification without imposing the encoder’s computational cost on the core synthesis path.

**Location:** `src/elevenx/speakers.py` + configuration flag `models.load_speaker_encoder` (default `false`).

---

## 3. Configurable Emotional / Prosody Presets

A system for applying named presets (“neutral,” “confident,” “empathetic,” “authoritative,” “aggressive”) that map to concrete sampling parameters (temperature, top-k, top-p, speed) controlling delivery style during synthesis.

**Location:** `src/elevenx/emotions.py` + `config.yaml` → `synthesis.emotion_presets`.

---

## 4. Serialized Request Handling with Backpressure

A concurrency-control mechanism in which synthesis requests are serialized by a process-wide lock; concurrent requests that cannot acquire the lock receive HTTP 503 with a `Retry-After` header instead of being placed in an unbounded queue. This provides explicit backpressure for model-call contention.

**Location:** `src/elevenx/api.py` — `_synthesis_slot` lock, non-blocking acquire, 503 + Retry-After response.

---

## 5. Threaded Blocking Model Inference

An architecture in which blocking model inference executes in a worker thread (via `run_in_threadpool` or equivalent) so that health-check and other API endpoints continue to be served by the asynchronous event loop during synthesis.

**Location:** `src/elevenx/api.py` — `await run_in_threadpool(engine.synthesize, …)`.

---

## 6. Dynamic Output Sample Rate Detection

A method that obtains the output sample rate from attributes exposed by the loaded model (when available) rather than blindly assuming a statically configured rate, thereby improving compatibility across models and versions and avoiding incorrect playback speed/pitch.

**Location:** `src/elevenx/engine.py` — inspection of synthesizer / TTS-config sample-rate candidates after model load.

---

## 7. Bundled Browser “Voice Studio” Interface

A local API that serves a browser-based UI supporting script entry, language and delivery-style selection, explicit voice-permission confirmation, reference-audio upload, in-browser playback, and WAV download—forming a self-contained human-computer interaction design for local voice prototyping.

**Location:** `src/elevenx/api.py` (root route) + `src/elevenx/static/index.html` + related studio helpers.

---

*These elements are derived from the public ElevenX repository (https://github.com/ljrseals-dev/ElevenX) and the accompanying technical analysis. They are offered solely as a documentation aid for provisional-application drafting.*
