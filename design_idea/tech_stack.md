0 · Foundation (everything else depends on these four)

Client platform

┌────────────────────────┬───────────────────────────────────────────────────────────────────────────────────────┐
│         Option         │                                        Verdict                                        │
├────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
│ Flutter                │ ✓ Pick                                                                                │
├────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
│ React Native           │ Weak custom-canvas perf on low-end tablets; worse offline DB story                    │
├────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
│ Native Kotlin +        │ Best perf & kiosk control, smallest APK — the real alternative if the team knows      │
│ Compose                │ Kotlin                                                                                │
├────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
│ Unity                  │ ✗ Great for 7 screens, terrible for the other 40                                      │
├────────────────────────┼───────────────────────────────────────────────────────────────────────────────────────┤
│ PWA / Capacitor        │ ✗ Background alarms unreliable, no true kiosk, on-device ML painful                   │
└────────────────────────┴───────────────────────────────────────────────────────────────────────────────────────┘

Why Flutter here specifically: the Calm Shell needs every pixel custom (no Material chrome, 15mm targets, zero text) — Flutter draws its own pixels, so a kiosk shell is natural rather than a fight. Games are 2D tap canvases (CustomPainter). Dashboard is charts and forms. One codebase covers patient shell + caregiver console + CHW mode.

Target: Android only, minSdk 24 (Android 7), targetSdk 34. iOS is irrelevant for NER tablets.

State management → Riverpod

Not a preference — an architectural fit. CareState is a global reactive store every feature subscribes to. Riverpod's provider graph maps 1:1 onto the kernel. (BLoC = more ceremony; GetX = unmaintainable.)

Local database

┌────────────────────────────┬───────────────────────────────────────────────────────────────────────┐
│           Option           │                                Verdict                                │
├────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ Drift (SQLite) + SQLCipher │ ✓ Pick — type-safe SQL, real migrations, reactive streams, encryption │
├────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ Isar                       │ Fast, but weaker encryption story + maintenance concerns              │
├────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ Hive                       │ ✗ Not relational enough for an append-only event log                  │
├────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ ObjectBox / Realm          │ Built-in sync, but paid / cloud-coupled                               │
└────────────────────────────┴───────────────────────────────────────────────────────────────────────┘

Passphrase in Android Keystore via flutter_secure_storage.

Inference runtime → TFLite (tflite_flutter)

NNAPI + XNNPACK delegates, smallest footprint, best Android support. (Alternatives: ONNX Runtime Mobile, ExecuTorch — both heavier here.)

---

(a) Cognitive games

┌───────────────┬────────────────────────────────────────┬──────────────────────────────────────────────────────┐
│     Layer     │                Options                 │                         Pick                         │
├───────────────┼────────────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Rendering     │ Flutter widgets + CustomPainter ·      │ Plain Flutter + CustomPainter. These are turn-based  │
│               │ Flame · Unity                          │ tap games — you don't need a game loop               │
├───────────────┼────────────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Celebration   │ Lottie · Rive · custom                 │ lottie — tiny, designer-editable                     │
│ anims         │                                        │                                                      │
├───────────────┼────────────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Game          │ JSON manifest · YAML · Protobuf · DB   │ Signed JSON manifests in a ZIP — non-programmers on  │
│ definition    │ rows                                   │ the content team can author them                     │
├───────────────┼────────────────────────────────────────┼──────────────────────────────────────────────────────┤
│ Difficulty    │ In-manifest: grid size, distractor     │ —                                                    │
│ params        │ count, exposure ms, hint delay         │                                                      │
└───────────────┴────────────────────────────────────────┴──────────────────────────────────────────────────────┘

Game-as-config is the leverage: 7 templates × 8 culture packs × 6 difficulty bands = enormous apparent content from ~7 screens of code. Localisation becomes a content problem, not a code problem.

---

(b) AI/ML — four separate pieces, deliberately

1 · Ability tracker θ (Rasch/IRT + Bayesian update)

┌───────────────────────┬───────────────────────────────────────────────┐
│        Option         │                    Verdict                    │
├───────────────────────┼───────────────────────────────────────────────┤
│ Pure Dart, ~100 lines │ ✓ Pick                                        │
├───────────────────────┼───────────────────────────────────────────────┤
│ TFLite model          │ ✗ Pointless — it's a logistic + Kalman update │
├───────────────────────┼───────────────────────────────────────────────┤
│ Server-side           │ ✗ Violates offline                            │
└───────────────────────┴───────────────────────────────────────────────┘

