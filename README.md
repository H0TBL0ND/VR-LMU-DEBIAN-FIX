# Le Mans Ultimate in VR on Linux (WiVRn + xrizer + Proton) — what actually blocked it, and the fixes

Français : [README.fr.md](README.fr.md)

Le Mans Ultimate (LMU, appid 2399420) can be started in VR on Linux with a standalone headset, **without SteamVR**,
using WiVRn (OpenXR runtime), xrizer (OpenVR → OpenXR) and a patched Proton-GE.
Out of the box it does not work: the game closes within seconds. We hit **four separate blockers**, one behind the other,
and each one hides the next. This document records the symptoms, how each was diagnosed, and the fix.

> **Status / honesty note.** Tested once, on one machine; the game starts and runs in VR with the headset (reported by the user wearing it).
> Not tested: long sessions, performance, online play. **The working setup runs the game *without Easy Anti-Cheat*** (see blocker 2),
> so online multiplayer is not expected to work with it.

## Test machine

| | |
|---|---|
| OS | Debian 13 (trixie), GNOME, Wayland session, Steam from the Debian package |
| GPU | Radeon RX 7700 XT (RADV) |
| Headset | Pico 4, streaming with WiVRn 26.9 (Flatpak `io.github.wivrn.wivrn`) |
| OpenVR layer | xrizer 0.6 (shipped inside the WiVRn Flatpak) |
| Proton | `GE-Proton10-34-LMU-hid_fixes-vr-bcrypt` (JacKeTUs) |
| Mesa | host 25.0.7 (Debian) → **26.1.6 (trixie-backports)**; WiVRn Flatpak runtime: 26.2.2 |

## Summary of the four blockers

