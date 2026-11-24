# Metal, RealityKit, SceneKit & SpriteKit — Deep Reference

GPU programming and high-performance rendering for Apple platforms.

## 1. Metal — the GPU API

### Core objects
```swift
import Metal

let device = MTLCreateSystemDefaultDevice()!        // MTLDevice — GPU access
let queue = device.makeCommandQueue()!               // MTLCommandQueue — command submission
let library = device.makeDefaultLibrary()!           // MTLLibrary — compiled shaders
let vertexFunc = library.makeFunction(name: "vertexShader")!
let fragmentFunc = library.makeFunction(name: "fragmentShader")!

let pipelineDesc = MTLRenderPipelineDescriptor()
pipelineDesc.vertexFunction = vertexFunc
pipelineDesc.fragmentFunction = fragmentFunc
pipelineDesc.colorAttachments[0].pixelFormat = .bgra8Unorm
let pipeline = try device.makeRenderPipelineState(descriptor: pipelineDesc)
```

- `MTLDevice` — factory for all GPU objects: `makeCommandQueue`, `makeBuffer`, `makeTexture`, `makeRenderPipelineState`, `makeComputePipelineState`, `makeHeap`, `makeArgumentEncoder`, `makeFence`, `makeSharedEvent`, `makeCounterSampleBuffer`, `makeIOHandle`, `makeResourceStateCommandEncoder`.
- `MTLCommandQueue` — `makeCommandBuffer()` → `MTLCommandBuffer`; `insertDebugCaptureBoundary`, `commandBufferDidComplete`? — use completion handlers.
- `MTLCommandBuffer` — `makeRenderCommandEncoder`, `makeComputeCommandEncoder`, `makeBlitCommandEncoder`, `present(_:)`, `commit()`, `addCompletedHandler`, `label`.
- `MTLRenderCommandEncoder` — `setRenderPipelineState`, `setVertexBuffer(_:offset:at:)`, `setVertexBytes(_:length:at:)`, `setFragmentBuffer`, `setVertexTexture`, `setFragmentTexture`, `setVertexSamplerState`, `setFragmentSamplerState`, `drawPrimitives(type:vertexStart:vertexCount:)`, `drawIndexedPrimitives`, `drawPrimitivesInstanced`, `setViewport`, `setScissorRect`, `setBlendColor`, `setDepthStencilState`, `setCullMode`, `setTriangleFillMode`, `memoryBarrier`.
- `MTLComputeCommandEncoder` — `setComputePipelineState`, `setBuffer`, `setTexture`, `setSamplerState`, `dispatchThreads(threadgroupsPerGrid:threadsPerThreadgroup:)`, `dispatchThreadgroups`, `memoryBarrier`.
- `MTLBlitCommandEncoder` — `copy(from:sourceOffset:to:destinationOffset:size:)`, `generateMipmaps`, `fill`, `synchronize`.
- `MTLTexture`/`MTLBuffer` — GPU memory; `replaceRegion`, `getBytes`, `contents()`, `newTextureView`, `newBufferWithBytes`? — `device.makeBuffer(bytes:length:options:)`.
- `MTLSamplerState` — texture sampling: `minFilter`, `magFilter`, `mipFilter`, `sAddressMode`, `tAddressMode`, `rAddressMode`, `maxAnisotropy`.
- `MTLDepthStencilState` — depth/stencil testing: `depthCompareFunction`, `isDepthWriteEnabled`, `frontFaceStencil`, `backFaceStencil`.
- `MTLRenderPassDescriptor` — render target setup: `colorAttachments[i]` (`texture`, `loadAction`, `storeAction`, `clearColor`), `depthAttachment`, `stencilAttachment`.
- `MTLVertexDescriptor` — vertex layout: `attributes[i]` (format, offset, bufferIndex), `layouts[i]` (stride, stepFunction, stepRate).
- `MTLArgumentEncoder` — argument buffers: `setArgumentBuffer(_:offset:atIndex:)`.
- `MTLIndirectCommandBuffer` — indirect command buffers (Metal 3): `reset`, `setKernelBuffer`, `executeCommandsInBuffer`.
- `MTLMeshRenderCommandEncoder` — mesh shaders (Metal 3): `setRenderPipelineState`, `drawMeshThreadgroups`, `drawMeshThreads`.
- `MTLRaytracingPipeline` — ray tracing (Metal 3, A-series/M-series).
- `MTLFence`/`MTLSharedEvent` — GPU synchronization.
- `MTLHeap` — resource heaps for memory reuse.
- `MTLCounterSampleBuffer` — GPU performance counters.

