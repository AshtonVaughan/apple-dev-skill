# Graphics, Media, ML & System Frameworks

AVFoundation (capture/playback/editing), Metal/SceneKit/RealityKit/SpriteKit (rendering), Core Image/vImage (image processing), Vision/Core ML/NaturalLanguage/Speech/Translation (ML), ARKit (AR), MapKit, StoreKit (IAP), PDFKit, PencilKit, WidgetKit/ActivityKit/AppIntents, PassKit, Core Spotlight.

## 1. Rendering stack overview

| Need | Framework | Notes |
|---|---|---|
| Simple 2-D drawing in SwiftUI | `Canvas` + `GraphicsContext` | immediate-mode, TimelineView for animation |
| Layer-based animation | Core Animation (`CALayer` subclasses) | implicit/explicit animations, GPU-composited |
| GPU compute/custom shaders | **Metal** | `MTLDevice`, `MTLRenderPipelineState`, `.metal` shaders |
| 3-D scenes/entities | RealityKit | entities/components/systems, USDZ assets |
| Legacy 3-D | SceneKit | `SCNScene`, `SCNNode` — maintenance mode, prefer RealityKit |
| 2-D games/sprites/physics | SpriteKit + GameplayKit | `SKScene`, `SKSpriteNode`, `SKPhysicsBody` |
| AR | ARKit + RealityKit/SceneKit | world tracking, planes, anchors, LiDAR |
| Image filters | Core Image (`CIFilter`, `CIImage`, `CIContext`) | GPU-accelerated, 200+ filters |
| Low-level pixels | vImage / Accelerate / Core Graphics | vDSP, vImage, CGImage/CGContext |
| Charts | Swift Charts | `Chart { BarMark/LineMark/AreaMark/PointMark/RuleMark }` |

## 2. Core Graphics & Core Animation

```swift
// Core Graphics — immediate mode drawing
let renderer = UIGraphicsImageRenderer(size: size)
let image = renderer.image { ctx in
  UIColor.red.setFill()
  ctx.cgContext.fill(CGRect(x: 0, y: 0, width: 100, height: 100))
  // or UIBezierPath, NSAttributedString.draw, etc.
}

// Core Animation layers (always layer-backed on modern systems)
let layer = CAGradientLayer()
layer.colors = [UIColor.red.cgColor, UIColor.blue.cgColor]
layer.startPoint = CGPoint(x: 0, y: 0); layer.endPoint = CGPoint(x: 1, y: 1)
view.layer.addSublayer(layer)

// CAAnimation — implicit (set property inside CATransaction) or explicit
let anim = CABasicAnimation(keyPath: "opacity")
anim.fromValue = 1.0; anim.toValue = 0.0; anim.duration = 0.3
layer.add(anim, forKey: "fade")
layer.opacity = 0.0   // set model value too
```

- `CALayer` subclasses: `CAShapeLayer` (vector paths), `CAGradientLayer`, `CATiledLayer` (large images), `CAEmitterLayer` (particles), `CAScrollLayer`, `CATransformLayer` (3-D transforms), `CAReplicatorLayer` (repeated sublayers — progress rings etc.).
- `CAShapeLayer` — `path` (CGPath), `fillColor`, `strokeColor`, `lineWidth`, `lineDashPattern`, `strokeStart`/`strokeEnd` for partial strokes (progress indicators).
- `UIView`/`NSView` properties: `layer` (CALayer), `wantsLayer`, `layerContentsRedrawPolicy`, `contentMode` → `contentsGravity`.
- Implicit animations: changing animatable layer property inside `CATransaction` auto-animates; disable via `CATransaction.setDisableActions(true)` or `UIView.setAnimationsEnabled(false)`.
- Explicit: `CABasicAnimation`, `CAKeyframeAnimation` (values/keyTimes), `CAAnimationGroup`, `CASpringAnimation` (iOS 9+), `CADisplayLink` for frame-synced callbacks.
- `UIView.animate(withDuration:delay:options:animations:completion:)` — block-based; `.curveEaseInOut`, `.autoreverse`, `.repeat`, `.beginFromCurrentState`, `.allowUserInteraction`, spring variants `usingSpringWithDamping`.

## 3. Metal

```swift
import Metal
let device = MTLCreateSystemDefaultDevice()!
let queue = device.makeCommandQueue()!
let pipeline = try device.makeRenderPipelineState(descriptor: {
  let d = MTLRenderPipelineDescriptor()
  d.vertexFunction = library.makeFunction(name: "vertexShader")
  d.fragmentFunction = library.makeFunction(name: "fragmentShader")
  d.colorAttachments[0].pixelFormat = .bgra8Unorm
  return d
}())

// Per-frame
let commandBuffer = queue.makeCommandBuffer()!
let pass = commandBuffer.makeRenderPassDescriptor()
pass.colorAttachments[0].texture = drawable.texture
pass.colorAttachments[0].loadAction = .clear
pass.colorAttachments[0].storeAction = .store
let encoder = commandBuffer.makeRenderCommandEncoder(descriptor: pass)!
encoder.setRenderPipelineState(pipeline)
encoder.setVertexBytes(&vertices, length: MemoryLayout<Vertex>.stride * count, index: 0)
encoder.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: count)
encoder.endEncoding()
commandBuffer.present(drawable)
commandBuffer.commit()
```

- Shaders: `.metal` files compiled to `default.metallib` (Xcode build phase); `MTLLibrary.makeFunction(name:)`.
- `MTLDevice` — GPU access; `makeCommandQueue`, `makeBuffer`, `makeTexture`, `makeRenderPipelineState`, `makeComputePipelineState`, `makeHeap`, `makeArgumentEncoder`.
- `MTLCommandBuffer`/`MTLCommandQueue` — command encoding; `MTLRenderCommandEncoder`, `MTLComputeCommandEncoder`, `MTLBlitCommandEncoder`.
- `MTLTexture`/`MTLBuffer` — GPU memory; `replaceRegion`, `getBytes`, `contents()` for CPU access.
- Argument buffers (Metal 2+/Apple silicon): `MTLArgumentEncoder` for binding many resources efficiently.
- Indirect command buffers (Metal 3): `MTLIndirectCommandBuffer` for GPU-driven rendering.
- Mesh shaders (Metal 3): `MTLMeshRenderCommandEncoder` for object-level culling.
- Ray tracing (Metal 3): `MTLRaytracingPipeline` — A-series/M-series only.
- Performance: minimize state changes, use argument buffers, fence/heap resources, profile with Metal System Trace + Xcode Metal debugger.
- SwiftUI integration: `MetalView` representable or `MTKView` (`MTKViewDelegate` — `draw(in:)` per frame); `CAMetalLayer` for layer-backed.

## 4. AVFoundation — capture

