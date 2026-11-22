# NVIDIA 470.x EOL Driver Issues and the nouveau + Plasma 6 Wayland Resolution

> **About this report.** It documents the identical MacBook Pro 11,3 mentioned in the
> [README](../README.md#gpu-driver-use-nouveau-never-nvidias-driver-on-kepler), which ran
> NVIDIA's end-of-life 470xx driver for four months before moving to nouveau with Plasma 6
> Wayland. It's included here because it is the evidence behind this fork's "use nouveau"
> advice, and because two of its findings match this fork's changes: pausing the animated
> wallpaper while a window is maximised (issue N9), and the idle `plasmashell` CPU cost of
> the wallpaper's QML Canvas (Remaining caveats). The fan-control fix it credits to a
> patched mbpfan is now in upstream mbpfan 2.5.0 (2026-11-17).

Nov 21, 2026

## Executive summary

From 2026-07-08 to 2026-11-11, this MacBook Pro ran NVIDIA's end-of-life 470.xx legacy driver on its GeForce GT 750M. On 2026-11-11 it moved to nouveau with Plasma 6.8 Wayland on Fedora 45. Every 470-specific failure is now gone, and so is the custom tooling built to contain them.

The 470 era produced four kinds of problem:

- **Display corruption.** After GL-heavy apps or an S3 resume, KWin's X11 compositor came back dark, with ghosted double-stroke text. It took a journal-watching auto-restart daemon to keep the desktop usable.
- **Lock-in.** The 470 driver has no GBM, so Plasma Wayland was impossible. That forced X11. When Plasma 6.8 dropped X11 on 2026-11-05, the box had to be rolled back to a snapshot and package-pinned, which also blocked the Fedora 45 upgrade.
- **Boot fragility.** The eDP EDID was unreadable and had to be force-fed. Every kernel needed an akmod rebuild. A dnf deadlock hung boots, and a reconcile service had to heal GRUB entries on every boot.
- **Heat and performance.** The GPU stayed pinned at 405 MHz because PowerMizer ignored every override. Firefox busy-waited in the driver, and video ran at 96–99 °C package temperature.

nouveau with KWin Wayland removed the cause of each one. KWin Wayland runs on Mesa EGL/GBM, and the nouveau governor clocks the GPU properly. Mainline kernels need no out-of-tree module. The user's verdict on 2026-11-18: *"nouveau is light-years ahead of nvidia EOL drivers. All of my problems were from closed-source Nvidia drivers."*

Two problems from the same period turned out not to be GPU-related: the kernel 7.3 speaker regression and the kernel 7.3 fan-path change. They are listed separately so they aren't counted against either driver.

## System profile

The machine is a MacBook Pro Retina 15" (MacBookPro11,3, late 2013 to 2014). It has a muxed GPU pair switched by apple-gmux. The internal panel is wired to the NVIDIA GPU.

| Component | Detail |
| --- | --- |
| CPU / iGPU | Intel i7-4870HQ (Haswell, Crystal Well), Iris Pro 5200, PCI `8086:0d26` |
| dGPU | NVIDIA GeForce GT 750M Mac Edition, GK107M Kepler, PCI `10de:0fe9`, subsystem Apple `0130` |
| Panel | Apple "Color LCD" eDP, 2880×1800 at 60 Hz, EDID product `a022` |
| Display switching | apple-gmux, muxed (not muxless Optimus) |
| Boot chain | GRUB 2.14 built from source, LUKS-encrypted `/boot`, btrfs root |

Driver and desktop history:

| Period | OS / kernel | GPU driver | Desktop session |
| --- | --- | --- | --- |
| From 2026-11-11 (current) | Fedora 45, kernel 7.3.x (7.3.4 as of this report) | nouveau + Mesa 26.2.7 | Plasma 6.8.3, KWin Wayland |
| 2026-11-06 to 11-11 | Fedora 44, kernel 7.2.13 / 7.3.3 | nouveau test boots | Plasma 6.7.5 X11, then Wayland in the fair test |
| 2026-07-08 to 11-06 | Fedora 44, kernels 7.0 to 7.2 | NVIDIA 470.256.02 (akmod-nvidia-470xx), nouveau blacklisted | Plasma 6.7 X11 (kwin_x11) |
| Before 2026-07-08 | Fedora 44 | nouveau without the governor running | Plasma X11 |

470.xx is the last driver branch that supports Kepler. It gets no new features, and its last build in the repos dates from 2024.

## Issue catalogue (NVIDIA 470.xx era)

The report found twelve problems that trace to the 470 driver or to the constraints it imposed. Each one is listed with its symptom, root cause and the workaround that was in place.

### Display and compositor

**N1. KWin compositor corruption after a GL reset**

- **Symptom:** after a game exited, or after `glxinfo` or `sudo inxi -v8`, the whole screen went dark. Regular-weight text rendered as ghostly double strokes; bold and coloured text looked normal. Only logout fixed it at first.
- **Root cause:** a GL client attaching or detaching hung a GT 750M channel. The 470 driver then did a robustness reset, and kwin_x11 logged `A graphics reset attributable to the current GL context occurred`. KWin rebuilt its scene in place with the transfer function applied twice. A diff of `nvidia-settings -q all` and `xrandr --verbose` between good and broken states showed no change in the driver's display pipeline.
- **Workarounds:** `fixdisplay` (`kwin_x11 --replace`, bound to Meta+Shift+R), then `kwin-reset-heal.service`. That user unit followed the journal and restarted `plasma-kwin_x11.service` within about 2 s of a reset line.
- **Prevention failed:** runtime `GPUPowerMizerMode=1` and X-level `RegistryDwords` were both ignored. ForceFullCompositionPipeline was catastrophic: spontaneous resets, a KWin crash loop and a wedged display engine that needed a forced reboot.

**N2. Silent corruption after S3 resume**

- **Symptom:** in about 1 of 4 resumes, KWin came back already corrupted, with no reset line, no Xid and no kernel message.
- **Root cause:** the resume path re-initialised the GPU with a corrupted GL state. Logs could not tell it apart from a clean resume.
- **Workaround:** the watchdog restarted KWin on every `PM: suspend exit`, twice (at +3 s and about +60 s), because a KWin started 3 s after resume could inherit the corruption. Five rapid resumes in a row crashed `kscreenlocker_greet` (QRhi context lost, SIGABRT) mid-PAM, and pam_faillock then locked the account.

**N3. Panel detected as a TV**

- **Symptom:** a dark image with ghosted thin text, plus corrupted video whenever a player opened.
- **Root cause:** the 470 driver could not read the eDP EDID (`card0-Unknown-1`, 0 bytes). It fell back to a generic TV profile (`TV-3` on DP-2) with limited range and TV signal filtering.
- **Workaround:** `ConnectedMonitor "DFP-3"` helped until a video player triggered a re-probe. The fix that held was `CustomEDID` from a 128-byte EDID pulled from a public hardware probe of the same model. `nvidia-edp-guard.service` rewrote `10-nvidia-edp.conf` on every boot, and a kernel-cmdline marker (`nv_customedid`) switched it on.

### Session lock-in

**N4. No Wayland session possible**

- **Root cause:** the 470 driver has no GBM support, which arrived in 495, and KWin has no EGLStreams backend. Plasma Wayland on 470 meant no hardware acceleration or no session at all.
- **Effect:** the box was held on Plasma X11 (`plasma-workspace-x11`, SDDM forced to `DisplayServer=x11`), along with every X11-only failure mode above.

**N5. Plasma 6.8 removed the X11 session**

- **Symptom:** on 2026-11-05, `dnf upgrade` took Plasma 6.7.5 to 6.8.2. `kwin` and `plasma-workspace` obsoleted their X11 packages, so `kwin_x11`, `startplasma-x11` and `plasma-kwin_x11.service` disappeared. The running session was left on deleted binaries; `fixdisplay` reported a false "done" because the old process still showed up.
- **Workaround:** a manual root-subvolume swap to snapper snapshot #55 (taken 53 s before the upgrade), a 97-package `excludepkgs` pin in `dnf.conf`, and a pinned kernel (7.2.13).

**N6. Fedora 45 upgrade blocked**

- **Root cause:** the Plasma pin conflicted with F45's ffmpeg 9 (kpipewire needs `libavutil.so.60`). RPM Fusion F45 still carried only an fc44 build of `xorg-x11-drv-nvidia-470xx`. Staying on 470 meant XFCE or Cinnamon on X11, or not upgrading.

### Boot and packaging

**N7. Out-of-tree module on every kernel**

- **Symptom 1:** a ~30-minute dnf deadlock. The akmods `%posttrans` ran a nested `dnf install` that waited on the rpm lock the outer `dnf-automatic` held.
- **Symptom 2:** a boot hang. `akmods.service` tried to build missing kmods at boot and hung every entry without one.
- **Workarounds:** the synchronous kernel-install plugin was deleted, `akmods.service` was masked, and builds were moved to the async `akmods@<kver>` units. Manual `akmods --force --kernels <ver>` covered each installed kernel.

**N8. Boot entries needed constant repair**

- **Root cause:** `grub2-mkconfig` rewrote every BLS `options` line from `/etc/kernel/cmdline`, which stripped per-entry NVIDIA arguments. The gmux handoff also left nouveau entries showing garbage, so for a while only one NVIDIA entry was known to boot. An earlier 470 attempt (June 2026) had bricked the box outright.
- **Workarounds:** blacklists and `nvidia-drm.modeset=1` moved into `/etc/kernel/cmdline`. `nvidia-edp-reconcile.service` ran at every boot: it checked kmods, healed BLS args, pruned entries, set `saved_entry` and regenerated the GRUB 2.14 config. nouveau was hard-masked (`install nouveau /bin/false`), and the broken rescue entry was purged.

### Thermal, video and suspend

**N9. Clocks stuck at 405 MHz**

- **Symptom:** the GPU ran 87–99 % busy at perf level 0 (405 of 925 MHz). Firefox's WebRender thread used ~95 % of a core busy-waiting in `eglSwapBuffers`. During YouTube TV the package sat at 96–99 °C.
- **Root cause:** the first boot at full clocks overheated within ~5 minutes (artifacts, then a freeze). A module-level `NVreg_RegistryDwords` low-power clamp was added to survive that. After that, the driver would not let any runtime or X-level PowerMizer override raise the clock, and Coolbits clock editing was disabled (`nvclockeditable=0`). A kernel-level override was left untried because of heat near the (then swollen) battery.
- **Workaround attempts:** `__GL_YIELD=USLEEP` (no measurable gain), and pausing the animated wallpaper while a window is maximised.

**N10. No usable hardware video decode**

- **Root cause:** Kepler NVDEC under 470 handles only MPEG-2, VC-1 and H.264, and only through `libva-nvidia-driver` with `NVD_BACKEND=direct` (the EGL backend failed with "Exporter failed").
- **Workaround:** AV1 and WebM were disabled in Firefox so YouTube would serve H.264. The VA-API environment (`LIBVA_DRIVER_NAME=nvidia`, `MOZ_DISABLE_RDD_SANDBOX=1`) was never made permanent.

**N11. Unsupported suspend path**

- **Symptom:** the box ran S3 for weeks without `nvidia-suspend`/`nvidia-resume` and with `PreserveVideoMemoryAllocations=0`, so the GPU was re-initialised blind on every resume.
- **Workaround:** `xorg-x11-drv-nvidia-470xx-power` was installed on 2026-08-30. Its `chvt` step could abort suspend outright if it failed. The resume hangs of 08-30 and 09-28 were never proven to be NVIDIA's fault (see Remaining caveats).

**N12. Ageing userspace API**

- **Symptom:** 470 reports only Vulkan 1.2.175. Zed happened to work because the extensions it needs were present. Any app that requires Vulkan 1.3 would silently fall back to llvmpipe.

## Workaround burden

Keeping 470 usable took at least ten custom services, scripts and config overrides. None of them exist today.

| Component | Purpose | Issue | Status now |
| --- | --- | --- | --- |
| `kwin-reset-heal.service` + `kwin-reset-watchdog` | Restart KWin on GL reset and on every resume | N1, N2 | Removed 2026-11-11 (`not-found`) |
| `fixdisplay` + Meta+Shift+R shortcut | Manual compositor restart | N1 | Removed |
| `TimeoutStopSec=5s` drop-in on `plasma-kwin_x11` | Stop a wedged KWin from stalling the heal | N1 | Gone with kwin_x11 |
| `nvidia-edp-guard.service` + `CustomEDID` blob | Feed the real panel EDID to X | N3 | Gone (`/etc/X11/nvidia` purge) |
| `nvidia-edp-reconcile.service` | Heal kmods, BLS args and `saved_entry` every boot | N7, N8 | Replaced by a generic `boot-reconcile` |
| `blacklist-nouveau.conf` + cmdline blacklists | Keep nouveau off the GPU | N8 | Removed; only `nova_core` is blacklisted |
| `nvidia-lowpower.conf` (`NVreg_RegistryDwords`) | Thermal clamp | N9 | Removed |
| `__GL_YIELD=USLEEP` env files | Reduce Firefox spin in the driver | N9 | Removed |
| `excludepkgs` pin (97 packages) + root-subvolume rollback | Keep Plasma X11 alive | N5, N6 | Pin removed at the F45 upgrade |
| Masked `akmods.service`, per-kernel kmod builds | Avoid boot hangs and deadlocks | N7 | Not needed; nouveau is in-tree |
| `xorg-x11-drv-nvidia-470xx-power` | Vendor suspend hooks | N11 | Removed |

As of 2026-11-21 the only NVIDIA-named package left is `nvidia-texture-tools`. It is a texture-compression library that 0 A.D. depends on, not a driver.

## Migration

The move took one day, 2026-11-11, after a fair nouveau test showed that the earlier "nouveau is laggy" verdict came from a misconfiguration.

1. **2026-11-06: nouveau test boots on Fedora 44 (X11).** nouveau worked on kernel 7.2.13 with Mesa 26.2.5. The Intel-iGPU panel path failed at X modeset, so that route was closed.
2. **2026-11-11: why earlier nouveau runs felt slow.** Every previous nouveau boot ran without the nouveau governor. It was either masked on the test cmdline or unlinked by the switch-over script, so the GPU sat at boot clocks (405 MHz core, 680 MHz memory). Firefox was also forced to software WebRender, and one test had `drm.debug` flooding the journal. In short, nouveau had never been tested fairly.
3. **2026-11-11: the fair test, nouveau + Plasma 6.7.5 Wayland.** This time the governor was running (pstate `0f`, stepping to `0a` at 87 °C), and Firefox used GPU WebRender. KWin Wayland ran on OpenGL/EGL on NVE7 (Mesa 26.2.6) with legacy KMS. There were zero nouveau errors or TRAPs. The user's verdict: "video effects are perfect."
4. **2026-11-11 19:33: step 1, nouveau as the default on Fedora 44.** The cmdline blacklists were swapped and `blacklist-nouveau.conf` removed. A udev rule (`90-nouveau-governor.rules`) now starts the governor, which fixes a boot-order race. SDDM's default session was set to Plasma (Wayland). An NVIDIA 470 fallback GRUB entry was kept. `check.sh` reported ALL GOOD.
5. **2026-11-11 ~20:20: step 2, the offline upgrade to Fedora 45.** The Plasma pin came out of `dnf.conf`, and `--exclude='grub2*,shim*'` protected the GRUB 2.14 / ESP chain. The result was kernel 7.3.3 and Plasma 6.8.2 Wayland on nouveau. A snapper snapshot and a `/boot` tarball were taken first.
6. **After 2026-11-11: NVIDIA purge.** The 470 packages, configs, fallback entry and reconcile service were removed. A generic `boot-reconcile.service` took over stale-entry pruning and `saved_entry` upkeep.

State on 2026-11-21: Fedora 45, kernel 7.3.4, Plasma 6.8.3, `XDG_SESSION_TYPE=wayland`, nouveau loaded, no `nvidia*` modules, and 0 nouveau error, trap or fault lines in this boot's kernel log.

## Resolution matrix

All twelve 470-era issues are resolved. Nine are gone because their cause is gone. The other three are resolved by the open stack working as designed.

| # | Issue | Why it is gone on nouveau + Wayland + Plasma 6 |
| --- | --- | --- |
| N1 | KWin corruption after GL reset | kwin_x11 and the 470 robustness-reset path no longer exist. KWin Wayland renders through Mesa EGL on nouveau, and no ghost-text resets have been seen since 2026-11-11. |
| N2 | Silent corruption on resume | nouveau is in-tree, and the kernel's own PM callbacks restore the GPU. The two-stage heal, and the lock-screen crash it could trigger, are gone. |
| N3 | Panel detected as a TV | nouveau and DRM read the eDP EDID natively (nouveau could always read it). There is no Xorg, no `CustomEDID` and no guard service. |
| N4 | No Wayland | Mesa provides GBM, so KWin Wayland runs with full OpenGL/EGL acceleration. |
| N5 | Plasma 6.8 dropped X11 | The box is on the session Plasma now ships as its only one: Plasma 6.8.3 Wayland. |
| N6 | F45 upgrade blocked | Upgraded on 2026-11-11 with nothing pinned except grub2/shim, which is a GRUB 2.14 matter, not a GPU one. |
| N7 | akmod per kernel, deadlock, boot hang | nouveau ships in every kernel. Kernel 7.3.4 arrived with nothing to build. |
| N8 | Boot entries needed repair | The cmdline only blacklists `nova_core`, and there are no per-entry driver args to lose. A generic `boot-reconcile` handles the GRUB 2.14 upkeep. |
| N9 | Clocks stuck at 405 MHz | The nouveau governor steps through pstates `07`/`0a`/`0e`/`0f` by temperature and AC state. It runs at `0f` (5016 MHz memory) until 87 °C, then drops a rung. |
| N10 | No usable HW decode | Not fixed in hardware, since nouveau has no Kepler video-decode firmware. But GPU WebRender plus CPU decode on the i7 gives smooth video; decode costs about 0.3–0.45 of a core at 1080p. |
| N11 | Unsupported suspend path | There are no vendor sleep hooks or `chvt` step left to fail. Suspend uses the kernel's standard path. |
| N12 | Ageing userspace API | Mesa 26.2 is current, and nothing depends on a frozen 2021 driver ABI. |

### Remaining caveats

These came up in the same period. They should not be credited to nouveau or charged to NVIDIA.

- **Temperatures under load are higher than with the 470 clamp.** In the fair test the GPU ran at 75–85 °C and the CPU at 80–94 °C. That is because nouveau actually clocks up. The user accepted this trade on 2026-11-06.
- **Kepler on nouveau exposes no GPU-load counter.** The governor uses temperature as a stand-in for load. Re-clocking is still marked experimental upstream, though no hangs have been seen.
- **S3 resume hangs (2026-08-30, 2026-09-28) are unexplained.** Kernel `pm_test` passed at all five levels. The leading suspect is the `reserve=` memory-hole workaround interacting with firmware S3, not the GPU driver.
- **The quiet internal speakers** were a kernel 7.3 regression in the CS4208 pin mapping, fixed with an HDA patch firmware.
- **Broken fan control** came from kernel 7.3 moving applesmc to the hwmon API, fixed with a patched mbpfan. `mbpfan.service` was stopped when this report was written only because the Power Bolt "fans max" mode had taken manual control, which is intended behaviour.
- **Idle plasmashell CPU (35–88 %)** came from the animated Matrix-rain wallpaper's CPU-rastered QML Canvas, under either driver.
- **The fullscreen-video green screen and stalls of July 2026** happened on nouveau under X11, with the compositor unredirecting and no governor running. They have not been seen since the move to Wayland.

## Conclusions

On a 2013/2014-era Kepler Mac, the EOL proprietary driver was the larger source of instability. The open stack is now the better choice.

- **Root cause pattern.** Almost every 470 problem came from the driver being out-of-tree and frozen: closed reset handling in KWin, a missing GBM, unreadable EDID, ignored PowerMizer keys and a per-kernel kmod. Patching around each one produced a stack of watchdogs, guards and pins that broke again with each upstream change.
- **The real lesson of the migration.** nouveau had been judged on runs without its governor and with software rendering forced on. Tested fairly, it matched or beat 470 on every point that mattered here.
- **Recommendations.**
  - Stay on nouveau with Plasma Wayland. Do not reinstall any proprietary NVIDIA driver on this machine or on the second, identical MBP.
  - Keep `90-nouveau-governor.rules` and the governor in any rebuild or migration script. Without them nouveau falls back to boot clocks and looks broken.
  - Keep Firefox on GPU WebRender. The software-WebRender override was a workaround from the X11 era.
  - Unmasking `akmods.service` is optional cleanup. It has nothing to build now.

## Sources

All facts come from the machine's own diagnostic notes (2026-06-26 to 2026-11-18) and from live checks run on 2026-11-21.
