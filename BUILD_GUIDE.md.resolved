# Real-Time Speech Transcription System - Complete Build Guide

## Problem Statement

**Goal:** Build a web-based real-time speech transcription system that provides:
1. **Live captions** - Text appears while you're speaking (realtime updates)
2. **Final transcriptions** - Accurate sentence-level transcriptions after speech ends
3. **Low latency** - Minimal delay between speech and text
4. **High accuracy** - Matching or exceeding existing solutions (RealtimeSTT)

## Architecture Overview

### Component Flow
```
┌─────────────────────────────────────────────────────────────────┐
│                         Browser (Frontend)                       │
│  ┌────────────┐   ┌──────────────┐   ┌─────────────────────┐  │
│  │ Microphone │ → │ AudioWorklet │ → │ WebSocket (PCM16)   │  │
│  └────────────┘   └──────────────┘   └─────────────────────┘  │
└───────────────────────────────┬─────────────────────────────────┘
                                │ PCM16 audio chunks
                                ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Backend (FastAPI/Python)                    │
│  ┌──────────────────┐                                           │
│  │ WebSocket Handler│                                           │
│  │  - Receive audio │                                           │
│  │  - Accumulate    │                                           │
│  │    raw_frames[]  │ ← KEY: Raw PCM16 buffer                  │
│  └────────┬─────────┘                                           │
│           │                                                      │
│           ├─→ [AudioProcessingWorker]                           │
│           │       │                                              │
│           │       ├─→ [SpeechDetector (VAD)]                    │
│           │       │      - Silero VAD                           │
│           │       │      - WebRTC VAD                           │
│           │       │      - Callbacks: on_speech_start/end       │
│           │       │                                              │
│           ├─→ [Realtime Worker]                                 │
│           │       │                                              │
│           │       ├─→ Reads raw_frames[]                        │
│           │       ├─→ Normalizes to -0.95 dBFS                  │
│           │       ├─→ WhisperASR (tiny.en, VAD=True)            │
│           │       └─→ Commonprefix stabilization                │
│           │                                                      │
│           └─→ [Final Transcription]                             │
│                   ├─→ When speech_end                           │
│                   ├─→ Async (non-blocking)                      │
│                   └─→ WhisperASR (base.en, VAD=True)            │
└─────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step Implementation

### Phase 1: Basic Setup

#### Step 1.1: Project Structure
```bash
transcription_subtitles/
├── backend/
│   ├── app.py                 # FastAPI main
│   ├── asr.py                 # Whisper wrapper
│   ├── speech_detector.py     # VAD
│   ├── audio_worker.py        # Worker thread
│   ├── preprocess.py          # Audio utils
│   ├── metrics.py             # Tracking
│   ├── outputs.py             # File generation
│   ├── util_audio.py          # Conversion
│   ├── schemas.py             # Pydantic models
│   └── requirements.txt
│
└── frontend/
    ├── index.html
    ├── main.js
    ├── style.css
    └── worklet-processor.js
```

#### Step 1.2: Dependencies (requirements.txt)
```txt
fastapi==0.104.0
uvicorn[standard]==0.24.0
websockets==12.0
faster-whisper==0.9.0
numpy==1.24.3
torch==2.1.0
pysilero-vad==1.0.0
webrtcvad==2.0.10
python-multipart==0.0.6
```

---

### Phase 2: Frontend Audio Capture

#### Step 2.1: AudioWorklet Processor (worklet-processor.js)

**Key Pattern:** Small buffer size for low latency

```javascript
class AudioProcessor extends AudioWorkletProcessor {
    constructor() {
        super();
        this.bufferSize = 512;  // 32ms chunks (8x smaller than default!)
        this.buffer = new Float32Array(this.bufferSize);
        this.bufferIndex = 0;
    }
    
