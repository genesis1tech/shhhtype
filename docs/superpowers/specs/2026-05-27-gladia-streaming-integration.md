# Gladia Solaria-1 Streaming Integration Spec

**Date:** 2026-05-27
**Status:** Ready to Build
**Goal:** Add real-time streaming transcription via Gladia's Solaria-1 API as a new transcription engine alongside existing Groq (batch) and Local Whisper modes. Eliminates the 3-5 second post-speech delay by showing text as the user speaks.

---

## Background

ShhhType currently uses Groq's Whisper Large V3 Turbo in batch mode: record full audio → wait for VAD silence → upload entire clip → get transcript back → inject text. This creates 3-5 seconds of dead air after the user stops speaking.

Glaido (glaido.com), a competing macOS dictation app, likely uses Gladia's Solaria-1 model which supports true real-time streaming with 103ms optimal / 270ms average partial latency. Text appears while the user is still speaking.

Gladia offers a free tier: 10 hours/month, 1 concurrent live streaming session — more than enough for typical dictation use (a moderate user doing 50 dictations/day at 10s each uses ~4 hrs/month).

### Head-to-Head: Whisper Large V3 Turbo vs Solaria-1

| Metric | Whisper V3 Turbo (Groq) | Solaria-1 (Gladia) |
|---|---|---|
| Streaming support | No (batch only) | Yes (real-time WebSocket) |
| Partial transcript latency | N/A | 103ms optimal, 270ms avg |
| Final transcript latency | ~200ms after full audio received | ~698ms |
| WER (real-world audio) | ~7.75% | ~6% (29% lower than alternatives) |
| Hallucination resistance | Standard Whisper levels | 99.9% reduction vs vanilla Whisper |
| Languages | ~100 | 100+ (42 exclusive) |
| Code-switching | Poor | Native (mid-sentence) |
| Price | $0.04/hr | $0.61/hr (free: 10 hrs/mo) |
| Batch speed (1hr audio) | 8-12 seconds | ~60 seconds |

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│  User presses Cmd+Alt+V                                      │
│                                                               │
│  ┌──────────┐    PCM chunks     ┌──────────────────────┐     │
│  │ Mic      │ ─── 20-50ms ───→ │ Rust Backend          │     │
│  │ Capture  │   audio frames    │                       │     │
│  │ (cpal)   │                   │  ┌─────────────────┐  │     │
│  └──────────┘                   │  │ WebSocket       │  │     │
│                                 │  │ (tungstenite)   │──┼──→ Gladia WSS
│                                 │  └─────────────────┘  │     │
│                                 │         │             │     │
│                                 │    partial/final      │     │
│                                 │    transcripts        │     │
│                                 │         │             │     │
│  ┌──────────┐                   │         ▼             │     │
│  │ Text     │ ← inject text ── │  ┌─────────────────┐  │     │
│  │ Injection│   (final only)   │  │ Transcript      │  │     │
│  │ (a11y/   │                  │  │ Accumulator     │  │     │
│  │ clipboard│                   │  └─────────────────┘  │     │
│  └──────────┘                   └──────────────────────┘     │
│                                                               │
│  ┌───────────────────────────────────────────────────────┐   │
│  │ Menu bar overlay: shows partial text as user speaks    │   │
│  └───────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

### UX Flow Comparison

**Current (Groq batch):**
```
[Hotkey] → [Recording indicator] → [Silence...] → [3-5s wait] → [Text appears all at once]
```

**New (Gladia streaming):**
```
[Hotkey] → [Partial text appears in menu bar overlay as they speak] → [Final text injected into focused app]
```

---

## Gladia Live API Protocol

### Step 1: Initiate Session (REST)

```
POST https://api.gladia.io/v2/live
Headers:
  x-gladia-key: <GLADIA_API_KEY>
  Content-Type: application/json

Body:
{
  "encoding": "wav/pcm",
  "bit_depth": 16,
  "sample_rate": 16000,
  "channels": 1,
  "model": "solaria-1",
  "endpointing": 50,
  "language_config": {
    "languages": [],
    "code_switching": true
  },
  "pre_processing": {
    "audio_enhancer": false,
    "speech_threshold": 0.6
  },
  "messages_config": {
    "receive_partial_transcripts": true,
    "receive_final_transcripts": true,
    "receive_speech_events": true
  }
}

Response (201):
{
  "id": "session-uuid",
  "created_at": "2026-05-27T...",
  "url": "wss://api.gladia.io/v2/live?token=TEMP_TOKEN"
}
```

### Step 2: Stream Audio via WebSocket

**Connect** to the `url` returned in Step 1.

**Send audio chunks** (20-50ms each, ~640-1600 bytes at 16kHz/16bit/mono):

```json
{
  "type": "audio_chunk",
  "data": {
    "chunk": "<base64-encoded PCM bytes>"
  }
}
```

Or send raw binary frames (more efficient, less overhead).

**Receive transcript messages:**

Partial (unstable, updates as more audio arrives):
```json
{
  "type": "transcript",
  "data": {
    "id": "msg-uuid",
    "is_final": false,
    "utterance": {
      "text": "schedule the meet",
      "start": 0.0,
      "end": 1.2,
      "confidence": 0.87
    }
  }
}
```

Final (stable, committed after endpointing silence):
```json
{
  "type": "transcript",
  "data": {
    "id": "msg-uuid",
    "is_final": true,
    "utterance": {
      "text": "Schedule the meeting for Tuesday at 3pm.",
      "start": 0.0,
      "end": 3.8,
      "confidence": 0.95,
      "words": [
        { "word": "Schedule", "start": 0.0, "end": 0.4, "confidence": 0.98 },
        ...
      ]
    }
  }
}
```

**End session:** Send `{ "type": "stop_recording" }` or close the WebSocket.

### Audio Format Requirements

- Encoding: `wav/pcm` (raw PCM, no WAV header on chunks)
- Sample rate: 16000 Hz
- Bit depth: 16-bit signed integer
- Channels: 1 (mono)
- Chunk size: 20-50ms recommended (640-1600 bytes per chunk)
- Max single chunk: 120 seconds

### Free Tier Limits

- 10 hours/month (refreshes automatically)
- 1 concurrent live streaming session
- Max session duration: 3 hours
- No credit card required to sign up

---

## Implementation Plan

### New Rust Crate/Module: `gladia_streaming`

#### Dependencies (add to Cargo.toml)

```toml
[dependencies]
tokio-tungstenite = { version = "0.24", features = ["native-tls"] }
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.12", features = ["json"] }
base64 = "0.22"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

#### Core Types

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize)]
pub struct SessionConfig {
    pub encoding: String,          // "wav/pcm"
    pub bit_depth: u16,            // 16
    pub sample_rate: u32,          // 16000
    pub channels: u8,              // 1
    pub model: String,             // "solaria-1"
    pub endpointing: u32,          // 50 (ms of silence before finalizing)
    pub language_config: LanguageConfig,
    pub messages_config: MessagesConfig,
}

#[derive(Serialize)]
pub struct LanguageConfig {
    pub languages: Vec<String>,    // empty = auto-detect
    pub code_switching: bool,
}

#[derive(Serialize)]
pub struct MessagesConfig {
    pub receive_partial_transcripts: bool,
    pub receive_final_transcripts: bool,
    pub receive_speech_events: bool,
}

#[derive(Deserialize)]
pub struct SessionInitResponse {
    pub id: String,
    pub url: String,               // wss:// URL with embedded token
}

#[derive(Deserialize)]
pub struct TranscriptMessage {
    #[serde(rename = "type")]
    pub msg_type: String,          // "transcript"
    pub data: TranscriptData,
}

#[derive(Deserialize)]
pub struct TranscriptData {
    pub id: String,
    pub is_final: bool,
    pub utterance: Utterance,
}

#[derive(Deserialize)]
pub struct Utterance {
    pub text: String,
    pub start: f64,
    pub end: f64,
    pub confidence: f64,
}
```

#### Session Manager

```rust
use tokio_tungstenite::{connect_async, tungstenite::Message};
use futures_util::{SinkExt, StreamExt};
use tokio::sync::mpsc;

pub struct GladiaSession {
    ws_write: SplitSink<WebSocketStream, Message>,
    session_id: String,
}

impl GladiaSession {
    /// Initiate a new live session with Gladia.
    /// Call once at app launch or when switching to streaming mode.
    /// Returns a GladiaSession + a receiver channel for transcript events.
    pub async fn connect(
        api_key: &str,
    ) -> Result<(Self, mpsc::UnboundedReceiver<TranscriptEvent>)> {

        // 1. POST to initiate session
        let config = SessionConfig {
            encoding: "wav/pcm".into(),
            bit_depth: 16,
            sample_rate: 16000,
            channels: 1,
            model: "solaria-1".into(),
            endpointing: 50,
            language_config: LanguageConfig {
                languages: vec![],
                code_switching: true,
            },
            messages_config: MessagesConfig {
                receive_partial_transcripts: true,
                receive_final_transcripts: true,
                receive_speech_events: true,
            },
        };

        let resp: SessionInitResponse = reqwest::Client::new()
            .post("https://api.gladia.io/v2/live")
            .header("x-gladia-key", api_key)
            .json(&config)
            .send()
            .await?
            .json()
            .await?;

        // 2. Connect WebSocket
        let (ws_stream, _) = connect_async(&resp.url).await?;
        let (ws_write, mut ws_read) = ws_stream.split();

        // 3. Spawn reader task — pushes transcript events to channel
        let (tx, rx) = mpsc::unbounded_channel::<TranscriptEvent>();

        tokio::spawn(async move {
            while let Some(Ok(msg)) = ws_read.next().await {
                if let Message::Text(text) = msg {
                    if let Ok(transcript) = serde_json::from_str::<TranscriptMessage>(&text) {
                        if transcript.msg_type == "transcript" {
                            let event = if transcript.data.is_final {
                                TranscriptEvent::Final(transcript.data.utterance.text)
                            } else {
                                TranscriptEvent::Partial(transcript.data.utterance.text)
                            };
                            let _ = tx.send(event);
                        }
                    }
                }
            }
        });

        Ok((Self { ws_write, session_id: resp.id }, rx))
    }

    /// Send a PCM audio chunk to Gladia.
    /// Call this every 20-50ms while the mic is active.
    pub async fn send_audio(&mut self, pcm_bytes: &[u8]) -> Result<()> {
        let payload = serde_json::json!({
            "type": "audio_chunk",
            "data": { "chunk": base64::engine::general_purpose::STANDARD.encode(pcm_bytes) }
        });
        self.ws_write
            .send(Message::Text(payload.to_string()))
            .await?;
        Ok(())
    }

    /// Signal end of audio stream.
    pub async fn stop(&mut self) -> Result<()> {
        let payload = serde_json::json!({ "type": "stop_recording" });
        self.ws_write
            .send(Message::Text(payload.to_string()))
            .await?;
        Ok(())
    }
}

pub enum TranscriptEvent {
    Partial(String),  // Unstable — display in overlay, replace on next partial
    Final(String),    // Stable — inject into focused app, append to history
}
```

#### Mic Capture Pipeline

Modify existing mic capture to output 16kHz/16bit/mono PCM chunks instead of (or in addition to) recording to a file:

```rust
use cpal::traits::{DeviceTrait, HostTrait, StreamTrait};
use tokio::sync::mpsc;

/// Start mic capture, returning a receiver of raw PCM chunks.
/// Each chunk is ~50ms of 16kHz/16bit/mono audio (1600 bytes).
pub fn start_mic_capture() -> Result<(cpal::Stream, mpsc::UnboundedReceiver<Vec<u8>>)> {
    let host = cpal::default_host();
    let device = host.default_input_device().expect("no input device");

    let config = cpal::StreamConfig {
        channels: 1,
        sample_rate: cpal::SampleRate(16000),
        buffer_size: cpal::BufferSize::Fixed(800), // 50ms at 16kHz
    };

    let (tx, rx) = mpsc::unbounded_channel::<Vec<u8>>();

    let stream = device.build_input_stream(
        &config,
        move |data: &[i16], _: &cpal::InputCallbackInfo| {
            // Convert i16 samples to little-endian bytes
            let bytes: Vec<u8> = data.iter()
                .flat_map(|sample| sample.to_le_bytes())
                .collect();
            let _ = tx.send(bytes);
        },
        |err| eprintln!("mic error: {}", err),
        None,
    )?;

    stream.play()?;
    Ok((stream, rx))
}
```

#### Glue: Hotkey → Mic → Gladia → UI/Injection

```rust
/// Called when user presses Cmd+Alt+V with Gladia mode active.
pub async fn on_dictation_start(session: &mut GladiaSession, app_handle: &AppHandle) {
    let (mic_stream, mut audio_rx) = start_mic_capture().unwrap();

    // Pipe mic chunks to Gladia
    let mut session_clone = session.clone(); // or use Arc<Mutex<>>
    tokio::spawn(async move {
        while let Some(chunk) = audio_rx.recv().await {
            if let Err(e) = session_clone.send_audio(&chunk).await {
                eprintln!("gladia send error: {}", e);
                break;
            }
        }
    });

    // Handle transcript events (in the main dictation loop)
    // transcript_rx comes from GladiaSession::connect()
    while let Some(event) = transcript_rx.recv().await {
        match event {
            TranscriptEvent::Partial(text) => {
                // Update menu bar overlay with partial text
                app_handle.emit_all("partial-transcript", &text).unwrap();
            }
            TranscriptEvent::Final(text) => {
                // Inject final text into focused app
                inject_text_into_focused_app(&text);
                // Update history
                save_to_history(&text);
                // Update menu bar
                app_handle.emit_all("final-transcript", &text).unwrap();
                break; // Or continue for multi-utterance mode
            }
        }
    }
}
```

#### Frontend: Partial Transcript Overlay

In the menu bar popover (Tauri webview), listen for transcript events:

```javascript
import { listen } from '@tauri-apps/api/event';

// Show partial text in real-time as user speaks
listen('partial-transcript', (event) => {
  document.getElementById('transcript-preview').textContent = event.payload;
  document.getElementById('transcript-preview').classList.add('streaming');
});

// Flash confirmation when final text is injected
listen('final-transcript', (event) => {
  document.getElementById('transcript-preview').textContent = event.payload;
  document.getElementById('transcript-preview').classList.remove('streaming');
  document.getElementById('transcript-preview').classList.add('final');

  // Clear after 2s
  setTimeout(() => {
    document.getElementById('transcript-preview').textContent = '';
    document.getElementById('transcript-preview').classList.remove('final');
  }, 2000);
});
```

---

## Settings Changes

### New Settings Fields

```rust
pub enum TranscriptionEngine {
    Streaming,  // Gladia Solaria-1 (default for new users)
    Cloud,      // Groq Whisper Large V3 Turbo (existing)
    Local,      // whisper.cpp (existing)
}

pub struct Settings {
    pub transcription_engine: TranscriptionEngine,
    pub gladia_api_key: Option<String>,
    pub groq_api_key: Option<String>,
    pub gladia_free_hours_used: f64,  // Track monthly usage
    pub gladia_usage_reset_date: String, // ISO date of next reset
    // ... existing settings
}
```

### Settings UI

```
Transcription Engine:
  ● Streaming (Gladia)  — Real-time, text appears as you speak
                           10 free hrs/mo · 4.2 hrs used this month
  ○ Cloud (Groq)        — Fast batch transcription
                           Unlimited · Bring your own free API key
  ○ Local (Whisper)     — Fully offline, on-device
                           Unlimited · No internet required

[Gladia API Key: ••••••••••••••••  ] [Get free key →]
[Groq API Key:   ••••••••••••••••  ] [Get free key →]
```

### Free Hour Tracking

```rust
impl GladiaUsageTracker {
    /// Call when a streaming session ends. duration_secs = session length.
    pub fn record_usage(&mut self, duration_secs: f64) {
        self.hours_used += duration_secs / 3600.0;
        self.save();
    }

    /// Check if free tier has hours remaining.
    pub fn has_free_hours(&self) -> bool {
        self.hours_used < 10.0
    }

    /// Hours remaining this month.
    pub fn hours_remaining(&self) -> f64 {
        (10.0 - self.hours_used).max(0.0)
    }

    /// Reset on the 1st of each month.
    pub fn check_reset(&mut self) {
        let now = chrono::Utc::now();
        if now.day() == 1 && self.last_reset_month != now.month() {
            self.hours_used = 0.0;
            self.last_reset_month = now.month();
            self.save();
        }
    }
}
```

### Auto-Fallback to Groq

```rust
async fn transcribe(audio: &AudioInput, settings: &Settings) -> Result<String> {
    match settings.transcription_engine {
        TranscriptionEngine::Streaming => {
            if usage_tracker.has_free_hours() {
                gladia_stream_transcribe(audio, &settings.gladia_api_key).await
            } else {
                // Notify user: "Gladia free hours exhausted, falling back to Groq"
                notify_user("Free streaming hours used up this month. Using Groq batch mode.");
                groq_batch_transcribe(audio, &settings.groq_api_key).await
            }
        }
        TranscriptionEngine::Cloud => {
            groq_batch_transcribe(audio, &settings.groq_api_key).await
        }
        TranscriptionEngine::Local => {
            local_whisper_transcribe(audio).await
        }
    }
}
```

---

## Session Lifecycle Management

### Keep-Alive Strategy

Don't create a new Gladia session per dictation — keep one persistent session:

```
App launch (if Gladia mode selected)
  → POST /v2/live → get WSS URL → connect WebSocket
  → Keep alive (Gladia sessions last up to 3 hours)

On hotkey press:
  → Start mic capture → pipe chunks to existing WebSocket
  → Receive partial/final transcripts

On silence (endpointing):
  → Gladia sends final transcript → inject → stop mic
  → WebSocket stays connected for next dictation

On session timeout (3 hours) or WebSocket drop:
  → Reconnect: POST /v2/live → new WebSocket

On app quit:
  → Send stop_recording → close WebSocket
```

### Reconnection Logic

```rust
impl GladiaSession {
    pub async fn ensure_connected(&mut self, api_key: &str) -> Result<()> {
        if self.is_connected() {
            return Ok(());
        }
        // Reconnect with exponential backoff
        let mut delay = Duration::from_millis(500);
        for attempt in 0..4 {
            match Self::connect(api_key).await {
                Ok((new_session, new_rx)) => {
                    *self = new_session;
                    return Ok(());
                }
                Err(e) => {
                    eprintln!("gladia reconnect attempt {}: {}", attempt, e);
                    tokio::time::sleep(delay).await;
                    delay *= 2;
                }
            }
        }
        Err(anyhow!("Failed to reconnect to Gladia after 4 attempts"))
    }
}
```

---

## What Changes, What Doesn't

| Component | Changes? | Details |
|---|---|---|
| Hotkey system (Cmd+Alt+V) | **No** | Same trigger |
| Mic capture | **Minor** | Output 16kHz/16bit/mono PCM chunks (may already do this) |
| Transcription backend | **New module** | `gladia_streaming` alongside existing Groq HTTP + local Whisper |
| VAD / silence detection | **Replaced** (Gladia mode only) | Gladia's server-side endpointing replaces client-side VAD |
| Menu bar UI | **Enhanced** | New partial transcript overlay shows text in real-time |
| Text injection | **No** | Same clipboard/accessibility injection, triggered by final transcript |
| AI Rewrite (Cmd+Alt+R) | **No** | Still works on the final committed text |
| Composition buffer | **No** | Accumulates final transcripts same as before |
| History | **No** | Logs final transcripts |
| Custom dictionary | **Gladia has this** | Can pass `custom_vocabulary` in session config |
| Settings UI | **Enhanced** | Engine selector + Gladia API key + usage tracking |
| Onboarding | **New** | Prompt to get free Gladia API key (similar to Groq setup) |

---

## Estimated Effort

| Task | Days |
|---|---|
| `gladia_streaming` module (session init, WebSocket lifecycle, types) | 2-3 |
| Mic capture → PCM chunk pipeline (modify existing capture) | 1-2 |
| Transcript event handler (partial/final routing, text injection) | 1 |
| Menu bar partial transcript overlay (frontend) | 1-2 |
| Settings UI (engine selector, API key input, usage display) | 1 |
| Free hour tracking + Groq auto-fallback | 1 |
| Reconnection logic + error handling | 1 |
| Testing + edge cases (network drops, long sessions, rapid dictation) | 2-3 |
| **Total** | **~10-14 days** |

---

## Testing Checklist

- [ ] Gladia session connects on app launch
- [ ] Partial transcripts appear in overlay within ~300ms of speech
- [ ] Final transcript injects correctly into: text editors, email clients, Slack, VS Code, browsers
- [ ] AI Rewrite works on Gladia final transcripts
- [ ] Composition buffer accumulates across multiple Gladia dictations
- [ ] Session reconnects after WebSocket drop
- [ ] Session reconnects after 3-hour timeout
- [ ] Fallback to Groq when free hours exhausted
- [ ] Usage tracker resets monthly
- [ ] Settings switch between Streaming/Cloud/Local works without restart
- [ ] History logs Gladia transcripts correctly
- [ ] Custom dictionary terms passed to Gladia session config
- [ ] Works on spotty WiFi (chunks buffer and retry)
- [ ] Memory usage stable over extended streaming sessions
- [ ] Multiple rapid dictations (hotkey spam) don't create duplicate sessions

---

## References

- [Gladia Live STT Getting Started](https://docs.gladia.io/chapters/live-stt/getting-started)
- [Gladia Session Init API](https://docs.gladia.io/api-reference/v2/live/init)
- [Gladia Live WebSocket API](https://docs.gladia.io/api-reference/v2/live/websocket)
- [Gladia Solaria-1 Product Page](https://www.gladia.io/solaria)
- [Gladia Pricing / Free Tier](https://www.gladia.io/pricing)
- [Gladia Concurrency & Rate Limits](https://docs.gladia.io/chapters/limits-and-specifications/concurrency)
- [tokio-tungstenite (Rust WebSocket)](https://github.com/snapview/tungstenite-rs)
- [Tauri WebSocket Plugin](https://v2.tauri.app/plugin/websocket/)
- [cpal (Rust audio capture)](https://github.com/RustAudio/cpal)
