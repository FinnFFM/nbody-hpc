# N-Body HPC

A student project comparing sequential, vectorized and parallel implementations of a gravitational N-body simulation in C++17. It combines an SFML visualization with a command-line benchmark to explore SIMD, OpenMP and OpenCL.

## Simulation

Bodies start in a disk around a heavier central body. All bodies interact through softened Newtonian gravity, using a direct O(N²) force calculation and kick–drift–kick time integration. The initialization uses a fixed random seed (`42`).

| Backend | Approach |
| --- | --- |
| `sequential` | Scalar CPU implementation with an array of body structures |
| `simd` | Explicit SIMD with `std::experimental::simd` and separate arrays for each property |
| `omp` | CPU parallelism with OpenMP |
| `simd-omp` | SIMD combined with OpenMP |
| `opencl` | OpenCL kernels on the first available platform and device |

## Build on Ubuntu / WSL

The supported setup is Ubuntu with GCC, CMake, SFML 2.x and OpenCL. GCC's standard library must provide `<experimental/simd>`.

```bash
sudo apt update
sudo apt install build-essential cmake libsfml-dev fonts-open-sans ocl-icd-opencl-dev
```

The OpenCL backend also needs a runtime. A CPU runtime is sufficient to try it:

```bash
sudo apt install pocl-opencl-icd
```

GPU execution requires a working OpenCL driver for that GPU. Installing the development headers alone does not provide a device.

From the repository root:

```bash
cmake -S HPCA-PC_Code-Final -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j 4
cd build
```

Run the executable **from the build directory**, which contains the copied OpenCL kernel and font. CMake uses the packaged Open Sans font; a custom location can be supplied with `-DNBODY_FONT=/path/to/OpenSans-Bold.ttf`.

Release builds use `-O3 -march=native`, so rebuild on the machine where you will run the program. Automatic compiler vectorization is disabled to keep it separate from the explicit SIMD implementation. The original `install_sfml.sh` is retained, but is not needed for the Ubuntu package-based setup above.

## Run

From `build/`, start a visualization:

```bash
./nbody sequential
./nbody simd
./nbody omp 4
OMP_NUM_THREADS=4 ./nbody simd-omp
./nbody opencl
```

Close the window to stop. Visualization requires a graphical desktop or working WSLg display. Running without arguments selects `sequential`.

Run all backends without opening a window:

```bash
OMP_NUM_THREADS=4 ./nbody benchmark
```

Or benchmark one backend:

```bash
./nbody sequential benchmark
./nbody omp 4 benchmark
OMP_NUM_THREADS=4 ./nbody simd-omp benchmark
./nbody opencl benchmark
```

The default workload is **5,000 orbiting bodies plus one central body**, for **1,000 benchmark steps per backend**. The full benchmark can take a while. If no OpenCL runtime is available, use the individual CPU benchmarks; the combined benchmark exits when OpenCL initialization fails.

## Quick check

For a small run, execute these commands from the repository root:

```bash
cmake -S HPCA-PC_Code-Final -B build-smoke -DNBODY_COUNT=64 -DNBODY_STEPS=5
cmake --build build-smoke -j 4
cd build-smoke
OMP_NUM_THREADS=2 ./nbody benchmark
```

This runs all five backends with 65 total bodies and five steps. The build and this check were tested on Ubuntu 24.04 in WSL with GCC 13.3, CMake 3.28.3 and SFML 2.6.1. This is an execution check, not a performance measurement or numerical validation. The full default benchmark and graphical window were not tested.

## Parameters and benchmark notes

Set `NBODY_COUNT` and `NBODY_STEPS` through CMake, then rebuild. The other simulation constants are in `HPCA-PC_Code-Final/src/nbody_helper.cpp`: `G = 1`, `dt = 0.05`, softening `1`, and central mass `1000`.

The benchmark prints elapsed milliseconds, time per step and speedup relative to the sequential run. Results depend on hardware, compiler, workload and thread count. Initialization and visualization are outside the timed loops, and there are no repeated trials or error bars.

A few limitations of the original benchmark remain:

- The `simd-omp` thread column is hard-coded to `1`; it does not report the actual OpenMP team size. OpenCL reports `0` CPU threads.
- The printed GPU name is found separately from OpenCL device selection and may not identify the device actually used. OpenCL can run on a CPU.
- Floating-point summation differs between implementations. There is no automatic accuracy comparison or energy-conservation check.

## Source layout

All source files are under `HPCA-PC_Code-Final/`:

- `nbody.cpp`: backend selection and command-line entry point.
- `src/backends/`: the five implementations.
- `src/nbody_helper.cpp` and `src/body_soa.cpp`: initialization and shared data structures.
- `src/benchmark.cpp`: timing and comparison table.
- `opencl/nbody.cl`: OpenCL kernels.

This is a learning project about parallel programming and performance comparison, rather than a production astrophysics simulator.
