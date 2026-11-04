# Android Notes — Codex implementation plan

## Purpose and working rules

Build a small native Android application for handwritten notes and freehand drawings on a Samsung tablet with S Pen. The development machine runs Gentoo Linux; VS Code and command-line tools must be sufficient.

This document is the implementation brief. Read it and any applicable AGENTS.md before changing code. The repository initially contains only LICENSE; preserve that license.

Implement one stage at a time. After each stage:
1. Run the relevant automated checks and build a debug APK.
2. Give Matthias exact build/install instructions and a short tablet test checklist.
3. Report what works, what could not be tested, and any decisions needed.
4. Stop until Matthias has tested and explicitly accepted the stage. Fix problems within that stage before advancing.

Do not implement all stages in a single pass. Do not claim device testing without access to the device. Keep each stage reviewable in Git; commit/tag accepted milestones when authorized by the active workflow. Do not push, merge, publish, or change repository settings merely because this plan describes development.

Confirmed requirements below take precedence over proposed defaults. Defaults are implementation suggestions, not previously approved product decisions. Resolve device compatibility before selecting the build toolchain. Ask about genuinely blocking choices at the relevant stage, not as a large questionnaire before starting.

## Confirmed scope

- Handwriting and freehand drawing only.
- Vertical scrolling through a note.
- A right-side handle that moves all content below a selected horizontal gap up or down.
- Upward movement preserves a minimum gap and never causes a collision.
- A temporary horizontal writing baseline derived from the first stroke on a line.
- Pen color and thickness controls; 27 colors formed from three intensities per RGB channel.
- One daily note by default, with additional notes created explicitly and distinguished by hour and minute.
- Open existing notes and share a note through Android's standard sharing mechanism.
- Local operation without a server, account, or network dependency.

The menu has exactly these seven actions:

| Action | Behavior |
| --- | --- |
| New note | Explicitly create an additional note for today, subject to the minute uniqueness rule. |
| Open note | Choose and open an existing note. |
| Share note | Invoke the Android share sheet for an exported note. |
| Move content below | Enter the gap selection and drag-handle mode. |
| Color | Choose one of the 27 pen colors. |
| Thickness | Choose the width of subsequent strokes. |
| Baseline | Enable or disable the temporary writing guide. |

Do not add Undo or Redo. Do not add text editing, shape tools, imports, selection editing, recognition, cloud sync, or a desktop backend to the initial implementation. Handwriting recognition on Matthias's own computer is a possible later project; retain useful stroke data now without building that feature.

## Platform and technical approach

Use Kotlin and a traditional Android custom View with Canvas rendering. Keep the app small; avoid introducing a UI framework or elaborate dependency injection system for this scope.

An earlier target was described as a Samsung Galaxy Tab A6 with S Pen. Verify the actual model and installed Android API level with Matthias or an attached device before fixing minSdk. Do not assume a modern Android version or require upgrading the tablet.

Use the Gradle wrapper and a compatible, pinned Android Gradle Plugin/Kotlin/JDK combination. Select versions against official documentation when implementing; do not insert guessed version numbers. Document Android SDK installation, required SDK packages, JAVA_HOME/ANDROID_HOME setup as applicable, and the first build on Gentoo. Keep local SDK paths out of Git.

Suggested separation:
- Activity and small dialogs: navigation and the seven menu actions.
- NoteCanvasView: input, coordinate conversion, drawing, and scrolling.
- Note/stroke model: serializable data, independent of Android UI classes.
- NoteRepository: loading, listing, atomic saves, and note identity.
- GapMovement: pure geometry and clamping.
- BaselineController: transient guide state.
- NoteExporter: shareable rendering/serialization.

Adapt filenames as needed. Keep geometry and storage logic testable without a tablet.

## Stroke model and coordinate system

Store vector strokes rather than only a bitmap. Each note needs a stable ID, schema version, creation date/time, modification time, and ordered strokes. Each stroke needs:
- A stable stroke ID and explicit stroke boundaries/order.
- The color and thickness captured when that stroke began.
- Ordered x/y samples, relative capture times, and pressure where the device supplies it.
- Enough timestamp information to reconstruct stroke order and timing.
- Position information independent of viewport size, scroll offset, and screen rotation.

Capture historical MotionEvent samples as well as the latest sample. Handle pen down, move, up, and cancellation explicitly. Preserve original sample coordinates/timing for possible future recognition; represent later content movement with a stroke placement offset or equivalent transform rather than destroying the original samples.

Use one documented logical document coordinate system. Convert screen coordinates through the current scroll/scale transform exactly once. Rendering, hit testing, gap detection, exporting, and persistence must agree on that system. Do not store viewport pixels as note coordinates. Pen width and minimum-gap units must be documented in the same coordinate model.

Capture pressure even if the initial renderer uses constant width. Pressure-dependent width is not required. Existing strokes must not change when the selected color/thickness changes.

Suggested input default, pending tablet testing: stylus draws; finger gestures scroll. Ignore palm/finger marks while the pen is active and keep handle interaction distinct from drawing. A mode or gesture must never unexpectedly reinterpret pen marks as scrolling. Test actual S Pen tool types, cancellation, and palm behavior on the target tablet.

