# MMA Vision Intelligence Platform - System Design

## Project Overview

**Objective:** Build an intelligent computer vision system that extracts semantic meaning and tactical significance from MMA footage in real-time or post-hoc analysis.

**Core Capability:** The system will identify and classify:
- **Positional states** (guard variations, mount, side control, back control, clinch positions, standing)
- **Submission attempts** (chokes, joint locks, cranks) with setup and defense detection
- **Control sequences** (positional transitions, dominance metrics, escape attempts)
- **Takedown techniques** (single/double leg, throws, trips, timing)
- **Striking patterns** (punch combinations, kick types, defensive movements, counter sequences)
- **Fight dynamics** (momentum shifts, fatigue indicators, damage assessment)

**Deployment Modes:**
1. **Live Analysis:** Real-time overlay on streaming footage
2. **Upload Analysis:** Batch processing of recorded fights
3. **API Access:** Programmatic access for third-party integrations

---

## System Architecture

### High-Level Components

```
┌─────────────────────────────────────────────────────────────┐
│                     Frontend (TanStack Start)                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │ Video Upload │  │ Live Stream  │  │  Analytics   │      │
│  │   Interface  │  │   Viewer     │  │  Dashboard   │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
└────────────────────────┬────────────────────────────────────┘
                         │ WebSocket / REST API
┌────────────────────────┴────────────────────────────────────┐
│                  Backend (TanStack Start SSR)                │
│  ┌──────────────────────────────────────────────────────┐   │
│  │              API Router & WebSocket Server           │   │
│  └─────┬────────────────────────────────────────────────┘   │
│        │                                                     │
│  ┌─────┴─────────────┐         ┌───────────────────────┐   │
│  │  Video Processing │         │   Analysis Storage    │   │
│  │  Queue Manager    │◄────────┤   (PostgreSQL/S3)     │   │
│  └─────┬─────────────┘         └───────────────────────┘   │
└────────┼─────────────────────────────────────────────────────┘
         │
┌────────┴─────────────────────────────────────────────────────┐
│              ML Inference Service (Python/FastAPI)            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                  Frame Preprocessor                   │   │
│  │          (Resize, Normalize, Augmentation)            │   │
│  └─────┬────────────────────────────────────────────────┘   │
│        │                                                     │
│  ┌─────┴─────────────┐         ┌───────────────────────┐   │
│  │  Object Detection │         │  Action Recognition   │   │
│  │  (Fighter Pose    │         │  (Temporal CNN/LSTM   │   │
│  │   Estimation)     │         │   or Transformer)     │   │
│  └─────┬─────────────┘         └─────┬─────────────────┘   │
│        │                             │                       │
│  ┌─────┴─────────────────────────────┴─────────────────┐   │
│  │          Semantic Classification Layer               │   │
│  │  (Position, Submission, Strike, Takedown Classifiers)│   │
│  └─────┬────────────────────────────────────────────────┘   │
│        │                                                     │
│  ┌─────┴─────────────────────────────────────────────────┐  │
│  │            Temporal Context Analyzer                   │  │
│  │   (Sequence tracking, momentum detection, scoring)     │  │
│  └────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────┘
```

---

## Technical Stack

### Frontend & Backend
- **Framework:** TanStack Start (full-stack React with SSR)
- **Styling:** Tailwind CSS
- **State Management:** TanStack Query for server state
- **Video Player:** Video.js or custom WebRTC for live streaming
- **Real-time:** WebSocket (ws/Socket.io) for live inference results

### ML/CV Pipeline
- **Framework:** PyTorch (primary), TensorFlow (if needed for specific models)
- **Inference Server:** FastAPI + ONNX Runtime (for production) or TorchServe
- **Preprocessing:** OpenCV, FFmpeg
- **Model Architecture Options:**
  - **Base Detection:** YOLOv8/v9 or RT-DETR for fighter detection
  - **Pose Estimation:** MediaPipe Pose or YOLOv8-Pose
  - **Action Recognition:**
    - SlowFast networks (two-pathway 3D CNN)
    - TimeSformer (spatial-temporal transformer)
    - X3D (efficient 3D CNN)
  - **Custom Classification Heads:** Fine-tuned on MMA-specific actions