```swift
import AVFoundation

let session = AVCaptureSession()
session.sessionPreset = .high  // or configure formats manually

// Inputs
guard let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back),
      let input = try? AVCaptureDeviceInput(device: camera) else { return }
session.addInput(input)

// Audio
guard let mic = AVCaptureDevice.default(for: .audio),
      let micInput = try? AVCaptureDeviceInput(device: mic) else { return }
session.addInput(micInput)

// Preview
let previewLayer = AVCaptureVideoPreviewLayer(session: session)
previewLayer.videoGravity = .resizeAspectFill
previewView.layer.addSublayer(previewLayer)

// Photo output
let photoOutput = AVCapturePhotoOutput()
session.addOutput(photoOutput)
let settings = AVCapturePhotoSettings()
settings.flashMode = .auto
photoOutput.capturePhoto(with: settings, delegate: self)

// Video output
let movieOutput = AVCaptureMovieFileOutput()
session.addOutput(movieOutput)
movieOutput.startRecording(to: fileURL, recordingDelegate: self)

// Data outputs (frame processing)
let videoDataOutput = AVCaptureVideoDataOutput()
videoDataOutput.videoSettings = [kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA]
videoDataOutput.setSampleBufferDelegate(self, queue: videoQueue)
session.addOutput(videoDataOutput)

// Run on background queue — startRunning blocks
DispatchQueue.global(qos: .userInitiated).async { session.startRunning() }
```

- `AVCaptureSession` — pipeline coordinator: inputs → outputs; `sessionPreset` (.high, .medium, .photo, .inputPriority) or manual format configuration.
- `AVCaptureDevice` — cameras/mics: `default(_:for:position:)`, `devices(for:)`, `deviceType` (.builtInWideAngleCamera, .builtInUltraWideCamera, .builtInTelephotoCamera, .builtInTrueDepthCamera, .builtInDualCamera, .builtInTripleCamera, .builtInDualWideCamera, .builtInLiDARDepthCamera, .external), `position` (.front/.back/.unspecified), `formats` (AVCaptureDevice.Format — resolution, fps, HDR, depth), `activeFormat`, `activeVideoMin/MaxFrameDuration`, `isFocusModeSupported`, `focusPointOfInterest`, `exposurePointOfInterest`, `whiteBalanceMode`, `torchMode`, `videoZoomFactor`.
- `AVCaptureDeviceInput`/`AVCaptureOutput` — connection graph; `AVCaptureConnection` for orientation/mirroring.
- `AVCaptureVideoPreviewLayer` — preview layer (subclass of CALayer); `videoGravity` (.resizeAspect, .resizeAspectFill, .resize); `connection.videoOrientation`, `isVideoMirrored`.
- `AVCapturePhotoOutput` — still capture: `AVCapturePhotoSettings` (flashMode, photoQualityPrioritization, maxPhotoDimensions, livePhotoMovieFileURL); delegate `photoOutput(_:didFinishProcessingPhoto:error:)` → `AVCapturePhoto.fileDataRepresentation()`.
- `AVCaptureMovieFileOutput` — video recording: `startRecording(to:recordingDelegate:)`, `stopRecording()`; delegate `didFinishRecordingTo:outputFileURL:from:error:`.
- `AVCaptureVideoDataOutput`/`AVCaptureAudioDataOutput`/`AVCaptureMetadataOutput` — frame/audio/metadata streaming; `CMSampleBuffer` processing; `alwaysDiscardsLateVideoFrames` for realtime.
- `AVCaptureDepthDataOutput`/`AVCaptureLiDARDepthDataOutput` — depth maps (portrait, AR).
- `AVCaptureSession` multi-cam: `AVCaptureMultiCamSession` for simultaneous front+back (iOS 13+).
- Permissions: `AVCaptureDevice.requestAccess(for: .video/.audio)` → Bool; Info.plist `NSCameraUsageDescription`/`NSMicrophoneUsageDescription`.
- Thermal: `ProcessInfo.thermalState` — reduce quality/fps when `.serious`/`.critical`; `AVCaptureSession.isMultitaskingCameraAccessSupported` for PiP.

## 5. AVFoundation — playback

```swift
let player = AVPlayer(url: videoURL)   // or AVPlayerItem(asset:)
let playerLayer = AVPlayerLayer(player: player)
playerLayer.videoGravity = .resizeAspect
view.layer.addSublayer(playerLayer)
player.play()
player.pause()
player.seek(to: CMTime(seconds: 30, preferredTimescale: 600))
player.rate = 1.0   // 0.5 = half speed, -1.0 = reverse (if supported)

// KVO/observers
NotificationCenter.default.addObserver(forName: .AVPlayerItemDidPlayToEndTime, object: item, queue: .main) { _ in ... }
// or async: for await notification in NotificationCenter.default.notifications(named:)

// Picture in Picture
let pip = AVPictureInPictureController(playerLayer: playerLayer)
pip.delegate = self   // canStartPictureInPicture, restoreUserInterface
pip.startPictureInPicture()

// Subtitles/closed captions
let legible = item.asset.mediaSelectionGroup(forMediaCharacteristic: .legible)
let options = AVMediaSelectionGroup.mediaSelectionOptions(from: legible)
item.select(legible.options.first { $0.extendedLanguageTag == "en" }, in: legible)
```

- `AVPlayer`/`AVPlayerItem`/`AVPlayerLayer` — playback stack; `AVQueuePlayer` for playlists.
- `AVPlayerItem` — asset + status (`.unknown`, `.readyToPlay`, `.failed`), `duration`, `currentTime`, `seekableTimeRanges`, `loadedTimeRanges`, `presentationSize`, `tracks` (audio/video/subtitle).
- `AVAsset` — media container: `AVURLAsset`, `AVAssetExportSession`, `AVAssetReader`/`Writer`, `AVComposition`, `AVMutableComposition`.
- `AVAudioSession` — audio routing/mixing: `category` (.ambient, .soloAmbient, .playback, .record, .playAndRecord, .multiRoute), `mode` (.default, .measurement, .moviePlayback, .spokenAudio, .videoRecording, .voiceChat, .gameChat), `options` (.mixWithOthers, .duckOthers, .allowBluetooth, .allowBluetoothA2DP, .allowAirPlay, .interruptSpokenAudioAndMixWithOthers), `setActive(_:options:)`, `overrideOutputAudioPort(.speaker/.none)`.
- `AVAudioSession.interruptionNotification` — handle interruptions (phone call, Siri); `routeChangeNotification` — handle headphone unplug/route changes; `mediaServicesWereResetNotification` — recreate objects.
- `AVPlayerViewController` — system player UI with controls, PiP, AirPlay.
- `AVQueuePlayer` — sequential playback; `insert(_:after:)`, `remove`, `advanceToNextItem`.
- `AVPictureInPictureController` — PiP for video; `canStartPictureInPictureAuto` for auto-PiP on background.