### Shader language (MSL)
```metal
#include <metal_stdlib>
using namespace metal;

struct VertexIn { float2 position [[attribute(0)]]; float2 texCoord [[attribute(1)]]; };
struct VertexOut { float4 position [[position]]; float2 texCoord; };

vertex VertexOut vertexShader(VertexIn in [[stage_in]]) {
  VertexOut out;
  out.position = float4(in.position, 0.0, 1.0);
  out.texCoord = in.texCoord;
  return out;
}

fragment float4 fragmentShader(VertexOut in [[stage_in]],
                               texture2d<float> tex [[texture(0)]],
                               sampler smp [[sampler(0)]]) {
  constexpr sampler s(coord::normalized, mag_filter::linear, min_filter::linear);
  return tex.sample(s, in.texCoord);
}
```

- `[[stage_in]]` — vertex input; `[[position]]` — clip-space output; `[[attribute(n)]]` — vertex attributes.
- `[[texture(n)]]` — texture bindings; `[[sampler(n)]]` — samplers; `[[buffer(n)]]` — buffers.
- `texture2d<T>`, `texture3d<T>`, `texturecube<T>`, `texture2d_array<T>`, `texture_buffer<T>` — texture types.
- `sampler` — `sample`, `gather`, `read`, `write`; addressing modes `coord::normalized`, `coord::pixel`.
- `device T*`, `thread T*`, `threadgroup T*`, `constant T*` — address spaces.
- `kernel void computeShader(device float4* output [[buffer(0)]], constant float4& params [[buffer(1)]], uint gid [[thread_position_in_grid]])` — compute kernels.
- `simd_float4`, `float4`, `half4`, `ushort4` — vector types.
- `atomic_uint`, `atomic_int` — atomics for compute.

### Per-frame rendering
```swift
// SwiftUI/MTKView integration
class Renderer: NSObject, MTKViewDelegate {
  let device: MTLDevice
  let commandQueue: MTLCommandQueue
  let pipelineState: MTLRenderPipelineState

  func mtkView(_ view: MTKView, drawableSizeWillChange size: CGSize) {}

  func draw(in view: MTKView) {
    guard let drawable = view.currentDrawable,
          let descriptor = view.currentRenderPassDescriptor else { return }
    let commandBuffer = commandQueue.makeCommandBuffer()!
    let encoder = commandBuffer.makeRenderCommandEncoder(descriptor: descriptor)!
    encoder.setRenderPipelineState(pipelineState)
    // Set buffers/textures...
    encoder.drawPrimitives(type: .triangle, vertexStart: 0, vertexCount: 3)
    encoder.endEncoding()
    commandBuffer.present(drawable)
    commandBuffer.commit()
  }
}
```

- `MTKView` — Metal-backed view: `delegate`, `device`, `currentDrawable`, `currentRenderPassDescriptor`, `preferredFramesPerSecond`, `isPaused`, `enableSetNeedsDisplay`, `sampleCount`, `colorPixelFormat`, `depthStencilPixelFormat`.
- `CAMetalLayer` — layer-backed Metal: `device`, `pixelFormat`, `drawableSize`, `framebufferOnly`, `presentsWithTransaction`, `displaySyncEnabled`.
- SwiftUI: `MetalView` representable or `UIViewRepresentable` wrapping `MTKView`.

### Performance
- Minimize pipeline state changes; use argument buffers for many resources.
- `MTLHeap`/`MTLBuffer` reuse — avoid per-frame allocations.
- `MTLFence`/`MTLSharedEvent` for CPU-GPU sync.
- `MTLCounterSampleBuffer` for GPU metrics.
- Profile with Metal System Trace + Xcode Metal debugger + Instruments Metal template.
- `MTLTexture` `storageMode`: `.shared` (CPU+GPU), `.private` (GPU only, fastest), `.memoryless` (transient).
- `MTLBuffer` `storageMode`: `.shared`, `.private`, `.memoryless`.
- `MTLResourceOptions`: `.storageModePrivate`, `.storageModeShared`, `.storageModeMemoryless`, `.cpuCacheModeWriteCombined`.
- `MTLCPUCacheMode`: `.defaultCache`, `.writeCombined`.

## 2. RealityKit — entity-component-system

