# GTA V on MediaTek MT6993 + Mali-G1-Ultra (Xiaomi 17T Pro): step by step

This is the exact path from a fresh install to GTA V Story Mode that works on a Xiaomi 17T Pro. The usual Android PC-gaming setup is a Snapdragon phone with an Adreno GPU and the open Turnip driver. This phone has none of that:

| | Usual setup | This device |
|---|---|---|
| SoC | Qualcomm Snapdragon | MediaTek MT6993 (`ro.soc.model`), sold as Dimensity 9500 |
| GPU | Adreno | Mali-G1-Ultra MC12, Arm proprietary Vulkan driver 54.1.0 (r54p1) |
| GPU driver path | Turnip (Mesa) through adrenotools, replaceable | vendor driver only, loaded through a Vulkan wrapper; no driver swap |
| Texture formats | BCn in hardware | no BCn; the wrapper decodes BC1 to BC7 at upload |
| CPU | ARM64 | ARM64; GTA V is x86-64 and runs through FEX in arm64ec Proton |

Device facts above come from the phone itself (`getprop` on 01.10.2026: `ro.product.marketname` Xiaomi 17T Pro, Android 16, HyperOS OS3.0.311.0.WPSEUXM, `ro.soc.model` MT6993; the Vulkan device string `Mali-G1-Ultra MC12 54.1.0` and `SoC Manufacturer - Mediatek` from emulator logs). "Dimensity 9500" is MediaTek's product name for MT6993.

Tested with GTA V Legacy (Steam), Story Mode only. GTA Online is out of scope (BattlEye).

## 0. What you need

- This fork's APK (release `gta5-mali-g1-v1`), built from branch `gta5-mali-g1`.
- `wrapper-leegao-fixed3b.tzst`: the leegao Vulkan wrapper with three patches (staging buffers freed per command buffer, no dedicated-allocation preference, 16-bit BCn targets).
- `dxvk-sarek-1.14.0-gn.wcp`: DXVK-Sarek 1.14.0 repacked as a GameNative content package.
- A Steam account that owns GTA V and the Rockstar account **linked to that Steam copy**.
- About 135 GB free on the phone, `adb` on a PC, USB or wireless debugging.

## 1. Install the app

1. HyperOS blocks `adb install` of new packages silently (`INSTALL_FAILED_USER_RESTRICTED`). Copy the APK to the phone and install it from the Files app.
2. Grant notifications (download and Wi-Fi messages arrive as notifications) and all-files access when asked.
3. The fork's release build is debuggable on purpose, so `adb shell run-as app.gamenative` works. That is the only way to swap the graphics driver package in step 4. Anyone with adb access to the phone can read the app's data; uninstall the fork when you are done testing if that matters to you.
4. Log in to Steam in the app, install GTA V Legacy (about 129 GB over Steam). With "Download on Wi-Fi only" on (the default) nothing happens on mobile data except a notification.

## 2. Container settings

Open GTA V, gear icon, container settings:

| Setting | Value |
|---|---|
| Container variant | Bionic |
| Experimental Bionic Steam | **on** (starts the Rockstar stub `Launcher.exe` directly; `PlayGTAV.exe` hangs under the headless Steam path) |
| Wine | proton-11.0-2-arm64ec-1 (downloadable in the app) |
| Emulator | FEXCore 2605, preset **Intermediate**. Performance (TSO off) crashes GTA after about 90 s |
| Graphics driver | Wrapper-leegao |
| Graphics driver config | BCn emulation **none** (DXVK passes BC formats through and the wrapper decodes them) |
| DX wrapper | DXVK, version `1.14.0-sarek-0` (after step 3) |
| Screen size | 1440x720 |
| Scaling | Stretch |
| Environment variable | `DXVK_CONFIG=dxgi.maxDeviceMemory=1024;dxgi.maxSharedMemory=1024;dxgi.maxFrameRate=30` |
| ntsync | leave on (default). `PROTON_NO_NTSYNC=1` drops the benchmark from 49.9 to 18.7 FPS |

With this fork the DX wrapper's "max device memory" setting reaches DXVK as well. The env line still works and also carries the 30 FPS cap.

Budget: 1024 MB is the sweet spot. With 512 or 2048 the GPU memory grows to 5+ GB again and the game is killed for low memory.