### Data & Storage
- **Database:** PostgreSQL (fight metadata, analysis results, user data)
- **Object Storage:** S3 or Cloudflare R2 (video files, model weights)
- **Cache:** Redis (inference results, session data)
- **Vector DB:** Pinecone/Weaviate (for semantic search of fight moments)

### Infrastructure
- **Containerization:** Docker
- **Orchestration:** Docker Compose (dev), Kubernetes (prod)
- **GPU Access:** NVIDIA GPUs with CUDA for inference
- **Deployment:** Vercel/Cloudflare (frontend), Railway/Fly.io (backend), Replicate/Modal (ML inference)

---

## ML Model Design

### Phase 1: Core Detection Models

#### 1.1 Fighter Detection & Pose Estimation
**Input:** Video frame (1920x1080 @ 30fps)
**Output:** Bounding boxes + 17-33 keypoint coordinates per fighter

**Approach:**
- Fine-tune YOLOv8-Pose or use MediaPipe Pose
- Track fighter IDs across frames (DeepSORT or ByteTrack)
- Extract skeletal features for downstream tasks

#### 1.2 Positional Classification
**Input:** Sequence of 16-32 frames (temporal window)
**Output:** Position class + confidence score

**Classes (v1):**
```python
POSITIONS = [
    # Ground positions
    "closed_guard", "open_guard", "half_guard", "butterfly_guard",
    "full_mount", "side_control", "back_control", "turtle",
    "north_south", "knee_on_belly",
    # Standing positions
    "standing_neutral", "clinch_over_under", "clinch_collar_tie",
    "against_cage", "separated",
]
```

**Architecture:**
- Spatial feature extraction from pose keypoints
- Temporal aggregation with LSTM or Temporal Shift Module (TSM)
- Multi-class classifier with softmax

#### 1.3 Action Recognition (Strikes, Takedowns, Submissions)
**Input:** 32-64 frame sequences (1-2 seconds)
**Output:** Action class + temporal localization

**Classes (v1):**
```python
STRIKES = [
    "jab", "cross", "hook", "uppercut", "overhand",
    "front_kick", "roundhouse", "leg_kick", "teep",
    "elbow", "knee", "spinning_backfist", "head_kick"
]

TAKEDOWNS = [
    "single_leg", "double_leg", "blast_double",
    "high_crotch", "ankle_pick", "hip_throw",
    "sweep", "trip", "suplex", "slam"
]

SUBMISSIONS = [
    "rear_naked_choke", "guillotine", "arm_triangle",
    "triangle_choke", "armbar", "kimura", "americana",
    "omoplata", "heel_hook", "kneebar", "ankle_lock"
]
```

**Architecture:**
- SlowFast or X3D for efficient temporal modeling
- Multi-task head: action class + temporal boundaries
- Focal loss to handle class imbalance (rare submissions)

#### 1.4 Semantic Significance Analyzer
**Input:** Aggregated predictions from previous models
**Output:** Fight narrative and tactical insights

**Features:**
- Control time per position
- Submission attempt frequency
- Strike differential and accuracy
- Damage indicators (significant strikes landed)
- Momentum scoring (based on action sequence)

---

## Data Pipeline

### Training Data Acquisition
1. **Public MMA datasets:**
   - YouTube fights (manual annotation required)
   - UFC Fight Pass (check licensing)
   - Open-source combat sports datasets

2. **Annotation Strategy:**
   - Use CVAT or Label Studio for video annotation
   - Bootstrap with weak supervision (existing fight stats)
   - Active learning loop: model predicts → human corrects → retrain

3. **Data Augmentation:**
   - Temporal jittering, frame sampling variations
   - Color/brightness adjustments (different lighting conditions)
   - Synthetic occlusions (simulate referee/cage interference)

### Annotation Schema
```json
{
  "video_id": "fight_123_round_1",
  "fps": 30,
  "annotations": [
    {
      "frame_start": 450,
      "frame_end": 480,
      "event_type": "takedown",
      "event_subtype": "single_leg",
      "fighter_id": "red_corner",
      "success": true,
      "significance": "high"
    },
    {
      "frame_start": 500,
      "frame_end": 750,
      "event_type": "position",
      "event_subtype": "side_control",
      "controller": "red_corner"
    }
  ]
}
```

---

## Frontend Architecture (TanStack Start)

### Key Routes
```
/                          # Landing page
/upload                    # Video upload interface
/live                      # Live stream analysis
/analysis/:fightId         # Post-fight analysis dashboard
/compare/:fightId1/:fightId2  # Compare two fights
/api/inference             # WebSocket endpoint for real-time
/api/upload                # REST endpoint for video upload
```

