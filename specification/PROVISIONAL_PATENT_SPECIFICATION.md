# PROVISIONAL PATENT APPLICATION

**Title of Invention**

Local-First Voice Synthesis System with Modular Speaker Analysis, Configurable Prosody Presets, and Backpressure-Aware Request Handling

**Inventor**

Lewis James Seals Jr.  
(Residence: [To be completed by inventor])

**Correspondence Address**

[To be completed by inventor]

**Entity Status**

Micro Entity (certification required under 37 CFR 1.29)

---

## CROSS-REFERENCE TO RELATED APPLICATIONS

None.

---

## FIELD OF THE INVENTION

The present invention relates generally to voice synthesis systems, and more particularly to local-first (on-device or local-network) text-to-speech systems that employ neural vocoders or multilingual multi-speaker models, optional modular speaker-embedding analysis, configurable delivery-style or prosody presets, and concurrency-control mechanisms that provide explicit backpressure to concurrent synthesis requests while keeping health-check and user-interface endpoints responsive.

---

## BACKGROUND OF THE INVENTION

Existing commercial voice-synthesis platforms are frequently cloud-dependent, requiring continuous network connectivity, transmitting user text and reference audio to remote servers, and exposing users to latency, privacy, and availability risks. Local-first alternatives built on open models such as Coqui XTTS-v2 can mitigate these concerns but introduce practical engineering challenges:

- Model loading and GPU/CPU inference are expensive; naïve eager loading increases startup latency and resident memory.
- Concurrent synthesis requests against a shared model context can produce unbounded queues, resource exhaustion, or degraded audio quality.
- Blocking inference performed on an asynchronous event loop stalls health checks and interactive user interfaces.
- Output sample rates may differ across models or model versions; hard-coding a rate produces incorrect playback speed or pitch.
- Speaker-embedding analysis (e.g., ECAPA-TDNN) is useful for comparison or verification but is not required for every synthesis path and should remain optional.
- Style or emotional delivery control is often absent or poorly calibrated in open models; a lightweight preset mechanism that modulates sampling parameters (temperature, top-k, top-p, speed) provides practical control without claiming calibrated emotion recognition.
- A self-contained browser-based prototyping interface that supports reference-audio upload, explicit consent, style selection, playback, and download improves usability for local development and evaluation.

There is therefore a need for a local-first voice-synthesis architecture that addresses the foregoing issues through lazy model loading, modular optional speaker analysis, configurable prosody presets, serialized request handling with explicit backpressure, threaded blocking inference, dynamic sample-rate detection, and an integrated local “Voice Studio” interface.

---

## SUMMARY OF THE INVENTION

The invention provides a local-first voice synthesis system comprising:

1. A synthesis engine that loads a multilingual multi-speaker text-to-speech model (exemplified by Coqui XTTS-v2) lazily on first use, thereby reducing startup latency and memory footprint when the model is not immediately required.

2. An optional, modular speaker-embedding subsystem (exemplified by SpeechBrain ECAPA-TDNN) that can be enabled or disabled independently of the core synthesis path, permitting speaker analysis without imposing its computational cost on every request.

3. A set of named emotional or delivery-style presets (neutral, confident, empathetic, authoritative, aggressive) that map to concrete sampling and speed parameters applied during synthesis.

4. A concurrency-control mechanism that serializes synthesis requests; when a synthesis is already in progress, additional concurrent requests receive an HTTP 503 response carrying a Retry-After header rather than being enqueued in an unbounded queue.

5. An architecture in which blocking model inference executes in a worker thread (or thread-pool) so that the asynchronous event loop remains free to service health checks and user-interface requests during synthesis.

6. Dynamic detection of the output sample rate declared by the loaded model, used in preference to a static configuration value, thereby improving compatibility across models and versions.

7. A bundled browser-based “Voice Studio” interface served by the same local API process, supporting script entry, language and style selection, explicit voice-permission confirmation, reference-audio upload, in-browser playback, and WAV download.

The system is intended for local or loopback deployment; remote exposure is intentionally disabled by default configuration.

