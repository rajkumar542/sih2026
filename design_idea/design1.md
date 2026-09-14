
Brainstorm only — nothing written to disk. Here's the full architecture, grounded in both the research and the a–h reading.

---
The Organizing Principle

Everything below follows from one decision:

▎ There is exactly one brain — a CareState kernel. Every feature is a renderer of it, not an owner of it.

CareState = { stage, ability θ (per domain), frustration, time-of-day, caregiver presence, connectivity }

Games, reminders, voice, UI density, and the dashboard all subscribe to it. Change the stage and eight features change at once, because none of them holds its own copy of the truth. That makes "the app changes as the disease changes" an architectural property, not 400 scattered if statements.

This is the differentiator, and it's free — it's the same code you'd write anyway, organized correctly.

---
End-to-End Layer Stack

┌─ PRESENTATION ───────────────────────────────────────────┐
│  Calm Shell (patient)  │  Console (caregiver)  │ CHW mode │
│  ← density, modality, and content all stage-rendered      │
├─ INTERACTION ────────────────────────────────────────────┤
│  Voice out (primary)  │  Touch  │  Passive sensing        │
├─ ⭑ CARESTATE KERNEL ⭑ ───────────────────────────────────┤
│  Stage Engine │ Ability Tracker │ Frustration Monitor     │
│  Biomarker Fusion │ Escalation Policy                     │
├─ DOMAIN MODULES ─────────────────────────────────────────┤
│ Cognitive │ Reminiscence │ Routine │ Reminder │ Behaviour │
├─ CONTENT ────────────────────────────────────────────────┤
│  Culture Packs (per community)  │  Memory Vault (family)  │
├─ DEVICE ABSTRACTION LAYER ───────────────────────────────┤
│  every device = signal producer + actuator; all optional  │
├─ DATA & SYNC ────────────────────────────────────────────┤
│  Append-only event log │ Outbox │ SQLCipher │ Consent ledger│
├─ TRANSPORT (degrades gracefully) ────────────────────────┤
│  WiFi → 4G → 2G/SMS digest → CHW sneakernet → QR export   │
└──────────────────────────────────────────────────────────┘

Two seams matter most: the kernel (makes staging architectural) and the Device Abstraction Layer (makes all hardware genuinely optional, so AR becomes a roadmap tier instead of a dependency).

---
Feature-by-Feature: Options, Pick, Impact

(a) Cognitive games — 5 domains

┌──────────────────────────────────────────────────┬──────────────────────────────────────────┐
│                      Option                      │                 Verdict                  │
├──────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Hardcode each game                               │ ✗ Every culture/stage variant = new code │
├──────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Game-as-config: template + content pack + params │ ✓ Pick                                   │
├──────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Procedural generation                            │ ✗ Unpredictable difficulty, unsafe here  │
├──────────────────────────────────────────────────┼──────────────────────────────────────────┤
│ Unity/third-party game lib                       │ ✗ Wrong for the other 80% of the app     │
└──────────────────────────────────────────────────┴──────────────────────────────────────────┘

Seven templates cover all five domains:

┌────────────────────┬──────────────────────────────┬────────────────────────────────────────────────────┐
│      Template      │            Domain            │                NER content example                 │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Match-Pair         │ Short-term memory            │ Puanchei / Mekhela Chador motif tiles              │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Sequence-It        │ Daily routine recall         │ Order the steps of making regional tea             │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Sort / Odd-One-Out │ Pattern & object recognition │ Local flora, fauna, household items                │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Hold-the-Target    │ Attention & concentration    │ Single stimulus, zero distractors, rising duration │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Who's This?        │ Long-term memory + social    │ Memory Vault family photos                         │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Story-Then-Recall  │ Verbal / long-term           │ Folk tale audio → 2 simple questions               │
├────────────────────┼──────────────────────────────┼────────────────────────────────────────────────────┤
│ Reminisce-Tap      │ Emotional engagement         │ No win state. Tap a photo, hear a story            │
└────────────────────┴──────────────────────────────┴────────────────────────────────────────────────────┘

Non-negotiable rule: every template must support a no-fail variant. Research is unambiguous — one frustrating session causes permanent abandonment. Above GDS 4, the fail state is removed entirely.

