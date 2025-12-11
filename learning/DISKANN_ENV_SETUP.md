# DiskANN Environment Setup (macOS-friendly)

This document provides recommended ways to get DiskANN building and running for development and experimentation. Two approaches are supported:

- Option A (Recommended): Use Docker to get a Linux-compatible environment with all dependencies already specified by the repository Dockerfiles.
- Option B (macOS native): Install dependencies on macOS and build natively (works with caveats; some features rely on Linux-specific libs like libaio).

---

## Option A — Docker (Recommended on macOS)

Why Docker? DiskANN is primarily tested on Linux and Windows. The easiest and most reproducible way to get a working environment on macOS is to use the repo's Docker images which match CI instructions and provide all dependencies.

Prerequisites:
- Docker Desktop for Mac (https://docs.docker.com/desktop/mac/install/)
- Optional: Increase memory / CPU settings for Docker if you plan to use bigger datasets.

Steps:

1. Build the dev image (fast, reproducible):

```bash
# From the repo root
# Build the development image defined by DockerfileDev
docker build -f DockerfileDev -t diskann-dev:local .
```

2. Start a container and mount your repo for iterative development:

```bash
docker run -it --name diskann-dev --rm -v "$PWD":/workspace -w /workspace diskann-dev:local /bin/bash
```

Inside the container (quick build):

```bash
mkdir -p build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
make -j$(nproc)
```

3. Run a small example (in-memory search executable):

```bash
# Replace with the actual binary path; built binaries are under apps or src output
./apps/search_memory_index -db_file <path-to-dataset> -query_file <path-to-queries> -indexType in-memory
```

For more guidance, see `/workflows/*` files in the repo root.

---

## Option B — macOS Native Build

macOS is not the officially tested platform for DiskANN. Native builds can succeed for core functionality but some Linux-specific features (like `libaio` and certain ASYNC file readers) may not be available or used on macOS.

### 1) Prerequisites

- Xcode Command Line Tools (compile toolchain)
- Homebrew (https://brew.sh/)
- Install common dependencies using brew:

```bash
# Install Homebrew first if you don't have it
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install packages
brew update
brew install cmake boost gperftools openmp wget clang-format python3
```

Note: On macOS `libaio` is not applicable. DiskANN uses libaio for asynchronous operations on Linux; the in-memory paths and some features will still work.

### 2) Install Intel oneAPI MKL (recommended)

DiskANN relies on MKL for certain operations. The repository expects MKL in some standard locations, so we recommend installing Intel oneAPI MKL on macOS.

1. Download and install Intel oneAPI Base Toolkit and MKL (https://software.intel.com/content/www/us/en/develop/tools/oneapi/base-toolkit.html)
2. Source the setvars script so the compiler can find MKL & OMP shared libs. For example, after installing to `/opt/intel/oneapi`:

```bash
source /opt/intel/oneapi/setvars.sh
```

3. If CMake cannot find MKL automatically, you can provide paths explicitly:

```bash
cmake -DMKL_PATH=/opt/intel/oneapi/mkl/latest -DMKL_INCLUDE_PATH=/opt/intel/oneapi/mkl/latest/include -DOMP_PATH=/opt/intel/oneapi/compiler/latest/linux/compiler/lib/intel64_lin -DCMAKE_BUILD_TYPE=Release ..
```

Adjust paths depending on your installation location.

If you are unable to use MKL on macOS, you may try to use OpenBLAS as a fallback. However, the repo uses MKL header includes; this may require small code changes or header wrapper to compile with OpenBLAS.

### 3) Build steps (native)

```bash
# From the repo root
git submodule init && git submodule update --recursive
mkdir -p build && cd build
# Provide MKL path if needed, and ensure OpenMP is found
cmake -DCMAKE_BUILD_TYPE=Release -DMKL_PATH=/opt/intel/oneapi/mkl/latest -DMKL_INCLUDE_PATH=/opt/intel/oneapi/mkl/latest/include ..
make -j$(sysctl -n hw.ncpu)
```

Possible caveats / overrides:
- If OpenMP not found via brew, you can install `libiomp` via brew (e.g., `brew install libomp`) and set environment variables so clang can use it.
- `libaio` is a Linux-only dependency; some async features (disk-based index operations) may not work.
- If you see MKL errors, make sure the `MKL_PATH` and `MKL_INCLUDE_PATH` CMake variables are set correctly.

### 4) Run a small example

```bash
# Run a built app, adjust binary path if needed
./apps/search_memory_index -db_file <path-to-dataset> -query_file <path-to-query> -indexType in-memory
```

If you do not have a dataset handy, you can generate a small random dataset in Python and then use DiskANN's `apps` utilities to build an index and test queries.

---

## Python wrapper (optional)

The repo includes `python/` submodule and `diskannpy` to use DiskANN from Python. If you'd like Python bindings, inside the `python` dir, follow `python/README.md` steps:

```bash
cd python
# Typically, it relies on building the C++ project and then installing diskannpy
pip install -e .
```

---

## Troubleshooting

- If `cmake` complains about missing `Boost`, install it via brew: `brew install boost` and then run `cmake` with `-DBOOST_ROOT=$(brew --prefix boost)`.
- If you run into MKL errors, verify paths and `source` Intel `setvars.sh`, or try installing an OpenBLAS alternative.
- If you prefer not to install MKL, use the Docker approach.

---

## Next Steps — Examples

If you want, I can add a step-by-step example (download a sample dataset, build an in-memory index, and run search) in the `learning/jack/` folder and add a short `notes.md` describing how indexes map to the `apps` directory.

---

References:
- Project README.md — top-level build instructions and links to `workflows/*.md`
- DockerfileDev — for reproducible dev environment
- Intel oneAPI (MKL) — https://www.intel.com/content/www/us/en/developer/tools/oneapi.html

