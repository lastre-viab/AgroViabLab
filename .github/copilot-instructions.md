# AgroViabLab — Copilot Instructions

## Project Overview

**AgroViabLab** is a C++17 numerical library that extends **ViabLab** (VIABLAB) for solving mathematical viability problems in agro-economical systems. It was developed by Anna DESILLES (LASTRE / LC2S - UMR CNRS 8053). The library computes viability kernels, capture basins, and optimal trajectories for controlled dynamical systems under state constraints, with a focus on farm planning and ecological modelling.

License: GNU Affero General Public License v3 (AGPL-3.0).

---

## Repository Layout

```
AgroViabLab/
├── .github/
│   ├── copilot-instructions.md   ← this file
│   └── workflows/
│       └── copilot-setup-steps.yml
├── INPUT/                        ← JSON parameter files (one per model)
├── source/
│   ├── CMakeLists.txt            ← single CMake file for everything
│   ├── include/                  ← all public C++ headers (.h)
│   ├── src/                      ← core library source files (.cpp) + main_main.cpp
│   ├── models/                   ← user model plugins (compiled as .so shared libs)
│   │   ├── agroEcoDiv_params_defs.h   ← AgroViabLab-specific global state header
│   │   ├── dataAgroEcoDivMonoParcelle.cpp
│   │   └── ...
│   └── build/                    ← CMake build directory (out-of-source, gitignored)
├── OUTPUT/                       ← computation results (gitignored)
├── LOG/                          ← spdlog log files (gitignored)
├── notes.org                     ← detailed developer notes in Org-mode (French)
├── Memo.txt                      ← quick reference for enum/param values (French)
└── Notes d'installation de ViabLab sous MAC.txt  ← macOS install walkthrough
```

---

## Build System

**Tool**: CMake ≥ 3.5, C++17 compiler (gcc/g++), ninja or make.

### Dependencies (must be installed before building)
- **Boost** (headers only needed: `property_tree`, `dynamic_bitset`, `foreach`)
- **spdlog** (header-only mode is used: `spdlog_header_only`)
- **OpenMP** (parallelism, `-fopenmp`)
- `libdl` (dynamic loading of model `.so` plugins)

### Configure and Build (Linux)
```bash
# From repository root
cmake -S source -B source/build -DCMAKE_BUILD_TYPE=Release
cmake --build source/build -j$(nproc)
```
The build produces:
- `source/build/agroViabLab` — the main executable
- `source/build/<ModelName>.so` — one shared library per file in `source/models/`

### macOS differences
On macOS, linker flags change (`-flat_namespace,-undefined dynamic_lookup`) and the compiler should be GCC from Homebrew (`gcc-14` / `g++-14`). See `Notes d'installation de ViabLab sous MAC.txt`.

### No test suite
There is no automated test framework. Validation is done by running specific models and checking output files in `OUTPUT/`.

---

## Running the Program

The executable must be run **from the build directory** (it resolves `../INPUT/` and `../LOG/` relative to the working directory):

```bash
cd source/build
./agroViabLab <ModelName>            # loads <ModelName>.so and <paramsFile> from INPUT/
./agroViabLab <ModelName> <params.json>   # override the parameters file
./agroViabLab <ModelName> <params.json> -t <N>  # use N OpenMP threads
```

Example:
```bash
cd source/build
./agroViabLab dataAgroEcoDivMonoParcelle
```

---

## Architecture: How Models Work

Models are **dynamically loaded shared libraries** (`dlopen`/`dlsym`). Each model is a `.cpp` file in `source/models/` that exports C symbols.

### Mandatory exports (all models)
```cpp
extern "C" {
    std::string paramsFile = "MyModel_params.json"; // points to INPUT/ file

    void dynamics(const double *x, const double *u, double *image);
    void jacobian(const double *x, const double *u, double **jacob);
    void localDynBounds(const double *x, double *bound);
}
```

### Optional exports (default no-op implementations exist in `WeakDeclarations.cpp`)
- `constraintsX`, `constraintsXU` — state/control constraints
- `target` — target set for capture basin
- `l`, `m` — cost functions
- `dynamics_fd`, `l_fd`, `constraintsX_fd` — discrete (full-discrete) variants
- `dynamics_tych`, `l_tych`, `m_tych` — tychastic (stochastic disturbance) variants
- `dynamics_hybrid_c`, `dynamics_hybrid_d`, `resetMap_hybrid` — hybrid system variants
- `loadModelData(const ParametersManager*)` — load model-specific data before computation
- `postProcess(const ParametersManager*)` — called after all computations

