# iosvm: iOS-on-QEMU Android app (arm64-v8a)

Kotlin/Compose shell + native C++ (EGL/GLES3 zero-copy renderer, QMP client) around an Apple-SoC
QEMU fork shipped as `libqemu.so`. Status of each phase is in `NOTES.md`: read it first.

```
app/src/main/cpp/         CPU feature detector, guest-RAM mapper, GLES3 renderer, QMP client, JNI
app/src/main/java/.../    NativeBridge, GuestSurfaceView
   vm/                    T8030Board (QEMU argv), DiskLayout (nvme ns + qcow2 overlays), QemuLauncher,
                          VmController (lifecycle/save/shutdown), QemuService (foreground service)
   ui/                    VmScreen, EdgeHandleMenu (draggable overlay), SetupScreen
   input/                 GuestInput (home/back swipes, side/volume buttons, screen-off -> lock)
   setup/                 SetupPipeline (IPSW -> helper VM -> restore), HelperVmBoard, TarExtract
helper/                   companion-VM payload: manifest, runner, init, build script, user-script stubs
qemu-patches/             DRAM-sharing + `x-ios-vm-info` QMP command + hardware-button skeleton
tools/build-qemu-android.sh
```

## Build order
1. Fork: `qemu-patches/apply.sh` (+ manual hook-ups in `qemu-patches/README.md`), then
   `tools/build-qemu-android.sh` -> `app/src/main/jniLibs/arm64-v8a/libqemu.so`, `libqemuimg.so`.
2. Helper VM: put your `forge-tickets.sh` in `helper/user-scripts/`, run `helper/build-helper-vm.sh`
   -> `app/src/main/assets/helper/{Image.gz,Image16k.gz,initrd.gz,asoft-setup.json}`.
3. App: `gradle wrapper` once, then `./gradlew assembleDebug` (NDK 29, CMake 3.22+).
4. On device: first launch shows Setup (IPSW + your keys.json + SEP ROM). After setup: Boot / Resume.

## Runtime layout (app-private, `files/vm/`)
`fw/` boot artifacts, `disks/` NVMe namespace images + qcow2 overlays, `helper/` companion VM files,
`logs/` QEMU + serial logs, `qmp.sock`, `helper-console.sock`.
