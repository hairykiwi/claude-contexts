# QET Fix Metrics Log

Per-commit record of Claude Code session efficiency.
Format: commit hash — subject, then token/tool/graph/first-attempt data.
Cross-reference: `git log upstream/qt6_cmake_joshua..HEAD`

---

## 0829472661424ac9f2f8fb2186a6ff96c5e4bda4 — Replace qAsConst() with std::as_const() throughout
Token cost: ~35k | Tool calls: 4 | Graph queries: 0 | First attempt: PASS

## 8fd5431be006645750703bc70430d45027636e34 — Fix QFont::setWeight weight-0 warning for Qt6
Token cost: ~80k | Tool calls: 36 | Graph queries: 1 | First attempt: PASS

---
*Entries below from folio-mirror-flip session, 2026-06-08*

## 269ab1dd3 — Add folio-level Mirror/Flip for placed elements
Token cost: ~180k | Tool calls: ~30 | Graph queries: 1 | First attempt: PASS

## cb77d37ed — Add missing French translations for Mirror/Flip actions
Token cost: (same session) | Tool calls: 3 | Graph queries: 0 | First attempt: PASS

## d97501d12 — Fix conductor orientation on folio-level mirror/flip
Token cost: (same session) | Tool calls: 4 | Graph queries: 0 | First attempt: PASS

## 99402931d — Add M/F keyboard shortcuts for Mirror/Flip in folio view
Token cost: ~12k | Tool calls: 5 | Graph queries: 0 | First attempt: PASS

## c6344261f — Fix rotated dynamic text under folio mirror/flip
First attempt: No — 2 math iterations (initial t=E·K⁻¹ wrong for rotated
text; corrected to t=R(rot0)⁻¹·E·K⁻¹ after recognising Qt QTransform
pre-multiplies / setTransform applies outside item rotation) + 1 live
debug cycle to confirm setTransform is honoured and isolate the bug as
pure composition-order math.
Graph queries: 0
Tool calls / token cost: not captured
Scope: ungrouped dynamic text only; grouped-rotated nested-transform defect
deferred.

## 05bcba506 — Fix grouped rotated dynamic text corruption on folio mirror/flip save
First attempt: PASS — patch compiled and passed full acceptance + regression
(3-line group, element-rot 90/180 + mirror, mirror+flip; all det-parity +1,
idempotent re-save) on first build. Reached via structured multi-session
diagnosis (Checks A/B/C: serialization dump, addToGroup matrix trace, live-vs-reload
sceneTransform) before any patch. One pre-apply review refinement (hard assert
-> non-fatal qWarning tripwire); bare correctAngle(group->rotation()) confirmed
byte-identical to old path.
Graph queries: 0
Tool calls / token cost: not captured
Scope: save-path serialization for grouped text. Sibling deferred: genuine ungroup
of a mirrored element (removeFromGroup/1367 path); pre-damaged-file healing.

## 843ba6898 — Fix live-rotate displacement of element text
First attempt: FAIL→FAIL→FAIL→FAIL→FAIL→PARTIAL→PASS
Five coded-and-tested fixes that FAILED outright — setTransformOriginPoint;
rotationChanged->updateAlignment; element->update repaint; group
prepareGeometryChange+update; forced-unblocked updateAlignment. Then one
PARTIAL — the first parentElementRotationChanged guard (parentGroup()-only)
fixed position but left own-rotation drifting. Then PASS — uniform
counter-rotation suppression for all element text, commit 843ba6898.
Reached via multi-session diagnosis: eliminated 4 prior hypotheses
(transformOriginPoint, rotationChanged->updateAlignment, element/group repaint,
forced-unblocked relayout — all FAILED with data), then a two-moment per-text
transform dump (A: live redo / B: reload fromXml) plus a selection-state probe
located the root cause in DynamicElementTextItem::parentElementRotationChanged
(keep_visual counter-rotation double-counting against the element transform).
The PASS version cleared two non-regression checks before coding (save path
resets text rotation in removeFromGroup before toXml; no consumer reads the
counter-rotated value) and passed the double-round-trip per-text verification
(grouped + ungrouped): live-180 own-rot = 0 matching reload, positions stable,
no 180°/cycle drift, live render == reopened-file render.
Graph queries: 0
Tool calls / token cost: not captured
Scope: layer 1 (rigid rotation / correct position) only. Layer 2 (text
readability de-rotation about own centre) deferred; m_keep_visual_rotation
kept intact for its reintroduction.