```swift
import RealityKit

// Entity hierarchy
let root = Entity()
let model = try await ModelEntity.load(named: "model.usdz")
model.position = SIMD3<Float>(0, 0, -1)
root.addChild(model)
arView.scene.addAnchor(AnchorEntity(world: .zero))

// Components
model.components.set(InputTargetComponent())
model.components.set(CollisionComponent(shapes: [.generateBox(size: model.visualBounds(relativeTo: model).extents]))
model.components.set(PhysicsBodyComponent(massProperties: .default, material: .default, mode: .dynamic))

// Gestures
let tap = EntityTargetGestureRecognizer(coordinateSpace: .local)
tap.allowedComponentTypes = [InputTargetComponent.self]
tap.onEnded = { value in print("tapped") }
root.gestureRecognizer = tap

// Physics simulation
// PhysicsBodyComponent(mode: .dynamic/.kinematic/.static)
// CollisionComponent(shapes:) — collision shapes
// PhysicsMotionComponent — linear/angular velocity, gravity scale

// Audio spatialization
let audio = Entity()
audio.components.set(SoundComponent(resource: try await AudioFileResource.load(named: "sound.wav")))
audio.components.set(SpatialAudioComponent())

// Materials
var material = SimpleMaterial()
material.color = .init(tint: .red, texture: nil)
model.components.set(ModelComponent(mesh: model.visualBounds().mesh, materials: [material]))

// Anchors
let planeAnchor = AnchorEntity(.plane(.horizontal, classification: .floor, minimumBounds: [0.5, 0.5]))
let imageAnchor = AnchorEntity(.image(groupName: "AR Resources", name: "poster"))
let faceAnchor = AnchorEntity(.face)
let worldAnchor = AnchorEntity(.world(origin: .camera))
```

- `Entity` — scene graph node: `children`, `parent`, `position`, `orientation`, `scale`, `transform`, `components`, `addChild`, `removeChild`, `findEntity`, `isEnabled`, `isEnabledInHierarchy`.
- `ModelEntity` — renderable entity: `ModelComponent` (mesh + materials), `MeshResource`, `PhysicallyBasedMaterial`, `UnlitMaterial`, `SimpleMaterial`, `OcclusionMaterial`, `CustomMaterial`.
- `AnchorEntity` — world anchors: `.plane`, `.image`, `.face`, `.world`, `.hand`.
- `ARView` — AR view: `scene`, `session`, `renderOptions`, `environment`, `cameraMode`, `automaticallyConfigureSession`, `debugOptions`.
- `RealityView` — SwiftUI integration: `RealityView { content in } update: { content in }`.
- Components: `ModelComponent`, `InputTargetComponent`, `CollisionComponent`, `PhysicsBodyComponent`, `PhysicsMotionComponent`, `AudioComponent`, `SoundComponent`, `SpatialAudioComponent`, `GroundingShadowComponent`, `OpacityComponent`, `HoverEffectComponent`, `SynchronizationComponent`.
- Gestures: `EntityTargetGestureRecognizer`, `DragGesture`, `TapGesture`, `LongPressGesture`, `RotationGesture`, `MagnifyGesture`.
- Physics: `PhysicsBodyComponent` (mass, material, mode), `CollisionComponent` (shapes), `PhysicsMotionComponent` (velocity, gravity), `PhysicsSimulation`.
- Materials: `PhysicallyBasedMaterial` (baseColor, metallic, roughness, normal, ambientOcclusion, clearcoat), `SimpleMaterial` (color, isMetallic), `UnlitMaterial` (color), `OcclusionMaterial` (depth-only), `CustomMaterial` (Metal shaders).
- Mesh: `MeshResource` — `generateBox`, `generateSphere`, `generatePlane`, `generateCylinder`, `generateCone`, `generateCapsule`, `generateTube`, `generateText`, custom `MeshDescriptor`.
- Audio: `AudioFileResource`, `AudioPlaybackController`, `SoundComponent`, `SpatialAudioComponent`.
- Animation: `AnimationResource`, `AnimationDefinition`, `SampledAnimation`, `PhysicsAnimation`.
- AR: `ARWorldTrackingConfiguration`, `PlaneDetectionProvider`, `ImageTrackingProvider`, `ObjectTrackingProvider`, `FaceTrackingProvider`, `SceneUnderstandingProvider`.
- Reality Composer Pro — visual scene editor for RealityKit assets.