Say this out loud in the pitch: this needs no neural network, and pretending otherwise is a weakness, not a strength. A well-chosen estimator beats a gratuitous LSTM — and it's explainable to a clinician, which the HITL requirement demands.

2 · Item difficulty calibration (dev-time, offline)

Python + py-irt or pymc → exported b constants in the manifest. v1 is expert-seeded; recalibrate after pilot data. Be honest about that.

3 · Frustration detector

┌──────────────────────────┬──────────────────────────────────────────────────┐
│          Option          │                     Verdict                      │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ Threshold rules (v1)     │ ✓ Pick — you have zero labelled frustration data │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ Logistic regression (v2) │ ✓ Coefficients exported, runs in Dart, ~20 lines │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ GBM → ONNX               │ Overkill                                         │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ LSTM                     │ ✗ A judge will ask where the labels came from    │
└──────────────────────────┴──────────────────────────────────────────────────┘

Features: latency z-score, tap-rate burst (mashing), back-press count, mid-task abandonment, correct→wrong oscillation.

4 · Decline estimator + digital biomarkers ← the headline AI

┌──────────────────────────────────────────────────────┬────────────────────────────────┐
│                        Option                        │            Verdict             │
├──────────────────────────────────────────────────────┼────────────────────────────────┤
│ Ordered HMM over GDS states + EWMA/CUSUM changepoint │ ✓ Pick                         │
├──────────────────────────────────────────────────────┼────────────────────────────────┤
│ Linear trend only                                    │ Too naive                      │
├──────────────────────────────────────────────────────┼────────────────────────────────┤
│ Random Forest → stage label                          │ Loses ordering + uncertainty   │
├──────────────────────────────────────────────────────┼────────────────────────────────┤
│ LSTM/Transformer                                     │ ✗ No data, no interpretability │
└──────────────────────────────────────────────────────┴────────────────────────────────┘

Why an HMM is genuinely right here: GDS stages are ordered and transitions are near-monotonic (patients rarely improve). An HMM gives a probability distribution over stages rather than a brittle hard label, runs in milliseconds, and the forward algorithm is ~50 lines of Dart. It's explainable to a doctor.

Free biomarkers from a bare tablet:

┌────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────┐
│       Signal       │                                           How                                            │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ Tap latency drift  │ Timestamp deltas                                                                         │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ Tremor proxy       │ Spectral power in the 4–6 Hz band of the raw PointerMoveEvent path (~60–120 Hz sampling) │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ Hesitation         │ Time-to-first-touch                                                                      │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ Speech pause ratio │ On-device VAD over voice interactions                                                    │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ Circadian drift    │ Session start-time distribution                                                          │
└────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────┘

The tremor one is real, free, and nobody else will have it.

