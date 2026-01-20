# Lipsync - Technical Documentation

A real-time lipsync library that analyzes audio frequencies to detect visemes (mouth shapes) for animating 2D/3D characters.

## Architecture Overview

```mermaid
flowchart LR
    A[Audio Source] --> B[Web Audio API]
    B --> C[AnalyserNode]
    C --> D[FFT Analysis]
    D --> E[Feature Extraction]
    E --> F[Viseme Scoring]
    F --> G[FSM State Machine]
    G --> H[Detected Viseme]
```

## Core Components

### 1. Lipsync Class

The main entry point located in `packages/wawa-lipsync/src/lipsync.ts`.

#### Constructor Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fftSize` | number | 2048 | FFT window size for frequency analysis |
| `historySize` | number | 10 | Number of frames to average for smoothing |

#### Key Properties

```typescript
class Lipsync {
  features: Feature | null;    // Current audio features
  viseme: VISEMES;             // Currently detected viseme
}
```

---

### 2. Visemes (Mouth Shapes)

Based on [Oculus LipSync viseme set](https://developers.meta.com/horizon/documentation/unity/audio-ovrlipsync-viseme-reference/).

| Viseme | Value | Category | Description |
|--------|-------|----------|-------------|
| `sil` | `viseme_sil` | Silence | Closed mouth |
| `PP` | `viseme_PP` | Plosive | P, B, M sounds |
| `FF` | `viseme_FF` | Fricative | F, V sounds |
| `TH` | `viseme_TH` | Fricative | Th sounds |
| `DD` | `viseme_DD` | Plosive | D, T sounds |
| `kk` | `viseme_kk` | Plosive | K, G sounds |
| `CH` | `viseme_CH` | Fricative | Ch, J, Sh sounds |
| `SS` | `viseme_SS` | Fricative | S, Z sounds |
| `nn` | `viseme_nn` | Plosive | N, L sounds |
| `RR` | `viseme_RR` | Fricative | R sounds |
| `aa` | `viseme_aa` | Vowel | "ah" as in "father" |
| `E` | `viseme_E` | Vowel | "eh" as in "bed" |
| `I` | `viseme_I` | Vowel | "ee" as in "see" |
| `O` | `viseme_O` | Vowel | "oh" as in "go" |
| `U` | `viseme_U` | Vowel | "oo" as in "too" |

---

## Audio Processing Pipeline

### Step 1: Audio Connection

```typescript
lipsync.connectAudio(audioElement);
// or
await lipsync.connectMicrophone();
```

The library creates an `AudioContext` and connects the source to an `AnalyserNode`.

### Step 2: Feature Extraction

The `extractFeatures()` method performs FFT analysis and extracts:

#### Frequency Bands

| Band | Range (Hz) | Purpose |
|------|------------|---------|
| 1 | 50-200 | Low energy detection |
| 2 | 200-400 | F1 formant (lower) |
| 3 | 400-800 | F1 formant (mid) |
| 4 | 800-1500 | F2 formant (front) |
| 5 | 1500-2500 | F2/F3 formants |
| 6 | 2500-4000 | Fricative detection |
| 7 | 4000-8000 | High fricative detection |

#### Extracted Features

```typescript
interface Feature {
  bands: number[];      // Energy in each frequency band (0-1)
  deltaBands: number[]; // Rate of change per band
  volume: number;       // Overall audio volume (0-1)
  centroid: number;     // Spectral centroid (Hz)
}
```

### Step 3: Viseme Detection

The `detectState()` method uses a scoring system:

1. **Compute base scores** for each viseme based on:
   - Volume thresholds
   - Spectral centroid ranges
   - Frequency band energy patterns
   - Delta (rate of change) values

2. **Apply consistency adjustments**:
   - Boost current viseme in early phase (0-100ms)
   - Decay boost over time (100-300ms)
   - Apply penalty if held too long (>300ms)

3. **Select highest-scoring viseme**

---

## State Machine (FSM)

The library uses a Finite State Machine with 4 states:

```mermaid
stateDiagram-v2
    [*] --> silence
    silence --> vowel: High volume, low centroid
    silence --> plosive: Sudden burst
    silence --> fricative: High-freq energy
    vowel --> silence: Volume drop
    vowel --> plosive: Sudden change
    plosive --> vowel: Sustained sound
    fricative --> silence: Energy drop
```

| State | Characteristics |
|-------|-----------------|
| `silence` | Low volume (<0.2) |
| `vowel` | Sustained mid-frequency, moderate centroid |
| `plosive` | Sudden energy burst, broad centroid |
| `fricative` | High-frequency energy, high centroid |

---

## Detection Algorithms

### Silence Detection
```
IF avg.volume < 0.2 AND current.volume < 0.2 THEN viseme = sil
```

### Plosive Detection
- Triggered by sudden volume changes (`dVolume > 0.01`)
- Centroid range: 1000-8000 Hz
- Higher centroid → DD, lower → PP/nn

### Fricative Detection
- High centroid (>6000 Hz)
- Strong energy in band 6 (2500-4000 Hz)
- Positive centroid delta (>1000)

### Vowel Detection
Uses formant analysis based on band energy ratios:

| Vowel | Band Pattern |
|-------|--------------|
| aa | b4 > b3 |
| E | b2 > b3 > b4 |
| I | b3 > b2 AND b3 > b4 |
| O | Small gaps between b2, b3, b4 |
| U | b1 ≈ b2 (gap < 0.25) |

---

## Usage Example

```typescript
import { Lipsync, VISEMES } from "wawa-lipsync";

// Create instance
const lipsync = new Lipsync({ fftSize: 2048, historySize: 10 });

// Connect audio
const audio = new Audio("speech.mp3");
lipsync.connectAudio(audio);

// Process in animation loop
function animate() {
  requestAnimationFrame(animate);
  lipsync.processAudio();
  
  // Use the detected viseme
  const currentViseme = lipsync.viseme;  // e.g., "viseme_aa"
  updateCharacterMouth(currentViseme);
}

animate();
audio.play();
```

---

## Performance Considerations

| Setting | Lower Value | Higher Value |
|---------|-------------|--------------|
| `fftSize` | Faster, less accurate | Slower, more accurate |
| `historySize` | More responsive, jittery | Smoother, more latency |

**Recommended defaults**: `fftSize: 2048`, `historySize: 10`

---

## 3D Model Requirements

### Required Morph Targets (Blend Shapes)

For the library to animate your 3D model, it must have morph targets/blend shapes corresponding to the 15 visemes:

| Morph Target Name | Required | Description |
|-------------------|----------|-------------|
| `viseme_sil` | ✅ Yes | Closed/neutral mouth |
| `viseme_PP` | ✅ Yes | Lips pressed (P, B, M) |
| `viseme_FF` | ✅ Yes | Lower lip to teeth (F, V) |
| `viseme_TH` | ⚠️ Optional | Tongue between teeth |
| `viseme_DD` | ✅ Yes | Tongue to roof (D, T) |
| `viseme_kk` | ✅ Yes | Mouth open, tongue back (K, G) |
| `viseme_CH` | ⚠️ Optional | Lips forward (Ch, J, Sh) |
| `viseme_SS` | ✅ Yes | Teeth together (S, Z) |
| `viseme_nn` | ⚠️ Optional | Mouth slightly open (N, L) |
| `viseme_RR` | ⚠️ Optional | Lips rounded (R) |
| `viseme_aa` | ✅ Yes | Mouth wide open (ah) |
| `viseme_E` | ✅ Yes | Mouth semi-open (eh) |
| `viseme_I` | ✅ Yes | Lips spread (ee) |
| `viseme_O` | ✅ Yes | Lips rounded (oh) |
| `viseme_U` | ✅ Yes | Lips pursed (oo) |

> **Note**: At minimum, include the 10 visemes marked as "Yes" for basic lipsync. Optional visemes improve accuracy but can fallback to similar shapes.

---

### Model Format Recommendations

#### glTF/GLB (Recommended)

```javascript
// Load model with morph targets
import { useGLTF } from '@react-three/drei';

const { scene } = useGLTF('/avatar.glb');
const head = scene.getObjectByName('Head');

// Access morph targets
const morphTargets = head.morphTargetDictionary;
const morphInfluences = head.morphTargetInfluences;
```

**Requirements**:
- Morph targets must be in `mesh.morphTargetDictionary`
- Target names must match viseme enum values exactly
- Weights should be 0.0-1.0 range

---

#### VRM Models

VRM models (common for VTubers) work well with this library:

```javascript
import { VRMLoaderPlugin } from '@pixiv/three-vrm';
import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader';

const loader = new GLTFLoader();
loader.register((parser) => new VRMLoaderPlugin(parser));

loader.load('/avatar.vrm', (gltf) => {
  const vrm = gltf.userData.vrm;
  const blendShapeProxy = vrm.blendShapeProxy;
  
  // Set viseme weights
  blendShapeProxy.setValue('viseme_aa', 0.8);
});
```

**VRM-specific notes**:
- Use `blendShapeProxy` instead of direct morph target access
- VRM models often use ARKit naming - you may need mapping

---

#### Ready Player Me

Ready Player Me avatars come with viseme support out of the box:

```javascript
// RPM models have Oculus visemes by default
const avatar = await loadRPMAvatar(url);
const head = avatar.getObjectByName('Wolf3D_Head');

// Morph targets are already named correctly
head.morphTargetDictionary; // Contains all 15 visemes
```

**RPM advantages**:
- Pre-configured viseme morph targets
- Optimized for web performance
- Consistent naming across all avatars

---

### Integration Guide

#### Three.js / React Three Fiber

```typescript
import { useFrame } from '@react-three/fiber';
import { useGLTF } from '@react-three/drei';
import { lipsyncManager } from './App';

function Avatar() {
  const { scene } = useGLTF('/avatar.glb');
  const headRef = useRef();

  useFrame(() => {
    if (!headRef.current) return;
    
    const { morphTargetDictionary, morphTargetInfluences } = headRef.current;
    const viseme = lipsyncManager.viseme;
    
    // Get index of current viseme
    const index = morphTargetDictionary[viseme];
    
    if (index !== undefined) {
      // Smooth transition using lerp
      morphTargetInfluences.forEach((influence, i) => {
        const target = i === index ? 1.0 : 0.0;
        morphTargetInfluences[i] += (target - influence) * 0.3;
      });
    }
  });

  return <primitive object={scene} ref={headRef} />;
}
```

---

### Troubleshooting

#### Model Not Animating

1. **Check morph target names**:
```javascript
console.log(mesh.morphTargetDictionary);
// Should show: { viseme_sil: 0, viseme_PP: 1, ... }
```

2. **Verify morph target influences**:
```javascript
console.log(mesh.morphTargetInfluences);
// Should be an array of numbers [0-1]
```

3. **Ensure mesh is correctly referenced**:
```javascript
// Find mesh with morph targets
scene.traverse((child) => {
  if (child.isMesh && child.morphTargetDictionary) {
    console.log('Found morph targets on:', child.name);
  }
});
```

---

#### Performance Issues

- **Use morph target limits**: Most 3D engines support setting max active targets
- **Optimize mesh geometry**: High poly counts affect morph target performance
- **Use LOD**: Lower detail models for distant characters

---

### Creating Custom Visemes

If your model doesn't have visemes, you can create them in Blender:

1. Import your character model
2. Enter Edit Mode on the head mesh
3. Create shape keys for each viseme
4. Export as glTF with morph targets enabled
5. Ensure shape key names match viseme enum values

**Blender Export Settings**:
- ✅ Include: Morphs
- ✅ Morph Normal: Tangent
- Format: glTF Binary (.glb)

---

## File Structure

```
packages/wawa-lipsync/src/
├── index.ts          # Exports Lipsync and VISEMES
├── lipsync.ts        # Main Lipsync class (424 lines)
├── visemes.ts        # VISEMES enum (15 visemes)
├── types.d.ts        # React Three Fiber type augmentation
└── utils/
    └── mathUtil.ts   # Average calculation helper
```