## Persistence and note identity

Proposed initial format: one versioned JSON file per note in app-private storage, with a lightweight index or directory scan for the note list. JSON is a practical default, not a requirement to introduce a database. Previous discussion of SVG/SQLite/sync does not authorize a backend implementation.

- Open or create today's default note on launch.
- Use the device's local calendar date, not UTC, for daily naming.
- Suggested display/storage names: YYYY-MM-DD for the default note and YYYY-MM-DD_HH-mm for explicitly created notes.
- Do not create two notes for the same minute. Never overwrite a note on a duplicate New note request; show/open the existing note for that minute.
- Use a stable internal ID separate from the displayed name.
- At midnight, do not silently move strokes out of an already open note. Switch to today's note on the next launch or an explicit note navigation action; document this proposed default.
- Save after completed strokes and completed layout edits; flush pending changes when navigating away or entering the background.
- Perform disk I/O off the input/rendering path. Serialize saves so an older snapshot cannot overwrite a newer one.
- Use temporary-file plus atomic replacement where supported. Keep a failed save visible and recoverable rather than silently reporting success.
- Reject unsupported schema versions clearly; do not replace unreadable/corrupt files with empty notes.
- App-private notes must survive app updates. Explain that uninstalling the app may remove them.

Viewport position, current pen settings, and baseline-enabled preference can be stored separately. The visible baseline itself is transient and must never be serialized as note content.

## Vertical scrolling

Provide an effectively growing vertical document, with a finite visible viewport and room below the last stroke to continue writing. Avoid allocating a bitmap as tall as the whole document.

- Scroll offset is nonnegative; all strokes keep their document positions.
- Cull rendering to visible strokes as note size grows.
- Keep horizontal behavior simple; horizontal scrolling and zoom are outside initial scope.
- Preserve the active note and document geometry across backgrounding and rotation.
- Test writing after scrolling, reopening a long note, and switching between long and short notes.

## Move content below

The user chooses an actual empty horizontal gap between strokes. Display one handle on the right side at that boundary. Dragging it translates every stroke below the gap by the same vertical displacement. Content above the gap stays fixed; no individual stroke is split or deformed.

Algorithm requirements:
1. Compute each stroke's visible vertical bounds, including its placement transform and half its rendered width.
2. Identify a horizontal band that intersects no stroke anywhere across the document width; do not treat whitespace in only one column as a valid full-width gap.
3. Choose a cut inside that band and freeze upper/lower membership for the entire drag.
4. Apply the same proposed displacement to every lower stroke.
5. Clamp upward movement so the lower group's top remains at least the minimum gap below the upper group's bottom. Preserve nonnegative document coordinates.
6. Update preview geometry during dragging; persist the completed change once. On cancellation, restore the pre-drag placement.
7. Recompute document extent and clamp viewport position after completion.

Allow downward movement to insert space. Permit upward movement only as far as the remaining gap allows. Do not move content automatically when no valid gap exists. Moving below a boundary with no lower content has no effect.

Proposed interaction: after selecting Move content below, tap near an empty horizontal band to choose it, then drag its right-side handle. Start with one active boundary. Exact tap tolerance, handle size, minimum-gap value, and whether the mode exits after a completed drag remain to be tuned with Matthias. Keep those constants centralized and expressed in documented logical units. Do not silently choose an arbitrary cut through strokes.

## Temporary baseline — agreed option A

Derive a faint horizontal guide from the first completed stroke on the current writing line. Once established, keep its vertical position fixed for that line. It should be slightly darker than the paper while staying unobtrusive.

- It is a drawing aid, never a stored stroke.
- It is absent from saved note data and all exports.
- It disappears when starting another line, switching tools/modes, or after inactivity.
- When a new line starts, establish a new guide from that line's first completed stroke.
- The Baseline menu action enables/disables this behavior; disabling removes the active guide immediately.

The exact baseline estimator, new-line threshold, and inactivity timeout are unresolved. Proposed first experiment: use the first stroke's lower bound as the guide estimate and a documented vertical-distance threshold for recognizing another line. This heuristic can be wrong for ascenders/descenders; keep it isolated and tune it during tablet testing. Do not continually move the guide as more strokes are added to the same line.

## Color, thickness, and sharing

Provide all 27 RGB combinations with a clear selected-color indication. Three intensity values and a hexagonal palette layout were discussed but not finalized. Proposed starting values are 0, 128, and 255; present a compact palette for tablet testing. Include a visible swatch outline so white is selectable on white paper.

Keep a small set of clearly previewed thickness values or a simple width control. Exact values are to be tuned on the tablet. Settings apply to new strokes only.

Use Android's share sheet with content URIs, temporary read grants, an appropriate MIME type, and an Android FileProvider or equivalent supported mechanism. Do not expose private filesystem paths or request broad storage permission for sharing.

Proposed first export: SVG for the complete note, keeping the writing as vectors. Offer the native JSON as a preservation format if needed. Confirm whether PNG or PDF is needed before adding either. Export the complete document rather than just the viewport, include layout movements, and exclude the handle and baseline. Keep very tall notes bounded in memory; define sensible nonzero bounds for empty exports.

