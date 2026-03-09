# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Engineering Principles

### Senior Engineering Mindset

**Personality Traits:**
- **Determined:** See tasks through to completion. No half-finished work.
- **Consistent:** Predictable patterns, reliable outputs. Every time.
- **Detail-oriented:** Catch edge cases, verify assumptions, question inputs.

**Technical Approach:**
- Deep ML/CV expertise with full-stack capability
- Test-Driven Development: write tests first, ensure coverage
- No premature optimization: solve the problem, then optimize if needed
- Relentless fault-checking of own work before delivery
- Minimalistic code: nothing more than what's needed, nothing less

---

## Testing Philosophy

### Test-Driven Development

**Always write tests first:**
1. Write the test that defines expected behavior
2. Run test (it should fail)
3. Write minimal code to make it pass
4. Refactor if needed
5. Verify test still passes

**Why:** Tests define the contract. Code fulfills it. Not the other way around.

### Test Coverage Requirements

**Unit tests:**
- Every function that contains logic (not simple getters/setters)
- Edge cases: empty inputs, null values, boundary conditions
- Error paths: what happens when things fail?

**Integration tests:**
- Critical user paths end-to-end
- External service interactions (APIs, databases, file I/O)
- Error recovery and graceful degradation

**For ML/CV specifically:**
- Shape tests: verify tensor dimensions, output formats
- Ground truth tests: known inputs must produce known outputs
- Regression tests: model accuracy must not degrade on validation set

### Test Quality Over Quantity

Bad test (brittle):
```python
def test_process_video():
    result = process_video("test.mp4")
    assert len(result) == 42  # Magic number, will break on any change
```

Good test (verifies behavior):
```python
def test_process_video_extracts_all_frames():
    video_30fps_2sec = "test_60frames.mp4"
    result = process_video(video_30fps_2sec)
    assert result.frame_count == 60
    assert result.fps == 30
    assert result.duration_sec == 2.0
```

---

## Code Documentation

### When to Document

**Document:**
- Non-obvious algorithmic decisions ("Using Floyd-Warshall because...")
- Performance constraints ("This must complete in <100ms")
- Data format expectations ("Expects tensor in NCHW format")
- Failure modes and recovery strategies
- ML model assumptions ("Requires normalized inputs in [0,1]")

**Don't document:**
- What the code obviously does (`x = x + 1  # increment x`)
- Information already in the function signature
- Implementation details that may change

### Documentation Standards

**Function docstrings (when needed):**
```python
def extract_pose_keypoints(frame: np.ndarray, model: PoseModel) -> List[Keypoint]:
    """
    Extract pose keypoints from a single frame.

    Args:
        frame: RGB image, shape (H, W, 3), values in [0, 255]
        model: Pre-loaded pose estimation model

    Returns:
        List of (x, y, confidence) tuples, 17 keypoints per person detected

    Raises:
        ValueError: If frame dimensions invalid or empty
    """
```

**Inline comments (sparingly):**
Only when the "why" isn't obvious from the code itself.

```python
# Temporal smoothing: interpolate missing keypoints from adjacent frames
# to handle occlusions (cage, referee blocking view)
if keypoint.confidence < 0.3:
    keypoint = interpolate_from_neighbors(keypoints, frame_idx)
```

---

## Iteration Process

### Feature Development Cycle

1. **Understand requirements fully**
   - Ask clarifying questions before writing code
   - Identify edge cases and failure modes
   - Confirm acceptance criteria

2. **Write tests first**
   - Define expected behavior in tests
   - Cover happy path and error cases

3. **Implement minimal solution**
   - Solve the problem, nothing more
   - No speculative features ("we might need this later")
   - No premature abstractions (three similar lines > early abstraction)

4. **Verify and fault-check**
   - Run all tests
   - Manual verification of critical paths
   - Check for security issues (injection, XSS, OWASP Top 10)

5. **Second look (critical)**
   - Review own code as if you didn't write it
   - Question every assumption
   - Look for edge cases you missed
   - Verify error handling exists and is correct

6. **Refactor only if needed**
   - Remove duplication only when pattern is clear
   - Simplify complex logic if it obscures intent
   - Keep changes minimal

### No Premature Optimization

**Optimize when:**
- Profiling shows it's a bottleneck
- Requirements demand specific performance (e.g., <100ms latency)
- Users report performance issues

**Don't optimize when:**
- "This might be slow" (prove it first)
- "This could be more efficient" (measure first)
- Making code more complex for theoretical gains

**Example:**
```python
# Don't do this prematurely:
cache = LRUCache(maxsize=10000)
@cache
def expensive_operation(x):
    return x * 2  # Not actually expensive

# Do this instead:
def expensive_operation(x):
    return x * 2

# Add caching later if profiling shows it's needed
```

---

## Code Quality Standards

### Minimalism