## 6. AVFoundation — editing & export

```swift
// Composition
let composition = AVMutableComposition()
let videoTrack = composition.addMutableTrack(withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid)
let audioTrack = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid)
try videoTrack?.insertTimeRange(CMTimeRange(start: .zero, duration: asset.duration), of: assetTrack, at: .zero)

// Trim/concat
let trimRange = CMTimeRange(start: CMTime(seconds: 5, preferredTimescale: 600),
  duration: CMTime(seconds: 30, preferredTimescale: 600))
try videoTrack?.removeTimeRange(trimRange)

// Export
let export = AVAssetExportSession(asset: composition, presetName: AVAssetExportPresetHighestQuality)!
export.outputURL = outputURL
export.outputFileType = .mp4
export.shouldOptimizeForNetworkUse = true
await export.export()   // async API (iOS 18+)
// or: export.exportAsynchronously { ... }
```

- `AVMutableComposition`/`AVMutableCompositionTrack` — assemble video/audio from multiple assets; `insertTimeRange`, `removeTimeRange`, `scaleTimeRange`.
- `AVAssetReader`/`AVAssetWriter` — low-level frame/sample processing: read `CMSampleBuffer`s, write to new asset; `AVAssetReaderVideoCompositionOutput` for composited video.
- `AVAssetExportSession` — high-level export: presets (`AVAssetExportPresetPassthrough`, `.LowQuality`, `.MediumQuality`, `.HighestQuality`, `.HEVCHighestQuality`, `.AppleM4V1080pHD`, `.AppleM4A` for audio); `outputURL`, `outputFileType` (`.mp4`, `.mov`, `.m4v`, `.m4a`, `.caf`), `shouldOptimizeForNetworkUse`, `timeRange`, `audioMix`, `videoComposition`.
- `AVMutableVideoComposition` — overlays, transitions, cropping, custom compositing via `AVVideoCompositing`.
- `AVMutableAudioMix` — volume ramps, ducking.
- `CIFilter`/`Core Image` — apply filters during export via `AVVideoCompositionCoreAnimationTool` or custom compositor.

## 7. Speech, Translation, NaturalLanguage

### Speech framework
```swift
import Speech

let recognizer = SFSpeechRecognizer(locale: Locale(identifier: "en-US"))!
recognizer.supportsOnDeviceRecognition = true   // check availability
let request = SFSpeechAudioBufferRecognitionRequest()
request.shouldReportPartialResults = true
request.requiresOnDeviceRecognition = true   // privacy: keep on-device

let recognitionTask = recognizer.recognitionTask(with: request) { result, error in
  if let result { print(result.bestTranscription.formattedString) }
}
// Feed audio buffers from AVAudioEngine
let engine = AVAudioEngine()
let input = engine.inputNode
let format = input.outputFormat(forBus: 0)
input.installTap(onBus: 0, bufferSize: 1024, format: format) { buffer, time in
  request.append(buffer)
}
try engine.start()
// Stop: engine.stop(); input.removeTap(onBus: 0); request.endAudio()
```

- `SFSpeechRecognizer` — speech-to-text: `supportsOnDeviceRecognition`, `defaultTaskHint` (.dictation, .search, .confirmation, .unspecified), `queue`.
- `SFSpeechAudioBufferRecognitionRequest` — audio buffer input; `shouldReportPartialResults`, `requiresOnDeviceRecognition`, `addsPunctuation`, `taskHint`, `contextualStrings` (vocabulary hints), `customizedLanguageModel` (iOS 16+).
- `SFSpeechURLRecognitionRequest` — file-based recognition.
- `SFSpeechRecognitionResult` — `bestTranscription` (`SFSpeechRecognitionResult.SpeechTranscription`? actually `SFTranscription` — `formattedString`, `segments` with `substring`, `timestamp`, `duration`, `confidence`), `isFinal`, `speechRecognitionMetadata` (dialogue? iOS 26).
- `SFSpeechRecognitionTask` — `cancel()`, `finish()`, state `.running`, `.canceling`, `.finished`.
- `AVAudioEngine` — real-time audio: `inputNode`, `outputNode`, `mainMixerNode`, `attach`, `connect`, `installTap`, `start`, `stop`, `prepare`.
- `AVAudioPCMBuffer` — audio data: `floatChannelData`, `int16ChannelData`, `frameLength`, `format` (`AVAudioFormat` — sampleRate, channelCount, commonFormat).
- `AVAudioFormat` — `.commonFormat` (`.pcmFormatFloat32`, `.pcmFormatInt16`, `.pcmFormatInt32`), `sampleRate` (44100/48000), `channelCount` (1=mono, 2=stereo).
- **SpeechAnalyzer** (iOS 26+, macOS 26+) — newer API replacing SFSpeechRecognizer path: `SpeechAnalyzer`, `SpeechTranscriber`, `DictationTranscriber`, `SpeechDetector`; module-based (`Speech` framework still works).
- `AVSpeechSynthesizer` — text-to-speech: `AVSpeechUtterance` (`voice`, `rate`, `pitchMultiplier`, `volume`, `prefersAssistiveTechnologySettings`), `AVSpeechSynthesisVoice` (`identifier`, `language`, `quality`); `speak`, `pauseSpeaking`, `continueSpeaking`, `stopSpeaking`, `write` (to buffer for file export).

### Translation framework (iOS 17.4+)
```swift
import Translation
let availability = LanguageAvailability()
if await availability.status(from: Locale.Language(identifier: "en"), to: Locale.Language(identifier: "es")) == .installed {
  let session = TranslationSession(installedSource: Locale.Language(identifier: "en"),
    target: Locale.Language(identifier: "es"))
  let response = try await session.translate("Hello, world!")
  print(response.targetText)
} else {
  // Present TranslationSession.Configuration for download UI
}
```
- `TranslationSession` — on-device translation; `translate(_:)` for text, `translate(batch:)`; requires language packs downloaded via `LanguageAvailability`.
- `LanguageAvailability` — `.status(from:to:)` → `.installed`, `.supported`, `.unsupported`.
- Privacy: fully on-device, no network.

### NaturalLanguage framework
```swift
import NaturalLanguage
let text = "I love this app! It's amazing."
let tokenizer = NLTokenizer(unit: .word)
tokenizer.string = text
for token in tokenizer.tokens(for: text.startIndex..<text.endIndex) {
  print(text[token.range])
}
// Sentiment
let sentiment = try NLTagger(tagSchemes: [.sentimentScore]).tag(at: text.startIndex, unit: .paragraph, scheme: .sentimentScore)
// Language ID
let recognizer = NLLanguageRecognizer()
recognizer.processString(text)
print(recognizer.dominantLanguage)
// Named entity recognition
let tagger = NLTagger(tagSchemes: [.nameType])
tagger.string = text
for tag in tagger.tags(in: text.startIndex..<text.endIndex) { print(tag) }
```
- `NLTokenizer` — word/sentence/paragraph/tokenization units.
- `NLTagger` — schemes: `.sentimentScore` (-1 to 1), `.nameType` (person/place/org), `.lexicalClass` (noun/verb/adj), `.language` (per-token language), `.script`.
- `NLLanguageRecognizer` — language identification; `dominantLanguage`, `languageHypotheses`.
- `NLModel` — custom Core ML-backed NLP models (train with Create ML).

