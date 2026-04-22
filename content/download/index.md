---
title: "Download"
---

<div class="card collapsible">
<div class="collapsible-header">
  <h2>External Dependencies (Platform-Imposed)</h2>
</div>
<p class="hint">Click to expand</p>

<div class="collapsible-body">

  <p>
    MayaFlux itself does <strong>not</strong> require external IDEs or large SDK
    bundles. Weave installs all internal framework dependencies across all
    supported platforms.
  </p>

  <p>
    However, some operating systems enforce their own compiler toolchains. This
    is <strong>not</strong> a MayaFlux design choice, it is how those platforms are structured.
  </p>

<hr />

  <h3>Linux</h3>

  <p>Linux behaves as expected:</p>

  <ul>
    <li>A C/C++ compiler</li>
    <li>Standard C/C++ libraries</li>
  </ul>

  <p>
    On supported distributions, these are already present or installed via the
    system package manager. Weave installs MayaFlux-specific dependencies
    automatically.
  </p>

  <p>
    <strong>No IDE required. No heavyweight SDK required.</strong>
  </p>

  <hr />

  <h3>Windows</h3>

  <p>Windows requires Microsoft's toolchain.</p>

  <p>You must install:</p>

  <ul>
    <li><strong>Visual Studio 2022</strong></li>
    <li>The <strong>"Desktop development with C++"</strong> workload</li>
  </ul>

  <h4>How to Install</h4>

  <ol>
    <li>Download Visual Studio 2022 from Microsoft</li>
    <li>Run the installer</li>
    <li>Select <strong>Desktop development with C++</strong></li>
    <li>Complete installation</li>
  </ol>

  <p>
    This requirement is enforced by Windows' compiler ecosystem. MayaFlux uses
    MSVC on Windows because that is the officially supported toolchain.
  </p>

  <p>
    Yes, this is a large installation. No, MayaFlux cannot replace Microsoft's
    compiler.
  </p>

  <hr />

  <h3>macOS</h3>

  <p>macOS requires Apple’s toolchain.</p>

  <p>You must install:</p>

  <ul>
    <li><strong>Xcode Command Line Tools</strong></li>
  </ul>

  <h4>How to Install</h4>

```bash
xcode-select --install
```

<p>
Follow the system dialog to complete installation.
</p>

<p>
Apple does not allow third-party compilers to fully replace the system toolchain.
MayaFlux uses Clang via Apple's command line tools because that is how macOS is designed.
</p>

<hr>

<h3>Important Clarification</h3>

<p>
Weave installs:
</p>

<ul>
<li>MayaFlux libraries</li>
<li>Vulkan components</li>
<li>Internal runtime dependencies</li>
<li>Project templates</li>
</ul>

<p>
It does <strong>not</strong> replace or bundle:
</p>

<ul>
<li>Microsoft’s compiler on Windows</li>
<li>Apple’s compiler on macOS</li>
</ul>

<p>
Those ecosystems require their own development toolchains.
Linux remains the only platform where a minimal compiler + standard library setup is sufficient.
</p>

</div>
</div>

<br />

# Get Started with Weave (Stable channel)

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
<a href="https://github.com/MayaFlux/Weave/releases/download/v0.3.1/Weave-macos.dmg"><strong>Download for macOS</strong></a>
</p>
<p class="hint">Click to expand</p>
</div>

<div class="collapsible-body">

### Install

1. Download and open `Weave-macos.dmg`
2. Double-click **Weave.app**
3. If macOS shows **"unverified developer"**:
   - Close dialog
   - System Settings → **Privacy & Security**
   - Find **Weave** → **Open Anyway**
4. Choose **"Install MayaFlux"**
5. Select a release channel and follow the prompts
6. Restart your terminal when done

### Prerequisites

- macOS 15.0 (Sequoia) or later

### Contents

- **MayaFlux** via Homebrew
- CLI tool (`~/.local/bin/weave`)
- Environment configured in `~/.zshenv`

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

<!-- ====================== -->

<!-- Linux CARD             -->

<!-- ====================== -->

<div class="card collapsible">

<div class="collapsible-header">
<h2>Linux</h2>
<p>
<a href="https://github.com/MayaFlux/Weave/releases/download/v0.3.1/Weave-linux.tar.gz"><strong>Download for Linux</strong></a>
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

<!-- ====================== -->

<!-- Windows CARD           -->

<!-- ====================== -->

<div class="card collapsible">

<div class="collapsible-header">
<h2>Windows</h2>
<p>
<a href="https://github.com/MayaFlux/Weave/releases/download/v0.3.1/Weave-windows.zip"><strong>Download for Windows</strong></a>
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