## 3. SceneKit (legacy)

```swift
import SceneKit
let scene = SCNScene()
let cameraNode = SCNNode()
cameraNode.camera = SCNCamera()
cameraNode.position = SCNVector3(0, 0, 5)
scene.rootNode.addChildNode(cameraNode)

let box = SCNBox(width: 1, height: 1, length: 1, chamferRadius: 0.1)
let material = SCNMaterial()
material.diffuse.contents = UIColor.red
box.materials = [material]
let boxNode = SCNNode(geometry: box)
scene.rootNode.addChildNode(boxNode)

let scnView = SCNView()
scnView.scene = scene
scnView.allowsCameraControl = true
scnView.autoenablesDefaultLighting = true
```

- `SCNScene`/`SCNNode`/`SCNGeometry`/`SCNMaterial`/`SCNCamera`/`SCNLight`/`SCNAction`/`SCNPhysicsBody`/`SCNParticleSystem`.
- `SCNView`/`SCNSceneRenderer` — rendering views.
- `SCNAnimation`/`SCNAction` — animations.
- `SCNPhysicsBody`/`SCNPhysicsWorld`/`SCNPhysicsBehavior` — physics.
- `SCNParticleSystem` — particles.
- `SCNText`/`SCNShape` — text/shapes.
- **Status**: maintenance mode; prefer RealityKit for new code.

## 4. SpriteKit & GameplayKit

```swift
import SpriteKit

class GameScene: SKScene {
  override func didMove(to view: SKView) {
    backgroundColor = .black
    let sprite = SKSpriteNode(color: .red, size: CGSize(width: 50, height: 50))
    sprite.position = CGPoint(x: size.width/2, y: size.height/2)
    addChild(sprite)
    sprite.run(.repeatForever(.sequence([.moveBy(x: 100, y: 0, duration: 1), .moveBy(x: -100, y: 0, duration: 1)])))
  }
  override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
    // Handle touches
  }
  override func update(_ currentTime: TimeInterval) {
    // Per-frame update
  }
}

let skView = SKView()
skView.presentScene(GameScene(size: skView.bounds.size))
skView.ignoresSiblingOrder = true
skView.isMultipleTouchEnabled = true
```

- `SKScene`/`SKNode`/`SKSpriteNode`/`SKShapeNode`/`SKLabelNode`/`SKEmitterNode`/`SKCropNode`/`SKVideoNode`/`SKTileMapNode`/`SKEffectNode`/`SKCameraNode`/`SKLightNode`/`SKAudioNode`/`SKReferenceNode`/`SKWarpGeometryGrid`.
- `SKAction` — animations: `.move`, `.scale`, `.rotate`, `.fade`, `.colorize`, `.sequence`, `.group`, `.repeat`, `.run`, `.wait`, `.removeFromParent`, `.customAction`.
- `SKPhysicsBody` — physics: `.circle`, `.rectangle`, `.edge`, `.polygon`, `.texture`, `.alphaMask`; `categoryBitMask`, `collisionBitMask`, `contactTestBitMask`; `velocity`, `angularVelocity`, `mass`, `friction`, `restitution`, `linearDamping`, `angularDamping`, `isDynamic`, `affectedByGravity`, `allowsRotation`.
- `SKPhysicsWorld` — `gravity`, `speed`, `contactDelegate`, `update`.
- `SKPhysicsContactDelegate` — `didBegin`, `didEnd`, `didSimulate`.
- `SKShader` — custom Metal shaders: `SKShader(source:)`, uniforms.
- `SKTileMapNode` — tile maps: `SKTileSet`, `SKTileGroup`, `SKTileDefinition`.
- `SKEmitterNode` — particles: `numParticlesToEmit`, `particleBirthRate`, `particleLifetime`, `particleSpeed`, `particleAcceleration`, `particleColor`, `particleColorBlendFactor`, `particleAlpha`, `particleScale`, `particleRotation`, `particlePosition`, `particleBlendMode`.
- `SKEffectNode` — Core Image filters: `filter`, `blendMode`.
- `SKVideoNode` — video playback.
- `SKCameraNode` — camera with `scaleMode`.
- `SKLabelNode` — text labels: `fontName`, `fontSize`, `fontColor`, `verticalAlignmentMode`, `horizontalAlignmentMode`.
- `SKShapeNode` — vector shapes: `path`, `fillColor`, `strokeColor`, `lineWidth`, `glowWidth`, `blendMode`.
- `GameplayKit` — `GKRandomSource`, `GKStateMachine`, `GKGoal`, `GKBehavior`, `GKAgent`, `GKPath`, `GKGridGraph`, `GKQuadtree`, `GKOctree`, `GKRuleSystem`.

