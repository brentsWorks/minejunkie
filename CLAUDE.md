# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Context

**MMA Vision Intelligence Platform** - A full-stack computer vision system that analyzes MMA footage to extract semantic meaning and tactical significance. The system identifies positions, strikes, takedowns, submissions, and fight dynamics from both live and uploaded video.

## System Architecture

### Two-Tier Service Model

**Frontend/Backend (TanStack Start):**
- Full-stack React with SSR
- Handles video upload, WebSocket streaming, API routing
- Manages video processing queue (BullMQ + Redis)
- Stores analysis results in PostgreSQL

**ML Inference Service (Python FastAPI):**
- Separate microservice for GPU-intensive inference
- Runs YOLOv8-Pose for fighter detection and pose estimation
- Processes temporal sequences with SlowFast/X3D networks
- Outputs structured predictions (positions, actions, events)

**Communication:** REST API for batch uploads, WebSocket for real-time streaming

### Critical Data Flow

1. **Upload mode:** Video → S3 → Queue → Frame extraction (FFmpeg) → Batched inference → PostgreSQL → Client
2. **Live mode:** Client streams frames via WebSocket → Inference service → Real-time predictions → Client overlay

## Development Commands

### TanStack Start (Frontend/Backend)
```bash
# Development server
npm run dev

# Build for production
npm run build

# Run tests
npm test

# Run single test file
npm test -- path/to/test.spec.ts

# Type checking
npm run typecheck

# Linting
npm run lint
```

### Python ML Service
```bash
# Activate virtual environment
source venv/bin/activate  # or `venv\Scripts\activate` on Windows

# Install dependencies
pip install -r requirements.txt

# Run inference service
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Run tests
pytest

# Run specific test
pytest tests/test_inference.py::test_position_classifier

# Type checking
mypy app/

# Linting
ruff check .
```

### Docker Compose (Full Stack)
```bash
# Start all services (web + inference + postgres + redis)
docker-compose up

# Start with rebuild
docker-compose up --build

# Run inference service only
docker-compose up inference-service

# View logs
docker-compose logs -f inference-service
```

## ML Model Architecture

### Model Pipeline Stages

**Stage 1: Detection & Pose**
- Input: 1920x1080 video frames @ 30fps
- Model: YOLOv8-Pose or MediaPipe Pose
- Output: Bounding boxes + 17-33 keypoint coordinates per fighter
- Tracking: DeepSORT/ByteTrack for fighter ID persistence across frames

**Stage 2: Temporal Feature Extraction**
- Input: 16-32 frame sequences (positional) or 32-64 frames (actions)
- Purpose: Capture motion dynamics beyond single frames
- Architecture: 3D CNN (SlowFast, X3D) or Temporal Shift Module with LSTM

**Stage 3: Multi-Task Classification**
- **Position Classifier:** 15 classes (guards, mounts, clinches, standing states)
- **Strike Detector:** 13 strike types with temporal localization
- **Takedown Detector:** 10 takedown techniques
- **Submission Detector:** 11 submission attempts (uses focal loss for rare events)

**Stage 4: Semantic Analyzer**
- Aggregates predictions into fight narrative
- Computes control time, momentum shifts, strike differential
- Outputs structured timeline for frontend visualization

### Model Location Convention
- Training code: `ml/training/`
- Inference code: `ml/inference/`
- Model weights: S3 bucket or `ml/models/` (not in git)
- ONNX exports: `ml/models/onnx/` for production inference

## Model Testing Patterns

**Shape tests:**
```python
def test_position_classifier_output_shape():
    model = load_model("position_classifier")
    sample_input = torch.randn(1, 3, 16, 224, 224)
    output = model(sample_input)
    assert output.shape == (1, NUM_POSITION_CLASSES)
```

**Ground truth tests:**
```python
def test_position_classifier_known_case():
    frames = load_test_frames("full_mount_sequence.mp4")
    prediction = model.predict(frames)
    assert prediction["class"] == "full_mount"
    assert prediction["confidence"] > 0.85
```

Models fail silently - shape tests catch architecture bugs, ground truth tests catch degradation.

## Performance Constraints

### Real-Time Inference Requirements
- **Target latency:** <100ms per frame for live mode
- **Optimization strategies:**
  - ONNX Runtime (2-3x faster than PyTorch)
  - TensorRT for NVIDIA GPUs (additional 2x speedup)
  - Frame skipping for non-critical events (infer every N frames)
  - Batched inference (accumulate frames, process together)

### Cost Optimization
- Use Modal/Replicate for on-demand GPU inference (auto-scaling)
- Redis caching for identical video segments
- Model quantization (INT8) for 4x speedup with <2% accuracy loss

## Data Annotation Schema

When adding training data or modifying annotation format:

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
    }
  ]
}
```

- Use CVAT or Label Studio for video annotation
- Store annotations in `data/annotations/`
- Always include temporal boundaries (`frame_start`, `frame_end`)

## API Contracts

### WebSocket Events
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

### REST Endpoints
```
POST   /api/fights/upload          # Upload video for analysis
GET    /api/fights/:id             # Get fight analysis results
GET    /api/fights/:id/events      # Get detailed event timeline
POST   /api/inference/batch        # Batch inference request
GET    /api/models/status          # ML model health check
```

## ML/CV Operational Details

**Model Storage:**
- Training code: `ml/training/`
- Inference code: `ml/inference/`
- Model weights: S3 bucket or `ml/models/` (not in git)
- ONNX exports: `ml/models/onnx/` for production

**Inference Service Behavior:**
- Model registry warms models on startup (first inference is slow)
- CPU fallback if GPU unavailable
- Health check verifies model loading, not just process alive

## Adding New Action Classes

1. Update class definition in `ml/constants.py` (e.g., `SUBMISSIONS.append("d'arce_choke")`)
2. Add training examples to dataset (minimum 50-100 annotated instances)
3. Retrain model with updated num_classes parameter
4. Export to ONNX for production
5. Update frontend TypeScript types to match new classes

**Model Naming:** `model_name_v{version}.pth` or `model_name_{timestamp}.pth`

## Known Constraints

### Occlusion Handling
- Cage/referee occlusions degrade pose estimation
- Current strategy: temporal smoothing (interpolate missing keypoints from adjacent frames)
- Future: multi-angle fusion if multiple camera feeds available

### Class Imbalance
- Submissions are rare events (<1% of fight time)
- Use focal loss for submission classifier
- Oversample rare classes during training
- Accept higher false positive rate for critical events (better to flag potential submission than miss it)

## Deployment Architecture

**Development:** Docker Compose (all services locally)
**Production:**
- Frontend: Vercel/Cloudflare Pages (SSR)
- Backend: Railway/Fly.io (Node.js + queue workers)
- Inference: Modal/Replicate (GPU auto-scaling)
- Database: Managed PostgreSQL (Neon/Supabase)
- Storage: S3/Cloudflare R2

## Reference Documentation

- TanStack Start: https://tanstack.com/start
- YOLOv8: https://docs.ultralytics.com/
- SlowFast Networks: https://github.com/facebookresearch/SlowFast
- ONNX Runtime: https://onnxruntime.ai/docs/

See `SYSTEM_DESIGN.md` for comprehensive architectural details, ML model specifications, and development phases.