## 3. DXVK-Sarek 1.14

GameNative, Settings, Components, "Install additional components", pick `dxvk-sarek-1.14.0-gn.wcp`. Then select version `1.14.0-sarek-0` in the container. Check: `adb logcat -s ContentsManager` prints `MATCH FOUND` for `dxvk-1.14.0-sarek-0` at the next start.

## 4. Patched wrapper

Start the game once with Wrapper-leegao so the app downloads its stock package, then stop the app and replace it:

```sh
adb push wrapper-leegao-fixed3b.tzst /data/local/tmp/w.tzst
adb shell am force-stop app.gamenative
adb shell "run-as app.gamenative sh -c 'cp files/assets/graphics_driver/wrapper-leegao.tzst files/assets/graphics_driver/wrapper-leegao.tzst.orig 2>/dev/null; cat /data/local/tmp/w.tzst > files/assets/graphics_driver/wrapper-leegao.tzst; md5sum files/assets/graphics_driver/wrapper-leegao.tzst'"
adb shell rm /data/local/tmp/w.tzst
```

Expected MD5: `78fa843cd4ec4cad634ffae9903bfbf7` (SHA-256 in `SHA256SUMS`). To go back, copy `.orig` over it.

## 5. GTA settings

`settings.xml` lives in the prefix under `files/imagefs_shared/home/xuser-STEAM_271590/.wine/drive_c/users/xuser/Documents/Rockstar Games/GTA V/`. Set it with the app stopped, or in the game menu:

- ScreenWidth 1440, ScreenHeight 720, Windowed 2 (borderless), VSync 0
- TextureQuality 0 or 1, the other quality settings at minimum
- **ShadowQuality not 0**: 0 freezes the game reliably

Smaller HUD: in Story Mode, pause menu, Display, Safezone to minimum. The profile only saves inside the game.

## 6. First start and Rockstar

1. Play. The first start after a fresh install sets up the prefix; if the Rockstar stub stops after `token:` and nothing happens, start once more (the Steam bootstrap library is unpacked on the first run).
2. Sign in to Rockstar when the form appears. "Activation Required" means the Rockstar account is not the one linked to this Steam copy; check Linked Accounts on the Social Club site.
3. In the menu choose Story Mode. Mouse mode in the overlay works like a touchpad; with a controller connected the overlay buttons are hidden.

## 7. Orientation

Controller grips with a camera cutout fit one way only. Settings, Emulation, Allowed Orientations: untick the side that does not fit. This fork labels them by the side of the front camera.

## Pitfalls

- **Container `LD_PRELOAD` after a reinstall.** If you ever put `LD_PRELOAD` into the container env, it replaces GameNative's own preloads, so you had to copy absolute paths like `/data/app/~~<hash>/app.gamenative-<hash>/lib/arm64/libevshim.so`. Every APK install changes the hash and wine dies with `CANNOT LINK EXECUTABLE ... libevshim.so not found`. Remove `LD_PRELOAD` from the container env, or update the paths from `dumpsys package app.gamenative` (`legacyNativeLibraryDir`) with the app stopped.
- **VPN.** A VPN on the phone broke the Steam login of the game process (`cache/sb_host.log`: `SteamHost: not logged on`) while the app itself stayed logged in, and GTA asked for activation again. Exclude GameNative in the VPN's split tunnelling (Proton VPN: mode "Exclude").
- **Container file.** The app rewrites `.container` from memory when a game ends. Edit it only with the app force-stopped, and keep `/` escaped as `\/`.
- **settings.xml resets** to 800x600 / VSync half after every crash.
- **Mods.** GameNative loads a `dinput8.dll` in the game folder automatically. Script Hook V then runs; move the files out for the plain game.
- **Battery.** At 7 % the phone ran about 5 FPS slower than at 70 %.

## What does not work, or not well

- Real driving runs at 38 to 40 FPS uncapped, CPU at 82 to 90 °C, big cores held at 1.8 to 2.7 GHz. Hence the 30 FPS cap: it holds 29.5 FPS with the fewest hitches.
- `wrapper-gamenative` (ASTC) starts with this fork's Present fix, but runs at about 20 FPS with block artefacts on car paint.
- Short stutters when streaming new areas at speed; the frame rate recovers on its own.
- Only GTA V Legacy (Steam) and only this phone are tested.
