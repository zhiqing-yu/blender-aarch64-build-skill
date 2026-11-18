---
name: blender-aarch64-build
description: Build and install Blender from source on aarch64 Linux (NVIDIA DGX Spark / GB10 / Grace, Ubuntu 24.04), where official x64-only binaries don't run. Use when asked to install Blender on DGX Spark, ARM64 Linux, or when a Blender source build or its `make deps` fails.
---

# Blender from source on aarch64 (DGX Spark, Ubuntu 24.04)

Official Blender Linux builds are x64 only and `lib/linux_arm64` prebuilt libs don't exist, so
build deps (`make deps`, ~2h+) then Blender (~1h on 20 cores). Quick alternatives first:
`sudo apt install blender` (4.0.2) or `flatpak install flathub org.blender.Blender`.
Verified: Blender 5.2.3 LTS, CUDA 13.0, GB10 sm_121 -> Cycles CUDA works (no OptiX).

## Sudo steps (agent can't sudo; ask the user, verify with `dpkg -s`)
```
sudo apt install -y gcc-14 g++-14 libxxf86vm-dev libxcursor-dev libxi-dev libxrandr-dev libxinerama-dev \
  libwayland-dev wayland-protocols libdecor-0-dev yasm patchelf meson libepoxy-dev libxfixes-dev \
  libsndfile1-dev autopoint help2man texinfo gettext python3-mako libxt-dev libgl-dev libglu1-mesa-dev
```
GCC >= 14 is required by Blender 5.x (Ubuntu 24.04 default is 13).

## Steps
1. Clone: `git clone --depth 1 --branch vX.Y.Z https://github.com/blender/blender.git ~/github_repo/blender`
   GitHub mirror has NO LFS objects -> checkout fails. Use `GIT_LFS_SKIP_SMUDGE=1 git checkout -f HEAD`,
   then later `git config lfs.url https://projects.blender.org/blender/blender.git/info/lfs && git lfs pull`
   (configure fails with "incomplete startup blend" otherwise).
2. Use **system cmake 3.28** (`export PATH=/usr/bin:$PATH`) for everything. cmake 4.x (e.g. uv/pip
   ~/.local/bin) breaks old deps (x265 policy CMP0025 errors, openal min version).
   Also `export CMAKE_POLICY_VERSION_MINIMUM=3.5`. Pick cmake BEFORE first `make deps`:
   switching later forces a full rebuild (LLVM etc.).
3. `make deps` from the blender dir (builds to `build_linux/deps_arm64`, installs `lib/linux_arm64`).
   Wrap in a retry loop; run with nohup, poll the log.

## Known fixes during `make deps`
- Downloads via proxy fail with "SSL connect error" (ffmpeg.org, gmplib.org). Put the file in
  `build_linux/deps_arm64/packages/` manually: try `curl -C - --noproxy '*'` with retries, or a
  mirror (gmp: ftp.gnu.org/gnu/gmp/gmp-6.3.0.tar.xz). Verify the hash in cmake/versions.cmake.
- OSL: clang 20 can't parse CUDA 13 headers -> in `build_files/build_environment/cmake/osl.cmake`
  set `-DOSL_USE_OPTIX=OFF`; delete `build/osl` and rerun.
- MaterialX: needs `libxt-dev`; delete `build/materialx` after installing.
- OpenImageIO install: "cannot find .../python/stubs/OpenImageIO/__init__.pyi" -> the source is
  re-extracted mid-build, so run a background loop that `touch`es that file every second.
- weston needs pixman etc. and is only for tests: comment out
  `include(cmake/wayland_weston.cmake)` in `build_files/build_environment/CMakeLists.txt`, then
  `mkdir -p build_linux/deps_arm64/Release/wayland/lib64` (the install step harvests it).
- Never use `pkill -f <name>` from the agent shell: it matches its own command line and kills it.

## Configure + build
```
export PATH=/usr/bin:$PATH
mkdir -p ../build_blender && cd ../build_blender
CC=gcc-14 CXX=g++-14 cmake -G Ninja -C ../blender/build_files/cmake/config/blender_release.cmake ../blender \
  -DCMAKE_BUILD_TYPE=Release -DWITH_CYCLES_CUDA_BINARIES=ON -DCYCLES_CUDA_BINARIES_ARCH=sm_121 \
  -DWITH_CYCLES_DEVICE_OPTIX=OFF -DCMAKE_INSTALL_PREFIX=$HOME/blender-X.Y.Z \
  -DCMAKE_EXE_LINKER_FLAGS="-Wl,--no-as-needed -ldrm"
ninja install
```
The `-ldrm` flag fixes the final link error (undefined drmGetDevice from static ffmpeg Vulkan).

## Verify
`~/blender-X.Y.Z/blender --version`, then headless render with `cycles.preferences.compute_device_type='CUDA'`
and check the GB10 appears in `preferences.devices`. HIPEW warning is harmless.

## Desktop launcher
`~/.local/share/applications/blender.desktop` with `Exec=<prefix>/blender %f`,
`Icon=<prefix>/blender.svg`, `Categories=Graphics;3DGraphics;`; run `update-desktop-database`.
