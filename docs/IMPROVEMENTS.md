# Wawa Lipsync - Improvements & Enhancements

This document outlines potential improvements and enhancements for the wawa-lipsync library.

---

## 1. Algorithm Improvements

### 1.1 Machine Learning-Based Detection

**Current**: Rule-based scoring with hardcoded thresholds  
**Proposed**: Train a lightweight ML model for viseme classification

```
Benefits:
- Higher accuracy across different voices
- Adapts to various languages and accents
- Reduce manual threshold tuning
```

**Implementation Ideas**:
- Use TensorFlow.js for browser-compatible inference
- Create a small CNN/RNN trained on labeled audio-viseme data
- Keep the rule-based system as fallback

---

### 1.2 Formant Detection Enhancement

**Current**: 7 fixed frequency bands  
**Proposed**: Dynamic formant tracking

```typescript
// Current approach
const bands = [
  { start: 50, end: 200 },
  { start: 200, end: 400 },
  // ...fixed ranges
];

// Enhanced approach
class FormantTracker {
  trackF1F2(spectrum: Float32Array): { f1: number; f2: number } {
    // Peak detection algorithm
    // Adaptive thresholds based on voice characteristics
  }
}
```

**Benefits**:
- Better vowel differentiation
- Handles different voice pitches (male/female/child)

---

### 1.3 Coarticulation Modeling

**Current**: Instant viseme transitions  
**Proposed**: Smooth transitions based on phoneme context

```typescript
interface VisemeTransition {
  from: VISEMES;
  to: VISEMES;
  duration: number;
  interpolation: 'linear' | 'ease' | 'spring';
}

// Example: P→A transition is different from S→A
const getTransitionParams = (from: VISEMES, to: VISEMES): VisemeTransition => {
  // Lookup table based on phonetic rules
};
```

---

## 2. API Enhancements

### 2.1 Event-Based Architecture

**Current**: Polling-based (`processAudio()` in animation loop)  
**Proposed**: Add event emitters

```typescript
lipsync.on('visemeChange', (event) => {
  console.log(event.previous, event.current, event.confidence);
});

lipsync.on('speechStart', () => { /* ... */ });
lipsync.on('speechEnd', () => { /* ... */ });
```

---

### 2.2 Confidence Scores

**Current**: Returns single viseme  
**Proposed**: Return probabilities for all visemes

```typescript
interface VisemeResult {
  primary: VISEMES;
  confidence: number;
  all: Record<VISEMES, number>;
}

// Enables smooth blending in 3D applications
lipsync.getVisemeWeights(); // { viseme_aa: 0.6, viseme_O: 0.3, ... }
```

---

### 2.3 Presets for Different Use Cases

```typescript
const lipsync = new Lipsync({
  preset: 'realtime',   // Low latency, less smoothing
  // or
  preset: 'animation',  // Higher accuracy, more smoothing
  // or  
  preset: 'music',      // Optimized for singing
});
```

---

## 3. Performance Optimizations

### 3.1 Web Worker Support

**Current**: Runs on main thread  
**Proposed**: Offload processing to Web Worker

```
Main Thread          Worker Thread
     │                    │
     ├─────Audio Data─────►
     │                    │
     │                    ├──► FFT Analysis
     │                    ├──► Feature Extraction
     │                    ├──► Viseme Detection
     │                    │
     ◄────Viseme Result────┤
```

**Benefits**:
- Prevents UI jank
- Enables higher FFT sizes without performance issues

---

### 3.2 Adaptive Processing

```typescript
// Skip processing during silence
if (volume < silenceThreshold) {
  return cachedSilenceResult;
}

// Reduce FFT size when CPU is constrained
if (frameRate < 30) {
  this.analyser.fftSize = 1024; // Reduced from 2048
}
```

---

### 3.3 WASM Acceleration

Replace JavaScript audio processing with WebAssembly:
- Faster FFT computation
- Potential 2-5x performance improvement
- Libraries: `fftw-wasm`, `kissfft-wasm`

---

## 4. New Features

### 4.1 Multi-Language Support

```typescript
const lipsync = new Lipsync({
  language: 'ja', // Japanese has different phoneme set
});

// Extend VISEMES enum for language-specific shapes
enum VISEMES_JA {
  // Japanese-specific visemes
}
```

---

### 4.2 Audio Preprocessing

```typescript
lipsync.setPreprocessing({
  noiseReduction: true,
  normalize: true,
  highPassFilter: 80, // Hz - remove rumble
});
```

---

### 4.3 Offline Analysis

```typescript
// Analyze entire audio file, return timed viseme data
const timeline = await lipsync.analyzeFile(audioBlob);
// Returns: [{ time: 0.0, viseme: 'sil' }, { time: 0.1, viseme: 'aa' }, ...]
```

---

### 4.4 Integration Helpers

```typescript
// Three.js/R3F helper
import { useLipsync } from 'wawa-lipsync/react';

function Avatar({ audioRef }) {
  const viseme = useLipsync(audioRef);
  // Automatically updates morph targets
}

// Ready Player Me integration
lipsync.applyToRPMAvatar(avatarMesh);
```

---

## 5. Developer Experience

### 5.1 Debug Visualization

```typescript
lipsync.enableDebug({
  showSpectrum: true,
  showBands: true,
  logScores: true,
});
```

Built-in canvas overlay showing:
- Real-time frequency spectrum
- Band energy levels
- Current scores for each viseme

---

### 5.2 TypeScript Improvements

```typescript
// Stricter typing
type VisemeHandler<T extends VISEMES> = (viseme: T) => void;

// Generic audio source type
connectAudio<T extends HTMLMediaElement | MediaStream>(source: T): void;
```

---

### 5.3 Testing Utilities

```typescript
import { MockAudioContext, generateTestTone } from 'wawa-lipsync/testing';

// Generate test signals for each phoneme
const vowelA = generateTestTone({ formants: [800, 1200] }); // Should detect 'aa'
```

---

## 6. Priority Roadmap

| Priority | Feature | Effort | Impact |
|----------|---------|--------|--------|
| 🔴 High | Confidence scores | Low | High |
| 🔴 High | Event-based API | Low | High |
| 🟡 Medium | Web Worker support | Medium | Medium |
| 🟡 Medium | Debug visualization | Medium | High |
| 🟢 Low | ML-based detection | High | High |
| 🟢 Low | WASM acceleration | High | Medium |

---

## 7. Breaking Changes to Consider

For v1.0 release:

1. **Rename `processAudio()` → `update()`** - More intuitive
2. **Return object instead of enum** - `{ viseme, confidence, weights }`
3. **Async initialization** - Required for Web Worker support
4. **Remove deprecated `webkitAudioContext`** - Modern browsers only
