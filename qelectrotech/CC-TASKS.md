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

### Grouped/element rotated text displaced on LIVE rotate (display-only)  · OPEN (diagnosed, fix parked)
**Area:** element rotation render · **Branch:** fix-grouped-text-rotation-pivot
**Location:** `Element` rotation path / `element.cpp` (NOT elementtextitemgroup.cpp)

**Symptom:** When a placed element is rotated live (interactive rotate), grouped
(and ungrouped) text displaces — orbits to a wrong position; at 90°/270° lines
overlap. Save→reload ALWAYS renders correctly. So this is DISPLAY-ONLY; saved
data is correct.

**Diagnosis (this session, confirmed by instrumentation — do NOT re-derive):**
- The text GROUP is a passive, correctly-positioned child at constant element-
  local pos (-12,151). updateAlignment() is NOT the bug site — it recomputes the
  group's already-correct internal layout. A `rotationChanged → updateAlignment`
  connection was tried: it FIRES on live rotate but does NOT fix the render.
  Confirmed inert; reverted.
- CORRECTION (supersedes an earlier wrong reading): `Element::fromXml` does NOT
  compute any pos-compensation. It simply calls `QGraphicsObject::setPos(saved_x,
  saved_y)` then `setRotation(90*saved_orientation)` [element.cpp:769-781] — it
  restores the literal saved x,y + orientation, no rotation-derived math. The
  earlier-observed pos difference (0°→(370,320) vs 180°→(850,350)) was just
  DIFFERENT SAVED x,y in two different .qet files, NOT a reconstruction
  calculation. So the divergence is NOT in the load path.
- Therefore the real question is in the SAVE/ROTATE path: reload of a saved file
  renders correctly, so the saved x,y + orientation ARE correct — meaning
  something sets the element's correct pos between the live rotate and the save,
  OR the data was always correct and only the live VIEW is stale (repaint issue).
  transformOriginPoint stays (0,0) and boundingRect is constant (-60,-105 120x210)
  throughout — confirmed.
- Still valid: the LIVE rotate path's `Element::itemChange(ItemRotationHasChanged)`
  hook NEVER FIRES on interactive rotate (zero dumps), while the group's
  updateAlignment fires normally via the rotationChanged SIGNAL. So itemChange's
  rotation-notification path isn't reached live — likely missing
  ItemSendsGeometryChanges flag, OR rotate bypasses the flagged itemChange.

**Fix direction (next diagnostic — let data decide, do NOT assume):**
Dump element pos()/rotation() at THREE moments for a 0°→180° live rotate + save:
  1. POST-ROTATE (after rotation applied in the rotate path) — render is stale here
  2. AT-SAVE (in Element::toXml, where it reads pos/orientation to write) + the
     x,y actually written
  3. compare.
Key question: does pos() CHANGE between post-rotate (stale) and at-save (correct)?
  - YES → something corrects pos pre-save; find and apply it live (layer-1 fix =
    set the corrected pos on rotate so the live render lands the block correctly).
  - NO (pos identical + already correct at both) → data is correct live, the VIEW
    just isn't repainting it → pure repaint/invalidation fix.
LAYER-1 SCOPE = position only: make the live render land the text BLOCK in the
correct LOCATION on rotate (matching post-save position), no save needed.
Upside-down text at 180° is the ACCEPTED layer-1 endpoint. Readability is layer 2.

**LAYER 2 (separate, AFTER layer 1 — see also memory note):** per-text de-rotation
about each text's own bbox centre, for ALL text (grouped/ungrouped dynamic AND
simple), so it reads correctly. Independent of mirror/flip (its own readability
logic). Standard (researched): text reads from BOTTOM or RIGHT, never inverted
(AutoCAD TORIENT / NASA GSFC / ISO); vertical = read-from-RIGHT. 90° case is the
likely current defect. QET rotation is 90° steps only, so cardinal cases only.

**Severity:** low — display-only, save+reload corrects it; data always sound.
**Relation:** this is the root of Finding 1 (pre-existing rotation bug, confirmed
on stock Qt5) AND likely Finding 2 (mirror/flip of rotated grouped text). Fixing
the element-level live pos-compensation may resolve both. Bug 2 (ungrouped text
de-rotation/position at 90/270, save doesn't fix) is SEPARATE — own task.

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
**Next:** own scoped session; read 1367 path, confirm same-shape vs trickier
(it touches the shared removeFromGroup that's *correct* for unmirrored ungroup),
write fix + verification matrix before implementing.

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

### Dependency sketch
- #1 -> #2 (anchor visible before drag-by-anchor).
- #3 -> #4 -> #5 (terminal identity -> netlist -> BOM); #6 also feeds #5.
- #3 is the highest-leverage cornerstone; #1 is the cheapest standalone win.
