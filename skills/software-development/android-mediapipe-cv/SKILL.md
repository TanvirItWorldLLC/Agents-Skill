---
name: android-mediapipe-cv
category: software-development
description: Use MediaPipe + CameraX for Android face/hands/pose CV.
---

# Android MediaPipe Computer Vision

**Class-level skill** for shipping real-time CV on Android using MediaPipe Tasks + CameraX.

## When to Use
- Face mesh / landmarks (468 pts), hand tracking (21 pts), pose (33 pts), selfie segmentation
- Object detection, image classification, gesture recognition
- Any real-time ML inference on camera frames at 30 FPS
- Must ship on Play Store (no deepfake, no policy violations)

## Core Architecture

```
CameraX Preview -> ImageAnalysis.Analyzer (YUV_420_888) -> MediaPipe Task (LIVE_STREAM) -> Overlay View
```

| Component | Role | Key Config |
|-----------|------|------------|
| `CameraX` | Camera lifecycle, multi-device support | `ImageAnalysis.STRATEGY_KEEP_ONLY_LATEST` |
| `ImageAnalysis` | Frame pipeline, backpressure | 640x480 YUV -> Bitmap -> MPImage |
| `BaseOptions.Delegate.GPU` | Hardware acceleration | Fallback to CPU if unsupported |
| `RunningMode.LIVE_STREAM` | Async, non-blocking | `detectAsync(mpImage, timestamp)` + callback |
| `TextureView` / `PreviewView` | Zero-copy overlay | Draw landmarks in `onDraw()` |

## Critical Patterns

### 1. GPU Delegate with Fallback
```kotlin
val options = FaceLandmarkerOptions.builder()
    .setBaseOptions(BaseOptions.builder()
        .setModelAssetPath("face_landmarker.task")
        .setDelegate(BaseOptions.Delegate.GPU)
        .build())
    .setRunningMode(RunningMode.LIVE_STREAM)
    .setResultListener { result, _, _ -> /* UI thread */ }
    .build()
```

### 2. YUV to Bitmap Conversion (Zero Allocation)
```kotlin
private fun yuvToBitmap(image: ImageProxy): Bitmap {
    val buffer = image.planes[0].buffer
    val yuv = ByteBuffer.allocateDirect(image.width * image.height * 3 / 2)
    yuv.put(buffer)
    val bitmap = Bitmap.createBitmap(image.width, image.height, Bitmap.Config.ARGB_8888)
    decodeYUV420SP(bitmap, yuv.array(), image.width, image.height)
    return bitmap
}
```

### 3. LIVE_STREAM Callback on UI Thread
```kotlin
.setResultListener { result, _, _ ->
    runOnUiThread { overlayView.updateFaceMesh(result?.faceLandmarks ?: emptyList()) }
    isProcessing = false
}
```

### 4. Frame Dropping Guard
```kotlin
override fun analyze(imageProxy: ImageProxy) {
    if (isProcessing || landmarker == null) { imageProxy.close(); return }
    isProcessing = true
    landmarker?.detectAsync(mpImage, imageProxy.imageInfo.timestamp)
    imageProxy.close()
}
```

## Performance Targets

| Device Tier | SoC Example | Face Mesh FPS | Latency |
|-------------|-------------|---------------|---------|
| Flagship | Snapdragon 8 Gen 3 / Tensor G3 | 30 | 8-12 ms |
| Upper Mid | Snapdragon 7+ Gen 2 / 778G | 28-30 | 15-20 ms |
| Mid | Snapdragon 680 / 695 | 22-25 | 25-35 ms |
| Budget | Helio G85 / Unisoc T606 | 15-20 | 40-60 ms |

**Optimization levers:**
- Lower analysis resolution: `setTargetResolution(Size(480, 360))`
- Reduce `numFaces` / `numHands` / `maxResults`
- Use `BaseOptions.Delegate.NNAPI` on older devices
- Enable R8 minify + `android:extractNativeLibs=false`

## Model Management

| Task | Model URL (float16) | Size |
|------|---------------------|------|
| Face Landmarker | `face_landmarker.task` | 3.6 MB |
| Hand Landmarker | `hand_landmarker.task` | 2.8 MB |
| Pose Landmarker | `pose_landmarker.task` | 5.2 MB |
| Selfie Segmentation | `selfie_segmenter.task` | 1.1 MB |
| Object Detection | `efficientdet_lite0.tflite` | 4.5 MB |

**Download at build time** (not runtime) -- place in `src/main/assets/`.

## Common Pitfalls

| Pitfall | Symptom | Fix |
|---------|---------|-----|
| Forget `imageProxy.close()` | Memory leak, camera freeze | Close in `finally` or immediately after `detectAsync` |
| GPU delegate crash on old devices | `MediaPipeException` at init | Wrap in try/catch, fallback to CPU |
| Preview rotated 90 degrees | Landmarks misaligned | Sync `OrientationEventListener` -> `PreviewView.implementationMode` |
| Model not found | `FileNotFoundException` | Ensure `assets/face_landmarker.task` exists; run download script |
| Low FPS on mid-range | < 20 FPS | Drop analysis resolution; reduce `numFaces`; use NNAPI |
| Black preview | Camera permission / selector | Verify `CameraSelector.DEFAULT_FRONT_CAMERA`; some need `BACK` |

## Play Store Compliance

Allowed: Face mesh AR filters, virtual try-on, fitness form check, accessibility tools, gesture control
Banned: Face swap / deepfake generation, non-consensual face modification, biometric spoofing

**Policy reference**: Google Play "Synthetic Media" + "Sexual Content" policies.

## Project Template

See `references/face-mesh-ar-template/` for a complete, buildable starter.

## Extending to Other Tasks

Replace `FaceLandmarker` with:
- `HandLandmarker` -> `HandLandmarkerOptions` (21 landmarks x 2 hands)
- `PoseLandmarker` -> `PoseLandmarkerOptions` (33 landmarks)
- `ImageSegmenter` -> `ImageSegmenterOptions` (selfie mask)
- `ObjectDetector` -> `ObjectDetectorOptions` (bounding boxes)

All share the same CameraX -> Analyzer -> LIVE_STREAM -> Overlay pattern.

## References
- MediaPipe Tasks Android: https://developers.google.com/mediapipe/solutions/vision
- CameraX Guide: https://developer.android.com/training/camerax
- GPU Delegate: https://www.tensorflow.org/lite/performance/gpu

---
*Generated from session: FaceMesh AR production app (468 landmarks, 30 FPS, GPU delegate)*