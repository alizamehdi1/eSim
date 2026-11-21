# eSim 2.5 Installation on Ubuntu 25.04: Issues and Fixes

**Task:** Task 4 — eSim Upgradation (CSE and related fields)

**Author:** Aliza Mehdi

**Forked Repository:** https://github.com/alizamehdi1/eSim

**Branch:** `installers`

**Date:** 21 November 2026

## 1. Objective & Summary

The objective of this task is to test the installation of **eSim 2.5** on **Ubuntu 25.04 (`plucky`)**, identify OS- and dependency-level issues in the installer scripts, implement clean automated fixes on the `installers` branch, and document the methodology and verification results.

All **8 issues** encountered during the initial installation and subsequent reinstallation attempts (starting with Issue 6 and the following errors) were resolved through code updates in the installer scripts, resulting in a complete, working installation of eSim 2.5 (including KiCad, NGHDL, Ngspice, and SKY130 PDK).

| \# | Issue Summary | Component Affected | Severity / Impact | Status | 
| ----- | ----- | ----- | ----- | ----- | 
| 1 | Main installer exits with `"Unsupported Ubuntu version: 25.04 ()"` | Core eSim Installer (`install-eSim.sh`) | **Critical** — Blocks entire installation | Fixed | 
| 2 | KiCad installation aborts with `404 Not Found` on legacy KiCad PPA | KiCad (Core Schematic/PCB Editor) | **Critical** — Halts core GUI/EDA setup | Fixed | 
| 3 | NGHDL sub-installer fails due to missing `25.04` script and dispatch case | NGHDL (Mixed-Signal Simulation) | **High** — Blocks NGHDL & Ngspice setup | Fixed | 
| 4 | NGHDL fails on missing `libcanberra-gtk-module` package | NGHDL Dependencies | **High** — Aborts NGHDL installation | Fixed | 
| 5 | GHDL 4.1.0 build fails with `"Unhandled version llvm 20.1.2"` | GHDL / NGHDL Compilation | **High** — Aborts VHDL compiler build | Fixed | 
| 6 | *(During reinstallation)* GHDL `./configure` fails with `cannot run /usr/bin/llvm-config-18-18` | GHDL / NGHDL Configuration (`installNghdl`) | **High** — Aborts GHDL `configure` step | Fixed | 
| 7 | *(During reinstallation)* GHDL `make` fails on interactive `mv` prompt for `grt/ghdl_main.ali` | GHDL / NGHDL Build (`installNghdl`) | **High** — Aborts GHDL compilation at `Makefile.inc:158` | Fixed | 
| 8 | *(During reinstallation)* Inline Python script raises `NameError: name 'LLVM_BIN' is not defined` | Installer Patching (`installNghdl`) | **High** — Aborts script patching before GHDL build | Fixed | 

## 2. Test Environment & Technologies Used

* **Operating System:** Ubuntu 25.04 (`plucky`, 64-bit) running on Oracle VirtualBox (8 GB RAM, 4 vCPUs, 60 GB disk) — verified via `lsb_release -a`.

* **eSim Version:** eSim 2.5 (`installers` branch)

* **Technologies Used:** Bash Scripting, Python, eSim, KiCad, NGHDL/GHDL, LLVM, Git, `apt`

## 3. Methodology

1. **Baseline Setup:** Installed a clean Ubuntu 25.04 VM in VirtualBox and created a baseline snapshot to ensure reproducible testing.

2. **Repository Setup:** Forked `FOSSEE/eSim`, cloned the repository locally, and checked out the `installers` branch.

3. **Diagnostic Execution:** Ran the installer while logging all terminal output:

   ```bash
   chmod +x install-eSim.sh
   ./install-eSim.sh --install 2>&1 | tee install-log.txt
   ```

4. **Root-Cause Analysis & Iterative Patching:** Traced each error to its exact line in `install-eSim.sh`, `install-eSim-scripts/install-eSim-25.04.sh`, or `nghdl/install-nghdl.sh`, implemented automated fixes in `installNghdl` and the supporting scripts for Ubuntu 25.04 without altering scripts for older Ubuntu releases, and re-tested across installation and reinstallation runs.

