# Damage Measurement Tool — Project Context

## Goal

Build a web-based tool to measure physical damage (dents, scratches,
cracks) from photographs, with millimetre-level accuracy, for internal
professional use. The tool must run on both mobile and desktop from a
single codebase.

## Current state

A working single-file HTML/CSS/JavaScript app: `index.html`.
Hosted on GitHub (private repo), deployed via GitHub Pages.

### Implemented features

- Photo loading from camera or gallery, including HEIC/HEIF support
  for modern iPhones and Android phones (converted client-side to
  JPEG via heic2any).
- Automatic scale calibration via ArUco marker detection (DICT_4X4_50
  dictionary). Marker IDs map to physical sizes via a configurable
  table:
    - ID 0 → 14.75 mm  (small damage, < 40 mm)
    - ID 1 → 49.75 mm  (medium damage, 50–200 mm)
    - ID 2 → 99.75 mm  (large damage, > 200 mm)
  When several markers are detected, the largest one (most pixels per
  side) wins, since it yields the most precise scale.
- Perspective correction (phase 6): after detecting the ArUco marker,
  the app applies cv.warpPerspective using the four detected corners
  to produce a rectified image where mm/pixel is constant across the
  whole frame. The original (un-rectified) image is kept in
  state.originalPhoto for the "view original" comparison button.
  Corner ordering is normalised to canvas-aligned TL/TR/BR/BL before
  the transform, so any physical orientation of the marker works.
  A sanity check (cv.perspectiveTransform on the four corners) verifies
  the matrix before applying it; if it fails the app falls back to
  non-rectified calibration and logs a warning.
- "View original" button (⊙ Original): visible in measure-idle phase
  when a rectified image is loaded. While held, shows the un-rectified
  photo with no overlays at the current zoom/pan, for before/after
  comparison. Releases back to the rectified view on pointer-up or
  pointer-leave.
- Safe zone overlay: a dashed cyan rectangle covering the central 70%
  of the image (SAFE_ZONE_RATIO = 0.70), shown from the moment a photo
  is loaded until the first calibration point or dimension is placed.
  Guides the inspector to keep both the marker and the damage within
  the low-distortion central region. Disappears permanently after first
  use and does not appear on exported JPEGs.
- Perspective tolerance check: photos with side-length variance above
  PERSPECTIVE_TILT_LIMIT = 1.10 trigger a blocking modal with two
  choices: retake the photo, or continue under user responsibility
  (badge turns red with ⚠).
- Visual marker representation: when calibration is automatic, the
  detected marker is drawn as a closed quadrilateral with a label
  showing its real side length. Tilted markers therefore appear
  visibly as trapezoids, making perspective distortion evident.
- Manual two-point calibration as fallback for photos without a
  marker, with onboarding modal and reference value entry.
- Enriched scale badge:
    - Automatic calibration: "ID X — Y.YYY mm/px" (yellow).
    - Manual calibration: "Y.YYY mm/px" (yellow).
    - Tilted photo accepted under user responsibility: red with ⚠.
- Multiple named dimensions per photo: add, rename, hide, delete.
- Pinch-zoom and pan for precise point placement.
- Clean view (overlays hidden, photo only), save as JPEG with marks
  baked in, native share via Web Share API.
- Optional password gate (SHA-256 hash, client-side only). Disabled
  by default; configurable at top of script.
- Loading overlay while OpenCV.js initialises (~10 MB WebAssembly,
  needs a few seconds on first load).

### Pending work (current focus)

- Phase 8 — Convert to PWA (manifest + service worker for installable,
  offline-capable app). This is the next task.

### Deferred to separate chats

- Phase 9 — Migrate to Azure Static Web Apps with Entra ID
  authentication (later phase).
- Native packaging (APK / .exe): deferred per original spec; only
  revisit if a concrete reason emerges.

## Implementation phases (historical reference)

1. ✅ Set up local dev environment (VS Code + clone repo).
2. ✅ Print physical ArUco markers (15/50/100 mm). Real measured
   sizes after printing: 14.75 / 49.75 / 99.75 mm. Validated against
   credit card and 1 € coin as ground-truth references.
3. ✅ Load OpenCV.js locally (bundled, no CDN).
4. ✅ Detect marker on photo load; compute mm/pixel automatically.
5. ✅ Integrate detection into the main photo-load flow
   (sub-edits F, G, H complete). Tilted-photo modal (Edición B)
   and visual marker rectangle (Edición H) both shipped.
