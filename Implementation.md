# AURA — 10-Week Implementation Plan

This breaks the full pipeline (`Capture → Encode → Stream → Network → Decode → Render → Measure → Analyze → Adapt`) into a week-by-week build order, with an MVP at the end of every week rather than one big integration at the end.

---

## Week 1 — Core Audio Capture & Local Playback

**Goal:** Get raw audio flowing through a minimal pipeline with no network involved yet.

**Work to do:**
- Set up the C++ project skeleton (build system, dependency management via CMake + vcpkg/conan).
- Implement audio capture from microphone or WAV file input.
- Implement local playback (speaker output).
- Build a basic circular/ring buffer for holding PCM samples between capture and playback.

**Required tasks:**
- Choose and integrate a cross-platform audio I/O library — **PortAudio** or **RtAudio** are the standard choices for this kind of project.
- Understand and correctly handle PCM parameters: sample rate (44.1kHz/48kHz), bit depth (16-bit typical), channel count (mono vs stereo).
- Implement a simple loopback: capture → buffer → playback, with no processing in between.
- Add basic logging/timestamps at capture and playback so you have a baseline for the telemetry work later.

**Deliverable:** A working local audio loopback — you speak into the mic, it plays back through the speaker with minimal delay, no network, no encoding yet.

---

## Week 2 — DSP & Audio Encoding

**Goal:** Add basic signal processing and get an encode/decode roundtrip working locally.

**Work to do:**
- Add simple DSP stages: gain control, a basic noise gate.
- Integrate an audio codec for encoding/decoding.
- Extend the pipeline: capture → DSP → encode → decode → playback (still local, no network).

**Required tasks:**
- Use **Opus** as the codec — it's the standard for real-time audio in WebRTC, has excellent low-bitrate/low-latency characteristics, and using it now means no codec swap later when you add WebRTC itself.
- Understand codec fundamentals that will matter for your ML features later: frame size (typically 20ms, matching your telemetry buckets), bitrate modes (CBR vs VBR), and encoder complexity settings.
- Verify encode → decode roundtrip doesn't introduce audible artifacts under normal conditions (sanity check before you add network impairment).

**Deliverable:** Local pipeline with working Opus encode/decode roundtrip — audio quality should be indistinguishable from Week 1's raw loopback under ideal conditions.

---

## Week 3 — Sender/Receiver Architecture & WebRTC Streaming

**Goal:** Split the pipeline into two separate endpoints and get audio actually moving over a network transport.

**Work to do:**
- Architect the system as two separate processes: sender and receiver.
- Set up a WebRTC-based (or RTP/UDP-based, if you want to defer full WebRTC complexity) transport between them.
- Get audio streaming end-to-end between the two processes.

**Required tasks:**
- Decide scope here carefully: full **libwebrtc** integration is heavy (it's Google's production C++ library, non-trivial to build and wire up). A pragmatic alternative for an MVP: implement your own minimal RTP-over-UDP sender/receiver first, and treat "swap in real WebRTC" as a later polish task once the rest of the pipeline works. This keeps Week 3 achievable.
- If using real WebRTC: set up peer connection establishment, and a local STUN server (or public STUN) for ICE negotiation, since even loopback testing needs ICE candidates resolved.
- If using custom RTP/UDP: implement RTP-style packetization (sequence numbers, timestamps in the packet header) — you'll need these fields regardless of transport choice, since your telemetry and jitter buffer depend on them.

**Deliverable:** Audio streams from sender to receiver over actual network sockets (even if just localhost initially), with sequence numbers and timestamps embedded in each packet.

---

## Week 4 — Jitter Buffer & Packet-Loss Resilience

**Goal:** Make the receiver resilient to out-of-order and missing packets.

**Work to do:**
- Implement a jitter buffer on the receiver side.
- Handle out-of-order packet arrival using sequence numbers.
- Add basic packet loss concealment (PLC).

**Required tasks:**
- Jitter buffer: a small reorder/delay queue that holds incoming packets briefly, sorts them by sequence number, and releases them to the decoder in order at a steady cadence — this is the mechanism documented conceptually in `methodology.md`, now actually built.
- Make jitter buffer depth configurable (this parameter becomes something your Week 9 adaptive controller will tune).
- PLC: on missing packet, don't just insert silence — repeat the last frame or do simple interpolation, since Opus also has a built-in PLC mode you can leverage instead of writing your own from scratch.
- Instrument buffer underrun events (buffer empties before new data arrives) — you'll need this exact signal for your telemetry engine in Week 6.

**Deliverable:** Receiver plays back cleanly even when packets arrive out of order or are occasionally dropped, without hard crashes or silent stalls.

---

## Week 5 — Network Condition Simulation

**Goal:** Get a controllable way to inject bad network conditions, so you can test (and later train on) realistic degraded scenarios.

**Work to do:**
- Build or integrate a network impairment layer: packet loss, jitter, bandwidth limiting, latency injection.
- Make all impairment parameters configurable per test run.