## 8. Vision framework

```swift
import Vision
let request = VNRecognizeTextRequest { request, error in
  guard let observations = request.results as? [VNRecognizedTextObservation] else { return }
  for obs in observations {
    if let candidate = obs.topCandidates(1).first {
      print(candidate.string, candidate.confidence)
    }
  }
}
request.recognitionLevel = .accurate   // or .fast
request.usesLanguageCorrection = true
request.recognitionLanguages = ["en-US", "es-ES"]

let handler = VNImageRequestHandler(cgImage: cgImage, options: [:])
try handler.perform([request])
```

- Requests: `VNRecognizeTextRequest`, `VNDetectFaceRectanglesRequest`, `VNDetectFaceLandmarksRequest`, `VNDetectHumanRectanglesRequest`, `VNDetectHumanBodyPoseRequest`, `VNDetectBarcodesRequest` (QR/UPC/EAN), `VNClassifyImageRequest`, `VNGenerateImageFeaturePrintRequest`, `VNDetectHorizonRequest`, `VNDetectRectanglesRequest`, `VNDetectContoursRequest`, `VNTrackObjectRequest`, `VNRecognizeAnimalsRequest`, `VNDetectDocumentSegmentationRequest`.
- `VNImageRequestHandler` — input: `CGImage`, `CIImage`, `CVPixelBuffer`, `Data` (via `VNImageRequestHandler(data:options:)`), `URL`.
- `VNRecognizedTextObservation` — `topCandidates(_:)` → `VNRecognizedText` (`string`, `confidence`, `stringRange`); bounding boxes via `boundingBox` (normalized, origin bottom-left).
- `VNFaceObservation` — `boundingBox`, `faceCaptureQuality`, `yaw`, `pitch`, `roll`, `landmarks` (`VNFaceLandmarks2D` — eyes, nose, mouth, faceContour).
- `VNHumanBodyPoseObservation` — `recognizedPoints` (`VNHumanBodyPoseObservation.JointName` — nose, eyes, ears, shoulders, elbows, wrists, hips, knees, ankles); confidence per joint.
- `VNBarcodeObservation` — `payloadStringValue`, `symbology` (.qr, .ean13, .upce, .code128, .pdf417, .aztec, .dataMatrix).
- `VNClassificationObservation` — `identifier` (label), `confidence` — uses built-in or custom Core ML models.
- `VNCoreMLRequest` — run custom Core ML models through Vision pipeline.
- `VNImageRequestHandler` orientation: `CGImagePropertyOrientation` — handle EXIF rotation.
- Real-time: process `CMSampleBuffer` from `AVCaptureVideoDataOutput` via `VNImageRequestHandler(cmSampleBuffer:)`.
- macOS: `VNImageRequestHandler` works on `NSImage` via `cgImage`.

## 9. Core ML

```swift
import CoreML
let config = MLModelConfiguration()
config.computeUnits = .all        // .cpuOnly, .cpuAndGPU, .cpuAndNeuralEngine, .all
let model = try MLModel(contentsOf: compiledModelURL, configuration: config)
// or: let model = try VNCoreMLModel(for: MyModel(configuration: .init()).model)

let input = try MLDictionaryFeatureProvider(dictionary: ["image": pixelBuffer])
let output = try model.prediction(from: input)
let label = output.featureValue(for: "label")?.stringValue
```

- Model types: `.mlpackage` (directory with manifest.json + weights), `.mlmodel` (compiled), `.mlmodelc` (compiled directory).
- `MLModel` — `prediction(from:)`, `predictions(from:options:)` (batch), `author`, `configuration`, `modelDescription` (`MLModelDescription` — input/output feature descriptions, metadata).
- `MLFeatureProvider`/`MLDictionaryFeatureProvider` — input container; `MLFeatureValue` types: `.int64`, `.double`, `.string`, `.image` (CVPixelBuffer), `.multiArray` (MLMultiArray), `.dictionary`, `.sequence`.
- `MLMultiArray` — n-dimensional array: `MLMultiArray(shape:dataType:)`; `dataPointer` for direct access.
- Compute units: `.cpuOnly`, `.cpuAndGPU`, `.cpuAndNeuralEngine` (ANE), `.all` (system chooses); profile with Instruments Core ML template.
- Model conversion: `coremltools` (Python) converts PyTorch/TensorFlow/ONNX → `.mlpackage`; Xcode compiles on build.
- Create ML — drag-drop training UI for simple models (image classification, text classification, tabular); generates `.mlmodel`.
- `MLModelCollection`/`MLModelConfiguration` — bundle multiple models.
- App Intents integration: `AppEntity`/`Intent` can call ML models via `AppIntent` `perform()`.
- Privacy: on-device inference; model files in app bundle or downloaded on-demand (ODR); encrypt models with `MLModelConfiguration`? — model encryption via `.mlpackage` + FairPlay? — Core ML supports encrypted models via `MLModelEncryptionKey`? — check current API.

## 10. ARKit

```swift
import ARKit
let config = ARWorldTrackingConfiguration()
config.planeDetection = [.horizontal, .vertical]
config.environmentTexturing = .automatic   // ARCore-like lighting
config.sceneReconstruction = .mesh          // LiDAR mesh (iOS 13.4+)
config.frameSemantics = [.sceneDepth, .smoothedSceneDepth]  // depth maps
config.userFaceTrackingEnabled = true        // front camera AR
arView.session.run(config)

// Anchors
let anchor = ARAnchor(transform: simd_float4x4(...))
arView.session.add(anchor: anchor)
// Plane anchors, image anchors, object anchors, face anchors, collaborative anchors

// Hit testing
let query = arView.raycastQuery(from: screenPoint, allowing: .estimatedPlane, alignment: .horizontal)
if let result = arView.session.raycast(query).first {
  let worldPosition = result.worldTransform.translation
}

// Frame data
arView.session.currentFrame?.camera.intrinsics   // 3x3 calibration
arView.session.currentFrame?.camera.transform    // 4x4 world-from-camera
arView.session.currentFrame?.capturedImage      // CVPixelBuffer
arView.session.currentFrame?.rawFeaturePoints   // sparse points
arView.session.currentFrame?.sceneDepth?.depthMap  // CVPixelBuffer depth
```