6. ✅ Perspective correction using marker's four corners (warpPerspective)
   + safe zone overlay (SAFE_ZONE_RATIO = 0.70) + "view original"
   comparison button. All shipped in this phase.
7. ✅ Safe zone overlay — folded into phase 6.
8. ⏸ Convert to PWA (next chat).
9. ⏸ Migrate to Azure Static Web Apps with Entra ID (later phase).

## Distribution strategy

- Primary: deploy as web app on GitHub Pages (free, works on any
  device with a modern browser).
- Offline / installable: convert to PWA (manifest + service worker)
  in the next dedicated chat.
- **Future corporate hosting**: migration to Azure Static Web Apps
  with Entra ID authentication is planned for the maturity phase,
  to provide real corporate authentication and align with the
  Microsoft ecosystem already in use at the organisation. The
  application code itself does not need to change; only the
  hosting target.
- Native APK / .exe: deferred unless a concrete reason a PWA can't
  cover emerges. Tools for that day: Capacitor (APK) or Tauri (.exe).

## Tech stack and constraints

- Vanilla HTML + CSS + JavaScript. No frameworks (React, Vue, etc.).
- Single-file structure for app logic (`index.html`). PWA adds two
  small files: `manifest.json` and `sw.js`.
- External dependencies bundled locally in `lib/`:
    - OpenCV.js: techstark build, version 4.12.0-release.1.
      Includes the objdetect/ArUco module. Bundled at
      `lib/opencv.js`. ~10 MB. Source verified: the official
      opencv.org build does NOT include ArUco; the techstark
      build does (confirmed by listing cv keys at runtime).
    - heic2any: version 0.0.4. Bundled at `lib/heic2any.min.js`.
      ~1 MB. Self-contained (WebAssembly embedded as base64).
- No npm, no build step, no transpiler.
- Must run identically on iOS Safari, Android Chrome, and desktop
  Chrome / Firefox / Edge.
- No backend. All processing client-side.

## Repository structure

    repo-root/
    ├── index.html              (main app file)
    ├── manifest.json           (PWA — to be created in phase 8)
    ├── sw.js                   (service worker — to be created in phase 8)
    ├── PROJECT_CONTEXT.md      (this file, also in Claude project context)
    └── lib/
        ├── opencv.js           (~9-10 MB, do not edit)
        └── heic2any.min.js     (~1 MB, do not edit)

## Data handling and privacy

- All image processing is performed client-side, in the user's browser.
- No images, measurements, or user data are transmitted to external
  servers at any point.
- No third-party AI, cloud vision, or analytics services are used.
- External dependencies (OpenCV.js, heic2any) are bundled within the
  repository and served locally. The application makes no external
  network requests after the initial page load.

## Measurement assumptions and physical limits

The measurement pipeline (ArUco scale calibration + perspective
correction) is geometric, not optical. It corrects how pixels are
arranged, not how they are lit or coloured. Honest documentation of
the assumptions underneath:

### What the system measures correctly
- Lengths and contours of features lying on the same plane as the
  ArUco marker.
- Damage on flat or slightly curved surfaces (fuselage panels, wing
  skin) where the local curvature around the marker and the damage
  is negligible (< 1° within the marker-to-damage radius).

### What the system does NOT measure
- Depth or relief of the damage. The pipeline is 2D over the surface
  plane; it has no notion of out-of-plane displacement.
- Features lying on a different plane from the marker (e.g. marker
  on one face of a corner, damage on the perpendicular face).
- Anything outside the photo. The marker must be in the same shot
  as the damage.

### Operational rules to keep error below the 2% target
- Place the ArUco marker as close as possible to the damage
  (ideally 5–15 cm), on the same surface, flat against it.
- Frame the photo so the marker is near the **optical centre** of
  the image (centre of the camera viewfinder), not at a corner. The
  marker acts as the ruler of the whole system; if it lies in a
  zone where the lens distorts, every measurement inherits that
  distortion.
- Use the phone's main camera (1x zoom). Do NOT use wide-angle
  (0.5x) or ultra-wide lenses: their geometric distortion (5–15%
  at the edges) is not corrected by this pipeline.
- Frame both the damage and the marker within the central 70% of
  the image (the "safe zone" overlay shows this boundary). Lens
  distortion grows toward the edges. Experimentally confirmed: error
  is ~0.3% at centre, rising to ~3% at image edges.
- Use the phone in whichever orientation (portrait / landscape) best
  fits the damage shape. No significant accuracy difference between
  the two orientations was found experimentally when framing is
  otherwise equivalent.
