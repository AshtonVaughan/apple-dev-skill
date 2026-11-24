# AVFoundation Deep Reference

Low-level media capture, playback, editing, and processing. Read `08-graphics-media-ml.md` §4-6 for basics.

## 1. Core Media primitives

- `CMTime` — time representation: `CMTime(seconds:preferredTimescale:)`, `CMTime.zero`, `CMTime.indefinite`, `CMTime.invalid`, `CMTime(seconds:)`; `CMTimeRange(start:duration:)`, `CMTimeRange.zero`, `CMTimeRangeMake`, `CMTimeRangeGetIntersection`, `CMTimeRangeContainsTime`, `CMTimeRangeEqual`, `CMTimeRangeGetUnion`, `CMTimeRangeGetEnd`, `CMTimeRangeGetDuration`, `CMTimeGetSeconds`, `CMTimeCompare`, `CMTimeAdd`, `CMTimeSubtract`, `CMTimeMultiply`, `CMTimeMultiplyByFloat`, `CMTimeMultiplyByRatio`, `CMTimeMakeWithSeconds`, `CMTimeMakeWithEpoch`, `CMTimeConvertScale`, `CMTimeGetMinimum`, `CMTimeGetMaximum`, `CMTimeRangeMake`.
- `CMTimebase` — timebase for sample timing: `CMTimebaseCreateWithSourceClock`, `CMTimebaseGetTime`, `CMTimebaseGetRate`, `CMTimebaseSetTime`, `CMTimebaseSetRate`.
- `CMClock` — clock source: `CMClockGetTime`, `CMClockGetHostTimeClock`.
- `CMSampleBuffer` — media sample: `CMSampleBufferCreate`, `CMSampleBufferGetImageBuffer` (CVPixelBuffer), `CMSampleBufferGetAudioBufferList`, `CMSampleBufferGetOutputPresentationTimeStamp`, `CMSampleBufferGetDuration`, `CMSampleBufferGetNumSamples`, `CMSampleBufferGetSampleTimingInfoArray`, `CMSampleBufferGetFormatDescription`, `CMSampleBufferGetDataBuffer`, `CMSampleBufferCreateCopyWithNewTiming`, `CMSampleBufferCreateReady`, `CMSampleBufferInvalidate`, `CMSampleBufferSetSampleAttachment`.
- `CMSampleTimingInfo` — `duration`, `presentationTimeStamp`, `decodeTimeStamp`.
- `CMFormatDescription` — media format: `CMFormatDescriptionCreate`, `CMFormatDescriptionGetMediaType`, `CMFormatDescriptionGetMediaSubType`, `CMFormatDescriptionGetExtensions`.
- `CMVideoFormatDescription` — video format: `CMVideoFormatDescriptionCreate`, `CMVideoFormatDescriptionGetDimensions`, `CMVideoFormatDescriptionGetCodecType`.
- `CMAudioFormatDescription` — audio format: `CMAudioFormatDescriptionCreate`, `CMAudioFormatDescriptionGetStreamBasicDescription`.
- `CVPixelBuffer` — pixel buffer: `CVPixelBufferCreate`, `CVPixelBufferLockBaseAddress`, `CVPixelBufferGetBaseAddress`, `CVPixelBufferGetBytesPerRow`, `CVPixelBufferGetWidth`, `CVPixelBufferGetHeight`, `CVPixelBufferGetPixelFormatType`, `CVPixelBufferPoolCreate`.
- `CVPixelBufferPool` — pixel buffer pool for reuse: `CVPixelBufferPoolCreatePixelBuffer`, `CVPixelBufferPoolFlush`.
- `CMBlockBuffer` — block buffer: `CMBlockBufferCreateWithMemoryBlock`, `CMBlockBufferGetDataPointer`.
- `CMMetadata` — metadata: `CMMetadataCreate`, `CMMetadataGetNumberOfItems`, `CMMetadataGetItem`.
- `CMTimeMapping` — time mapping: `CMTimeMappingMake`, `CMTimeMappingGetSourceTimeRangeAtTime`.
- `CGAffineTransform` — affine transform: `CGAffineTransformMakeTranslation`, `CGAffineTransformMakeRotation`, `CGAffineTransformMakeScale`, `CGAffineTransformConcat`, `CGAffineTransformInvert`.