- `ARSession`/`ARSCNView`/`ARSKView`/`ARView` (RealityKit) — session management; `run(_:options:)` (`.resetSceneReconstruction`, `.removeExistingAnchors`, `.stopTrackedRaycasts`), `pause()`, `initialWorldMap`/`getCurrentWorldMap` for relocalization.
- `ARWorldTrackingConfiguration` — world tracking: `planeDetection`, `userFaceTrackingEnabled`, `sceneReconstruction` (.mesh, .meshWithClassification), `frameSemantics` (.sceneDepth, .smoothedSceneDepth, .personSegmentationWithDepth), `environmentTexturing`, `lightEstimationEnabled`, `isAutoFocusEnabled`, `videoFormat`, `worldAlignment` (.gravity, .gravityAndHeading, .camera).
- `ARAnchor` subclasses: `ARPlaneAnchor` (extent, center, alignment, classification — wall/floor/ceiling/table/seat/window/door), `ARImageAnchor` (reference image), `ARObjectAnchor` (scanned object), `ARFaceAnchor` (blend shapes, geometry), `ARParticipantAnchor` (multi-user), `ARCollaborationData`.
- `ARRaycastQuery` — `raycast(_:)` returns hits sorted by distance; `.existingPlaneGeometry`, `.estimatedPlane`, `.existingPlaneInfinite`, `.featurePoint`.
- `ARFrame` — `camera` (intrinsics, transform, imageResolution, exposureDuration, exposureOffset), `capturedImage`, `rawFeaturePoints` (`ARPointCloud` — `points` buffer of `vector_float3` + identifiers), `anchors`, `sceneDepth` (`ARDepthData` — `depthMap` CVPixelBuffer, `confidenceMap`, `unprojectPoint`), `segmentationBuffer`.
- `ARReferenceImage` — detect 2-D images: `ARReferenceImage(referenceImageWith:orientation:physicalWidth:)` from CGImage; add to `config.detectionImages`.
- `ARObjectScanningConfiguration` — scan 3-D objects: `createReferenceObject` → `.arobject` file.
- `ARMeshAnchor` — scene reconstruction mesh: `geometry` (`ARMeshGeometry` — vertices, faces, classification), `skeleton`.
- LiDAR: `ARWorldTrackingConfiguration.supportsSceneReconstruction(.mesh)` — iPad Pro/iPhone Pro only.
- `ARQuickLookView`/`ARView` — SwiftUI/RealityKit AR views.
- `ARCoachingOverlayView` — guidance UI for plane detection/focus.

## 11. MapKit

```swift
import MapKit
// SwiftUI
Map(initialPosition: .region(MKCoordinateRegion(center: CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194), span: MKCoordinateSpan(latitudeDelta: 0.05, longitudeDelta: 0.05)))) {
  Marker("San Francisco", coordinate: CLLocationCoordinate2D(latitude: 37.7749, longitude: -122.4194))
  Annotation("Custom", coordinate: coord) { Image(systemName: "star") }
}
.mapStyle(.standard/.imagery/.hybrid)
.userAnnotation()

// UIKit
let mapView = MKMapView()
mapView.delegate = self
mapView.showsUserLocation = true
let annotation = MKPointAnnotation()
annotation.coordinate = coord; annotation.title = "Place"
mapView.addAnnotation(annotation)
mapView.selectAnnotation(annotation, animated: true)
mapView.setRegion(MKCoordinateRegion(center: coord, span: span), animated: true)
mapView.setCamera(MKMapCamera(lookingAtCenter: coord, fromDistance: 1000, pitch: 45, heading: 0), animated: true)

// Search
let request = MKLocalSearch.Request()
request.naturalLanguageQuery = "coffee shops"
request.region = mapView.region
let search = MKLocalSearch(request: request)
search.start { response, error in ... }

// Directions
let directions = MKDirections(request: MKDirections.Request())
directions.calculate { response, error in ... }
```

- `MKMapView` — full map: `mapType` (.standard, .satellite, .hybrid, .mutedStandard), `showsUserLocation`, `userTrackingMode` (.none, .follow, .followWithHeading), `region`, `camera`, `annotations`, `overlays`, `pointOfInterestFilter` (`MKPointOfInterestFilter` — `.includingAll`, `.excludingAll`, `.including(_:categories:)`).
- `MKAnnotation`/`MKPointAnnotation` — `coordinate`, `title`, `subtitle`; `MKAnnotationView`/`MKMarkerAnnotationView` (iOS 11+ — colored markers with glyph).
- `MKOverlay`/`MKOverlayRenderer` — custom overlays: `MKPolyline`, `MKPolygon`, `MKCircle`, custom `MKOverlayPathView`.
- `MKLocalSearch` — POI search: `naturalLanguageQuery`, `region`, `resultTypes` (.pointOfInterest, .address, .query); `MKLocalSearch.Response` → `mapItems` (`MKMapItem` — `name`, `placemark`, `pointOfInterestCategory`, `phoneNumber`, `url`).
- `MKDirections` — route calculation: `MKDirections.Request` (source/destination `MKMapItem`, `transportType` .automobile/.walking/.transit); `MKRoute` (`polyline`, `expectedTravelTime`, `distance`); `MKDirectionsRenderer` for display.
- `MKMapItem` — `openInMaps` for navigation handoff; `MKMapItem.forCurrentLocation()`.
- `MKLookAround`/`MKLookAroundSnapshotter` — Street View equivalent.
- `MKMapSnapshotter` — static map images: `MKMapSnapshotter.Options` (size, scale, region, showsBuildings, pointOfInterestFilter); `MKMapSnapshotter` → `MKMapSnapshotter.Snapshot` (image + pointOfInterestCoordinates).
- `MKGeoJSONDecoder`/`MKGeoJSONEncoder` — GeoJSON import/export.
- SwiftUI `Map` — declarative; `MapReader`/`MapCamera` for camera control; `MapCircle`, `MapPolygon`, `MapPolyline` shapes; `UserAnnotation`.
- Privacy: location permission required for user location; `NSLocationWhenInUseUsageDescription`.

## 12. StoreKit 2 (In-App Purchases)

```swift
import StoreKit

// Fetch products
let products = try await Product.products(for: ["com.app.pro", "com.app.coins100"])
for product in products {
  print(product.displayName, product.displayPrice, product.price, product.type)  // .consumable/.nonConsumable/.autoRenewable/.nonRenewable
}

// Purchase
let result = try await product.purchase()
switch result {
case .success(let verification):
  let transaction = try verification.payloadValue   // VerificationResult<Transaction>
  // Deliver content, then finish:
  await transaction.finish()
case .userCancelled: break
case .pending: break   // Ask to Buy / deferred
@unknown default: break
}

// Listen for transactions (app launch + throughout)
for await result in Transaction.updates {
  if case .verified(let transaction) = result {
    await transaction.finish()
  }
}

// Restore purchases
try await AppStore.sync()
for await result in Transaction.currentEntitlements {
  if case .verified(let transaction) = result { /* has entitlement */ }
}
```

