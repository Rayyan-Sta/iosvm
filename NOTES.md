# NOTES: what is verified, what is assumed, what will bite

## Verification status
* **Compiled/tested on a Linux host with stubs:** `cpu_features.cpp`, `guest_ram_map.cpp`, `qmp_client.cpp`
  (syntax with `-Wall -Wextra`; QMP client exercised against a fake QMP server: handshake, id-matched
  replies, event skipping, abs/key/mtt encodings, input sanitising).
* **Tested:** `asoft_runner.py` handlers (zip-from-block-device incl. trailing padding, BuildManifest
  resolution, USTAR export) against a fake IPSW. Shell scripts pass `bash -n`.
* **NOT compiled:** `gl_renderer.cpp`, `jni_bridge.cpp` (need the NDK headers) and **all Kotlin/Gradle**.
  Expect a few compile fixes; version pins (AGP 8.11.1, Kotlin 2.2.0, Compose BOM, NDK 29.0.13846066,
  Linux 6.12.30, Alpine 3.22.0) are starting points to verify.
* **Never run against a real fork build.** Every fork-facing detail below is from the fork's public docs or
  is marked as an assumption in code.

## Facts about the fork that change the plan
* The public Apple-SoC fork (ChefKissInc/QEMUAppleSilicon, from qemu-t8030) targets the **iPhone 11
  (t8030)**. The documented bring-up I found is for **iOS 14.0 beta 5**. iOS 16.7 is not documented in
  public sources I could find; treat iOS 16 as fork work, not configuration.
* **Kernel patching is done by the fork at load time** (its KPF pass patches AMFI etc. while loading
  the kernelcache), so the helper VM does not patch the kernelcache offline. It extracts/decrypts
  components, forges the restore tickets, and drives the restore.
* Restore needs a **companion Linux VM + idevicerestore over USB-over-TCP** (`usb-tcp-remote`), started
  *before* the iPhone machine. That is the "helper VM". The fork connects to it at `127.0.0.1:<usbPort>`.
* The restore writes the NVMe namespaces (root, firmware, syscfg, ctrl_bits, nvram, effaceable, panic_log,
  sep_nvram) + a pflash `sep_ssc`. nsid/nstype numbering in `T8030Board` follows the fork docs: re-check at your pin.

## Where the original spec was bent (and why)
| spec | what the code does |
|---|---|
| Native EL0 fast path | Detector + `ExecPlan` are real; the engine is a gated stub. A stock Android host has no EL2/KVM, and XNU `svc`/Mach traps must reach a guest EL1, so native EL0 execution would need a separate Mach/BSD syscall-translation layer. Everything runs on TCG (`-cpu max`). |
| `qemu-img` snapshot "quick boot ~10 s" | Live snapshots need VMState on every Apple device in the fork (unlikely today). Default = freeze + power-cut + qcow2 disk checkpoint, i.e. a **cold boot** from saved disk state. Flip `VmController.liveSnapshotsSupported` once the fork supports it. |
| `Image16k.gz` | Built and selected by host page size, but under TCG the guest page size is independent of the host's. It matters only if a hardware-assisted path ever appears. |
| Guest RAM file | Uses `ASharedMemory` (memfd) handed to QEMU via `/proc/<pid>/fd/<fd>`: no flash write-back. If your device's SELinux policy blocks that open, QEMU logs it; fall back to `IOSVM_RAM_FILE=<file>` in app storage (works, but dirty pages hit flash). |
| Back gesture / Home | iPhone 11 has no Home button: Home = swipe up from the bottom, Back = left-edge swipe (synthetic touches). "Slide to power off" coordinates are heuristics (`GuestInput.PowerOffUi`). |

## Risks to plan for
* **TCG speed.** Booting a modern XNU under software emulation on a phone CPU will be very slow, and the
  restore (6+ GB through emulated USB) slower still. I have no measurements; don't promise numbers.
* **Android phantom-process killer (12+)** can kill exec'd children. Mitigations: foreground service (done),
  at most two children at a time (iOS + helper; done). If processes still die on a device, disable the monitor
  via adb for that device (`settings put global settings_enable_monitor_phantom_procs false` on 12L+).
* **APFS write access** from the helper: stock kernel is read-only for APFS. The optional fs-patch stage
  needs an out-of-tree writable driver or an offline macOS patch.
* **Touch/button wire formats are assumptions:** `mtt` event shape (`qmp_client.cpp::mttEv`), QKeyCode ->
  GPIO mapping (`apple_hwbuttons.c`), framebuffer location (`ios_vm_bridge_set_fb` hook).
* **Firmware material:** no IPSW, keys, SEP ROM, or ticket-forging recipe is bundled. You supply them
  (`keys.json`, SEP ROM, `helper/user-scripts/forge-tickets.sh`), and you are responsible for having the
  right to use them.
