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

### Shared live-compensation fix: correctTextReadability at three attach sites  · RESOLVED (ade229881, pushed) · branch mirror-flip-rotate
**What:** a post-attach `Element::correctTextReadability(item)` at the live text-attach SITES so a text
attached to an already-mirrored/flipped element compensates immediately instead of drawing geometrically
reflected until the next M/F/R or reload. Closes the "new text on already-mirrored/flipped element" OPEN
entry and the geometric-mirror half of the ungroup entry, via one shared helper (existing public
`correctTextReadability`, element.cpp:1210 — readability + `compensateMirrorFlip` when mirror/flip; no-op
otherwise). Placed at the attach sites, NOT inside `addDynamicTextItem` (load path already compensates via
`applyMirrorFlip`; an in-function hook would double-fire). **Committed sites (3):**
- `Element::removeTextFromGroup` (element.cpp, ungroup) → `correctTextReadability(text)`.
- `AddElementTextCommand::redo` (create) → `m_element->correctTextReadability(m_text)`.
- `DeleteQGraphicsItemCommand::undo`, UNGROUPED-text branch → `correctTextReadability(deti)`.
**Grouped delete-undo branch — DELIBERATELY no group-level call (a code comment marks the spot, not a TODO).**
Originally added `correctTextReadability(group)` there; T4 measurement removed it. Rationale: the group is not
deleted (only the text), so its mirror/flip compensation persists; a group-level correction was measured a
no-op (the two untouched grouped texts stayed pristine — a group-level call would have moved all three). The
restored text's misposition is the pre-existing addToGroup local-position recompute (deferred, see the
"Genuine ungroup… displaces text" entry), which a group-level call cannot fix. Same precedent as the
ISO-layout removal — don't ship currently-non-functional code on a future-use bet; if addToGroup is fixed to
recompute the local position correctly under reflection, there's likely no residual state for a downstream
group correction to fix anyway. Recoverable from this commit's history if that bet proves out.
**Scope guard (held):** the removeFromGroup/addToGroup positional-displacement (~14–24px) fix is SEPARATE/
out of scope; the non-cardinal-angle DEFERRED entry is out of scope. `addDynamicTextItem` itself untouched.
**Structural confirmation (measured):** `toXml` serializes only pos/rotation (dynamicelementtextitem.cpp:93-95),
NOT the QTransform; the change sets only the transform and is a no-op on pos/rotation for a fresh text (net=0).
Proof from two POST-FIX save files: every text shared between the mirror+flip file and the mirror-only file
has IDENTICAL saved x/y/rotation despite the differing flip flag ⇒ saved bytes are fix-independent; reload is
governed entirely by the unchanged load path.
**Test result (2026-06-24, binary mtime 10:04 then 19:02 after commit):** T1 ungroup PASS, T2 create + reload
PASS (the earlier "looked different on reload" was User's own uncertain recollection — re-observed clean,
live==reload), T3 delete-undo ungrouped PASS, T6 load sanity PASS. T4 delete-undo GROUPED → the group-level
call was found redundant/ineffective (above) and removed. T5 → non-mirror/flip ungroup confirmed position-
neutral (the "unmirrored-unflipped-saved-pass.qet" file); mirror/flip case intentionally reflects position
(that IS the fix). The toolbar-cursor SIGTRAP crash that interrupted the first T2 run is unrelated (own entry).

### macOS/Qt6 toolbar-cursor CGImage crash (SIGTRAP, unrelated to feature work)  · OPEN (known-issue candidate; pre-existing) · branch mirror-flip-rotate
**Crash:** SIGTRAP / EXC_BREAKPOINT (NOT the SIGSEGV/0x24 panel-UAF family). Report
`~/Library/Logs/DiagnosticReports/qelectrotech-2026-06-24-103714.ips`. Backtrace:
`CGImageCreate ← QImage::toCGImage ← QCocoaCursor::createCursorData ← QCocoaCursor::changeCursor ←
QToolBar::event (cursor change) ← mouse event` — CoreGraphics rejects an invalid/null image colorspace
when converting a cursor QImage to CGImage. Fires on mousing over a toolbar; zero frames touch text/
mirror/flip/fromXml/our code. Surfaced during the live-compensation T2 reload (interrupted the test) but
is NOT caused by it. Distinct from the elements-panel UAF (different signal, different stack). Severity:
medium-nuisance — can interrupt any GUI session; not data-related. Next if pursued: identify which QET
toolbar cursor pixmap has an invalid colorspace under Qt6 (likely a tool/drag cursor).

### Delete-button tooltip says "Delete selection" for a button that also ungroups  · OPEN (low-priority GUI mod) · branch mirror-flip-rotate
**Finding (User, 2026-06-24):** the Delete toolbar/UI button tooltip reads "Delete selection", but the same
button also performs the ungroup function — potentially confusing. Consider differentiating the
ungroup vs delete affordance (separate tooltip text, or split the action). Low priority, cosmetic UX.