- `Product` — `Product.products(for:)` (async batch fetch), `Product.purchase(options:)` (`.appAccountToken(UUID)`, `.offer` for intro/offer codes); properties: `id`, `displayName`, `displayPrice` (localized), `price` (Decimal), `type` (`Product.ProductType`), `subscription` (`Product.SubscriptionInfo` — `subscriptionGroupID`, `subscriptionPeriod`, `introductoryOffer`, `promotionalOffers`), `priceFormatStyle`.
- `Transaction` — `jwsRepresentation` (signed), `verificationResult` (`.verified(payload)` / `.unverified(payload, error)`); `finish()`, `expirationDate`, `revocationDate`, `offerType`, `offer`.
- `Transaction.updates` — AsyncSequence of new transactions (purchases, renewals, refunds); listen continuously.
- `Transaction.currentEntitlements` — AsyncSequence of active entitlements (non-consumables, active subs).
- `AppStore.sync()` — restore purchases; `AppStore.showManageSubscriptions(in:)` — subscription management UI.
- Subscription offers: `Product.SubscriptionOffer` — `introductoryOffer` (free trial, pay-as-you-go, pay-up-front), `promotionalOffers` (win-back, offer codes); `options:` parameter on `purchase` for offer selection.
- `Product.SubscriptionInfo.RenewalInfo` — `willAutoRenew`, `autoRenewPreference`, `renewalDate`, `gracePeriodExpirationDate`, `priceIncreaseStatus`.
- `Transaction.all` — all transactions (for history/restore); `Transaction.latest(for:)` — most recent for product.
- Win-back offers, offer codes: `AppStore.showOfferCodeRedeemSheet(in:)`.
- App Store Server API — server-side: `AppStoreServerAPI` (Swift) or REST API for webhook notifications (`APP_STORE` notification types V2), refund requests, transaction history.
- Testing: `StoreKitTest` framework — `SKTestSession` with `.storekit` config file; local testing without App Store Connect.
- RevenueCat — third-party wrapper: `Purchases.configure(withAPIKey:)`, `getOfferings`, `purchase`, `restorePurchases`, receipt validation, analytics — handles edge cases, webhooks.

## 13. PDFKit

```swift
import PDFKit
let pdfView = PDFView()
pdfView.document = PDFDocument(url: pdfURL)
pdfView.autoScales = true
pdfView.displayMode = .singlePageContinuous
pdfView.displayDirection = .vertical
pdfView.usePageViewController(false)

// Annotations
let annot = PDFAnnotation(bounds: rect, forType: .highlight, withProperties: nil)
annot.color = .yellow
pdfView.document?.page(at: 0)?.addAnnotation(annot)

// Text extraction
if let text = pdfView.document?.string { ... }
// Search
pdfView.document?.findString("keyword", withOptions: [.caseInsensitive])
```

- `PDFView` — display: `document`, `displayMode` (.singlePage, .singlePageContinuous, .twoUp, .twoUpContinuous), `displayDirection` (.horizontal, .vertical), `autoScales`, `minScale`/`maxScale`, `scaleFactor`, `backgroundColor`, `pageBreakMargins`, `displaysPageBreaks`, `interpolationQuality`, `usePageViewController`.
- `PDFDocument` — model: `pageCount`, `page(at:)` → `PDFPage`, `string`, `findString`, `write(to:)`, `dataRepresentation`, `outlineRoot`, `attributes`, `bookmarks`, `permissions` (`PDFDocumentPermissions` — `.printing`, `.copying`, `.annotating`, `.modifying`, `.extractingForAccessibility`, `.assembling`).
- `PDFPage` — `bounds`, `rotation`, `string`, `selection`, `thumbnail`, `annotations`, `addAnnotation`, `removeAnnotation`, `selection(for:)`.
- `PDFAnnotation` — markup: `.highlight`, `.underline`, `.strikeOut`, `.squiggly`, `.ink`, `.stamp`, `.text`, `.link`, `.popup`, `.line`, `.square`, `.circle`, `.polygon`, `.polyLine`, `.caret`, `.freeText`, `.redact`; `color`, `contents`, `page`, `bounds`, `flags`.
- `PDFSelection` — text selection: `string`, `pages`, `bounds(for:)`.
- `PDFOutline` — document outline/bookmarks: `children`, `label`, `destination`, `action`.
- `PDFAction` — named actions: `.named(.nextPage, .previousPage, .goToFirstPage, .goToLastPage, .zoomIn, .zoomOut)`.
- `PDFThumbnailView` — thumbnail sidebar.
- `PDFKit` on macOS — same API; `PDFView` in `NSScrollView`.

## 14. PencilKit

```swift
import PencilKit
let canvas = PKCanvasView()
canvas.drawingPolicy = .anyInput    // .pencilOnly or .anyInput
canvas.tool = PKInkingTool(.pen, color: .black, width: 5)
canvas.isRulerActive = false
canvas.drawing = PKDrawing()
canvas.drawing.dataRepresentation()   // serialize
canvas.drawing = try PKDrawing(data: data)  // deserialize

// Delegate for drawing events
canvas.drawingDelegate = self   // canvasViewDidBeginUsingTool, didEndUsingTool, etc.
```

- `PKCanvasView` — drawing canvas: `drawing`, `tool` (`PKTool` — `PKInkingTool`, `PKEraserTool`, `PKLassoTool`, `PKInkingTool.InkType` `.pen`, `.pencil`, `.marker`), `drawingPolicy` (`.anyInput`, `.pencilOnly`), `isRulerActive`, `rulerOverlay`, `backgroundColor`, `isOpaque`, `drawingGestureRecognizer`, `onRulerOverlayVisibilityChanged`.
- `PKDrawing` — stroke data: `strokes` (`PKStroke` array), `bounds`, `image(from:scale:)`, `dataRepresentation()`, `init(data:)`, `init(strokes:)`.
- `PKStroke` — `path` (`PKStrokePath` — points, `PKStrokePoint`), `ink` (`PKInk` — `color`, `variety` `.monoline`/`.pen`/`.pencil`/`.watercolor`/`.crayon`, `width`), `startTimestamp`, `endTimestamp`, `altitude`, `azimuth`.
- `PKToolPicker` — system tool palette: `PKToolPicker.shared(for: canvas)` or `PKToolPicker(toolItems:)` for custom; `setVisible`, `selectedToolItem`, `addObserver`, `observedCanvas`.
- `PKToolPickerObserver` — `toolPickerSelectedToolItemDidChange`, `toolPickerVisibilityDidChange`.
- `PKCanvasView` SwiftUI: `PKCanvasViewRepresentable` wrapper needed (no native SwiftUI view).
- `PKStroke` `PKInk` varieties: `.monoline` (uniform width), `.pen` (pressure-sensitive), `.pencil` (tilt-sensitive), `.watercolor` (bleed), `.crayon` (textured).
- iPadOS only (Apple Pencil); macOS supports via tablet input.

