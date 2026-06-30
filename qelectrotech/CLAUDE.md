# QElectroTech — Claude Code Context

## Claude Code instructions
- AT SESSION START: also read `CC-TASKS.md` (same dir as this file, reached via
  the repo-root symlink) — specifically its ACTIVE section — for in-flight work,
  diagnostic narratives, and parked tasks. CLAUDE.md holds durable project/build
  rules; CC-TASKS.md holds what's currently in flight. (Read ACTIVE on startup;
  the ARCHIVE and FEATURE IDEAS sections only when a task calls for them, to keep
  startup context lean.)
- Before committing, show BOTH the proposed commit message AND the staged
  file list (`git status --short`) for review, and WAIT for explicit
  approval. Do not self-approve with "diff looks right" and commit in the
  same step. Staging and committing are separate from approval.
- NEVER commit sources/main.cpp — it carries the temporary "QElectroTech-Qt6"
  app-name workaround and must stay unstaged. Never use `git add -A` or
  `git add .` blindly — stage named files only, so the workaround is
  never swept into a commit. main.cpp is the ONLY local-only build-enabler.
- Pre-commit staging check (run before EVERY commit): `git status` and confirm
  the "Changes to be committed" list contains ONLY the named files this commit
  is meant to include. main.cpp must NEVER be staged onto a feature/fix branch.
  If it appears under "Changes to be committed," `git restore --staged
  sources/main.cpp` before committing. WATCH: recovery checkouts
  (`git checkout <ref> -- <file>`) AUTO-STAGE the files they restore — historically
  the most common way build-enablers silently ended up in the index.
