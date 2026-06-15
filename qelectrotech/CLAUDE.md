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
  never swept into a commit. NOTE: the working tree normally carries SEVERAL
  uncommitted local build-enablers (see "macOS dev build — recovery recipe"),
  not just main.cpp — so `git add <named files>` discipline matters more than ever.
- Pre-commit staging check (run before EVERY commit): `git status` and confirm
  the "Changes to be committed" list contains ONLY the named files this commit
  is meant to include. The working tree permanently carries local build-enablers
  (main.cpp, CMakeLists.txt, cmake/define_definitions.cmake,
  cmake/paths_compilation_installation.cmake, cmake/fetch_kdeaddons.cmake, and the
  TerminalStrip/elementscene/projectview Qt6 fixes) that must NEVER be staged onto
  a feature/fix branch. If any appears under "Changes to be committed" that isn't
  part of this specific fix, `git restore --staged <file>` it before committing.
  WATCH: recovery checkouts (`git checkout <ref> -- <file>`) AUTO-STAGE the files
  they restore — the most common way build-enablers silently end up in the index.
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
    - CLAUDE.md  (already gitignored)
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

## Fix metrics logging
For each source code fix, append an entry to:
  ~/claude-contexts/qelectrotech/fix-metrics.md

Do NOT include metrics in commit messages — they are personal development
records, not upstream history.

Each entry format:
  ## <commit hash> — <commit subject>
  Token cost: ~ | Tool calls:  | Graph queries:  | First attempt:

Example:
  ## 269ab1dd3 — Add folio-level Mirror/Flip for placed elements
  Token cost: ~180k | Tool calls: 30 | Graph queries: 1 | First attempt: PASS

Honesty requirement: metrics must be observed, never guessed. Record
`First attempt` truthfully (e.g. "No — N iterations" when corrected within
session, not PASS). When a counter is not introspectable (e.g. tool-call
count or token cost not visible this session), write `not captured` rather
than estimating a number. A blank or honest "not captured" is always
preferred over a fabricated figure.

Purpose: quantify Claude Code / Understand-Anything's contribution to fix
efficiency over time. Personal record — not for upstream.

First attempt notation:
    PASS = correct first time
    FAIL→PASS = corrected within session
    FAIL→FAIL→PASS = multiple corrections (extend as needed)
    PASS (untested) = compiled but not functionally verified
    PARTIAL = some call sites fixed, others deferred

## Project overview
QElectroTech (QET) is an open source Qt/C++ EDA tool for industrial
electrical schematic documentation. It is NOT a PCB tool — it targets
IEC 60617 wiring diagrams, terminal strips, and multi-folio
documentation.

## Active branch / current work
- Long-running integration branch on the fork: `qt6_cmake_joshua`.
- CURRENT feature/fix work happens on dedicated branches cut from a tree
  that BUILDS (see "Branching lesson" below), e.g. `folio-mirror-flip`
  (feature, done/posted) and `fix-grouped-text-rotation-pivot` (active).
- Do NOT assume Qt5 APIs are available. Targets Qt 6.x with KF6.

### Two-PR plan (folio mirror/flip work)
1. **macOS Qt6 build fixes** — own branch; the genuine upstream-worthy build
   fixes from the recovery recipe below (NOT main.cpp, NOT the obsolete
   projectview signature changes). Awaiting Laurent's reply on the CMakeLists
   guard form before sending. Target: master (build fixes are version-agnostic).
2. **Rotation-pivot bug fix** — own branch (`fix-grouped-text-rotation-pivot`);
   `elementtextitemgroup.cpp` updateAlignment(). Target master at PR time
   (bug file is byte-identical master/joshua). Rebase after PR #1 lands.
- Feature PR (folio mirror/flip) is already implemented on `folio-mirror-flip`;
  rebase/submit after the above land. Laurent invited it to master (pid 23013).

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

