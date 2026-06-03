# Building Moonlight PC for ChromeOS ARM64 (from Windows)

This guide covers building a native ARM64 Linux binary of Moonlight PC on your Windows PC and deploying it to an ARM64 Chromebook running the Linux development environment (Crostini).

**Output**: A self-contained ARM64 AppImage you can run directly in ChromeOS Linux.

---

## Choose a Build Method

| Method | Difficulty | Build time | Requirements |
| ------ | --------- | ---------- | ------------ |
| [GitHub Actions](#method-1-github-actions-recommended) | Easy | ~30 min (unattended) | GitHub account, repo push access |
| [Docker Desktop](#method-2-docker-desktop-on-windows) | Medium | ~60 min | Docker Desktop, ~20 GB disk |
| [WSL2 + QEMU](#method-3-wsl2--qemu-chroot-advanced) | Hard | ~90 min | WSL2 (Ubuntu), ~25 GB disk |

---

## Method 1: GitHub Actions (Recommended)

GitHub provides free native ARM64 Linux runners (`ubuntu-24.04-arm`). Push your code and GitHub builds the ARM64 AppImage for you — no local setup needed.

### 1.1 — Push your fork to GitHub

If you haven't already:

```powershell
git remote add origin https://github.com/<your-username>/moonlight-chromeos.git
git push -u origin master
```

### 1.2 — Add the ARM64 workflow

Create the file `.github/workflows/build-chromeos-arm64.yml` in the repo:

```yaml
name: Build - ChromeOS ARM64

on:
  push:
    branches: [master]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-24.04-arm
    permissions:
      contents: read

    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          fetch-depth: 1

      - name: Install system dependencies
        run: |
          sudo apt-get update
          sudo apt-get install -y \
            build-essential cmake ninja-build nasm pkg-config python3-pip \
            qt6-base-dev qt6-declarative-dev libqt6svg6-dev qt6-wayland \
            qml6-module-qtquick-controls qml6-module-qtquick-templates \
            qml6-module-qtquick-layouts qml6-module-qtqml-workerscript \
            qml6-module-qtquick-window qml6-module-qtquick \
            libssl-dev libopus-dev libfreetype-dev \
            libegl1-mesa-dev libgl1-mesa-dev libgles2-mesa-dev \
            libgbm-dev libdrm-dev libdbus-1-dev \
            libasound2-dev libpulse-dev libpipewire-0.3-dev \
            libx11-dev libxcursor-dev libxext-dev libxi-dev \
            libxinerama-dev libxkbcommon-dev libxrandr-dev libxss-dev \
            libxt-dev libxv-dev libxxf86vm-dev libxcb-dri3-dev \
            libx11-xcb-dev libxfixes-dev libxtst-dev \
            libva-dev wayland-protocols libvulkan-dev \
            libpipewire-0.3-dev liburing-dev \
            autoconf automake libtool
          sudo pip3 install meson --break-system-packages

      - name: Build SDL3
        run: |
          git clone --branch release-3.4.10 --depth 1 https://github.com/libsdl-org/SDL.git deps/SDL
          cmake -DSDL_KMSDRM=OFF -DSDL_TEST_LIBRARY=OFF -DSDL_INSTALL_DOCS=OFF \
            -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON -S deps/SDL -B deps/SDL/build
          cmake --build deps/SDL/build -j$(nproc)
          sudo cmake --install deps/SDL/build

      - name: Build sdl2-compat
        run: |
          git clone --branch release-2.32.68 --depth 1 https://github.com/libsdl-org/sdl2-compat.git deps/sdl2-compat
          cmake -DSDL2COMPAT_TESTS=OFF -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON \
            -S deps/sdl2-compat -B deps/sdl2-compat/build
          cmake --build deps/sdl2-compat/build -j$(nproc)
          sudo cmake --install deps/sdl2-compat/build

      - name: Build SDL_ttf
        run: |
          git clone https://github.com/libsdl-org/SDL_ttf.git deps/SDL_ttf
          cd deps/SDL_ttf
          git checkout a883e490e30fb44a5336ea3dcb990c6982c5216f
          git submodule update --init --recursive
          ./autogen.sh && ./configure && make -j$(nproc) && sudo make install

      - name: Build libva
        run: |
          git clone --branch 2.23.0 --depth 1 https://github.com/intel/libva.git deps/libva
          cd deps/libva && ./autogen.sh && ./configure --enable-x11 --enable-wayland
          make -j$(nproc) && sudo make install

      - name: Build libplacebo
        run: |
          git clone https://github.com/haasn/libplacebo.git deps/libplacebo
          cd deps/libplacebo
          git checkout b915882db8d349cc1831c3d3978ee7d5f914b10b
          git submodule update --init --recursive
          meson setup build -Dvulkan=enabled -Dopengl=disabled -Ddemos=false
          ninja -C build && sudo ninja -C build install

      - name: Build dav1d
        run: |
          git clone --branch 1.5.3 --depth 1 https://code.videolan.org/videolan/dav1d.git deps/dav1d
          meson setup deps/dav1d/build deps/dav1d \
            -Ddefault_library=static -Dbuildtype=release \
            -Denable_tools=false -Denable_tests=false
          ninja -C deps/dav1d/build && sudo ninja -C deps/dav1d/build install

      - name: Build FFmpeg
        run: |
          git clone --branch n8.1.1 --depth 1 https://github.com/FFmpeg/FFmpeg.git deps/FFmpeg
          cd deps/FFmpeg
          ./configure \
            --enable-pic --enable-lto --disable-static --enable-shared \
            --disable-all --disable-autodetect \
            --enable-avcodec --enable-avformat --enable-swscale \
            --enable-decoder=h264 --enable-decoder=hevc --enable-decoder=av1 \
            --enable-vaapi \
            --enable-hwaccel=h264_vaapi \
            --enable-hwaccel=hevc_vaapi \
            --enable-hwaccel=av1_vaapi \
            --enable-libdrm \
            --enable-vulkan \
            --enable-hwaccel=h264_vulkan \
            --enable-hwaccel=hevc_vulkan \
            --enable-hwaccel=av1_vulkan \
            --enable-libdav1d \
            --enable-decoder=libdav1d
          make -j$(nproc) && sudo make install

      - name: Install linuxdeploy
        run: |
          wget -q https://github.com/linuxdeploy/linuxdeploy/releases/download/continuous/linuxdeploy-aarch64.AppImage
          wget -q https://github.com/linuxdeploy/linuxdeploy-plugin-qt/releases/download/continuous/linuxdeploy-plugin-qt-aarch64.AppImage
          chmod +x linuxdeploy-aarch64.AppImage linuxdeploy-plugin-qt-aarch64.AppImage
          sudo mv linuxdeploy-aarch64.AppImage /usr/local/bin/linuxdeploy-aarch64.AppImage
          sudo mv linuxdeploy-plugin-qt-aarch64.AppImage /usr/local/bin/linuxdeploy-plugin-qt-aarch64.AppImage

      - name: Build Moonlight
        run: |
          sudo ldconfig
          mkdir -p build/build-release build/deploy-release build/installer-release
          cd build/build-release
          qmake6 ../../moonlight-qt.pro \
            CONFIG+=release \
            CONFIG+=disable-libvdpau \
            PREFIX=../../build/deploy-release/usr
          make -j$(nproc) release
          make install

      - name: Package AppImage
        run: |
          export QML_SOURCES_PATHS=$PWD/app/gui
          export QMAKE=qmake6
          cd build/installer-release
          VERSION=chromeos-arm64 \
          LD_LIBRARY_PATH=/usr/local/lib \
          linuxdeploy-aarch64.AppImage \
            --appdir ../deploy-release \
            --library=/usr/local/lib/libSDL3.so.0 \
            --plugin qt \
            --output appimage

      - name: Upload AppImage
        uses: actions/upload-artifact@v4
        with:
          name: Moonlight-ChromeOS-arm64
          path: build/installer-release/Moonlight-*.AppImage
          compression-level: 0
          if-no-files-found: error
```

### 1.3 — Trigger the build and download

1. Commit and push the new workflow file.
2. Go to **github.com/\<your-username\>/moonlight-chromeos → Actions**.
3. Select **Build - ChromeOS ARM64** → click the latest run → scroll to **Artifacts**.
4. Download **Moonlight-ChromeOS-arm64.zip**.
5. Extract it — you get `Moonlight-chromeos-arm64.AppImage`.

Skip to [Deploying to the Chromebook](#deploying-to-the-chromebook).

---

## Method 2: Docker Desktop on Windows

Builds an ARM64 binary locally using QEMU emulation inside Docker. Slower than GitHub Actions but entirely offline after the initial image pull.

### 2.1 — Install Docker Desktop

Download from [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/). During install, keep **WSL 2 backend** selected. After install, open Docker Desktop and wait for the engine to start.

Docker Desktop includes QEMU binfmt support automatically — no extra setup needed to run ARM64 containers on an x86-64 Windows PC.

### 2.2 — Create a build script

Create `build-arm64.sh` in the root of the repo:

```bash
#!/bin/bash
set -e

apt-get update && apt-get install -y \
  build-essential cmake ninja-build nasm pkg-config python3-pip \
  qt6-base-dev qt6-declarative-dev libqt6svg6-dev qt6-wayland \
  qml6-module-qtquick-controls qml6-module-qtquick-templates \
  qml6-module-qtquick-layouts qml6-module-qtqml-workerscript \
  qml6-module-qtquick-window qml6-module-qtquick \
  libssl-dev libopus-dev libfreetype-dev \
  libegl1-mesa-dev libgl1-mesa-dev libgles2-mesa-dev \
  libgbm-dev libdrm-dev libdbus-1-dev \
  libasound2-dev libpulse-dev libpipewire-0.3-dev \
  libx11-dev libxcursor-dev libxext-dev libxi-dev \
  libxinerama-dev libxkbcommon-dev libxrandr-dev libxss-dev \
  libxt-dev libxv-dev libxxf86vm-dev libxcb-dri3-dev \
  libx11-xcb-dev libxfixes-dev libxtst-dev \
  libva-dev wayland-protocols libvulkan-dev \
  libpipewire-0.3-dev liburing-dev git wget \
  autoconf automake libtool

pip3 install meson --break-system-packages

cd /src

# SDL3
git clone --branch release-3.4.10 --depth 1 https://github.com/libsdl-org/SDL.git /tmp/SDL
cmake -DSDL_KMSDRM=OFF -DSDL_TEST_LIBRARY=OFF -DSDL_INSTALL_DOCS=OFF \
  -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON -S /tmp/SDL -B /tmp/SDL/build
cmake --build /tmp/SDL/build -j$(nproc) && cmake --install /tmp/SDL/build

# sdl2-compat
git clone --branch release-2.32.68 --depth 1 https://github.com/libsdl-org/sdl2-compat.git /tmp/sdl2-compat
cmake -DSDL2COMPAT_TESTS=OFF -DCMAKE_INTERPROCEDURAL_OPTIMIZATION=ON \
  -S /tmp/sdl2-compat -B /tmp/sdl2-compat/build
cmake --build /tmp/sdl2-compat/build -j$(nproc) && cmake --install /tmp/sdl2-compat/build

# SDL_ttf
git clone /src/deps/SDL_ttf /tmp/SDL_ttf 2>/dev/null || \
  git clone https://github.com/libsdl-org/SDL_ttf.git /tmp/SDL_ttf
cd /tmp/SDL_ttf
git checkout a883e490e30fb44a5336ea3dcb990c6982c5216f
git submodule update --init --recursive
./autogen.sh && ./configure && make -j$(nproc) && make install
cd /src

# libva
git clone --branch 2.23.0 --depth 1 https://github.com/intel/libva.git /tmp/libva
cd /tmp/libva && ./autogen.sh && ./configure --enable-x11 --enable-wayland
make -j$(nproc) && make install
cd /src

# libplacebo
git clone https://github.com/haasn/libplacebo.git /tmp/libplacebo
cd /tmp/libplacebo
git checkout b915882db8d349cc1831c3d3978ee7d5f914b10b
git submodule update --init --recursive
meson setup build -Dvulkan=enabled -Dopengl=disabled -Ddemos=false
ninja -C build && ninja -C build install
cd /src

# dav1d
git clone --branch 1.5.3 --depth 1 https://code.videolan.org/videolan/dav1d.git /tmp/dav1d
meson setup /tmp/dav1d/build /tmp/dav1d \
  -Ddefault_library=static -Dbuildtype=release -Denable_tools=false -Denable_tests=false
ninja -C /tmp/dav1d/build && ninja -C /tmp/dav1d/build install

# FFmpeg
git clone --branch n8.1.1 --depth 1 https://github.com/FFmpeg/FFmpeg.git /tmp/FFmpeg
cd /tmp/FFmpeg
./configure \
  --enable-pic --enable-lto --disable-static --enable-shared \
  --disable-all --disable-autodetect \
  --enable-avcodec --enable-avformat --enable-swscale \
  --enable-decoder=h264 --enable-decoder=hevc --enable-decoder=av1 \
  --enable-vaapi \
  --enable-hwaccel=h264_vaapi --enable-hwaccel=hevc_vaapi --enable-hwaccel=av1_vaapi \
  --enable-libdrm \
  --enable-vulkan \
  --enable-hwaccel=h264_vulkan --enable-hwaccel=hevc_vulkan --enable-hwaccel=av1_vulkan \
  --enable-libdav1d --enable-decoder=libdav1d
make -j$(nproc) && make install
cd /src

# linuxdeploy
wget -q https://github.com/linuxdeploy/linuxdeploy/releases/download/continuous/linuxdeploy-aarch64.AppImage -O /usr/local/bin/linuxdeploy-aarch64.AppImage
wget -q https://github.com/linuxdeploy/linuxdeploy-plugin-qt/releases/download/continuous/linuxdeploy-plugin-qt-aarch64.AppImage -O /usr/local/bin/linuxdeploy-plugin-qt-aarch64.AppImage
chmod +x /usr/local/bin/linuxdeploy-aarch64.AppImage /usr/local/bin/linuxdeploy-plugin-qt-aarch64.AppImage

# Moonlight
ldconfig
mkdir -p /src/build/build-release /src/build/deploy-release /src/build/installer-release
cd /src/build/build-release
qmake6 /src/moonlight-qt.pro \
  CONFIG+=release \
  CONFIG+=disable-libvdpau \
  PREFIX=/src/build/deploy-release/usr
make -j$(nproc) release
make install

export QML_SOURCES_PATHS=/src/app/gui
export QMAKE=qmake6
cd /src/build/installer-release
VERSION=chromeos-arm64 \
LD_LIBRARY_PATH=/usr/local/lib \
linuxdeploy-aarch64.AppImage \
  --appdir /src/build/deploy-release \
  --library=/usr/local/lib/libSDL3.so.0 \
  --plugin qt \
  --output appimage

echo ""
echo "Build complete: /src/build/installer-release/Moonlight-chromeos-arm64.AppImage"
```

### 2.3 — Run the Docker build

Open **PowerShell** in the repo root:

```powershell
# Start the ARM64 container with the repo mounted
docker run --rm --platform linux/arm64 `
  -v "${PWD}:/src" `
  arm64v8/debian:bookworm `
  bash /src/build-arm64.sh
```

This will take 45–90 minutes on a typical PC (QEMU emulation is ~5–8x slower than native). When it finishes, `build\installer-release\Moonlight-chromeos-arm64.AppImage` will be in your repo folder on Windows.

> **Tip:** If Docker asks about sharing the drive, click **Share**.

Skip to [Deploying to the Chromebook](#deploying-to-the-chromebook).

---

## Method 3: WSL2 + QEMU chroot (Advanced)

This method uses WSL2 with QEMU user-mode emulation to run an ARM64 chroot directly on Windows. Slightly faster than Docker for repeated builds since the chroot persists.

### 3.1 — Install WSL2 and prerequisites

```powershell
wsl --install -d Ubuntu-24.04
```

Inside the WSL2 Ubuntu shell:

```bash
sudo apt update && sudo apt install -y \
  qemu-user-static binfmt-support debootstrap
```

### 3.2 — Create an ARM64 Debian chroot

```bash
sudo debootstrap --arch=arm64 --foreign bookworm /opt/arm64-chroot \
  http://deb.debian.org/debian/

sudo chroot /opt/arm64-chroot /debootstrap/debootstrap --second-stage
```

### 3.3 — Enter the chroot and build

```bash
sudo chroot /opt/arm64-chroot /bin/bash
```

Inside the chroot, follow all the same steps as the [GitHub Actions build script](#12--add-the-arm64-workflow) — install deps, clone, build SDL/FFmpeg/Moonlight, run linuxdeploy.

Copy the AppImage out:

```bash
exit   # leave the chroot
cp /opt/arm64-chroot/src/build/installer-release/Moonlight-*.AppImage /mnt/c/Users/$USER/Desktop/
```

---

## Deploying to the Chromebook

### Step 1 — Enable Linux on your Chromebook (if not already done)

Go to Settings → Advanced → Developers → Linux development environment → Turn On.

Wait for the install to finish and note your Linux username (shown in the terminal prompt).

### Step 2 — Transfer the AppImage

#### Option A — ChromeOS Files app (simplest)

1. Open the **Files** app on your Chromebook.
2. Navigate to **Linux files** in the left sidebar — this folder maps directly to your Linux home directory (`/home/<username>/`).
3. Drag `Moonlight-chromeos-arm64.AppImage` from your Windows PC (via Google Drive, USB, or any method) into that folder.

#### Option B — Google Drive

1. Upload the AppImage to Google Drive from Windows.
2. In Crostini terminal: the Drive is accessible at `/mnt/chromeos/GoogleDrive/MyDrive/`.

```bash
cp /mnt/chromeos/GoogleDrive/MyDrive/Moonlight-chromeos-arm64.AppImage ~/
```

#### Option C — USB drive

1. Copy the AppImage to a USB drive on Windows.
2. Plug the USB drive into the Chromebook. It appears in the Files app under **removable storage**.
3. In Crostini: it's at `/mnt/chromeos/removable/<DRIVE_LABEL>/`.

```bash
cp "/mnt/chromeos/removable/MY_USB/Moonlight-chromeos-arm64.AppImage" ~/
```

### Step 3 — Run the AppImage

Open the Linux terminal on your Chromebook:

```bash
chmod +x ~/Moonlight-chromeos-arm64.AppImage

# Run under Wayland (recommended — ChromeOS uses Wayland via Sommelier)
QT_QPA_PLATFORM=wayland ~/Moonlight-chromeos-arm64.AppImage

# Fallback to XWayland if Wayland has display issues
DISPLAY=:0 QT_QPA_PLATFORM=xcb ~/Moonlight-chromeos-arm64.AppImage
```

### Step 4 — Create a desktop shortcut (optional)

```bash
mkdir -p ~/.local/share/applications

cat > ~/.local/share/applications/moonlight.desktop << EOF
[Desktop Entry]
Name=Moonlight
Comment=NVIDIA GameStream / Sunshine client
Exec=env QT_QPA_PLATFORM=wayland /home/$(whoami)/Moonlight-chromeos-arm64.AppImage
Icon=moonlight
Terminal=false
Type=Application
Categories=Game;
EOF
```

Moonlight will now appear in the ChromeOS app launcher under Linux apps.

---

## Hardware Decode Notes

| Method | Status in Crostini |
| ------ | ------------------ |
| **Software (CPU)** | Always works — Moonlight falls back automatically |
| **VA-API** | Works if `/dev/dri/renderD128` is exposed to the VM (common on Intel-GPU Chromebooks) |
| **Vulkan** | Works on newer Chromebooks with Mesa + virtio-gpu |
| **VDPAU** | Not available (NVIDIA only) — disabled in this build |
| **V4L2** | Not accessible from Crostini currently (VirtIO Video is still in development) |

Check what's available in your Crostini terminal:

```bash
ls /dev/dri/            # DRI devices — needed for VA-API/Vulkan
vainfo 2>/dev/null      # VA-API decode profiles (install: sudo apt install vainfo)
```

If no `/dev/dri/` devices appear, software decode is your only option — it still works well for most use cases.

---

## Troubleshooting

### Docker build fails with `exec format error`

QEMU binfmt is not active. Restart Docker Desktop, or run:

```powershell
docker run --privileged --rm tonistiigi/binfmt --install arm64
```

### AppImage exits immediately with no error

Run it from the terminal (not the desktop icon) to see the error output. Often a missing `FUSE` dependency — install it with:

```bash
sudo apt install libfuse2
```

### `QT_QPA_PLATFORM=wayland` gives a blank window

Try `xcb` (XWayland) instead. Some Sommelier versions have compatibility issues with certain Qt 6.x Wayland backends.

### GitHub Actions build fails at linuxdeploy step

The aarch64 linuxdeploy AppImage requires FUSE at runtime. Add this step before the linuxdeploy step in the workflow:

```yaml
- name: Extract linuxdeploy (no-FUSE mode)
  run: |
    linuxdeploy-aarch64.AppImage --appimage-extract
    mv squashfs-root /tmp/linuxdeploy
    ln -sf /tmp/linuxdeploy/AppRun /usr/local/bin/linuxdeploy-aarch64.AppImage
```

**Out of disk space (Docker)**
The build uses ~15 GB inside the container. Increase Docker Desktop's disk image size: **Docker Desktop → Settings → Resources → Disk image size**.