5. **End-to-End Verification:** Confirmed full installation completion and launched the eSim GUI (`esim`) from the terminal.

## 4. Issues Faced and Fixes Applied

### Issue 1: Main Installer Rejects Ubuntu 25.04 (`Unsupported Ubuntu version`)

* **Error Observed:**

  ```text
  Detected Ubuntu Version:
  Unsupported Ubuntu version: 25.04 ()
  ```

* **Root Cause:**

  `install-eSim.sh` checks `$VERSION_ID` from `/etc/os-release` against a hardcoded `case` block that only supports `22.04`, `23.04`, and `24.04`. Ubuntu `25.04` hits the default `*)` branch and exits immediately.

* **Fix:**

  1. Created `install-eSim-scripts/install-eSim-25.04.sh` from the existing installer template and made it executable.

  2. Added a `"25.04")` dispatch entry in `install-eSim.sh`:

     ```bash
     "25.04")
         SCRIPT="$SCRIPT_DIR/install-eSim-25.04.sh"
         ;;
     ```

* **Verification:** `install-eSim.sh` detects `25.04` and launches `install-eSim-scripts/install-eSim-25.04.sh`.

### Issue 2: KiCad Installation Fails Due to Unsupported PPA (`404 Not Found`)

* **Error Observed:**

  ```text
  Adding KiCad PPA to local apt repository: kicad/kicad-6.0-releases
  Err:6 https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release
    404  Not Found
  E: The repository 'https://ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu plucky Release' does not have a Release file.
  ```

* **Root Cause:**

  In `installKicad`, any Ubuntu version not explicitly matched falls into an `else` block that adds `ppa:kicad/kicad-6.0-releases`, which has no packages for Ubuntu 25.04 (`plucky`). However, Ubuntu 25.04's official `plucky/universe` repository already includes KiCad 8 (`8.0.8+dfsg-1`) and all required library packages (`kicad-footprints`, `kicad-symbols`, `kicad-templates`, `kicad-libraries`). Additionally, `copyKicadLibrary` and `--uninstall` referenced `~/.config/kicad/6.0` instead of `~/.config/kicad/8.0`.

* **Fix:**

  1. Removed the stale PPA entry left by the failed attempt (`/etc/apt/sources.list.d/kicad-ubuntu-kicad-6_0-releases-plucky.sources`).

  2. Updated `installKicad` in `install-eSim-25.04.sh` to skip adding any external PPA (`kicadppa=""`) on Ubuntu 25.04 and only invoke `add-apt-repository` when `kicadppa` is non-empty, allowing `apt-get` to install KiCad 8 directly from Ubuntu's `plucky/universe` repository.

  3. Updated KiCad configuration paths in `copyKicadLibrary` and `--uninstall` from `~/.config/kicad/6.0` to `~/.config/kicad/8.0`.

* **Verification:** KiCad `8.0.8+dfsg-1` and its symbol/footprint libraries installed cleanly and copied the eSim symbol table to `~/.config/kicad/8.0/`.

### Issue 3: NGHDL Sub-Installer Exits with `Unsupported Ubuntu version: 25.04`

* **Error Observed:**

  ```text
  Installing NGHDL...........................
  Detected Ubuntu Version:
  Unsupported Ubuntu version: 25.04 ()
  ```

* **Root Cause:**

  `installNghdl` extracts `nghdl.zip` and runs `nghdl/install-nghdl.sh --install`. Inside `nghdl.zip`, `install-nghdl-scripts/` lacks an `install-nghdl-25.04.sh` script, and `install-nghdl.sh` lacks a `"25.04")` `case` branch.