## macOS dev build — RECOVERY RECIPE (apply BEFORE attempting to build/run a fresh checkout)
Fresh `upstream/qt6_cmake_joshua` (and `master`) FAIL to build AND fail to load
resources on macOS/Qt6. macOS+Qt6+recent-Clang is the canary combination almost
nobody else builds, so these gaps survive upstream undetected. Apply this recipe
FIRST on any fresh checkout — before cmake, before make, before launch.
ALL fixes already exist on `origin/qt6_cmake_joshua` and are recoverable in
one command per file — do NOT re-derive them.

Recover each file (run from REPO ROOT, not build_qt6/):
    git checkout origin/qt6_cmake_joshua -- <file>

### Set A — make it BUILD (genuine upstream fixes; PR #1 candidates)
- `CMakeLists.txt` — guard the Linux-only install(FILES) MIME + appdata lines
  in `if(UNIX AND NOT APPLE) ... endif()`. Empty QET_APPDATA_PATH /
  QET_MIME_PACKAGE_PATH on macOS => "install FILES given no DESTINATION".
  USE THE GUARDED FORM (matches the existing paths-file idiom), NOT the old
  comment-out. Pending Laurent's preference on form.
- `cmake/fetch_kdeaddons.cmake` — add `set(ECM_SKIP_QDOC_GENERATION ON)`
- `sources/TerminalStrip/ui/terminalstripmodel.h` — qHash `uint`→`size_t` (x2:
  QColor and QPointer<Element> overloads). Qt6 qHash is size_t-based; uint
  overload is ambiguous.
- `sources/TerminalStrip/terminalstrip.cpp` — `static_cast<int>(indexOf(...))`
  (Qt6 indexOf returns qsizetype)
- `sources/editor/elementscene.cpp` — `static_cast<int>(selectedItems().size())`
- `sources/projectview.h` — **CAUTION, do NOT bulk-copy from the fork.** The
  fork's version carries STALE removeDiagram/moveDiagram signature + slot
  changes that DISAGREE with current upstream projectview.cpp and break the
  build. Instead: take CURRENT upstream projectview.h and apply ONLY
  `event->delta()` → `event->angleDelta().y()` on the two wheel-event lines
  (~47 and ~52). Nothing else.

### Set B — make it RUN (find elements/lang/titleblocks; dev-only)
These make a Debug build use binary-relative resource paths so QET finds data
next to the binary in build_qt6/ instead of the install path.
- `cmake/paths_compilation_installation.cmake` — APPLE block splits on
  `CMAKE_BUILD_TYPE STREQUAL "Debug"`: Debug => bare `elements/` `lang/`
  `titleblocks/` + `add_compile_definitions(..._RELATIVE_TO_BINARY_PATH)` and
  set the `..._RELATIVE_TO_BINARY_PATH_VAR ON`; Release => keeps
  `../Resources/...` bundle paths.
- `cmake/define_definitions.cmake` — `if(DEFINED ..._RELATIVE_TO_BINARY_PATH_VAR)`
  guards that emit the bare path WITHOUT prepending INSTALL_PREFIX (/usr/local/).
- Symptom if missing: French-only UI, no symbol library, no templates. NOTE the
  language symptom is a PATH problem, not a setting — setting lang=en alone does
  NOT fix it; the path fix does.

### Local-only (NEVER a PR)
- `sources/main.cpp` — app name "QElectroTech-Qt6" (SingleApplication conflict
  avoidance vs a parallel Qt5 build). Recover from fork, keep unstaged forever.

### After recovering Set A + B
- Re-run the canonical `cmake ..` configure (re-runs configure => bakes the new
  path -D definitions) then build.
- PREFER reconfigure WITHOUT wiping: `rm -rf build_qt6/*` is usually NOT needed
  and it (a) forces a slow KF6-from-source rebuild and (b) DESTROYS the
  build_qt6/ symlinks (see Dev build setup). On 2026-06-14, reconfigure-only
  picked up the path changes and ran correctly without a wipe.
