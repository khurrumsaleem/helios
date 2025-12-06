# Helios++

### A Continuous-Energy Monte Carlo Reactor Physics Code

---

## What is Helios++?

**Helios++** is a Monte Carlo neutron transport code designed for reactor physics calculations. It simulates the random walk of neutrons through matter, tracking their interactions with nuclei until they are absorbed or leak out of the system.

But Helios++ isn't just another transport code—it's an exploration of how **design patterns** can tame the inherent complexity of Monte Carlo simulations while maintaining performance and extensibility.

---

## The Philosophy

Monte Carlo codes are notoriously complex. Particles interact with geometry. Geometry references materials. Materials contain isotopes. Sources spawn particles in cells. Tallies accumulate results across threads. Everything connects to everything else.

**The problem?** This web of interconnections makes code brittle, testing painful, and extensions risky.

**Our approach:** Apply classical design patterns—*Mediator*, *Factory*, *Policy-based design*—to isolate changes, decouple modules, and make the simulation core untouchable when adding new features.

> *"The goal of design patterns is to isolate changes in the code. This is accomplished by adding layers of abstraction."*

The simulation routine in Helios++ remains **exactly the same** whether you're using macroscopic cross sections or ACE tables. Add a new surface type? The rest of the code never knows. Change the parallelization strategy? Swap a template parameter.

---

## Architecture Overview

```
+------------------------------------------------------------------+
|                        McEnvironment                             |
|                    (Mediator / Director)                         |
+--------------+--------------+--------------+--------------+------+
|   Geometry   |  Materials   |  ACE Module  |    Source    |Tally |
|    Module    |    Module    |              |    Module    |      |
+--------------+--------------+--------------+--------------+------+
| - Cells      | - Materials  | - Isotopes   | - Distrib.   | Keff |
| - Surfaces   | - MacroXs    | - Reactions  | - Samplers   | Rates|
| - Universes  |              | - Grid       | - Sources    |      |
+--------------+--------------+--------------+--------------+------+
                                 ^
                                 | (Parser-independent)
                          +------+------+
                          |   Parser    |
                          |  (XML/API)  |
                          +-------------+
```

### Core Modules

| Module | Purpose |
|--------|---------|
| **Geometry** | Mediator for cells, surfaces, universes, and lattices. Handles recursive universe fills with a single global coordinate system—no frame transformations during tracking. |
| **Materials** | Central registry for materials. Supports both isotopic compositions and macroscopic cross sections. |
| **AceModule** | Reads and manages ACE-format nuclear data tables. Handles energy grid management and reaction law assembly using policy-based design. |
| **Source** | Composes particle distributions. Mix ACE-sampled spectra with custom spatial distributions. Build lattices of sources for keff problems. |
| **Tallies** | Thread-safe accumulators with MPI reduction support. Currently estimates keff via analog, collision, and track-length estimators. |

### Design Patterns in Action

- **Mediator Pattern**: `McEnvironment` orchestrates all inter-module communication
- **Factory Pattern**: Each geometric entity type (surfaces, cells) has its own factory
- **Policy-based Design**: Parallelism (OpenMP/TBB/Serial) and ACE reaction laws are compile-time policies
- **Prototype Pattern**: Tallies create thread-local clones for lock-free accumulation

---

## Project Structure

```
helios/
├── Common/              # Shared types, logging, random numbers (TRNG)
├── Environment/         # McEnvironment, McModule, Settings, Simulation
│   └── Simulation/      # Parallel simulation policies (OpenMP, TBB)
├── Geometry/            # Cells, Surfaces, Universes, geometric tracking
│   └── Surfaces/        # Cylinder, Sphere, Plane implementations
├── Material/            # Materials, Isotopes, cross sections
│   ├── AceTable/        # ACE format reader and reaction samplers
│   │   ├── AceReader/   # Low-level ACE file parsing
│   │   └── AceReaction/ # Energy/angle distribution samplers
│   ├── Grid/            # Master energy grid management
│   └── MacroXs/         # Macroscopic cross section support
├── Parser/              # Parser interface
│   └── XMLParser/       # XML input parser (OpenMC-style)
├── Tallies/             # Tally classes, histograms, accumulators
├── Transport/           # Particle, Source, Distributions
│   └── Distribution/    # Spatial, angular, energy distributions
├── DevUtils/            # Testing framework and utilities
│   ├── Gtest/           # Google Test
│   ├── PngPlotter/      # Geometry visualization tool
│   └── Testing/         # Unit tests
├── CMake/               # CMake modules and configuration
├── Main.cpp             # helios++ entry point
└── CMakeLists.txt       # Build configuration
```