- Avoid auto-switching to macro mode at very close range; some
  phones change lens automatically without warning.
- Avoid extreme oblique angles. Perspective correction (phase 6)
  handles moderate tilt well, but extreme angles (> 45°) degrade
  both detection and correction.
- Use the marker ID appropriate for the damage size:
    - ID 0 (15 mm): only when the damage is so small that larger
      markers do not fit. This marker occupies very few pixels and
      is susceptible to detection noise (~3–5% error, variable
      between photos). Avoid if ID 1 fits.
    - ID 1 (50 mm): primary marker for most inspections. Best
      accuracy of the three in real-world tests (~0.3% at centre).
    - ID 2 (100 mm): for large damage where ID 1 is visually too
      small relative to the damage extent.

### What we deliberately do NOT do
- No automatic brightness, contrast, colour or filter adjustments
  are applied to the photo. The image stored in the final JPEG is
  geometrically rectified but otherwise unmodified, preserving
  traceability for professional documentation.
- No AI-based "enhancement" of the image. The pipeline is fully
  deterministic and defensible.
- For photos where lighting is so poor the damage is not visible,
  the correct response is to retake the photo with better light,
  not to post-process the existing one.

## Experimental findings (May 2026)

### Pre-phase-6 results (without perspective correction)

Real-world calibration testing with the three printed markers
(14.75 / 49.75 / 99.75 mm). Test conditions: handheld phone, main
camera at 1×, moderate inclination, marker side-variance below 1.10.

Measuring a 1 € coin (real diameter 23.25 mm) at different distances
from the marker, with each of the three markers:

| Marker | Coin near marker | Coin far from marker |
|--------|------------------|----------------------|
| ID 0   | -1.08 %          | +4.52 %              |
| ID 1   | -1.94 %          | +7.53 %              |
| ID 2   | -3.23 %          | +6.24 %              |

Measuring a credit card (real long side 85.60 mm) with the small
marker (ID 0):

| Position             | Error    |
|----------------------|----------|
| Card near marker     | +3.04 %  |
| Card far from marker | +4.56 %  |

Key finding: error scaled systematically with distance from marker.
Root cause: constant-scale assumption is invalid under any camera tilt.

### Post-phase-6 results (with perspective correction)

Setup: marker ID 1 (49.75 mm), objects centred in image central 70%,
main camera at 1×, handheld.

Credit card long side (real: 85.60 mm) — four photos, three
measurements each:

| Photo | Orientation | Distance | Mean    | Error   |
|-------|-------------|----------|---------|---------|
| 1     | Portrait    | Normal   | 85.83   | +0.27 % |
| 2     | Landscape   | Normal   | 85.80   | +0.23 % |
| 3     | Portrait    | Far      | 85.77   | +0.20 % |
| 4     | Landscape   | Far      | 85.97   | +0.43 % |

1 € coin (real: 23.25 mm) near marker, centred: 23.0–23.3 mm
(−1.1% to +0.2%). Note: measuring circular diameters introduces
user error of ±0.5 mm from estimating the diametral line; objects
with straight edges give more reliable results.

### Interpretation

1. **Error scales monotonically with the distance from the measured
   point to the marker (pre-phase-6).** This is not noise; it is a
   systematic geometric bias.

2. **Post-phase-6: the systematic bias has been eliminated.** Error
   is now ~0.3% at centre regardless of distance from marker.

3. **Root cause of residual error: radial lens distortion.** Error
   rises from ~0.3% at centre to ~3% at image edges. warpPerspective
   does not correct lens distortion; only perspective. Mitigated by
   the safe zone operational rule.

4. **The tilt warning (PERSPECTIVE_TILT_LIMIT = 1.10) is a weak
   signal for small markers.** A marker occupying 7-10% of the image
   width can show negligible side variance even when the camera is
   tilted enough to cause 5+% measurement error. Useful as a coarse
   filter but not sufficient on its own.

5. **Marker ID 0 is unreliable** due to limited pixel coverage.
   Error is 3–5% and variable between photos (not just biased but
   noisy). Use only when ID 1 does not fit the scene.

6. **Measurement repeatability is high**: three taps by the same
   user on the same photo vary by at most 0.4 mm. Residual error
   is systematic, not random.

### Implications for the roadmap

These findings justify prioritising the safe zone overlay (done, in
phase 6) and confirm the 2% accuracy target is achievable within the
operational rules. Per-device lens calibration (checkerboard) would
push best-case error below 0.5% but is deferred to future phases.