    process(inputs, outputs, parameters) {
        const input = inputs[0];
        if (!input || !input[0]) return true;
        
        const inputChannel = input[0];
        
        for (let i = 0; i < inputChannel.length; i++) {
            this.buffer[this.bufferIndex++] = inputChannel[i];
            
            if (this.bufferIndex >= this.bufferSize) {
                // Resample to 16kHz
                const resampled = this.resample(this.buffer, 48000, 16000);
                
                // Convert to PCM16
                const pcm16 = new Int16Array(resampled.length);
                for (let j = 0; j < resampled.length; j++) {
                    pcm16[j] = Math.max(-32768, Math.min(32767, 
                        Math.floor(resampled[j] * 32767)));
                }
                
                // Send to main thread
                this.port.postMessage({
                    audio: pcm16.buffer,
                    sampleRate: 16000
                });
                
                this.bufferIndex = 0;
            }
        }
        
        return true;
    }
}
```

**Critical:** Don't filter out silence in frontend! Send ALL audio.

#### Step 2.2: WebSocket Communication (main.js)

```javascript
// Connect WebSocket
websocket = new WebSocket('ws://localhost:8000/ws/transcribe');

websocket.onopen = async () => {
    // Send configuration
    websocket.send(JSON.stringify({
        type: 'start',
        session_id: 'auto',
        config: {
            model: 'tiny.en',
            language: 'en',
            preprocess: false,
            vad_enabled: true
        }
    }));
    
    // Start audio capture
    await startAudioCapture();
};

// AudioWorklet message handler
audioWorkletNode.port.onmessage = (event) => {
    if (websocket && websocket.readyState === WebSocket.OPEN) {
        // Send PCM16 audio directly
        websocket.send(event.data.audio);
    }
};

// Receive messages
websocket.onmessage = (event) => {
    const data = JSON.parse(event.data);
    
    if (data.type === 'realtime') {
        // Live captions (gray, italic)
        liveCaptions.textContent = data.text;
    } else if (data.type === 'fullSentence') {
        // Final transcription
        addTranscriptSegment(data.text);
    }
};
```

---

### Phase 3: Backend Core Components

#### Step 3.1: ASR Wrapper (asr.py)

**Critical Pattern:** Enable VAD filter (double VAD is intentional!)

```python
class WhisperASR:
    def __init__(self, model_name="base", device="cpu"):
        self.model_name = model_name
        self._model = None
    
    def _ensure_model_loaded(self):
        if self._model is None:
            from faster_whisper import WhisperModel
            self._model = WhisperModel(
                self.model_name,
                device="cpu",
                compute_type="int8"
            )
    
    def transcribe_audio(self, audio, sample_rate=16000, language=None):
        self._ensure_model_loaded()
        
        # Normalize
        if np.abs(audio).max() > 1.0:
            audio = audio / np.abs(audio).max()
        
        # ✅ CRITICAL: VAD filter ENABLED (RealtimeSTT uses this!)
        segments_iter, info = self._model.transcribe(
            audio,
            language=language,
            beam_size=5,
            vad_filter=True,  # Double VAD is intentional!
            vad_parameters={
                "threshold": 0.3,
                "min_speech_duration_ms": 100,
                "min_silence_duration_ms": 300,
            },
            word_timestamps=False
        )
        
        # Collect segments
        segments = []
        full_text = []
        for seg in segments_iter:
            segments.append({
                "start": round(seg.start, 2),
                "end": round(seg.end, 2),
                "text": seg.text.strip()
            })
            full_text.append(seg.text.strip())
        
        return {
            "text": " ".join(full_text),
            "segments": segments,
            "language": info.language
        }