## 2. Asset reading & writing

### AVAssetReader/Writer
```swift
let asset = AVURLAsset(url: videoURL)
let reader = try AVAssetReader(asset: asset)
let writer = try AVAssetWriter(outputURL: outputURL, fileType: .mp4)

// Video track
let videoTrack = asset.tracks(withMediaType: .video).first!
let readerOutput = AVAssetReaderTrackOutput(track: videoTrack, outputSettings: [
  kCVPixelBufferPixelFormatTypeKey as String: kCVPixelFormatType_32BGRA
])
reader.add(readerOutput)

let writerInput = AVAssetWriterInput(mediaType: .video, outputSettings: [
  AVVideoCodecKey: AVVideoCodecType.h264,
  AVVideoWidthKey: 1920,
  AVVideoHeightKey: 1080
])
writer.add(writerInput)

// Audio track
let audioTrack = asset.tracks(withMediaType: .audio).first!
let audioReaderOutput = AVAssetReaderTrackOutput(track: audioTrack, outputSettings: [
  AVFormatIDKey: kAudioFormatMPEG4AAC,
  AVSampleRateKey: 44100,
  AVNumberOfChannelsKey: 2,
  AVEncoderBitRateKey: 128000
])
reader.add(audioReaderOutput)

let audioWriterInput = AVAssetWriterInput(mediaType: .audio, outputSettings: [
  AVFormatIDKey: kAudioFormatMPEG4AAC,
  AVSampleRateKey: 44100,
  AVNumberOfChannelsKey: 2,
  AVEncoderBitRateKey: 128000
])
writer.add(audioWriterInput)

writer.startWriting()
writer.startSession(atSourceTime: .zero)
reader.startReading()

while let sampleBuffer = readerOutput.copyNextSampleBuffer() {
  while !writerInput.isReadyForMoreMediaData { Thread.sleep(forTimeInterval: 0.01) }
  writerInput.append(sampleBuffer)
}
while let sampleBuffer = audioReaderOutput.copyNextSampleBuffer() {
  while !audioWriterInput.isReadyForMoreMediaData { Thread.sleep(forTimeInterval: 0.01) }
  audioWriterInput.append(sampleBuffer)
}
writerInput.markAsFinished()
audioWriterInput.markAsFinished()
await writer.finishWriting()
```

- `AVAssetReader` — read media: `add(_:)`, `startReading()`, `cancelReading()`, `status` (`.unknown`, `.reading`, `.completed`, `.failed`, `.cancelled`), `error`.
- `AVAssetReaderTrackOutput` — track output: `copyNextSampleBuffer()`, `alwaysCopiesSampleData`, `outputSettings`.
- `AVAssetReaderVideoCompositionOutput` — composited video: `videoComposition`, `customVideoCompositorClass`.
- `AVAssetReaderOutputMetadataAdaptor` — metadata output: `metadataItems(for:)`, `append(_:with:)`.
- `AVAssetWriter` — write media: `add(_:)`, `startWriting()`, `startSession(atSourceTime:)`, `finishWriting()`, `cancelWriting()`, `status`, `error`, `outputURL`, `fileType`, `shouldOptimizeForNetworkUse`, `metadata`.
- `AVAssetWriterInput` — writer input: `isReadyForMoreMediaData`, `append(_:)`, `markAsFinished()`, `expectsMediaDataInRealTime`, `transform`.
- `AVAssetWriterInputPixelBufferAdaptor` — pixel buffer adaptor: `pixelBufferPool`, `append(_:withPresentationTime:)`.
- `AVAssetWriterInputMetadataAdaptor` — metadata adaptor: `append(_:with:)`.
- `AVAssetWriterInputGroup` — grouped inputs (e.g. video+audio).
- `AVAssetWriter` `outputFileType` — `.mp4`, `.mov`, `.m4v`, `.m4a`, `.caf`, `.wav`, `.aiff`, `.3gp`, `.3g2`, `.amr`.

