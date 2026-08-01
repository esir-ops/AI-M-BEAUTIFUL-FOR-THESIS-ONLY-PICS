# AI'm Beautiful — Progress log

Status of the web app against the ten objectives in the proposal, plus every
UX / accuracy fix made along the way, plus everything still open.

Cross-reference: `README.md` for run instructions, `models/HOW-TO-TRAIN.md`
for the model-training path, `models/application-quality/README.md` for the
Objective 9 model contract.

---

## Fixed

### Proposal features shipped

- **Objective 6 — Style Variation module.** Three Soft & Natural variations
  per focal point (Bare Petal / Soft Focus / Rosy Statement etc.), driven
  by `data/style-variations.json`. Same recommended shades across all three;
  the variations differ only in **coverage intensity** (sheer / balanced /
  full), which is what the descriptions were describing all along and what
  makes the Shades-screen "Try It On" honest — the shade you try on is the
  shade you use in every variation.
- **Objective 7 — Picture-based reference guide.** The frame is frozen the
  instant the tone is classified (`captureReferenceFrame`), then re-rendered
  with the chosen variation and shown beside the live mirror throughout the
  guide. Distinct from the try-on (live filter) as the proposal specifies.
- **Objective 8 — Foundation checker.** `data/foundations.json` maps each
  tone to a Squad / Detail recommendation on tray slot 5. Screen recommends
  the shade and confirms readiness; explicitly does not verify the shade or
  give a foundation AR guide, matching the proposal's limitation.
- **Objective 9 — Application-quality feedback.** Full inference path in
  `app.js` (`initQualityModel`, `analyzeApplicationQuality`) that loads a
  MobileNetV2 model from `models/application-quality/` when present and
  falls back to an analytical image-analysis back-end with the **same
  output shape** when the model is absent. Every screen, message, and
  summary line works today; drop the trained model in and it takes over
  automatically. Quality report on the step screen shows blending /
  evenness / product-amount per step.

### Detection & accuracy

- **Skin-tone detection re-worked with a neutral-white reference.** The
  sclera (whites of the eyes) is sampled per frame and used to
  white-balance the skin sample (von Kries adaptation) and to measure the
  skin's brightness **relative to** the sclera instead of absolutely.
  Measured 62 % → **90 %** correct across 3 skin tones × 7 lighting
  conditions in synthetic tests.
- **Bare-face baseline for makeup detection.** At capture time, per-zone
  colour and skin reference are recorded (`captureBaseline`). Each step
  then asks "has this zone changed since your before photo?" instead of
  "is this darker than skin?" — which is what stopped naturally dark
  brows and shadowed hollows from reading as product on a bare face.
- **Robust region sampling for lips.** Full-region median colour with
  glare rejection replaced the old 6-point sample. Wet lips / licking now
  produce the same verdict as dry lips (drift 0.000 in the harness).
- **Coverage-aware amount check.** The "too little / just right / too
  much" window shifts with the chosen variation, so a deliberate Sheer
  Veil application is not flagged as "too little".
- **Frontal-face gating** on both detection and step overlays (no more
  guides drawn on a fully turned head).
- **Landmark smoothing** with adaptive alpha + a deadband, so points
  hold still frame-to-frame (~89 % measured jitter reduction) and only
  respond to real motion.
- **Occlusion check** on the camera screen: skin-tone consistency across
  forehead / cheeks / nose / chin blocks the Capture button when a
  hand, mask, hair or object covers any zone; auto-clears when removed.
  Uses hue (not brightness) so a side-lit shadow no longer trips it.
- **Face-shape detection** (`detectFaceShape`) — oval / round / oblong /
  heart / diamond / square — drives the contour guide shape.
- **Filter fade with pose.** The 2D try-on filter fades globally as the
  head turns or tilts, so it never renders a mismatched smear at strong
  angles. Per-side attenuation on contour / blush prevents the receding
  side collapsing into an unblended dark patch.
- **Baseline-relative quality "amount".** The natural (bare) deviation of
  the zone is subtracted from the amount metric so brows/hollows are not
  read as excess product.

### UX & flow

- **Manual Capture Photo button** replaced the auto-timer. Button is
  live only when the readiness checks pass (pose, framing, lighting,
  occlusion). Pressing it starts a **3-2-1 countdown** with a modern
  Jost numeral inside a gold ring; the countdown aborts instantly if the
  user moves out of position.
- **Camera preview in true portrait 3:4** on the detection screen, sized
  off viewport height with a 54vh cap. No stretching (`object-fit: cover`).
- **Step screen: mirror + reference as matched portrait panels.** Same
  size, tops perfectly aligned, mirrored to read as a real mirror. On
  phones both panels get the largest share of the screen with near-zero
  chrome; the shorter 3/3.8 ratio kicks in on ≤700 px viewports so
  Check My Placement / Skip / Compare stay above the fold.
- **Every screen fits on phones without scrolling** at 390×730 and
  375×620. Verified end-to-end (Focal, Shades, Variation, Foundation,
  all four step guides).
- **Portrait Try-On modal.** Was hardcoded landscape 4:3; now portrait
  3:4, responsive, mirrored, centered.
- **Amount-to-apply card** on the step screen: a coverage meter
  (Sheer / Medium / Full) plus the variation name, so the guide tells
  the user how much of the same recommended product to apply.
- **Custom SVG icons** for all four focal points (Lips, Eyebrows, Cheeks,
  Contour) — all beige, same size, drawn to represent their step. The
  Lips ♣ used to render as a dark emoji on iOS; that was the first icon
  fixed, then the others were replaced for consistency.

