# CLAUDE.md

Guidance for Claude Code in this repository.

## What this is

**SimulIDE 1.1.0-SR1**, a real-time electronic circuit simulator with microcontroller emulation (AVR/Arduino, PIC, 8051, 6502, Z80) and a built-in code editor and debugger. It is a C++/Qt5 desktop app built with qmake. Its models favor speed over accuracy (see `README.md`).

`origin` is a GitHub fork. `upstream` is the original project on Launchpad (`https://git.launchpad.net/simulide`). Everything here except `.claude/` comes from upstream.

## Build and run

```bash
cd build_XX
qmake              # reads SimulIDE_Build.pro, which includes ../SimulIDE.pro
make -j"$(nproc)"
executables/SimulIDE_1.1.0-SR1/simulide [circuit.sim1]
```

- The build first runs `lrelease`, which compiles `resources/translations/*.ts` into `resources/qm/`. It then copies `resources/data` and `resources/examples` next to the executable, which finds them at run time.
- Outputs go into `build_XX/`: `Makefile`, `.qmake.stash`, `build/` (objects, moc), `executables/`. `.gitignore` does not cover them, so they show up as untracked.
- `SimulIDE.pro` adds `-Ofast` to every build. For a debuggable binary, run `qmake CONFIG+=debug`. qmake appends its debug flags (`-O0`) after `-Ofast`, and GCC obeys the last `-O` it sees.
- There are no automated tests. To check a change, build it and open the circuits it affects: `resources/examples/`, or the `*_test.sim1` circuit that sits next to many parts in `resources/data/`.

## Code principles

Code is simple, minimal, easy to understand, and DRY. These come before cleverness, speculative flexibility, and personal style.

- **Simple:** use the most straightforward approach that works. No clever tricks, premature abstraction, or premature optimization.
- **Minimal:** write only what the task needs. No speculative features, unused options, or "just in case" code. Fewer lines, files, and dependencies win. Delete dead code.
- **Easy to understand:** a newcomer should follow it on first read. Clear names, small functions, shallow nesting, obvious control flow.
- **DRY:** one source of truth for each piece of logic, data, and config. Search for existing code to reuse before writing new code; never copy-paste logic. Extract it once instead.
- **When in doubt, choose the simpler option.** Before finishing, re-read your diff and cut anything that isn't needed.

## How to explain things

These rules apply to explanations to the user and to code comments, not to commit messages.

For code comments:
- Explain why and the mechanism; don't restate what the code plainly does.
- Define a term once, at its first use in the file.
- Add a worked numeric example only for non-obvious math (scaling, units, bit layouts).
- Keep comments as short as the point allows; "Simple", "Minimal", "Easy to understand", and "DRY" still apply.

Audience: someone with a BS in EE who remembers the fundamentals (circuits, signals, calculus, basic probability and statistics) but not every formula or every field's jargon. Write for a sharp engineer outside the specialty.

Goal: as simple as possible, without saying anything false.

1. Start with the plain-language answer in 1-2 sentences, then the reasoning.
2. Explain the mechanism (what causes what) in words before any math.
3. Define each technical term the first time it appears, in a short phrase. Introduce as few new terms as possible.
4. Use an equation only when it makes the point clearer than words. If you use one, define every symbol and show one worked example with real numbers.
5. Prefer a concrete numeric example over abstract notation.
6. Use an analogy (EE analogies are welcome) only if it maps accurately, and say where it breaks down.
7. When you simplify, say what you left out and when it would matter.
8. No handwaving: back each claim with a reason, a number, or a source. If the answer depends on something, say what. If unsure, say so.
9. Before sending, check: could I follow this on the first read without looking anything up? If not, rewrite it.

## Layout

About 370 `.cpp` and 390 `.h` files (~160k lines), built as a single qmake target.

