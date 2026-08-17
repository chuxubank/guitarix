# AGENTS.md

## Cursor Cloud specific instructions

Guitarix is a modular virtual guitar amplifier for Linux. It is a single C/C++
codebase under `trunk/` built with **Waf** (`trunk/waf` + `trunk/wscript`, Python 3).
It produces the standalone JACK/GTK application (`guitarix`) plus a suite of LV2
plugins. There is no JS/web package manager, no database, and no automated test or
lint suite — CI (`.github/workflows/build.yml`) only compiles and packages.

### Build / lint / test / run

| Task | Command (run from `trunk/`) | Notes |
| --- | --- | --- |
| Configure | `python3 ./waf configure --prefix=/usr --includeresampler --includeconvolver` | Same flags as CI. Add `--optimization` for a tuned build (see `README.md`). |
| Build | `python3 ./waf build` | ~1000+ compile tasks; takes a few minutes. Binary lands at `trunk/build/src/gx_head/guitarix`. |
| Install | `sudo python3 ./waf install` | Needed before running the GUI (see caveat below). |
| Lint | — | No linter is configured; source style is documented in `trunk/README.developers` only. |
| Test | — | No automated test suite. `python3 ./waf distcheck` only re-verifies a source tarball compiles. |

### Non-obvious caveats

- **Submodules are mandatory for the build.** `trunk/src/NAM/NeuralAmpModelerCore`,
  `trunk/src/RTNeural/RTNeural` (and their nested submodules) must be checked out or
  `waf configure`/`build` will fail. The startup update script runs
  `git submodule update --init --recursive`.
- **Running the GUI requires installed resources.** The compiled binary hard-codes
  resource paths under `/usr/share/gx_head` (skins, factory settings, sounds). Running
  straight from `trunk/build/...` will not find its skin. Run `sudo python3 ./waf install`
  first, then launch `guitarix`.
- **JACK is required at runtime.** Start a dummy backend for headless VMs:
  `jackd -r -d dummy -r 48000 -p 1024`. The `Cannot lock down ... memory` /
  `mlockall failed` / "Cannot create RT messagebuffer thread" warnings are expected in a
  container (no realtime scheduling / memory locking) and are non-fatal.
- **GUI startup dialog.** On launch the GUI shows a `GUITARIX ERROR: mlockall failed`
  dialog — just close it; the app works normally. Launch the GUI on the desktop display
  with `DISPLAY=:1 guitarix`.
- **Headless engine + JSON-RPC.** `guitarix -N -p 7000` runs the engine with no GUI and
  exposes a JSON-RPC 2.0 API over TCP port 7000 (`get`, `set`, `banks`, `pluginlist`,
  `getversion`, ...). It registers JACK clients `gx_head_amp` / `gx_head_fx`. Note:
  `get_parameter_value` returns the live audio-thread value (0 with the dummy driver and
  no signal); use `get`/`get_parameter` to read the stored parameter value instead. Do not
  run the headless engine and the standalone GUI at the same time — they use the same JACK
  client names and will conflict.
- **Split GUI/engine dev mode** (from `trunk/README.developers`): terminal 1
  `guitarix -N`, terminal 2 `guitarix -G -H localhost`.
