---
title: "Download"
---

# Get Started with Weave

The unified installer and project creator for **MayaFlux**.

Weave downloads the framework, installs dependencies, and sets up your first project in minutes.

---

<!-- ====================== -->
<!-- macOS CARD             -->
<!-- ====================== -->

<div class="card collapsible">

<div class="collapsible-header">
<h2>macOS</h2>
<p>
<a href="https://github.com/MayaFlux/Weave/releases/download/v0.2.1/Weave-installer-macos.pkg"><strong>Download for macOS</strong></a>
</p>
<p class="hint">Click to expand</p>
</div>

<div class="collapsible-body">

### Install

1. Download the `.pkg` file
2. Double-click to open
3. If macOS shows **“unverified developer”**:
   - Close dialog
   - System Settings → **Privacy & Security**
   - Find **Weave.pkg** → **Open Anyway**
4. Run installer normally
5. A Terminal window will open and complete setup
6. When it closes, Weave is ready

### Prerequisites

- **Apple Silicon:** macOS 14.0 (Sonoma) or later
- **Intel:** macOS 15.0 (Sequoia) or later

### Contents

- **Weave.app** and CLI (`~/.local/bin/weave`)
- **MayaFlux** via Homebrew
- Environment variables in `~/.zshenv`

### Create & Build a Project

```bash
weave new MyProject ~/Projects/

cd MyProject && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --parallel
./MyProject
```

📘 **Full guide:**
[macOS Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/MACOS.md)

</div>
</div>

---

<!-- ====================== -->

<!-- Linux CARD             -->

<!-- ====================== -->

<div class="card collapsible">

<div class="collapsible-header">
<h2>Linux</h2>
<p>
<a href="https://github.com/MayaFlux/Weave/releases/download/v0.2.1/Weave-linux.tar.gz"><strong>Download for Linux</strong></a>
</p>
<p class="hint">Click to expand</p>
</div>

<div class="collapsible-body">

### Install

```bash
tar -xzf Weave-X.X.X-linux.tar.gz -C ~/.local/
~/.local/Weave-X.X.X/Weave
```

### Supported Distributions

- Arch Linux (pacman)
- Fedora >= 43 (dnf)
- Ubuntu (25)/Debian (ci build)
- openSUSE (ci build)

### Mode Selection

- **Install MayaFlux** — first-time setup
- **Create Project** — if MayaFlux already installed

### Supported Behaviours

- **Arch Linux**: installs `mayaflux-dev-bin` from AUR
- **Fedora 43+**: installs from COPR
- **Ubuntu 25+, openSUSE Tumbleweed, others**: downloads MayaFlux locally into `~/MayaFlux`

> Requires: **GCC 15+**, **LLVM 21+**, **GLFW 3.4+**

### Create & Build

**GUI:**

```bash
~/.local/Weave-X.X.X/Weave
```

**CLI:**

```bash
weave new MyProject ~/Projects/
```

**Build:**

```bash
cd MyProject && mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --parallel
./MyProject
```

📘 **Full guide:**
[Linux Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/LINUX.md)

</div>
</div>

---

<!-- ====================== -->

<!-- Windows CARD           -->

<!-- ====================== -->

<div class="card collapsible">

<div class="collapsible-header">
<h2>Windows</h2>
<p>
<a href="https://github.com/MayaFlux/Weave/releases/download/v0.2.1/Weave-windows.zip"><strong>Download for Windows</strong></a>
</p>
<p class="hint">Click to expand</p>
</div>

<div class="collapsible-body">

### Install

1. Right-click → **Run as Administrator**
2. Accept UAC prompt
3. Choose a mode:
   - **Install MayaFlux**
   - **Create Project**

4. Installer performs:
   - System checks
   - Download MayaFlux (~1.5 MB)
   - Launch Vulkan SDK installer
     - Install all components **except ARM**

   - Install dependencies (~90 MB of DLLs)
   - Configure environment variables

5. **Restart your terminal**
6. Run Weave again and select **Create Project**

### Prerequisites

- Windows 10 or later (64-bit only)

### Create & Build

```powershell
Weave.exe   # then select "Create Project"

cd MyProject
mkdir build
cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . --config Release
.\Release\MyProject.exe
```

📘 **Full guide:**
[Windows Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/WINDOWS.md)

</div>
</div>

---

# Documentation

- **macOS Guide** — installation, troubleshooting, uninstall
- **Linux Guide** — distro notes, troubleshooting, uninstall
- **Windows Guide** — step-by-step walk-through
- **Package Overview** — FAQ, architecture
- **Development Guide** — build Weave from source

View all at:
➡️ [Weave Docs](https://github.com/MayaFlux/Weave/tree/main/docs)