## Implementation stages and acceptance gates

### Stage 1 — Buildable shell and pen drawing

Create the Gradle project, minimal Activity, and custom drawing View. Support black constant-width pen strokes in memory; capture the full stroke model from the start. Document setup and debug build/install commands.

Acceptance: debug build succeeds; the app launches on the target tablet; continuous and quick pen strokes work; finger/palm contact does not create unwanted marks; cancellation does not leave a stuck active stroke. Nothing from later stages is required yet.

Stop for Matthias's test and acceptance.

### Stage 2 — Save and reopen

Implement versioned local storage, today's default note, and reliable autosave. Restore notes after process restart. Keep persistence separate from rendering.

Acceptance: a completed stroke survives force-stop/relaunch; multiple strokes retain order/appearance; corrupt data is reported without being overwritten; interrupted saves leave a valid prior file. Add focused serialization and save-order tests.

Stop for Matthias's test and acceptance.

### Stage 3 — Vertical scrolling

Add finger scrolling and growing document extent without changing stored geometry.

Acceptance: write at several scroll positions; return to earlier strokes; reopen the note; verify there are no coordinate jumps or lost marks. Check a reasonably long note on the tablet for responsiveness.

Stop for Matthias's test and acceptance.

### Stage 4 — Move content below

Implement gap selection, one right-side handle, preview, clamping, cancellation, and persistence of stroke placement transforms.

Acceptance: downward dragging inserts space; upward dragging stops at the minimum gap; upper strokes stay fixed; all lower strokes move equally; no stroke is split; cancel restores positions; reopen preserves the move. Unit-test gap finding and clamping, including wide strokes, overlapping vertical ranges, empty notes, and no lower group.

Stop for Matthias's test and acceptance.

### Stage 5 — Temporary baseline

Implement option A and the Baseline toggle. Tune the estimator, line-change threshold, and inactivity timeout with real handwriting.

Acceptance: first-stroke guide appears and remains fixed for the line; next line establishes a new guide; tool switch/inactivity clears it; disabling works; it never appears in saved content.

Stop for Matthias's test and acceptance.

### Stage 6 — Color and thickness

Implement the 27-color chooser and width control, with persisted current settings.

Acceptance: every color is selectable, including white; changing settings affects only subsequent strokes; styles survive reopening; palette and controls are usable with the S Pen.

Stop for Matthias's test and acceptance.

### Stage 7 — Note navigation and sharing

Complete the seven-action menu, additional minute-indexed notes, note list, and Android sharing. The note list can use a simple newest-first dialog; no extra main menu actions are needed.

Acceptance: launch opens today's note; explicit notes use hour/minute names; a repeated request in one minute never overwrites data; opening another note saves the current one; the share sheet opens; a receiving application reads the complete export with correct positions/colors/widths and no editing guides.

Stop for Matthias's test and acceptance.

### Stage 8 — Reliability and handover

Address issues found during real use, rotation/background lifecycle, large notes, low-space save errors, and accidental palm input. Keep optimization tied to observed problems.

Acceptance: run the accumulated automated checks and repeat the tablet checklist for writing, scrolling, gap movement, baseline, save/reopen, navigation, and sharing. Document remaining limitations and how to build/install an update without uninstalling.

Stop for final user acceptance. Do not begin recognition or synchronization automatically.

## Validation and implementation reporting

Maintain a short stage/status section in the repository as work proceeds: current stage, completed checks, tablet feedback, known issues, and the next acceptance gate. Update this plan only when requirements or explicit decisions change; do not mark a stage accepted on the user's behalf.

Use focused JVM tests for geometry, serialization, identity/date naming (with an injected clock), and save ordering. Add Android tests only where they verify platform behavior that cannot be checked meaningfully in pure code. Do not create tests merely to mirror trivial getters or UI labels.

Each stage's handover must include the actual Gradle commands used and their results, plus:
- APK location and installation command.
- A short manual test sequence and expected results.
- Any untested platform behavior.
- The one or two decisions that stage needs from Matthias.

Document USB ADB as the baseline deployment path. Wi-Fi ADB is desired; inspect the tablet's capabilities and document the supported route rather than assuming modern wireless pairing. Do not require Android Studio for routine building, installing, or debugging.

## Decisions to settle when they become relevant

| Decision | When to resolve |
| --- | --- |
| Actual tablet model/API level and compatible toolchain | Before Stage 1 build configuration |
| Pen/finger/palm interaction details | Stage 1 tablet test |
| JSON schema and duplicate-minute behavior | Stage 2; exercise additional-note UI in Stage 7 |
| Minimum gap, handle interaction, and touch tolerances | Stage 4 |
| Baseline estimator, line threshold, and timeout | Stage 5 |
| RGB intensities, palette geometry, and width values | Stage 6 |
| Export format and date/time note list presentation | Stage 7 |

Backend synchronization and handwriting recognition remain deferred. Preserve vector timing/pressure data so future work does not require replacing the note model.
