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

### Dependency sketch
- #1 -> #2 (anchor visible before drag-by-anchor).
- #3 -> #4 -> #5 (terminal identity -> netlist -> BOM); #6 also feeds #5.
- #3 is the highest-leverage cornerstone; #1 is the cheapest standalone win.