## b0a405329 — Fix grouped dynamic text displacement on folio flip
Token cost: not captured | Tool calls: not captured | Graph queries: 0 | First attempt: PASS
Gate-2 (layer-2) sub-fix, separated from the orientation correction. Measure-first:
[L2GRP] dump compared live-flip vs reload across 0/90/180/270 in one run, proving
the displacement is consistently baked into applyMirrorFlip()'s compensate() (live
== reload) and quantitatively isolating the cause — group bbox top-left offset
(br.y()=-33) vs compensate's implicit (0,0)-origin assumption (mirror keys off x
where br.x()=0, hence mirror-clean / flip-displaced). Fix = reflect about the bbox
near+far edges (left+right / top+bottom) instead of bare width/height; no-op for
origin-(0,0) bboxes so ungrouped + mirror byte-unchanged. Derived the +2*top
correction from the data BEFORE coding; implemented once, verified all four
rotations land at the exact reflection about element origin, live == reload.

## 6f8dd772c — Make rotated element text ISO-readable (folio mirror/flip/rotate) [WIP]
Token cost: not captured | Tool calls: not captured | Graph queries: 0 | First attempt: PASS (gate-driven)
Layer-2 readability — the deferred follow-on to layer-1 (843ba6898). Notably
cleaner path than layer-1's FAIL×5→PARTIAL→PASS, BY METHOD not luck: a measure-first
gate process. Gate 1 proved the I/R classifier empirically against the observation
table BEFORE any correction was wired — and caught a real classifier bug there
(sceneTransform-determinant can't see QET's position-rewrite mirror; switched to
logical-state parity = mirror XOR flip + summed rotation). Only after the classifier
matched all 24 cells did the correction get wired. The wired correction then passed
all three gates on first build: α (layer-1 plain-rotate regression clean — protected
by the A' guard that skips compensate on unmirrored rotate), β (idempotency, no
rotation drift over >=2 360 laps across {MVR off/on}x{mirror off/on}), γ (24-cell
orientation + position + save/reload). Architecture (A'): all triggers funnel through
applyMirrorFlip, orientation-correct-then-compensate, so the readability pivot never
sees a stale reflection transform. The "catch the bug at the classifier gate, not in
the field" discipline is the signal worth recording — the gates front-loaded the
failure that layer-1 paid for in re-coding.
Known limitation at time of this commit (WIP): manual text-rotate (corner pivot)
vs readability (centre pivot) mismatch causes cumulative position drift under
repeated element rotation; common no-manual-rotate case unaffected.
[UPDATE bd61ca17c: this pivot-mismatch drift is now fixed. The residual WIP item
is the correction-trigger lag — own-rotation-change doesn't re-fire
correctReadability until the next element-level event. Tracked in CC-TASKS.md.]

## bd61ca17c — Pivot directly-selected text rotation about its bounding-rect center
Token cost: not captured | Tool calls: not captured | Graph queries: 0 | First attempt: PASS (functional)
This is the Phenomenon B fix (the limitation logged just above). Option (b) from the
scoping: command-local pos compensation, no transformOriginPoint. The dialog path
(RotateTextsCommand) was already done but uncommitted from the prior session; this
commit extracts the centre-pivot end-pos math into a shared header (rotationpivot.h)
and applies the same parallel "pos" compensation to the R90° quick-rotate path
(RotateSelectionCommand dynamic-text + group branches). Confirmed RotateSelectionCommand's
QPropertyUndoCommand stores old+new explicitly, so undo reverses for free — different
mechanism from RotateTextsCommand's animation-group but same clean-undo result.
Fix LOGIC correct first build/first test: matrix 1-4 + 6 PASS; item 5 PARTIAL =
the pre-existing correction-trigger lag (own-rotation-change doesn't re-fire
correctReadability until the next element-level event), a separate known issue, NOT
this pivot fix. So First attempt = PASS on the fix itself.
Where the session time actually went (the honest signal): almost none on the fix logic,
nearly all on PR-hygiene and commit-craft. (1) Caught that bbox / British "centre" /
"layer-1/2" / "architecture A'" / "mvr" are fork-local idioms absent from upstream
(grep-verified vs master + qt6_cmake_joshua) and unfit for the upstream-bound files;
stripped them, renamed mvr->keep_visual_rotation (matching the existing member + XML
attr) and rotateAboutOwnCentre->rotateAboutOwnCenter. (2) Verified the LSP "10 errors"
panic was a missing-compile_commands.json artifact (every error rooted in a Qt header
"file not found"), not real — make built clean, 0 warnings. (3) Two-commit split
(feeef3cea = comment/naming normalization, no behavioural change; this = the fix) with
each commit kept self-consistent message<->tree, including splitting the rotationpivot.h
rotateAboutOwnCent* comment ref so it matches the function's actual name at each commit.
Lesson: for upstream-bound files, run the fork-idiom grep (bbox|centre|internal phase
labels) BEFORE writing comments, not after — would have saved several correction rounds.

## 023070fd8 — Extract compensateMirrorFlip helper from applyMirrorFlip
Token cost: not captured | Tool calls: not captured | Graph queries: 0 | First attempt: PASS
Pure extract-method refactor of applyMirrorFlip's per-item reflection-compensation
lambda into a private member, so the trigger-lag fix (0a734cdc5) can reuse it. No
behavioural change. The notable discipline here: the developer caught that the
"no behavioural change, verified" claim must describe what was tested AT THAT
commit's tree, not inferred from the final combined state — Group C had only been
run against the combined tree (extract + fix). So we rebuilt an EXTRACT-ONLY binary
(stash the command files + temporarily remove correctTextReadability) and re-ran
Group C against that, THEN committed. Set up so each commit stages whole files (no
partial-staging of element.cpp/.h): isolate commit-1 tree → commit 1 → restore fix
→ commit 2. Worth remembering as the clean pattern for split commits sharing a file.

## 0a734cdc5 — Re-correct element text readability and reflection on its own rotation
Token cost: not captured | Tool calls: not captured | Graph queries: 1 (graph stale, predated 4 commits; used /understand only as a map, verified all from live tree) | First attempt: FAIL→PASS (two gaps; second found by test)
The trigger-lag fix proper (the residual WIP from 6f8dd772c/bd61ca17c). Investigation-
first paid off: traced every correctReadability/compensate trigger from the live tree
(graph was stale), settled single-vs-two-gap for the ORIENTATION trigger from the
connection logic (not the one observed sequence) — correctReadability reads item
rotation live, so once any element event fires it's correct; the gap was purely the
missing own-rotation trigger. Animation hazard ruled out the naive one-line connect
(per-frame firing + setRotation re-entrancy), so option (i) command-driven finalize:
public Element::correctTextReadability wrapper, called from RotateTextsCommand's
anim-group finished() and RotateSelectionCommand's redo()/undo() (those branches made
synchronous so re-redo reads the settled rotation).
First-attempt = FAIL→PASS because testing exposed a SECOND, distinct gap the
orientation trace didn't predict: in an already-mirrored/flipped element the POSITION
(compensate) also goes stale on own-rotation — applyMirrorFlip pairs correctReadability
WITH compensate per item, but the wrapper only did the first. Found via the test
matrix (test 5 combined-M|F), measured with MFDBG instrumentation on the mirror/flip
path, fixed by extracting compensate (023070fd8) and having the wrapper call it under
mirror/flip. That second gap also forced the undo() correction call (compensate sets
an absolute item transform the rot/pos reversal doesn't restore).
Instrumentation discipline carried the session: per-item center/pos/rot/xform dumps +
mvr= tag (to catch MVR-ON contamination), single-run before/after comparisons, and a
SETTLED phase that empirically killed the QParallelAnimationGroup race question. The
"measure, don't hypothesise" rule earned its keep — the second gap would have been
guessed wrong (I'd hypothesised a parity-classifier miss; the dump showed parity/net
correct and the issue was the absent compensate trigger).