### AVComposition & editing
```swift
let composition = AVMutableComposition()
let videoTrack = composition.addMutableTrack(withMediaType: .video, preferredTrackID: kCMPersistentTrackID_Invalid)!
let audioTrack = composition.addMutableTrack(withMediaType: .audio, preferredTrackID: kCMPersistentTrackID_Invalid)!

// Insert time range
let asset = AVURLAsset(url: url)
let assetVideoTrack = asset.tracks(withMediaType: .video).first!
let timeRange = CMTimeRange(start: .zero, duration: asset.duration)
try videoTrack.insertTimeRange(timeRange, of: assetVideoTrack, at: .zero)
try audioTrack.insertTimeRange(timeRange, of: asset.tracks(withMediaType: .audio).first!, at: .zero)

// Trim
do {
  let removeRange = CMTimeRange(start: CMTime(seconds: 5, preferredTimescale: 600),
    duration: CMTime(seconds: 10, preferredTimescale: 600))
  try videoTrack.removeTimeRange(removeRange)
  try audioTrack.removeTimeRange(removeRange)
}

// Scale time range
try videoTrack.scaleTimeRange(CMTimeRange(start: .zero, duration: asset.duration),
  toDuration: CMTime(seconds: 30, preferredTimescale: 600))

// Export
let export = AVAssetExportSession(asset: composition, presetName: AVAssetExportPresetHighestQuality)!
export.outputURL = outputURL
export.outputFileType = .mp4
export.shouldOptimizeForNetworkUse = true
export.timeRange = CMTimeRange(start: .zero, duration: composition.duration)
await export.export()   // iOS 18+ async
// or: export.exportAsynchronously { ... }
```

- `AVMutableComposition`/`AVMutableCompositionTrack` — `insertTimeRange`, `removeTimeRange`, `scaleTimeRange`, `preferredTransform`, `naturalTimeScale`.
- `AVCompositionTrack` — `assetTrack`, `timeRange`, `preferredTransform`.
- `AVMutableVideoComposition` — `instructions` (`AVMutableVideoCompositionInstruction`), `renderSize`, `frameDuration`, `colorPrimaries`, `colorTransferFunction`, `colorYCbCrMatrix`, `animationTool`.
- `AVMutableVideoCompositionInstruction` — `timeRange`, `layerInstructions` (`AVMutableVideoCompositionLayerInstruction`), `backgroundColor`, `enablePostProcessing`, `containsTweening`, `passthroughTrackID`.
- `AVMutableVideoCompositionLayerInstruction` — `trackID`, `setTransform`, `setOpacity`, `setCropRectangle`.
- `AVVideoCompositing` protocol — custom compositing: `sourcePixelBufferAttributes`, `requiredPixelBufferAttributesForRenderContext`, `renderContextChanged`, `startRequest`, `cancelAllPendingVideoCompositionRequests`, `sourceFrameByTrackID`.
- `AVMutableAudioMix` — `inputParameters` (`AVMutableAudioMixInputParameters`), `volumeRamp`, `setVolume`, `setVolumeRamp`.
- `AVMutableAudioMixInputParameters` — `audioTracks`, `setVolume`, `setVolumeRamp`.
- `AVAssetExportSession` — `presetName` (`AVAssetExportPresetPassthrough`, `.LowQuality`, `.MediumQuality`, `.HighestQuality`, `.HEVCHighestQuality`, `.AppleM4V1080pHD`, `.AppleM4A`), `outputURL`, `outputFileType`, `shouldOptimizeForNetworkUse`, `timeRange`, `audioMix`, `videoComposition`, `metadata`, `fileLengthLimit`.

## 3. Playback deep