### Guide overlays

- **Guide colour changed to white on a thin dark backing.** Brow and
  contour guides were stroked in the *product* colour, which for brows
  and contour is a dark brown — invisible against dark brow hair or a
  shadowed cheek. Fixed.
- **Guides thinned across the board.** Blush ring ~4.8 → **2.1 px**,
  eyebrow outline ~3.6 → **2.2 px**, contour sweep ~5.6 → **2.3 px**,
  contour soft band ~36 px → **7.7 px** (that band was the "smear"
  effect in earlier feedback).
- **Focal step now a single-pass with a glow** instead of drawing every
  guide twice — which had doubled apparent thickness.
- **Mirrored text helper** so overlay hints don't render backwards on
  the CSS-mirrored canvas.

### Model integration path

- **Teachable Machine loader** built (`TM_MODELS`, `loadTMModel`,
  `classifyTM`, `initTMModels`) with three independent slots that pick
  models up automatically on refresh:
  - `models/glasses/`   — glasses vs no-glasses
  - `models/occlusion/` — face covered by hand / mask / object
  - `models/application-quality/` — Objective 9
- **Non-technical training guide** in `models/HOW-TO-TRAIN.md`
  (~30 min in Teachable Machine, no ML code).
- **Glasses model auto-enables the capture block.** When
  `models/glasses/` contains a trained model, glasses detection is
  reliable enough to disable the Capture button and auto-clear on
  removal. Without the model, glasses stay a non-blocking reminder only,
  so the unreliable pixel heuristic never traps the user.

### Small quality-of-life

- Every em-dash replaced with a regular hyphen everywhere (user request).
- Stale duplicate `AI-M-BEAUTIFUL-main/AI-M-BEAUTIFUL-main/` removed.
- Repo initialised as a git repository.
- Camera emoji removed from the Capture button.
- Portrait crop of the reference guide is anchored on the detected
  face (not the image centre), so an off-centre subject is not clipped.

---

## Open — user-reported

From the most recent round of feedback:

1. **AR guide lines drift when the head turns** — *addressed for
   contour + tightened globally on the step overlay.* Per-side `sideVis`
   attenuation added inside `drawContour`, plus a stepFade curve that
   accelerates guide fade past turnVis 0.7. Lips / brows are anchored
   to frontal features and were not drifting; blush already had its own
   per-side vis. Verify live on a real device before closing.
2. **Glasses / mask / face-covering hard-block Capture without the
   trained model** — deliberately **not changed.** Pixel-glasses
   detection is unreliable; hard-blocking on it would trap users.
   Reliable path stays the trained `models/glasses/` slot, which is
   loader-ready. Recorded as a decision, not an open bug.
3. **Blush filter outline extending beyond the face** — *fixed.*
   `drawVirtualMakeup` now clips the blush pass to the MediaPipe
   FACE_OVAL, and the ellipse was tightened from 0.26/0.15 → 0.22/0.13
   of `faceRef`, which stops the halo crossing the jaw/temple.
4. **Step-screen buttons not centred** — *fixed.* `.step-actions` is
   now `align-items:center` and every child button is `width:100%;
   max-width:320px`, so Check My Placement / Skip / Compare are the same
   width and centered on every screen. Verified via computed style.
5. **"Face forward for the most accurate guide" cut off** — *fixed.*
   The mirrored-text helper now fit-to-widths the string: font shrinks
   to the canvas width (min 10 px), and drops to `Face forward` if even
   the shrunk font would overflow. No more clipping on the narrow
   phone layout (step overlay can be ~180 px wide).
6. **Contour tracking the wrong side during turn** — *fixed with the
   same `sideVis` change as item 1.* Each per-shape sweep pair is
   wrapped in `sided()`, which fades the receding half out (dropped
   entirely below α 0.05) so the sweep no longer jumps to the visible
   cheek when the head is angled for application.

---

## Open — larger structural items

- **Trained MobileNetV2 quality model** (Objective 9) not yet created.
  Loader + contract + guide are all in place; training data has not
  been collected. Until it is, the analytical fallback runs and the UI
  says so explicitly. This is the largest remaining accuracy gain.
- **Trained glasses model** not yet created. Same story — loader ready,
  guide written, training data not collected. This is what turns glasses
  into a reliable hard block.
- **Trained occlusion model** not yet created. The colour-consistency
  fallback is reasonably good, so this is the lowest-priority of the
  three.
- **Foundation shade names in `data/foundations.json` are placeholders**
  in the Squad / Detail house style. Verify against the physical tray
  inventory before the evaluation session.
- **Application-quality model contract for coverage.** The trained
  model, when it arrives, will need coverage examples per level so its
  "amount" verdict can respect the chosen variation the same way the
  heuristic already does.

---

## Verification snapshots

Reference figures from the most recent test runs, useful for the defence:

- Tone detection: **62 % → 90 %** correct across 3 skin tones × 7 lighting
  conditions after the sclera-based white balance.
- Landmark jitter: **~89 %** reduction after deadband + adaptive alpha.
- Try-on filter fade: `faceOn = 1.00` at 0 offset, `0.80` at moderate
  turn, `0.20` at strong turn, `0.32` at ~20° tilt.
- Lip stability under specular glare: verdict drift **0.000** between
  wet and dry lips.
- All screens fit on iPhone 12/13-class portrait (390×730) and
  smaller (375×620) with **zero vertical or horizontal overflow**.