## 15. WidgetKit & ActivityKit

### WidgetKit (home screen/lock screen widgets)
```swift
struct MyWidget: Widget {
  var body: some WidgetConfiguration {
    StaticConfiguration(kind: "com.example.widget", provider: Provider()) { entry in
      MyWidgetEntryView(entry: entry)
        .containerBackground(.fill.tertiary, for: .widget)
    }
    .configurationDisplayName("My Widget")
    .description("Shows data.")
    .supportedFamilies([.systemSmall, .systemMedium, .systemLarge, .systemExtraLarge,
      .accessoryCircular, .accessoryRectangular, .accessoryInline])
  }
}

struct Provider: TimelineProvider {
  func placeholder(in context: Context) -> Entry { Entry(date: .now, data: placeholderData) }
  func getSnapshot(in context: Context, completion: @escaping (Entry) -> Void) { ... }
  func getTimeline(in context: Context, completion: @escaping (Timeline<Entry>) -> Void) {
    let entries = [Entry(date: .now, data: data)]
    let timeline = Timeline(entries: entries, policy: .atEnd)  // or .after(date)
    completion(timeline)
  }
}
```

- `Widget` protocol — `WidgetConfiguration`: `StaticConfiguration`, `AppIntentConfiguration` (configurable via intents); `kind` unique ID.
- `TimelineProvider`/`Timeline` — `Entry` (date + data), `Timeline(entries:policy:)` — policy `.atEnd`, `.after(Date)`, `.never`.
- `AppIntentTimelineProvider` — configurable widgets: `IntentConfiguration`/`AppIntentConfiguration` with `AppIntent` for user configuration (e.g. pick a list).
- Families: `.systemSmall`, `.systemMedium`, `.systemLarge`, `.systemExtraLarge` (iPad), `.accessoryCircular`, `.accessoryRectangular`, `.accessoryInline` (watchOS/lock screen), `.accessoryCorner`.
- `containerBackground(_:for:)` — required background for widget content.
- `Link(destination:)` — deep link on tap; `widgetURL(_:)` for whole-widget tap.
- Interactive widgets (iOS 17+): `Button(intent:)`/`Toggle(intent:)` with `AppIntent` — buttons work without opening app.
- `ControlWidget` (iOS 18+) — Control Center controls: `ControlWidget` protocol, `StaticControlConfiguration`, `AppIntentControlConfiguration`.
- `WidgetBundle` — multiple widgets in one extension: `@main struct Widgets: WidgetBundle { var body: some Widget { Widget1(); Widget2() } }`.
- `WidgetCenter` — `WidgetCenter.shared.reloadTimelines(ofKind:)` to force refresh; `reloadAllTimelines()`.
- Deep links: `.widgetURL(URL)` or `Link` — handle in app via `onOpenURL`/`handlesExternalEvents`.

### ActivityKit (Live Activities)
```swift
struct MyAttributes: ActivityAttributes {
  public struct ContentState: Codable, Hashable {
    var progress: Double
    var status: String
  }
  var name: String
}

let attributes = MyAttributes(name: "Workout")
let state = MyAttributes.ContentState(progress: 0.5, status: "Running")
let activity = try Activity.request(attributes: attributes, contentState: state, pushType: .token)

// Update
await activity.update(ActivityContent(state: newState, staleDate: nil))
// or: activity.contentState = newState (iOS 16.2+)

// End
await activity.end(ActivityContent(state: finalState, staleDate: nil), dismissalPolicy: .default)

// Push updates (server → device)
// POST to APNs with apns-push-type: liveactivity, topic: <bundle>.push-type.liveactivity
// payload: { "aps": { "timestamp": ..., "event": "update", "content-state": {...}, "attributes-type": "...", "attributes": {...} } }
// Token: activity.pushTokenStream — send to server
```

- `ActivityAttributes` — static + dynamic data; `ContentState` must be `Codable`.
- `Activity.request` — start Live Activity; `pushType: .token` enables push updates.
- `Activity` — `update`, `end`, `contentState`, `pushTokenStream` (AsyncSequence of tokens), `activityState` (`.active`, `.stale`, `.ended`, `.dismissed`).
- `ActivityContent` — `state`, `staleDate` (when to show stale UI), `relevanceScore`.
- Push-to-start: server can start Live Activity via push (iOS 17.2+) with `event: "start"` + attributes.
- `ActivityConfiguration`/`ActivityFamily` — widget UI for Live Activity (lock screen + Dynamic Island: `.compactLeading`, `.compactTrailing`, `.minimal`, `.expanded`).
- Dynamic Island: `.dynamicIsland` in widget configuration — leading/trailing/center/bottom regions.

## 16. App Intents & Siri AI integration (critical for iOS 27)

```swift
import AppIntents

struct OrderCoffeeIntent: AppIntent {
  static var title: LocalizedStringResource = "Order Coffee"
  static var description = IntentDescription("Order your usual coffee")
  @Parameter(title: "Size") var size: CoffeeSize
  @Parameter(title: "Milk") var milk: Milk?
  static var parameterSummary: some ParameterSummary {
    Summary("Order \(\.$size) coffee") { \.$milk }
  }
  func perform() async throws -> some IntentResult & ProvidesDialog {
    // Do the action
    return .result(dialog: "Ordering \(size) coffee")
  }
}

struct AppShortcuts: AppShortcutsProvider {
  static var appShortcuts: [AppShortcut] {
    AppShortcut(intent: OrderCoffeeIntent(), phrases: ["Order coffee in \(.applicationName)"],
      shortTitle: "Order Coffee", systemImageName: "cup.and.saucer")
  }
}
```