```

#### Step 3.2: Speech Detector (speech_detector.py)

**Critical Pattern:** Dual VAD (Silero + WebRTC)

```python
class SpeechDetector:
    def __init__(self, on_speech_start, on_speech_end, 
                 vad_sensitivity=2, silero_sensitivity=0.4):
        self.on_speech_start = on_speech_start
        self.on_speech_end = on_speech_end
        
        # Dual VAD
        self.webrtc_vad = webrtcvad.Vad(vad_sensitivity)
        self.silero_vad = torch.hub.load(...)
        
        self.state = SpeechState.SILENCE
        self.speech_buffer = []
        self.ring_buffer = collections.deque(maxlen=50000)
        
    def feed_audio(self, audio_float32):
        """Feed audio samples (float32)"""
        self.ring_buffer.extend(audio_float32)
    
    def _processing_loop(self):
        """Background thread for VAD processing"""
        while self.is_running:
            # Extract frames from ring_buffer
            if len(self.ring_buffer) >= 480:  # 30ms @ 16kHz
                frame = list(itertools.islice(self.ring_buffer, 480))
                
                # Dual VAD check
                webrtc_speech = self._webrtc_vad_check(frame)
                silero_speech = self._silero_vad_check(frame)
                
                is_speech = webrtc_speech or silero_speech
                
                if is_speech and self.state == SpeechState.SILENCE:
                    self.state = SpeechState.SPEAKING
                    self.speech_buffer = []
                    self.on_speech_start()
                
                if self.state == SpeechState.SPEAKING:
                    self.speech_buffer.extend(frame)
                
                # Check for silence to end speech
                if not is_speech and self.state == SpeechState.SPEAKING:
                    if self.silence_duration > self.post_speech_silence:
                        audio = np.array(self.speech_buffer, dtype=np.float32)
                        self.on_speech_end(audio)
                        self.state = SpeechState.SILENCE
    
    def get_current_audio(self):
        """For realtime transcription"""
        with self.lock:
            if self.state == SpeechState.SPEAKING:
                return np.array(self.speech_buffer, dtype=np.float32)
        return None
```

#### Step 3.3: Audio Worker (audio_worker.py)

**Pattern:** Queue-based, non-blocking

```python
class AudioProcessingWorker:
    def __init__(self, speech_detector):
        self.speech_detector = speech_detector
        self.audio_queue = queue.Queue(maxsize=1000)
        self.is_running = False
        self.worker_thread = None
    
    def start(self):
        self.is_running = True
        self.worker_thread = threading.Thread(
            target=self._process_loop,
            daemon=True
        )
        self.worker_thread.start()
    
    def feed_audio(self, audio):
        """Non-blocking!"""
        try:
            self.audio_queue.put_nowait(audio)
        except queue.Full:
            pass  # Drop if full
    
    def _process_loop(self):
        while self.is_running:
            try:
                audio = self.audio_queue.get(timeout=0.01)
                self.speech_detector.feed_audio(audio)
            except queue.Empty:
                continue