### AVPlayer & observers
```swift
let player = AVPlayer(url: url)
player.actionAtItemEnd = .pause   // or .none, .advance
player.automaticallyWaitsToMinimizeStalling = true
player.appliesMediaSelectionCriteriaAutomatically = true
player.isMuted = false
player.volume = 0.5
player.rate = 1.0

// Time observers
let interval = CMTime(seconds: 0.1, preferredTimescale: 600)
let observer = player.addPeriodicTimeObserver(forInterval: interval, queue: .main) { time in
  print("Current time: \(CMTimeGetSeconds(time))")
}
player.removeTimeObserver(observer)

// Boundary observers
let boundary = CMTime(seconds: 30, preferredTimescale: 600)
let boundaryObserver = player.addBoundaryTimeObserver(forTimes: [NSValue(time: boundary)], queue: .main) { }
player.removeTimeObserver(boundaryObserver)

// Item status
let item = player.currentItem!
item.status = .readyToPlay   // KVO
item.presentationSize = CGSize(width: 1920, height: 1080)
item.duration = CMTime(seconds: 120, preferredTimescale: 600)
item.loadedTimeRanges = [NSValue(timeRange: CMTimeRange(start: .zero, duration: CMTime(seconds: 30, preferredTimescale: 600)))]
item.seekableTimeRanges = [NSValue(timeRange: CMTimeRange(start: .zero, duration: CMTime(seconds: 120, preferredTimescale: 600)))]
item.tracks = [AVPlayerItemTrack]
item.asset = asset
item.error = nil
item.isPlaybackLikelyToKeepUp = true
item.isPlaybackBufferEmpty = false
item.isPlaybackBufferFull = true
```

- `AVPlayer` — `play()`, `pause()`, `rate`, `volume`, `muted`, `actionAtItemEnd`, `automaticallyWaitsToMinimizeStalling`, `appliesMediaSelectionCriteriaAutomatically`, `currentItem`, `currentTime()`, `seek(to:toleranceBefore:toleranceAfter:)`, `seek(to:completionHandler:)`, `addPeriodicTimeObserver`, `addBoundaryTimeObserver`, `removeTimeObserver`, `replaceCurrentItem(with:)`, `insert(_:)` (queue), `advanceToNextItem()`, `canPlayReverse`, `defaultRate`, `reasonForWaitingToPlay`, `timeControlStatus`.
- `AVPlayerItem` — `status`, `error`, `duration`, `currentTime`, `seekableTimeRanges`, `loadedTimeRanges`, `presentationSize`, `tracks`, `asset`, `isPlaybackLikelyToKeepUp`, `isPlaybackBufferEmpty`, `isPlaybackBufferFull`, `canPlayFastForward`, `canPlaySlowForward`, `canPlayReverse`, `audioTimePitchAlgorithm`, `preferredForwardBufferDuration`, `preferredPeakBitRate`.
- `AVPlayerItemTrack` — `assetTrack`, `isEnabled`, `isPlaybackLikelyToKeepUp`, `isPlaybackBufferEmpty`, `isPlaybackBufferFull`, `estimatedDataRate`, `preferredFrameRate`.
- `AVQueuePlayer` — `insert(_:after:)`, `remove`, `advanceToNextItem`, `items()`.
- `AVPlayerLooper` — `init(player:templateItem:timeRange:)`, `disableLooping()`, `timeRange`.
- `AVPlayerViewController` — system player UI: `player`, `showsPlaybackControls`, `videoGravity`, `allowsPictureInPicturePlayback`, `canStartPictureInPictureAutomaticallyFromInline`, `entersFullScreenWhenPlaybackBegins`.
- `AVPictureInPictureController` — `playerLayer`, `canStartPictureInPictureAutomaticallyFromInline`, `startPictureInPicture`, `stopPictureInPicture`, `delegate`.
- `AVAudioSession` — `category`, `mode`, `options`, `setActive`, `overrideOutputAudioPort`, `interruptionNotification`, `routeChangeNotification`, `mediaServicesWereResetNotification`.
- `AVAudioSession.Category` — `.ambient`, `.soloAmbient`, `.playback`, `.record`, `.playAndRecord`, `.multiRoute`.
- `AVAudioSession.Mode` — `.default`, `.measurement`, `.moviePlayback`, `.spokenAudio`, `.videoRecording`, `.voiceChat`, `.gameChat`.
- `AVAudioSession.Options` — `.mixWithOthers`, `.duckOthers`, `.allowBluetooth`, `.allowBluetoothA2DP`, `.allowAirPlay`, `.interruptSpokenAudioAndMixWithOthers`, `.overrideMutedMicrophoneInterruption`? — check.
- `AVAudioSession.RouteChangeReason` — `.newDeviceAvailable`, `.oldDeviceUnavailable`, `.categoryChange`, `.override`, `.wakeFromSleep`, `.noSuitableRouteForCategory`, `.routeConfigurationChange`.
- `AVAudioSession.InterruptionType` — `.began`, `.ended`.
- `AVAudioSession.InterruptionOption` — `.shouldResume`.
- `AVAudioSession.MediaServicesWereResetNotification` — recreate objects.
- `AVAudioSession.silenceSecondaryAudioHintNotification` — silence hint.
- `AVAudioSession.routeChangeNotification` — route change.
- `AVAudioSession.interruptionNotification` — interruption.
- `AVAudioSession.mediaServicesWereResetNotification` — media services reset.
- `AVAudioSession.silenceSecondaryAudioHintNotification` — silence hint.