Impact: the visible product — but per the thesis, dominant only GDS 2–4.

---
(b) Adaptive AI — the most important decision

┌───────────────────────────────────────────────────────────┬───────────────────────────────────────────┐
│                          Option                           │                  Verdict                  │
├───────────────────────────────────────────────────────────┼───────────────────────────────────────────┤
│ Rule-based ladder                                         │ Safe, explainable, but no real "AI" story │
├───────────────────────────────────────────────────────────┼───────────────────────────────────────────┤
│ Q-learning / RL (your research doc's proposal)            │ ✗ Reject — and say why on stage           │
├───────────────────────────────────────────────────────────┼───────────────────────────────────────────┤
│ Bayesian latent-ability (IRT/Rasch + Kalman-style update) │ ✓ Pick                                    │
├───────────────────────────────────────────────────────────┼───────────────────────────────────────────┤
│ Contextual bandit                                         │ Viable, but harder to explain to judges   │
└───────────────────────────────────────────────────────────┴───────────────────────────────────────────┘

Why reject the RL your own doc specifies — three independent reasons, and presenting this rejection is a strength:
1. Cold start. A dementia patient may play 10–30 sessions lifetime. Q-learning needs thousands. It will never converge.
2. Exploration is clinically harmful. 20% ε-greedy means 1 interaction in 5 is deliberately the wrong difficulty, for a user the same document says abandons permanently after one bad session.
3. Non-stationarity. RL assumes a stable environment. The patient is monotonically declining — the thing RL needs to be fixed is the thing being measured.

What to build instead:

- P(correct) = σ(θ − b) — Rasch model, items pre-calibrated for difficulty b
- Each answer Bayesian-updates θ with its uncertainty σ²
- Process noise between sessions models decline + daily fluctuation. A long gap → higher uncertainty → the system automatically plays it safe
- Target P(correct) ≈ 0.80–0.85, not 0.5. Adaptive testing targets 0.5 for maximum information; adaptive therapy targets high success for flow. Getting this backwards is the single most common mistake in cognitive-training apps
- Asymmetric envelope: 2 consecutive misses → drop immediately. Never raise more than one level per session. ← you independently derived this in your own requirements doc

Two models, deliberately separated by timescale:

┌─────────────────────┬─────────────────┬──────────────────────────────────┐
│                     │    Timescale    │             Purpose              │
├─────────────────────┼─────────────────┼──────────────────────────────────┤
│ Ability tracker (θ) │ Seconds–minutes │ In-session difficulty            │
├─────────────────────┼─────────────────┼──────────────────────────────────┤
│ Decline estimator   │ Weeks–months    │ Stage proposal + dashboard trend │
└─────────────────────┴─────────────────┴──────────────────────────────────┘

Conflating them is the standard failure. A bad afternoon must not look like disease progression.

Digital biomarkers feed model 2, free from the tablet: tap latency distribution, touch-trace jitter (tremor), hesitation before first tap, speech rate and pause ratio, session time-of-day drift. This is your strongest AI credibility and costs zero hardware.

Impact: satisfies (b)'s two stated inputs — "performance" (θ) and "cognitive condition" (stage).

---
(c) Multilingual + voice — contrarian pick

┌───────────────────────────────────────────────────────────────────┬──────────────────────────────────────────────────┐
│                              Option                               │                     Verdict                      │
├───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────┤
│ Cloud Bhashini at runtime                                         │ ✗ Violates (g) outright                          │
├───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────┤
│ On-device Indic TTS                                               │ Fallback only                                    │
├───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────┤
│ On-device full ASR (IndicWhisper)                                 │ ✗ Collapses on elderly dysarthric dialect speech │
├───────────────────────────────────────────────────────────────────┼──────────────────────────────────────────────────┤
│ Pre-recorded native human voice + closed-grammar keyword spotting │ ✓ Pick                                           │
└───────────────────────────────────────────────────────────────────┴──────────────────────────────────────────────────┘

The app's speech is a finite set — roughly 300–500 phrases. Record them with native speakers from the actual community. Result: perfect prosody, real local accent, 100% offline, near-zero latency, tiny footprint. Dynamic values (names, times) handled by concatenative fragments.

A real grandmother-voice from the same valley outperforms any TTS for this population — and it directly satisfies (d)'s "culturally familiar sounds," which synthetic speech never will.

Voice input stays off the critical path. A ~20-word closed keyword spotter per language (tiny CNN on mel-spectrograms) is far more robust than open ASR and trainable with modest data. Touch is always the guaranteed fallback.

Free high-value addition: an always-on local distress phrase detector — "help," "I'm lost" in the native language → instant caregiver alert, fully offline. Your research explicitly calls for exactly this.

Language scope: ship 3 deeply — Assamese (largest base, ICMR-NCTB validated in Assamese), Khasi (Meghalaya's 2022 policy, NEIGRIHMS, matrilineal caregiving), Mizo (active ARDSI chapter, high DALY). Architect for N; claim "Bhashini-ready," don't claim 20.

Impact: also neatly sidesteps the Kokborok script war and the Assamese Unicode collation bug — you're not rendering much text at all.

---
(d) Cultural themes

Pick: Culture Packs + Memory Vault, together.

- Culture Pack = signed, versioned bundle per community: local objects/flora/fauna, textile motif textures, instrument audio (Pepa, Gogona, Khuang, Pena, Tangmuri), folk songs, voice pack, lexicon. ~50–200MB, downloadable or side-loaded by the CHW.
- Memory Vault = family-uploaded photos, labeled relatives, recorded voice messages. Costs you nothing, is the highest-value reminiscence asset, and creates real switching cost.

Architectural payoff: the Memory Vault's labeled face embeddings are the same dataset that would later power any AR face-recognition tier. Software and hardware share one source of truth.

⚠️ Generic "pan-NER" content fails. A Mizo elder needs Khuang, not a generic Northeast sampler. Per-community or it doesn't work.

Partnership angle: NEHHDC holds motif archives and supports 21,250 artisans — a real, citable content pipeline.

---
(e) Reminders — your best 30-second demo

Pick: local escalation ladder whose recipient shifts by stage.

gentle chime + photo card
   → spoken prompt, native voice
      → repeat, higher salience
         → escalate to caregiver (in-app / SMS)
            → log as missed

Fires 100% on-device via the OS alarm scheduler. No server, ever.

┌─────────┬─────────────────────────────────────┬──────────────────────────────────────────────────────────────┐
│  Stage  │           Who receives it           │                             Tone                             │
├─────────┼─────────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ GDS 2–4 │ Patient                             │ "Shall we take your morning medicine?" — autonomy-preserving │
├─────────┼─────────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ GDS 5   │ Patient, caregiver notified on miss │ Supportive                                                   │
├─────────┼─────────────────────────────────────┼──────────────────────────────────────────────────────────────┤
│ GDS 6–7 │ Caregiver is primary                │ "Time to help with hydration"                                │
└─────────┴─────────────────────────────────────┴──────────────────────────────────────────────────────────────┘

This one feature demonstrates the entire thesis in half a minute. Same reminder, three different products.

No-hardware adherence logging: tap a photo to confirm the dose. Works offline, costs nothing, replaces the pillbox for v1.

---
(f) Caregiver + CHW dashboards

Pick: one APK, three PIN-gated roles. (The patient must not be able to wander into settings — that's a real dementia UX requirement, not a nicety.)

The dashboard's content changes by stage:

┌──────────┬───────────────────────────────────────────────────────────────┐
│  Stage   │                   What the console becomes                    │
├──────────┼───────────────────────────────────────────────────────────────┤
│ Early    │ Cognitive trends, domain breakdown, baseline                  │
├──────────┼───────────────────────────────────────────────────────────────┤
│ Moderate │ DICE behavior log, sundowning heatmap, trigger correlation    │
├──────────┼───────────────────────────────────────────────────────────────┤
│ Severe   │ Nursing log — fluids, sleep, agitation, meds → Doctor Summary │
└──────────┴───────────────────────────────────────────────────────────────┘

DICE in 4 taps: caregiver taps a chip (agitated / wandering / refused food) → app asks 2–3 context questions (Describe + Investigate) → suggests an intervention from a library (Create) → asks "did it help?" next time (Evaluate). That turns a clinical framework into something an exhausted caregiver will actually use.

Sundowning detection: plot agitation events by hour; cluster at 16:00–20:00 → flag it and pre-emptively warm the UI and queue calming audio before 16:00. Clinically real, computationally trivial.

The Doctor Summary — weeks of offline data compiled into one shareable page, synced when the CHW arrives — is straight from your research and is the single most compelling artifact for a region where the patient physically cannot reach a neurologist.

Caregiver wellbeing check-in (short Zarit-style burden scale + surfacing ARDSI Guwahati/Aizawl, Tele-MANAS 14416, Senior Citizen helpline 14567). Research calls the caregiver the "invisible second patient" with elevated depression, CVD, and their own dementia risk. No competing team will build for the caregiver's health. Cheap, compassionate, unforgettable in a pitch.

---
(g) Offline

Pick: offline-first, append-only event log, degrading transport ladder.

WiFi → 4G/3G → 2G SMS digest → CHW sneakernet (BLE/WiFi-Direct) → QR export

- Medical logs append-only → they can never conflict. Config is last-writer-wins with caregiver priority.
- SMS digest for zero-data areas: "Ma: 5/7 activities, meds 90%, 2 agitation events." Research confirms 2G/GSM is the realistic NER floor.
- CHW sneakernet — the patient device encrypts and signs a bundle; the CHW phone carries it opaquely and cannot decrypt it; it uploads on reaching signal. Your stages.txt literally describes this workflow ("or a visiting health worker arrives"). Nobody else will build a human transport layer, and it is exactly right for this terrain.

---
(h) Elderly UI — the "Calm Shell"

Pick: purpose-built kiosk shell, stage-rendered density.

Grounded principles, each traceable to a research finding:

┌──────────────────────────────────────┬───────────────────────────────────────────────────────────────────┐
│              Principle               │                                Why                                │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ One decision per screen, max depth 2 │ Executive function decline → menus cause confusion                │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Touch targets ≥15mm, wide spacing    │ Tremor, motor decline                                             │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Warm high contrast, no thin fonts    │ Lens yellowing — avoid blue-on-dark                               │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Near-zero text                       │ 12.23% prevalence in no-formal-education cohort + script politics │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ No timers, no countdowns             │ Time pressure = anxiety                                           │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ No red, no "Wrong!"                  │ Always "let's try this one"                                       │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Tap only above GDS 4                 │ Visuospatial decay kills drag gestures                            │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Auto-warming evening theme           │ Sundowning                                                        │
├──────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ Caregiver gesture to exit shell      │ Patient must not escape into settings                             │
└──────────────────────────────────────┴───────────────────────────────────────────────────────────────────┘

Density literally falls with stage: GDS 3 = 6 tiles → GDS 5 = 2 tiles → GDS 6 = full-screen photo + music, no tiles at all. The thesis, visible.

---
⚠ Unlettered: Secure data management

(The trap — in the Expected Solution, absent from a–h.)

Pick: on-device encryption + strict data minimization.

- Raw never leaves the device. Audio → features on-device → only derived labels sync. Photos stay local by default.
- DPDP 2023 consent ledger — consent by legally authorized representative, because capacity is impaired by definition. Recorded, scoped, timestamped, revocable. Dementia makes informed consent genuinely hard; say so, don't hide it.
- ICMR HITL: the app never diagnoses. It reports observations and says "discuss with a doctor." Ethics and legal shielding.
- Per-patient keys (CHW mule can't decrypt); full audit log of dashboard access.

---
⚠ Orphan: Social interaction

(Stated twice as a goal, has no letter and no deliverable — nothing in a–h produces it.)

Pick: Voice Postcards. Family records a 20-second voice note + photo. It lands as a big tappable face in the patient's shell. Patient taps to hear it, and can send back a one-tap "thinking of you." Audio is tiny — works on 2G.

The extra that makes a room go quiet: grandchildren record the folk tales used in Story-Then-Recall. The game content is the family's voice. Intergenerational transmission, reminiscence therapy, and a language-preservation contribution in one feature.

Plus: CHW-facilitated group CST in a village hall — CST is evidence-based, ARDSI already runs day-care, and it attacks the social isolation the problem statement explicitly names.

---
The Unique Full Flow

1 · Enrollment (CHW or family, ~10 min, fully offline)
Language + community → Culture Pack loads. A short calibration — framed as calibration, never diagnosis — sets initial θ and stage band, using a non-literate-friendly adaptation (arrange sun positions for time of day rather than draw a clock face; ICMR-NCTB cited as the clinical reference). Consent captured. Memory Vault seeded with 10 photos in the caregiver's own voice.

2 · Daily patient loop (10–20 min)
Greeting by name in native voice → 3–4 activities chosen by CareState → every tap emits telemetry → frustration monitor can abort a game mid-way and pivot to a no-fail reminiscence activity → reminders fire independently → evening sundowning window warms the UI and offers calming Culture Pack audio.

3 · Daily caregiver loop (2–5 min)
One glanceable card. Optional 4-tap DICE entry. Weekly: one plain-language insight — "evenings have been harder this week."

4 · Sync — opportunistic, whenever any transport appears.

5 · CHW visit (monthly) — pulls encrypted bundles over BLE, pushes Culture Pack and app updates, walks the family through the Doctor Summary, carries bundles to signal.

6 · ⭑ Stage transition ⭑ — the part nobody else will have
The slow model watches θ trend + biomarker drift + caregiver ADL flags + BPSD frequency. On sustained decline it proposes — never silently applies — a change:

▎ "We've noticed evenings have been harder, and picture-matching seems tiring now. Shall we switch to gentler activities?"

Caregiver confirms (HITL). On confirm: UI simplifies, games become no-fail reminiscence, voice becomes primary, reminders re-route to the caregiver, the dashboard grows DICE and nursing tools.

The app becomes a different app. Same install, same history, same person.

That is your 60-second demo.

---
AR & Devices — Optional Tiers

The Device Abstraction Layer means every device is just another signal producer. With none attached, the tablet's own sensors fill every role. Hardware is genuinely optional, not bolted on.

┌──────┬────────────────────────────────────────┬─────────┬────────────────────────────────────────────────────────────────────────┐
│ Tier │                  Item                  │  Cost   │                                  Take                                  │
├──────┼────────────────────────────────────────┼─────────┼────────────────────────────────────────────────────────────────────────┤
│ 0    │ Tablet only — camera, mic, touch,      │ ₹0      │ Ship this. Delivers biomarkers + distress detection                    │
│      │ accelerometer, GPS                     │         │                                                                        │
├──────┼────────────────────────────────────────┼─────────┼────────────────────────────────────────────────────────────────────────┤
│ 1    │ BLE fitness band → sleep, steps, HR    │ ₹1.5–3k │ Best value/effort in the whole list. Sleep disruption is a GDS 6       │
│      │                                        │         │ marker and a top caregiver pain point                                  │
├──────┼────────────────────────────────────────┼─────────┼────────────────────────────────────────────────────────────────────────┤
│ 1    │ Smart pillbox (ESP32 + IR + load cell  │ ~₹1.5k  │ Great table demo. Research: 100% missed-dose detection, +47% adherence │
│      │ + SIM800 2G)                           │         │                                                                        │
├──────┼────────────────────────────────────────┼─────────┼────────────────────────────────────────────────────────────────────────┤
│ 1    │ BLE tag on keys/wallet                 │ ₹300    │ Solves the exact problem AR glasses solve, for 1% of the cost          │
├──────┼────────────────────────────────────────┼─────────┼────────────────────────────────────────────────────────────────────────┤
│ 2    │ mmWave radar fall/overstay             │ ₹2–4k   │ "It sees the fall, not the person." Ethically outstanding — no camera  │
│      │                                        │         │ in a bathroom                                                          │
├──────┼────────────────────────────────────────┼─────────┼────────────────────────────────────────────────────────────────────────┤
│ 2    │ Bed-exit pressure pad                  │ ~₹800   │ Night wandering, cheaper and less intrusive than smart socks           │
└──────┴────────────────────────────────────────┴─────────┴────────────────────────────────────────────────────────────────────────┘

On AR glasses — I'd push back, then reframe

Your own research's numbers: 61% object detection, 88.89% face recognition in good light, Raspberry Pi 4 on the head, 9–18h on a tethered 10,000mAh battery bank — for a 78-year-old in a dim wooden house. The same doc concedes ergonomic/battery problems for older adults and that pendants are preferred over 75.

61% means wrong 2 times in 5. For a dementia patient, a confidently wrong answer is worse than silence — it manufactures the confusion you're treating.

So: the form factor is wrong; the idea is right. Three better options:

- (a) Pendant camera — research-preferred for 75+, 16.2h battery, no head weight
- (b) 🏆 The tablet as a handheld AR window — hold it up, it names the face. Zero new hardware, reuses Memory Vault embeddings. This is what I'd demo — it proves the capability for ₹0
- (c) Audio-only AR — cheap BLE earbud, no display. The "AR" is a whisper: "That's Rina, your daughter."

The original argument worth making on stage: AR for dementia should be auditory, not visual. Your own research says the auditory cortex and musical memory survive into severe dementia while visuospatial processing degrades early. A HUD overlay targets the failing channel; a whispered name targets the intact one. That's a research-backed inversion of the obvious, and no other team will say it.

Also: wandering/geofencing needs no glasses at all — a phone in a pocket or a ₹1,000 GPS tag does it. Decouple the two.

The device idea I like most

NFC stickers on real objects. ₹5 each, stuck on a physical photo album, a teacup, a shawl. Tap the tablet to the object → the related reminiscence plays. For someone whose procedural memory is intact but digital literacy is zero, touching a real object beats navigating any menu. It bridges physical familiarity to digital content at essentially no cost.

---
Alternative Whole-System Architectures

┌─────────────────────┬────────────────────────────────────────┬───────────────────────────────────────────────────────────────────┐
│         Alt         │                 Pitch                  │                              Verdict                              │
├─────────────────────┼────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ A · Voice-only      │ No games. A talking presence in the    │ ✗ as hero — fails (a), (f). ✓ But it's exactly right as the GDS   │
│ companion           │ local language                         │ 6–7 mode                                                          │
├─────────────────────┼────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ B · Caregiver-first │ Caregiver is the real adopting user;   │ ✗ as hero — under-serves (a)(b)(c)(h). ✓ But it means the console │
│                     │ burden is the measurable outcome       │  must be first-class                                              │
├─────────────────────┼────────────────────────────────────────┼───────────────────────────────────────────────────────────────────┤
│ C ·                 │ Matches the research's #1 policy rec   │ ✗ as hero — PS never asks for screening, and the demo becomes a   │
│ CHW/clinic-first    │ (task-shifting to ASHAs)               │ spreadsheet. ✓ But keep CHW mode as sync mule + deployment        │
│                     │                                        │ channel                                                           │
└─────────────────────┴────────────────────────────────────────┴───────────────────────────────────────────────────────────────────┘

The closing argument: the primary architecture already absorbs all three — A becomes the late-stage face, B becomes a first-class console, C becomes CHW mode. That isn't a compromise. That's the stage-adaptive thesis doing its job.

---
Stack (brief)

Flutter, Android-only (iOS is irrelevant for NER tablets) — one codebase, full custom rendering control for the Calm Shell, real kiosk support. Drift/SQLite + SQLCipher; TFLite/ONNX Runtime for on-device models; WorkManager for alarms and sync. Backend deliberately small — FastAPI + Postgres as a sync endpoint and report generator. The backend is not the product.

(Unity → wrong for 80% of the app. React Native → weaker custom rendering. Native Kotlin → best control, slowest to build.)

⚠️ Naming caution: avoid a Sanskrit/Hindi name. In NER, Hindi-origin branding reads as imposition — the script and language politics in your own research make that a real risk.

---
What I Need From You

1. Hardware tier — 0, 1, or 2? My vote: Tier 0 for the build, Tier 1 as one physical prop, AR reframed as audio-first on the roadmap.
2. Does the demo show the stage transition? My vote: yes — it's the only thing nobody else will have.
3. Three languages, or two done beautifully?
4. Voice Postcards + family-recorded folk tales — in or out? (My vote: in. Cheapest emotional payload in the entire design.)
5. Is the CHW role in v1, or software-only patient+caregiver?


※ recap: We're designing the SIH dementia-care platform for Northeast India, and I've laid out a full stage-adaptive architecture covering all features plus AR and device options. Next, you pick from the five open decisions, starting with hardware tier and whether the demo shows the stage transition. (disable recaps in /config)