Expected error budget post phase 6, assuming the operational rules
are followed:
- Damage centred in the image (≤ 70% zone): ~0.3%.
- Damage extending to image edges (~90%): 2–3% from lens distortion.
- Severely tilted photos (> 45°): 2–3% from amplified pixel noise.
- Marker ID 0 in any condition: 3–5%, use with caution.

## Code conventions

- All code (variable names, function names, strings shown to the user)
  and comments in English.
- Comments explain *why*, not *what*. Avoid restating what the code
  obviously does.
- Function and variable names descriptive and consistent with
  existing style.
- Configuration block pattern: clearly marked sections near the top
  of the script for any value that might need tweaking (auth,
  marker sizes, perspective tolerance, safe zone ratio, etc.) with
  comments explaining how to change them without programming knowledge.
- All edits should be applied as small, reviewable changes (one
  conceptual change per step), not large rewrites.
- When a technical assertion depends on external library behaviour
  (especially OpenCV), mark it explicitly as an assumption and
  propose a small empirical verification before building code on it.
  Do not write "X does Y" with confidence if it has not been verified
  in this specific build and real photos.

## Success criteria

- A user can take a photo with an ArUco marker visible, open it in
  the tool, and measure a damage feature in under 15 seconds with no
  manual calibration.
- Measurement error under 2% when the marker is fully visible,
  flat, and well lit, and both marker and damage are within the
  central 70% of the image. Achieved post phase 6.
- Works offline once installed as a PWA (phase 8, pending).
- Total app size under 20 MB (OpenCV.js ~10 MB + heic2any ~1 MB +
  app code, with headroom for future additions).

## Working preferences (mandatory for AI assistants)

- All chat replies in Spanish, always.
- Code and code comments in English.
- I'm not a programmer. Assume no prior coding knowledge unless I
  explicitly say otherwise. Explain concepts briefly the first time
  they appear (e.g. async loading, Promises, WebAssembly memory
  management).
- Prefer fewer, well-explained incremental changes over large
  rewrites I cannot follow.
- When proposing code, explain the reasoning before pasting the code.
- Flag trade-offs and limitations honestly, even if I don't ask.
- Don't add libraries or complexity without justifying why simpler
  options won't work.
- Before any non-trivial edit: read the relevant fragment of
  index.html by asking for the specific lines or function needed.
  Do NOT rely on memory of the file from earlier in the chat —
  the file grows chat after chat and memory desyncs. Do NOT request
  the whole file unless strictly necessary.
- Give the user only what they need to copy-paste into their editor
  (specific fragments, new files), not whole-file replacements.
  Explain at each step what the fragment does and why, so the user
  understands what they are pasting.
- When proposing a library, download URL, or technical decision,
  verify before asserting. Prefer "let me check" over assuming.
- When a technical assertion depends on external library behaviour
  (especially OpenCV quirks, browser API differences), explicitly
  mark it as an assumption. If the cost of being wrong is high
  (e.g. would produce a wrong transform on real photos), propose
  a verification step BEFORE writing the code that depends on it.
- Ask before opening new sub-tasks or expanding scope. If something
  unexpected comes up mid-implementation, stop and confirm with me.
- If a user observation contradicts your reasoning, take it
  seriously and re-evaluate. Real-world data beats theoretical
  expectations. Do not defend a previous answer against real
  evidence; acknowledge the discrepancy and investigate.
- Before asserting facts about what the user can see in an image
  or screenshot they have shared, look carefully at the image.
  Do not describe image contents based on what "should" be there
  according to theory; describe what is actually visible.

## Future ideas (not in roadmap)

These are ideas captured to avoid losing them. They are **not
committed work** and will only enter the roadmap when an explicit
decision is taken. They are organised by ambition, from
"realistic incremental improvement" to "speculative". Items the
project owner has explicitly flagged as more interesting are
marked with ★.

### Workflow and traceability

- **★ Structured inspection session as a first-class entity.**
  Today each photo is independent. An "inspection" would become
  a container: aircraft tail number, date, inspector, list of
  documented damages, each with its photos and measurements. The
  session ends by generating a signed PDF technical report. This
  aligns naturally with the Microsoft 365 ecosystem already in use
  at the organisation.
- **Damage type catalogue integrated with the existing classifier.**
  Each measured damage would be classified (Dent, Blend-out, Rivet
  Pull-in, Out of Contour) using the logic of the separate
  classification tool already developed. The two tools would
  converge into a single deliverable: "measure + classify in one
  flow", removing the need for inspectors to use two apps.
