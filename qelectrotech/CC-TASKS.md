# CC-TASKS — grouped-rotated text: mirror corrupts on save/reload

**Branch:** folio-mirror-flip @ c6344261f (+ current debug instrumentation)
**Scope:** `Element::applyMirrorFlip()` (element.cpp ~1064–1125), `ElementTextItemGroup`, `DynamicElementTextItem`.
**Rule:** diagnose only. No fix until A/B/C land.

## What the trace already established (don't re-check)
- `det(ct)` is **not** diagnostic. Group `ct` is det −1 for BOTH mirror (`[[0,-1],[-1,0]]`)
  and flip (`[[0,1],[1,0]]`), and det −1 ALSO appears in the PASSING ungrouped rotated
  cases (rotation-330 child: `[[0.5,-0.866],[-0.866,-0.5]]`). The det-mirror-vs-flip check
  from the prior session is dropped.
- Real signal: the rotated group's children differ between states.
  - Loaded from last session's mirrored save: group bbox `QRectF(136, ~0, 136x38)`,
    children at local `(136, ~0)` and `(136, 19)`.
  - Clean / unmirrored: bbox `QRectF(~0, 0, 136x38)`, children at `(0,0)` and `(~0, 19)`.
  - Group width = 136. Child frame is shifted +136 in x (one group width) in the
    saved→reloaded state — a position-level x-reflection — while the orientation
    reflection lives in the runtime `ct` (recomputed at load).
  - Net: live = children@0 + one ct reflection = correct. Reload = children@136 + ct
    reflection = mirror applied twice = visible Y-flip.
- Live mirror clicks this session kept children at `(0,0)`. So the mutation is at SAVE
  or RELOAD, not at mirror-click. No same-session save→reload capture exists yet.

## Check A — serialization (central)
Save a file with the rotated group mirrored, then inspect the on-disk XML.
- Dump the `<element>` + its `<texts>/<text_group>` fragment for the rotated-group element
  (Section 3-2 "ROTATED Group Text Test").
- Report verbatim: what positions are written for each child text — the live `(0,0)`
  values or the shifted `(136, *)` values? And is any transform/matrix/det written, or
  only rotation / x / y / alignment / keep_visual flags?
- Locate the writers: `ElementTextItemGroup::toXml()` and `DynamicElementTextItem::toXml()`.
  List exactly which attributes each emits.
Decisive: confirms whether the +136 shift is persisted in child positions (expected) and
whether the reflection has any serialized representation (expected: none).

## Check B — find the mutation point (replaces the old det check)
One session, no restart:
1. Mirror the rotated group. Log each child `pos()` immediately after. (expect `(0,0)`/`(0,19)`)
2. Save. Re-log each child `pos()`.
3. Close + reload from the .qet. Re-log each child `pos()`.
Identify which step flips child local x from ~0 to ~136.
Then instrument the suspects so we catch the writer that moves the children:
- `ElementTextItemGroup::autoPos()`
- `ElementTextItemGroup::updateAlignment()`
- `DynamicElementTextItem::parentElementRotationChanged()` / keep_visual re-assert
Add a one-line qDebug at each that prints child pos before/after it writes.
Decisive: names the exact code path baking the mirror into serialized child positions.

## Check C — live vs reload scene ground truth
For the rotated group + each child, log `sceneTransform()` (m11,m12,m21,m22,dx,dy) and
`mapToScene(boundingRect().center())` at: (a) post-mirror, (b) post-save, (c) post-reload.
Diff (a) vs (c).
Expected: orientation identical (both det −1); child scene POSITIONS differ by ~136 along
the group's axes — confirming a position round-trip bug, not an orientation one.

## Order
A and B first (jointly decisive on where the +136 enters and whether it's persisted).
C confirms end-to-end and gives the exact residual the fix must remove.
Paste raw qDebug back to claude.ai per check; no fix proposal until all three are in.

---

## RESOLVED — save-path serialization (committed 05bcba506, branch folio-mirror-flip)
Grouped rotated text saved displaced by one group width: toXml detached each child via
the visual-preserving ElementTextItemGroup::removeFromGroup(), baking the element mirror
into the child x/y (half-applied — position only — then re-asserted by applyMirrorFlip on
load). Fixed by serializing each child's element-relative "design" geometry directly from
the group's pos/rotation in one frame (element.cpp Element::toXml grouped-text loop). Matrix
verified: acceptance + regression (3-line group, element-rot 90/180 + mirror, mirror+flip)
all det-parity +1 and idempotent on re-save.

## DEFERRED — genuine ungroup of a mirrored element displaces text
Genuine ungroup of a mirrored element (Properties → Delete-group) displaces grouped text
~19px and persists to disk (confirmed via round-trip). Same removeFromGroup-through-reflection
family as the just-committed save-path fix, but on the Element::removeTextFromGroup /
elementtextitemgroup.cpp:1367 path — likely the same shape of fix (re-express child geometry
in one mirror-free frame instead of relying on the scene-preserving detach). Low severity:
reachable only via Properties → Delete-group; no right-click ungroup gesture exists.
