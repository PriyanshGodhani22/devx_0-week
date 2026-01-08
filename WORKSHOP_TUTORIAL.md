# Build a Real-Time Speech Transcription System 🎙️
## Interactive Workshop Tutorial

> **Duration:** 2-3 hours  
> **Difficulty:** Intermediate  
> **Prerequisites:** Python basics, JavaScript basics, understanding of async programming

---

## 🎯 What We're Building

A web application that transcribes your speech in real-time as you speak, similar to live captions on YouTube or Zoom.

**Features:**
- ✅ Live captions appear while you're speaking
- ✅ Final accurate transcriptions after you finish
- ✅ Works in browser (no app installation)
- ✅ Low latency (~300ms)

**Demo Flow:**
1. Click "Start Recording"
2. Speak: "Hello, how are you today?"
3. See live text appear: "Hello..." → "Hello how..." → "Hello how are you..."
4. Stop speaking
5. Final text: "Hello, how are you today?"

---

## 📋 Workshop Outline

### Part 1: Setup & Frontend (30 min)
- Project structure
- Audio capture from microphone
- WebSocket connection

### Part 2: Backend Core (45 min)
- FastAPI server
- WebSocket handler
- Audio conversion utilities

### Part 3: Speech Detection (30 min)
- Voice Activity Detection (VAD)
- Speech start/end detection
- State management

### Part 4: Transcription (45 min)
- Whisper ASR integration
- Realtime transcription
- Final transcription
- Text stabilization

### Part 5: Integration & Testing (30 min)
- Connect all components
- Debug and test
- Performance tuning

---

## 🛠️ Part 1: Setup & Frontend

### Step 1.1: Create Project Structure

```bash
# Create project directory
mkdir realtime-transcription
cd realtime-transcription

# Create subdirectories
mkdir -p frontend backend
```

**Checkpoint:** You should have this structure:
```
realtime-transcription/
├── frontend/
└── backend/
```

---

### Step 1.2: Frontend HTML Structure

**File:** [frontend/index.html](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/frontend/index.html)

Create a simple UI with:
- Microphone selection dropdown
- Start/Stop buttons
- Live captions area (gray, italic)
- Final transcriptions area (black, permanent)

