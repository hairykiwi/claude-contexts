# CC-TASKS — QElectroTech contributions (hairy_kiwi)

Working task log for QET dev. Branch context noted per item (most work on
`folio-mirror-flip` off `qt6_cmake_joshua`). Newest/active items at top;
resolved items archived at the bottom with their commit; feature ideas in a
parking lot below that.

Item template: **Status** · **Area** · **Location (file:line)** · finding ·
open question / next action. Status ∈ OPEN / DEFERRED / NEEDS-TEST / RESOLVED.

Git holds the authoritative *what/when* (commits, diffs, blame). This log holds
what git holds poorly: the diagnostic narrative and method, for fast human
orientation and future-fix insight. Don't duplicate diffs here; do keep reasoning.

---

## ACTIVE

### Remove MVR-OFF readability forcing — free rotation is the new default  · COMMITTED (207d3cc5b, pushed) · branch mirror-flip-rotate
**What changed (sources/qetgraphicsitem/element.cpp):** `correctReadability` no longer
re-orients text when keep_visual_rotation is OFF. The OFF-branch 180° inverted→readable actuator
and the classification it depended on (`parity = m_mirror^m_flip`, the snapped `net`, and
`inverted`) are deleted; only `net_raw` and the keep_visual_rotation-ON tops-up de-rotation
remain. Result: with the switch OFF, text rotates to exactly the angle the user set, correctly
pivoted (rotationpivot.h / `centerPivotEndPos`, unchanged) and positioned under mirror/flip
(`compensateMirrorFlip`, unchanged) — no automatic re-orientation, because the code that did it no
longer exists. The stale `applyMirrorFlip` ordering comment ("compensate reads the corrected
rotation/pos — no stale-transform window") was rewritten to describe the actual two-step sequence;
the claim was disproven empirically (case "5volt": identical placement whether compensate read the
corrected or the raw rotation — see the scoping entry below). **keep_visual_rotation-ON is fully
unchanged** — explicit tops-up forcing still de-rotates by -net_raw about the item's own
bounding-rect center.
**Split confirmed clean before editing (measured, not assumed):** `net_raw` is the ONLY variable
the ON branch uses; the snapped `net` and `parity` were exclusively OFF-branch, so removing the
forcing orphaned them — removed too, no dead code (the task premise "MVR-ON still needs net" was
imprecise: it needs net_raw, never the snapped net or parity). `rotateAboutOwnCenter` is reachable
only from inside `correctReadability`, so on plain OFF load it now never fires ⇒ the silent
orientation rewrite of authored OFF text disappears at the root. Legacy keep_visual_rotation-ON /
`<input>`-converted text still rewrites once on load by design (ON branch stays live) — known and
accepted, not a regression to chase.

**24-cell R/I classifier table — RETIRED, archived not re-baselined.** The 4×3 observations table
(`{0,90,180,270}` × `{plain, M↮F, M&F}`, identical grouped/ungrouped — it is the "Observations
table" in the "Rotated-element text readability — LAYER 2" entry below) validated the `inverted`
classifier that this pass deletes. It is NOT re-baselined to a new default; it stays in place below
(and in the gate-α/β/γ history, commit 6f8dd772c) as the reference R/I pattern for whenever
ISO-layout (the future opt-in, FEATURE IDEA #7) is actually scoped. Do not treat it as a current
default-behaviour test — under free-rotation default there is no forced orientation to assert.

**Replacement test (keep_visual_rotation-ON net-based de-rotation still works):** with a text's
switch ON, for element states {plain, mirror, flip, mirror+flip} × own/element rotation summing to
{90,180,270}, confirm the text is driven tops-up (net→0) about its own bounding-rect center,
grouped and ungrouped, live and after save/reload. This replaces the retired 24-cell table as the
only orientation-forcing test still meaningful after this pass.

**Test note (silent-rewrite check):** "byte-identical" in that check means the SPECIFIC text
`rotation`/`x`/`y` values are unchanged across load+save — NOT the whole .qet file. Unrelated
parts (e.g. a timestamp, ordering, formatting) may differ for normal reasons; don't read a
whole-file diff as a regression. Compare only the affected text fields.

**Test result (2026-06-21):** Tests 1–4 (free-rotation OFF across plain/mirror/flip/mirror+flip
× grouped/ungrouped; keep_visual_rotation-ON tops-up regression; Phenomenon B + trigger-lag
regression; silent-rewrite field-level check) all PASS against the 12:07 build.

### element.cpp CCDUMP instrumentation — REVERTED (2026-06-21, tree clean)
**Resolved:** the temporary `CCDUMP` instrumentation described below was reverted
(`git checkout -- sources/qetgraphicsitem/element.cpp`) and the binary rebuilt clean before the
"Remove MVR-OFF readability forcing" implementation above. The tree no longer carries it; the
binary no longer includes it. The captured logs and test inputs remain in `debug-logs/`
(gitignored) — `prefix_mirrorflip_test.qet`, `corrected_saved_test.qet`, `ccdump.log`,
`ccdump2.log`, `ccdump-active.log`, `ccdump-gated.log` — kept as evidence for the scoping
verdict and useful again whenever ISO-layout is scoped. Historical detail of what the
instrumentation did is preserved below for that future reference.

<details><summary>Historical: what the CCDUMP instrumentation was (now reverted)</summary>

**As of 2026-06-20, HEAD 0a734cdc5:** `sources/qetgraphicsitem/element.cpp` carries
an uncommitted temporary `CCDUMP` instrumentation block inside `Element::fromXml`
(a `{ ... }` block replacing the bare `applyMirrorFlip();` call at ~line 848). It
logs, per text, the stored-XML rotation/pos (BEFORE applyMirrorFlip), the live
displayed values (AFTER), and what `toXml` would serialize (WOULD-SAVE) — written to
stderr with the `[CCDUMP]` prefix. **Deliberately LEFT IN** to support the
silent-rewrite / ISO-forcing-reframe investigation below. **Must NEVER be staged or
committed.** Revert with `git checkout -- sources/qetgraphicsitem/element.cpp` once
the investigation is resolved (then `make -j8` to rebuild clean). The currently-built
`build_qt6/qelectrotech` binary INCLUDES this instrumentation. Test inputs +
captured logs live in `debug-logs/` (gitignored): `prefix_mirrorflip_test.qet`
(pre-fix authored values), `corrected_saved_test.qet` (post-correction values),
`ccdump.log` (run 1 = pre-fix load), `ccdump2.log` (run 2 = corrected reload).
(We lost a CC connection mid-session once; this note exists so a fresh session learns
the tree state from the file, not from memory.)
**UPDATE 2026-06-21 (HEAD still 0a734cdc5):** the in-tree instrumentation was EXTENDED
for the compensate-independence test (below) and the binary rebuilt (mtime 21 Jun 09:21):
(a) the `dumpTexts` lambda now also logs `sceneRect` (`sceneBoundingRect()`) +
`sceneCenter` (`mapToScene(boundingRect().center())`); (b) `Element::correctReadability`
now has a temporary env-gate at its top — `QET_GATE_READABILITY=1` makes it a pure no-op
(free rotation) and logs `[CCDUMP] correctReadability GATED (env) — no-op`; unset = shipping
behaviour. BOTH additions are part of the same do-not-commit instrumentation; the SAME
`git checkout -- sources/qetgraphicsitem/element.cpp` reverts everything. New captured
logs: `ccdump-active.log` (correction ON), `ccdump-gated.log` (`QET_GATE_READABILITY=1`).

</details>

### Silent rewrite of authored text orientation/position on open+save  · INVESTIGATION COMPLETE (empirical)
**Question answered (2026-06-20, measured via the CCDUMP harness above, not inferred):**
Opening a pre-fix .qet with a mirrored/flipped/rotated element carrying template text,
then saving, **silently rewrites the stored text `rotation` and `x`/`y`** whenever the
readability correction is non-trivial — with NO prompt and NO modified-flag indication.
- **Mechanism:** `fromXml` calls `applyMirrorFlip()` UNCONDITIONALLY (element.cpp:848,
  even mirror=flip=false) → `correctReadability` → `rotateAboutOwnCenter` (setRotation+
  setPos). `DynamicElementTextItem::toXml` serializes LIVE pos/rotation (dynamicelement
  textitem.cpp:93-95), so the correction lands on disk. `compensateMirrorFlip` sets only
  a `setTransform` (NOT serialized) → never contributes to the rewrite.
- **No indication:** `m_modified` defaults false (qetproject.h:255); the load path never
  calls `setModified(true)` (those are user-action-bound). So no asterisk, no save-on-close
  prompt — in-memory geometry diverges from disk silently; written the instant the user
  saves for any reason.
- **Measured cases (run 1 → ccdump.log), stored → after-load/would-save:**
  - "12" mirror, orient 0, MVR-on: rot0,(-14,13.5) → UNCHANGED (net 0 = no-op).
  - "10K" mirror, orient 1, MVR-on: rot0,(-36.5,-4.5) → rot270,(-34.1875,14.8125). REWRITE.
  - "5volt" flip, orient 2, MVR-off: rot0,(6,-27.5) → rot180,(32.7031,-10.5). REWRITE.
- **One-time, not cumulative (run 2 → ccdump2.log):** reloading the corrected file is a
  fixed point — BEFORE==AFTER==would-save for all three. The correction is idempotent, so
  the rewrite is a one-time migration on first open of a pre-fix file, not ongoing churn.

### Reframe scoping: is ISO orientation-forcing separable from the geometric bug fixes?  · SCOPING COMPLETE (no implementation)
**Hypothesis under test:** `correctReadability`'s I→R 180°-flip is a separable POLICY layer
bolted onto two clean GEOMETRIC fixes (rotationpivot.h centre-pivot; compensateMirrorFlip
position-under-reflection). If so, OFF-MVR could default to free rotation ("rotate where the user
put it, correctly pivoted + positioned under mirror/flip, no auto re-orientation"), with
ISO folded back as an explicit per-text opt-in (ISO-layout) rather than forced.

**Findings (traced live against the instrumented tree; line refs are clean-tree, the
in-tree CCDUMP block shifts them +~28):**
- **Q1 — policy vs geometry split is CLEAN at the code boundary.** `correctReadability`
  (element.cpp ~1185-1202) is 100% policy, no geometric-necessity code. Its body = the
  gate-1 classifier (`parity = m_mirror^m_flip`, `net` = snapped rotation sum) + TWO
  single-statement actuators: MVR-ON `rotateAboutOwnCenter(item,-net_raw)` (force tops-up)
  and MVR-OFF `if(inverted) rotateAboutOwnCenter(item,180)` (the I→R flip). Each is gateable
  behind one condition. The geometric fixes live in SEPARATE functions —
  `centerPivotEndPos` (rotationpivot.h, the user-rotate centre-pivot) and
  `compensateMirrorFlip` (reflection position) — invoked as their own steps. The user-rotate
  commands already apply geometry (centerPivotEndPos) and policy (correctTextReadability
  finalize) as two distinct stages (rotateselectioncommand.cpp:71/84 + :152; rotatetexts
  command.cpp:84/91 + :175).
- **Q2 — removing forcing DOES kill the orientation rewrite at the root (confirmed, not
  faith).** `rotateAboutOwnCenter` is reachable ONLY from inside `correctReadability` (grep:
  two call sites, both there). So free rotation (both branches gated) ⇒ correctReadability is a
  no-op ⇒ rotateAboutOwnCenter never fires on plain load ⇒ no serialized rewrite. CAVEAT: the
  MVR-ON tops-up de-rotate is ALSO a forcing and ALSO rewrites (case "10K" proved it for ON
  text). So "remove the forcing" must be precise: free rotation must be the DEFAULT (flip default
  ON→OFF) for the common case to stop rewriting; explicit MVR-ON text keeps de-rotating by
  design.
- **Q3 — compensateMirrorFlip still must run on load, and is already rewrite-free.** It is the
  geometric reflection-position fix; mirrored/flipped text is mis-placed without it,
  independent of orientation mode. It calls ONLY `setTransform` (element.cpp:1109/1152) — not
  setPos/setRotation — and the transform is NOT serialized, so it NEVER causes a rewrite
  (confirmed by case "12": compensate fired, zero serialized change). ⇒ The ENTIRE silent
  rewrite is from rotateAboutOwnCenter (policy), none from the geometry.
- **Q4 — downstream entanglement is at the VALIDATION/semantics level, not the code.** Gating
  the forcing redefines "correct default behaviour" so these need re-baselining (not
  re-coding): (a) the gate-1 classifier still runs but its `inverted` output goes unused
  unless an ISO-layout opt-in consumes it — becomes dormant; (b) the trigger-lag fix (0a734cdc5)
  is HALF still needed — its orientation re-fire becomes a no-op for free rotation, but its
  compensateMirrorFlip re-fire on own-rotation under mirror/flip (incl. the undo-recompute
  reasoning) STILL applies; keep correctTextReadability, just its correctReadability sub-call
  no-ops; (c) the 24-cell R/I table currently asserts default = all-R (ISO); under free rotation it
  must be re-baselined to the raw layer-1 (843ba6898) R/I pattern — the table as a classifier
  test stays valid, as a displayed-orientation test it changes meaning per orientation mode.

**Verdict:** orientation-forcing is cleanly GATEABLE at the code level (isolated single
statements + a solely-owned actuator); the geometric fixes are already separate and stay.
The orientation-half of the silent rewrite disappears at the root for free-rotation-default text.
What remains of the modified-flag fix: ONLY the legacy explicit-MVR-ON migration case, which
run 2 showed is one-time/idempotent — so it shrinks to at most "don't silently rewrite legacy
ON text on first load," possibly moot entirely if the default flips to OFF.

**The "should-be-no-op, wasn't" risk — NOW EMPIRICALLY CONFIRMED CLEAR (2026-06-21).**
Tested exactly as planned: env-gated `correctReadability` to a no-op (`QET_GATE_READABILITY=1`),
loaded `prefix_mirrorflip_test.qet`, logged each text's `sceneBoundingRect` + scene center,
active vs gated (`ccdump-active.log` / `ccdump-gated.log`). Discriminator: a 180°-about-own-center
rotation maps an axis-aligned rect onto itself, so if compensate is placement-correct at BOTH the
raw and corrected rotation, the on-screen footprint must be identical between the two runs.
- **Case "5volt" (flip, MVR-off, inverted — the free-rotation default case):** active `sceneRect`
  `(507.297,72.5 26.70×17)` rot180 pos(32.7,−10.5); gated `sceneRect` **identical**
  `(507.297,72.5 26.70×17)` rot0 pos(6,−27.5); scene center identical (520.648,81). Same region,
  only the glyph orientation flips. ⇒ `compensateMirrorFlip` reads the UN-re-oriented rotation and
  STILL places the text correctly. The applyMirrorFlip ordering comment is a convenience, not a
  correctness dependency. **No hidden coupling.**
- **Case "12" (net 0):** footprint byte-identical both runs (both no-op) — sanity anchor.
- **Case "10K" (mirror, MVR-on):** footprint differs (tall↔wide, the −net de-rotation) but scene
  center identical — MVR-ON forcing acting as designed, not the free-rotation path.
- **Bonus:** in the gated run WOULD-SAVE == stored XML for all three (rotation/x/y unchanged) ⇒
  gating `correctReadability` ALSO eliminates the silent rewrite at the root, confirming Q2: the
  ENTIRE orientation rewrite is from correctReadability/rotateAboutOwnCenter, none from geometry.
**Decision input resolved: the reframe is geometrically sound — GO.** IMPLEMENTED 2026-06-21
(green light given) — see the "Remove MVR-OFF readability forcing — free rotation is the new
default" entry at the top of ACTIVE for the change actually made.

### Rotated-element text readability — LAYER 2 (de-rotation + layout)  · RESOLVED — [WIP] on 6f8dd772c CLOSED (Phenomenon B + correction-trigger lag both fixed; OFF-default orientation-forcing since removed)
**Status (Jun 2026):** classifier gate-1 PASSED (logical-state parity/theta reproduces the 24-cell table); orientation correction built and gates α (layer-1 regression clean), β (no rotation drift; OFF confirmed forward-acting — orientation PASS), γ (24-cell orientation+position+save/reload) all PASS. Position fix for the grouped+flip ~group-height displacement committed separately as b0a405329. Orientation correction committed as 6f8dd772c, flagged [WIP] at the time pending Phenomenon B (below).
**UPDATE (Jun 2026, later same week):** Phenomenon B is now FIXED — see RESOLVED entry below (bd61ca17c, pivot fix on the "Rotate 90°" quick-rotate path; the dialog path, `RotateTextsCommand`, was already correct). A nomenclature-normalization follow-up also landed as feeef3cea (strips internal layer/MVR/gate shorthand from element.cpp/.h comments + identifiers — `mvr`→`keep_visual_rotation`, `rotateAboutOwnCentre`→`rotateAboutOwnCenter`, `bbox`→`bounding rect`; no behavioural change). [WIP] on 6f8dd772c is NOT closed: a separate, newly-identified correction-trigger lag remains open (own-rotation-change doesn't re-fire `correctReadability` until the next element-level rotate/mirror/flip) — see new ACTIVE entry below. The tempered commit message on 6f8dd772c (no unqualified "ISO-conformant by default" claim) remains accurate; the trigger lag is now the documented residual caveat in place of Phenomenon B.
**UPDATE (2026-06-21): [WIP] on 6f8dd772c CLOSED.** Both items that held it open are resolved:
Phenomenon B (bd61ca17c) and the correction-trigger lag (extracted as 023070fd8, fixed as
0a734cdc5 — see its RESOLVED entry below). Nothing about this layer remains in flight.
Separately, the keep-visual-rotation-OFF orientation-forcing that 6f8dd772c introduced has since
been REMOVED (207d3cc5b — free rotation is now the default; see the "Remove MVR-OFF readability
forcing" entry at the top of ACTIVE). So the "ISO-conformant by default" behaviour this commit
shipped no longer applies to OFF text — but that is a deliberate design reversal, not unfinished
work from this commit. NOTE: 6f8dd772c's literal commit SUBJECT still carries the "[WIP]" tag in
git history; that is left as-is (an additive correction is not worth rewriting pushed history) —
the [WIP] is closed here at the task-tracking level, which is what this log governs.
**Area:** element text rotation/readability · **Branch:** mirror-flip-rotate (integration; off qt6_cmake_joshua, clean-merges folio-mirror-flip + cherry-picked layer-1 843ba6898)
**Depends on:** layer-1 (DONE, 843ba6898 — text rotates rigidly with element; that's the substrate this corrects). See memory note for the standards detail.
**Related (separate, not a dependency):** shares the ungroup/`removeFromGroup` path with the "Genuine ungroup of a mirrored element displaces text" entry. That one is a mirror-geometry positional bug (persists to disk); this is rotation-orientation display logic. Don't merge them — but whoever codes second must not regress the first in the shared function.

**Goal (SUPERSEDED 2026-06-21 re: default):** originally "make rotated-element text
ISO-conformant by default, with an opt-in force-horizontal override". The default was
subsequently REVERSED — free rotation is now the default and ISO-layout becomes the future
opt-in. See the "Remove MVR-OFF readability forcing" entry at the top of ACTIVE. The geometry
and the keep_visual_rotation-ON (force-horizontal) path described in this entry remain accurate;
only the OFF-default ISO-forcing was removed.

**Observations table (confirmed empirically; R = readable/ISO-correct, I = inverted/wrong; identical for grouped & ungrouped):**
| TER  | plain | M↮F | M&F |
|------|-------|-----|-----|
| 0°   | R     | R   | R   |
| 90°  | I     | R   | I   |
| 180° | I     | I   | I   |
| 270° | R     | I   | R   |
- M&F (mirror AND flip) == 180° rotation (its column matches plain). M↮F (single
  reflection, mirror XOR flip) has the offset pattern. So readability must be
  computed from each text's NET scene transform, not from rotation alone.
- Confirmed: R@270° = tops-left (ISO-correct vertical, read-from-right); I@90° = inverted.

**MVR switch ("Maintain Visual Rotation", per-text):**
- ON (opt-in): force text to tops-up horizontal / folio orientation regardless of
  TER (de-rotate by −TER). The ASME unidirectional "read from bottom" allowance.
- OFF (DEFAULT): ISO-conformant — for any text whose net orientation is I, apply a
  180° correction; leave R cases genuinely untouched (true no-op, not "rotate 0").
  Result: always read-from-bottom or read-from-right, never inverted.
- Layer-1 (rigid rotation) is the substrate that produces the R/I pattern; OFF then
  corrects the I's. So default shipping behaviour after layer-2 = ISO-conformant.

**Grouping-dependent pivot (this is what makes LAYOUT come out right, one mechanism):**
- The I→R 180° correction pivots about the GROUP bbox centre if grouped (whole block
  rotates coherently → orientation AND line order/spacing self-correct — fixes the
  "rigid block mislocated" symptom), or about EACH TEXT's own bbox centre if ungrouped
  (each corrected in place, inter-line spacing preserved — fixes the overlap@90/270 and
  order-reversal@180 symptoms). Orientation + layout solved by one op; pivot varies by grouping.

**Group/ungroup MVR rule (CHOSEN): adopt-group-value, no dormant state.**
- Invariant: a text's MVR is always exactly what it currently displays; grouping/
  ungrouping carries NO hidden history. Group imposes one MVR (coherent block); on
  ungroup, every text adopts the group's current MVR as its own. Reversal = undo stack,
  not hidden item state. Chosen because it needs zero new persistent state and structurally
  cannot produce a displayed-vs-stored divergence (the live-vs-reload bug class we just killed).
- REJECTED — AND-semantics: group ON only if all members ON else OFF; ungroup keeps it.
  Lossy + arbitrary collapse rule; no advantage over chosen.
- REJECTED — dormant/lossless: members retain hidden MVR through grouping, restored on
  ungroup. Lossless but adds invisible per-member state (extra XML serialization) and can
  cause visual jumps on ungroup; invisible state is exactly the bug class to avoid.

**Existing setting to PRESERVE (do not disturb):** "Keep at bottom of page" (grouped only)
— pins group to page bottom (bbox-centre aligned to element centre), keeps text readable,
makes group non-selectable; on undo it stays drawn but becomes selectable/movable again.
Works correctly today. Layer-2 must not change it.

**Deferred sub-idea:** rename "Maintain Visual Rotation" → "Match drawing orientation" (or a
crisp two-word equivalent). Likely EN-lang-file-only. Research KiCad/other-CAD naming first.

**Mechanism note for implementer:** mirror/flip and layer-1 confirmed fully independent
(clean merge, no conflict — separate code paths). The suppressed
DynamicElementTextItem::parentElementRotationChanged keep_visual counter-rotation is the
hook MVR-ON re-uses; MVR-OFF is the new I→R correction. Decide readability from the text's net
LOGICAL orientation (see implementation note below), gated by the MVR switch and the grouping-dependent pivot.
(Implementation note: readability is computed from LOGICAL state — parity = mirror XOR flip,
theta = snapped sum of rotations — NOT from sceneTransform determinant, because QET mirror/flip
rewrites positions and leaves the linear matrix [1,0,0,1], det always +1. The apply-architecture
funnels rotate+mirror+flip through applyMirrorFlip with ordered correct-then-compensate, plus an
(A′) guard: plain unmirrored rotate short-circuits compensate to protect the layer-1 hot path.)

**Known quirks — INTENDED behaviour, documented, pending community feedback (do NOT file as bugs):**
1. **MVR=ON on multiple ungrouped text lines overlaps at 90°/270°.** Each ON line is forced to
   tops-up horizontal about its OWN bbox centre; independent lines have no shared frame, so they
   collapse onto each other at vertical orientations. KEPT (not restricted to grouped-only) because
   (a) older .qet files may already use ungrouped MVR=ON and disallowing it could break their
   layout on open; (b) it's user-recoverable (drag the lines apart) and occasionally useful;
   (c) affects few users rarely. Stacking is a GROUPED concept — ungrouped text has no ISO
   "stacking", it's only ISO-conformant by rigid rotation from its authored start.
2. **MVR ON→OFF carries no history; OFF is forward-acting, not retroactive.** Selecting OFF does
   NOT undo what ON did — it does not restore a prior orientation/position. OFF's contract is:
   from this point forward, keep the text ISO-conformant through subsequent M/F/R transforms
   (flip the I cases, leave R). If ON left text in an unwanted orientation, the user de-rotates it
   manually ONCE; OFF maintains ISO from there. This is the same stateless principle as
   "MVR = what's currently displayed, no hidden history". Confirmed by test: set OFF after an ON
   episode, then rotate 90/180/270 — each subsequent rotation is ISO-correct (PASS).
3. **Non-cardinal angles generalize acceptably (confirmed, Jun 2026).** Ungrouped text set to
   non-cardinal angles (tested 20°/70°, boundary 44°/46°) retains the angle relative to the
   element correctly — the readability correction works beyond the assumed 90°-step design
   intent. Supersedes an earlier assumption that non-cardinal angles were out of scope; users do
   reach them and current behaviour is acceptable (genuine ambiguity only at the 46° boundary
   case, where either orientation reading has a reasonable argument).

**Group/ungroup MVR (as-built, LEAVE AS-IS pending community input):** the group exposes no MVR
control; grouping a mixed set leaves each text's own MVR untouched, and ungroup reverts each text
to its individual pre-group MVR value (de-facto pass-through, since ElementTextItemGroup has no
keep-visual concept). The earlier "adopt-group-value / ungroup→all-OFF" idea is NOT adopted —
keep the pre-existing revert-to-individual behaviour for backward-compat until community feedback.

### Phenomenon B — cumulative text position drift on user-rotate + element-rotate  · RESOLVED (bd61ca17c)
**RESOLUTION (Jun 2026):** fixed per the plan below, option (b) — shared `rotationpivot.h` helper (`centerPivotEndPos`, renamed from `centrePivotEndPos` in the nomenclature pass), parallel "pos" compensation applied at `RotateSelectionCommand`'s directly-selected dynamic-text (:60) and group (:67) branches. The dialog path (`RotateTextsCommand`) needed no fix — confirmed already centre-pivoting correctly; the defect was isolated to the right-click "Rotate 90°" quick-increment button, which routes through `RotateSelectionCommand`, a different command to the one originally scoped and patched first. Test matrix: items 1–4, 6 PASS clean. Item 5 (interleaved R90°+rotate+mirror+flip, save/reload) returned PARTIAL — text position momentarily diverged from expected, then converged to zero divergence after one subsequent element rotation (90° minimum observed to clear; no further accumulation beyond that), with save/reload showing no accumulated error in either state. Root-caused as a DIFFERENT, pre-existing defect — not a flaw in this fix — see the new "correction-trigger lag" entry below, which this PARTIAL result is the founding evidence for. Pushed as bd61ca17c (fast-forward on top of 6f8dd772c); nomenclature normalization landed alongside as feeef3cea.
**Area:** element text rotation/readability · **Branch:** mirror-flip-rotate · **Blocked (historically):** layer-2 orientation-correction commit — no longer blocking, see RESOLUTION above
**Symptom (confirmed via screenshots, frames 04-07 + retest):** when a text's OWN rotation is changed by the user (right-click "choose text orientation" / R90° UI), then the element is rotated repeatedly, the text's position creeps progressively further from the element reference point — accumulating per element rotation. NOT orientation (orientation re-establishes ISO correctly — that's Phenomenon A, working as designed); this is POSITION accumulation. Confirmed for MVR=ON text (and all text defaults to MVR=ON at instantiation); also reproduced with MVR=OFF (select OFF, rotate element 90°, user-rotate text 90° → bbox-centre rotates about the text top-left corner, then subsequent element rotation corrects about bbox-centre).
**Root cause (confirmed by observation):** PIVOT MISMATCH. The layer-2 readability correction pivots about the text's bbox CENTRE; the user-rotate UI pivots about the bbox TOP-LEFT CORNER (or thereabouts). The two don't share a pivot, so position doesn't round-trip — each correction recomputes from a position the user-rotate displaced, and the residual accumulates. Same drift family as the layer-1 rotation drift, but in POSITION, triggered by own-rotate interleaved with element-rotate. NOT caught by gate β-1 (which tested clean rotation only, no user own-rotate mid-sequence).
**Fix direction (agreed):** make the text user-rotate UI pivot about the bbox CENTRE (match the correction). Centre-pivot is rotation-invariant for position (centre stays fixed → no translation on repeated rotation) and makes user-rotate + correction COMMUTE → idempotent by construction. Preferred over making the correction match the corner-pivot (corner-pivot inherently translates the bbox per rotation — keeps drift potential alive). Likely also fixes a latent pre-existing "text jumps when rotated" UI inconsistency.

**Scoping done (CC, /understand-anything, graph@8fd5431be used as map + verified live) — DECIDED: option (b).**
Headline: there is no pivot *function* to edit. `RotateTextsCommand` (qetdiagrameditor.cpp:1540 sole caller → rotatetextscommand.cpp:73-76) just animates the `rotation` Q_PROPERTY; `QGraphicsObject::setRotation` pivots about `transformOriginPoint`, never set (default (0,0)) — for a text whose boundingRect starts at (0,0) that IS the bbox top-left. Comment at diagramtextitem.cpp:397 confirms: rotation about (0,0), redefinable in subclasses. So "corner pivot" = absence of position compensation, not a routine.

Two structurally different fixes were on the table:
- **(a)** `setTransformOriginPoint(centre)` — item-wide pivot policy; every `setRotation` on that item centre-pivots from then on.
- **(b)** command-local position compensation — add a parallel "pos" animation to `RotateTextsCommand` that holds bbox centre fixed (same math as the committed `rotateAboutOwnCentre`), pivot machinery untouched.

(a) is the better long-term substrate (it's literally "rotate about point P" — the mechanism parking-lot #1/#2 anchor-point rotate/move will need), so it wasn't dismissed lightly. Tiebreaker: how does the already-committed `correctReadability`/`rotateAboutOwnCentre` (element.cpp:1151-1163, commit 6f8dd772c) achieve ITS centre-pivot? Read the code — option (2): `setRotation` about default (0,0) **+ manual pos compensation** (`newPos = pos + R(own)·C − R(own+δ)·C`); comment explicit that transformOriginPoint is NOT mutated. So (a) now would double-count against that committed compensation → **(b) wins for this fix**; adopt transformOrigin-as-pivot later, in the same pass that migrates `rotateAboutOwnCentre`, when #1/#2 are actually scoped. Guardrail: don't build the transformOrigin-as-single-pivot-truth refactor now — that's anchor-feature scope, not B's.

Two reasoning checks run during scoping (both relevant if (a) is ever revisited for #1/#2):
1. transformOriginPoint governs only the item's OWN transform (rotation/scale), not how the parent element's transform carries the text in the scene — "inherited element transform" is NOT part of (a)'s blast radius (initial claim corrected).
2. transformOriginPoint IS item-wide: under (a), the directly-selected-child branches in rotateselectioncommand.cpp:60 (dynamic text) and :67 (group) — which fire only when text/group is selected directly, not via its parent element — would also silently flip to centre-pivot. Confirmed, and would widen test surface + collide with correctReadability's compensation on those items. Not wanted now; relevant scope note for whenever (a) is revisited.

**Implementation plan (b):** `RotateTextsCommand` — add a parallel "pos" animation alongside the existing "rotation" animation in the same `QParallelAnimationGroup` (rotatetextscommand.cpp ~73-76); undo/redo (:88/:100) reverses for free since the group runs backward/forward. No `transformOriginPoint` mutation. No touch to `rotateAboutOwnCentre`. `fromXml` confirmed NOT in this path (dynamicelementtextitem.cpp: setRotation at :164, setPos at :221, no RotateTextsCommand involved) — existing-file load is provably unaffected; this is a live-UI-action-only change.

**Test matrix (from scoping, supersedes the one-line "Verify" below):**
*Existing-file regression (must be byte/render-identical):*
1. Open several existing .qet with rotated dynamic text (0/90/180/270) → unchanged (load path doesn't invoke RotateTextsCommand).
2. Open → save without touching text → x/y/rotation byte-identical.
*Phenomenon B acceptance:*
3. Dialog-rotate (90/180/270) + element-rotate ≥2 full 360° laps → no cumulative drift, returns to lap-start each lap.
4. Interleaved dialog-rotate + element-rotate + mirror + flip, mixed order → save/reload → live == reload.
5. Undo/redo: dialog-rotate → undo → exact original pos+rotation; redo restores (confirms parallel pos animation reverses cleanly).
6. Grouped text: dialog-rotate a group → pivots about group bbox centre coherently, member spacing preserved.
7. Layer-2 consistency: re-run a γ-subset ({plain, M↮F, M&F} × a few of {0,90,180,270}) with the now-consistent pivot → orientation still R, position correct.

**Caution:** changing the user-rotate pivot is a behavioural change to a PRE-EXISTING UI action — verify it does NOT shift text in EXISTING .qet files on load (stored positions should be untouched; only the moment-of-rotation behaviour changes). Check whether the corner-pivot is text-rotate-only or shared by other rotation paths (element rotate already uses centre) — surgical if isolated.
**Verify (measure, don't assume):** per-text dump of own + pos/ctr across interleaved user-rotate + element-rotate ≥2 full laps; ctr must return to start (no accumulation). Plus existing-file load unperturbed. THEN this fix + the layer-2 orientation correction commit together as one clean piece (tempered commit message).

### Readability-correction trigger lag (orientation AND position)  · RESOLVED (023070fd8 extract + 0a734cdc5 fix, pushed)
**Area:** element text rotation/readability · **Branch:** mirror-flip-rotate · **Related:** found via Phenomenon B's test item 5; orthogonal to it — B was WHERE rotation pivots, this is WHEN `correctReadability` re-fires.
**Symptom:** after a user changes a text's OWN rotation (via "Choose text orientation" dialog OR the "Rotate 90°" quick button), the text is NOT immediately re-evaluated by `correctReadability` — it only re-fires on the FOLLOWING element-level rotate/mirror/flip event. Affects POSITION as well as orientation; in a mixed-order sequence it converged only after a subsequent element rotation, zero residual once cleared, save/reload clean in any state.

**Investigation complete (Jun 2026, traced against live tree post-feeef3cea — graph predated it, so verified from code not graph):**
- `correctReadability` (element.cpp:1175, private) has exactly TWO call sites, both element-level: `applyMirrorFlip()` (:1128/:1134) and `onElementRotated()` (:1211/:1215). `onElementRotated` is wired only to `Element::rotationChanged` (element.cpp:134); `applyMirrorFlip` to `setMirror`/`setFlip`/`fromXml`-load/`onElementRotated`. `rotateAboutOwnCenter` is reachable only from inside `correctReadability`. CONFIRMED: nothing is wired to a text's own-rotation event.
- The text DOES already emit + self-connect an own-rotation signal: `DynamicElementTextItem::rotationChanged` → `thisRotationChanged` (dynamicelementtextitem.cpp:1302-1306/1501-1502), but that slot only bookkeeps `m_visual_rotation_ref` (:1315-1317) — no `correctReadability`. `parentElementRotationChanged` early-`return`s (:1297; layer-1 suppression). `ElementTextItemGroup::setRotation` emits `rotationChanged(angle)` (:577), connected only to UI. So the signal exists; it just doesn't reach the correction.
- **SINGLE gap, confirmed from code (not just the one sequence):** `correctReadability` reads `item->rotation()` LIVE at fire time (`net_raw = rotation() + item->rotation()`, element.cpp:1176) — not cached, not `m_visual_rotation_ref`. So once ANY element-level event fires it reads the user's current own-rotation and corrects properly; the mirror/flip calls are not stale. Two-gap possibility ruled out. `m_visual_rotation_ref` is vestigial for readability (only used in the dead `parentElementRotationChanged` path).
- **Animation hazard (why it's not a one-line connect):** both rotate commands ANIMATE the rotation property → `rotationChanged` fires per frame. A naive connect to `rotationChanged` would (a) fire `correctReadability` every frame and (b) re-enter (`correctReadability`→`rotateAboutOwnCenter`→`setRotation`→`rotationChanged`). So the trigger must fire ONCE on finalize, not per frame. Signal-driven approach (ii) NOT adopted.

**Fix — option (i), command-driven finalize call (DONE):**
- New thin PUBLIC `Element::correctTextReadability(QGraphicsItem *item)` wraps the private `correctReadability` (resolves the item's `keepVisualRotation()` per type). `correctReadability` stays private.
- `RotateTextsCommand` (always animates via its `QParallelAnimationGroup`): connect the group's `finished()` once → `finalizeReadability()` corrects each text/group via its parent element. Fires after forward (redo) AND backward (undo) animations; idempotent so both land correct. (Verified live: the `finished()` group emits only after BOTH the rotation and pos child animations apply their end values — no race; the `SETTLED` dump showed rot/pos already settled.)
- `RotateSelectionCommand` directly-selected dynamic-text + group branches: those branches' rotation+pos commands are made SYNCHRONOUS (element/conductor/image keep animation), and `correctSelectedTexts()` runs in BOTH `redo()` and `undo()`.
- Re-entrancy verified LIVE (not just trace): the correction's `setRotation` → `thisRotationChanged` bookkeeping only; no loop, no anim restart.

**SECOND gap found in testing — combined mirror+flip POSITION lag (now also fixed in 0a734cdc5):** the original scope only re-fired `correctReadability` (orientation). But `applyMirrorFlip` does, per item, `correctReadability` THEN `compensate` (the per-item reflection POSITION transform). On a text own-rotation inside an already-mirrored/flipped element, orientation re-corrected but the reflection position stayed stale until the next element event — text "converged only after a subsequent M|F". Fix: extracted `compensate` from the `applyMirrorFlip` lambda into private `Element::compensateMirrorFlip` (commit 023070fd8, no behavioural change — verified against an EXTRACT-ONLY build, Group C element-driven matrix), and `correctTextReadability` now calls `compensateMirrorFlip(item)` when `m_mirror||m_flip` (matching `onElementRotated`'s short-circuit: plain rotation skips compensate). This is ALSO why the correction must run on `undo()`: `compensate` sets an ABSOLUTE item transform that the rotation/pos reversal does not restore, so undo re-runs the correction to recompute it for the reverted rotation. (Confirmed with instrumentation: undo's `xform`/`pos`/`rot` == pre-rotate `BEFORE` exactly.)

**Test matrix (all PASS):** A — text own-rotate (dialog AND R90°) while element already Mirror / Flip / Mirror+Flip, grouped+ungrouped → settles immediately, no subsequent element event; `xform` non-identity confirms compensate fired; bbox center invariant across pivot-comp + correction (no compounding). B — undo→exact original (pos+rot+**transform**), redo restores. C — extract regression: element mirror/flip/rotate 0/90/180/270 × {plain,M,F,M+F} × {grouped,ungrouped} unchanged; classifier `parity`/`net`/`inverted` match the table, gate-1 cells (plain 90°=I, 270°=R) intact.

**Guardrails (held):** rotation-sign / +y-south idea NOT touched (still its own feature-spec). Classifier (`parity = mirror ^ flip`) untouched — only a TRIGGER + the existing compensate were added; gate-1 table verified unaffected (Group C). Element-creation-time text (terminal numbers/idents) is a SEPARATE open entry, untouched.

**Commit structure:** 2-commit split — 023070fd8 (pure `compensateMirrorFlip` extract, no behaviour change, verified against an isolated extract-only build so the "verified" claim matches that commit's tree) then 0a734cdc5 (the trigger + compensate-on-own-rotation fix). main.cpp never staged.

### Genuine ungroup of a mirrored element displaces text  · DEFERRED
**Area:** element text groups · **Branch:** folio-mirror-flip
**Location:** `Element::removeTextFromGroup` → `elementtextitemgroup.cpp:1367`
Genuine ungroup of a mirrored element (Properties → Delete-group) displaces
grouped text ~19px; persists to disk (confirmed via round-trip). Same
`removeFromGroup`-through-reflection family as the committed save-path fix
(05bcba506) — likely the same shape of fix: re-express child geometry in one
mirror-free frame instead of relying on the scene-preserving detach.
**Severity:** low — reachable only via Properties → Delete-group; no
right-click ungroup gesture exists.
**Related (separate, not a dependency):** shares the ungroup/`removeFromGroup` path with the LAYER 2 readability entry. Distinct concern (this = mirror geometry to disk; that = rotation orientation display). Whoever codes the second of the two must verify it doesn't regress the first in the shared function. (This is the smaller, known-shape fix — cf. 05bcba506 — so it's a low-risk standalone session whenever convenient; no strict ordering required.)
**Next:** own scoped session; read 1367 path, confirm same-shape vs trickier
(it touches the shared removeFromGroup that's *correct* for unmirrored ungroup),
write fix + verification matrix before implementing.

### Grouped text at non-cardinal angles: offset overridden on ungroup/regroup  · DEFERRED (edge case)
**Area:** element text rotation/readability · **Branch:** mirror-flip-rotate
**Symptom (Jun 2026):** grouped text set to a non-cardinal angle has its angular offset overridden on ungroup/regroup despite MVR=OFF; one case showed a "wild pirouette" away from the element on ungroup. At an intermediate stage, the group's selection-box position decoupled from the text's actual bounding-rect (looked like the selection box mirrored/flipped about the text's real position). Not a crash; recoverable; narrow use case (ungrouped non-cardinal text is unaffected — see LAYER 2 known-quirks #3).
**Lead:** deleting and reinstating 3 lines of text into a FRESH group (MVR=OFF before grouping) made behaviour match the well-behaved ungrouped-multiline case — suggests stale state surviving regroup, not a flaw in the angle math itself. Check what state regroup fails to reset.
**Severity:** low — edge case, no crash, user-recoverable. Own scoped session when convenient.

### Element-creation-time text rotation — same class as DynamicElementTextItem  · NEEDS RE-TEST (likely not a separate defect)
**Area:** element text rotation/readability · **Branch:** mirror-flip-rotate
**Framing carried into this scoping:** the trigger-lag work showed one gap can hide a second once mirror/flip interaction is checked — for `DynamicElementTextItem` the orientation-trigger and the position-compensate were two separate gaps, not one. So this re-test must check ALL THREE (pivot, own-rotation trigger, stale-compensate-under-mirror/flip), not just the originally-flagged top-left pivot.
**Findings (CC, Jun 2026):** no separate class exists. Legacy `<input>` template fields (`Element::parseInput`, element.cpp:549) — including tagged ones, i.e. terminal idents/labels via `setTextFrom(ElementInfo)`/`setInfoName` — are converted to `DynamicElementTextItem` and appended to `m_dynamic_text_list`. Modern `<dynamic_text>` (`parseDynamicText`) does the same. Both land in the SAME list as user-added dynamic text — origin differs (template vs user), class and runtime path are identical. The only other template text, `<text>` → `QGraphicsSimpleTextItem` (`ElementPictureFactory`, static picture text drawn in `Element::paint`), is not a child item and not in `selectedTexts()` (diagramcontent.cpp:143) — not individually selectable/rotatable, rotates rigidly with the picture, outside the rotate-command/readability system entirely. So the rotatable terminal text goes through the EXACT path just fixed: `RotateTextsCommand`/`RotateSelectionCommand`'s `DynamicElementTextItem` branch → `centerPivotEndPos` (pivot, bd61ca17c) → `correctTextReadability` → `correctReadability` + `compensateMirrorFlip` (trigger+compensate, 0a734cdc5). Handled by `type()`, not `TextFrom` — `ElementInfo` vs user-text is irrelevant to the path.
**Verdict:** NOT a separate defect surface. All three potential gaps apply identically to user-added `DynamicElementTextItem` and are ALREADY FIXED. The original "also pivots top-left" observation almost certainly predates the R90° pivot fix (when every `DynamicElementTextItem` pivoted top-left). No new fix code indicated.
**Next — RE-TEST, not a fix:** on a terminal-number/ident text (`ElementInfo` `DynamicElementTextItem`), rotate (dialog + R90°), plain and under mirror/flip, grouped+ungrouped — expect identical pass to user-added text (centre pivot, immediate readability + position, undo restores transform). CAVEAT: `<input>`-converted texts default to `keepVisualRotation=true` (MVR=ON) — set MVR explicitly so the retest isn't read under the wrong mode. If it passes, CLOSE this entry. If it unexpectedly fails, check selection routing first: `RotateSelectionCommand`'s direct-text branch fires only when the text is selected with its parent element NOT selected.
**Reusability:** uses `rotationpivot.h`'s `centerPivotEndPos` directly via the existing command path — no class-specific variant needed (it IS `DynamicElementTextItem`).
**Boundary:** distinct from the DEFERRED ungroup/regroup bugs nearby in this subsystem — separate concerns, don't fold together.

### Title-block vars: id/total braced form fails  · OPEN (docs)
**Area:** title block variable substitution · **Branch:** qt6_cmake_joshua @ f16cf7d
**Location:** parser `titleblocktemplate.cpp:1729-1731` (interpreteVariables);
injection `qetproject.cpp:903/1931/1935` (setFolioData)
Parser does BOTH `%{key}` (1729) and `%key` (1731) for every DiagramContext key,
so syntax is never the discriminator — context membership is. But folio-position
vars `id`/`total` are injected via setFolioData on a path that emits only the
bare form; they never reach the context loop, so `%{id}`/`%{total}` pass through
literal while `%id`/`%total` work. Metadata vars (projecttitle, saved*, plant,
custom) DO go through interpreteVariables and take either form.
Not a regression — history (28fca988b, ad2989384) shows both behaviours
long-standing, predating the Joshua autonum refactor.
**Next:** doc fix only — manual + in-app tooltip ("%filename") imply one
universal `%{}` syntax; should state metadata = either form, folio-position
(id/total) = bare `%` only. Report to Laurent / forum.

### Title-block vars: projecttitle/saved* render literal  · NEEDS-TEST
**Area:** title block variable substitution · **Branch:** qt6_cmake_joshua @ f16cf7d
**Location:** `qetproject.cpp::updateDiagramsFolioData:1918-1935`
projecttitle/saved* render literal in BOTH forms even with project saved + title
set. Context built WITH these keys (1918-1920) but rich propagation is entangled
with the `folio().contains("%autonum")` gate (1928); a `%id / %total` folio takes
the else branch (1935). Both branches pass project_wide_properties so it SHOULD
arrive — hence suspected, not confirmed.
**Open question (1-min test):** set a folio's Folio field to contain `%autonum`;
does `%{projecttitle}` then render?
 - yes → metadata propagation gated behind autonum path = real defect at
   `qetproject.cpp:1928`.
 - no → key drops between addValue (1918) and template context; trace
   setFolioData → BorderTitleBlock → DiagramContext merge.
Resolve before reporting to Laurent.

### Copy of element with grouped text → duplicate embedded definitions?  · OPEN (investigate)
**Area:** project file hygiene / .qet serialization · **Branch:** —
Copying an element that contains a group pollutes uuid referencing. NOT
"should resolve back to on-disk .elmt" — embedding is intentional (portability;
project stays self-contained if the source .elmt moves/changes/vanishes). The
real question: does copy create DUPLICATE embedded definitions of the same
element type (file bloat) vs multiple instances sharing ONE embedded definition?
**Next:** copy an element, save, check whether the .qet has two embedded
definition blocks for the same type or one shared block with two instance refs.
Only a bug if duplication; if so the fix is de-dup, NOT un-embedding. Don't open
as tracked work until confirmed (may be a non-problem).

---

## ARCHIVE — resolved / superseded

### RESOLVED — live-rotate displacement of element text (rigid rotation)
**Commit:** 843ba6898 · **Branch:** fix-grouped-text-rotation-pivot
On live element rotation, `DynamicElementTextItem::parentElementRotationChanged`
(dynamicelementtextitem.cpp:1302, on `Element::rotationChanged`) counter-rotated
each text (`m_keep_visual_rotation`) to hold its visual orientation. For text
carried by the element transform this double-counts: text stays readable but is
displaced, and its own-rotation drifts 180° per save/reload cycle. Reload never
runs this path, so saved files always reopened correctly — bug was display-only,
live diverging from reload. Fix: suppress the counter-rotation so ALL element
text (grouped, ungrouped, single/multi-line) rotates rigidly with the element,
keeping design own-rotation 0 — matching what reload produces (upside-down but
correctly placed at 180°). `m_keep_visual_rotation` left intact (still
serialized/editable) for readability reintroduction. Save path unaffected
(removeFromGroup resets text rotation before toXml). VERIFIED: double round-trip
per-text dump, grouped + ungrouped — live own-rot 0 == reload, positions stable
through 900° (2.5 turns), no drift, live render == reopened-file render.
First attempt: FAIL×5 → PARTIAL → PASS (see fix-metrics 843ba6898).

<details>
<summary>Diagnostic playbook — kept for layer-2 reference (the de-rotation/readability follow-on)</summary>

Eliminated, each with data (do NOT re-try as a layer-2 approach without cause):
1. setTransformOriginPoint(bbox.center()) in updateAlignment — no-op (group's own
   rotation always 0; all rotation is on the parent Element).
2. connect(Element::rotationChanged -> updateAlignment) — fires but render unchanged.
3. element->update() — element repaint does NOT invalidate child group's cached render.
4. group prepareGeometryChange()+update() — no change.
5. forced-unblocked updateAlignment on rotate — runs but does NOT reproduce reload.
6. first parentElementRotationChanged guard (grouped-only) — PARTIAL: fixed position
   but own-rotation still drifted (live-180 own-rot 180 vs reload 0).
Decisive method: single-run per-text dump (element+group+each text:
pos/rotation/transform/mapToScene) at (A) live RotateSelectionCommand::redo and
(B) reload fromXml, SAME placement. Showed element+group+text POSITIONS identical
A=B; ONLY difference = each text's own rotation (live 180 vs reload 0); all
transform matrices [1,0,0,1] (no mirror) -> pure rotation double-count. Selection
probe ruled out RotateSelectionCommand (only the element is selected; group/texts
not in selectedItems). Root located at parentElementRotationChanged keep_visual.

LAYER 2 (next, deferred — see memory note): reintroduce readability properly —
per-text de-rotation about each text's own bbox centre, for ALL text types, with
correct relative LAYOUT preserved (inter-line spacing + order), not just
orientation. Current intermediate state: all text upside-down at 180° / vertical
(rigid), correctly placed. Standard: text reads from BOTTOM or RIGHT, never
inverted (AutoCAD TORIENT / NASA GSFC / ISO); vertical = read-from-RIGHT; 90° the
likely classically-mishandled case. Evidence layer-2 must own layout: pre-fix
ungrouped text stayed readable but lines overlapped at 90/270 and reversed order
(3,2,1) at 180 — readable-but-mislaid. Mirror/flip (folio-mirror-flip) relies on
keep_visual semantics — check interaction when building layer 2.
</details>

### RESOLVED — grouped rotated text: mirror corrupts on save/reload
**Commit:** 05bcba506 · **Branch:** folio-mirror-flip
toXml detached each child via the visual-preserving
ElementTextItemGroup::removeFromGroup(), baking the element mirror into child x/y
(half-applied — position only — then re-asserted by applyMirrorFlip on load), so
grouped rotated text saved displaced by one group width. Fixed by serializing
each child's element-relative "design" geometry directly from the group's
pos/rotation in one frame (element.cpp Element::toXml grouped-text loop).
Verified: acceptance + regression (3-line group, element-rot 90/180 + mirror,
mirror+flip) all det-parity +1 and idempotent on re-save.

<details>
<summary>Diagnostic playbook used (Checks A/B/C) — kept for method reference</summary>

Mechanism established before fix: det(ct) was NOT diagnostic (det -1 for both
mirror and flip, and for passing ungrouped cases). Real signal was child frame
shifted +136 (one group width) in x in the saved->reloaded state while the
orientation reflection lived in the runtime ct.
- **A - serialization dump:** confirmed +136 persisted in child x/y; no reflection
  serialized (only x/y/rotation/alignment/keep_visual flags written).
- **B - mutation point:** the reflected group transform is present at addToGroup
  on reload (det -1), interacting with compensate's re-derivation; the mirror
  lands in serialized child x/y, double-counting against the recomputed ct.
- **C - scene-transform parity:** group +1 vs child -1 at toXml(save) confirmed
  an orientation-parity bug at the SAVE path, not reload; the +136 is the
  positional shadow of that reflection.
Order that worked: A+B jointly located it; C confirmed end-to-end. Same
division of labour to reuse for the 1367 sibling: establish scope + reference
math here, CC implements + visually verifies.
</details>

---

## FEATURE IDEAS — parking lot (not scoped, not committed)

Unfiltered intent capture. Each needs scoping (feasibility, scope, QET-fit,
upstream appetite via Laurent) before becoming ACTIVE work — especially the
architectural ones (#3-#5). KiCad cited as the UX gold-standard reference
throughout; research its exact behaviour when scoping. Don't let an idea jump to
implementation without a feasibility pass — same scope-before-build discipline
as the bug work.

### 1. Visible Dynamic-Text anchor points
Render a visible depiction of an Element's Dynamic Text anchor (insertion)
points — currently invisible, so positioning is guesswork. Foundation for #2.

### 2. KiCad-style "move-and-place" for text & components
Make moving/placing as intuitive as KiCad. Target behaviour:
- Hovering a component / selected components (element editor OR placed element at
  folio level), pressing **M** enters move mode relative to the item's
  highlighted reference/insertion point.
- Mouse drag moves the selection; the pointer "jumps" to hover near the item's
  anchor point so you drag *by the anchor*, not an arbitrary grab point.
- A subsequent left-click / Return places the item at the element-or-folio
  coordinate currently under the floating anchor.
- Snap-to-grid as a toggle (currently effectively always-on); likely overridable
  per-drag via a modifier (Shift?) for free placement.
Depends on #1 (anchor must be visible to drag by it). Research KiCad's exact
move / snap / modifier semantics when scoping.

### 3. Named Dynamic Text on Terminals
Give Terminals dynamic-text labels, KiCad-style. Likely needs to become the
DEFAULT terminal style, with backward-compat for existing un-named terminals
(they keep working unchanged - no forced auto-naming/migration). Unlocks:
- a wiring netlist (#4) can feed a project BOM (#5);
- "every unconnected terminal in a project is trivially identifiable";
- at element-definition level, associativity between a Terminal and its
  description - not currently possible.
Cornerstone feature: #4 and #5 partly depend on it.

### 4. Project netlist
Wiring netlist functionality across a project. Depends on #3 for terminal
identity / association.

### 5. Project BOM
Bill-of-materials generation. Fed by element data fields (#6) and, for wiring,
the netlist (#4).

### 6. Saner element data-field entry
Ease filling element data fields, using KiCad's component-data model as the
gold standard. A few well-named core fields defined; the rest (auxiliary1,
auxiliary2, ...) revert to user-defined / created-on-demand rather than always
present. The fixed auxiliaryN slots currently clutter the UI and obscure which
field is the right home for a given datum.

### 7. MVR-aware text context menu (expose MVR + gate rotate)
Two paired UX affordances around the "Maintain visual rotation" (MVR) switch:
(a) expose the MVR toggle at the right-click / context-menu level on a text
(currently only in text properties), so the user can see/toggle the setting that
governs orientation right where they'd reach for rotate; (b) grey out / disable
the text-rotate controls (right-click "choose text orientation" + the properties
rotate field) when MVR=ON, since manual rotation is overridden by the ISO
correction while MVR holds orientation — a dead/confusing control otherwise.
The two pair naturally: greying-out rotate prompts "why?", and exposing MVR in
the same menu answers it. Either alone is half the story. Separate from the
layer-2 correctness work (this is UI affordance over that behaviour); touches
menu/UI code, not the geometry — cleaner as its own commit/PR.

### 8. Non-destructive Rotate/Mirror/Flip RESET
Clear all transforms on a selected element, returning it to its on-disk render,
while KEEPING all user-added text. Grouped-rotated text transforms are exempt
from the reset (their rotation is relative to the group, not the element's
transform stack — resetting the element shouldn't move them). Useful as an
"undo all my fiddling" escape hatch distinct from the regular undo stack.

### 9. Enable Mirror and Flip while placing an element
Rotate already works during placement (before the element is dropped onto the
folio); Mirror and Flip currently don't. Parity fix — let the user orient an
element fully before committing its position, same as Rotate already allows.

### Dependency sketch
- #1 -> #2 (anchor visible before drag-by-anchor).
- #3 -> #4 -> #5 (terminal identity -> netlist -> BOM); #6 also feeds #5.
- #3 is the highest-leverage cornerstone; #1 is the cheapest standalone win.
- #8, #9 are standalone — no dependencies either direction.