### AgroViabLab-specific header
Models in `source/models/` typically `#include "agroEcoDiv_params_defs.h"` which declares global variables for farm data management (species names, parcel data, cost coefficients, etc.) via `FarmDataManager`.

---

## JSON Parameter Files (`INPUT/`)

Each model declares a `paramsFile` string pointing to a JSON in `INPUT/`. The JSON has four top-level sections:

### `GRID_PARAMETERS`
| Key | Type | Description |
|-----|------|-------------|
| `STATE_DIMENSION` | int | Number of state variables |
| `STATE_GRID_POINTS` | int[] | Grid resolution per axis |
| `STATE_MIN_VALUES` / `STATE_MAX_VALUES` | float[] | State space bounds |
| `STATE_PERIODIC` | bool[] | Periodic axes |
| `GRID_METHOD` | string | `"BS"` (BitSet), `"MM"` (MicroMacro), `"HBS"` (Hybrid BS), `"HMM"` (Hybrid MM) |
| `OUTPUT_FILE_PREFIX` | string | Prefix for output files |
| `GRID_MAIN_DIR` | int | Main axis for BitSet storage |

### `CONTROL_PARAMETERS`
| Key | Type | Description |
|-----|------|-------------|
| `CONTROL_DIMENSION` | int | Number of control variables |
| `CONTROL_GRID_POINTS` | int[] | Control discretization per axis |
| `CONTROL_MIN_VALUES` / `CONTROL_MAX_VALUES` | float[] | Control bounds |

### `ALGORITHM_PARAMETERS`
| Key | Type | Description |
|-----|------|-------------|
| `SET_TYPE` | string | `"VIAB"` (viability kernel), `"CAPT"` (capture basin), `"VIABG"` (guaranteed viability) |
| `COMPUTE_VIABLE_SET` | bool | If false, loads a previously saved set |
| `GRID_REFINMENTS_NUMBER` | int | Number of grid refinement steps |
| `SAVE_BOUNDARY` / `SAVE_SLICE` / `SAVE_PROJECTION` | bool | What to save after computation |
| `ITERATION_STOP_LEVEL` | int | 0 = run until convergence |

### `SYSTEM_PARAMETERS`
| Key | Type | Description |
|-----|------|-------------|
| `DYNAMICS_TYPE` | string | `"CC"` (continuous), `"DC"` (discrete-time), `"DD"` (full discrete), `"DH"`, `"CH"` (hybrid) |
| `TIME_DISCRETIZATION_SCHEME` | string | `"EL"` (Euler), `"RK2"`, `"RK4"` |
| `LIPSCHITZ_CONSTANT_COMPUTE_METHOD` | string | `"ANALYTICAL"`, `"NUMERICAL_CALC"` |
| `DYN_BOUND_COMPUTE_METHOD` | string | `"ANALYTICAL"`, `"NUMERICAL_CALC"` |

### `TRAJECTORY_PARAMETERS` (list)
| Key | Type | Description |
|-----|------|-------------|
| `TRAJECTORY_TYPE` | string | `"VD"`, `"VL"`, `"OP"`, `"CAUTIOUS"`, `"STOCHASTIC"`, `"WEIGHTED_CONTROLS"`, etc. |
| `INITIAL_POINT` | float[] | Starting state |
| `TRAJECTORY_TIME_HORIZON` | float | Simulation duration |
| `BUBBLE_RADIUS` | float | For CAUTIOUS trajectory type |

---

## Key Classes and Files

