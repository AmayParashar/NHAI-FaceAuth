# NHAI Datalake 3.0 — Offline-First Biometric Authentication Module

**Hackathon 2026 Submission by Amay Parashar**
`amay.parashar2@gmail.com` · Independent Developer · Pre-UG · Jaipur, Rajasthan

---

## What This Is

A fully offline facial recognition and liveness detection system designed specifically
for NHAI toll plazas and remote highway check-posts where internet connectivity is
unreliable or completely absent.

**Zero network dependency. 100% on-device. Auth in under 200ms.**

---

## Architecture at a Glance

```
[Camera Frame - 30fps]
        │
        ▼
[Face Detection]          ← Google ML Kit / YuNet TFLite (~2ms)
        │
        ▼
[Passive Liveness]        ← MobileNetV2 TFLite: texture, moiré, depth, specular
        │
     ┌──┴──┐
  Pass    Borderline → [Active Challenge] ← EAR blink / head pose / smile
     └──┬──┘
        ▼
[Score Fusion]            ← 55% passive + 45% active ≥ 0.85 to proceed
        │
        ▼
[MobileFaceNet]           ← 128-dim embedding extraction (INT8 quantised)
        │
        ▼
[Cosine Match]            ← vs encrypted Room DB vectors (AES-256 / SQLCipher)
        │
     GRANT / DENY → [AuthAuditEntity queue] → [Sync when online via WorkManager]
```

---

## Project Structure

```
DatalakeFaceAuth/
├── app/
│   ├── build.gradle                          ← All dependencies
│   ├── proguard-rules.pro                    ← TFLite / SQLCipher protection
│   └── src/main/
│       ├── AndroidManifest.xml
│       ├── assets/                           ← Place .tflite models here (see below)
│       │   ├── face_detection_short_range.tflite
│       │   ├── mobile_face_net.tflite
│       │   └── liveness_passive.tflite
│       └── java/com/datalake/faceauth/
│           ├── DatalakeApp.kt                ← Application class
│           ├── db/
│           │   └── FaceAuthDatabase.kt       ← Room DB + SQLCipher + DAOs
│           ├── model/
│           │   └── FaceModels.kt             ← TFLite inference wrappers
│           ├── liveness/
│           │   └── ActiveLiveness.kt         ← EAR, head pose, smile, score fusion
│           ├── session/
│           │   └── FaceAuthSession.kt        ← Main orchestrator
│           ├── enrollment/
│           │   └── EnrollmentManager.kt      ← 5-frame enrollment flow
│           └── sync/
│               └── OfflineSyncWorker.kt      ← WorkManager audit sync
└── README.md
```

---

## Key Technical Decisions

### Why TFLite + NNAPI?
Models run on the device's Neural Networks API (hardware accelerator) or GPU delegate,
keeping latency under 200ms even on mid-range Android hardware without a network call.

### Why SQLCipher for Room?
The Room database file is AES-256 encrypted at rest using a key stored in Android Keystore.
If a toll plaza device is stolen, biometric embeddings cannot be extracted.
**No raw face image is ever written to disk** — only the 128-dimensional float vector.

### Why score fusion (55% passive + 45% active)?
Passive liveness runs every frame silently. Active challenges (blink/nod/smile) are only
triggered when the passive score is borderline (0.65–0.85). This minimises friction for
legitimate users while maintaining high security against 2D photo/video spoof attacks.

### Why WorkManager for sync?
WorkManager guarantees audit log delivery even if the app is killed mid-sync.
Logs accumulate in the local encrypted queue and are batched to NHAI central DB
the moment any network (Wi-Fi or 4G) is detected — no sync gaps ever.

---

## Model Files (Not Included — Download Separately)

Place these in `app/src/main/assets/`:

| File | Source | Size |
|---|---|---|
| `face_detection_short_range.tflite` | [MediaPipe Models](https://storage.googleapis.com/mediapipe-models/) | ~1.4 MB |
| `mobile_face_net.tflite` | [sirius-ai/MobileFaceNet_TF](https://github.com/sirius-ai/MobileFaceNet_TF) → export INT8 | ~4 MB |
| `liveness_passive.tflite` | [minivision-ai/Silent-Face-Anti-Spoofing](https://github.com/minivision-ai/Silent-Face-Anti-Spoofing) → TFLite | ~6 MB |

Total on-device footprint: **~15 MB**

---

## Quick Integration into Datalake 3.0

```kotlin
// 1. Initialise session
val session = FaceAuthSession(context, userId = driverId)

// 2. Handle results
session.onResult = { result ->
    when (result) {
        is AuthResult.Granted -> DatalakeAuthManager.issueToken(result.userId)
        is AuthResult.Denied  -> DatalakeAuthManager.fallbackToPIN(userId)
        else -> showEnrollmentScreen()
    }
}

// 3. Start and feed camera frames
session.start()
cameraFeed.setOnFrameAvailable { bitmap -> session.processFrame(bitmap) }

// 4. Clean up
session.release()
```

---

## Security & Compliance

| Property | Detail |
|---|---|
| **DPDPB-2023 Compliant** | 0 bytes of raw biometric data stored or transmitted |
| **Encryption** | AES-256 / SQLCipher, key in Android Keystore |
| **Anti-replay** | Every auth session has a unique nonce + timestamp |
| **Lockout** | 5 denials in 5 minutes triggers supervisor escalation |
| **Audit trail** | Every GRANT/DENY/SPOOF logged, synced to NHAI central DB |

---

## Performance Targets

| Metric | Target | Notes |
|---|---|---|
| Auth latency | < 200ms | On Snapdragon 680+ with NNAPI |
| Liveness accuracy | > 98% | On [Silent-Face benchmark](https://github.com/minivision-ai/Silent-Face-Anti-Spoofing) |
| False Accept Rate | < 0.1% | Threshold 0.85 fusion score |
| False Reject Rate | < 1% | Threshold 0.70 cosine similarity |
| Model footprint | ~15 MB | INT8 quantised, all three models combined |

---

*Built for Bharat's highways — offline by design, secure by architecture, ready to scale nationally.*