- Source code change workflow — full sequence for every source code change,
  from edit through push:
    1. Edit source
    2. cd ~/qelectrotech/build_qt6 && make -j8
       (compile check — confirms the edit builds cleanly)
    3. Confirm fresh binary before testing: `ls -la build_qt6/qelectrotech`
       shows a current mtime — this is the AUTHORITATIVE freshness check; it
       updates on every relink. NOTE: QET > About "Built ... Date" and
       GitRevision do NOT reliably update for a single-file UNCOMMITTED change
       — the date macro only refreshes when its own translation unit
       recompiles, and GitRevision only at commit + configure. So trust the
       binary mtime, NOT the About strings, for pre-commit test freshness.
       Never run a functional test against a binary whose mtime predates the
       edit — a stale-binary test produces false FAILs (and false PASSes).
    4. Functional test (run against this pre-commit build)
    5. git commit (named files only — never main.cpp)
    6. cd ~/qelectrotech/build_qt6
       cmake .. [see Build environment section > Canonical CMake configure command]
       make -j8
       (cmake .. is required — git_last_commit_sha.cmake runs at configure
       time only, so make alone won't refresh GitRevision)
    7. git push origin <branch>
- When committing a source code fix, remove the corresponding item
  from "Known remaining issues" in the same CLAUDE.md commit
- Before preparing any PR to upstream, exclude these fork-only artifacts —
  they belong on the fork, not in upstream history:
    - .understand-anything/  (generated knowledge graph, committed in bfc312780)
    - CLAUDE.md and CC-TASKS.md  (gitignored; fork-local context, never upstream)
  bfc312780 already committed the graph, so gitignoring now will NOT
  un-commit it. Cut the PR from a clean branch off the PR-target and
  cherry-pick only the genuine source/build fixes onto it (this also drops
  the .gitignore / CLAUDE.md / graph housekeeping commits). Confirm with
  Laurent/Joshua before including the graph in-tree.
- Understand-Anything graph: auto-update is NOT installed in this clone and
  will NOT be reinstalled — graph updates are MANUAL. The post-commit hook
  lives in .git/hooks/ (per-clone, never committed, does not travel); a fresh
  clone won't have it and we're not adding it here. Do not narrate graph files
  as "auto-patched" — knowledge-graph.json / meta.json / fingerprints.json
  change ONLY when manually rebuilt via /understand-chat. When rebuilt, commit
  those three tracked JSONs as a standalone tooling commit — never fold them
  into a source commit. Never stage .understand-anything/intermediate/ or
  diff-overlay.json (local scratch; keep gitignored). Graph is fork-only —
  exclude from any upstream PR.
- The graph predates recent commits and must always be treated as a map only,
  verified against the live tree — this is already the standing rule, FOLLOW
  IT SILENTLY. Don't restate "treating the graph as stale / verifying against
  live tree" as a sentence in every report that uses it; that's noise once
  it's a default, not new information each time.

## Coding guidelines
**Highlight ambiguity before coding**
If multiple interpretations exist, present them — don't pick silently.
**Simplicity first**
Minimum code that solves the problem — no speculative features,
no abstractions for single-use code, no "flexibility" that wasn't
asked for. If you write 200 lines and it could be 50, rewrite it.
**Surgical changes**
Touch only what the task requires:
- Don't improve adjacent code, comments, or formatting.
- Match existing style even if you'd do it differently.
- If your changes make imports/variables/functions unused, remove them.
- Don't remove pre-existing dead code unless asked — mention it instead.

**Nomenclature: no internal shorthand in upstream-facing text**
Session-local terms (layer-1/2, MVR, gate-1/2/α/β/γ, "phenomenon X", architecture
labels like "A'") are working shorthand for chat/CC-TASKS.md ONLY.
They must never appear in source comments, @briefs, or commit messages — those are
read by Laurent and upstream, who have no context for them. Describe the actual
mechanism instead (what the code does, which function, which commit hash). Sweep
the whole diff for this before presenting it, not just the line being discussed —
caught 3 times in one session by only fixing the line that was pointed out.

**Spelling consistency (English variants)**
QET is an international project; perfect consistency across the whole codebase
isn't realistic and isn't the goal — don't drive-by-fix unrelated pre-existing
spelling elsewhere in a file as part of an unrelated change. For any NEW text you
write — code comments, @briefs, commit messages, new identifiers — match Qt's own
spelling conventions wherever you're naming or referencing parameters, identifiers,
classes, or methods; otherwise match whatever convention the immediate surrounding
file already uses where one is established. If you hit a genuine inconsistency you
can't resolve by matching adjacent style — conflicting conventions in the same area,
or a case this rule doesn't cover — flag it explicitly rather than silently picking
one; User will decide whether it needs a rule here, a tracked CC-TASKS.md item,
or no action.

**Commit message/comment text must match the tree AT THAT COMMIT**
When a single piece of work spans multiple commits (e.g. a rename landing in a
later commit than the code it describes), each commit's message and any doc
comments in its files must be accurate to what exists in THAT commit's tree —
not the final state after later commits. Caught twice in one session: a doc
comment forward-referencing a not-yet-renamed function, and a commit message
forward-referencing a not-yet-renamed function. Check this explicitly before
each commit in a sequence, don't assume the first edit pass got it right.

**Amending commits: local-only is free, pushed requires force-push — know which**
Amending/rewriting a commit always creates a new SHA; the old one is discarded.
If that commit was never pushed, this has no external consequence (nothing else
references the old SHA) — safe by default. If it WAS pushed, amending requires a
force-push and must be confirmed explicitly first, never done as a side-effect of
an unrelated approval. Prefer a new follow-up commit over rewriting pushed history
whenever the fix is additive (wording/nomenclature corrections, comment fixes) —
only rewrite pushed history when the content itself must not survive (credentials,
genuinely wrong information that can't simply be corrected forward).

## Diagnosing hard / confusing bugs (measure, don't hypothesise)
Hard-won from the rotation-text fix (multi-session, ~6 wrong turns, all from
guessing ahead of data). When behaviour is confusing or a fix attempt fails:
- MEASURE before hypothesising. Instrument and dump real values; don't reason
  from the code alone. Every wrong turn came from a plausible hypothesis that a
  one-line dump would have killed in minutes.
- Compare in a SINGLE run under IDENTICAL conditions. A false "off by (40,10)"
  conclusion came from comparing two runs with different element placements.
  Capture both states (e.g. live vs reload, before vs after) in one run, same setup.
- Measure the FULL picture, assume nothing about scope. Dumping element + group +
  EACH text (pos, rotation, transform matrix, mapToScene) in one shot isolated the
  variable; narrow measurements hid it (a bbox masked an orientation flip; a
  group-level value masked per-text divergence). Include the transform matrix —
  a mirror/flip shows as a sign flip there that pos/rotation alone hide.
- Side-by-side correct-vs-wrong is the fastest way to see divergence (live vs the
  known-good reload; grouped vs ungrouped). Let the one quantity that differs name
  the cause — then fix THAT, not a symptom.
- After 2-3 failed fixes, STOP coding and go back to measurement. Repeated fix
  failure means the mental model is wrong, not the patch

## Debug / instrumentation logs
Write all instrumentation/debug logs to ~/qelectrotech/debug-logs/ (gitignored)
with descriptive names, e.g. `qet 2>debug-logs/<feature>-<purpose>.log`.
Not /tmp (purged on reboot, hidden from Finder). CC reads its own output from
there; recreate the dir after a fresh clone (it's gitignored, so not tracked).

For mixed human/CC test sessions specifically (CC launches/logs, User drives
the GUI): CC names the test/log at the HEAD of the test schedule it generates
— that name is the anchor. If User ends up saving a .qet file as part of the
test, save it under that SAME name (swap extension only) rather than CC
trying to match a filename after the fact. CC usually can't name-match a test
file in advance anyway, since the file frequently doesn't exist yet at launch
time — it gets created fresh during the session.
  
## Testing requirements
Before pushing any commit to origin, Claude must propose and the developer
must perform a brief functional test relevant to the change:

- Test format: numbered steps, clear ✅ expected / ❌ before-fix outcomes
- Scope: targeted to the specific behaviour changed — not a full regression suite
- Result: PASS or FAIL recorded in the session before git push proceeds
- If FAIL: fix before pushing, do not push broken commits
- Ordering: the functional test (Source code change workflow > step 3) must
  PASS before commit (step 4) and push (step 6).
- Before/after evidence: where a "before" Terminal state is cheaply
  reproducible, capture both before and after for the worked issue. Where
  the fix is latent or the test only confirms ABSENCE of a warning (no
  reproducible before-state), state "absence-confirmed" explicitly instead
  of implying a diff.

This applies to all source code commits. CMake, documentation, and
tooling-only commits may be pushed without a functional test at discretion.

## Project overview
QElectroTech (QET) is an open source Qt/C++ EDA tool for industrial
electrical schematic documentation. It is NOT a PCB tool — it targets
IEC 60617 wiring diagrams, terminal strips, and multi-folio
documentation.

## Active branch / current work
- Active integration branch: `mirror-flip-rotate` (off `qt6_cmake_joshua`; clean-merges
  `folio-mirror-flip` + cherry-picked layer-1 `843ba6898`). Latest commits, in order:
  `b0a405329` (grouped+flip position fix), `6f8dd772c` (readability correction, [WIP] —
  see CC-TASKS.md ACTIVE for the residual trigger-lag item), `bd61ca17c` (text-rotate
  pivot fix), `feeef3cea` (nomenclature normalization). Push is fast-forward on origin;
  no force-push so far this branch — keep it that way (new commit > rewrite, when the
  fix is additive).
- Do NOT assume Qt5 APIs are available. Targets Qt 6.x with KF6.

### Separating upstream-able fixes from in-progress feature work
General principle, durable: a dev branch built for one feature often accumulates other
fixes along the way — build fixes, pre-existing bugs found incidentally, unrelated
cleanup — that are independently valuable and not entangled with the feature itself.
Split these out before submitting anything upstream, rather than letting them ride
along buried in feature-branch history. Two shapes this takes, pick based on whether
the fix's files/structure match upstream's current layout:
- **PR-shaped** (files/structure match upstream closely enough that a branch or patch
  would apply cleanly): prepare via squash/rebase into a small number of well-described
  commits, own branch, submit as a normal PR.
- **Report-shaped** (fork's files have diverged in path/structure from upstream's
  current layout, so a literal patch wouldn't apply): write up the diagnosis + fix
  direction as a bug report instead — describe what's wrong and how it was fixed on
  the fork, let upstream apply the equivalent fix to their own current code shape.

**Current candidates (live snapshot — update or remove each as it resolves; don't let
this list go stale once something's actually done):**
1. **Rotation-pivot bug fix** — DONE, folded into `mirror-flip-rotate` via the
   `843ba6898` cherry-pick; the originating branch is superseded, don't develop on it.
2. **Elements-panel crash fixes** — report-shaped (fork's elements-panel files live at
   different paths than upstream's current layout, e.g.
   `sources/elementspanel/genericpanel.cpp` vs. upstream's `sources/genericpanel.cpp`).
   Pre-existing upstream defects (origin `a82f6de23`), unrelated to the Qt6 port,
   discovered incidentally while testing the mirror/flip feature. Family A (orphan
   item) RESOLVED `ad69d989b`; Family B (project-open crash) root-caused to a narrow
   freed-pointer race, repro elusive, defensive guard committed `917c01da9`. See
   CC-TASKS.md for full detail.
- Feature PR (folio mirror/flip) is already implemented on `folio-mirror-flip`;
  ready to rebase/submit. Laurent invited it to master (pid 23013).

### Branching lesson (IMPORTANT — avoids re-deriving build fixes)
Develop on a tree that ALREADY BUILDS; reconcile to the PR-target base only
at PR time. Do NOT cut a working branch off FRESH upstream and then rediscover
every macOS build fix — that cost most of a session on 2026-06-14. Fresh
`upstream/qt6_cmake_joshua` and `master` do NOT build/run on macOS out of the
box. The fix file for a given bug is usually byte-identical across branches, so
you can develop on the fork's working tree and still produce a clean PR diff of
just the touched file(s) against the target base.

## Build environment (macOS Apple Silicon)
- Hardware: Mac Mini M4, macOS Tahoe (26.5)
- Qt: 6.11.1 via Homebrew (`/opt/homebrew/opt/qt`)
- KF6: built from source via FetchContent (BUILD_KF6=ON), v6.22.0
- pugixml: system install via Homebrew
- Build directory: `build_qt6/`
- Canonical CMake configure command:
  cmake .. -DCMAKE_PREFIX_PATH=/opt/homebrew/opt/qt \
           -DQt6_DIR=/opt/homebrew/opt/qt/lib/cmake/Qt6 \
           -DBUILD_KF6=ON \
           -DCMAKE_POLICY_VERSION_MINIMUM=3.5 \
           -DCMAKE_POLICY_DEFAULT_CMP0002=OLD \
           -DCMAKE_BUILD_TYPE=Debug
- Build command: make -j8
  (-j8 is plenty for single-file test rebuilds; core count is within noise
   there and only matters for a full KF6-from-source rebuild. M4 has 10 cores.)
- After ANY `cmake ..` configure (full or reconfigure), VERIFY the dev paths
  resolved. Run the grep SEPARATELY — do NOT pipe cmake through grep: piping
  hides non-matching errors and makes the exit status grep's, not cmake's
  (so a failed configure could silently proceed to make). Either:
    grep -iE "QET_LANG_PATH|relative to binary" <configure output>
  or just scan the printed "Paths used for compilation" block. Confirm the
  paths show e.g. `QET_LANG_PATH lang/ (relative to binary)` — NOT `/usr/local/...`.

## macOS dev build — local-only build-enablers
As of 2026-06-30, the only local-only, never-committed file is `sources/main.cpp`
(line 209, app-name workaround for SingleApplication conflict vs a parallel Qt5
build — apply manually for dev builds only, never stage or commit). The fixes
this section used to track (CMakeLists.txt install(FILES) guards, KF6
FetchContent doc-gen skip, dev-build resource paths, several Qt6 migration
casts) are confirmed committed natively on both `qt6_cmake_joshua` and
`mirror-flip-rotate` (live grep against both branches, 2026-06-30) — a fresh
checkout of either builds and runs with no manual file recovery.

## Context files (CLAUDE.md + CC-TASKS.md) accessibility and management

**Context file accessibility**
Claude Code has direct filesystem access to ~/claude-contexts — there's no
need to go through the symlinks below for editing these files. WRITE TO THE
REAL PATH, NEVER THROUGH THE SYMLINK: some editor tools follow symlinks fine
for reading but fail to write through one.
- Both files live in a separate repo: ~/claude-contexts/qelectrotech/
- QET-repo-side symlinks exist only so the build tooling can find the files
  in-place:
  ln -s ~/claude-contexts/qelectrotech/CLAUDE.md   ~/qelectrotech/CLAUDE.md
  ln -s ~/claude-contexts/qelectrotech/CC-TASKS.md ~/qelectrotech/CC-TASKS.md
- The symlinks are per-clone (not committed) — recreate BOTH after any fresh
  clone of the QET repo. Backed up via: github.com/hairykiwi/claude-contexts

**Context file management — division of labour (explicit, this is the default)**
- **CC-TASKS.md: ONE narrow no-gate exception, everything else needs User's
  input BEFORE drafting, not just before pushing.** Pure post-build status
  (build succeeded/failed, binary rebuilt, mtime, fresh/clean) — CC drafts and
  commits directly, no check needed first; push still waits for confirmation,
  same as always. EVERYTHING ELSE that goes into CC-TASKS.md — test results,
  CC's own analysis/interpretation, root-cause framing, severity calls, ANY
  conclusion that isn't a plain build-status fact — gets the SAME standard as
  a QET source-code commit: report the findings to User first, propose the
  entry/commit message, wait for explicit go-ahead, THEN commit. Don't draft
  silently and report afterward; say plainly that CC-TASKS.md is being held
  pending User's input, so it's never ambiguous whether it's already written.
  Reason: User's response routinely adds a recollection or correction that
  changes the right framing — catching that before it's written down beats
  fixing a committed entry after. CC-TASKS.md being a low-stakes, easy-to-fix
  personal record is not a reason to skip an easy chance to get it right the
  first time when User's input is reliably one message away.
- **CLAUDE.md: User's file, maintained working in conjunction with Claude
  chat.** CC must NOT edit or push CLAUDE.md directly, ever, by default.
- **Exceptions to either default must be stated explicitly, in the moment,
  by User — never inferred or assumed by CC.** If User wants a specific
  CC-TASKS.md push held back, or wants CC to make a specific CLAUDE.md edit
  directly, User will say so plainly for that instance. Don't generalize a one-off
  instruction into a new standing default either direction.

Both files are explicitly listed in QET's .gitignore so they cannot be
accidentally committed into the QET source repo.

**What belongs in these files — before adding any sentence, ask:** would a cold
reader (a fresh session with no memory of how this was discovered, or User after
a gap) need this to act correctly — or does it already exist in more detail
elsewhere, or is it about to resolve/become moot on its own before anyone reads
it back? If either of the latter, cut it or reduce it to a one-line pointer.
Aim for this on the first draft rather than relying on a trim pass after.

## Dev build setup (after a clean reconfigure / wipe)
The cmake path fix makes a Debug build look for `elements/`,
`titleblocks/`, `lang/` NEXT TO THE BINARY (i.e. inside build_qt6/). Satisfy
that:
- `lang/` is GENERATED into build_qt6/lang/ by the build (the .qm files) —
  no action needed.
- `elements/` and `titleblocks/` are source data dirs — symlink them in:
    cd ~/qelectrotech/build_qt6
    ln -s ../elements elements
    ln -s ../titleblocks titleblocks
These symlinks are covered by .gitignore (build_qt6/) and MUST be recreated
after `rm -rf build_qt6/*` — which is the main reason to AVOID wiping when a
plain reconfigure will do.

## macOS development quirks
- After sleep/wake or reboot required: QET may refuse to launch due to
  SingleApplication stale shared memory. A reboot clears it reliably.
  Ad-hoc codesign does not help.
- If launch silently exits with code 0: check
  `ls /tmp/ | grep -i qet` and `ps aux | grep -i qelectrotech`
- Reboot is the reliable fix until SingleApplication is replaced or
  binary is properly notarized

**QET process launch/exit protocol for CC — CORE MECHANICS CONFIRMED (first
trial, 2026-06-22); two refinements added below from that trial.** Supersedes
any prior CC-internal note about always handing QET launch to User. After an
earlier incident (CC launched and force-killed the app, likely contributing
to the SingleApplication crash above), this protocol was trialed instead of
an outright "CC never launches it" rule. The launch/handoff/exit-poll
mechanics themselves worked cleanly both directions, no premature exit, no
stale-process surprises — that part is confirmed, not provisional anymore.
- CC MAY launch the QET binary itself (backgrounded, output captured to a
  log file) to gather data from a running session.
- CC MUST NEVER send the process any termination signal, for any reason —
  not `kill`, not `pkill`, not even a "gentle" SIGTERM. A clean shutdown
  through the app's own Quit path is the only termination method known not
  to risk SingleApplication corruption; CC has no way to trigger that path
  itself, so it must not attempt termination at all.
- Before any test run: commit/push anything in flight and verify it landed
  (don't just assume a commit succeeded), and check for a lingering stale
  process from a previous session (`ps aux | grep qelectrotech`) before
  launching fresh.
- While waiting for User's handoff signal, CC may periodically re-read
  (tail) the log file rather than only reading it once at the end — lets
  instrumentation output get digested as it's produced, not cold all at once
  after the fact.
- CC signals explicitly when it's done gathering data and ready for User to
  quit — an unambiguous handoff, not implied. User then quits via the normal
  GUI path. CC's only remaining job is to poll for the process actually
  exiting, never to help end it.
- If the process disappears before that explicit handoff — treat it as an
  unplanned crash, report it as such, and check whether a reboot is needed
  before any relaunch attempt.
- CC has NO visual/GUI access in this workflow — backgrounded process, log
  file only. Don't ask CC to report on anything that can only be judged
  visually (e.g. how an animation looks); that has to come from whoever's
  actually watching the screen, or from frame-by-frame instrumentation that
  doesn't depend on visual judgment at all.

- WARNING: After any `brew upgrade extra-cmake-modules`, re-apply the
  if(NOT TARGET ...) guards to:
  `/opt/homebrew/share/ECM/modules/ECMGenerateQDoc.cmake`
  or the build will fail with duplicate target errors
- zsh: use heredoc (git commit -F - << 'EOF') for commit messages
  containing ! characters to avoid zsh history expansion errors
- QSettings domain for this build is `org.qelectrotech.QElectroTech-Qt6`
  (because of the main.cpp app-name hack), NOT `qelectrotech`. Matters for
  `defaults read/write org.qelectrotech.QElectroTech-Qt6 <key>`.

## Known harmless diagnostics (non-runtime, editor/LSP only)
- Without a `compile_commands.json`, clangd cannot resolve Qt includes at all —
  one failure ("QUndoCommand not found" or similar) cascades into "unknown type"
  on every downstream Qt-typed line in that file. Symptom looks alarming (can
  report "N new diagnostic issues") but is pure LSP noise, unrelated to any edit
  just made. VERIFY against the actual `make` build log (warnings/errors count),
  not the editor's diagnostics panel — that's the authoritative source. Don't
  let a large diagnostics count block a commit without checking it traces to
  this cause first.

## Known harmless runtime messages (running uninstalled from build tree)
Do not re-investigate these — known, non-blocking:
- `failed to load qet_en ".../Resources/lang/"` — only fires if the dev-build
  path fix is absent OR a requested lang .qm is missing; with the path fix
  present and English set, UI is English. Fallback to English source strings
  is harmless.
- `QObject::connect: No such signal QSignalMapper::mapped(QWidget *)` (x2,
  qetapp.cpp:122, qetdiagrameditor.cpp:100) — Qt6 removed mapped(); connect
  fails gracefully. Latent port gap, candidate for a future fix. Non-blocking.

## Fixes already committed
See git log for full details:
  git log upstream/qt6_cmake_joshua..HEAD
To get the current count: git rev-list --count upstream/qt6_cmake_joshua..HEAD
Known remaining issues are listed below — keep in sync with commits.

## Known remaining issues (not yet fixed)
- `sources/main.cpp` app-name workaround — see "Claude Code instructions" above;
  never stage it.
- .understand-anything/ knowledge graph committed to branch (bfc312780) —
  must be excluded from any upstream PR (see PR-prep instructions above)
- `The requested buffer size is too big` — Qt6 SVG renderer stricter
- `QVariant::Type` / `.type()` / `.canConvert()` → QMetaType equivalents — `userproperties.cpp`, `numerotationcontext.cpp`
- `setNamedColor()` → `QColor::fromString()` — `terminalstripbridge.cpp`
- `QLocale::nativeCountryName()` → `nativeTerritoryName()` — `machine_info.cpp`
- `QString::count()` → `size()` — `elementscollectionwidget.cpp`
- Display menu missing entries for Collections and Templates windows
  (found via Help → Search "Collections" as workaround) — pre-existing
  Qt6 migration issue
- `QWidget::setLayout: Attempting to set QLayout on StyleEditor which
  already has a layout` — seen in element editor, Qt6 migration issue
- `QBoxLayout::insert: index out of range` — seen on new diagram creation
- `QObject::disconnect: wildcard call disconnects from destroyed signal of
  AutoNumberingDockWidget` — fires on quit, minor
- Font weight inconsistency between elements in Qt5-origin .qet files —
  some elements saved with explicit Thin weight render lighter than Normal
  weight elements; cosmetic, not introduced by Qt6 migration, and separate
  from the weight-0 load fix (8fd5431be)

## Contribution context
- Fork: github.com/hairykiwi/qelectrotech-source-mirror
- Upstream: github.com/qelectrotech/qelectrotech-source-mirror
- Key upstream contacts: scorpio810 (Laurent, project manager),
  Joshua-cla (Qt6 branch author)
- PR target: master for version-agnostic fixes (Laurent's request);
  qt6_cmake_joshua for Qt6-era feature work. Confirm per-PR.
- Commit style: short summary line, blank line, bullet detail
- Folio mirror/flip feature posted to forum (Code section, pid 23010);
  Laurent invited merge to master (pid 23013). Credits plc-user's
  Element-Editor rotate/flip/mirror (0.100-dev) + QET_ElementScaler (2022)
  as the element-definition-level predecessor to this folio-level work.

## Qt6 migration notes (patterns; several already applied — see recovery recipe)
- qAsConst() → std::as_const() (warnings, not errors)
- QWheelEvent::delta() → angleDelta().y()   [applied: projectview.h]
- qHash seed: uint → size_t                 [applied: terminalstripmodel.h]
- QSignalMapper::mapped(T) → mappedInt / mappedObject / mappedString
- Brace-initialised int from qsizetype → static_cast<int>()  [applied:
  terminalstrip.cpp, elementscene.cpp]
- SQLite: QSqlDatabase::exec() deprecated → use QSqlQuery::exec()
- QFont weight: Qt6 requires 1–1000 (Qt5 serialised Thin as 0, which Qt6
  rejects). Remap font weight <1 → Thin (100) at load via
  QETApp::sanitizeFontString(); read-time only, never written back.
- QDomDocument: device must be open before passing to setContent()
