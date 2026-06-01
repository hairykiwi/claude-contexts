# QElectroTech — Claude Code Context

## Claude Code instructions
- Before committing, show BOTH the proposed commit message AND the staged
  file list (`git status --short`) for review, and WAIT for explicit
  approval. Do not self-approve with "diff looks right" and commit in the
  same step. Staging and committing are separate from approval.
- NEVER commit sources/main.cpp — it carries the temporary "QElectroTech-Qt6"
  app-name workaround and must stay unstaged. Never use `git add -A` or
  `git add .` blindly — stage named files only, so the workaround is
  never swept into a commit.
- Source code change workflow — full sequence for every source code change,
  from edit through push:
    1. Edit source
    2. cd ~/qelectrotech/build_qt6 && make -j8
       (compile check — confirms the edit builds cleanly)
    3. Functional test (run against this pre-commit build)
    4. git commit (named files only — never main.cpp)
    5. cd ~/qelectrotech/build_qt6
       cmake .. [see Build environment section > Canonical CMake configure command]
       make -j8
       (cmake .. is required — git_last_commit_sha.cmake runs at configure
       time only, so make alone won't refresh GitRevision)
    6. git push origin qt6_cmake_joshua
- When committing a source code fix, remove the corresponding item
  from "Known remaining issues" in the same CLAUDE.md commit
- Before preparing any PR to upstream/qt6_cmake_joshua, exclude these
  fork-only artifacts — they belong on the fork, not in upstream history:
    - .understand-anything/  (generated knowledge graph, committed in bfc312780)
    - CLAUDE.md  (already gitignored)
  bfc312780 already committed the graph, so gitignoring now will NOT
  un-commit it. Cut the PR from a clean branch off upstream and cherry-pick
  only the genuine source/build fixes onto it (this also drops the
  .gitignore / CLAUDE.md / graph housekeeping commits). Confirm with
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
For each source code fix, record the following alongside the commit entry:
- Token cost: approximate session tokens consumed (visible in Claude Code footer)
- Tool calls: number of search/read/bash calls before fix was proposed
- Graph queries: number of /understand-chat calls used
- First attempt: PASS/FAIL (did the fix compile and test correctly first time?)

Example:
  Token cost: ~12k | Tool calls: 8 | Graph queries: 1 | First attempt: FAIL→PASS

Purpose: quantify Understand-Anything's contribution to fix efficiency.
Useful for sharing with QET devs and the broader Qt6 migration community.

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

## Active branch
`qt6_cmake_joshua` — Qt6 port branch. Do NOT assume Qt5 APIs are
available. This branch targets Qt 6.x with KF6.

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

## CLAUDE.md management
This file lives in a separate repo: ~/claude-contexts/qelectrotech/CLAUDE.md
It is symlinked into the QET project root:
  ln -s ~/claude-contexts/qelectrotech/CLAUDE.md ~/qelectrotech/CLAUDE.md
CLAUDE.md is listed in QET's .gitignore — it is not tracked by the QET repo.
Backed up via: github.com/hairykiwi/claude-contexts

## Dev build setup (one-time, after clean reconfigure)
Symlink source resource directories into the build dir so QET finds
them when run from `build_qt6/`:
  cd ~/qelectrotech/build_qt6
  ln -s ../elements elements
  ln -s ../titleblocks titleblocks
  ln -s ../lang lang

These symlinks are covered by .gitignore (build_qt6/) and must be
recreated after `rm -rf build_qt6/*`.

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

## Fixes already committed
See git log for full details:
  git log upstream/qt6_cmake_joshua..HEAD
To get the current count: git rev-list --count upstream/qt6_cmake_joshua..HEAD
Known remaining issues are listed below — keep in sync with commits.

## Known remaining issues (not yet fixed)
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
- PR target: upstream/qt6_cmake_joshua
- Commit style: short summary line, blank line, bullet detail

## Qt6 migration notes
- qAsConst() → std::as_const() (warnings, not errors)
- QWheelEvent::delta() → angleDelta().y()
- qHash seed: uint → size_t
- QSignalMapper::mapped(T) → mappedInt / mappedObject / mappedString
- Brace-initialised int from qsizetype → static_cast<int>()
- SQLite: QSqlDatabase::exec() deprecated → use QSqlQuery::exec()
- QFont weight: Qt6 requires 1–1000 (Qt5 serialised Thin as 0, which Qt6
  rejects). Remap font weight <1 → Thin (100) at load via
  QETApp::sanitizeFontString(); read-time only, never written back.
- QDomDocument: device must be open before passing to setContent()
