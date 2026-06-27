---
title: "Download"
---

<div class="card collapsible">
<div class="collapsible-header">
  <h2>External Dependencies (Platform-Imposed, Required)</h2>
</div>
<p class="hint">Click to expand</p>
<div class="collapsible-body">

  <p>MayaFlux itself does <strong>not</strong> require external IDEs or large SDK bundles. Weave installs all internal framework dependencies across all supported platforms.</p>
  <p>However, some operating systems enforce their own compiler toolchains. This is <strong>not</strong> a MayaFlux design choice. It is how those platforms are structured.</p>

  <hr />

  <h3>Linux</h3>
  <p>No IDE required. No heavyweight SDK required. On supported distributions, a C/C++ compiler and standard libraries are already present or installed via the system package manager. Weave installs MayaFlux-specific dependencies automatically.</p>

  <hr />

  <h3>Windows</h3>
  <p>Windows requires Microsoft's compiler toolchain. Weave installs it automatically via winget if not already present. No manual Visual Studio installation is needed.</p>

  <hr />

  <h3>macOS</h3>
  <p>macOS requires Apple's command line tools. Install them with:</p>

```bash
xcode-select --install
```

  <p>Follow the system dialog to complete installation. Apple does not allow third-party compilers to fully replace the system toolchain.</p>

  <hr />

  <h3>What Weave installs</h3>
  <ul>
    <li>MayaFlux libraries</li>
    <li>Vulkan components</li>
    <li>Internal runtime dependencies</li>
    <li>Project templates</li>
  </ul>

</div>
</div>

<br />

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
  <p><a href="https://github.com/MayaFlux/Weave/releases/download/v0.4.0/Weave-macos.dmg"><strong>Download for macOS</strong></a></p>
  <p class="hint">Click to expand</p>
</div>
<div class="collapsible-body">

### Prerequisites

- macOS 15.0 (Sequoia) or later
- Xcode Command Line Tools: `xcode-select --install`

### Install

1. Open `Weave-macos.dmg` and double-click **Weave.app**
2. If macOS shows an "unverified developer" warning: close it, go to **System Settings > Privacy & Security**, find Weave, click **Open Anyway**
3. Choose **Install MayaFlux** and follow the prompts
4. Restart your terminal when done

### Create and build a project

```bash
weave new MyProject ~/Projects/

cd MyProject
cmake --preset release
cmake --build --preset release
./build/MyProject
```

📘 [macOS Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/MACOS.md)

</div>
</div>

<!-- ====================== -->

<!-- Linux CARD             -->

<!-- ====================== -->

<div class="card collapsible">
<div class="collapsible-header">
  <h2>Linux</h2>
  <p><a href="https://github.com/MayaFlux/Weave/releases/download/v0.4.0/Weave-linux.AppImage"><strong>Download for Linux</strong></a></p>
  <p class="hint">Click to expand</p>
</div>
<div class="collapsible-body">

### Supported distributions

- Arch Linux
- Fedora 43+
- Ubuntu 25+

### Install

```bash
chmod +x Weave-linux.AppImage
./Weave-linux.AppImage
```

Choose **Install MayaFlux** and follow the prompts. Weave detects your distribution and installs via the native package manager. Enter your sudo password when prompted — it is not stored.

### Create and build a project

```bash
weave new MyProject ~/Projects/

cd MyProject
cmake --preset release
cmake --build --preset release
./build/MyProject
```

📘 [Linux Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/LINUX.md)

</div>
</div>

<!-- ====================== -->

<!-- Windows CARD           -->

<!-- ====================== -->

<div class="card collapsible">
<div class="collapsible-header">
  <h2>Windows</h2>
  <p><a href="https://github.com/MayaFlux/Weave/releases/download/v0.4.0/Weave-windows.zip"><strong>Download for Windows</strong></a></p>
  <p class="hint">Click to expand</p>
</div>
<div class="collapsible-body">

### Prerequisites

- Windows 10 or 11 (64-bit)
- MSVC Build Tools: installed automatically by Weave if not already present

### Install

1. Extract `Weave-windows.zip`
2. Right-click `Weave.exe` and choose **Run as Administrator**
3. Accept the UAC prompt
4. Choose **Install MayaFlux** and follow the prompts
5. Restart your terminal when done

### Create and build a project

All project operations go through the Weave GUI on Windows. Open `Weave.exe` and choose **Create Project**.

```powershell
cd MyProject
cmake --preset release
cmake --build --preset release
.\build\MyProject.exe
```

📘 [Windows Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/WINDOWS.md)

</div>
</div>

---

# Documentation

- **macOS Guide:** installation, troubleshooting, uninstall
- **Linux Guide:** distro notes, troubleshooting, uninstall
- **Windows Guide:** step-by-step walk-through
- **Package Overview:** FAQ, architecture
- **Development Guide:** build Weave from source

View all at:
➡️ [Weave Docs](https://github.com/MayaFlux/Weave/tree/main/docs)
