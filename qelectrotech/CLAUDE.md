# QElectroTech — Claude Code Context

## Claude Code instructions
- Always show commit messages for review before committing
- Never commit sources/main.cpp — contains a temporary workaround

## Testing requirements
Before pushing any commit to origin, Claude must propose and the developer
must perform a brief functional test relevant to the change:

- Test format: numbered steps, clear ✅ expected / ❌ before-fix outcomes
- Scope: targeted to the specific behaviour changed — not a full regression suite
- Result: PASS or FAIL recorded in the session before git push proceeds
- If FAIL: fix before pushing, do not push broken commits

This applies to all source code commits. CMake, documentation, and
tooling-only commits may be pushed without a functional test at discretion.

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
- CMake configure command:
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

## Fixes already committed to fork
### Commit 1: macOS/Apple Silicon fixes for Qt6 build on Homebrew
- `CMakeLists.txt`: PACKAGE_TESTS=OFF, two empty-destination install
  lines commented out
- `cmake/fetch_kdeaddons.cmake`: ECM_SKIP_QDOC_GENERATION=ON added
- `sources/titleblock/helpercell.h`: explicit QGraphicsLayoutItem
  include added for macOS MOC
- `sources/projectview.h`: QWheelEvent::delta() → angleDelta().y()
- `sources/TerminalStrip/ui/terminalstripmodel.h`: qHash uint → size_t
- `sources/editor/elementscene.cpp`: static_cast<int> for qsizetype
- `sources/TerminalStrip/terminalstrip.cpp`: static_cast<int> for
  qsizetype

### Commit 2: Fix QSignalMapper::mapped() deprecation for Qt6
- exportdialog.cpp: SIGNAL(mapped(int)) → mappedInt (6 occurrences)
- qetdiagrameditor.cpp: mapped(QWidget*) → mappedObject + lambda cast
- qetapp.cpp: same pattern as qetdiagrameditor.cpp
- titleblock/templatelogomanager.cpp: mapped(int) → mappedInt

### Commit 3: Fix mangled .gitignore entry for build_qt6/
- `.gitignore`: `!doc/doc-utilsbuild_qt6/` was two entries accidentally
  merged; corrected to separate lines `!doc/doc-utils` and `build_qt6/`

### Commit 4: Fix dev build resource paths on macOS (Debug builds)
- `cmake/paths_compilation_installation.cmake`: Apple Debug builds now
  set QET_COMMON_COLLECTION_PATH/QET_COMMON_TBT_PATH/QET_LANG_PATH
  relative to binary ("elements/", "titleblocks/", "lang/") instead
  of the bundle-relative "../Resources/" paths used by Release builds.
  Adds QET_COMMON_COLLECTION_PATH_RELATIVE_TO_BINARY_PATH_VAR and
  QET_LANG_PATH_RELATIVE_TO_BINARY_PATH_VAR CMake flags to signal
  the relative-path mode.
- `cmake/define_definitions.cmake`: Guards added around lines 27-41 so
  that when the _VAR flags are set, INSTALL_PREFIX (/usr/local/) is
  not prepended to the resource paths. Release and Linux builds are
  unchanged.

Verified: 8598 elements load, EN language resolves, titleblocks visible.

### Commit 5: Fix silent signal/slot connection failures on BorderTitleBlock
- `sources/diagramview.cpp`: replace non-existent SIGNAL(diagramTitleChanged)
  with &BorderTitleBlock::informationChanged (new-style); pre-existing bug
  masked by old SIGNAL() macro syntax — window title now updates when
  titleblock fields change
- `sources/bordertitleblock.h/.cpp`: add setAutoPageNum(const QString &) public
  slot (stores btb_auto_page_num_, emits needFolioData())
- `sources/autoNum/ui/autonumberingdockwidget.cpp`: replace
  SLOT(slot_setAutoPageNum(QString)) with &BorderTitleBlock::setAutoPageNum
  using qOverload<QString> to disambiguate the overloaded signal

## Known remaining issues (not yet fixed)
- `sources/main.cpp`: app name temporarily set to "QElectroTech-Qt6"
  to avoid SingleApplication conflict with Qt5 build — revert before PR
- `QFont::setWeight: Weight must be between 1 and 1000, attempted to
  set 0` — Qt6 font weight range changed
- `The requested buffer size is too big` — Qt6 SVG renderer stricter
- `qAsConst()` → `std::as_const()` warnings throughout
- Display menu missing entries for Collections and Templates windows
  (found via Help → Search "Collections" as workaround) — pre-existing
  Qt6 migration issue
- `QDomDocument called with unopened QIODevice` — Qt6 stricter I/O,
  seen when loading titleblock templates
- `QWidget::setLayout: Attempting to set QLayout on StyleEditor which
  already has a layout` — seen in element editor, Qt6 migration issue
- `QBoxLayout::insert: index out of range` — seen on new diagram creation

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
- QFont weight: Qt6 requires 1-1000 range (ThinLight=25 not 0)
- QDomDocument: device must be open before passing to setContent()