- `AppIntent` — perform actions; `AppEntity` — represent app data for Siri/Spotlight; `AppEnum` — enumerations; `AppShortcutProvider` — donate shortcuts to Siri/Spotlight.
- `@Parameter` — intent inputs: `title`, `description`, `default`, `requestValueDialog`, `inputConnectionBehavior`; supports entities, enums, queries.
- `IntentParameter`/`EntityQuery` — resolve entities from user queries ("Order my usual coffee" → find user's default).
- `IntentResult` — `.result()`, `.result(dialog:)`, `.result(opensIntent:)`, `.result(value:)`.
- `AppEntity` — `defaultQuery`, `EntityStringQuery`, `EntityPropertyQuery` for Spotlight/Siri resolution.
- **Siri AI (iOS 27)**: App Intents are THE integration point — Siri understands app entities, intents, and can chain them; semantic index makes app content discoverable. Apps with rich AppEntity/AppIntent coverage get better Siri AI integration.
- `IndexedEntity` — make app content searchable in Spotlight/Siri: `IndexedEntity` protocol + `CSSearchableItem`.
- `AppIntentsTesting` — test intents: `IntentDefinitions`? — use `AppIntents` testing via `XCTest` + `IntentTest`? — check current testing API.
- Widgets use `AppIntentTimelineProvider` for configuration; interactive widgets use `AppIntent` for button actions.
- `AppShortcutsProvider` — donate phrases for Siri suggestions; `static var appShortcuts`.

## 17. PassKit (Apple Wallet)

```swift
import PassKit
let pass = PKPass(data: passData)   // .pkpass file data
PKAddPassesViewController(pass:)   // add-to-wallet UI
// or: PKAddPassesViewController(packURL:)
let controller = PKAddPassesViewController(packURL: passURL)
present(controller, animated: true)
```

- `PKPass` — `passTypeIdentifier`, `serialNumber`, `authenticationToken`, `webServiceURL`, `isSecureElementPass`, `expirationDate`, `relevantDate`, `locations` (geofence), `beacons`.
- `PKAddPassesViewController` — system UI to add pass; requires `PKPassLibrary`.
- `PKPassLibrary` — `passes()`, `pass(withPassTypeIdentifier:serialNumber:)`, `containsPass`, `removePass`.
- Pass types: `storeCard`, `eventTicket`, `boardingPass`, `coupon`, `generic`, `pkMultiPass`? — `PKPass.PassType`.
- Signing: passes must be signed with Pass Type ID certificate + WWDR cert; `PassTypeID` + `Team ID` in pass.json; manifest.json + signature.
- `PKPaymentPass`? — Apple Pay cards (not for developers to create).

## 18. Core Spotlight

```swift
import CoreSpotlight
import MobileCoreServices

let attributes = CSSearchableItemAttributeSet(contentType: .content)
attributes.title = "Meeting Notes"
attributes.contentDescription = "Q4 planning"
attributes.thumbnailData = image.pngData()

let item = CSSearchableItem(uniqueIdentifier: "note-123", domainIdentifier: "notes",
  attributeSet: attributes)
CSSearchableIndex.default().indexSearchableItems([item]) { error in ... }

// Delete
CSSearchableIndex.default().deleteSearchableItems(withIdentifiers: ["note-123"]) { ... }
```

- `CSSearchableItem` — `uniqueIdentifier` (app-specific ID), `domainIdentifier` (group for batch delete), `attributeSet` (`CSSearchableItemAttributeSet` — title, contentDescription, thumbnailData, keywords, contentURL, relatedUniqueIdentifier, supportedMediaTypes).
- `CSSearchableIndex` — `indexSearchableItems`, `deleteSearchableItems`, `deleteAllSearchableItems`, `deleteSearchableItems(withDomainIdentifiers:)`.
- `NSUserActivity` — donate for Handoff/Siri: `activityType`, `title`, `contentAttributeSet`, `userInfo`, `isEligibleForSearch`, `isEligibleForPrediction`, `eligibleForHandoff`, `requiredUserInfoKeys`; `becomeCurrent()`, `resignCurrent()`, `update(_:)`.
- `CSSearchableItemActionType` — handle tap: `continue userActivity` in scene delegate.
- `IndexedEntity` (iOS 18+) — modern SwiftUI-friendly indexing: conform model to `IndexedEntity`, provide `attributeSet`; system handles indexing.
- Privacy: Spotlight indexing is on-device; no data leaves device.

## 19. Common media/ML bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Camera preview black | session not started on background queue, no input/output | `session.startRunning()` on bg queue; check `canAddInput`/`canAddOutput` |
| Photo capture fails | settings incompatible with session preset | use `photoOutput.availablePhotoCodecTypes`, set `sessionPreset` before adding output |
| Video recording no audio | audio input not added or muted | add `AVCaptureDeviceInput` for `.audio`, check `AVAudioSession` category |
| Vision request returns nothing | wrong orientation, image too small | set `CGImagePropertyOrientation`, check `VNImageRequestHandler` input type |
| Core ML model slow | running on CPU not ANE | set `computeUnits = .cpuAndNeuralEngine` or `.all`; profile |
| Speech recognition fails | no permission, on-device not supported | check `SFSpeechRecognizer.authorizationStatus()`, `supportsOnDeviceRecognition` |
| Live Activity not updating | push token not sent to server, wrong topic | use `activity.pushTokenStream`, send to server, topic = `<bundle>.push-type.liveactivity` |
| Widget not refreshing | timeline policy too infrequent, no reload | use `.atEnd` for next refresh, `WidgetCenter.reloadTimelines` for manual |
| AR plane detection flaky | poor lighting, featureless surfaces | use `ARCoachingOverlayView`, check `ARWorldTrackingStatus` |
| StoreKit purchase stuck pending | Ask to Buy, payment issue | handle `.pending` case, listen to `Transaction.updates` |
| Map not showing user location | missing permission or `showsUserLocation` false | request `CLLocationManager` auth, set `showsUserLocation = true` |
| PDF text extraction empty | scanned PDF (image-based) | use OCR via Vision `VNRecognizeTextRequest` on rendered pages |

## 20. Performance tips

- **AVFoundation**: use `AVCaptureSessionPreset` appropriately; `videoDataOutput.alwaysDiscardsLateVideoFrames = true` for realtime; process frames on dedicated serial queue; downsample before ML inference.
- **Metal**: profile with Metal System Trace; minimize pipeline state changes; use argument buffers; `MTLHeap` for resource reuse; `IOAccel` family instruments.
- **Core Image**: `CIContext` reuse (expensive to create); `CIContext(options: [.useSoftwareRenderer: false])` for GPU; `CIFilter` chain optimization — render once to `CIImage` then display.
- **Vision**: batch requests where possible; use `.fast` recognition level for realtime; downsample input images; `VNImageRequestHandler` reuse.
- **Core ML**: `.mlpackage` + ANE; quantize models (8-bit/16-bit) for size/speed; `MLModelConfiguration.computeUnits`; warm up model before first inference.
- **ARKit**: `frameSemantics` only what you need; `sceneReconstruction` is expensive — enable only when needed; `ARWorldTrackingConfiguration` vs `AROrientationTrackingConfiguration` for simpler use cases.
- **WidgetKit**: keep timeline entries small; use `TimelineProvider` caching; `WidgetCenter.reloadTimelines` sparingly (rate-limited by system).
- **StoreKit**: cache `Product.products` results; `Transaction.updates` listener must be early in app lifecycle.
- **MapKit**: `MKMapSnapshotter` for static images (faster than live map); `MKMapView` reuse; annotation view reuse via `dequeueReusableAnnotationView`.