**Only build what's explicitly requested:**
- No additional features "while we're here"
- No refactoring unrelated code
- No adding error handling for impossible scenarios

**Examples of over-engineering to avoid:**
```python
# BAD: Premature abstraction
class VideoProcessorFactory:
    def create_processor(self, type: str) -> IVideoProcessor:
        # Only one processor exists, no need for factory

# GOOD: Direct implementation
def process_video(path: str) -> VideoResult:
    # Just do the thing
```

**Delete unused code completely:**
- No commented-out code
- No `# TODO: remove this later`
- No renaming variables to `_unused_var` to avoid warnings
- If it's not used, delete it

### Security First

**Always check for vulnerabilities:**
- Command injection (shell=True, os.system with user input)
- SQL injection (raw SQL queries with string formatting)
- XSS (rendering user input without sanitization)
- Path traversal (user-controlled file paths)
- OWASP Top 10

**If you write insecure code, immediately fix it before moving on.**

### Explicit Over Implicit

**Prefer:**
```python
def predict(frames: np.ndarray, confidence_threshold: float = 0.8) -> List[Prediction]:
    # Clear what the threshold does
```

**Avoid:**
```python
def predict(frames, **kwargs):
    thresh = kwargs.get('thresh', 0.8)  # What is thresh?
```

---

## Verification and Fault-Checking

### Self-Review Checklist

Before considering code complete:

1. **Does it work?**
   - All tests pass
   - Manual verification of critical paths
   - Edge cases handled

2. **Is it secure?**
   - No injection vulnerabilities
   - User input validated
   - Sensitive data not logged/exposed

3. **Is it minimal?**
   - No unused code
   - No speculative features
   - No premature abstractions

4. **Is it correct?**
   - Logic matches requirements
   - Error handling exists
   - Assumptions documented

5. **Does it degrade gracefully?**
   - What happens when external service fails?
   - What happens with invalid input?
   - Are error messages helpful?

### Second Look Process

**After writing code, step back and ask:**

- "What did I assume that might be wrong?"
- "What edge case did I miss?"
- "What happens if this input is null/empty/huge?"
- "Is this the simplest solution?"
- "Would I understand this code in 6 months?"

**Common blind spots to check:**
- Off-by-one errors in loops/indices
- Race conditions in async code
- Memory leaks in long-running processes
- Floating point comparison issues
- Time zone handling

---

## ML/CV Specific Practices

### Model Development

**Reproducibility:**
- Fix random seeds everywhere (Python, NumPy, PyTorch, etc.)
- Log all hyperparameters with every run
- Version training data (data changes = different model)
- Pin dependency versions in requirements.txt

**Testing ML code:**
```python
# Shape test: Architecture didn't break
def test_model_output_shape():
    model = load_model()
    batch = torch.randn(4, 3, 224, 224)
    output = model(batch)
    assert output.shape == (4, NUM_CLASSES)

# Behavior test: Model still works on known examples
def test_model_known_input():
    model = load_model()
    # Known input that should always produce same output
    test_input = load_fixture("canonical_test_input.pt")
    output = model(test_input)
    expected = load_fixture("canonical_test_output.pt")
    assert torch.allclose(output, expected, rtol=1e-4)
```

**Don't trust silent failures:**
- Models can produce garbage without raising errors
- Always verify outputs make sense (check ranges, distributions)
- Use validation sets, not just training metrics

### Inference Optimization (Only When Needed)

**When performance matters, optimize in this order:**
1. Algorithm choice (O(n²) → O(n log n) matters more than micro-optimizations)
2. Model architecture (smaller model = faster inference)
3. Batch processing (accumulate inputs, process together)
4. Framework optimization (PyTorch → ONNX → TensorRT)
5. Hardware (CPU → GPU, fp32 → fp16 → int8)

**Measure before and after. Don't assume optimization worked.**

---

## Working with Uncertainty

### When Requirements Are Unclear

**Don't guess. Ask:**
- "What should happen in this edge case?"
- "What's the priority: speed or accuracy?"
- "Should this fail loudly or degrade gracefully?"

**Document assumptions if answers aren't available:**
```python
def process_frame(frame: np.ndarray) -> Result:
    """
    Assumption: Frames are RGB, not BGR. If this assumption
    is wrong, all downstream predictions will be garbage.
    """
```

### When Multiple Solutions Exist

**Choose based on:**
1. Simplicity (simplest solution wins)
2. Performance (only if requirements demand it)
3. Maintainability (will I understand this later?)

**Not based on:**
- "This is more clever"
- "This uses fewer lines" (if it's harder to understand)
- "This might be useful someday"

---

## Repository Structure

This repository contains multiple projects in subdirectories:
- `mma/` - MMA Vision Intelligence Platform
- Additional projects will be added as needed

Each project contains its own `SYSTEM_DESIGN.md` with architecture specifics, tech stack choices, and implementation details.

This document defines **how we work**, not **what we build**.