- **Zone coding on the aircraft.** Before taking the photo, the
  inspector tags "panel L-23, frame 14-15". The system stores it
  and allows searching historical damages by zone.
- **Integration with corporate maintenance systems** (e.g. SAP).
  Saved damages push automatically into the existing ticketing or
  maintenance-planning system, removing manual data re-entry.

### Measurement core improvements

- **★ Automatic damage detection (without AI).** Instead of the
  inspector marking the two endpoints of a damage manually, classic
  computer vision (edge detection, contour analysis on the
  perspective-corrected image) could propose the measurement
  automatically. The inspector then confirms or adjusts. Works well
  for high-contrast defects (scratches, dents with shadow). No
  training data required.
- **Multi-marker support for curved surfaces.** Today a single
  marker rectifies a single plane. Placing one marker at each
  corner of a curved fuselage panel would allow piecewise
  reconstruction of the real geometry. Useful for large panels
  where a single plane does not hold.
- **Temporal comparison across inspections.** If the same zone is
  inspected every X weeks, the app could align successive photos
  (the marker provides the alignment anchor) and highlight new
  defects or growth of existing ones.
- **Light 3D via stereometry.** Two photos of the same damage from
  slightly different angles, both with markers, would allow
  estimating depth — the dimension the current pipeline does not
  measure. Would extend the tool from "surface extent" to
  "dent volume".
- **Per-device lens-distortion calibration.** If an inspector
  always uses the same phone, a one-time calibration of that
  phone's lens with a checkerboard pattern would remove the
  residual radial distortion that survives perspective correction.
  Would push best-case error below 0.5%.

### Capture and quality

- **★ Real-time capture assistant.** When the inspector opens the
  camera, the app overlays guidance: "marker too far from centre",
  "you are too tilted, straighten up", "too dark, turn on the
  light". Reduces drastically the number of bad photos that reach
  the measurement step. Requires moving part of the OpenCV
  pipeline to the live video stream rather than only the final
  still photo.
- **Oblique-lighting documentation.** Not a software item, but
  worth noting: a torch held at grazing angle reveals dents
  invisible under frontal light. Worth standardising as part of
  the inspection procedure.

### Speculative (only worth revisiting once the basic flow is mature)

- **Trained AI model on the organisation's own damage dataset.**
  With hundreds or thousands of pre-classified real damages, a
  custom model could classify automatically without human input.
  Only realistic once a labelled dataset exists; not something
  to start from scratch.
- **Augmented reality for location.** Move the phone over the
  aircraft and see historical damages of that zone overlaid.
  Could leverage LiDAR on modern iPhones. Visually impressive but
  the practical inspection ROI is debatable.
- **Automatic generation of technical drawings.** Convert the
  measured dimensions into the stylised drawing format used in
  official reports, with dimensions formatted per the
  organisation's standard.

### Suggested medium-term order (after phase 8)

If pursued, a reasonable order of attack — focused on highest
practical value before complexity:

1. Phase 9 (Azure + Entra ID) — once PWA is stable.
2. Structured inspection session + PDF report — turns the tool
   from "measurer" into "documentation system".
3. Damage type catalogue integration — merges this app with the
   existing classification tool. From two apps into one.
4. Real-time capture assistant.

Visual or AI-flavoured items are deliberately deferred until the
basic flow is polished and real users are asking for them.
**Solve the simple problem well before adding complexity.**

## How to start the next session (phase 8 — PWA)

When opening a new chat:

1. Confirm that the latest index.html and this PROJECT_CONTEXT.md
   are present in project files (uploaded by the user after the
   previous chat).
2. The assistant should read this PROJECT_CONTEXT.md first.
3. When code inspection is needed, ask for specific fragments
   (line ranges or function names) — do NOT ask for the whole file
   and do NOT rely on memory of earlier chat content.
4. The assistant should not start writing code until the plan has
   been approved in plain language.
5. Deliver changes as copy-pasteable fragments for VS Code, not as
   whole-file replacements. Explain each fragment before presenting
   it.

The phase 8 plan should answer these questions before any code:
- Which files to cache (index.html, lib/opencv.js ~10MB,
  lib/heic2any.min.js, manifest.json, icons): does opencv.js get
  cached on first install or on first use?
- Which cache strategy (cache-first, network-first,
  stale-while-revalidate)?
- Do icons need to be created, and is there a reasonable shortcut?
- Any GitHub Pages restrictions that affect the service worker
  (scope, HTTPS, paths)?
