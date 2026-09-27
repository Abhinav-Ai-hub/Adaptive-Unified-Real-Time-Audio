# AURA — Adaptive Unified Real-Time Audio

**QoS, Latency Analytics & Intelligent Audio Streaming**

AURA is a miniature, production-style real-time audio streaming system, inspired by the engineering problems found in cloud gaming platforms: capturing audio, streaming it under imperfect network conditions, measuring what happens along the way, and adapting the pipeline in response.

This document is the standing vision/reference for the full project — architecture, feature set, and how the pieces fit together. Individual modules (like AudioQoS, the ML degradation predictor) have their own detailed roadmaps and documentation.

---

## Core Pipeline

```
Capture → Encode → Stream → Network → Decode → Render → Measure → Analyze → Adapt
```

At a system level, this looks like:

```
                         AURA
       Adaptive Unified Real-Time Audio Engine

 ┌─────────────────────────────────────────────┐
 │                  SENDER                      │
 │                                               │
 │ Microphone / WAV                             │
 │       ↓                                       │
 │ C++ Audio Capture                            │
 │       ↓                                       │
 │ Pre-processing / DSP                         │
 │       ↓                                       │
 │ Encoder                                       │
 │       ↓                                       │
 │ WebRTC / Streaming                           │
 └─────────────────────┬─────────────────────────┘
                       │
                       │ Network
                       │
        ┌──────────────┴──────────────┐
        │                             │
     Packet Loss                   Jitter
     Delay                         Bandwidth
     RTT                           Congestion
        │                             │
        └──────────────┬──────────────┘
                       │
 ┌─────────────────────┴─────────────────────────┐
 │                  RECEIVER                      │
 │                                                 │
 │ Network Receive                                │
 │       ↓                                         │
 │ Jitter Buffer                                  │
 │       ↓                                         │
 │ Decoder                                        │
 │       ↓                                         │
 │ Audio Renderer                                 │
 │       ↓                                         │
 │ Speaker                                        │
 └─────────────────────┬─────────────────────────┘
                       │
                       ↓
                TELEMETRY ENGINE
                       │
          ┌────────────┼────────────┐
          ↓            ↓            ↓
       Audio        Network       System
       Metrics      Metrics       Metrics
          │            │            │
          └────────────┼────────────┘
                       ↓
                Elasticsearch
                       ↓
                    Kibana
                       ↓
               ML Quality Model
                       ↓
              Adaptive Controller
                       │
                       └──────→ Audio Pipeline
```

The system is a closed loop: telemetry feeds the ML model, the model's predictions feed an adaptive controller, and the controller adjusts the streaming pipeline itself (e.g., dropping bitrate, resizing jitter buffers) — closing the loop back to the top.

---

## Final Feature Set

### Core Audio
- Audio capture
- Audio rendering
- PCM processing
- Buffer management
- Audio pipeline
- Stereo audio
- Basic DSP

### Streaming
- Sender/receiver architecture
- WebRTC-based streaming
- Encoding/decoding
- Jitter buffer

### Network Simulation
- Packet loss
- Jitter
- Latency
- Bandwidth limitation
- Variable network conditions

### Telemetry
- Capture timestamp
- Encoding timestamp
- Transmission timestamp
- Receive timestamp
- Decoding timestamp
- Rendering timestamp

### QoS
- Packet loss
- Jitter
- RTT
- Bitrate
- Throughput
- Transmission delay
- Buffer health

### Visualization
- Kibana dashboards
- Latency breakdown
- QoS graphs
- Degradation events

### ML
- Audio-quality prediction
- Degradation classification
- Network-condition → quality prediction

### Adaptive System
- Detect bad network
- Predict degradation
- Modify streaming parameters
- Observe whether quality improves

### Advanced (stretch goals)
- Stereo → surround channel mapping
- 5.1 simulation
- Cross-platform abstraction

---

## Build Philosophy

This is an ambitious, full-stack systems project spanning C++ audio engineering, real-time networking, observability infrastructure, and ML. It's planned over **10 weeks**, with a working MVP at every stage rather than a single big-bang finish — each week should leave something demoable, even if advanced features are still pending.

## Module Map

| Module | What it covers | Status |
|---|---|---|
| **AudioQoS** | Standalone ML module — predicts audio degradation (classification) and a continuous quality/latency score (regression) from network/system telemetry | In progress — see `AudioQoS/methodology.md` and its 10-day roadmap |
| Capture/Encode/Stream (sender) | C++ audio capture, DSP preprocessing, encoding, WebRTC streaming | Planned |
| Network/Receiver | Network simulation, jitter buffer, decoding, rendering | Planned |
| Telemetry Engine | Per-stage timestamps, QoS metric collection, Elasticsearch ingestion | Planned |
| Visualization | Kibana dashboards for latency, QoS, degradation events | Planned |
| Adaptive Controller | Consumes ML predictions, adjusts pipeline parameters, closes the feedback loop | Planned |

AudioQoS is being built first and independently, since it's a self-contained, demonstrable ML problem that doesn't require the rest of the streaming infrastructure to exist yet — it becomes AURA's ML layer once the sender/receiver pipeline is in place.

---

## Why This Structure

Framing AudioQoS as a standalone module first (rather than building the full pipeline before touching ML) means there's a concrete, evaluable deliverable early, and a natural, focused way to get feedback from an experienced engineer before the scope grows into the full system.
