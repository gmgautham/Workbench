# Unified Engineering Workbench

A small, tested engineering workbench built for the PASS Science Works R&D internship assignment (2026-2027). A **C++17 backend** does all computation and file processing, and a dependency-free **browser UI** presents the results.

| View | What it does |
| --- | --- |
| **Geometry** | Loads an ASCII STL mesh and a nodal temperature CSV; 3D rotate / pan / zoom / fit-to-view, colour legend, node inspection |
| **Thermal Analysis** | 1D steady-state finite-element solver for a heated rod, compared with the analytical solution, with mesh convergence and CSV export |
| **Performance** | Serial vs. multithreaded batch of independent thermal solves, with speedup, efficiency and an agreement check |
| **Hardware** | **Simulated** microcontroller, FPGA and JTAG adapter: discovery, capabilities, connect/disconnect, telemetry and a recoverable timeout |

> **Note:** all device behaviour is simulated. No hardware is connected or touched. The sample temperature CSV is **synthetic**, not a real ANSYS export.

## Quick start (Windows 11 x64)

The repository contains source only (executables are git-ignored). Build it as below, or use the packaged `Launch.exe` from the submission ZIP.

**Packaged app:** extract the ZIP, double-click `Launch.exe`, and the app opens in your browser. Click OK in the small message box to stop it. No terminal, installer, admin rights or internet is needed. Windows SmartScreen may warn because the executable is unsigned: choose *More info*, then *Run anyway*.

**From source** (Visual Studio 2022 Build Tools with the C++ workload). Open **x64 Native Tools Command Prompt for VS 2022** in the project folder:

```
cl /EHsc /std:c++17 /O2 /MT src\app\server.cpp /Fe:workbench_server.exe
workbench_server.exe --open
```

This serves `web/` and `data/` at `http://127.0.0.1:8080/`. Run it from the project root. Options: `--port N`, `--open`.

To rebuild the launcher:

```
cl /EHsc /std:c++17 /O2 /MT /DUNICODE /D_UNICODE src\app\launcher.cpp /Fe:Launch.exe /link /SUBSYSTEM:WINDOWS
```

Put `Launch.exe`, `workbench_server.exe`, `web/` and `data/` in one folder to package it.

**VS Code:** open the folder, install the C/C++ extension (`ms-vscode.cpptools`), open Run and Debug (`Ctrl+Shift+D`), choose *Run packaged Workbench (Launch.exe)* and press F5. This expects the packaged files in `dist/Workbench/`.

## Tests

Each test is a standalone program. Build and run from the project root:

```
cl /EHsc /std:c++17 /O2 tests\test_fem.cpp /Fe:test_fem.exe
test_fem.exe
cl /EHsc /std:c++17 /O2 tests\test_parsers.cpp /Fe:test_parsers.exe
test_parsers.exe
cl /EHsc /std:c++17 /O2 tests\test_devices.cpp /Fe:test_devices.exe
test_devices.exe
```

Each should print `ALL ... TESTS PASSED`. Other tools:

| Tool | Command | Output |
| --- | --- | --- |
| FEM convergence | `convergence.exe` | `results/convergence.csv`, `results/profile_N*.csv` |
| Backprop + gradient check | `backprop.exe` | gradients, finite-difference check, `results/backprop_loss.csv` |
| Batch benchmark | `bench.exe [cases] [elements]` | `results/timing.csv` |
| File checker / generator | `inspect.exe stl\|csv <file>` | summary or a line-numbered error |
| Device demo | `devices_demo.exe` | simulated discovery, timeout and recovery |

A `CMakeLists.txt` that builds the same targets is included (`cmake -S . -B build`, `cmake --build build --config Release`, `ctest --test-dir build -C Release`). Development used the direct `cl` commands above.

## Architecture

```
src/core/     header-only C++17 core (no UI or network code)
  fem1d.hpp     two-node linear 1D thermal FEM, midpoint error
  batch.hpp     seeded batch generator, serial and thread-pool solvers
  stl.hpp       ASCII STL parser with validation
  nodes_csv.hpp nodal temperature CSV parser with validation
  io.hpp        file reading, size limit, finite-number checks
  devices.hpp   SIMULATED MCU / FPGA / JTAG adapters
src/app/      executables
  server.cpp    HTTP/JSON wrapper on 127.0.0.1 (cpp-httplib)
  launcher.cpp  Launch.exe: starts the server, opens the browser
  convergence.cpp, backprop.cpp, bench.cpp, inspect.cpp, devices_demo.cpp
web/          browser UI (single HTML file, no external libraries)
tests/        FEM, parser and device tests
data/         sample STL, synthetic CSV
docs/         data provenance, embedded design
results/      CSV outputs
third_party/  httplib.h (cpp-httplib)
```

The server contains no numerical logic. It validates parameters, calls the core, and converts every exception into a JSON `{"error": ...}` response that the UI shows in its status bar.

### HTTP API