**What to do:**
1. Create [index.html](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/minimal_ui/index.html) in [frontend/](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/app.py#117-124) folder
2. Add these sections:
   - Header with title
   - Controls (mic select, buttons)
   - Live captions display
   - Transcription history

**Key HTML elements we need:**
```html
<select id="mic-select"></select>
<button id="start-btn">Start Recording</button>
<button id="stop-btn">Stop Recording</button>
<div id="live-captions"></div>  <!-- Real-time text -->
<div id="transcript-area"></div> <!-- Final text -->
```

**Test:** Open [index.html](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/minimal_ui/index.html) in browser, verify all elements are visible.

---

### Step 1.3: Audio Capture - The AudioWorklet

**Why AudioWorklet?**
- Runs in separate thread (doesn't block UI)
- Real-time audio processing
- Low latency

**File:** [frontend/worklet-processor.js](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/frontend/worklet-processor.js)

**What it does:**
1. Receives audio from microphone (48kHz)
2. Buffers it in small chunks (512 samples = ~10ms)
3. Resamples to 16kHz (Whisper requirement)
4. Converts to PCM16 format
5. Sends to main thread

**Implementation Steps:**

1. **Create the processor class:**
```javascript
class AudioProcessor extends AudioWorkletProcessor {
    constructor() {
        super();
        this.bufferSize = 512;  // Small for low latency
        this.buffer = new Float32Array(this.bufferSize);
        this.bufferIndex = 0;
    }
}
```

2. **Implement the process() method:**
   - Accumulate samples
   - When buffer full → resample and send
   - Return `true` to keep processing

3. **Register the processor:**
```javascript
registerProcessor('audio-processor', AudioProcessor);
```

**Checkpoint:** 
- File exists: [frontend/worklet-processor.js](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/frontend/worklet-processor.js)
- Contains AudioProcessor class
- Implements process() method

---

### Step 1.4: WebSocket Connection

**File:** [frontend/main.js](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/frontend/main.js)

**What we'll build:**
```
User clicks "Start" 
    ↓
Connect WebSocket to server
    ↓
Start microphone capture
    ↓
Send audio chunks via WebSocket
    ↓
Receive transcription updates
    ↓
Update UI
```

**Implementation checklist:**

- [ ] **Initialize WebSocket:**
```javascript
websocket = new WebSocket('ws://localhost:8000/ws/transcribe');
```

- [ ] **Handle connection open:**
  - Send start message with config
  - Start audio capture

- [ ] **Handle messages from server:**
  - Type "realtime" → Update live captions
  - Type "fullSentence" → Add to transcript

- [ ] **Send audio data:**
  - Get audio from worklet
  - Send via websocket.send(audioBytes)

**Test at this point:**
- Page loads without errors
- Click "Start" attempts WebSocket connection
- Console shows connection attempt (will fail - server not ready yet)

---

## ⚙️ Part 2: Backend Core

### Step 2.1: Install Dependencies

**File:** [backend/requirements.txt](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/requirements.txt)

```txt
fastapi==0.104.0
uvicorn[standard]==0.24.0
websockets==12.0
faster-whisper==0.9.0
numpy==1.24.3
torch==2.1.0
pysilero-vad==1.0.0
webrtcvad==2.0.10
```

**Install:**
```bash
cd backend
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

**Checkpoint:** Run `python -c "import fastapi; print('OK')"` - should print "OK"

---

### Step 2.2: Audio Conversion Utilities

**Why we need this:**
- Frontend sends PCM16 (Int16)
- Whisper needs Float32
- We need conversion functions

**File:** [backend/util_audio.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/util_audio.py)

**Functions to create:**

1. **`pcm16_to_float32(pcm_bytes)`**
   - Input: Raw bytes from WebSocket
   - Output: Float32 numpy array
   - Conversion: int16 → float32, scale by 32768

2. **`float32_to_pcm16(float_array)`** (optional, for saving)
   - Reverse conversion

**Implementation guide:**
```python
import numpy as np

def pcm16_to_float32(pcm_bytes):
    # Step 1: Convert bytes to int16 array
    int16_array = np.frombuffer(pcm_bytes, dtype=np.int16)
    
    # Step 2: Convert to float32 and normalize to [-1.0, 1.0]
    float32_array = int16_array.astype(np.float32) / 32768.0
    
    return float32_array
```

**Test:**
```python
# Create test PCM16 data
test_data = np.array([0, 16384, -16384], dtype=np.int16).tobytes()
result = pcm16_to_float32(test_data)
# Should get: [0.0, 0.5, -0.5]
```

---

### Step 2.3: Basic FastAPI Server

**File:** [backend/app.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/app.py)

**Start simple:**
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"status": "Server running"}

@app.get("/health")
async def health():
    return {"status": "OK"}
```

**Run server:**
```bash
uvicorn app:app --reload --port 8000
```

**Test:** Open `http://localhost:8000` in browser → Should see JSON response

---

### Step 2.4: WebSocket Endpoint (Basic)

**Add to [app.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/app.py):**

```python
from fastapi import WebSocket, WebSocketDisconnect

@app.websocket("/ws/transcribe")
async def websocket_transcribe(websocket: WebSocket):
    # Accept connection
    await websocket.accept()
    print("Client connected")
    
    try:
        # Main loop
        while True:
            # Receive message
            message = await websocket.receive()
            
            # Handle different message types
            if "text" in message:
                # Control message (start/stop)
                print(f"Received: {message['text']}")
            elif "bytes" in message:
                # Audio data
                print(f"Received {len(message['bytes'])} bytes of audio")
    
    except WebSocketDisconnect:
        print("Client disconnected")
```

**Test:**
1. Start server: `uvicorn app:app --reload --port 8000`
2. Open frontend in browser
3. Click "Start Recording"
4. Check server console - should see "Client connected"
5. Speak into mic - should see "Received X bytes of audio"

**Checkpoint:** WebSocket connection working, receiving audio data ✅

---

## 🎤 Part 3: Speech Detection

### Step 3.1: Understanding VAD

**What is Voice Activity Detection (VAD)?**
- Determines if audio contains speech or silence
- Helps us know when user starts/stops speaking
- Prevents transcribing silence (saves CPU)

**Why Dual VAD?**
- **WebRTC VAD:** Fast, simple, good for clear speech
- **Silero VAD:** ML-based, better for noisy environments
- **Together:** More reliable detection

---

### Step 3.2: Create Speech Detector

**File:** [backend/speech_detector.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/speech_detector.py)

**What it needs to do:**

```
Incoming audio → VAD processing → Detect speech state
    ↓
SILENCE state → SPEAKING state → SILENCE state
    ↓                ↓                  ↓
  (do nothing)  (collect audio)  (trigger transcription)
```

**Class structure:**

```python
class SpeechState:
    SILENCE = "silence"
    SPEAKING = "speaking"

class SpeechDetector:
    def __init__(self, on_speech_start, on_speech_end):
        # Store callbacks
        self.on_speech_start = on_speech_start
        self.on_speech_end = on_speech_end
        
        # Initialize VAD models
        self.webrtc_vad = ...
        self.silero_vad = ...
        
        # State tracking
        self.state = SpeechState.SILENCE
        self.speech_buffer = []
        
    def start(self):
        # Start processing thread
        pass
    
    def feed_audio(self, audio):
        # Add audio to processing queue
        pass
    
    def _processing_loop(self):
        # Background thread: process audio, detect speech
        pass
```

**Implementation Steps:**

1. **Initialize VAD models:**
   - WebRTC: `webrtcvad.Vad(sensitivity)`
   - Silero: Load from torch.hub

2. **Create processing thread:**
   - Runs in background
   - Processes audio from queue
   - Checks VAD on each frame

3. **Implement state machine:**
```
if VAD says "speech" AND state is SILENCE:
    → Change to SPEAKING
    → Call on_speech_start()
    → Start collecting audio

if VAD says "silence" AND state is SPEAKING:
    → Wait for X seconds (post_speech_silence)
    → If still silent, change to SILENCE
    → Call on_speech_end(audio)
```

**Key parameters:**
- `vad_sensitivity`: 2 (0-3, lower = more sensitive)
- `silero_sensitivity`: 0.4 (0-1, lower = more sensitive)
- `post_speech_silence`: 0.7 seconds (silence to end speech)

---

### Step 3.3: Integrate Speech Detector

**In [app.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/app.py), update WebSocket handler:**

```python
async def websocket_transcribe(websocket: WebSocket):
    await websocket.accept()
    
    # State variables
    is_speaking = False
    
    # Callbacks
    def on_speech_start():
        nonlocal is_speaking
        is_speaking = True
        print("🎤 Speech started!")
    
    def on_speech_end(audio):
        nonlocal is_speaking
        is_speaking = False
        print(f"🔇 Speech ended, got {len(audio)} samples")
        # TODO: Transcribe this audio
    
    # Create speech detector
    speech_detector = SpeechDetector(
        on_speech_start=on_speech_start,
        on_speech_end=on_speech_end
    )
    speech_detector.start()
    
    # Main loop
    while True:
        message = await websocket.receive()
        
        if "bytes" in message:
            # Convert PCM16 to Float32
            audio_float = pcm16_to_float32(message["bytes"])
            
            # Feed to speech detector
            speech_detector.feed_audio(audio_float)
```

**Test:**
1. Run server
2. Start frontend
3. Speak into microphone
4. Server console should show:
   - "🎤 Speech started!" when you speak
   - "🔇 Speech ended..." when you stop

**Checkpoint:** Speech detection working! ✅

---

## 🤖 Part 4: Transcription

### Step 4.1: Whisper ASR Wrapper

**File:** [backend/asr.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/asr.py)

**What this does:**
- Loads Whisper model
- Converts audio to text
- Returns transcription results

**Create the class:**

```python
from faster_whisper import WhisperModel

class WhisperASR:
    def __init__(self, model_name="base", device="cpu"):
        self.model_name = model_name
        self._model = None  # Lazy load
    
    def _ensure_model_loaded(self):
        if self._model is None:
            print(f"Loading {self.model_name} model...")
            self._model = WhisperModel(
                self.model_name,
                device="cpu",
                compute_type="int8"
            )
            print("Model loaded!")
    
    def transcribe_audio(self, audio, sample_rate=16000, language=None):
        self._ensure_model_loaded()
        
        # Transcribe with VAD filter
        segments_iter, info = self._model.transcribe(
            audio,
            language=language,
            beam_size=5,
            vad_filter=True,  # Important!
            vad_parameters={
                "threshold": 0.3,
                "min_speech_duration_ms": 100,
                "min_silence_duration_ms": 300,
            }
        )
        
        # Collect results
        segments = []
        full_text = []
        
        for seg in segments_iter:
            segments.append({
                "start": seg.start,
                "end": seg.end,
                "text": seg.text.strip()
            })
            full_text.append(seg.text.strip())
        
        return {
            "text": " ".join(full_text),
            "segments": segments
        }
```

**Test the ASR:**
```python
# Create test
asr = WhisperASR(model_name="tiny.en")

# Create sample audio (1 second of silence, just for testing)
test_audio = np.zeros(16000, dtype=np.float32)

result = asr.transcribe_audio(test_audio)
print(result)  # Will be empty since it's silence
```

---

### Step 4.2: Final Transcription

**Update [on_speech_end](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/app.py#418-468) callback in [app.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/app.py):**

```python
# Create ASR instance (at module level)
asr = WhisperASR(model_name="base.en")

def on_speech_end(audio):
    nonlocal is_speaking
    is_speaking = False
    
    print(f"🔇 Speech ended, transcribing...")
    
    # Schedule async transcription
    asyncio.run_coroutine_threadsafe(
        transcribe_final_audio(audio),
        main_loop  # Event loop reference
    )

async def transcribe_final_audio(audio):
    # Run in executor (don't block)
    result = await asyncio.get_event_loop().run_in_executor(
        None,
        lambda: asr.transcribe_audio(audio, sample_rate=16000)
    )
    
    text = result.get("text", "").strip()
    
    if text:
        # Send to client
        await websocket.send_json({
            "type": "fullSentence",
            "text": text
        })
        print(f"✅ Transcribed: {text}")
```

**Test:**
1. Run server
2. Start frontend
3. Speak a sentence: "Hello world"
4. Stop speaking
5. Wait ~1 second
6. Should see final text appear in UI

**Checkpoint:** Final transcription working! ✅

---

### Step 4.3: Realtime Transcription - The Critical Part

**This is what makes it "real-time"!**

**Key insight:** We need TWO separate audio buffers:
1. **Speech detector buffer** - For VAD processing
2. **Raw frames buffer** - For transcription

**Why?**
- Speech detector optimizes for VAD
- We need unprocessed audio for best transcription

**Implementation:**

**Step 4.3.1: Add raw frames buffer**

In WebSocket handler:
```python
# Add these state variables
raw_frames = []  # Store raw PCM16 chunks
text_storage = []  # For stabilization
```

**Step 4.3.2: Accumulate raw frames**

In message handling loop:
```python
if "bytes" in message:
    pcm_bytes = message["bytes"]
    
    # Store raw PCM16 if speaking
    if is_speaking:
        raw_frames.append(pcm_bytes)
    
    # Also feed to speech detector
    audio_float = pcm16_to_float32(pcm_bytes)
    speech_detector.feed_audio(audio_float)
```

**Step 4.3.3: Create realtime worker**

```python
async def realtime_transcription_worker():
    while is_running:
        # Only process if speaking
        if not is_speaking:
            await asyncio.sleep(0.1)
            continue
        
        # Wait 200ms between updates
        await asyncio.sleep(0.2)
        
        # Skip if finalizing
        if awaiting_speech_end:
            continue
        
        # Get accumulated audio
        if len(raw_frames) == 0:
            continue
        
        # Join all PCM16 chunks
        audio_bytes = b''.join(raw_frames)
        
        # Need at least 200ms
        if len(audio_bytes) < 6400:  # 200ms of PCM16
            continue
        
        # Convert to float32
        audio_int16 = np.frombuffer(audio_bytes, dtype=np.int16)
        audio_float = audio_int16.astype(np.float32) / 32768.0
        
        # CRITICAL: Normalize to -0.95 dBFS
        peak = np.max(np.abs(audio_float))
        if peak > 0:
            audio_float = (audio_float / peak) * 0.95
        
        # Transcribe (non-blocking)
        result = await asyncio.get_event_loop().run_in_executor(
            None,
            lambda: asr_realtime.transcribe_audio(audio_float)
        )
        
        partial_text = result.get("text", "").strip()
        
        # Stabilize text
        text_storage.append(partial_text)
        
        if len(text_storage) >= 2:
            import os
            stable_text = os.path.commonprefix([
                text_storage[-2],
                text_storage[-1]
            ]).strip()
        else:
            stable_text = partial_text
        
        if stable_text:
            await websocket.send_json({
                "type": "realtime",
                "text": stable_text
            })
```

**Step 4.3.4: Start realtime worker**

In WebSocket handler init:
```python
# Start realtime worker
is_running = True
realtime_task = asyncio.create_task(realtime_transcription_worker())
```

**Step 4.3.5: Add awaiting_speech_end flag**

Update on_speech_start:
```python
def on_speech_start():
    nonlocal is_speaking, raw_frames, awaiting_speech_end
    is_speaking = True
    awaiting_speech_end = False
    raw_frames = []  # Clear for new speech
```

Update on_speech_end:
```python
def on_speech_end(audio):
    nonlocal is_speaking, awaiting_speech_end
    is_speaking = False
    awaiting_speech_end = True  # Skip realtime during final
    
    asyncio.run_coroutine_threadsafe(...)
```

Update transcribe_final_audio:
```python
async def transcribe_final_audio(audio):
    nonlocal awaiting_speech_end
    
    # ... transcription ...
    
    awaiting_speech_end = False  # Clear flag
```

**Test:**
1. Run server
2. Start frontend
3. Speak slowly: "Hello, how are you today?"
4. Watch live captions appear as you speak
5. Final transcription after you stop

**Checkpoint:** Realtime transcription working! 🎉

---

## 🔗 Part 5: Integration & Testing

### Step 5.1: Audio Worker (Optional Optimization)

**Why?**
- VAD processing can block main thread
- Separate worker thread keeps WebSocket responsive

**File:** [backend/audio_worker.py](file:///Users/priyanshgodhani/Desktop/devx/RealtimeSTT/transcription_subtitles/backend/audio_worker.py)

Create a queue-based worker:
```python
class AudioProcessingWorker:
    def __init__(self, speech_detector):
        self.speech_detector = speech_detector
        self.audio_queue = queue.Queue(maxsize=1000)
        self.is_running = False
    
    def start(self):
        self.is_running = True
        self.worker_thread = threading.Thread(
            target=self._process_loop,
            daemon=True
        )
        self.worker_thread.start()
    
    def feed_audio(self, audio):
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

**Update WebSocket handler** to use worker instead of direct feed.

---

### Step 5.2: Complete Testing

**Test Scenarios:**

1. **Single word:** "Hello"
   - ✅ Realtime shows "Hello"
   - ✅ Final shows "Hello"

2. **Short sentence:** "How are you?"
   - ✅ Realtime progressively shows words
   - ✅ Final shows complete sentence

3. **Long speech:** Talk for 10+ seconds
   - ✅ Realtime updates continuously
   - ✅ Final transcription accurate

4. **Silence test:** Speak, wait 5 seconds, speak again
   - ✅ No hallucinations during silence
   - ✅ Both speeches transcribed separately

5. **Noise test:** Background music/talking
   - ✅ Only your speech transcribed
   - ✅ Background filtered out

---

### Step 5.3: Performance Tuning

**If transcriptions are slow:**
- Use `tiny.en` model for realtime
- Use `base.en` only for final
- Reduce realtime frequency (0.3s instead of 0.2s)

**If audio is choppy:**
- Increase buffer size (1024 instead of 512)
- Check CPU usage
- Reduce concurrent sessions

**If accuracy is poor:**
- Ensure audio normalization working
- Check VAD sensitivity
- Try larger model (base→small)

---

## ✅ Final Checklist

Before you're done, verify:

- [ ] Frontend loads without errors
- [ ] WebSocket connects successfully
- [ ] Microphone permission granted
- [ ] Speech detection works (start/end callbacks)
- [ ] Realtime text appears while speaking
- [ ] Final text appears after speaking
- [ ] No hallucinations during silence
- [ ] Text is stable (not flickering)
- [ ] Multiple sentences work
- [ ] Server handles disconnect gracefully

---

## 🎓 What You've Learned

1. **Audio Processing**
   - Microphone capture
   - AudioWorklet API
   - Resampling techniques
   - Audio format conversions

2. **Real-Time Communication**
   - WebSocket bidirectional streaming
   - Non-blocking async operations
   - Event-driven architecture

3. **Speech Recognition**
   - Voice Activity Detection (VAD)
   - Whisper ASR integration
   - Model selection trade-offs

4. **System Design**
   - Multi-threaded architecture
   - State management
   - Buffer management
   - Performance optimization

---

## 🚀 Next Steps

**Enhancements you can add:**
- Multiple language support
- Speaker diarization
- Punctuation restoration
- Export to SRT/VTT
- Cloud deployment
- Mobile support

**Resources:**
- Whisper documentation
- FastAPI docs
- Web Audio API guide

---

## 🐛 Troubleshooting

**Problem:** No audio being sent
- Check microphone permissions
- Verify AudioWorklet loading
- Check WebSocket connection

**Problem:** Empty transcriptions
- Verify audio normalization
- Check if audio reaches ASR
- Try speaking louder

**Problem:** Server hangs
- Ensure async/await used correctly
- Check for blocking operations
- Use thread pool for heavy tasks

**Problem:** Text flickering
- Verify commonprefix logic
- Check text_storage updates
- Adjust update frequency

---

**Congratulations! 🎉**  
You've built a production-ready real-time transcription system!

---

*Workshop Tutorial v1.0*  
*Perfect for live coding sessions, workshops, and self-paced learning*