## 4. Capture deep

### AVCaptureSession configuration
```swift
let session = AVCaptureSession()
session.beginConfiguration()

// Session preset or manual format
session.sessionPreset = .high
// OR manual:
let camera = AVCaptureDevice.default(.builtInWideAngleCamera, for: .video, position: .back)!
let format = camera.formats.first { format in
  let dims = CMVideoFormatDescriptionGetDimensions(format.formatDescription)
  return dims.width == 3840 && dims.height == 2160 &&
    format.videoSupportedFrameRateRanges.contains { $0.maxFrameRate >= 60 }
}!
try camera.lockForConfiguration()
camera.activeFormat = format
camera.activeVideoMinFrameDuration = CMTime(seconds: 1.0/60.0, preferredTimescale: 600)
camera.activeVideoMaxFrameDuration = CMTime(seconds: 1.0/60.0, preferredTimescale: 600)
camera.unlockForConfiguration()

// Inputs
let input = try AVCaptureDeviceInput(device: camera)
if session.canAddInput(input) { session.addInput(input) }

// Outputs
let photoOutput = AVCapturePhotoOutput()
if session.canAddOutput(photoOutput) { session.addOutput(photoOutput) }

session.commitConfiguration()
DispatchQueue.global(qos: .userInitiated).async { session.startRunning() }
```