* **Fix:**

  Updated `installNghdl` in `install-eSim-25.04.sh` to automatically generate `install-nghdl-25.04.sh` and inject a `"25.04")` branch into `install-nghdl.sh` using `awk` right after `unzip -o nghdl.zip`:

  ```bash
  if [ ! -f install-nghdl-scripts/install-nghdl-25.04.sh ] && [ -f install-nghdl-scripts/install-nghdl-24.04.sh ]; then
      cp install-nghdl-scripts/install-nghdl-24.04.sh install-nghdl-scripts/install-nghdl-25.04.sh
  fi

  if ! grep -q '"25\.04")' install-nghdl.sh; then
      awk '
          /"24\.04"\)/ { found24 = 1 }
          found24 && /^[[:space:]]*;;/ {
              print
              print "    \"25.04\")"
              print "        SCRIPT=\"$SCRIPT_DIR/install-nghdl-25.04.sh\""
              print "        ;;"
              found24 = 0
              next
          }
          { print }
      ' install-nghdl.sh > install-nghdl.sh.tmp && mv install-nghdl.sh.tmp install-nghdl.sh
  fi
  ```

* **Verification:** `install-nghdl.sh` recognizes Ubuntu `25.04` and runs `install-nghdl-25.04.sh`.

### Issue 4: Missing Package `libcanberra-gtk-module` During NGHDL Setup

* **Error Observed:**

  ```text
  Installing Gtk Canberra modules...........................
  Package libcanberra-gtk-module is not available, but is referred to by another package.
  Error: Package 'libcanberra-gtk-module' has no installation candidate
  ```

* **Root Cause:**

  `libcanberra-gtk-module` (GTK 2) was deprecated and removed from Ubuntu 25.04 (`plucky`). Its GTK 3 equivalent, `libcanberra-gtk3-module`, is available in Ubuntu 25.04's repositories.

* **Fix:**

  Added an automated `sed` patch inside `installNghdl` in `install-eSim-25.04.sh` to replace `libcanberra-gtk-module` with `libcanberra-gtk3-module` in `install-nghdl-25.04.sh`:

  ```bash
  sed -i 's/\blibcanberra-gtk-module\b/libcanberra-gtk3-module/g' install-nghdl-scripts/install-nghdl-25.04.sh
  ```

* **Verification:** The GTK Canberra dependency step completes without error during the patched `install-nghdl-25.04.sh` run (`screenshots/Resolution%20of%20NGHDL%20package%20error.png`).

### Issue 5: GHDL 4.1.0 Fails to Configure Against Default LLVM 20.1.2

* **Error Observed:**

  ```text
  Changing directory to ghdl-4.1.0 installation
  Configuring ghdl-4.1.0 build as per requirements
  Use full IEEE library
  Build machine is: x86_64-linux-gnu
  Unhandled version llvm 20.1.2
  ```

* **Root Cause:**

  Ubuntu 25.04 ships with LLVM `20.1.2` as `/usr/bin/llvm-config`, whereas GHDL `4.1.0`'s `./configure` script does not support LLVM 20. However, Ubuntu 25.04's `plucky/universe` repository provides `llvm-18` and `llvm-18-dev`.

* **Fix:**

  Updated `installNghdl` in `install-eSim-25.04.sh` to install `llvm-18` and `llvm-18-dev` if not present, detect the available LLVM 18 binary (`/usr/bin/llvm-config-18` or `/usr/lib/llvm-18/bin/llvm-config`), and update `--with-llvm-config=` in `install-nghdl-25.04.sh` automatically during execution.

* **Verification:** GHDL 4.1.0 moves past the LLVM 20 rejection and uses LLVM 18 during configuration.

### Issue 6: (During Reinstallation) Repeated Path Patching Produces `/usr/bin/llvm-config-18-18` in GHDL `./configure`

* **Error Observed:**

  ```text
  Use full IEEE library
  Build machine is: x86_64-linux-gnu
  cannot run /usr/bin/llvm-config-18-18


  Error! Kindly resolve above error(s) and try again.

  Aborting Installation...
  ```

* **Root Cause:**

  During reinstallation, `install-nghdl-25.04.sh` already existed on disk from an earlier run and contained a duplicated `-18-18` suffix (`--with-llvm-config=/usr/bin/llvm-config-18-18`). Because the script only copied `install-nghdl-24.04.sh` when `install-nghdl-25.04.sh` did not exist and earlier replacement logic did not collapse repeated `-18` suffixes, reinstallation kept the broken `/usr/bin/llvm-config-18-18` path.

