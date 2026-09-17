# Linux Build Environment Setup (Field Notes)

This document complements [`docs/building_and_distribution.md`](../building_and_distribution.md)
with the practical steps needed to get a **native development build** working
on a Linux workstation whose default tooling does not meet the project's
minimum versions out of the box. It is written to be reproducible on machines
different from any single contributor's — check versions on your own machine
rather than assuming the paths below.

Read `docs/building_and_distribution.md` first for the authoritative
requirements list and CMake options. This file only covers the gaps that are
easy to hit on Ubuntu/Debian-based systems and how to work around them
**without touching system-wide state** (no `sudo` package upgrades/removals
required beyond the packages already listed in the main guide).

## 1. Check what you actually have

Before doing anything, check the four things that most commonly block
configure:

```bash
cmake --version          # need 3.30+
nvcc --version            # need 12.8+ (this may NOT be the version CMake picks by default)
gcc --version              # default `gcc`/`g++` may be too old for C++23
g++ --version
echo "$VCPKG_ROOT"        # vcpkg toolchain root, often unset
```

It is common for a machine to have **multiple CUDA toolkits installed** (e.g.
an old `nvidia-cuda-toolkit` apt package alongside a newer
`/usr/local/cuda-X.Y` install from NVIDIA's own installer). The version that
ends up on `PATH` depends on shell `PATH` ordering, not on what `nvidia-smi`
reports as the driver's supported CUDA version. Always verify with
`nvcc --version`, not `nvidia-smi`.

Check for multiple toolkits:

```bash
ls -d /usr/local/cuda-* 2>/dev/null
which -a nvcc
```

## 2. CMake 3.30+ without root

The apt-packaged `cmake` on Ubuntu 22.04 is 3.22, which is below this
project's `cmake_minimum_required`. Rather than fighting apt/PPA pinning,
install the official Kitware wheel into your user site-packages:

```bash
pip3 install --user --upgrade cmake
```

This installs to `~/.local/bin/cmake`. Make sure `~/.local/bin` comes
**before** `/usr/bin` on `PATH` for any shell/session you build from:

```bash
export PATH="$HOME/.local/bin:$PATH"
cmake --version   # should now report 3.30+
```

## 3. Point the build at CUDA 12.8+

If `/usr/local/cuda-12.8` (or newer) exists but `nvcc --version` still shows
an older release, an older toolkit is winning on `PATH`. Prepend the correct
toolkit's `bin` directory for the build session:

```bash
export CUDA_HOME=/usr/local/cuda-12.8            # adjust to your installed version
export PATH="$CUDA_HOME/bin:$PATH"
export LD_LIBRARY_PATH="$CUDA_HOME/lib64:$LD_LIBRARY_PATH"
nvcc --version   # confirm 12.8+ now
```

This is scoped to the current shell session only — it does not modify
`~/.bashrc`/`~/.zshrc`. Add it there yourself if you want it permanent, but
be aware that can affect other CUDA projects on the same machine.

## 4. Use a C++23-capable host compiler

The distro-default `gcc`/`g++` (11.x on Ubuntu 22.04) does not support the
C++23 features this project uses. Install `gcc-14`/`g++-14` per
`docs/building_and_distribution.md`'s package list (already covered if you
followed the "Linux Prerequisites" section), then pass them explicitly to
CMake — do not rely on `update-alternatives` or the bare `gcc`/`g++` names:

```bash
-DCMAKE_C_COMPILER=gcc-14 \
-DCMAKE_CXX_COMPILER=g++-14 \
-DCMAKE_CUDA_HOST_COMPILER=g++-14
```

## 5. vcpkg

If `VCPKG_ROOT` is not already set to a shared/team vcpkg checkout, clone one
locally. The repository's `.gitignore` already ignores `vcpkg/`, so cloning
it inside the repo root is convenient and keeps the toolchain colocated with
the project that pins its baseline (see `vcpkg.json` /
`vcpkg-configuration.json`):

```bash
git clone https://github.com/microsoft/vcpkg.git vcpkg
./vcpkg/bootstrap-vcpkg.sh
export VCPKG_ROOT="$(pwd)/vcpkg"
```

The first configure after this will build ~70+ dependency ports from source
(SDL3, ffmpeg with NVENC, assimp, libplacebo, draco, Vulkan-related tooling,
etc.). This is the slowest part of a first-time setup — expect it to take a
long time and a few GB of disk space, depending on network speed and core
count. Subsequent configures reuse the vcpkg binary cache and are fast.

## 6. Initialize git submodules

Configure fails late (after vcpkg has already built everything) if git
submodules are missing, with an error like:

```
CMake Error at cmake/FetchLibvterm.cmake:14 (message):
  libvterm sources are missing from external/libvterm.  Initialize the
  submodule with: git submodule update --init --recursive external/libvterm
```

Avoid the wasted vcpkg build time by doing this up front:

```bash
git submodule update --init --recursive
```

## 7. Configure and build

Putting it together, from the repository root:

```bash
export PATH="$HOME/.local/bin:$CUDA_HOME/bin:$PATH"
export VCPKG_ROOT="$(pwd)/vcpkg"          # or your shared vcpkg checkout
export LD_LIBRARY_PATH="$CUDA_HOME/lib64:$LD_LIBRARY_PATH"

cmake -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=gcc-14 \
  -DCMAKE_CXX_COMPILER=g++-14 \
  -DCMAKE_CUDA_HOST_COMPILER=g++-14

cmake --build build -j "$(nproc)"

./build/LichtFeld-Studio --help
```

`CMAKE_CUDA_ARCHITECTURES` defaults to `native`, so it auto-detects the
compute capability of the GPU(s) present at configure time (this project
requires compute capability 7.5+, i.e. GTX 16-series/RTX 20-series or newer).

## Troubleshooting quick reference

| Symptom | Cause | Fix |
|---|---|---|
| `CMake 3.3x or higher is required` | apt `cmake` is too old | `pip3 install --user --upgrade cmake`, put `~/.local/bin` first on `PATH` |
| Configure picks CUDA < 12.8 despite a newer toolkit being installed | An older `nvcc` (e.g. from `nvidia-cuda-toolkit`) is earlier on `PATH` | Prepend the correct `/usr/local/cuda-X.Y/bin` to `PATH` for the build shell |
| Compile errors referencing missing C++23 features/concepts | Default system `gcc`/`g++` too old | Pass `-DCMAKE_C_COMPILER=gcc-14 -DCMAKE_CXX_COMPILER=g++-14 -DCMAKE_CUDA_HOST_COMPILER=g++-14` |
| `libvterm sources are missing from external/libvterm` | Git submodules not initialized | `git submodule update --init --recursive` |
| `pkg-config could not locate gtk+-3.0` | Missing `libgtk-3-dev` | Install per the Linux Prerequisites package list in `docs/building_and_distribution.md` |
| `SDL3 was found, but the resolved SDL build does not expose an X11 or Wayland video backend` | Stale vcpkg SDL3 binary cache entry | See the dedicated section in `docs/building_and_distribution.md` |