**Required tasks:**
- On Linux, `tc netem` (traffic control / network emulator) is the standard tool for this — it can inject loss, delay, jitter, and rate limiting at the OS network-interface level without touching your application code, which is the fastest path to realistic impairment.
- Alternative/complement: build a lightweight software proxy between sender and receiver that deliberately drops/delays/reorders packets according to configured probabilities — useful if you want fine-grained programmatic control for automated test sweeps (this is also how you'd generate more *realistic* labeled data for AudioQoS beyond the synthetic dataset).
- Define a set of standard "test regimes" (good / congested / high-jitter / bandwidth-starved) matching the regimes used in the AudioQoS synthetic dataset — this lets you eventually validate the ML model against your actual pipeline's behavior, not just synthetic data.

**Deliverable:** A configurable impairment tool you can point at the sender-receiver link, with named test regimes you can reproduce on demand.

---

## Week 6 — Telemetry Engine

**Goal:** Instrument every pipeline stage so you can measure exactly what's happening, end to end.

**Work to do:**
- Add timestamps at every pipeline stage: capture, encode, transmit, receive, decode, render.
- Compute derived QoS metrics from raw timestamps and packet metadata: packet loss, jitter, RTT, bitrate, throughput, buffer health.
- Design a structured telemetry record schema.

**Required tasks:**
- Timestamp propagation: each packet needs to carry (or allow lookup of) its capture timestamp, so that end-to-end latency can be reconstructed at the receiver.
- Compute the same feature set defined in `methodology.md` — `loss_rate`, `jitter_mean`, `jitter_p95`, `rtt_mean`, `rtt_p95`, `throughput_variation`, `buffer_occupancy`, `buffer_underflow_count`, `cpu_load` — but now from real pipeline data instead of synthetic generation.
- Emit telemetry as structured JSON records on a fixed window (e.g., every 1–2 seconds of a session), matching the granularity your ML model expects.
- This is the point where your AudioQoS model's input schema and your live pipeline's telemetry output schema need to match exactly — treat this as an explicit integration contract, not an afterthought.

**Deliverable:** A telemetry engine emitting real, structured QoS records for every streaming session, schema-compatible with the AudioQoS model's expected input.

---

## Week 7 — Observability: Elasticsearch + Kibana

**Goal:** Get telemetry into a real observability stack so you can visually inspect session health.

**Work to do:**
- Stand up Elasticsearch (via Docker) and ingest telemetry records into it.
- Build Kibana dashboards for latency breakdown, QoS metrics over time, and degradation events.

**Required tasks:**
- Docker Compose setup for Elasticsearch + Kibana — keep this containerized so it's reproducible and easy to demo.
- Write a small ingestion client (from your telemetry engine) that pushes JSON records into an Elasticsearch index — either directly via the Elasticsearch REST API, or via Logstash/Filebeat if you want a more "production-shaped" ingestion path.
- Design at least three dashboard panels: (1) time-series of packet loss/jitter/RTT per session, (2) a latency breakdown showing time spent at each pipeline stage (capture→encode→transmit→decode→render), (3) a marker/annotation layer showing when degradation events (buffer underruns, PLC triggers) occurred.

**Deliverable:** A live Kibana dashboard you can pull up during a streaming session and watch QoS metrics update in real time.

---

## Week 8 — ML Integration (Plugging in AudioQoS)

**Goal:** Connect your already-built AudioQoS model to the live pipeline's real telemetry.

**Work to do:**
- Feed live telemetry windows into the trained AudioQoS model for real-time inference.
- Validate model predictions against actual observed degradation events from the pipeline.
- Recalibrate/retrain if real telemetry distributions diverge meaningfully from your synthetic dataset.

**Required tasks:**
- Build a small inference service (even a simple Python process reading from the same telemetry stream) that scores each incoming window and emits a degradation prediction alongside the raw telemetry.
- Push predictions into Elasticsearch too, so Kibana can show "predicted degradation" next to "actual degradation events" — this comparison is your validation, and a compelling thing to show Ashish.
- If predictions look poor on real data: this is expected and worth documenting explicitly — synthetic data rarely matches real distributions perfectly, and showing you diagnosed *why* (e.g., real jitter has different statistical structure than your generator assumed) is a stronger engineering story than pretending it worked perfectly first try.

**Deliverable:** Real-time degradation predictions running against live pipeline telemetry, visible on the dashboard, with a documented comparison against actual observed degradation.

---

## Week 9 — Adaptive Controller

**Goal:** Close the loop — use ML predictions to actually change pipeline behavior.

**Work to do:**
- Build a controller that consumes degradation predictions and adjusts pipeline parameters.
- Start with a simple rule-based policy before anything more sophisticated.
- Test adaptive vs. non-adaptive behavior under the same simulated degrading network conditions.

**Required tasks:**
- Define the controller's action space: reduce encoder bitrate, resize the jitter buffer, or request a smaller frame size under sustained predicted degradation.
- Implement a simple threshold-based policy first (e.g., "if predicted degradation probability > 0.7 for 3 consecutive windows, drop bitrate by 20%") — resist over-engineering this into a full reinforcement-learning controller for the MVP.
- Run an A/B-style comparison: same impaired-network test regime, once with the adaptive controller active and once with it disabled, and measure the difference in actual audio quality outcomes (fewer underruns, lower PLC trigger rate, etc.).

**Deliverable:** A working closed-loop adaptive system with a measurable, documented improvement over the static baseline under at least one impaired-network scenario.

---

## Week 10 — Polish, Advanced Features & Final Documentation

**Goal:** Consolidate everything into a coherent, demoable system with complete documentation.

**Work to do:**
- Clean up cross-platform rough edges if time allows.
- Attempt stretch goals (stereo→surround mapping, 5.1 simulation) only if core system is solid.
- Finalize all documentation and prepare a demo walkthrough.

**Required tasks:**
- Consolidate `README.md`, `methodology.md`, and per-module docs into a coherent whole — make sure the module map in the README accurately reflects final status.
- Produce a results section: before/after adaptive controller comparison, AudioQoS model performance (from the earlier 10-day roadmap), and dashboard screenshots.
- Prepare a short, sequenced demo script: start clean → inject network degradation live → show the dashboard reacting → show the adaptive controller kicking in → show quality recovering. This narrative sequence is far more compelling in a walkthrough than describing components individually.

**Deliverable:** A complete, documented, demoable AURA prototype with a clear before/after story for the adaptive ML loop.