* **Fix:**

  Updated the automated patching code inside `installNghdl` in `install-eSim-25.04.sh` to collapse any repeated `llvm-config-18(?:-18)+` back to `llvm-config-18` and rewrite `--with-llvm-config=` to the detected LLVM 18 executable path (`LLVM_BIN`) on every run:

  ```python
  text = re.sub(r"llvm-config-18(?:-18)+", "llvm-config-18", text)
  text = re.sub(r"(--with-llvm-config=)\S+", rf"\g<1>{llvm_bin}", text)
  ```

* **Verification:** `installNghdl` automatically normalizes `install-nghdl-25.04.sh` to `/usr/bin/llvm-config-18` (or `/usr/lib/llvm-18/bin/llvm-config`) before running GHDL `./configure`.

### Issue 7: (During Reinstallation) Interactive `mv` Prompt During GHDL Build (`grt/ghdl_main.ali`)

* **Error Observed:**

  ```text
  w.X  -cargs  -fPIC
  gnatmake: objects up to date.
  sed -e '/^P /s/P /P NR /' < grt/ghdl_main.ali > grt/ghdl_main-tmp.ali
  mv grt/ghdl_main-tmp.ali grt/ghdl_main.ali
  mv: replace 'grt/ghdl_main.ali', overriding mode 0644 (rw-r--r--)?
  make: *** [src/grt/Makefile.inc:158: grt/run-bind.adb] Error 1
  ```

* **Root Cause:**

  During reinstallation, leftover build/ALI artifacts inside `ghdl-4.1.0` from earlier interrupted runs or an active `mv` alias caused `mv grt/ghdl_main-tmp.ali grt/ghdl_main.ali` to prompt interactively during `make`, failing at `src/grt/Makefile.inc:158`.

* **Fix:**

  Added automated cleanup code inside `installNghdl` in `install-eSim-25.04.sh` right before invoking `./install-nghdl.sh --install` to remove stale `ghdl-4.1.0` build artifacts and clear any `mv` alias:

  ```bash
  # Remove leftover GHDL build/ALI artifacts from interrupted runs so mv never prompts
  rm -rf ghdl-4.1.0 2>/dev/null || true
  unalias mv 2>/dev/null || true
  ```

* **Verification:** Reinstallation removes stale `ghdl-4.1.0` build state and unaliases `mv` automatically, allowing `make` to complete without interactive prompts.

### Issue 8: (During Reinstallation) `NameError` for `LLVM_BIN` in Inline Python Patching Block

* **Error Observed:**

  ```text
  Traceback (most recent call last):
    File "<stdin>", line 7, in <module>
  NameError: name 'LLVM_BIN' is not defined


  ```

* **Root Cause:**

  During reinstallation testing of the updated `installNghdl` code, `LLVM_BIN` was set as a Bash variable, but the inline `python3` snippet referenced `LLVM_BIN` directly as a Python variable instead of receiving it from the shell.

* **Fix:**

  Updated the inline `python3` invocation inside `installNghdl` in `install-eSim-25.04.sh` to pass `"$LLVM_BIN"` as a command-line argument (`sys.argv[1]`) into the single-quoted `'PY'` heredoc:

  ```bash
  if [ -n "$LLVM_BIN" ]; then
      python3 - "$LLVM_BIN" <<'PY'
  import re
  import sys
  from pathlib import Path

  llvm_bin = sys.argv[1]
  p = Path("install-nghdl-scripts/install-nghdl-25.04.sh")
  text = p.read_text(encoding="utf-8")
  text = re.sub(r"llvm-config-18(?:-18)+", "llvm-config-18", text)
  text = re.sub(r"(--with-llvm-config=)\S+", rf"\g<1>{llvm_bin}", text)
  p.write_text(text, encoding="utf-8")
  PY
  fi
  ```

### Final `installNghdl` Function Code (`install-eSim-25.04.sh`)