### Rebase mirror-flip-rotate onto the rebased qt6_cmake_joshua  · DONE (force-pushed 2026-06-23) · branch mirror-flip-rotate
**What:** moved the feature divergence onto the freshly-rebased qt6_cmake_joshua (`d00f3613d`).
`git rebase --onto qt6_cmake_joshua <merge-base 082947266> mirror-flip-rotate`. Of 19 commits, **17
replayed cleanly**; the 2 redundant cherry-picks dropped — `4ed901169` (CMake guard) skipped, already
in base via the `e342213d4` squash; `39b502ce3` (gitignore) auto-dropped "patch contents already
upstream". **ZERO semantic conflicts** despite 10 feature files overlapping upstream's 197-commit
changes (element.cpp/.h, dynamicelementtextitem.cpp, terminal.cpp, elementspanel.cpp,
qetdiagrameditor.cpp/.h, CMakeLists.txt, qet_compilation_vars.cmake, qet_fr.ts) — upstream's edits sat
in different regions. New tip `7ebf84876`; force-pushed `917c01da9..7ebf84876` with `--force-with-lease`,
verified on remote. Rollback = tag `archive-pre-upstream-rebase-mirror-flip-rotate` (`917c01da9`).
**Hash-rewrite map (pre-rebase → current tip), for cross-referencing older entries:** `207d3cc5b`→`1d9b3d34d`,
`0a734cdc5`→`391b888c4`, `023070fd8`→`c72a57fb9`, `feeef3cea`→`6107f5217`, `bd61ca17c`→`a7bff9ea3`,
`6f8dd772c`→`e1d246852`, `3025d380f`→`e724e14ab`, `ad69d989b`→`adbf52abf`, `917c01da9`→`7ebf84876`,
`b0a405329`→`a2e0495fc`. **`a2e0495fc` ≡ `b0a405329` confirmed same-content-different-hash** (NOT two commits):
identical `git patch-id --stable` (`82ef6adb…`); the only `git show` diff is blob-index hashes + `@@` line numbers
(`1082` vs `1073`, upstream shifted surrounding lines). The `b0a405329` grouped-flip fix (`compensateMirrorFlip`
reflection-extent `br.left()+br.right()` / `br.top()+br.bottom()`) is PRESENT and INTACT at HEAD (element.cpp:1145-1147,
extracted into `compensateMirrorFlip` by `c72a57fb9`, math unchanged) — code-level confirmed; behavioural re-test
(grouped flip 0/90/180/270, live==reload, no ~group-height jump) still pending a GUI run.
**Validation:** fresh configure+build clean (0 errors); all 6 feature fingerprints present (mirror/flip
command, `compensateMirrorFlip`, `itemForDiagram`, addProject guard, synchronous RotateTextsCommand,
free-rotation default). User regression `mfr-rebase-regression`: tests 1,2,3,5 PASS; 4 & 6 PARTIAL but
**both empirically confirmed PRE-EXISTING** (reproduce identically on the `917c01da9` pre-rebase build,
rebuilt + retested this session) — NOT rebase regressions → two new OPEN entries below.
**LFS push gotcha (recurring):** a git-lfs `pre-push` hook (`.git/hooks/pre-push`, appeared during the
archive-tag checkout) exits 2 and blocks ALL pushes because `git-lfs` isn't installed. `.gitattributes`
only LFS-tracks `*.qch` (none in tree), so it's spurious. Bypass per-push with `git push --no-verify`,
or delete the hook (the hook's own message says so). The earlier qt6 push pre-dated the hook, hence
worked without `--no-verify`.

### keep_visual_rotation toggle doesn't re-fire readability correction live  · OPEN (pre-existing; missing-trigger class) · branch mirror-flip-rotate
**Symptom:** cycling a text's keep_visual_rotation switch ON→OFF→ON→OFF — on the final OFF the live view
does NOT re-orient the text; it corrects only after a save/reload or a subsequent element-level
mirror/flip/rotate. Text POSITION stays correct throughout; only orientation lags.
**Class — same "missing trigger" family** as the RESOLVED trigger-lag (`023070fd8`/`0a734cdc5`,
post-rebase `c72a57fb9`/`391b888c4`) and its compensate-on-own-rotation gap: `correctReadability` has only
element-level call sites (`applyMirrorFlip`, `onElementRotated`, and the command-driven
`correctTextReadability` in the two rotate commands); nothing is wired to the switch toggle. Scope together
with the create-text gap below (and ISO-layout, FEATURE IDEA #7) as one trigger-coverage pass, not as
separate one-offs.
**Trigger wiring traced (code, 2026-06-23):** the diagram-side toggle routes
`DynamicElementTextModel` (dynamicelementtextmodel.cpp:625) → `QPropertyUndoCommand(deti,
"keepVisualRotation", …)` → `DynamicElementTextItem::setKeepVisualRotation` (dynamicelementtextitem.cpp:1508).
That setter ONLY sets `m_keep_visual_rotation`, emits `keepVisualRotationChanged`, and connects/disconnects
the rotation signals — it calls NO correction. `keepVisualRotationChanged` has ZERO diagram-side readability
listener (its only `connect` is element-EDITOR side, `PartDynamicTextField` at dynamictextfieldeditor.cpp:199 —
a different class). So toggling changes the flag but never recomputes the displayed orientation; the text holds
whatever the last correction left it at until the next `applyMirrorFlip` (any M/F/R) or `fromXml`→`applyMirrorFlip`
(reload) re-fires. Position is untouched because `compensateMirrorFlip`'s per-item transform isn't flag-dependent
and the toggle doesn't disturb it.
**NOT a `207d3cc5b` side-effect (verified, not assumed):** the setter is BYTE-IDENTICAL at `207d3cc5b^` and has
never called `correctReadability`, so the toggle triggered nothing before that commit either. `207d3cc5b`
(post-rebase `1d9b3d34d`) changed only what `correctReadability` DOES in its OFF branch (I→R 180° force → no-op),
i.e. the self-heal DESTINATION once some later trigger clears the lag — NOT WHEN correction fires. ⇒ the
live-update-on-toggle gap is pre-existing and independent of `207d3cc5b`. ("Reproduces on `917c01da9`" only proves
not-a-rebase-regression — `917c01da9` already contains `207d3cc5b`; the setter-identity at `207d3cc5b^` is what
proves it predates the ISO-layout removal.) **Severity:** low — cosmetic orientation lag; self-heals on any
transform/reload; position never wrong.
**UPDATE (2026-06-25) — now also gates the new tri-state selection (was T2 FAIL).** After the
RotationMode migration the path is renamed but unchanged in substance: `DynamicElementTextModel`
→ `QPropertyUndoCommand(deti, "rotationMode", …)` → `DynamicElementTextItem::setRotationMode`, which
still sets the member + emits `rotationModeChanged` + manages connections but calls NO correction;
`rotationModeChanged` has zero diagram-side readability listener. So selecting Upright/ISO/Free in
Text Properties does not re-orient on Accept — it applies only on the next element M/F/R (T2 in the
ISO-layout implementation entry below). Severity bumps from cosmetic to mild usability now that mode
is a user-facing 3-way control. Fix = wire a diagram-side correction on `rotationModeChanged` (or call
`Element::correctTextReadability` from the setter), as part of the one trigger-coverage pass.

### New text on an already-mirrored/flipped element draws reflected until next transform  · RESOLVED (ade229881) · branch mirror-flip-rotate
**RESOLVED 2026-06-24 by the shared live-compensation fix (top entry, ade229881):** `AddElementTextCommand::redo`
now calls `Element::correctTextReadability(m_text)` post-attach, and `removeTextFromGroup` does the same on
ungroup — closing both the create-text manifestation and the geometric-mirror half of the ungroup entry. The
positional (~19px) half of the ungroup case is the separate deferred addToGroup/removeFromGroup issue below.
**Symptom (historical):** create a new dynamic text on an element already mirrored and/or flipped → the text renders
geometrically mirrored/flipped (unreadable) until a subsequent M|F|R or save/reload redraws it correctly.
Glancingly seen earlier during the T5 crash work (debug-logs file named "...subsequent mirror redraws text
with geometric mirror.qet") but never logged as its own defect — now confirmed it is a benign display lag,
distinct from that (since-fixed) panel crash.
**Class:** missing-trigger family — the text-creation path doesn't fire `correctReadability` /
`compensateMirrorFlip`; only element-level events do. Scope with the toggle gap above.
**Shared gap (cross-ref 2026-06-24):** this same `addDynamicTextItem`-misses-`compensateMirrorFlip` hole also
produces the geometric-mirror manifestation of the DEFERRED "Genuine ungroup of a mirrored element…" entry
(ungroup re-attaches each text via `addDynamicTextItem`). One post-attach compensation hook may close both.
**Mechanism traced (code, 2026-06-23) — it's the `compensateMirrorFlip` half, NOT readability:**
`applyMirrorFlip` (element.cpp:1072) applies the reflection as an ELEMENT-level `setTransform(scale(sx,sy))`.
Dynamic texts are direct child items (`deti->setParentItem(this)`, addDynamicTextItem:1259) with no
`ItemIgnoresTransformations`, so they INHERIT that reflection scale; the per-child `compensateMirrorFlip` is what
cancels the inherited reflection (and reflects the position) so each text reads correctly. New-text creation goes
`AddElementTextCommand::redo()` (addelementtextcommand.cpp:72) → `Element::addDynamicTextItem` (element.cpp:1259):
append + `setParentItem` only, calling NEITHER `compensateMirrorFlip` NOR `correctTextReadability`. So a text added
onto an already-reflected element inherits the element's reflection scale with an identity local transform →
draws geometrically mirrored/flipped, until the next `applyMirrorFlip` iterates the now-larger `m_dynamic_text_list`
and compensates it. The VISIBLE defect is purely the uncompensated reflection scale, not orientation (a fresh
`DynamicElementTextItem` defaults keep_visual_rotation=ON at rotation 0, so `correctReadability` is a no-op anyway)
— a `compensateMirrorFlip` gap, cleanly separate from the readability logic.
**Origin = the folio mirror/flip feature itself (NOT `207d3cc5b`):** `998f597d0` introduced the element-scale +
per-child compensation; `addelementtextcommand.cpp` has ZERO commits on the feature range `d00f3613d..HEAD` and
`addDynamicTextItem` was never modified by it — the pre-existing add-text command was never taught about the new
reflection state. `207d3cc5b` touched only `correctReadability`'s OFF orientation branch, unrelated.
**Confirmed pre-existing:** reproduces identically on `917c01da9` (pre-rebase; correct re the rebase/UAF work — the
deeper origin is the feature-design coverage gap above). **Severity:** low-medium — new text temporarily
unreadable; corrects on any transform/reload, no data corruption.

### Rebase qt6_cmake_joshua onto current upstream/master (squash + force-push)  · DONE (force-pushed 2026-06-23) · branch qt6_cmake_joshua
**What:** brought the fork's Qt6 port branch current with upstream. Squashed the 27-commit
divergence (since merge-base `b1466ec649`) into ONE commit, rebased onto `upstream/master`
(`d3cf8f263`). New tip `d00f3613d` "Qt6/CMake port and macOS build fixes" — 1 ahead / 0 behind
upstream, linear (parent = upstream tip). Force-pushed with `--force-with-lease`
(`b537842ac…d00f3613d`); verified on remote via `git ls-remote`. Scope was qt6_cmake_joshua ONLY —
mirror-flip-rotate / master / folio-mirror-flip / fix-grouped-text-rotation-pivot untouched.
**Safety net:** archive tags pushed to origin BEFORE any rewrite —
`archive-pre-upstream-rebase-qt6_cmake_joshua` (`b537842ac`) and
`archive-pre-upstream-rebase-mirror-flip-rotate` (`917c01da9`). The qt6 tag is the rollback.
**Method:** dry run first on throwaway `dryrun-qt6-joshua-squash` (never pushed; left for reference),
to characterize conflicts before touching the real branch.

**Conflict reality — far cleaner than feared.** 27 fork commits vs **197** upstream commits since
merge-base, yet only ONE file had git conflicts: `CMakeLists.txt` (4 regions), all on the
**Qt5↔Qt6 build-system boundary** — **upstream/master is still a Qt5 codebase** (`find_package(NAMES
Qt5)`, `qt5_*_translation`, `KF5_*`, plain `add_executable`, includes `hoto_update_cmake_message.cmake`).
Resolved by taking the fork's Qt6 constructs; took upstream's version bump `0.100.1`; dropped the
`hoto_update` include (the Qt6 branch author `1ba97c7e9` deliberately deleted that file — verified no
other refs). So this rebase carries the Qt6 port FORWARD over an evolving Qt5 master — the
CMakeLists.txt conflict will RECUR on every future upstream build-system touch.

**Two SILENT auto-merge artifacts (no conflict markers, but build-breaking) in
`cmake/qet_compilation_vars.cmake`** — the dangerous part, since they don't show as conflicts:
upstream added `Qt::GuiPrivate` (its new PDF-hyperlink feature, `QPdfEngine::drawHyperlink`) which
needs `GuiPrivate` in the Qt6 `find_package` COMPONENTS (added); and `qetsvg.cpp/h` got listed twice
(upstream add + fork's existing → de-duplicated). LESSON: new upstream features using
Qt-version-sensitive APIs need small Qt6-porting touch-ups at each integration, and they surface as
silent semantic breakage, not git conflicts — grep the configure/build for them, don't trust
"no conflict markers."

**Re-validation (smoke-test scope, not the rotation matrices — those live on mirror-flip-rotate, NOT
here):** fresh `cmake ..` + `make -j8` clean (exit 0, 0 errors, 59 normal Qt6-deprec warnings, binary
mtime 15:00). User smoke test `qt6-rebase-smoke` PASS: launch, symbol library populated, English UI,
new-diagram → drop element → edit text → save `qt6-rebase-smoke.qet` → reload, titleblock shown.
**main.cpp discipline held:** the app-name workaround was stashed across the whole op and never
entered the squash (verified committed main.cpp has no `QElectroTech-Qt6` string); restored unstaged
afterward for the runtime binary.

**Follow-ups (not done here):** (1) upstream PR-prep is a SEPARATE task — the squash bundles fork-only
artifacts (knowledge-graph JSON, CLAUDE.md/CC-TASKS.md gitignore housekeeping) that must be stripped
before any upstream submission. (2) Rebasing mirror-flip-rotate onto the new qt6_cmake_joshua is the
explicitly-deferred next task. (3) throwaway `dryrun-qt6-joshua-squash` branch still exists locally —
delete when no longer wanted for reference.

### Make Dialog-rotate (RotateTextsCommand) fully synchronous — drop the animation  · COMMITTED (3025d380f, pushed) · branch mirror-flip-rotate
**Landed 2026-06-22 as `3025d380f` "Rotate texts via the orientation dialog synchronously"** (push
`39b502ce3..3025d380f`, fast-forward). Two named files only (rotatetextscommand.cpp + .h); main.cpp
never staged. Post-commit `cmake ..` reconfigure (GitRevision refresh) + `make -j8` clean; dev paths
verified relative-to-binary. T1–T4,T6 + T5-drift PASS; T5's crash is the SEPARATE pre-existing
elements-panel UAF (own OPEN entry below), not this change. No fix-metrics entry (requirement removed
from CLAUDE.md this session; metrics never went in commit messages anyway).
**Decision (User, 2026-06-22):** the "Choose text orientation" dialog path should be
**instant**, exactly like the R90° quick-rotate (`RotateSelectionCommand`) — NOT animated.
This **supersedes** the earlier arc-interpolator investigation idea (reshape the tween curve);
that plan is dropped. Founding observation: text visibly deviated off-path during the dialog
animation (both forward and undo), settling correctly only at the endpoints. Root cause was
confirmed analytically (no instrumentation needed): two independent linearly-interpolated
`QPropertyAnimation`s ("rotation" + "pos") in one `QParallelAnimationGroup` — a straight-line
`pos` interpolation can't trace the circular arc that true center-pivot rotation needs, so only
the two endpoints are geometrically exact (worst at 180°: full half-bbox swing mid-tween).
R90° never showed it because its directly-selected text/group branches are already synchronous
(`QPropertyUndoCommand`, "stay synchronous" note at rotateselectioncommand.cpp:101).

**Implemented (NOT yet committed, NOT yet tested — User rebooting first):**
- `sources/undocommand/rotatetextscommand.{h,cpp}`. Removed the `QParallelAnimationGroup`, both
  `QPropertyAnimation`s, `setupAnimation()`, the `finished()`→correction connect, and the
  `m_anim_group` member — **fully deleted, no dead code/comment-out** (recoverable via git).
- Constructor now creates, per text and per group, two **synchronous** `QPropertyUndoCommand`
  children parented to `this` ("rotation" cur→`m_rotation`; "pos" cur→`centerPivotEndPos(item,
  m_rotation)`). `QPropertyUndoCommand` defaults `m_animate=false` → synchronous, so no
  `setAnimated` call needed. `centerPivotEndPos` / endpoint math **unchanged** — just called
  synchronously now. Multi-selection preserved (both loops; all children atomic under one command).
- `redo()`/`undo()` now call `QUndoCommand::redo()/undo()` (drive children) → conductor-text
  `forceMovedByUser` bookkeeping (unchanged) → `correctSelectedTexts()`. Mirrors
  `RotateSelectionCommand`'s exact placement: correction runs on BOTH redo and undo, reading the
  settled rotation (undo MUST re-run it — `compensateMirrorFlip` sets an absolute transform the
  pos/rotation reversal doesn't restore).
- `finalizeReadability()` → renamed `correctSelectedTexts()` (old "animation has settled" wording
  was untrue-to-tree).
- **NEW guard `if(isObsolete()) return;`** after `openDialog()`. Necessary because geometry now
  applies synchronously in `redo()`: a cancelled dialog must create no child commands, else cancel
  could apply an instant jump-to-`m_rotation` (and, since the command is then discarded as obsolete,
  leave it un-undoable). `QUndoStack::push` semantics on a pre-obsolete command couldn't be verified
  from headers (no .cpp shipped with Homebrew Qt), so the guard makes cancel a guaranteed no-op
  either way. This also closes a latent cancel-path issue the animated version may have had.

**Verified so far:** clean `make -j8`, no warnings from the file; binary relinked (mtime 18:39,
2026-06-22). `grep` confirms zero animation primitives remain in either file. Diff stat: cpp
+39/-47-ish, h small. clangd panel showed the usual QUndoCommand-not-found cascade (no
compile_commands.json) — ignored per CLAUDE.md, build log is authoritative.

**PENDING — functional test matrix proposed, User to run AFTER reboot, then report PASS/FAIL.**
Do NOT commit until it passes. Named files only (`rotatetextscommand.cpp` + `.h`); never main.cpp.
Test items (❌ = old animated behaviour):
- **T1 Core:** dialog-rotate 90° → instant snap, no tween/arc, identical feel to R90° (❌ ~300ms
  eased sweep that visibly deviated off-path).
- **T2 Undo/redo:** undo → exact original orientation+position+**transform**; redo restores exactly.
- **T3 Multi-select:** several texts + a group rotated together → all pivot about own bbox center,
  one atomic undo/redo for the whole set.
- **T4 ON-MVR:** keep_visual_rotation ON → ends tops-up (correction applied ONCE), correct position,
  undo exact; repeat under element mirror / flip / mirror+flip, grouped + ungrouped → settles
  immediately incl. reflected position, no double-apply/ordering surprise now both steps sync.
- **T5 Phenomenon-B regression:** dialog-rotate 90° then element-rotate ≥2 full laps → no drift,
  returns to lap-start; save/reload live==reload.
- **T6 Cancel (new guard):** open dialog on a rotated text → Cancel → text unchanged, no jump, no
  new undo-stack entry.

**Next on PASS:** propose commit message + staged file list (the two named files), wait for explicit
approval, commit, then `cmake ..` + `make -j8` (refresh GitRevision), push origin mirror-flip-rotate.
On FAIL: fix before committing.

#### TEST RESULT (2026-06-22): T1–T4 + T6 PASS, T5 PARTIAL (save/reload CRASH — separate bug)
T1 (instant, no animation), T2 (undo/redo exact), T3 (multi-select atomic), T4 (ON-MVR no
double-apply), T6 (cancel = no-op, new guard works) all **PASS**. T5: the no-drift / returns-to-
lap-start part **passed**; but **saving then reloading a mirror+flip element CRASHES QET on open**.
Reproduced twice (a save, and a byte-identical-start file saved-as). Test files in debug-logs/:
`mirror-flip-rotate-test-00.qet` (original, orientation=0, no mirror/flip — loads fine),
`-02-01.qet` and `-02-02 - FAIL - qet crashed on opening.qet` (both mirror+flip, crash on open),
`-01-01 - FAIL - qet crashed after subsequent mirror redraws text with geometric mirror.qet`
(a LIVE crash during a mirror op, flip + orientation=2).

**Data comparison done (xmllint --format + diff, files copied to /tmp; measure-don't-assume):**
- Crash is **NOT malformed data.** vs original, the only diffs in a crashing file are: the placed
  `<element>` gains valid `flip="true" mirror="true"` (or `flip="true" orientation="2"`), and the
  live `<dynamic_elmt_text>` rotations/positions change to well-formed values (e.g. rotation 0→180,
  x 39→90.1875). No NaN/inf/garbage. So the crash is in **load/redraw LOGIC for mirror+flip, not bad
  serialized values.** (`dynamic_elmt_text` = live instance text; `dynamic_text` = embedded-definition
  template — the latter is byte-identical across all 5 files, ruling out the definition.)
- **Common signature across ALL three crash files:** placed element with `mirror`/`flip` set + exactly
  one `<texts_group>` + at least one rotated text. Same fragile family as RESOLVED 05bcba506
  ("grouped rotated text: mirror corrupts on save/reload") and DEFERRED "Genuine ungroup of a mirrored
  element displaces text". 02-01 (rot270) and 02-02 (rot180) have IDENTICAL element tags; both crash.

**Assessment — almost certainly PRE-EXISTING, NOT caused by the synchronous conversion (verify before
relying on it):** the change touches only `RotateTextsCommand` (live dialog rotate). The crash is on
LOAD (`fromXml`→`applyMirrorFlip`→`compensateMirrorFlip`, all UNCHANGED) of a file whose saved values
are the deterministic result of UNCHANGED correction logic with identical endpoints — so a pre-change
binary would save the same bytes and crash the same way. The sync change just surfaced it via the T5
sequence. ⇒ The RotateTextsCommand conversion (T1–T4,T6 pass, T5-drift passes) is sound on its own
merits; the load crash is a SEPARATE bug to track/fix independently. BUT confirm with a backtrace first.

**ROOT CAUSE FOUND (2026-06-22, from macOS crash reports — DEFINITIVE, not hypothesised).** The earlier
"mirror+flip load-logic" hypothesis was WRONG; the mirror/flip correlation was coincidental (just the
test files). macOS had already written `.ips` crash reports for BOTH observed crashes
(`~/Library/Logs/DiagnosticReports/qelectrotech-2026-06-22-{210628,185251}.ips` = the 21:06 reload crash
and the 18:52 "live mirror" crash). BOTH have the **identical** backtrace, SIGSEGV / EXC_BAD_ACCESS
(fault addr 0x24 — small-offset null/freed deref), and **zero** RotateTextsCommand / mirror / flip / text
frames:
`QTreeModel::index ← QTreeModel::parent ← QHeaderView::currentChanged ← QItemSelectionModel::setCurrentIndex
← QTreeWidget::setCurrentItem ← QETDiagramEditor::addProjectView()::$_1 lambda (qetdiagrameditor.cpp:1904)
← syncElementsPanel ← subWindowActivated ← MDI window activation`.
Line 1904 = `pa->elementsPanel().setCurrentItem(item)` in the `syncElementsPanel` lambda; `pa`+`item` are
null-guarded, so the crash is INSIDE Qt — `getItemForDiagram` returned a **stale/dangling QTreeWidgetItem\***
(elements-panel tree rebuilt by a collection reload — startup log shows "Elements collection reload"), and
`setCurrentItem` derefs freed memory. **Use-after-free in the elements-panel tree sync, intermittent /
timing-dependent** (race between panel reload and project-view activation) — which is why it did NOT
reproduce under lldb (timing perturbed) nor on a clean single-file reboot (collection already settled).
`git blame` → both sync lambdas are from upstream commit **a82f6de23** "Add highlight current page in
ProjectView", an ancestor of `upstream/qt6_cmake_joshua` ⇒ **PRE-EXISTING UPSTREAM bug**, not this branch.

**VERDICT: the synchronous RotateTextsCommand conversion is SOUND and unrelated to the crash.** T1–T4,T6
PASS, T5 drift PASS; T5's "crash" is this separate pre-existing panel UAF. Test gate satisfied. Diagnostic
method that worked: got the actual backtrace (macOS .ips reports persist across reboot — no debugger needed
for an unattached crash) instead of guessing from the file content. The unattached fresh-reload retest was
rendered UNNECESSARY by the existing reports. CONVERSION CLEARED TO COMMIT (named files only:
rotatetextscommand.cpp + .h; main.cpp never staged).

### Elements-panel crash highlighting current page (orphan-item, NOT use-after-free)  · RESOLVED (ad69d989b, pushed) · branch mirror-flip-rotate
**Area:** UI / elements panel · **Branch:** fix lands on mirror-flip-rotate (pre-existing upstream bug, a82f6de23
"Add highlight current page in ProjectView", ancestor of qt6_cmake_joshua; cherry-pick to a clean upstream PR
branch later). **Plan file:** `~/.claude/plans/review-claude-md-cc-tasks-md-ancient-globe.md`.
**Crash:** intermittent SIGSEGV (EXC_BAD_ACCESS, fault 0x24) at `qetdiagrameditor.cpp:1904`,
`setCurrentItem(item)` in the `syncElementsPanel` lambda (sibling `diagramActivated` lambda at :1892 same
shape). Both 2026-06-22 crash reports share stack `QTreeModel::index ← parent ← setCurrentItem ← lambda`.

**ROOT CAUSE — refined by reading the code (supersedes the earlier "use-after-free" framing; the dangling-
pointer hypothesis was from the backtrace alone and is WRONG).** Not freed memory: `GenericPanel::deleteItem`
calls `unregisterItem` BEFORE `delete` (genericpanel.cpp:907), so `diagrams_` holds no stale entries, and no
free-without-unregister path exists. Actual mechanism: `getItemForDiagram` (genericpanel.cpp:315) is a
GET-OR-CREATE helper — on a cache miss it calls `makeItem(QET::Diagram)` → `new QTreeWidgetItem(nullptr,…)`
(genericpanel.cpp:325,882), a DETACHED ORPHAN with no parent, not in the tree model. The highlight lambdas use
it as a found-or-null lookup (`if(item) setCurrentItem(item)`); the orphan is non-null, so `setCurrentItem` on
an item the `QTreeModel` doesn't own derefs invalid memory in `QTreeModel::index()/parent()`. Intermittent
because it only fires when the panel hasn't yet populated that diagram's item at activation time (startup /
collection-reload window) — explains why it didn't repro under lldb or on a settled reboot.

**FIX — committed as ad69d989b (2026-06-23).** Added pure-lookup `GenericPanel::itemForDiagram(Diagram*)`
(mirrors `itemForProject` exactly: `if(!diagram) return nullptr; return diagrams_.value(diagram,nullptr);`) in
genericpanel.h/.cpp (decl+def after getItemForDiagram), and repointed both highlight handlers
(qetdiagrameditor.cpp:1888 diagramActivated, :1901 syncElementsPanel) `getItemForDiagram(`→`itemForDiagram(`;
their existing `if(item)` guard now skips on a cache miss instead of fabricating an orphan. Staged named files
only: genericpanel.h, genericpanel.cpp, qetdiagrameditor.cpp (never main.cpp).

**TEST — PASS (2026-06-23, Trigger 1 only; log `debug-logs/famA-retest-trigger1-livemirror.log`).** Validated
the live-mirror + MDI/folio-activation trigger (the 18:52 "live mirror" crash) on an already-open project in the
fresh Qt6 build (binary mtime 23:49, the one that still crashed at the SEPARATE addProject site — see below):
live mirror op + repeated folio-tab / MDI-subwindow switching, plus extra coverage (multiple opens, save-as,
button-Open re-open) → no crash; current page highlights when its item exists. Clean GUI quit; process exit +
crash-free log confirmed. The 21:06 "reload" trigger was NOT separately exercised — deliberately, per the
no-File-Open boundary set this session to avoid re-hitting the still-open Family-B addProject crash during a
Family-A validation; it is code-level covered by the identical two-handler fix (same lambdas, same pure lookup).

**NOTE — a SEPARATE, still-open crash survives this fix:** the project-open `setCurrentItem(child(0))` crash in
`ElementsPanel::addProject` (elementspanel.cpp:166) — same origin commit a82f6de23, same backtrace signature,
but a DIFFERENT mechanism (a real spliced child, not a fabricated getItemForDiagram orphan). Tracked as its own
ACTIVE entry below ("Elements-panel crash on project open"). The 7 latent elementspanelwidget.cpp F3–F9 sites
(the old "commit 2" — same orphan mechanism as this fix, preventive) are split into their own ACTIVE entry below.
Both are kept as SEPARATE entries by decision; shared origin commit is not a reason to merge their tracking.
**Severity:** medium — data-safe (.qet reloads fine once past it) but a hard crash.

### Elements-panel crash on project open — setCurrentItem(child(0)) in ElementsPanel::addProject  · DEFENSIVE GUARD COMMITTED (917c01da9, pushed); underlying race UNCONFIRMED, investigation BANKED · branch mirror-flip-rotate
**Area:** UI / elements panel · **Priority: HIGHER than the (now-RESOLVED) activation crash above** — triggered by
File ▸ Open, a core/frequent action, vs the narrower project-view-activation trigger. **Pre-existing upstream**
(origin commit a82f6de23 "Add highlight current page in ProjectView", same commit as the activation crash, but a
DISTINCT defect family — do NOT fold the two together).
**Crash:** SIGSEGV (EXC_BAD_ACCESS, KERN_INVALID_ADDRESS at 0x24), report
`~/Library/Logs/DiagnosticReports/qelectrotech-2026-06-22-235807.ips`. Same backtrace signature as the activation
crash (`QTreeModel::index ← parent ← QItemSelectionModel::setCurrentIndex ← QTreeWidget::setCurrentItem`) but the
caller is `ElementsPanel::addProject` elementspanel.cpp:166 `setCurrentItem(qtwi_project->child(0))` ←
`projectWasOpened`:416 ← `QETDiagramEditor::addProject`:1209 ← `openProject`:1043 (File ▸ Open).
**Proves it is a SEPARATE site, not the activation crash:** the .ips launched 23:51:32 against the build_qt6
binary that ALREADY carried the activation fix (itemForDiagram, mtime 23:49), and still crashed at 23:58:02. Qt6
dev build (procPath `/Users/USER/*/qelectrotech`, parent zsh/Terminal, Qt 6.11.1) — not the installed Qt5 release.
**Mechanism — genuinely UNKNOWN; distinct from the activation crash.** Here `child(0)` is a REAL child of the
freshly-built project subtree (built detached via `makeItem`, then spliced into the model at elementspanel.cpp:158
via `invisibleRootItem()->insertChild()`), NOT a fabricated `getItemForDiagram` orphan — so the itemForDiagram fix
neither covers nor could cover it. Candidate hypotheses (to TEST, not fix):
  - (a) `child(0)` is null for a 0-diagram project → `setCurrentItem(nullptr)` on a path that still derefs.
  - (b) the manual `invisibleRootItem()->insertChild()` splice (line 158) of a pre-built DETACHED subtree leaves
    child(0) without valid model linkage (treeWidget()/parent inconsistent) when setCurrentItem walks it.
  - (c) reload path (`first_reload_`) re-inserts an already-parented project item (getItemForProject returns the
    registered existing item, then it is insertChild'd again).
  - (d) SingleApplication second-launch forwarding (User recollection): repeated in-app File▸Open ▸ select ▸ Open
    BUTTON never crashed; the one observed crash coincided with a DOUBLE-CLICK in File▸Open while the build was
    already running. Candidate: a double-click second-launch is IPC-forwarded into the running instance rather than
    handled by its normal in-app menu action, and that forwarded path's timing/threading exposes child(0). Same
    SingleApplication seam CLAUDE.md documents as the reason for main.cpp's app-name workaround — not a new concern.
**INVESTIGATION (instrumented 2026-06-23; logs `debug-logs/famB-confirm-{lateopen,earlyopen,doubleclick}.log`).**
A temporary `qDebug` `[FAMB]` dump was placed in addProject's first_add branch immediately before
`setCurrentItem(qtwi_project->child(0))`, logging common_tbt_collection_item_, indexOfTopLevelItem, qtwi_project,
childCount, qtwi_project->treeWidget(), child(0). Three user-driven runs, ~a dozen opens total (startup arg-open,
button-Open, and the original double-click-while-running-with-files-already-open action, incl. close/reopen with
address reuse). **CRASH NOT REPRODUCED in any run.** Instrumentation reverted (tree clean), binary rebuilt clean.

**KEY FINDINGS (measured, not hypothesised):**
- **The detached state is ROUTINE and HARMLESS — refutes hypothesis (b) / the earlier "uninitialized
  common_tbt_collection_item_ → detached → crash" theory.** Every open BEFORE the panel's first reload() logs
  `common_tbt_item=0x0, indexOfTopLevelItem=-1, qtwi_treeWidget=0x0` — so `insertChild(-1)` no-ops and qtwi_project
  is genuinely detached from the model — and `setCurrentItem(child0)` on it did NOT crash. A detached item resolves
  to row -1 gracefully; **detached ≠ crash.**
- **reload() self-heals it:** once the 250 ms `firstActivated→reload()` timer fires, reload() re-adds every project
  in projects_to_display_ with a now-valid common_tbt index, re-attaching the SAME qtwi_project pointers (observed:
  the first two opens, detached at 11:51, reappear attached at valid indices 1,2 at 11:52).
- **(a) ruled out** — child(0) non-null in every logged open (childCount 2–4). **(c)/(d) exercised without crash** —
  close/reopen (same project ptr, new qtwi, childCount 2→4) and the double-click-while-running action both ran clean.
- **The .ips fault (deref at offset 0x24) is a FREED/garbage QTreeWidgetItem, not a merely-detached one** (detached
  is now proven graceful). ⇒ refined leading hypothesis: a **dangling qtwi_project (or child) in a narrow lifetime
  race** — e.g. the 250 ms reload() timer firing across a setCurrentItem, or a specific close+open interleaving —
  not hittable on demand. Back toward a use-after-free, but of a freed tree item, not the uninitialized-pointer path.

**STATUS: BANKED** (investigation paused, not reproduced; chasing further has diminishing returns). Resume with the
freed-pointer race in mind: instrument `currentItem()` / old-current validity and target a close-all-then-open sequence.

**FIX — defensive guard COMMITTED 917c01da9 (2026-06-23, pushed).** elementspanel.cpp addProject first_add branch:
the auto-select (`setCurrentItem(qtwi_project->child(0))` + `child(0)->setSelected(true)`) now runs only when
`qtwi_project->treeWidget() != nullptr` (item actually attached to the model). When it skips, reload() re-adds the
project attached and performs the selection, so the settled UI is unchanged (verified: startup arg-open, button-Open,
double-click-while-running all PASS, no regression). This makes the unsafe setCurrentItem-on-not-in-model call
impossible at this site. NOTE it is DEFENSIVE: the detached state alone was proven NOT to crash, so the guard does
not "fix the race" — it removes the unsafe call that the rare freed-pointer race would have detonated through. The
guard's protective effect on the actual crash is LATENT (can't trigger on demand) — absence-confirmed, not
demonstrated. Underlying race remains unconfirmed; investigation stays BANKED (resume notes above).
**Severity:** high — File ▸ Open is a core action; intermittent hard crash on project open (rare; not reproducible on demand as of 2026-06-23).

### Elements-panel: 7 F3–F9 move sites still use getItemForDiagram get-or-create (preventive hardening)  · ACTIVE — not started · branch mirror-flip-rotate
**Area:** UI / elements panel · **Family:** SAME mechanism as the RESOLVED activation crash above
(getItemForDiagram fabricates a detached orphan on a cache miss) — but PREVENTIVE: no observed crash at these
sites. Keep tracked SEPARATELY from the project-open (child(0)) crash above — different mechanism; the shared
origin commit a82f6de23 is not a reason to merge tracking.
**What:** repoint the 7 `setSelectedItem(getItemForDiagram(selected_diagram))` call sites in
elementspanelwidget.cpp:489–530 (the F3–F9 page-move ops: up, down, top, +x10, +x100, up x10, up x100) to the
pure-lookup `itemForDiagram` added in ad69d989b. **Verified safe:** `setSelectedItem` body is just
`m_selected_item = selectedItem;` (stores the pointer, no deref → a null arg is fine); all 7 are guarded by
`if (Diagram *selected_diagram = elements_panel->selectedDiagram())`, and `selectedDiagram()` returns an
already-displayed diagram, so they never hit the orphan path today — hence no crash report. The swap removes the
orphan-fabrication CAPABILITY; a genuine cache-miss would then yield no selection instead of a crash.
**Commit when done:** message must say PREVENTIVE (not "fixes an observed crash"). Staged file:
`sources/elementspanelwidget.cpp` only (never main.cpp).
**Test (same rigor as the crash-site fix):** with a diagram selected in the elements panel, exercise
F3/F4/F5/F6/F7/F8/F9 → each move works, no crash, selection/highlight correct.
**Severity:** low — latent/defensive; no user-visible change in normal use.

### Sync master's .gitignore to canonical fork content (resolves Findings 1 & 2 below)  · DONE (pushed) · branch master
**What:** deliberate decision (not a conflict-driven default) to bring master's `.gitignore` to
byte-exact parity with `qt6_cmake_joshua` HEAD (the b537842 14-line gold standard). Commit
`3f26d2cc0` on master: fixed the fork-only concatenated `!doc/doc-utilsbuild/` line back to the
canonical `!doc/doc-utils` (real upstream has these as two clean separate lines — fork-only defect,
not attributable upstream), and added the three entries master still lacked: `build_qt6/`,
`.DS_Store`, `CLAUDE.md`. Resolves Finding 1 (parity) and Finding 2 (concatenated line) below.
**Verified via fresh, explicitly-scoped clones — NOT a fetch into an existing clone** (an ambiguous-
refspec fetch gave a FALSE "still missing" reading earlier this session, caught before acting):
one `git clone --branch <br> --single-branch --depth 1` per branch, read the file, diff vs the
qt6_cmake_joshua gold reference. master (tip `3f26d2c`), mirror-flip-rotate (`39b502c`),
folio-mirror-flip (`f32706b`) ALL byte-identical to gold (14 lines). fix-grouped-text-rotation-pivot
left out of scope, untouched. main.cpp build-enabler stashed across the switch and restored unstaged.

### Propagate two build/housekeeping fixes across fork branches  · DONE (pushed) · branches mirror-flip-rotate, master, folio-mirror-flip
**What:** replicated two already-decided+verified commits from `qt6_cmake_joshua` onto the
other live branches — `e342213d4` (CMake: guard Linux-only install rules in
`if(UNIX AND NOT APPLE)`, Laurent's fix from real upstream) and `b537842ac`
(.gitignore: exclude CC-TASKS.md + debug-logs/). Mechanical replication, not new judgment.
**Per-branch outcome (all verified against the actual remote after push — hash + file content,
not just local "push succeeded"):**
- **mirror-flip-rotate** → fixed+pushed, tip `39b502ce3` (`4ed901169` CMake, `39b502ce3` gitignore).
  Both cherry-picks applied CLEANLY — branch's pre-images were byte-identical to the commits' parents.
- **master** → fixed+pushed, tip `a0bb8a7ae` (`a2ffddb31` CMake, `a0bb8a7ae` gitignore). CMake
  cherry-pick CONFLICTED (master's install block lacked the `if(NOT QMFILES_AS_RESOURCE)` wrapper +
  had extra blank lines); resolved by taking the incoming canonical guarded block verbatim. gitignore
  cherry-pick also CONFLICTED (see Finding 1) → resolved per user decision to add ONLY the two intended
  lines. gitignore commit message AMENDED on master: original body claimed "CLAUDE.md was already
  excluded", which is false on master (CLAUDE.md is untracked + not gitignored here) — reworded to be
  accurate to master's tree per the "message must match the tree at that commit" rule.
- **folio-mirror-flip** → fixed+pushed, tip `f32706b64` (`fa2caae7c` CMake, `f32706b64` gitignore).
  Both applied CLEANLY; original b537842 message kept verbatim (accurate here — this branch's
  .gitignore already had CLAUDE.md).
- **fix-grouped-text-rotation-pivot** → SKIPPED (retired/superseded branch, not actively developed).
  Confirmed state: neither fix present; CMakeLists.txt in the ORIGINAL UNGUARDED form (the
  qelectrotech.xml / appdata.xml install lines are LIVE+uncommented, not the commented-out workaround
  the other branches carried); .gitignore has neither CC-TASKS.md nor debug-logs/. Left untouched.
**main.cpp discipline held throughout:** stashed the local app-name build-enabler before branch
switches, restored it unstaged onto qt6_cmake_joshua afterward (`git restore --source=stash@{0}`,
never `git checkout <ref> --` which auto-stages); main.cpp never appeared staged on any branch.

#### Finding 1 — master's .gitignore diverges structurally from the qt6 lineage  · RESOLVED (3f26d2cc0)
master's `.gitignore` does NOT contain `CLAUDE.md`, `.DS_Store`, or `build_qt6/` — entries the
qt6_cmake_joshua lineage (which b537842 was based on) does have. That's WHY b537842 conflicted on
master rather than applying clean. Per decision, this pass added ONLY `debug-logs/` + `CC-TASKS.md`
to master and did NOT import the other three — whether master SHOULD also gitignore
CLAUDE.md/.DS_Store/build_qt6 is a separate, undecided question, not to be resolved implicitly as a
side-effect of this propagation. Parked for a deliberate decision.
→ Resolved by `3f26d2cc0`: that deliberate decision was subsequently made — master synced to full
byte-exact gold-standard parity (CLAUDE.md/.DS_Store/build_qt6 added). See top entry.

#### Finding 2 — broken concatenated line in master's .gitignore: `!doc/doc-utilsbuild/`  · RESOLVED (3f26d2cc0)
master's `.gitignore` line 9 reads `!doc/doc-utilsbuild/` — looks like a pre-existing defect where
`!doc/doc-utils` and a `build/` (or similar) entry got mashed into one line. Pre-existing on master,
found incidentally during this propagation pass; NOT introduced here and deliberately left untouched
to keep the gitignore commit narrowly about the two intended lines. Needs its own scoped fix later
(decide intended split: almost certainly `!doc/doc-utils` on its own line, plus whatever the trailing
`build/` was meant to be).
→ Resolved by `3f26d2cc0`: line corrected to canonical `!doc/doc-utils`, with `build_qt6/` restored
as its own separate line (the gold-standard split). See top entry.

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

### ISO-layout tri-state — persistence/encoding scoping  · SCOPING COMPLETE (no implementation) · RECOMMENDATION RECORDED · branch mirror-flip-rotate
**Question:** if `keep_visual_rotation` becomes a genuine tri-state (MVR-ON / ISO-layout /
free-rotation=current OFF; UI exposure = FEATURE IDEA #7), how is the third state PERSISTED in
the `.qet`/`.elmt` XML without breaking file readability by a pre-ISO-layout build?

**What an ABSENT `keep_visual_rotation` attribute means today, per origin (each confirmed
independently against the live tree, not assumed):**
- Legacy `<input>` text (`Element::parseInput`, element.cpp:555-598): the `<input>` schema never
  had this attribute. Converted via `new DynamicElementTextItem(this)`; `parseInput` never calls
  `setKeepVisualRotation` → inherits the in-class default `m_keep_visual_rotation = true`
  (dynamicelementtextitem.h:174) → **ON**.
- Folio `<dynamic_elmt_text>` (`DynamicElementTextItem::fromXml:165`): `attribute(...,"true")` →
  absent → **ON** (constructor default also true, consistent).
- Element-editor `<dynamic_text>` (`PartDynamicTextField::fromXml:213`): absent → **ON at load**.
  BUT a freshly-created, not-yet-loaded part defaults `m_keep_visual_rotation = false`
  (partdynamictextfield.h:132) → **OFF**. This creation-default (OFF) and the load-from-absent
  default (ON) genuinely DISAGREE — the "opposite default" — but only the load default governs
  file interpretation, and a new part is written with the attribute EXPLICIT, so its OFF never
  reaches disk as absence.
- Unifying fact: BOTH save paths write the attribute UNCONDITIONALLY today
  (dynamicelementtextitem.cpp:100, partdynamictextfield.cpp:149). So no current build ever
  PRODUCES absence. Absence in any real file comes only from (a) legacy `<input>`, or (b)
  pre-attribute files (the 0.7-dev retrocompat boundary, dynamicelementtextitem.cpp:173) — all of
  which resolve to ON. Effective per-file answer today: "absent ⇒ ON."

**Mechanism A — overload absence (rejected as the carrier).** Candidate: ISO-layout selection
DELETES the attribute; MVR/OFF write it explicitly. Collision: a file with the attribute absent
because the user chose ISO is byte-identical to a legacy-absent file (→ ON). Disambiguation is
POSSIBLE but only via the existing project version gate: the `.qet` root carries
`version="<currentVersion>"`, written unconditionally on every save (qetproject.cpp:937 →
`QetVersion::toXmlAttribute`, currentVersion()=0.100.1), read into `m_project_qet_version`
(qetproject.cpp:1381), and ALREADY used as a behavioural reinterpretation gate
(`if (m_project_qet_version <= QetVersion::versionZeroDotSix())`, qetproject.cpp:1411). So
"absence means ISO iff version ≥ ISO-release" maps onto the established
`m_project_qet_version < QetVersion::someVersion()` idiom — VIABLE, no new marker needed — BUT
contingent on a DELIBERATE `currentVersion()` bump at the ISO release (currentVersion is one
global bumped per release, not per commit: pre-ISO and ISO dev builds stamp the SAME number, so
the gate is clean only at a release boundary, muddy for dev-build files). Also note `.elmt`
definitions DO write the same version attr (elementscene.cpp:452) but it is WRITE-ONLY — nothing
reads element-definition version for any gate (only qetproject reads `fromXmlAttribute`), so an
`.elmt`-level gate would need a reader added with no precedent.

**Mechanism B — single tri-valued attribute · ACCEPTED RECOMMENDATION.**
Encode the state directly in one attribute: `keep_visual_rotation="true" | "false" | "iso"`.
- Why it removes the version-bump requirement: it does NOT overload an existing signal. Absence
  keeps its legacy meaning (→ ON) untouched; ISO is signalled only by the NEW value `"iso"`,
  which legacy files never carry. Nothing to disambiguate ⇒ no version gate, no currentVersion()
  bump needed for this feature.
- Pre-ISO build reads it gracefully (confirmed against code): load reads only NAMED attributes via
  `attribute(name,default)`; `valideXml` (element.cpp:664) checks only required attrs and never
  rejects extras/unknown values; `QDomDocument::setContent` is non-validating. An old build hits
  `attribute("keep_visual_rotation","true") == "true"` → `"iso" == "true"` is FALSE → sets
  false → **OFF (free rotation)** — the conservative fallback (a build lacking the ISO actuator
  does nothing rather than reorienting). File stays fully readable.
- Precedent: this IS QET's standard additive schema-evolution idiom — `keep_visual_rotation`
  itself, and `font` (the dynamicelementtextitem.cpp:173 "added lately" retrocompat comment), were
  all introduced this way. The versionZeroDotSix gate is the EXCEPTION, used only to reinterpret
  EXISTING data — which Mechanism B specifically avoids needing.
- Preferred over a two-flag form (`keep_visual_rotation="false"` + new `iso_layout="true"`):
  functionally equivalent forward-compat (old build ignores the unknown `iso_layout`, reads
  false → OFF; same graceful degrade), but two flags can express the contradictory combination
  `keep="true"`+`iso="true"`, forcing the loader to define a precedence rule. The single tri-value
  attribute cannot represent an undefined state, so no normalization rule is needed.
- THE ONE CAVEAT (true of Mechanism B AND the two-flag form, and of all additive XML evolution):
  round-trip THROUGH a pre-ISO build that RE-SAVES degrades ISO → OFF permanently, because
  `toXml` is constructive (dynamicelementtextitem.cpp:89-149 builds a fresh element, writes only
  known fields — it never preserves unknown attributes/values). Milder than Mechanism A's
  version-cliff: bites only on an actual save by an old build, not on mere reading, and degrades
  to a sensible state.

**What a genuine tri-state then needs (if/when implemented — viable per Mechanism B):**
- XML: one attribute, three string values (above); save writes the value; load maps it. No
  version dependency.
- Data model: `m_keep_visual_rotation` is a plain `bool` in BOTH classes
  (dynamicelementtextitem.h:174, partdynamictextfield.h:132), exposed as
  `Q_PROPERTY(bool keepVisualRotation …)` with a `keepVisualRotationChanged(bool)` signal.
  Tri-state ⇒ enum (e.g. `RotationMode {KeepUpright, IsoLayout, Free}`). Consumers needing update:
  getter/setter + Q_PROPERTY + signal (both classes); toXml/fromXml both classes
  (dynamicelementtextitem.cpp:100/165, partdynamictextfield.cpp:149/213);
  `Element::correctReadability(item,bool)` (element.cpp:1196), which today is only
  `if(keep) rotateAboutOwnCenter(item,-net_raw)` — the tri-state RE-ADDS the ISO actuator (the
  `if(inverted) rotateAboutOwnCenter(item,180)` that 207d3cc5b removed), gated on `IsoLayout`,
  fed by the now-dormant gate-1 `inverted` classifier; the 6 bool call sites
  (element.cpp:1088,1094,1213,1217,1243,1247); the `QPropertyUndoCommand` round-trips
  (dynamicelementtextmodel.cpp:625, dynamictextfieldeditor.cpp:438) which serialize via QVariant —
  an enum needs Q_ENUM/registration or int-packing.
- UI (FEATURE IDEA #7): the right-click 3-way (radio-style) exposure needs a single tri-valued
  accessor (the enum) replacing the current bool checkbox at dynamicelementtextmodel.cpp:350 and
  dynamictextfieldeditor.cpp:142/432; the "grey out rotate" half (#7b) needs to distinguish
  MVR (rotate dead) / ISO (rotate overridden) / Free (rotate live) — which a bool cannot express.

**UI terminology — FINALIZED (decided spec point, no implementation):** applies to the Text
Properties / right-click exposure (FEATURE IDEA #7). The Element-editor (PartDynamicTextField)
side gets the same nomenclature alignment in a LATER pass — out of scope for this decision.
- Right-click menu row label: **"Text Orientation:"** — chosen to read distinctly from any nearby
  numeric rotation-angle control. Sanity check done: the folio Text Properties model already has a
  numeric **"Rotation"** row (dynamicelementtextmodel.cpp:329/793) and the current
  keep_visual_rotation checkbox **"Conserver la rotation visuel"** (line 345); neither uses
  near-identical wording, so "Text Orientation:" is unambiguous against both.
- Three radio options → RotationMode enum:
  - **"Upright"** → `KeepUpright` (the current behaviour; the internal shorthand "MVR" must NEVER
    appear in this or any other user-facing string — same standing nomenclature rule as
    code/commits)
  - **"ISO"** → `IsoLayout`
  - **"Free"** → `Free`
- New lang-file entries required when implemented: "Text Orientation:", "Upright", "ISO", "Free" —
  flag for translation at implementation time; not needed today.

**IMPLEMENTATION LANDED (2026-06-25) — Mechanism B, combobox UI · two commits on mirror-flip-rotate.**
What landed:
- Data model: `m_keep_visual_rotation` bool → shared `DynamicElementTextItem::RotationMode`
  {KeepUpright, IsoLayout, Free} enum (Q_ENUM), used by both DynamicElementTextItem and
  PartDynamicTextField; getter/setter/Q_PROPERTY/signal renamed `rotationMode`; QPropertyUndoCommand
  round-trips via `QVariant::fromValue`. `Element::correctReadability` dropped its bool param and now
  resolves the mode internally (static `rotationModeForItem`) — sidesteps the element.h↔
  dynamicelementtextitem.h include cycle; all 6 call sites simplified.
- Persistence: single tri-valued `keep_visual_rotation="true"|"false"|"iso"`, both classes'
  toXml/fromXml. Absent/"true"→KeepUpright, "false"→Free, "iso"→IsoLayout.
- Actuator: ISO branch re-added to correctReadability (the recovered `207d3cc5b` classifier —
  parity = m_mirror^m_flip, snapped net, inverted→180° about own centre), gated on IsoLayout.
- UI: Text Properties tree row "Text Orientation:" combobox (Upright/ISO/Free), mirroring the
  existing `textFrom` combobox pattern (default setEditorData/setModelData round-trip; read-back maps
  DisplayRole label→mode like the `src_txt_row` textFrom case). Radios deferred to a follow-up — the
  tree is rebuilt wholesale per element, so always-visible radios need persistent editors across the
  rebuild + rowsInserted + commitData-on-toggle (fragile); value layer is widget-agnostic so the swap
  stays localized to the delegate.
- **Lang .ts: DEFERRED.** The 4 strings are tr()-wrapped (translatable, English fallback at runtime);
  populating the 34 lang/*.ts is a project-wide `lupdate` sync = a separate maintainer/release commit,
  NOT bundled into the feature diff. Outstanding action at release: run lupdate to extract
  "Text Orientation:", "Upright", "ISO", "Free" (+ the renamed undo text "Modifier l'orientation
  d'un texte d'élément").

Test findings (2026-06-25, solid PARTIAL):
- T1 (MVR→ISO no-op when upright, both parities) PASS — matches Step 0 (net=0 readable for all parity).
- T2 (select ISO/Upright applies on Accept) **FAIL → it's the pre-existing toggle-lag**, see the
  "keep_visual_rotation toggle doesn't re-fire readability correction live" OPEN entry above. Selecting
  a mode sets the property but fires no correction; Upright/ISO snap only on the next element M/F/R.
  NOT a regression (the old bool setter never called correctReadability either). Fixing it = the
  tracked trigger-coverage pass; do NOT treat as new.
- T3 (R90° under ISO, incl. after M/F/R) PASS.
- T4.1 (Upright unaffected) PASS. T4.2 (Free unaffected) PASS, with a **minor caveat**: the reference
  0° direction shifts with element M/F/R (most visible during element rotation). Stable, repeatable;
  arguably should also be corrected during M/F/R. **Low priority.**
- T5 reopen-ISO in THIS build PASS. T5 reopen-ISO in a PREVIOUS (pre-tri-state) build: the ISO
  orientation is **RETAINED, not degraded to Free** — because the actuator's correction is baked into
  the serialized rotation/pos, so an old build renders the already-corrected geometry (the MODE
  degrades to Free, but the frozen angle persists). **Open question to revisit:** if MVR's intended
  semantics were "hold the user-selected orthogonal orientation" rather than "force tops-up/Upright",
  then the new `KeepUpright` may need further compensation to match MVR's true intent. Park until the
  MVR-semantics intent is confirmed.
- T6 (save/reload byte-stability, all three modes) PASS.

**SESSION HANDOFF (2026-06-26) — trigger-lag fix IN FLIGHT, test INCOMPLETE. Resume here after /clear.**
Git state on `mirror-flip-rotate` (NOT pushed — origin has neither commit yet):
- `d3dca3c91` Migrate element-text keep-visual-rotation to a RotationMode tri-state (commit A: data
  model + tri-value persistence + "Text Orientation:" combobox; ISO inert here).
- `2f29dd770` Apply ISO readability correction for IsoLayout element text (commit B: the actuator).
- **UNCOMMITTED working-tree edit** = the trigger-lag fix (commit C, not yet made): in
  `sources/qetgraphicsitem/dynamicelementtextitem.cpp`, `setRotationMode` now calls
  `m_parent_element->correctTextReadability(this)` after the connection block, guarded by
  `m_parent_element && !parentGroup() && m_parent_element->dynamicTextItems().contains(this)` (excludes
  construction + load + grouped; load re-settles via applyMirrorFlip element.cpp:857 regardless).
  **Builds GREEN**, binary mtime 2026-06-25 23:55. Also note `sources/main.cpp` is modified = the
  permanent app-name workaround, NEVER stage (standing rule).
- Proposed commit C message (held): "Re-fire readability correction when element-text orientation
  mode changes / DynamicElementTextItem::setRotationMode now re-runs the element-level
  correctTextReadability, so changing a text's orientation mode in Text Properties applies immediately
  instead of lagging until the next element rotate/mirror/flip. Restricted to an ungrouped text
  already registered with its element, which excludes construction and the load path (where
  applyMirrorFlip re-settles orientation); grouped texts are corrected through their group."

Why the trigger-lag fix went IN setRotationMode (not a custom undo command or a subclass): the Text
Properties apply() reconstructs a single-edit command via `QPropertyUndoCommand`'s copy ctor
(dynamicelementtextitemeditor.cpp:89), which would strip any QUndoCommand subclass — so the correction
must live in the setter, which the copy still reaches. Fires on both redo and undo (both call
setRotationMode), so orientation follows the mode through undo/redo. NO new silent-rewrite risk:
load-time correction already happens today via applyMirrorFlip(857), so the early setter fire is
redundant-but-harmless on load.

NEXT ACTIONS on resume (in order):
1. Run the trigger-lag functional test TL1–TL4 (INCOMPLETE — not yet run). Definitions:
   - TL1: Free text on element, rotate element 180° (text inverted), Text Properties → ISO → Accept ⇒
     ✅ snaps readable IMMEDIATELY (❌ before-fix: stayed inverted until next M/F/R).
   - TL2: text on a 90/180/270°-rotated element → Text Properties → Upright → Accept ⇒ ✅ tops-up at once.
   - TL3: after TL1/TL2, Undo ⇒ orientation+dropdown revert; Redo ⇒ re-applies.
   - TL4 (regression): re-run T6 (Upright+ISO+Free, save/reload/save) ⇒ byte-identical, no silent
     rewrite; a Free text still doesn't auto-orient.
   Use the binary built from the committed C state (rebuild after committing, or test the current
   23:55 binary which already contains the fix — it is functionally identical to what C will commit).
2. On PASS: commit C (stage ONLY dynamicelementtextitem.cpp — NOT main.cpp), with the message above.
3. Push decision: T2 was the push-blocker; commit C clears it. Remaining open items are NOT blockers —
   T4.2 (0° reference shifts with M/F/R, low priority) and T5 (MVR-semantics open question). If
   pushing, follow the Source-code workflow step 6 (cmake .. reconfigure to refresh GitRevision, then
   make, then push origin mirror-flip-rotate).
4. Still outstanding (not blockers): lang .ts lupdate sync (4 strings, deferred to release); radios as
   a follow-up to the combobox (value layer already widget-agnostic); the T5 MVR-semantics question
   (park until intent confirmed).

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

### Genuine ungroup of a mirrored element displaces text / renders it geometrically mirrored  · PARTIALLY RESOLVED (geometric-mirror half: ade229881) / positional displacement still DEFERRED
**Area:** element text groups · **Branch:** folio-mirror-flip (repro also on mirror-flip-rotate)
**UPDATE 2026-06-24:** the LIVE geometric-mirror half is RESOLVED by the shared `correctTextReadability` fix
(ade229881, top entry) — `removeTextFromGroup` now compensates the ungrouped text post-attach. The POSITIONAL
displacement (~14–24px, persists to disk) remains DEFERRED, untouched by that fix (it reflects from the
already-displaced pos; see the fix-shape note below). New diagnostic clue from the delete-undo-grouped test
(T4, files `debug-logs/livecomp-rerun-T4-delete-undo-grouped-*.qet`): restoring one grouped text via
delete+undo on a mirror+flip element moved ITS local position **(0, 19) → (−38.31, 0)** while the other two
grouped texts stayed pristine. **−38.31 ≈ 2 × 19.16**, with a 19→0 / 0→−38.31 axis swap ⇒ the `addToGroup`
local-position recompute under reflection has a DOUBLING-or-sign-flip character (not a generic miscalc) —
a concrete lead for whoever scopes the positional fix. Same ~19px family as the original ungroup symptom.
**Location (current line numbers):** `Element::removeTextFromGroup` (element.cpp:1451) →
`ElementTextItemGroup::removeFromGroup` (elementtextitemgroup.cpp:111, which `resetTransform()` :119 +
`setRotation(group rotation)` :120) → `Element::addDynamicTextItem` (element.cpp:1259). (Old "1367" ref was stale.)
**Symptoms (two coupled manifestations of one ungroup-through-reflection defect):**
- (orig) Genuine ungroup of a mirrored element (Properties → Delete-group) displaces grouped text ~19px;
  persists to disk (confirmed via round-trip).
- (new, 2026-06-24) With ROTATED grouped text, ungroup keeps the overall rotation/placement but renders the
  text GEOMETRICALLY MIRRORED with 2 of 3 lines OVERLAPPING — i.e. unreadable, materially worse than a ~19px nudge.
**Root cause — SAME family, traced from code (cross-check 2026-06-24):** the ungroup path detaches each text and
re-attaches it via `addDynamicTextItem`, which (as found for the "New text on an already-mirrored/flipped element"
OPEN entry) does NOT re-apply `compensateMirrorFlip` nor `correctTextReadability`. So each now-individual text becomes
a direct child of the still-reflected element (`applyMirrorFlip`'s element-level `setTransform(scale)`), inheriting
the reflection with an identity local transform ⇒ the geometric mirror; `removeFromGroup`'s `resetTransform()` +
`setRotation` leaves the per-text geometry in the group's baked/reflected frame rather than the clean mirror-free
"design" frame ⇒ the ~19px positional residual + line overlap. Same `removeFromGroup`-through-reflection family as the
committed save-path fix (05bcba506). **Shared gap with the create-text OPEN entry:** both flow through
`addDynamicTextItem` missing the post-attach `compensateMirrorFlip` — the geometric-mirror manifestation here IS that
live gap; the persisted ~19px is the additional `removeFromGroup`-baking issue.
**MEASURED — two issues are genuinely SEPARATE (2026-06-24, from the existing 23-Jun re-test artifacts
`debug-logs/qet-re-test-b0a405329-{00,ungroup-FAIL-geometric-mirror-of-TUs-01}.qet`; element `flip="true"`):**
- **Displacement PERSISTS to disk — confirmed, and bigger than "~19px": 14–24px.** The three `rotation="40"` group
  members moved on ungroup: ef969dbc (138.009,151.996)→(129.01,162.721) ~14px; 39005911 (125.673,165.142)→
  (139.814,148.289) ~22px; 838b8933 (152.793,134.377)→(129.01,139.721) ~24px. **Two of the three now share x=129.01**
  — that IS the "2 of 3 lines overlapping," baked into the saved file. `texts_groups` is empty in the after-file
  (group genuinely dissolved); rotation (40) and the element `flip` flag intact ⇒ purely positional, not orientation.
- **Geometric mirror is a LIVE-only artifact that self-heals — now FULLY CONFIRMED (visual + on-disk), test
  `ungroup-mirror-split` 2026-06-24.** Mirror is NOT serialized (transform-level); the 3 texts are written into the
  instance `m_dynamic_text_list` block (not a group), so any later `applyMirrorFlip` re-compensates them. User
  observations: S0 baseline readable/correct; S2 live-ungrouped = geometrically mirrored + 2 of 3 overlap; **S4 after
  close+reopen = mirror GONE, all 3 readable, same 2 still overlap**; **S6 after a live Mirror×2 transform (no reload) =
  mirror ALSO cleared, same 2 still overlap** — so the mirror clears on reload OR on any element transform (the
  missing-trigger self-heal). On-disk proof: the three texts' x/y are **byte-identical across S1 (live-broken save),
  S2 (post-reload save), and S3 (post-transform save)** — ef969dbc (129.01,162.721), 838b8933 (129.01,139.721),
  39005911 (139.814,148.289) — i.e. S1 *looked* mirrored and S2 didn't, yet the saved geometry is identical, proving
  the mirror was purely a live render state, never on disk; and the displacement is pixel-stable through reload and
  through a transform (compensateMirrorFlip reflects from the displaced pos, never corrects it).
**Severity:** RAISED low → MEDIUM — the new manifestation yields unreadable, overlapping text (not a minor ~19px
nudge). Still reachable only via Properties → Delete-group; no right-click ungroup gesture exists, which caps exposure.
**Fix shape — TWO fixes, not one (scoped 2026-06-24 via `addDynamicTextItem` caller map):** the two manifestations
need different fixes.
- **Geometric mirror (live): DONE (ade229881, 2026-06-24).** Added the post-attach `Element::correctTextReadability(item)`
  call (readability + `compensateMirrorFlip` when `m_mirror||m_flip`, mirroring `applyMirrorFlip`'s per-item pass) at the
  LIVE attach sites — `removeTextFromGroup` (ungroup), `AddElementTextCommand::redo` (create), and the UNGROUPED-text
  branch of `DeleteQGraphicsItemCommand::undo` (delete-undo). Closed this AND the create-text entry with one shared
  helper. The delete-undo GROUPED branch was trialled then dropped (group compensation persists across a text-only
  delete ⇒ the call is a no-op there; a comment marks the spot — see top entry).
- **Do NOT bake an unconditional hook inside `addDynamicTextItem`:** it is also on the LOAD path
  (instance `fromXml` element.cpp:807, `m_mirror` parsed at :789) and the definition-build path
  (`parseDynamicText` :620) — BOTH already compensate separately via `applyMirrorFlip()` at :857. An in-function call
  would be redundant on load AND fires at :807 *before* `deti->fromXml()` sets geometry (un-geometried no-op);
  `compensateMirrorFlip` is idempotent/transform-only so it wouldn't corrupt, but `correctReadability` mutates rotation,
  making an unconditional in-function call semantically muddy on load. Split at the call sites, leave load untouched.
- **Positional displacement (persists to disk):** SEPARATE fix in `removeFromGroup`/`removeTextFromGroup` — re-express
  the detached child geometry in the clean mirror-free "design" frame (cf. 05bcba506) instead of the scene-preserving
  detach. The live `compensateMirrorFlip` hook above does NOT fix this — it reflects from whatever displaced `pos()`
  the detach left, so the baked-in 14–24px / overlap survives regardless.
**Related (separate, not a dependency):** shares the ungroup/`removeFromGroup` path with the LAYER 2 readability entry.
Distinct concern (this = mirror geometry; that = rotation orientation display). Whoever codes second must not regress
the first in the shared function.
**Next:** the live `correctTextReadability` hook is DONE (ade229881). Remaining work is the ONE positional fix: the
`removeFromGroup`/`addToGroup` design-frame re-expression (trickier — it touches the shared detach that is *correct*
for unmirrored ungroup, so write a verification matrix first). Start from the −38.31 ≈ 2×19.16 doubling/sign-flip
clue above. NOTE: explicitly logged as future work — not today's task.

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
menu/UI code, not the geometry — cleaner as its own commit/PR. PERSISTENCE/encoding
for the third state is scoped: see "ISO-layout tri-state — persistence/encoding scoping"
(accepted recommendation: single tri-valued `keep_visual_rotation="true"|"false"|"iso"`).
UI TERMINOLOGY also finalized there: row label "Text Orientation:" with radios
"Upright"/"ISO"/"Free" → RotationMode KeepUpright/IsoLayout/Free.

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