| # | Symptom | Cause | Fix |
|---|---|---|---|
| 1 | With the game's own `openvr_api.dll` (bundled OpenComposite): popup "call openxr fail"; `opencomposite.log`: `XR_ERROR_FUNCTION_UNSUPPORTED … xrCreateXDevListMNDX` | OpenComposite asks for a Monado extension function (`xrCreateXDevListMNDX`) that the runtime path through Proton's `wineopenxr` apparently does not provide (we did not dig into which layer drops it) | Don't use OpenComposite: replace the DLL with Valve's `openvr_api.dll` and let xrizer handle OpenVR (known: [xrizer#301](https://github.com/Supreeeme/xrizer/issues/301)) |
| 2 | After replacing the DLL, the game crashes ~2 s after start, before any VR call: `c0000005` inside Wine's `ntdll` `load_dll` (null pointer) | Observed only while the EAC launcher (`start_protected_game.exe`) is used | Start `Le Mans Ultimate.exe` directly instead (EAC off, see below) |
| 3 | xrizer panics at `compositor.rs:1067` "Failed to create swapchain"; WiVRn prints `vk_create_image_from_native size mismatch, exported 10616832 but requires 13107200` / `XR_ERROR_RUNTIME_FAILURE in xrCreateSwapchain` | **Mesa version mismatch** between the WiVRn Flatpak runtime (26.2) and the host Mesa used by the game (25.0) — same as [WiVRn#957](https://github.com/WiVRn/WiVRn/issues/957) | Bring the host Mesa close to the Flatpak's (Debian: `trixie-backports`, 26.1.6) |
| 4 | Game dies at `VRUI.cpp: D3DX11CreateTextureFromMemory(... vrCursorFile ...) FATAL!!!` (visible in LMU's own `UserData/Log/trace*.txt`) | The game loads a Windows **cursor (`.cur`) file** as a texture; Wine's `d3dx` can't, and this Proton build carries a workaround ("Hacky workaround to CUR file loading issue — Fixes LMU VR initialization") **in its builtin `d3dx11_43`**. But the game's prefix contained Microsoft's **native** `d3dx11_43.dll` (probably installed by Steam's DirectX redistributable script — not verified), which overrides the builtin one and has no workaround | Put the Proton builtin `d3dx11_43.dll` back into the prefix |

Background: [Proton#7512](https://github.com/ValveSoftware/Proton/issues/7512) (LMU), which documents the `.cur` problem
(mdias, srounce) and the VR work behind the `-vr-bcrypt` build.

## Procedure that worked

Paths use the Debian Steam layout (`~/.steam/steam` → `~/.steam/debian-installation`); adapt to yours.
Close Steam before editing its config files. **A game update or "verify integrity of game files" will most likely undo steps 4–5 (not tested).**

1. **Mesa.** Make the host Mesa match the one inside the WiVRn Flatpak (`flatpak list --runtime | grep Mesa`, and `vulkaninfo --summary | grep driverInfo` for the host).
   On Debian 13 we enabled `trixie-backports` and upgraded the 64- and 32-bit Mesa packages together (nothing was removed; dry-run first with `apt-get -s`):
   ```
   echo 'deb http://deb.debian.org/debian trixie-backports main contrib non-free non-free-firmware' | sudo tee /etc/apt/sources.list.d/backports.list
   sudo apt update
   sudo apt -t trixie-backports install mesa-vulkan-drivers libgl1-mesa-dri libglx-mesa0 libegl-mesa0 mesa-libgallium \
        mesa-vulkan-drivers:i386 libgl1-mesa-dri:i386 libglx-mesa0:i386 libegl-mesa0:i386 mesa-libgallium:i386
   ```
   To revert: reinstall the previous version (`25.0.7-2+deb13u1`) of the same packages. Log out/in or reboot afterwards (we did not reboot during our test and it still worked).
2. **Proton.** Install `GE-Proton10-34-LMU-hid_fixes-vr-bcrypt` from [JacKeTUs/proton-ge-custom](https://github.com/JacKeTUs/proton-ge-custom/releases) in `~/.steam/steam/compatibilitytools.d/`
   and select it for LMU. We also had "Proton EasyAntiCheat Runtime" (appid 1826330) installed from Steam; it is needed for EAC and we did not test without it.
3. **OpenVR runtime = xrizer.** `~/.config/openvr/openvrpaths.vrpath` must list xrizer's directory from the WiVRn Flatpak, and `~/.config/openxr/1/active_runtime.json`
   must point to WiVRn's `openxr_wivrn.json`. In our working setup the vrpath contains the *hashed* deployment path
   (`{"runtime":["/var/lib/flatpak/app/io.github.wivrn.wivrn/x86_64/stable/<commit>/files/xrizer"],"version":1}`); that hash changes at each Flatpak update, so re-check this file after updating WiVRn
   (the stable alias `/var/lib/flatpak/app/io.github.wivrn.wivrn/current/active/files/xrizer` should avoid that, but we did not test it in the vrpath file).
4. **Valve's `openvr_api.dll`.** In the game folder, keep the original (`mv openvr_api.dll openvr_api.dll.opencomposite`) and put Valve's
   64-bit `openvr_api.dll` ([ValveSoftware/openvr](https://github.com/ValveSoftware/openvr/tree/master/bin/win64), we used 2.15.6) in its place.
5. **Skip EAC (this is what disables online play).** In the game folder: `mv start_protected_game.exe start_protected_game.exe.orig`, then
   `cp "Le Mans Ultimate.exe" start_protected_game.exe` (Steam still "launches" `start_protected_game.exe`, which is now the game itself).
   To undo: move the `.orig` back.
6. **Prefix `d3dx11_43`.** Replace
   `…/steamapps/compatdata/2399420/pfx/drive_c/windows/system32/d3dx11_43.dll` (Microsoft's, 276,832 bytes in our prefix) with the one from the Proton build:
   `…/compatibilitytools.d/GE-Proton10-34-LMU-hid_fixes-vr-bcrypt/files/lib/wine/x86_64-windows/d3dx11_43.dll` (307,949 bytes). Keep a backup of the original.
   Check with `PROTON_LOG=1`: the log must say `Loaded "C:\windows\system32\d3dx11_43.dll" … builtin` (it said `native` before).
7. **Steam launch options** (what we used; we did not test which of these are strictly required):
   ```
   PRESSURE_VESSEL_IMPORT_OPENXR_1_RUNTIMES=1 PRESSURE_VESSEL_FILESYSTEMS_RW=/var/lib/flatpak/app/io.github.wivrn.wivrn %command% +VR
   ```
   Add `PROTON_LOG=1 PROTON_LOG_DIR=$HOME/lmu-logs` to get logs (don't use `WINEDEBUG=+module`: multi-GB logs, and a crash loop can fill the disk).
8. Start the WiVRn server, connect the headset, then launch LMU from Steam with the normal "Play" button (not "Play with SteamVR/OpenXR").

## Exact versions and checksums of our working setup

| Item | Value |
|---|---|
| LMU Steam build | buildid `25661166` (manifest updated 2026-10-09) |
| Kernel | 6.12.107+deb13-amd64 |
| WiVRn Flatpak | 26.9, commit `61d58e18b177b19a34f4a0f3063c571bfc7585ef8a3145327a5beb8b5e268332` |
| Proton | `GE-Proton10-34-LMU-hid_fixes-vr-bcrypt` (`version` file: `1785273675`) |
| Valve `openvr_api.dll` (2.15.6, win64) | sha256 `bab8ac6ef64e68a9ca53315b0014d131088584b2efdfa6db511d67ec03cfcb4a` |
| Game's original (OpenComposite) `openvr_api.dll` | sha256 `21a8b36425d65691b26b1d6e10a0f19f3483f30065bac99ea2d041a491bacf5a` |
| `Le Mans Ultimate.exe` (= our `start_protected_game.exe`) | sha256 `66942337813b301b87d12edfa330f33354a2a4b8cd9b048885cbf77eb6f940cf` |
| Original `start_protected_game.exe` (EAC launcher) | sha256 `e86f518b447a90790f8458bd7be36bc42ae3cdaf723e6b9efcdcea3b544fd95c` |
| Prefix `d3dx11_43.dll`, before (Microsoft, native) | sha256 `981e42629df751217406e7150477cddc853b79abd6a8568a1566298ed8f7bd59` |
| Prefix `d3dx11_43.dll`, after (Proton builtin) | sha256 `373f2989779c9a52206bde854d4aaffd438f41e794a5aa7129804d9a7f389551` |

Also listed on the community [VR on Linux database entry for LMU](https://db.vronlinux.org/games/2399420.html).

## How to diagnose your own failure (what each log says)

- **Proton log** (`PROTON_LOG=1`): crash `Unhandled exception code c0000005` and `backtrace:` lines tell whether it is in Wine or in the game's `.exe`.
  Here the offsets pointed into `ntdll.dll` `load_dll` (disassemble the Proton `ntdll.dll` with `objdump -d` to see the function name).
- **Game log**: `<game folder>/UserData/Log/trace_<date>.txt` — the last lines before `FATAL` name the failing call (blocker 4).
- **OpenComposite log** (only when its DLL is used): `…/pfx/drive_c/users/steamuser/AppData/Local/OpenComposite/logs/opencomposite.log` (blocker 1).
- **xrizer log**: `~/.local/state/xrizer/xrizer.txt`. **WiVRn server log**: `~/.var/app/io.github.wivrn.wivrn/.local/state/wivrn/wivrn-dashboard/`.
  The line `vk_create_image_from_native size mismatch` is the signature of the Mesa mismatch (blocker 3).
- Quick bisecting that was useful: no `+VR` → game starts (so it is VR-specific); `+VR` only, no other launch options → same crash (so it is not your wrapper/env vars).

## Open questions

- **Is blocker 2 really "EAC"?** We saw it with two different replacement DLLs (Valve 2.15.6 and a recent OpenComposite build) and not with the original,
  and it disappeared when the EAC launcher was replaced. We did not identify the mechanism. If someone can keep EAC *and* use xrizer, please say how.
- Whether the original OpenComposite DLL would work with a `wineopenxr` that forwards `xrCreateXDevListMNDX` (blocker 1) — untested.
- Whether the Mesa gap that matters is "major version" or "exact version": 26.1.6 (host) vs 26.2.2 (Flatpak) worked for us.
- Which of `PRESSURE_VESSEL_*` variables are still needed with xrizer.

## Credits

The groundwork is other people's: JacKeTUs (LMU Proton builds), mdias and srounce (`.cur` analysis and workaround), xytovl and the WiVRn team
(Mesa mismatch explanation), ImSapphire (replace the DLL with Valve's, use xrizer), Supreeeme (xrizer). This write-up only documents how the pieces fit on one setup.
Diagnosed with Claude Code (Anthropic) together with the machine's owner.