```bash
function installNghdl
{

    echo "Installing NGHDL..........................."

    unzip -o nghdl.zip
    cd nghdl/ || return 1

    # Only generate 25.04 from 24.04 if 25.04 does not already exist
    if [ ! -f install-nghdl-scripts/install-nghdl-25.04.sh ] && [ -f install-nghdl-scripts/install-nghdl-24.04.sh ]; then
        cp install-nghdl-scripts/install-nghdl-24.04.sh install-nghdl-scripts/install-nghdl-25.04.sh
    fi

    if [ -f install-nghdl-scripts/install-nghdl-25.04.sh ]; then
        # Replace removed GTK2 libcanberra-gtk-module with libcanberra-gtk3-module on Ubuntu 25.04
        sed -i 's/\blibcanberra-gtk-module\b/libcanberra-gtk3-module/g' install-nghdl-scripts/install-nghdl-25.04.sh

        # Ensure llvm-18 is available and point GHDL configure to llvm-config-18 instead of LLVM 20
        if [ ! -x /usr/bin/llvm-config-18 ] && [ ! -x /usr/lib/llvm-18/bin/llvm-config ]; then
            sudo apt-get update && sudo apt-get install -y llvm-18 llvm-18-dev
        fi

        LLVM_BIN=""
        if [ -x /usr/bin/llvm-config-18 ]; then
            LLVM_BIN="/usr/bin/llvm-config-18"
        elif [ -x /usr/lib/llvm-18/bin/llvm-config ]; then
            LLVM_BIN="/usr/lib/llvm-18/bin/llvm-config"
        fi

        if [ -n "$LLVM_BIN" ]; then
            python3 - "$LLVM_BIN" <<'PY'
import re
import sys
from pathlib import Path

llvm_bin = sys.argv[1]
p = Path("install-nghdl-scripts/install-nghdl-25.04.sh")
text = p.read_text(encoding="utf-8")
text = re.sub(r"llvm-config-18(?:-18)+", "llvm-config-18", text)
text = re.sub(r"(--with-llvm-config=)\S+", rf"\g<1>{llvm_bin}", text)
p.write_text(text, encoding="utf-8")
PY
        fi
    fi

    if ! grep -q '"25\.04")' install-nghdl.sh; then
        awk '
            /"24\.04"\)/ { found24 = 1 }
            found24 && /^[[:space:]]*;;/ {
                print
                print "    \"25.04\")"
                print "        SCRIPT=\"$SCRIPT_DIR/install-nghdl-25.04.sh\""
                print "        ;;"
                found24 = 0
                next
            }
            { print }
        ' install-nghdl.sh > install-nghdl.sh.tmp && mv install-nghdl.sh.tmp install-nghdl.sh
    fi

    # Remove leftover GHDL build/ALI artifacts from interrupted runs so mv never prompts
    rm -rf ghdl-4.1.0 2>/dev/null || true
    unalias mv 2>/dev/null || true

    chmod +x install-nghdl.sh install-nghdl-scripts/install-nghdl-25.04.sh 2>/dev/null

    trap "" ERR
    ./install-nghdl.sh --install
    trap error_exit ERR

    ngspiceFlag=1
    cd ../

}
```

* **Verification:** The inline Python patcher runs without `NameError`, updates `install-nghdl-25.04.sh` cleanly, and allows NGHDL installation to complete.

## 5. Final Verification & Submission Details

* **Installation Result:** Completed with `-----------------eSim Installed Successfully-----------------` after Ngspice, NGHDL, SKY130 PDK, and desktop/menu shortcuts were set up.

* **Application Launch & GUI:** Running `esim` (lowercase `esim` rather than `eSim`) starts eSim 2.5, initializes `/home/aliza-mehdi/eSim-Workspace`, and opens the **eSim-2.5** GUI window along with the Desktop launcher icon.

* **Repository & Branch:** All updated installer scripts (`install-eSim.sh` and `install-eSim-scripts/install-eSim-25.04.sh`) are committed to the `installers` branch of https://github.com/alizamehdi1/eSim.