- `SimulIDE.pro` holds the whole build: version (`VERSION`, `RELEASE`), Qt modules, compiler flags, and `INCLUDEPATH`. It finds sources with a recursive glob of `src/`, so a new `.cpp`, `.h` or `.ui` file builds without editing the `.pro`. A new source directory still has to be added to `INCLUDEPATH`, because code includes headers by bare name (`#include "e-resistor.h"`).
- `build_XX/SimulIDE_Build.pro` is the file `qmake` runs on. It sets `BUILD_DIR` and includes `../SimulIDE.pro`.
- `src/`: all the C++ code.
  - `main.cpp`, `mainwindow.*`: startup, language selection, settings, and lookup of the data folder. A user add-ons folder is searched as well.
  - `simulator/`: the circuit engine, with no widget code. `Simulator` (run loop and event queue), `CircMatrix` (builds and solves the node matrix), `eNode` (one electrical node), `ePin`, and `eElement` (base class of every electrical model). `elements/` holds shared analog models: `e-resistor`, `e-reactive`, `e-coil`, `e-diode`, `e-bjt`, `e-mosfet`, `e-led`, and `e-clocked_device` (clock-input handling for digital parts).
  - `gui/circuitwidget/`: the schematic editor and the part framework.
    - Core classes: `Circuit` (the `QGraphicsScene` that loads and saves `.sim1`), `Component`/`CompBase` (base of every placeable part, plus the property system), `Pin`, `IoPin` (digital pin), `Connector` (wire), `ItemLibrary` (registry of built-in parts), and `Updatable` (refresh hook called at GUI rate).
    - `components/<group>/`: one class per built-in part, grouped as in the parts list: `active`, `logic`, `meters`, `micro` (sensors and modules such as DHT22 and DS1307), `other`, `outputs/{displays,leds,motors}`, `passive/{reactive,resistors,resist_sensors}`, `sources`, `switches`.
    - `properties/`: property types (`DoubProp`, `IntProp`, `StrProp`, …) and the property dialog.
    - `subcircuits/`: parts defined by data files instead of C++. `Chip` → `SubCircuit` → `LogicSubc`, `BoardSubc` → `ShieldSubc` → `ModuleSubc`. `SubPackage` is the "Package" item that draws a subcircuit's pinout.
  - The rest of `gui/`: `editorwidget/` (code editor, compiler runner, and one class per toolchain in `debuggers/`: avr-gcc, Arduino, gputils, SDCC, XC8, GCBasic, …), `dataplotwidget/` (oscilloscope and logic analyzer views), `memory/` (MCU RAM and register monitor), `serial/` (serial port and terminal), `componentselector/` (parts tree), `filebrowser/`, `dialogs/`.
  - `microsim/`: microcontroller emulation.
    - `Mcu` is the part on the schematic, and `eMcu` is the running chip. `McuCreator` builds an `eMcu` from a `.mcu` file. The generic peripherals (`mcutimer`, `mcuuart`, `mcuadc`, `mcuspi`, …) live here too.
    - `cores/<family>/`: each instruction set and the peripherals specific to that family. `avr`; `pic` (12-, 14- and enhanced 14-bit cores); `i51` (8051); `mcs65` (6502); `z80` (including the ZX Spectrum ULA); `scripted` (cores written in AngelScript).
    - `modules/`: SPI, TWI (I²C) and USART protocol logic. MCU peripherals share it, and so do parts such as the serial port and ESP-01. Also `ScriptModule`, the AngelScript support for scripted parts.
  - `angel/`: vendored AngelScript engine and JIT. It is third-party code, so don't edit or restyle it.
  - `icons/`, `application.qrc`: icons, fonts and the compiled translations, embedded in the binary.
- `resources/`: files the build copies next to the executable or embeds.
  - `data/`: part definitions loaded at run time. Each top-level `*.xml` file (`avr.xml`, `ic74.xml`, …) lists entries for the parts list and points into the subfolders:
    - `ICs/<name>/`: a `.sim1` circuit plus a `.package` pinout (`_LS.package` is the logic-symbol variant).
    - `AVR/`, `PIC/`, `MCS51/`, `MCS65/`, `Z80/`: `.mcu` files, which are XML descriptions of the memory map, registers and peripherals.
    - `arduino/`, `shields/`, `scripted/`, `dig_pot/`, `led_display/`, `tools/`: further parts.
    - `codeeditor/`: compiler definitions and syntax highlighting.
    - `help/`: per-part help text in `<name>.txt`, where the name is lowercased with spaces removed. A translation is `_<locale>/<name>_<locale>.txt`.
  - `examples/`: example circuits (`.sim1`). The `Micro/` examples include firmware source and `.hex` files.
  - `translations/`: `.ts` sources. `qm/` is the build output (`resources/qm/*.*` is gitignored).
  - `fonts/`, `icons/`: embedded fonts and the app icons.
- `.claude/` (local, untracked, not from upstream): `CLAUDE.md` (this file), `settings.json`, `skills/wrapup/SKILL.md`, `rules/markdown.md`, `PROGRESS.md`.
  - `settings.json`: `model: opus`, `effortLevel: xhigh`, a permission allowlist, and a `SessionStart` hook that prints `PROGRESS.md` into context. Personal overrides go in the gitignored `settings.local.json`, never in this file.
  - `/wrapup` rewrites `PROGRESS.md` as a handoff covering the work since the previous one (base: the last commit that touched it), then commits. Earlier handoffs: `git log --follow -p -- .claude/PROGRESS.md`.
  - `rules/markdown.md` says how to fold `##` sections in `README.md` and `docs/`. Claude Code loads it by itself when working on those files.
