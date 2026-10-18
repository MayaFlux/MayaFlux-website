---
title: "Download"
---

# Download Weave 0.5

Download Weave for your operating system, open it, and choose **Install MayaFlux**. Weave installs MayaFlux and everything the framework needs. You can then choose **Create Project** to make a project with CMake presets. Use any editor or IDE you prefer.

---

<!-- ====================== -->
<!-- macOS CARD             -->
<!-- ====================== -->

<div class="card collapsible download-details-card">
<div class="collapsible-header">
  <h2>macOS</h2>
  <p><a href="https://github.com/MayaFlux/Weave/releases/download/v0.5.0/Weave-macos.dmg"><strong>Download Weave 0.5 for macOS</strong></a></p>
  <p class="hint">Click to expand</p>
</div>
<div class="collapsible-body">

**Requires:** macOS 26 (Tahoe) or later on Apple silicon (arm64).

### Install

1. Open `Weave-macos.dmg` and drag **Weave.app** to **Applications**.
2. Open **Weave.app**. If macOS blocks it, go to **System Settings > Privacy & Security**, click [**Open Anyway**](https://support.apple.com/en-us/102445), then click **Open** in the confirmation dialog. [Other open-source macOS releases](https://github.com/neovim/neovim/releases) document the same routine step.
3. Choose **Install MayaFlux** and follow the prompts. Restart your terminal when installation finishes.

<details>
<summary>Why does macOS show this warning?</summary>
<p>MayaFlux does not pay Apple's Developer Program fee for Developer ID notarization. The fee is a financial burden, but our refusal is also a deliberate stance. Requiring community projects to pay for ordinary access to users is hostile to open knowledge and the spirit of free software. We regard it as "protection money".</p>
<p>Many independent open-source projects make the same choice for financial and principled reasons. MayaFlux is part of that broader community. Because Apple has not notarized this build, macOS may say it cannot check Weave for malicious software. This warning does not mean macOS found malware.</p>
</details>

### Create and build a project

**GUI:** Open **Weave.app**, choose **Create Project**, enter a name, and select a destination.

**Optional CLI:** You can create the same project from a terminal:

```bash
weave new MyProject ~/Projects/
```

**Build either way:** Open a terminal in the new project folder and run:

```bash
cmake --preset release
cmake --build --preset release
```

Run the executable named after your project from the `build/` folder.

📘 [macOS Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/MACOS.md)

<details>
<summary>Homebrew reports missing or outdated Xcode Command Line Tools</summary>
<p>If the tools are missing, run <code>xcode-select --install</code> and retry. If Homebrew says they are outdated, follow its update instructions and retry.</p>
</details>

</div>
</div>

<!-- ====================== -->

<!-- Linux CARD             -->

<!-- ====================== -->

<div class="card collapsible">
<div class="collapsible-header">
  <h2>Linux</h2>
  <p><a href="https://github.com/MayaFlux/Weave/releases/download/v0.5.0/Weave-linux.AppImage"><strong>Download Weave 0.5 for Linux</strong></a></p>
  <p class="hint">Click to expand</p>
</div>
<div class="collapsible-body">

**Requires:** Linux x86_64 on Arch Linux, Fedora 43+, or Ubuntu 25+.

### Install

```bash
chmod +x Weave-linux.AppImage
./Weave-linux.AppImage
```

Choose **Install MayaFlux** and follow the prompts. Weave uses your distribution's package manager to install MayaFlux and its dependencies. Enter your sudo password when prompted; Weave does not store it.

### Create and build a project

**GUI:** Open Weave, choose **Create Project**, enter a name, and select a destination.

**Optional CLI:** You can create the same project from a terminal:

```bash
weave new MyProject ~/Projects/
```

**Build either way:** Open a terminal in the new project folder and run:

```bash
cmake --preset release
cmake --build --preset release
```

Run the executable named after your project from the `build/` folder.

📘 [Linux Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/LINUX.md)

</div>
</div>

<!-- ====================== -->

<!-- Windows CARD           -->

<!-- ====================== -->

<div class="card collapsible download-details-card">
<div class="collapsible-header">
  <h2>Windows</h2>
  <p><a href="https://github.com/MayaFlux/Weave/releases/download/v0.5.0/Weave-windows.zip"><strong>Download Weave 0.5 for Windows</strong></a></p>
  <p class="hint">Click to expand</p>
</div>
<div class="collapsible-body">

**Requires:** Windows 10 or 11 (64-bit).

### Install

1. Extract `Weave-windows.zip`
2. Open `Weave.exe` and choose **Install MayaFlux**
3. Accept the UAC prompt when Weave requests administrator permission for installation
4. Follow the prompts and restart your terminal when done

Weave installs the required Microsoft C++ components if they are missing. You **do not** need to install Visual Studio.

<details>
<summary>Why does Weave install MSVC Build Tools, Windows SDK, and VC++ Redistributable?</summary>
<p>MayaFlux uses native Windows backends, including WASAPI, Win32 windowing, WinMM, and COM, to maximize compatibility and performance. Building against these platform APIs needs Windows SDK files and Microsoft C++ build tools. Weave requests only the components its workflow needs: x64 C++ tools, the Windows SDK, and the current VC++ Redistributable. It does not download the full Visual Studio IDE or its roughly 10 GB default installation. These components can still take time and disk space to install.</p>
</details>

### Create and build a project

Open `Weave.exe`, choose **Create Project**, enter a name, and select a destination. Project creation on Windows is handled in the Weave GUI. Open PowerShell in the new project folder to build it:

```powershell
cmake --preset release
cmake --build --preset release
```

Run the executable named after your project from the `build\` folder.

📘 [Windows Documentation](https://github.com/MayaFlux/Weave/tree/main/docs/WINDOWS.md)

</div>
</div>

---

## Check a new project

Each project created with Weave 0.5 includes a validation script. Run `./validate.sh` on macOS or Linux, or `.\validate.bat` on Windows, from the project folder to check its files, tools, and MayaFlux installation. After building, use `./run-report.sh` or `.\run-report.bat` to collect logs when reporting a problem. These scripts are included in newly created projects; existing projects keep their files.

---

# Documentation

- **macOS Guide:** installation, troubleshooting, uninstall
- **Linux Guide:** distro notes, troubleshooting, uninstall
- **Windows Guide:** step-by-step walk-through
- **Package Overview:** FAQ, architecture
- **Development Guide:** build Weave from source

View all at:
➡️ [Weave Docs](https://github.com/MayaFlux/Weave/tree/main/docs)