## 5. GameController framework

```swift
import GameController

// Physical controllers
GCController.controllers().forEach { controller in
  controller.extendedGamepad?.valueChangedHandler = { gamepad, element in
    // Handle input
  }
  controller.motion?.valueChangedHandler = { motion, element in
    // Handle motion (gyro/accelerometer)
  }
}

// Virtual controller (gamepad UI)
let virtual = GCVirtualController(configuration: GCVirtualController.Configuration())
virtual.connect()

// Controller elements
GCControllerButtonInput  // buttons
GCAxisInput              // analog sticks/triggers
GCMotion                 // gyro/accelerometer
GCDeviceHaptics          // haptics
```

- `GCController` — `controllers()`, `controller(withVendorName:)`? — `controller(withVendorName:)`; `vendorName`, `productCategory`, `battery`, `physicalInputProfile`, `extendedGamepad`, `microGamepad`, `motion`, `haptics`, `buttonHome`.
- `GCExtendedGamepad` — `dpad`, `buttonA/B/X/Y`, `leftThumbstick`, `rightThumbstick`, `leftTrigger`, `rightTrigger`, `leftShoulder`, `rightShoulder`, `buttonOptions`, `buttonMenu`, `buttonHome`.
- `GCMicroGamepad` — `dpad`, `buttonA/X`, `buttonY/B`.
- `GCMotion` — `gravity`, `userAcceleration`, `rotationRate`, `attitude`, `quaternion`.
- `GCDeviceHaptics` — `createEngine(withLocality:)`, `hapticCapability`, `supportsPlayerInteraction`? — `createEngine(withLocality:)` returns `GCHapticsEngine`.
- `GCHapticsEngine` — `play`, `stop`, `isMuted`, `capabilities`, `supportedLocalities`.
- `GCHapticsLocality` — `.leftHandle`, `.rightHandle`, `.leftTrigger`, `.rightTrigger`.
- `GCHapticsPattern` — `init(events:parameters:)`, `createCustom`.
- `GCHapticsEvent` — `init(eventType:intensity:atTime:relativeTime:)`, `eventType` (`.hapticTransient`, `.hapticContinuous`, `.audioCustom`, `.audioContinuous`), `parameters` (`.hapticIntensity`, `.hapticSharpness`, `.audioPitch`, `.audioVolume`).
- `GCHapticsParameter` — `.hapticIntensity`, `.hapticSharpness`, `.audioPitch`, `.audioVolume`.
- `GCKeyboard` — `keyChangedHandler`, `keyboardInput`.
- `GCDevice` — `didConnect`, `didDisconnect` notifications.

## 6. Common graphics bugs → fixes

| Bug | Cause | Fix |
|---|---|---|
| Metal pipeline creation fails | invalid shader, wrong pixel format | check shader compilation, `pixelFormat` matches drawable |
| Texture appears black | wrong texture binding, sampler issue | check `setFragmentTexture` index, `MTLSamplerState` |
| RealityKit entity not visible | no camera/light, wrong position | check `ARView` camera, entity `position`, `isEnabled` |
| Physics objects fall through | no collision shapes, wrong collision mask | add `CollisionComponent`, check `collisionBitMask` |
| SpriteKit physics wrong | wrong category/collision masks | check `categoryBitMask`/`collisionBitMask`/`contactTestBitMask` |
| SceneKit deprecated | maintenance mode | migrate to RealityKit for new code |
| Metal performance poor | per-frame allocations, state changes | reuse buffers, minimize pipeline changes, use heaps |
| RealityKit gesture not firing | no `InputTargetComponent` | add `InputTargetComponent` to entity |
| SpriteKit action jerky | too many actions, main thread work | optimize, use `SKAction` efficiently |
| GameController not detected | not calling `GCController.controllers()` | observe `GCController.didConnectNotification` |
| Metal memory leak | buffers/textures not released | use `MTLHeap`/`MTLBuffer` pools, `didComplete` handlers |
| RealityKit anchor not appearing | wrong anchor type, no tracking | check `ARWorldTrackingConfiguration`, anchor type |