- VERIFY the dev paths resolved (see Build environment > path-verify step):
  the configure output must show `... (relative to binary)`, NOT `/usr/local/...`.
- Once PR #1 (build fixes) is prepared as a clean branch, THAT branch becomes
  the canonical recovery source instead of cherry-picking individual files.

## CLAUDE.md management
This file lives in a separate repo: ~/claude-contexts/qelectrotech/CLAUDE.md
It is symlinked into the QET project root:
  ln -s ~/claude-contexts/qelectrotech/CLAUDE.md ~/qelectrotech/CLAUDE.md
CLAUDE.md is listed in QET's .gitignore — it is not tracked by the QET repo.
Backed up via: github.com/hairykiwi/claude-contexts

## Dev build setup (after a clean reconfigure / wipe)
Set B (cmake path fix) makes a Debug build look for `elements/`,
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
- WARNING: After any `brew upgrade extra-cmake-modules`, re-apply the
  if(NOT TARGET ...) guards to:
  `/opt/homebrew/share/ECM/modules/ECMGenerateQDoc.cmake`
  or the build will fail with duplicate target errors
- zsh: use heredoc (git commit -F - << 'EOF') for commit messages
  containing ! characters to avoid zsh history expansion errors
- QSettings domain for this build is `org.qelectrotech.QElectroTech-Qt6`
  (because of the main.cpp app-name hack), NOT `qelectrotech`. Matters for
  `defaults read/write org.qelectrotech.QElectroTech-Qt6 <key>`.

## Known harmless runtime messages (running uninstalled from build tree)
Do not re-investigate these — known, non-blocking:
- `failed to load qet_en ".../Resources/lang/"` — only fires if Set B path fix
  is absent OR a requested lang .qm is missing; with Set B present and English
  set, UI is English. Fallback to English source strings is harmless.
- `QObject::connect: No such signal QSignalMapper::mapped(QWidget *)` (x2,
  qetapp.cpp:122, qetdiagrameditor.cpp:100) — Qt6 removed mapped(); connect
  fails gracefully. Latent port gap, candidate for a future fix. Non-blocking.

## Fixes already committed
See git log for full details:
  git log upstream/qt6_cmake_joshua..HEAD
To get the current count: git rev-list --count upstream/qt6_cmake_joshua..HEAD
Known remaining issues are listed below — keep in sync with commits.

## Known remaining issues (not yet fixed)
- ACTIVE FIX IN PROGRESS — Grouped rotated text rotation/transform defects.
  Two related findings (see CC-TASKS.md for full diagnostic detail):
    * Finding 1 (pre-existing, NOT this feature's bug; confirmed reproducible
      on STOCK Qt5): grouped text rotates about its LOCAL ORIGIN (0,0) — the
      first-char lower-left corner of line 1 — instead of boundingRect().center(),
      because nothing calls setTransformOriginPoint(). Root cause in
      `elementtextitemgroup.cpp` updateAlignment() (~184-261). At 180° the block
      orbits that corner to a displaced spot; at odd 90° steps from saved
      orientation the lines overlap (even steps OK — a parity effect). Saving
      re-baselines the reference orientation (good/bad angles swap). Qt5
      repaints correctly on SAVE alone; Qt6 needs save+REOPEN (separate repaint/
      invalidation staleness layered on top). Affects PDF preview/output too,
      but only until saved+reopened — saved data is always correct.
    * Finding 2 (this feature): mirror/flip of grouped rotated text at element
      rotation 90/270 is incorrect (non-mirror/non-flip move, wrong side,
      overlap); 180° close-but-not-exact; 0° fine. Saved+reopened always
      correct. May largely dissolve once Finding 1's pivot bug is fixed.
- `sources/main.cpp`: app name temporarily set to "QElectroTech-Qt6"
  to avoid SingleApplication conflict with Qt5 build — revert before PR
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