- `AVCaptureSession` — `sessionPreset` (`.high`, `.medium`, `.photo`, `.inputPriority`, `.hd4K3840x2160`, `.hd1920x1080`, `.vga640x480`, `.cif352x288`, `.low`, `.1280x720`), `beginConfiguration`/`commitConfiguration`, `canAddInput`/`canAddOutput`, `inputs`, `outputs`, `connections`, `startRunning`, `stopRunning`, `isRunning`, `isInterrupted`, `interruptionReason`, `runtimeError`, `multitaskingCameraAccessEnabled`, `automaticallyConfiguresApplicationAudioSession`, `automaticallyConfiguresCaptureDeviceForWideColor`, `usesApplicationAudioSession`, `videoStabilizationEnabled`? — check.
- `AVCaptureDevice` — `default(_:for:position:)`, `devices(for:)`, `deviceType`, `position`, `formats`, `activeFormat`, `activeVideoMinFrameDuration`, `activeVideoMaxFrameDuration`, `isFocusModeSupported`, `focusMode`, `focusPointOfInterest`, `isExposureModeSupported`, `exposureMode`, `exposurePointOfInterest`, `isWhiteBalanceModeSupported`, `whiteBalanceMode`, `deviceWhiteBalanceGains`, `setWhiteBalanceModeLocked`, `torchMode`, `torchLevel`, `videoZoomFactor`, `isSmoothAutoFocusEnabled`, `isLowLightBoostEnabled`, `isGeometricDistortionCorrectionEnabled`, `isVirtualDevice`, `constituentDevices`, `formats`, `activeDepthDataFormat`, `activeFormat`, `activeVideoMinFrameDuration`, `activeVideoMaxFrameDuration`.
- `AVCaptureDevice.Format` — `formatDescription`, `mediaType`, `videoSupportedFrameRateRanges`, `videoFieldOfView`, `videoMaxZoomFactor`, `isVideoBinned`, `isVideoHDR`, `isVideoStabilizationModeSupported`, `isDepthDataSupported`? — check.
- `AVCaptureDeviceInput`/`AVCaptureOutput` — `connection(with:)`, `connections`.
- `AVCaptureConnection` — `videoOrientation`, `videoMirroring`, `isVideoMirrored`, `isVideoOrientationSupported`, `isVideoMirroringSupported`, `videoMinFrameDuration`, `videoMaxFrameDuration`, `isVideoStabilizationModeSupported`, `videoStabilizationMode`.
- `AVCapturePhotoOutput` — `availablePhotoCodecTypes`, `availablePhotoFileTypes`, `maxPhotoQualityPrioritization`, `photoQualityPrioritization`, `isHighResolutionCaptureEnabled`, `isZeroShutterLagEnabled`, `isLivePhotoCaptureSupported`, `isLivePhotoCaptureEnabled`, `isDepthDataDeliveryEnabled`, `isPortraitEffectsMatteDeliveryEnabled`, `isAppleProRAWEnabled`, `maxPhotoDimensions`.
- `AVCapturePhotoSettings` — `flashMode`, `photoQualityPrioritization`, `maxPhotoDimensions`, `livePhotoMovieFileURL`, `isHighResolutionPhotoEnabled`, `isAutoDeferredPhotoDeliveryEnabled`, `isPortraitEffectsMatteDeliveryEnabled`, `isDepthDataDeliveryEnabled`, `isAppleProRAWEnabled`, `processedFileType`, `isFastCapturePrioritizationEnabled`, `isResponsiveCaptureEnabled`.
- `AVCaptureMovieFileOutput` — `startRecording(to:recordingDelegate:)`, `stopRecording()`, `isRecording`, `recordedDuration`, `recordedFileSize`, `minFreeDiskSpaceLimit`, `maxRecordedDuration`, `maxRecordedFileSize`, `movieFragmentInterval`, `outputSettings`, `connections`.
- `AVCaptureVideoDataOutput` — `videoSettings`, `alwaysDiscardsLateVideoFrames`, `deliversVideoFrameQualityMetadata`, `setSampleBufferDelegate(_:queue:)`, `sampleBufferDelegateQueue`, `connections`.
- `AVCaptureAudioDataOutput` — `audioSettings`, `setSampleBufferDelegate(_:queue:)`, `sampleBufferDelegateQueue`, `connections`.
- `AVCaptureMetadataOutput` — `metadataObjectTypes`, `setMetadataObjectsDelegate(_:queue:)`, `metadataObjectsDelegateQueue`, `connections`.
- `AVCaptureDepthDataOutput` — `depthDataCallbackQueue`, `delegate`, `isFilteringEnabled`, `isDepthDataDeliveryEnabled`, `connections`.
- `AVCaptureLiDARDepthDataOutput` — `depthDataCallbackQueue`, `delegate`, `connections`.
- `AVCaptureDeviceFormat`? — `AVCaptureDevice.Format`.
- `AVCaptureVideoPreviewLayer` — `session`, `videoGravity`, `connection`, `automaticallyAdjustsMirroring`.
- `AVCaptureSession` multi-cam: `AVCaptureMultiCamSession` — simultaneous front+back.
- `AVCaptureSession` runtime errors: `AVCaptureSessionRuntimeError` — `.mediaServicesWereReset`, `.sessionWasInterrupted`, `.sessionInterruptionReason`.
- `AVCaptureSessionInterruptionReason` — `.videoDeviceNotAvailableInBackground`, `.audioDeviceInUseByAnotherClient`, `.videoDeviceInUseByAnotherClient`, `.videoDeviceNotAvailableWithMultipleForegroundApps`, `.videoDeviceNotAvailableDueToSystemPressure`.
- `AVCaptureSession` `isInterrupted` — check interruption state.
- `AVCaptureSession` `interruptionReason` — reason for interruption.
- `AVCaptureSession` `runtimeError` — runtime error.
- `AVCaptureSession` `automaticallyConfiguresApplicationAudioSession` — auto audio session.
- `AVCaptureSession` `automaticallyConfiguresCaptureDeviceForWideColor` — auto wide color.
- `AVCaptureSession` `usesApplicationAudioSession` — use app audio session.
- `AVCaptureSession` `multitaskingCameraAccessEnabled` — PiP camera.
- `AVCaptureSession` `isMultitaskingCameraAccessSupported` — PiP support.
- `AVCaptureSession` `videoStabilizationEnabled`? — check.
- `AVCaptureSession` `automaticallyConfiguresApplicationAudioSession` — auto audio session.
- `AVCaptureSession` `automaticallyConfiguresCaptureDeviceForWideColor` — auto wide color.
- `AVCaptureSession` `usesApplicationAudioSession` — use app audio session.
- `AVCaptureSession` `multitaskingCameraAccessEnabled` — PiP camera.
- `AVCaptureSession` `isMultitaskingCameraAccessSupported` — PiP support.
- `AVCaptureSession` `videoStabilizationEnabled`? — check.