| File | Purpose |
|------|---------|
| `src/main_main.cpp` | Entry point: loads model `.so`, reads params, runs viability/trajectory algorithms |
| `include/ViabProblemFactory.h/.cpp` | Factory that instantiates the right `Viabi` subclass based on `GRID_METHOD` |
| `include/Viabi.h` | Abstract base class for viability algorithms |
| `include/ViabiBitSet.h/.cpp` | BitSet-based viability kernel (most common, `GRID_METHOD=BS`) |
| `include/ViabiMicroMacro.h/.cpp` | MicroMacro epigraphical algorithm (`GRID_METHOD=MM`) |
| `include/ParametersManager.h/.cpp` | Reads JSON params, provides typed accessors |
| `include/FarmDataManager.h/.cpp` | Loads farm/crop data from CSV/text files for AgroViabLab models |
| `include/Grid.h` / `GridBitSet.h` | Grid representation and indexing |
| `include/SysDyn.h` / `SimpleSysDyn.h` | Dynamical system wrappers around the user-supplied functions |
| `include/TrajectorySimulation.h` | Trajectory computation |
| `include/Enums.h` | All enum definitions (`TypeTraj`, `SetType`, `GridMethod`, `DynType`, etc.) |
| `include/Params.h` | All parameter struct definitions (`gridParams`, `controlParams`, `systemParams`, `algoViabiParams`, `trajectoryParams`) |
| `include/defs.h` | Common includes, global typedefs, small utility functions |
| `include/WeakDeclarations.h/.cpp` | Default (no-op) implementations of optional model functions |
| `models/agroEcoDiv_params_defs.h` | Global variable declarations for AgroViabLab model plugins |

---

## Code Conventions

- **Language**: Code is C++17. Comments and some variable names are in **French** (this is normal and expected). Do not translate French comments.
- **Naming**: Member variables use `UPPER_SNAKE_CASE` in structs (from JSON keys). Local variables and functions use `camelCase` or `snake_case`.
- **Headers**: Include guards use `#ifndef FOO_H_ / #define FOO_H_` style.
- **Logging**: Use `spdlog::info(...)`, `spdlog::warn(...)`, `spdlog::error(...)`, `spdlog::debug(...)`. Do not use `cout` for new diagnostic messages.
- **Error handling**: Errors are logged with `spdlog::error` then the program either returns non-zero or calls `std::exit(1)`.
- **OpenMP**: Parallelism is handled via OpenMP pragmas and the thread count is passed via command-line (`-t N`) or defaulted to 1.
- **Dynamic loading**: Model symbols are loaded via `dlsym`. Optional symbols fall back to no-op implementations declared in `WeakDeclarations.h`.
- **Encoding**: Some older source files have mojibake (garbled French characters from Latin-1). Do not attempt to fix encoding in unrelated files.

---

## Adding a New Model

1. Create `source/models/MyModel.cpp` with `extern "C"` exports for at least `paramsFile`, `dynamics`, `jacobian`, `localDynBounds`.
2. Create `INPUT/MyModel_params.json` with the four required sections.
3. Rebuild (`cmake --build source/build`). CMake automatically discovers all `.cpp` files in `source/models/` and compiles each as a separate `.so`.
4. Run: `cd source/build && ./agroViabLab MyModel`.

No CMakeLists.txt modification is needed when adding a model.

---

## Known Issues / Workarounds

- **`LOG/` directory must exist** before running; otherwise spdlog will fail to create the log file. Create it manually: `mkdir -p source/build/../LOG` (i.e., `LOG/` at the repo root when running from `source/build`).
- **`OUTPUT/` directory must exist** for results to be saved. Create it: `mkdir -p OUTPUT` at repo root.
- **macOS linking**: Uses `-flat_namespace,-undefined dynamic_lookup` instead of `-Wl,--no-as-needed`. The CMakeLists.txt handles this via `IF(CMAKE_SYSTEM_NAME STREQUAL Darwin)`.
- **spdlog header-only**: The CMakeLists.txt links `spdlog::spdlog_header_only`. If your system only provides compiled spdlog, change this to `spdlog::spdlog`.
- **Encoding**: Some `.cpp`/`.h` files in `source/` contain Latin-1 encoded French characters that display as mojibake. This is cosmetic only.

---

## Common Workflow

```bash
# 1. Install dependencies (Ubuntu/Debian)
sudo apt-get install -y cmake g++ libboost-dev libspdlog-dev

# 2. Prepare required output directories
mkdir -p LOG OUTPUT

# 3. Configure (first time only)
cmake -S source -B source/build

# 4. Build
cmake --build source/build -j$(nproc)

# 5. Run a model
cd source/build
./agroViabLab dataAgroEcoDivMonoParcelle
```