---

## DETAILED DESCRIPTION OF THE INVENTION

### 1. System Architecture Overview

The preferred embodiment is implemented as a Python package (“elevenx”) exposing:

- An engine module (`engine.py`) responsible for configuration loading, lazy model instantiation, reference-audio validation, application of emotion presets, synthesis, and WAV serialization.
- An emotions module (`emotions.py`) defining an enumeration of delivery styles and a frozen dataclass of sampling parameters.
- A speakers module (`speakers.py`) providing reference-audio reading/validation and optional ECAPA embedding computation.
- An API module (`api.py`) exposing a FastAPI application that serves a health endpoint, a synthesis endpoint, and a static browser UI.
- A configuration file (`config.yaml`) that controls sample rates, model names, device selection, preset values, security limits, and feature flags.

A typical request flow is:

1. Client submits text, language, emotion/style, explicit consent flag, and reference audio via HTTP multipart form.
2. API validates consent, text length, emotion value, and reference-audio size.
3. Reference audio is written to a temporary file.
4. A non-blocking acquisition of a synthesis lock is attempted. Failure yields HTTP 503 + Retry-After.
5. Blocking synthesis is delegated to a worker thread via `run_in_threadpool`.
6. The engine lazily loads the TTS model if necessary, validates the reference, applies the selected preset, performs synthesis, normalizes peak amplitude if required, and returns a `SynthesisResult` containing the waveform, sample rate, emotion label, and duration.
7. The result is written as a temporary WAV; a `FileResponse` is returned with a background cleanup task that deletes both temporary files after the response is sent.
8. The synthesis lock is released.

### 2. Lazy-Loaded XTTS-v2 Synthesis Pipeline

The engine holds a private `_tts` attribute initialized to `None`. On the first call to `synthesize`, `_load_tts` imports the Coqui TTS API, instantiates the model named in configuration (default `tts_models/multilingual/multi-dataset/xtts_v2`), moves it to the selected device (auto-detected CUDA or CPU), and caches the instance. Subsequent calls reuse the cached instance. This defers both download of model weights (if not already cached) and the memory allocation until synthesis is actually required.

### 3. Optional ECAPA-TDNN Speaker Embedding Analysis

Configuration flag `models.load_speaker_encoder` (default false) controls whether the SpeechBrain ECAPA-TDNN encoder is ever loaded. When enabled, a `SpeakerEncoder` instance is created on demand and can compute normalized embeddings for comparison or analysis. The embedding path is completely independent of the XTTS conditioning path; XTTS synthesis never requires the encoder. Reference audio is validated (duration 3–30 s, finite samples, optional resampling to 16 kHz) in both paths.

### 4. Configurable Emotional / Prosody Presets

An `Emotion` enumeration defines five named styles. Each maps, via configuration, to an `EmotionPreset` containing:

- temperature (0.1–2.0)
- top_k (positive integer)
- top_p (0, 1]
- speed (0.5–2.0)

These parameters are passed directly to the underlying TTS call. The presets are intentionally described as style hints rather than calibrated emotion classifiers. Default values are supplied in `config.yaml` and may be overridden without code changes.

### 5. Serialized Request Handling with Backpressure

A process-wide `threading.Lock` (`_synthesis_slot`) guards synthesis. Acquisition is non-blocking (`acquire(blocking=False)`). If the lock is already held, the API returns HTTP 503 with body text explaining that synthesis is in progress and a `Retry-After: 3` header. No queue is maintained. This provides explicit backpressure and prevents unbounded accumulation of GPU/CPU work.

### 6. Threaded Blocking Model Inference

FastAPI’s asynchronous request handlers never call the blocking TTS inference directly. Instead they use `fastapi.concurrency.run_in_threadpool` (or equivalent) so that the event loop continues to process `/health` and static-file requests while synthesis runs on a worker thread. Temporary files are cleaned up via Starlette `BackgroundTask` after the response is delivered.

### 7. Dynamic Output Sample Rate Detection

