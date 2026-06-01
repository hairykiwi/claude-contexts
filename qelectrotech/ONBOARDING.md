# QElectroTech — Developer Onboarding Guide

## Project Overview

**QElectroTech (QET)** is an open-source Qt6/C++ application for industrial electrical schematic documentation targeting IEC 60617 wiring diagrams, terminal strips, and multi-folio project documentation. It is **not** a PCB tool.

| | |
|---|---|
| **Languages** | C++, CMake, XML, Markdown, Shell, YAML |
| **Frameworks** | Qt 6.x, KF6 (KDE Frameworks 6), pugixml |
| **Build system** | CMake with FetchContent for KF6 |
| **CI** | GitHub Actions |
| **Platform targets** | Linux, macOS, Windows |

The Qt6 port lives on the `qt6_cmake_joshua` branch. Qt5 APIs are not available on this branch — assume Qt6 throughout.

---

## Architecture Layers

The codebase is organised into 8 logical layers, outermost to innermost:

### 1. Application Shell (21 nodes)
Bootstrap and lifecycle. Entry point → application singleton → global state. `main.cpp` calls `QETApp`, which manages paths, Qt resource loading, and `QSettings`.

Key files: `sources/main.cpp`, `sources/qetapp.cpp`, `sources/qetapp.h`

### 2. Diagram Core (34 nodes)
The document model. A `QETProject` owns one or more `Diagram`s (folios). Serialisation to/from `.qet` XML lives here.

Key files: `sources/qetproject.cpp`, `sources/diagram.cpp`, `sources/diagramposition.cpp`

### 3. Graphics Items (56 nodes)
Everything painted on the `QGraphicsScene`. Elements, conductors, texts, and connectors all derive from `QGraphicsItem` subclasses here.

Key files: `sources/conductor.cpp`, `sources/element.cpp`, `sources/dynamicelementtextitem.cpp`, `sources/conductortextitem.cpp`

### 4. Element Editor (89 nodes)
A standalone editor for `.elmt` element definition files. Has its own scene, toolbox, and undo stack, entirely separate from the diagram editor.

Key files: `sources/editor/qetelementeditor.cpp`, `sources/editor/elementscene.cpp`, `sources/editor/elementview.cpp`

### 5. Domain Features (154 nodes)
High-level electrical domain logic: auto-numbering of wires/folio references, terminal strips, title-block templates, and the cross-reference system.

Key files: `sources/autoNum/`, `sources/TerminalStrip/`, `sources/titleblock/`

### 6. UI Forms & Dialogs (150 nodes)
`MainWindow`, dock widgets, property dialogs, and `.ui`-generated form classes. Thin layer — mostly wires signals to domain/core objects.

Key files: `sources/mainwindow.cpp`, `sources/diagramview.cpp`, `sources/ui/`

### 7. Shared Utilities (142 nodes)
Cross-cutting helpers: XML utilities, custom Qt containers, `QET` namespace constants, logging, and string helpers.

Key files: `sources/qet.h`, `sources/qetxml.cpp`, `sources/elementslocation.cpp`, `sources/qetgraphicsitem/`

### 8. Project Support (216 nodes)
Build scripts, CI configuration, element library XML, title-block templates, translation files, and icon resources. No runtime C++ here.

Key files: `CMakeLists.txt`, `.github/workflows/`, `elements/`, `titleblocks/`, `lang/`

---

## Key Concepts

**Folio / Diagram**: QET's term for a single schematic sheet. One `.qet` project file contains one or more folios.

**Element**: A reusable symbol (motor, relay contact, terminal, etc.) stored as an `.elmt` XML file. The element library ships in `elements/` and is also user-extensible.

**Conductor**: A wire drawn between element terminals. Conductors carry numbering metadata and participate in cross-references.

**Terminal Strip**: A physical connector block modelled as a domain object. Has its own UI (`TerminalStrip/`) separate from the diagram editor.

**Title Block**: The information panel at the bottom of each folio. Templates are XML files in `titleblocks/`. The default ships as a Qt resource (`:/titleblocks/default.titleblock`).

**Event Strategy**: Diagram mouse interaction uses a Strategy pattern (`DiagramEvent*` classes) rather than a state machine. The active strategy handles mouse press/move/release.

**Undo/Redo**: Every diagram mutation is a `QUndoCommand` subclass. The undo stack lives on `Diagram`, not `MainWindow`.

**Auto-numbering**: Wires, folio cross-references, and element identifiers can all be auto-numbered via configurable formulas in `sources/autoNum/`.

**XML serialisation**: `.qet` project files are QDom-based. Element definitions use pugixml for faster batch loading. Both XML subsystems coexist — do not mix them.

---

## Guided Tour

Work through these 17 steps to build a mental map of the codebase:

| Step | Topic | Starting point |
|------|-------|---------------|
| 1 | Project README & goals | `README.md` |
| 2 | Application entry point | `sources/main.cpp` |
| 3 | Application singleton & global state | `sources/qetapp.cpp` |
| 4 | Global constants & type aliases | `sources/qet.h` |
| 5 | Project document model | `sources/qetproject.cpp` |
| 6 | Diagram (folio) model | `sources/diagram.cpp` |
| 7 | Main window & diagram view | `sources/mainwindow.cpp`, `sources/diagramview.cpp` |
| 8 | Graphics item hierarchy | `sources/element.cpp`, `sources/conductor.cpp` |
| 9 | Dynamic element text rendering | `sources/dynamicelementtextitem.cpp` |
| 10 | Mouse interaction strategies | `sources/diagramevent*.cpp` |
| 11 | Undo/redo commands | `sources/undocommand/` |
| 12 | Element editor | `sources/editor/qetelementeditor.cpp` |
| 13 | Element library & locations | `sources/ElementsCollection/`, `sources/elementslocation.cpp` |
| 14 | Auto-numbering & title blocks | `sources/autoNum/`, `sources/titleblock/` |
| 15 | Terminal strips | `sources/TerminalStrip/` |
| 16 | XML serialisation utilities | `sources/qetxml.cpp` |
| 17 | Build system & CI | `CMakeLists.txt`, `.github/workflows/` |

---

## File Map

### Application Shell
| File | Purpose |
|------|---------|
| `sources/main.cpp` | `main()`, SingleApplication setup, QETApp launch |
| `sources/qetapp.cpp/h` | Application singleton: paths, resources, settings, font sanitisation |

### Diagram Core
| File | Purpose |
|------|---------|
| `sources/qetproject.cpp/h` | `.qet` project container; owns Diagram list and XML root |
| `sources/diagram.cpp/h` | QGraphicsScene subclass representing one folio |
| `sources/diagramposition.cpp/h` | Column/row coordinate system for folio grid |

### Graphics Items
| File | Purpose |
|------|---------|
| `sources/element.cpp/h` | Base class for placed elements |
| `sources/conductor.cpp/h` | Wire connecting two terminals |
| `sources/dynamicelementtextitem.cpp/h` | Formula-driven text bound to an element |
| `sources/conductortextitem.cpp/h` | Wire number label |

### Element Editor
| File | Purpose |
|------|---------|
| `sources/editor/qetelementeditor.cpp/h` | Top-level editor window |
| `sources/editor/elementscene.cpp/h` | Editor's own QGraphicsScene |
| `sources/editor/elementview.cpp/h` | Viewport for the element editor scene |

### Domain Features
| File | Purpose |
|------|---------|
| `sources/autoNum/` | Auto-numbering formulas and logic |
| `sources/TerminalStrip/` | Terminal strip domain model and UI |
| `sources/titleblock/templatescollection.cpp` | Manages title-block template files |
| `sources/titleblocktemplate.cpp/h` | Parses and renders a single title-block template |

### UI Forms & Dialogs
| File | Purpose |
|------|---------|
| `sources/mainwindow.cpp/h` | Application main window |
| `sources/diagramview.cpp/h` | QGraphicsView with zoom, pan, and context menus |

### Shared Utilities
| File | Purpose |
|------|---------|
| `sources/qet.h` | Global constants, enums, and type aliases |
| `sources/qetxml.cpp/h` | XML file I/O helpers (QDom-based) |
| `sources/elementslocation.cpp/h` | Resolves element paths (filesystem, Qt resource, collection) |

---

## Complexity Hotspots

Approach these files carefully — they are the densest and most interconnected in the codebase:

| File | Layer | Why complex |
|------|-------|-------------|
| `sources/editor/qetelementeditor.cpp` | Element Editor | Largest single file; owns editor lifecycle, toolbar, undo, and file I/O |
| `sources/diagram.cpp` | Diagram Core | Central document object; touched by almost every other subsystem |
| `sources/mainwindow.cpp` | UI Forms | Wires every major feature together; high signal/slot density |
| `sources/qetproject.cpp` | Diagram Core | XML serialisation of entire project tree; error recovery logic |
| `sources/conductor.cpp` | Graphics Items | Path computation, segment splitting, terminal snapping |
| `sources/dynamicelementtextitem.cpp` | Graphics Items | Formula evaluation, binding to element properties, Qt6 text layout |
| `sources/ElementsCollection/elementslocation.cpp` | Shared Utilities | Resolves paths across three backends (filesystem, resource, collection) |
| `sources/titleblock/templatescollection.cpp` | Domain Features | Manages file and resource template registries simultaneously |

---

## Qt6 Migration Notes

This branch is a Qt6 port. Common patterns to recognise when reading or fixing code:

- `qAsConst(x)` → `std::as_const(x)` (Qt5 helper removed in Qt6)
- `QWheelEvent::delta()` → `angleDelta().y()`
- `QFont` weight must be 1–1000; Qt5 serialised Thin as 0 (remapped at load in `QETApp::sanitizeFontString()`)
- `QDomDocument::setContent(QIODevice*)` requires the device to be **open** before the call; Qt5 silently failed, Qt6 logs a warning
- `QSqlDatabase::exec()` deprecated → use `QSqlQuery::exec()`
- `qHash` seed parameter: `uint` → `size_t`

---

*Generated from Understand-Anything knowledge graph (graph commit `28bfb177`, 1585 nodes / 2945 edges, 864 analysed files). THIS document is not auto-updated. It reflects the QET codebase state only at time of generation: expect the information it contains to drift over time.*