## 5. Common AVFoundation bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Camera preview black | session not started on background queue, no input/output | `session.startRunning()` on bg queue; check `canAddInput`/`canAddOutput` |
| Photo capture fails | settings incompatible with session preset | use `photoOutput.availablePhotoCodecTypes`, set `sessionPreset` before adding output |
| Video recording no audio | audio input not added or muted | add `AVCaptureDeviceInput` for `.audio`, check `AVAudioSession` category |
| Player stalls | network slow, buffer empty | `automaticallyWaitsToMinimizeStalling = true`, `preferredForwardBufferDuration` |
| Audio session conflict | wrong category, not handling interruptions | set category/mode, observe `interruptionNotification`/`routeChangeNotification` |
| Video orientation wrong | `videoOrientation` not set | set `connection.videoOrientation` or `AVPlayerLayer.videoGravity` |
| Export fails | invalid time range, codec unsupported | check `timeRange`, `outputFileType`, preset compatibility |
| Memory spike on large video | `AVPlayerItem` loading all into memory | use `AVAssetReader` streaming, `AVPlayer` handles streaming |
| Picture in Picture fails | `AVPictureInPictureController` not configured | check `playerLayer`, `canStartPictureInPictureAutomaticallyFromInline` |
| Subtitle not showing | wrong media selection group | use `asset.mediaSelectionGroup(forMediaCharacteristic: .legible)` |
| AirPlay not working | wrong audio session category | set `AVAudioSession.Category.playback`, check `routeChangeNotification` |
| Background audio stops | missing `audio` background mode | add `UIBackgroundModes` = `audio`, keep `AVAudioSession` active |
| Thermal throttling | sustained capture/processing | check `ProcessInfo.thermalState`, reduce quality/fps |
| Multi-cam fails | not using `AVCaptureMultiCamSession` | use `AVCaptureMultiCamSession` for simultaneous front+back |
| Depth data unavailable | device doesn't support, wrong format | check `AVCaptureDevice.Format.isDepthDataSupported`? — verify |
| Object tracking fails | no metadata output, wrong object type | add `AVCaptureMetadataOutput`, set `metadataObjectTypes` |