After the model is loaded, the engine inspects candidate attributes on the model object (synthesizer output sample rate, TTS configuration audio sample rate, etc.). When a positive integer rate is found, it replaces the configured default. This avoids mis-labeling the waveform, which would otherwise alter perceived playback speed and pitch and can trigger downstream audio-queue errors.

### 8. Bundled Browser “Voice Studio” Interface

The same FastAPI process serves a static HTML page at the root path. The page provides:

- Text-area for script entry
- Language and delivery-style selectors
- Explicit consent checkbox (required before synthesis)
- Reference-audio file upload
- Client-side playback of the returned WAV
- Download of the WAV file

The service is bound by default to 127.0.0.1 and remote API exposure is disabled by configuration, reinforcing the local-first design.

### 9. Minimum Working Configuration

The absolute minimum set of components required for synthesis is:

- The XTTS-v2 model (or compatible multi-speaker TTS model)
- A reference audio file satisfying duration and format constraints
- A configuration file supplying at least sample-rate, model name, and one emotion preset
- The engine module capable of lazy loading and synthesis

Optional components (ECAPA encoder, browser UI, FastAPI server, HarmoniaCodec path) may be omitted without breaking core synthesis.

### 10. Variations and Alternatives

- Alternative TTS backends that expose a similar `tts(text, speaker_wav, language, …)` interface may be substituted.
- Alternative speaker encoders (or none) may replace ECAPA-TDNN.
- Additional or differently named presets may be added via configuration.
- Device selection may be forced to CPU or a specific GPU index.
- Output may be streamed in future embodiments; the present embodiment returns a complete waveform.

### 11. Specific Parameters (Exemplary)

- Default sample rate: 24 000 Hz (overridable by model declaration)
- Reference duration: 3.0–30.0 seconds
- Maximum text length: 5 000 characters
- Maximum reference upload: 25 MiB
- Retry-After hint: 3 seconds
- Emotion preset ranges as listed above

These values are illustrative; the claims are not limited to the exact numeric constants.

---

## ABSTRACT

A local-first voice synthesis system loads a multi-speaker neural TTS model lazily, optionally computes independent speaker embeddings, applies named prosody/style presets that modulate sampling parameters, serializes concurrent synthesis requests with explicit HTTP 503/Retry-After backpressure, executes blocking inference on a worker thread to keep the event loop responsive, detects the model’s declared output sample rate dynamically, and exposes a browser-based Voice Studio for script entry, consent, reference upload, playback, and download. The architecture prioritizes privacy, predictable resource use, and developer usability for on-device or local-network deployment.

---

## CLAIMS (INFORMATIONAL – NOT REQUIRED FOR PROVISIONAL)

*Note: Claims are optional in a provisional application. The following are provided for clarity and may be refined in a subsequent non-provisional filing.*

1. A computer-implemented method for local voice synthesis comprising:
   - receiving a synthesis request containing text and a reference audio recording;
   - lazily loading a multi-speaker text-to-speech model only upon first synthesis request;
   - applying a named style preset that maps to temperature, top-k, top-p and speed parameters;
   - generating a waveform using the model conditioned on the reference audio;
   - determining an output sample rate from an attribute exposed by the loaded model when available; and
   - returning the waveform as a local audio file.

2. The method of claim 1 further comprising rejecting concurrent synthesis requests with an HTTP 503 status and Retry-After header when a synthesis is already in progress.

3. The method of claim 1 further comprising executing the model inference on a worker thread so that an asynchronous event loop remains free to service health and user-interface requests.

4. The method of claim 1 further comprising optionally computing an independent speaker embedding with an ECAPA-TDNN encoder that is not required for the synthesis path.

5. A system comprising a processor and memory storing instructions that, when executed, perform the method of any preceding claim, together with a local web interface supporting script entry, style selection, consent confirmation, reference upload, playback and download.

---

**END OF SPECIFICATION**

*This document is intended as an educational template for a United States provisional patent application. It is not legal advice. The inventor should review all statements for accuracy, complete personal information, and consider consultation with a registered patent practitioner before filing.*