### State Management
- **TanStack Query** for server state (fight data, analysis results)
- **Zustand/Jotai** for client state (video playback position, UI preferences)
- **WebSocket** connection for live inference stream

### Video Player Integration
```typescript
interface VideoAnalysisOverlay {
  currentPosition: PositionLabel;
  activeEvents: Event[];
  stats: {
    controlTime: Record<string, number>;
    strikesLanded: Record<string, number>;
    takedownAttempts: number;
  };
  timeline: TimelineEvent[];
}
```

---

## Backend Architecture (TanStack Start + Python)

### API Design

#### REST Endpoints
```
POST   /api/fights/upload          # Upload video for analysis
GET    /api/fights/:id             # Get fight analysis results
GET    /api/fights/:id/events      # Get detailed event timeline
POST   /api/inference/batch        # Batch inference request
GET    /api/models/status          # ML model health check
```

#### WebSocket Events
```typescript
// Client → Server
type ClientMessage =
  | { type: 'subscribe', fightId: string }
  | { type: 'stream_frame', data: ArrayBuffer };

// Server → Client
type ServerMessage =
  | { type: 'inference_result', data: InferenceOutput }
  | { type: 'error', message: string };
```

### Video Processing Queue
- **Queue:** BullMQ (Redis-backed)
- **Workers:** Process uploaded videos in chunks
- **Flow:**
  1. Video uploaded → S3
  2. Job enqueued with video URL
  3. Worker extracts frames (FFmpeg)
  4. Frames sent to ML inference service in batches
  5. Results aggregated and stored in PostgreSQL
  6. WebSocket notification sent to client

---

## ML Inference Service (Python)

### FastAPI Service Structure
```python
# app/main.py
from fastapi import FastAPI, WebSocket
from app.models import ModelRegistry
from app.inference import InferencePipeline

app = FastAPI()
models = ModelRegistry()
pipeline = InferencePipeline(models)

@app.post("/predict/frame")
async def predict_frame(frame: UploadFile):
    """Single frame inference"""
    result = await pipeline.process_frame(frame)
    return result

@app.websocket("/predict/stream")
async def predict_stream(websocket: WebSocket):
    """Real-time streaming inference"""
    await websocket.accept()
    async for frame in websocket.iter_bytes():
        result = await pipeline.process_frame(frame)
        await websocket.send_json(result)
```

### Model Registry
- Load models on startup
- Version management (A/B testing)
- Fallback to CPU if GPU unavailable
- Model warming (first inference is slow)

### Inference Optimization
- **Batching:** Accumulate frames, infer in batches
- **ONNX Runtime:** 2-3x faster than PyTorch for inference
- **TensorRT:** Further optimization for NVIDIA GPUs
- **Frame Skipping:** Infer every N frames for less critical events
- **Multi-threaded preprocessing:** OpenCV operations in parallel

---

## Development Phases

### Phase 1: MVP (Weeks 1-4)
- [ ] TanStack Start project setup
- [ ] Basic video upload and playback
- [ ] Fighter detection with YOLOv8
- [ ] Simple position classifier (standing vs ground)
- [ ] Display bounding boxes and position labels

### Phase 2: Core ML Models (Weeks 5-10)
- [ ] Collect and annotate training data (500+ fight clips)
- [ ] Train positional classification model (10-15 classes)
- [ ] Train strike detection model
- [ ] Train takedown detection model
- [ ] Integrate models into inference pipeline

### Phase 3: Temporal Analysis (Weeks 11-14)
- [ ] Sequence tracking and temporal smoothing
- [ ] Control time calculation
- [ ] Submission attempt detection
- [ ] Fight momentum scoring
- [ ] Timeline visualization in frontend

### Phase 4: Live Analysis (Weeks 15-18)
- [ ] WebSocket streaming pipeline
- [ ] Real-time inference optimization (<100ms latency)
- [ ] Overlay rendering system
- [ ] Screen capture integration (for live broadcasts)

### Phase 5: Advanced Features (Weeks 19+)
- [ ] Multi-fight comparison
- [ ] Fighter style profiling
- [ ] Predictive analytics (probable next action)
- [ ] Semantic search ("find all triangle choke attempts")
- [ ] API for third-party integrations