- `README.md` (upstream overview and Qt dependencies), `COPYING` (GPLv3 text), `copyright.txt` (license split: files by Santiago González are AGPLv3, other contributors' code is GPLv3).

## How the code fits together

- **Time** is an integer count of picoseconds (`uint64_t`). Circuits store a step size; `stepSize="1000000"` in a `.sim1` file means 1 µs.
- **Run loop:** a GUI timer fires every 50 ms by default (`1000/fps`). Each tick does three things: it waits for the previous run to finish, calls `updateStep()` on every `Updatable` to repaint meters, LEDs and plots, and then starts `runCircuit()` on a worker thread (`QtConcurrent`). `runCircuit()` advances the circuit by one frame's worth of simulated time.
- **Events:** a model schedules itself with `Simulator::addEvent(time, this)` and receives `runEvent()` at that time. Digital logic, clocks and MCU cores all advance this way, not on a fixed step.
- **Analog solve:** each model writes ("stamps") conductances and currents into its nodes. The engine re-solves only the node groups that changed, using LU factorization. A node group is a set of nodes connected to each other, which can be solved on its own. Then it calls `voltChanged()` on the models that asked for it. Nonlinear models (diodes, transistors) repeat this until the solution converges, up to the circuit's `NLsteps` limit.
- **A part is a GUI class plus an electrical model,** joined by multiple inheritance: `class Resistor : public Comp2Pin, public eResistor`. Lifecycle: `initialize()` and `stamp()` at start, then `runEvent()` and `voltChanged()` while running.
- **Adding a built-in part:** write a class with static `construct()` and `libraryItem()`. Register it in `ItemLibrary::loadItems()` (`src/gui/circuitwidget/itemlibrary.cpp`). Add its icon to `src/icons/components/` and `application.qrc`, and optionally a `resources/data/help/<name>.txt`.
- **Adding an IC or MCU needs no C++** if it can be built from existing parts or peripherals. Add its files under `resources/data/` and an entry in the matching `*.xml`.

## Conventions

- **NEVER `git commit` or `git push` on your own.** Do either only when the user's current message asks for it. Exception: `/wrapup` always commits, never pushes.
- **Line endings are mixed and there is no `.gitattributes`.** `src/` is LF, except some files in `src/angel/`. Under `resources/`, most `.sim1` files are CRLF (697 of 923). So are some `.package`, help `.txt`, `.hex` and `.xml` files, while `.mcu` files are all LF. Keep each file's existing line endings, and never normalize them in bulk.
- **Match upstream style:** 4-space indent, `m_` prefix on members, singletons reached through `Class::self()`, and an `e` or `e-` prefix on electrical-model classes and files (`eResistor` in `e-resistor.h`). New source files start with the same copyright banner as their neighbors; it points to `copyright.txt`.
- New toolchains extend the root `.gitignore` (no parallel ignore files). Keep new entries consistent with the existing ones.
- **Read first, edit once, check once.** Before changing anything, list every file the change touches and read them in one pass. Then make all the edits, then run whatever verifies them once. Don't loop read, edit, rerun per file. If a check fails, fix it and rerun that check.

## Development environment

- **Debian 13 (trixie)** with `docker` installed. Base tooling via `apt`: `git`, `gh`, `python3`, `python3-dev`, `build-essential`, `curl`, `jq`, `wget`; plus `uv` from its standalone installer into `~/.local/bin`.
- **Qt5 build dependencies.** The names below were checked against the Debian 13 archive, but no build has been run with them yet:

  ```bash
  sudo apt install -y qtbase5-dev libqt5svg5-dev qtmultimedia5-dev libqt5serialport5-dev qttools5-dev-tools libqt5multimedia5-plugins
  ```

  `qtbase5-dev` covers Core, Gui, Widgets, Xml and Concurrent, and pulls in `qt5-qmake`, whose `qtchooser` dependency puts `qmake` and `lrelease` on PATH. `qttools5-dev-tools` provides `lrelease`. The multimedia plugins are needed at run time.
- **Install missing tools yourself:** `sudo apt install -y <package>` (`sudo apt update` first if not found). Use `apt` for system tools only. If the project will keep relying on a new tool, add it to the install commands in this section.
- **Python: use `uv` only.** Never run `pip`, `pip3`, `python3 -m pip`, `python3 -m venv`, or `pipx`; `uv pip …` is uv's own subcommand and is fine. uv handles both venvs and installs, and Debian 13 blocks system-wide `pip install` anyway (PEP 668). One venv per project:

  ```bash
  uv sync                                          # pyproject.toml project; add packages with: uv add <pkg>
  uv venv && uv pip install -r requirements.txt    # requirements.txt-only project
  uv run <cmd>                                     # run a command in the project venv
  ```
- `node`/`npm` and .NET are not used.