| Method and path | Purpose |
| --- | --- |
| `GET /api/fem?N=` | FE solution for N elements (1-100000) |
| `GET /api/convergence` | N = 10, 20, 40, 80 |
| `GET /api/bench?count=&N=` | Serial vs. 1/2/4-worker benchmark (count x N at most 40,000,000) |
| `POST /api/stl`, `POST /api/csv` | Parse and validate an uploaded file body |
| `GET /api/devices`, `POST /api/devices/{inject,connect,disconnect}`, `GET /api/devices/telemetry` | Simulated devices |

## Thermal FEA (Task C)

Solves `-d/dx(kA dT/dx) = qA` with L = 1 m, A = 1e-4 m², k = 10 W/(m K), q = 10,000 W/m³, T(0) = 100 C, T(L) = 20 C, using two-node linear elements. Boundary conditions are imposed by row replacement. Exact solution: `T = 100 - 80x/L + [q/(2k)]x(L-x)`.

| N | Max nodal error (C) | Max midpoint error (C) | Observed order |
| --- | --- | --- | --- |
| 10 | 5.7e-14 | 1.25 | - |
| 20 | 2.6e-13 | 0.3125 | 2.00 |
| 40 | 7.7e-13 | 0.078125 | 2.00 |
| 80 | 1.1e-12 | 0.01953125 | 2.00 |

Nodal values are exact to rounding. The error appears between nodes and equals `125 h²`, because a piecewise-linear solution cannot follow the parabola.

## Parallel performance (Task D)

`solveParallel` starts a bounded pool of `std::thread` workers that take case indices from an atomic counter. Each case writes only its own output slot, so results are independent and in input order. Serial and parallel runs agree exactly (difference 0 C, limit 1e-8 C).

Measured on an Intel Core i7-1360P (12 cores, 16 threads, hybrid), 16 GB RAM, MSVC 19.44, `/O2`; 1 warm-up and 5 runs, median; 500 cases x 5,000 elements, seed 42:

| Workers | Median (s) | Speedup (T1/Tp) | Efficiency |
| --- | --- | --- | --- |
| 1 | 0.1453 | 1.00 | 1.00 |
| 2 | 0.0807 | 1.80 | 0.90 |
| 4 | 0.0409 | 3.56 | 0.89 |

Timings on a hybrid laptop CPU vary between runs. One earlier run (2,000 cases x 20,000 elements) showed an unexplained slow 1-worker result; see the report.

## Supported file formats

- **STL:** ASCII only (metres by convention). Binary, truncated and empty files are rejected with a clear message.
- **CSV:** header must be exactly `node_id,x_m,y_m,z_m,temperature_C` (metres, degrees C). Rejected: `nan`/`inf`, wrong column count, non-integer or duplicate `node_id`, empty file.
- Limit: 200 MB per file. Tested: a generated 1,000,000-triangle STL parsed in 2.64 s and a 2,000,000-node CSV in 2.68 s. The browser draws at most 100,000 triangles and 200,000 nodes (subsampled).
- Sample data and provenance: [`docs/data_notes.md`](docs/data_notes.md).

## Embedded design (Task E)

A host-connected design for one microcontroller board, one FPGA board and one JTAG adapter, with voltage domains, grounding, level shifting and JTAG signal ownership, is in [`docs/embedded_design.md`](docs/embedded_design.md). A JTAG adapter is a debug interface, not an interchangeable compute board. Pins and voltages are design assumptions, not verified against specific boards.

## Backpropagation (Task F)

`backprop.cpp` computes the gradients of `J = 0.5 (v tanh(wx+b) + c - y)²` by hand-derived chain rule (no automatic differentiation) and checks each against central finite differences (epsilon 1e-6, largest error 1.1e-10, limit 1e-5). With x = 1, y = 2, w = 0.5, b = 0, v = 1, c = 0 and learning rate 0.1, one update gives w = 0.620946, b = 0.120946, v = 1.071068, c = 0.153788. The loss falls from 1.1825 to 1.7e-20 over 100 further steps.

## Known limitations

- The Windows package was tested by double-clicking on the development PC only. No clean-machine test and no launch-to-ready timing yet.
- The executables are unsigned, so SmartScreen may warn.
- Port 8080 is fixed. If another program uses it, the launcher shows an error. If the app is already running, `Launch.exe` reuses it and its OK button does not stop that server.
- The benchmark runs to completion (no mid-run cancel); a second request is rejected while one is running.
- Large meshes are subsampled and slow to rotate. Binary STL is not supported. No mesh-to-result registration.
- All devices are simulated. No wiring, firmware or FPGA programming was done.
- The CMake build is provided, but development used direct `cl` builds.

## Dependencies and licences

- [cpp-httplib](https://github.com/yhirose/cpp-httplib) v0.18.3 (MIT licence), single header in `third_party/httplib.h`.
- No other third-party code. The UI uses no external libraries or network requests.

## AI use

This project was built with AI assistance (Claude, Anthropic). All code was run, tested and committed by the author. Mistakes found in the AI's output, such as a wrong hand-computed gradient value and a missing `#include`, are documented in the report's AI-use log.

## Author

Gautham Mourya
