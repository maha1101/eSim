# eSim 2.5 on Ubuntu 25.04: Installation Issues Report

**Task:** eSim Semester Long Internship, Autumn 2026, Task 4 (eSim Upgradation)
**Author:** Mahalakshmi Balimelu
**Fork:** https://github.com/maha1101/eSim (branch `installers`)

All commit hashes below are in this fork. Raw evidence (installer logs and package state) is in this `report/` folder: `esim-install.log` (first run), `esim-install2.log` (second run) and `evidence.txt`.

## 1. Environment

| Item | Value |
|---|---|
| OS | Ubuntu 25.04 (plucky), VirtualBox VM |
| Architecture | x86_64 |
| Python | 3.13.3 |
| RAM / Swap | 2.6 GiB / 2.9 GiB (swap already in use) |
| CPU cores | 2 |
| Disk | 30 GB (15 GB free) |
| eSim source | Tag `v2.5`, obtained by cloning the fork and checking out the tag (`git describe --tags` prints `v2.5`) |
| Installer | `Ubuntu/install-eSim.sh` and `Ubuntu/install-eSim-scripts/install-eSim-25.04.sh` from the `installers` branch |

## 2. Methodology

1. Forked eSim and checked out the `installers` branch.
2. Cloned the fork again, checked out tag `v2.5` as the source tree, and copied the `Ubuntu/` installer folder into it. The `installers` branch contains only installer scripts (no `library/`, `src/`, `images/`), so the installer has to be run from a full source tree, because it uses `pwd` as the eSim home.
3. Ran `./Ubuntu/install-eSim.sh --install 2>&1 | tee <log>`.
4. For each failure, found the cause in the script, patched it, committed, and reran. Each rerun got further, and each new failure was recorded as a separate issue.

## 3. Summary

| # | Issue | Severity | Fix commit | Evidence |
|---|---|---|---|---|
| 1 | Launcher `install-eSim.sh` has a syntax error (bad merge) | High | a3c5fd58 | Later runs start and reach the 25.04 script (both logs, line 179) |
| 2 | Ubuntu 25.04 version string not detected | Medium | a3c5fd58 | Same as above |
| 3 | KiCad PPA package needs `libgit2-1.8`, not available on 25.04 | High | 7879d714 | `evidence.txt`: KiCad 8.0.8 installed from the Ubuntu archive |
| 4 | Intermittent apt download failures (HTTP 503) | Low-Medium | c9af2632 | Preventive change; commit message records the 503s |
| 5 | Installer ends early when KiCad is already installed | Medium | 7879d714 | Both logs, lines 179-181: continues past KiCad into the library step |
| 6 | `kicadLibrary.tar.xz` missing from the v2.5 source tree | High | f466ae26 | Error in `esim-install.log` lines 181-182; fixed in `esim-install2.log` line 181 |
| 7 | KiCad config folder hardcoded to `6.0` | Medium | f466ae26 | `esim-install2.log` line 182 |
| 8 | `nghdl.zip` missing from the v2.5 source tree | High | d3c792d6 | Error in `esim-install2.log` line 185; fix not yet run to completion |

Issues 1 to 7 are fixed. Issue 8 has a fix in the script that is **not verified** (see section 4).

## 4. Issues in detail

### Issue 1: Launcher syntax error
- **File:** `Ubuntu/install-eSim.sh`
- **Symptom:** The launcher failed with a shell syntax error, so nothing ran. (The exact terminal output was not saved.)
- **Cause:** A bad merge left stale, duplicated installer code (functions such as `installNghdl`, `installKicad`, `installDependency`, `copyKicadLibrary`) inside `run_version_script`, directly after the `bash "$SCRIPT" "$ARGUMENT"` line, so the `if [[ -f "$SCRIPT" ]]` block was never closed.
- **Fix:** Removed the stale block (267 lines deleted, 9 added) and restored the `else` branch, which prints `Installation script not found: $SCRIPT` and exits with status 1.
- **Commit:** a3c5fd58

### Issue 2: Ubuntu 25.04 not detected
- **File:** `Ubuntu/install-eSim.sh`, function `get_ubuntu_version`
- **Symptom:** `FULL_VERSION` came out empty, so the matching version script could not be chosen.
- **Cause:** The regex `\d+\.\d+\.\d+` needs a three-part version, but on this machine `lsb_release` reports `Ubuntu 25.04`, which has two parts.
- **Fix:** Changed the regex to `\d+\.\d+(\.\d+)?`.
- **Commit:** a3c5fd58

### Issue 3: KiCad PPA dependency problem
- **File:** `Ubuntu/install-eSim-scripts/install-eSim-25.04.sh`, function `installKicad`
- **Symptom:** KiCad installation from the PPA failed.
- **Cause:** KiCad 8.0.9 from the PPA depends on `libgit2-1.8`, while Ubuntu 25.04 ships libgit2 1.9.
- **Fix:** Commented out `add-apt-repository` and pinned the companion packages (`kicad-footprints`, `kicad-libraries`, `kicad-symbols`, `kicad-templates`) to the Ubuntu archive 8.0.8 versions, alongside the existing `kicad=8.0.8+dfsg-1` pin.
- **Evidence:** `evidence.txt` shows `kicad 8.0.8+dfsg-1` and its four companion packages installed at 8.0.8. `apt-cache policy kicad` still lists the PPA's 8.0.9 as the candidate, which is why the explicit pins are needed.
- **Known leftover:** The message `Adding KiCad PPA to local apt repository` is still printed even though the PPA is no longer added (cosmetic).
- **Commit:** 7879d714