```

---

### Phase 4: WebSocket Handler (app.py)

**THE MOST CRITICAL PART** - This is where everything comes together

```python
@app.websocket("/ws/transcribe")
async def websocket_transcribe(websocket: WebSocket):
    await websocket.accept()
    
    # Session state
    is_speaking = False
    is_running = False
    awaiting_speech_end = False
    
    # ✅ CRITICAL: Raw audio frames buffer (like RealtimeSTT!)
    raw_frames = []
    
    # Text stabilization
    text_storage = []
    
    main_loop = asyncio.get_running_loop()
    
    # Callbacks
    def on_speech_start():
        nonlocal is_speaking, raw_frames, awaiting_speech_end
        is_speaking = True
        awaiting_speech_end = False
        raw_frames = []  # Clear for new speech
    
    def on_speech_end(audio):
        nonlocal is_speaking, awaiting_speech_end
        is_speaking = False
        awaiting_speech_end = True  # Skip realtime during final
        
        # Schedule async transcription
        asyncio.run_coroutine_threadsafe(
            transcribe_final_audio(audio),
            main_loop
        )
    
    # ✅ Realtime transcription worker
    async def realtime_transcription_worker():
        while is_running:
            if not is_speaking:
                await asyncio.sleep(0.1)
                continue
            
            await asyncio.sleep(0.2)  # 200ms updates
            
            # Skip if finalizing
            if awaiting_speech_end:
                continue
            
            # ✅ KEY: Use raw_frames, not speech_detector buffer!
            if len(raw_frames) == 0:
                continue
            
            # Join PCM16 chunks
            audio_bytes = b''.join(raw_frames)
            if len(audio_bytes) < 6400:  # ~200ms
                continue
            
            # Convert to float32
            audio_int16 = np.frombuffer(audio_bytes, dtype=np.int16)
            audio_float = audio_int16.astype(np.float32) / 32768.0
            
            # ✅ CRITICAL: Normalize to -0.95 dBFS
            peak = np.max(np.abs(audio_float))
            if peak > 0:
                audio_float = (audio_float / peak) * 0.95
            
            # Transcribe (non-blocking)
            result = await asyncio.get_event_loop().run_in_executor(
                None,
                lambda: asr.transcribe_audio(audio_float, sample_rate=16000)
            )
            
            partial_text = result.get("text", "").strip()
            
            # ✅ Commonprefix stabilization
            text_storage.append(partial_text)
            if len(text_storage) >= 2:
                import os
                stable_text = os.path.commonprefix([
                    text_storage[-2], text_storage[-1]
                ]).strip()
            else:
                stable_text = partial_text
            
            if stable_text:
                await websocket.send_json({
                    "type": "realtime",
                    "text": stable_text
                })
    
    # Final transcription
    async def transcribe_final_audio(audio):
        nonlocal awaiting_speech_end
        
        result = await asyncio.get_event_loop().run_in_executor(
            None,
            lambda: asr.transcribe_audio(audio, sample_rate=16000)
        )
        
        text = result.get("text", "").strip()
        if text:
            await websocket.send_json({
                "type": "fullSentence",
                "text": text
            })
        
        awaiting_speech_end = False
    
    # Initialize components
    speech_detector = SpeechDetector(
        on_speech_start=on_speech_start,
        on_speech_end=on_speech_end
    )
    speech_detector.start()
    
    audio_worker = AudioProcessingWorker(speech_detector)
    audio_worker.start()
    
    is_running = True
    
    # Start realtime worker
    realtime_task = asyncio.create_task(realtime_transcription_worker())
    
    # Main loop
    while is_running:
        try:
            message = await asyncio.wait_for(websocket.receive(), timeout=0.01)
            
            if "bytes" in message:
                pcm_bytes = message["bytes"]
                
                # ✅ CRITICAL: Accumulate raw frames!
                if is_speaking:
                    raw_frames.append(pcm_bytes)
                
                # Convert and feed to worker
                audio_float = pcm16_to_float32(pcm_bytes)
                audio_worker.feed_audio(audio_float)
        
        except asyncio.TimeoutError:
            continue
        except WebSocketDisconnect:
            break
    
    # Cleanup
    audio_worker.stop()
    speech_detector.stop()
```

---

## Critical Patterns from RealtimeSTT

### 1. Raw Frames Buffer
**Problem:** We were transcribing speech_detector's processed buffer  
**Solution:** Accumulate raw PCM16 in `raw_frames[]` and transcribe THAT

### 2. Double VAD
**Problem:** Thought VAD in ASR was duplicate  
**Solution:** RealtimeSTT uses BOTH speech_detector VAD + ASR VAD intentionally

### 3. Audio Normalization
**Problem:** Transcriptions were empty  
**Solution:** Normalize to -0.95 dBFS before transcription

### 4. awaiting_speech_end Flag
**Problem:** Hallucinations during silence  
**Solution:** Skip realtime transcription while finalizing

### 5. Commonprefix Stabilization
**Problem:** Flickering text  
**Solution:** Use `os.path.commonprefix()` of last 2 texts

---

## Testing Checklist

1. ✅ Realtime text appears while speaking (gray/italic)
2. ✅ Final transcription after speech ends
3. ✅ No hallucinations during silence
4. ✅ Text doesn't flicker
5. ✅ Low latency (~200-500ms)
6. ✅ No server hanging/blocking

---

## Common Pitfalls

1. ❌ Filtering silence in frontend → Don't! Send all audio
2. ❌ Using speech_detector buffer → Use raw_frames
3. ❌ Disabling ASR VAD → Keep it ON
4. ❌ Not normalizing audio → Always normalize to -0.95 dBFS
5. ❌ Blocking on_speech_end → Make it async
6. ❌ Missing awaiting_speech_end → Causes hallucinations

---

## Performance Tuning

- **Model choice:** `tiny.en` (fast) vs `base.en` (accurate)
- **Buffer size:** 512 samples = 32ms latency
- **Update frequency:** 200ms for realtime
- **VAD sensitivity:** Lower = more sensitive