5 · Face recognition (Who's This? + audio-AR)

┌────────────────┬───────────────────────────────────────────────────────────────────────┐
│     Layer      │                                 Pick                                  │
├────────────────┼───────────────────────────────────────────────────────────────────────┤
│ Detect + align │ ML Kit Face Detection (google_mlkit_face_detection) — on-device, free │
├────────────────┼───────────────────────────────────────────────────────────────────────┤
│ Embed          │ MobileFaceNet TFLite, INT8 quantized, ~5MB → 128-d vector             │
├────────────────┼───────────────────────────────────────────────────────────────────────┤
│ Match          │ Cosine similarity vs Memory Vault embeddings in Drift                 │
└────────────────┴───────────────────────────────────────────────────────────────────────┘

100% on-device. Raw images never leave. Enrol 3–5 photos per relative.

---

(c) Voice — the contrarian stack

Output

┌───────────────────────────────────────────────────────┬────────────────────────────────────────────┐
│                        Option                         │                  Verdict                   │
├───────────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Pre-recorded native human audio                       │ ✓ Primary (~95%)                           │
├───────────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Concatenative fragments (numbers/names/times)         │ ✓ For dynamic strings                      │
├───────────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ AI4Bharat IndicTTS / Piper (tiny ONNX VITS) on-device │ Fallback tier                              │
├───────────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ flutter_tts (Android system)                          │ Assamese only — no Mizo/Khasi voices exist │
├───────────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Bhashini cloud TTS                                    │ Authoring-time only. Never at runtime      │
└───────────────────────────────────────────────────────┴────────────────────────────────────────────┘

Format: Opus-in-OGG @ ~24kbps mono. ~400 phrases × 3 languages ≈ 40–60MB total.
Playback: just_audio with ConcatenatingAudioSource for gapless fragment stitching.

Input

┌───────────────────────────────────────────┬────────────────────────────────────────────────────────────────────┐
│                  Option                   │                              Verdict                               │
├───────────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Custom keyword spotter — 20-word closed   │ ✓ Pick                                                             │
│ vocab                                     │                                                                    │
├───────────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Android SpeechRecognizer                  │ ✗ No Mizo/Khasi, usually needs network                             │
├───────────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Vosk                                      │ Hindi yes; Assamese/Mizo no                                        │
├───────────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ Whisper.cpp / IndicWhisper quantized      │ ✗ 75MB+, too slow on ₹8k tablets, poor on low-resource langs       │
├───────────────────────────────────────────┼────────────────────────────────────────────────────────────────────┤
│ openWakeWord                              │ ✓ Good off-the-shelf training pipeline if you don't want to roll   │
│                                           │ your own                                                           │
└───────────────────────────────────────────┴────────────────────────────────────────────────────────────────────┘

Architecture: DS-CNN on mel-spectrograms (the Google Speech Commands recipe) — ~30–50k params, <200KB TFLite. Training data: 30–50 utterances × 20 words × 3 languages from community volunteers. Completely feasible for a student team, and far more robust on dysarthric elderly speech than open ASR.

Distress detector: same model, always-on branch, VAD-gated + duty-cycled (battery). Capture via record / native AudioRecord @16kHz mono.

---

(d) Cultural content

┌────────────┬────────────────────────────────┬──────────────────────────────────────────────────────────────────┐
│   Layer    │            Options             │                               Pick                               │
├────────────┼────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ Pack       │ ZIP · tar.zst · Play Asset     │ Signed ZIP (Ed25519 via cryptography), versioned,                │
│ format     │ Delivery                       │ delta-updatable                                                  │
├────────────┼────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ Images     │ WebP q80 · JPEG · AVIF         │ WebP — ~30% smaller than JPEG, native since API 18 (AVIF is      │
│            │                                │ Android 12+)                                                     │
├────────────┼────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ Audio      │ Opus/OGG · AAC · MP3           │ Opus — best quality-per-byte at low bitrate                      │
├────────────┼────────────────────────────────┼──────────────────────────────────────────────────────────────────┤
│ Delivery   │ In-APK · CDN · CHW sideload ·  │ All four. Base language in APK; rest downloadable or pushed by   │
│            │ SD card                        │ CHW over Nearby                                                  │
└────────────┴────────────────────────────────┴──────────────────────────────────────────────────────────────────┘

Memory Vault ingestion — one option worth serious thought

┌─────────────────────┬──────────────────────────────────────────────────────────────────────────────────────────┐
│       Option        │                                           Note                                           │
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ In-app upload       │ Baseline                                                                                 │
│ (caregiver phone)   │                                                                                          │
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ Web portal          │ Needs the family to have data                                                            │
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ WhatsApp bot (Meta  │ This is how NER families actually communicate — your own research notes ARDSI Mizoram    │
│ Cloud API, free     │ runs support over WhatsApp groups. A relative sends a photo + voice note to a number; it │
│ tier)               │  lands in the Vault. Culturally correct and removes all onboarding friction              │
├─────────────────────┼──────────────────────────────────────────────────────────────────────────────────────────┤
│ QR-paired local     │ Offline fallback                                                                         │
│ transfer            │                                                                                          │
└─────────────────────┴──────────────────────────────────────────────────────────────────────────────────────────┘

---

(e) Reminders

┌────────────────┬──────────────────────────────────────┬────────────────────────────────────────────────────────┐
│     Layer      │               Options                │                          Pick                          │
├────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────┤
│                │ flutter_local_notifications + exact  │ AndroidScheduleMode.exactAllowWhileIdle +              │
│ Scheduling     │ AlarmManager · WorkManager · FCM     │ USE_EXACT_ALARM (medication apps qualify for this      │
│                │                                      │ permission)                                            │
├────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Background     │ android_alarm_manager_plus ·         │ Foreground service for ambient/audio mode              │
│ execution      │ foreground service                   │                                                        │
├────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Escalation —   │ Local notification                   │ —                                                      │
│ same device    │                                      │                                                        │
├────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Escalation —   │ FCM · sync-on-connect                │ FCM when available                                     │
│ remote, online │                                      │                                                        │
├────────────────┼──────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Escalation —   │ On-device SMS from the tablet's own  │                                                        │
│ remote, 2G     │ SIM · Twilio · MSG91/Gupshup         │ See below ↓                                            │
│ only           │                                      │                                                        │
└────────────────┴──────────────────────────────────────┴────────────────────────────────────────────────────────┘

⚠ Two real-world traps worth naming on stage

1 · OEM battery killers. Xiaomi, Oppo, Vivo and Realme — overwhelmingly common in India — aggressively kill background alarms. This is the most common failure mode of Indian reminder apps. Mitigation: REQUEST_IGNORE_BATTERY_OPTIMIZATIONS, a foreground service, and an onboarding step that walks the caregiver through their specific OEM's autostart settings (device_info_plus to detect manufacturer and show the right instructions).

2 · SEND_SMS is heavily restricted on Play. On-device SMS from the patient's own SIM is the perfect zero-infra 2G path — but Play will reject it without a declared core-SMS use case. Three honest routes: sideload/enterprise distribution for pilots · intent-based SMS (requires a tap) · server-side DLT-registered gateway (MSG91/Gupshup) that fires when the server sees a missed dose at sync time. Pick per deployment channel.

---

(f) Dashboard

┌──────────────────┬───────────────────────────────────────────────────┬────────────────────────────────────────┐
│      Layer       │                      Options                      │                  Pick                  │
├──────────────────┼───────────────────────────────────────────────────┼────────────────────────────────────────┤
│ Charts           │ fl_chart · Syncfusion (free community licence) ·  │ fl_chart — MIT, light, sufficient      │
│                  │ custom                                            │                                        │
├──────────────────┼───────────────────────────────────────────────────┼────────────────────────────────────────┤
│ Sundowning       │ Syncfusion · custom GridView of coloured cells    │ Custom — it's 30 lines                 │
│ heatmap          │                                                   │                                        │
├──────────────────┼───────────────────────────────────────────────────┼────────────────────────────────────────┤
│ DICE log         │ —                                                 │ Plain Drift tables + chip-based form   │
├──────────────────┼───────────────────────────────────────────────────┼────────────────────────────────────────┤
│ Doctor Summary   │ pdf + printing (pure Dart, offline) · server-side │ pdf + printing, shared via share_plus  │
│ PDF              │  · HTML print                                     │ → WhatsApp                             │
└──────────────────┴───────────────────────────────────────────────────┴────────────────────────────────────────┘

⚠ Technical risk: complex-script PDF

The Dart pdf package has limited shaping support for Bengali-Assamese conjuncts. Text will render wrong. Mitigation: render the report as a Flutter widget → capture to image (RepaintBoundary) → embed the image in the PDF. Sidesteps shaping entirely. Ugly but bulletproof, and worth knowing before you discover it the night before demo.

---

(g) Offline sync

┌────────────────┬───────────────────────────────────────────────────┬──────────────────────────────────────────┐
│     Layer      │                      Options                      │                   Pick                   │
├────────────────┼───────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Strategy       │ Custom outbox + idempotent upsert · PowerSync ·   │ Custom outbox for v1                     │
│                │ ElectricSQL · Couch/Pouch · Firestore             │                                          │
├────────────────┼───────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ IDs            │ ULID (ulid pkg)                                   │ Sortable, client-generated, no           │
│                │                                                   │ coordination                             │
├────────────────┼───────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Conflict       │ Append-only events never conflict; config = LWW   │ —                                        │
│ policy         │ w/ caregiver priority                             │                                          │
├────────────────┼───────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Sneakernet     │ Google Nearby Connections (nearby_connections) ·  │ Nearby — auto-upgrades BLE→WiFi Direct,  │
│                │ WiFi Direct · BLE-only · QR chunks                │ high bandwidth, zero internet            │
├────────────────┼───────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Crypto for the │ X25519 + XChaCha20-Poly1305 (cryptography)        │ CHW device carries the bundle opaquely   │
│  mule          │                                                   │ and cannot decrypt it                    │
├────────────────┼───────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Tiny-data      │ QR chunks                                         │ Universal — works from any phone camera  │
│ fallback       │                                                   │                                          │
└────────────────┴───────────────────────────────────────────────────┴──────────────────────────────────────────┘

Why custom over PowerSync: your writes are append-only events with client-generated ULIDs. The server just inserts and ignores duplicates. That's ~200 lines, no vendor, self-hostable for DPDP. (PowerSync is the right answer if you have budget and want it done in a day — name it as the alternative.)

---

(h) UI shell

┌─────────────────┬─────────────────────────────────────────────────┬────────────────────────────────────────────┐
│      Layer      │                     Options                     │                    Pick                    │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Kiosk           │ Device Owner + Lock Task Mode · screen pinning  │ Device Owner for deployment; screen        │
│                 │ · kiosk_mode                                    │ pinning for demo                           │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Home            │ CATEGORY_HOME intent filter                     │ ✓ App becomes the launcher — very          │
│ replacement     │                                                 │ effective for elderly devices              │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Fonts           │ Noto Sans Assamese/Bengali embedded; 24sp min,  │ —                                          │
│                 │ 32sp+ in patient shell                          │                                            │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Evening theme   │ Timer + theme swap                              │ Sundowning response                        │
│ shift           │                                                 │                                            │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Haptics         │ HapticFeedback                                  │ Tactile confirmation matters a lot here    │
├─────────────────┼─────────────────────────────────────────────────┼────────────────────────────────────────────┤
│ Screen wake     │ wakelock_plus                                   │ For ambient mode                           │
└─────────────────┴─────────────────────────────────────────────────┴────────────────────────────────────────────┘

---

Security & DPDP

┌──────────────┬─────────────────────────────────────────────────────────────────────────────────────────────────┐
│   Concern    │                                              Stack                                              │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ At rest      │ SQLCipher (AES-256) + per-patient file encryption; key in Android Keystore                      │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ In transit   │ TLS 1.3 + certificate pinning (Dio interceptor / http_certificate_pinning)                      │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Sync bundles │ X25519 + XChaCha20-Poly1305 end-to-end                                                          │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Caregiver    │ PIN hashed with Argon2id + optional local_auth biometric                                        │
│ auth         │                                                                                                 │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Consent      │ Append-only, device-signed Drift table — authority migrates to the legally authorized           │
│ ledger       │ representative around GDS 4–5                                                                   │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Audit        │ Every dashboard read logged                                                                     │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Analytics    │ ⚠ No Firebase Analytics, no Crashlytics. Self-hosted Sentry with PII scrubbing, or nothing      │
├──────────────┼─────────────────────────────────────────────────────────────────────────────────────────────────┤
│ Residency    │ ⚠ India region mandatory — AWS ap-south-1/ap-south-2, or a MeitY-empanelled provider for any    │
│              │ government pilot                                                                                │
└──────────────┴─────────────────────────────────────────────────────────────────────────────────────────────────┘

---

Social & devices

┌────────────────┬───────────────────────────────────────────────────────────────────────────────────────────────┐
│    Feature     │                                             Stack                                             │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ Voice          │ record → Opus → encrypt → outbox; just_audio playback                                         │
│ Postcards      │                                                                                               │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ Family side    │ Caregiver app or WhatsApp Cloud API bot                                                       │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ BLE band       │ ⭐ Health Connect via the health package — reads steps/sleep/HR from any band that writes to  │
│                │ it. Avoids reverse-engineering Mi Band's proprietary auth entirely                            │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ Smart pillbox  │ ESP32 + TCRT5000 IR + HX711 load cell + SIM800L; ESP-IDF/Arduino; BLE to tablet via           │
│                │ flutter_blue_plus, GSM as independent path                                                    │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ mmWave fall    │ Seeed MR60FDA2 (~₹2,500) or Hi-Link LD2410 (presence only, ~₹400) → UART → ESP32 → BLE        │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ BLE object     │ Generic nRF beacons, RSSI proximity via flutter_blue_plus                                     │
│ tags           │                                                                                               │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ NFC object     │ NTAG213 stickers (~₹10) + nfc_manager                                                         │
│ triggers       │                                                                                               │
├────────────────┼───────────────────────────────────────────────────────────────────────────────────────────────┤
│ GPS geofence   │ geolocator + foreground service                                                               │
└────────────────┴───────────────────────────────────────────────────────────────────────────────────────────────┘

---

Backend (deliberately thin)

┌───────────────────────────────┬────────────────────────────────────────────────────────────────────────────────┐
│            Option             │                                    Verdict                                     │
├───────────────────────────────┼────────────────────────────────────────────────────────────────────────────────┤
│ FastAPI + PostgreSQL +        │ ✓ Pick — Python also serves the IRT recalibration and model-training jobs      │
│ MinIO/S3                      │                                                                                │
├───────────────────────────────┼────────────────────────────────────────────────────────────────────────────────┤
│ Self-hosted Supabase          │ ✓ Strong alternative — fastest to build, Postgres underneath, self-hostable    │
│                               │ for DPDP                                                                       │
├───────────────────────────────┼────────────────────────────────────────────────────────────────────────────────┤
│ Node/NestJS                   │ Fine                                                                           │
├───────────────────────────────┼────────────────────────────────────────────────────────────────────────────────┤
│ Firebase                      │ ✗ Residency + lock-in                                                          │
└───────────────────────────────┴────────────────────────────────────────────────────────────────────────────────┘

Scope: auth, idempotent event ingest, Culture Pack CDN, report generation, SMS gateway relay, admin. The backend is not the product.

---

Dev-time content pipeline

┌──────────────────┬──────────────────────────────────────────────────────────────┐
│       Task       │                             Tool                             │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│ Voice recording  │ Audacity/Reaper → ffmpeg loudnorm → Opus                     │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│ Image prep       │ ImageMagick / sharp → WebP                                   │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│ Pack build       │ Python script → validate → Ed25519 sign → ZIP                │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│ Item calibration │ Python + py-irt / pymc                                       │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│ Model training   │ PyTorch/Keras → TFLite (INT8)                                │
├──────────────────┼──────────────────────────────────────────────────────────────┤
│ Translation QA   │ Bhashini for first draft; native speaker review is mandatory │
└──────────────────┴──────────────────────────────────────────────────────────────┘

CI: GitHub Actions (flutter analyze → test → build APK). Distribution: Firebase App Distribution for testers, direct APK for pilots.

---

Master summary

┌───────────┬────────────────────────────────────────────────────────────┐
│  Feature  │                     The one-line stack                     │
├───────────┼────────────────────────────────────────────────────────────┤
│ Shell     │ Flutter + Riverpod, Device Owner Lock Task                 │
├───────────┼────────────────────────────────────────────────────────────┤
│ Games     │ CustomPainter + signed JSON manifests                      │
├───────────┼────────────────────────────────────────────────────────────┤
│ Ability θ │ Pure Dart Rasch + Bayesian update                          │
├───────────┼────────────────────────────────────────────────────────────┤
│ Decline   │ Ordered HMM + EWMA changepoint over touch/voice biomarkers │
├───────────┼────────────────────────────────────────────────────────────┤
│ Faces     │ ML Kit detect + MobileFaceNet TFLite                       │
├───────────┼────────────────────────────────────────────────────────────┤
│ Voice out │ Recorded native human audio, Opus, just_audio              │
├───────────┼────────────────────────────────────────────────────────────┤
│ Voice in  │ 20-word DS-CNN keyword spotter, <200KB                     │
├───────────┼────────────────────────────────────────────────────────────┤
│ Content   │ Signed ZIP packs: WebP + Opus + JSON                       │
├───────────┼────────────────────────────────────────────────────────────┤
│ Reminders │ Exact AlarmManager + OEM-aware onboarding + SMS fallback   │
├───────────┼────────────────────────────────────────────────────────────┤
│ Dashboard │ fl_chart + Drift + pdf (widget→image for Indic script)     │
├───────────┼────────────────────────────────────────────────────────────┤
│ Sync      │ Drift/SQLCipher outbox + Nearby Connections sneakernet     │
├───────────┼────────────────────────────────────────────────────────────┤
│ Crypto    │ Keystore + SQLCipher + X25519/XChaCha20                    │
├───────────┼────────────────────────────────────────────────────────────┤
│ Backend   │ FastAPI + Postgres, India region                           │
└───────────┴────────────────────────────────────────────────────────────┘

---

Build tiers — what's actually achievable

┌────────────────────┬──────────────────────────────────────────────────────────────────────────────┬────────────┐
│        Tier        │                                   Contents                                   │ Share of   │
│                    │                                                                              │   effort   │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────┼────────────┤
│ A — must ship      │ Flutter shell, Drift+SQLCipher, 3 game templates, Dart IRT, 1 language fully │ ~60%       │
│                    │  recorded, local reminders, caregiver dashboard, offline outbox, PDF summary │            │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────┼────────────┤
│ B — the            │ Stage transition, DICE log, 2 more languages, keyword spotter, face          │ ~30%       │
│ differentiators    │ recognition, Nearby sneakernet                                               │            │
├────────────────────┼──────────────────────────────────────────────────────────────────────────────┼────────────┤
│ C — demo props     │ Pillbox, NFC stickers, band via Health Connect                               │ ~10%       │
└────────────────────┴──────────────────────────────────────────────────────────────────────────────┴────────────┘