### Issue 4: Intermittent apt failures
- **File:** same script, same function
- **Symptom:** KiCad package downloads failed intermittently with HTTP 503 errors from the PPA.
- **Fix:** Added `-o Acquire::Retries=5`, HTTP and HTTPS timeouts of 30 seconds, and `--fix-missing` to the KiCad `apt-get install` command.
- **Note:** This is a robustness fix for a transient network error, so it cannot be demonstrated on demand.
- **Commit:** c9af2632

### Issue 5: Installer stops after KiCad
- **File:** same script, function `installKicad`
- **Symptom:** When KiCad was already installed, the installer ended after the KiCad step and never reached the remaining steps.
- **Cause:** The "KiCad 8.0 is already installed" branch used `exit 0`, which ends the entire script, not just the function.
- **Fix:** Changed `exit 0` to `return 0`.
- **Verification:** In both logs, `KiCad 8.0 is already installed.` (line 180) is followed by the next installer step (line 181).
- **Commit:** 7879d714

### Issue 6: `kicadLibrary.tar.xz` not found
- **File:** same script, function `copyKicadLibrary`
- **Symptom** (`esim-install.log`, lines 181-182):
  ```
  tar (child): library/kicadLibrary.tar.xz: Cannot open: No such file or directory
  tar (child): Error is not recoverable: exiting now
  ```
- **Cause:** The v2.5 source tree ships `library/kicadLibrary/` already extracted and has no `kicadLibrary.tar.xz`, but the function only knew how to extract the archive.
- **Fix:** The function now uses the archive if it exists (extracted into a temporary directory), otherwise the extracted `library/kicadLibrary/` folder, otherwise it stops with an error that names the missing paths and tells the user to run from the eSim source root. Only the temporary directory is removed afterwards.
- **Verification** (`esim-install2.log`, lines 181-182): `kicadLibrary.tar.xz not found, using extracted library/kicadLibrary`, then `Using KiCad config folder: /home/maha/.config/kicad/8.0`.
- **Commit:** f466ae26

### Issue 7: Wrong KiCad config directory
- **File:** same script, function `copyKicadLibrary`
- **Symptom:** The eSim symbol table was copied to `~/.config/kicad/6.0`, which KiCad 8 does not use (this VM has `~/.config/kicad/8.0`).
- **Cause:** The path was hardcoded for KiCad 6.
- **Fix:** The function reads the installed KiCad version with `dpkg-query` and uses `~/.config/kicad/<major.minor>`, defaulting to `8.0`.
- **Verification:** `esim-install2.log` line 182: `Using KiCad config folder: /home/maha/.config/kicad/8.0`.
- **Commit:** f466ae26

### Issue 8: `nghdl.zip` missing
- **File:** same script, function `installNghdl`
- **Symptom** (`esim-install2.log`, line 185 onward):
  ```
  Installing NGHDL...........................
  unzip:  cannot find or open nghdl.zip, nghdl.zip.zip or nghdl.zip.ZIP.

  Error! Kindly resolve above error(s) and try again.

  Aborting Installation...
  ```
- **Cause:** `installNghdl` runs `unzip -o nghdl.zip`, but the v2.5 source tree does not contain `nghdl.zip`. The only copy I found is under `MacOS/` in the fork. I did not check whether the official eSim release package contains it.
- **Fix:** `installNghdl` now uses `nghdl.zip` if present, otherwise an existing `nghdl/` directory, otherwise clones `https://github.com/FOSSEE/nghdl.git`, and exits with an error if that fails.
- **Status: not verified.** The patch passes `bash -n`, but I have not run the NGHDL build to completion on this VM (2.6 GiB RAM, 2 cores), so I cannot say whether the NGHDL installer itself works on Ubuntu 25.04.
- **Commit:** d3c792d6

## 5. Remaining work and limitations

- NGHDL installation is untested after the fix (Issue 8). NGHDL's own `install-nghdl.sh` may have further Ubuntu 25.04 problems that I did not reach.
- Issues 1 to 4: exact error output from before each fix was not saved, so those entries describe the cause found in the code, not captured terminal text.
- The `Adding KiCad PPA` message is still printed (Issue 3).
- The VM has limited memory, so heavy builds (NGHDL, ngspice, GHDL) may be killed.

## 6. How to reproduce

```bash
git clone -b installers https://github.com/maha1101/eSim.git eSim
git clone https://github.com/maha1101/eSim.git eSim-src
cd eSim-src && git checkout v2.5
cp -r ../eSim/Ubuntu .
chmod +x Ubuntu/install-eSim.sh
./Ubuntu/install-eSim.sh --install 2>&1 | tee ~/esim-install.log
```