---

## Model Training Strategy

### Training Infrastructure
- **Compute:** Google Colab Pro / Lambda Labs / Modal
- **Experiment Tracking:** Weights & Biases or MLflow
- **Data Versioning:** DVC (Data Version Control)

### Training Loop (Typical)
```python
# Pseudocode for action recognition training
for epoch in range(num_epochs):
    for batch in train_loader:
        frames, labels = batch
        outputs = model(frames)  # (B, num_classes)
        loss = focal_loss(outputs, labels)  # Handle class imbalance
        loss.backward()
        optimizer.step()

    # Validation
    val_accuracy = evaluate(model, val_loader)
    if val_accuracy > best_accuracy:
        save_checkpoint(model)
```

### Metrics to Track
- **Detection:** mAP (mean Average Precision), IoU
- **Classification:** Accuracy, F1-score per class, confusion matrix
- **Temporal:** Temporal IoU for action localization
- **Business:** User-reported accuracy, false positive rate

---

## Testing Strategy

### Unit Tests
- Model output shapes and ranges
- Data preprocessing correctness
- API endpoint responses

### Integration Tests
- End-to-end video upload → inference → results
- WebSocket connection handling
- Queue processing and error recovery

### Model Tests (TDD Approach)
```python
def test_position_classifier_output_shape():
    model = load_model("position_classifier")
    sample_input = torch.randn(1, 3, 16, 224, 224)  # 16 frames
    output = model(sample_input)
    assert output.shape == (1, NUM_POSITION_CLASSES)

def test_position_classifier_known_case():
    """Test on annotated ground truth"""
    frames = load_test_frames("full_mount_sequence.mp4")
    prediction = model.predict(frames)
    assert prediction["class"] == "full_mount"
    assert prediction["confidence"] > 0.85
```

---

## Deployment Considerations

### Scalability
- **Horizontal scaling:** Multiple inference workers behind load balancer
- **Autoscaling:** Scale based on queue depth
- **Caching:** Cache results for identical video segments

### Monitoring
- **Inference latency:** Track p50, p95, p99
- **Model drift:** Monitor prediction distribution over time
- **Error rates:** Track failed inferences and user corrections
- **GPU utilization:** Ensure efficient resource usage

### Security
- **Video storage:** Signed URLs, expiration policies
- **API rate limiting:** Prevent abuse
- **Input validation:** Sanitize uploaded videos (check format, duration)

### Cost Optimization
- **On-demand GPU:** Use Modal/Replicate for bursty workloads
- **Spot instances:** For batch processing
- **Model quantization:** INT8 quantization for 4x speedup with minimal accuracy loss

---

## Open Questions & Decisions

1. **Cage/referee occlusion handling:** How to maintain tracking when fighters are obscured?
2. **Multi-angle fusion:** If multiple camera angles available, how to fuse predictions?
3. **Real-time vs accuracy tradeoff:** What's acceptable latency for live analysis?
4. **Annotation budget:** In-house vs crowdsourced vs semi-supervised learning?
5. **Model versioning in production:** How to A/B test new models safely?

---

## Success Metrics

### Technical
- Position classification accuracy > 90%
- Strike detection F1 > 0.85
- Inference latency < 100ms per frame (live mode)
- System uptime > 99.5%

### Product
- User-reported accuracy > 85% satisfaction
- <5% false positive rate on critical events (submissions, knockdowns)
- Support for 720p @ 30fps minimum, 1080p @ 60fps target

---

## References & Resources

### Papers
- "SlowFast Networks for Video Recognition" (Feichtenhofer et al., 2019)
- "YOLOv8: Real-Time Object Detection"
- "Temporal Shift Module for Efficient Video Understanding" (Lin et al., 2019)

### Datasets
- UCF101, Kinetics-400 (action recognition pre-training)
- COCO (person detection pre-training)

### Tools
- FFmpeg: Video processing
- CVAT: Video annotation
- ONNX: Model optimization
- TensorRT: NVIDIA GPU inference optimization

---

## Next Steps

1. **Review and refine** this system design with stakeholders
2. **Set up development environment** (TanStack Start + Python inference service)
3. **Create initial data collection pipeline** (YouTube scraping + annotation tool)
4. **Build Phase 1 MVP** with basic fighter detection
5. **Iterate rapidly** with weekly model evaluation on validation set