---

## Parallelism

Helios++ uses a **hybrid MPI + shared-memory** approach:

```
Node 0                        Node 1
+------------------------+    +------------------------+
| MPI Process            |    | MPI Process            |
| +--------------------+ |    | +--------------------+ |
| |   TBB / OpenMP     | |    | |   TBB / OpenMP     | |
| | +----+ +----+      | |<-->| | +----+ +----+      | |
| | |Task| |Task| ...  | |    | | |Task| |Task| ...  | |
| | +----+ +----+      | |    | | +----+ +----+      | |
| +--------------------+ |    | +--------------------+ |
+------------------------+    +------------------------+
```

**Key advantages:**

- **Task-based parallelism** handles load imbalance (some histories are much longer than others)
- **Shared memory within nodes** means cross sections, geometry, and sources aren't duplicated per core
- **Reproducibility** is maintained via parallel random number streams (TRNG)

---

## Building Helios++

### Dependencies

| Library | Purpose | Installation (Debian/Ubuntu) |
|---------|---------|------------------------------|
| **Boost** | MPI bindings, serialization, program_options | `sudo apt install libboost-all-dev` |
| **OpenMPI** | Distributed parallelism | `sudo apt install libopenmpi-dev openmpi-bin` |
| **Intel TBB** | Task-based parallelism | `sudo apt install libtbb-dev` |
| **Blitz++** | High-performance arrays | `sudo apt install libblitz0-dev` |
| **TRNG** | Parallel random numbers | [numbercrunch.de/trng](http://numbercrunch.de/trng/) |
| **libpng** | Geometry plotting | `sudo apt install libpng-dev` |
| **CMake** | Build system | `sudo apt install cmake` |

### Build Steps

```bash
# Clone the repository
git clone https://github.com/pellegre/helios.git
cd helios

# Create build directory
mkdir build && cd build

# Configure
cmake ..

# Build
make -j$(nproc)

# Install (optional)
sudo make install
```

This produces:

- **helios++** — The main Monte Carlo executable
- **plottermc++** — Geometry visualization tool (PNG output)
- **testmc++** — Unit test suite

---

## Running Helios++

### Basic Usage

```bash
helios++ --output results.out geometry.xml materials.xml source.xml settings.xml
```

### With MPI

```bash
mpiexec -n 4 helios++ --output results.out input.xml
```

### Input Format

Helios++ uses an XML format similar to [OpenMC](https://docs.openmc.org/). Input is split across multiple files (or combined):

- **Geometry**: Surfaces, cells, universes, lattices
- **Materials**: Isotopic compositions or macroscopic cross sections
- **Source**: Particle source definitions
- **Settings**: Batches, particles per batch, inactive cycles

---

## ACE Table Support

Helios++ reads **ACE-format** nuclear data tables (the same format used by MCNP and Serpent). It supports all continuous-energy tables distributed with Serpent, including:

- Elastic and inelastic scattering
- Fission (prompt and delayed neutrons)
- All standard energy distribution laws (Law 1, 4, 7, 9, 11, 44, 61)
- Angular distributions from the AND block
- NU data for fission multiplicity

Point your `xsdir` file to your ACE library and go.

---

## Geometry Visualization

Generate PNG slices of your geometry:

```bash
plottermc++ --input geometry.xml --output slice.png --axis z --position 0.0
```

---

## Testing

```bash
cd build
./testmc++
```

Unit tests cover ACE table parsing, geometric tracking, reaction sampling, and source distributions.

---

## License

Helios++ is open source under the **BSD 3-Clause License**.

---

## References

1. *Design Patterns: Elements of Reusable Object-Oriented Software* — Gamma, Helm, Johnson, Vlissides
2. *Modern C++ Design: Generic Programming and Design Patterns Applied* — Andrei Alexandrescu
3. [Policy-based design](https://en.wikipedia.org/wiki/Policy-based_design)
4. [OpenMC](https://docs.openmc.org/) — Input format inspiration
5. [Serpent](http://serpent.vtt.fi/) — ACE table compatibility

