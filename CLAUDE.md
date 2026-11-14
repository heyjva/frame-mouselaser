# CLAUDE.md

Context for AI coding sessions on this repo. Read it before changing anything.

## What this is
`mouselaser` is an experimental, **AI-written ("vibecoded")** SteamVR driver for the Steam Frame. It adds a virtual controller whose pose is aimed by a physical mouse, so the mouse can drive SteamVR's laser pointer across all overlays. Start with:
- [README.md](README.md): usage, settings, recovery.
- [docs/development-log.md](docs/development-log.md): where this came from, what was verified, what's next.
- [docs/steamvr-primer.md](docs/steamvr-primer.md): SteamVR and Steam Frame concepts for newcomers.
- [docs/how-it-works.md](docs/how-it-works.md): internals.
- [docs/steam-frame-background.md](docs/steam-frame-background.md): the Frame's display and input stack, and why this approach was chosen.

## Environment facts that bite
- The session usually runs **on the headset itself**, in a terminal inside the nested KDE desktop. That desktop sits inside gamescope, which is `PartOf=steamvr.service`. **Restarting SteamVR (or gamescope) kills the desktop and this session.** Ask the owner to reboot instead, and continue in a new session.
- Inside the nested desktop `XDG_RUNTIME_DIR=/run/user/1000/nested_plasma`. For `systemctl --user`, prefix the command with `XDG_RUNTIME_DIR=/run/user/1000`.
- `/usr` belongs to the read-only OS image. Don't modify it. Everything here is user-level: `vrpathreg` and `~/.config/openvr`.
- The driver `.so` and bindings are only loaded when SteamVR starts. Code changes need a rebuild **and** a reboot to test. (`systemctl --user restart steamvr` also works and re-loads the `.so` in place, but kills the nested desktop session with it.)
- Check the logs with `grep -a 'mouselaser:' ~/.local/share/Steam/logs/vrserver.txt | tail`.
- `gamescope-session.service` is `RefuseManualStart` and `PartOf=steamvr.service`: restart the `steamvr` user unit to restart the gamescope session. If the restarted gamescope logs `openvr: Unable to init VR runtime`, it is stuck on the headless backend (no cursor in the VR overlay) — restart `steamvr` again once the VR server is up.
- `/` is a btrfs subvolume with the **read-only property** (`btrfs property get /` → `ro=true`) even though the mount says rw. Write to `/usr` etc. only after `sudo btrfs property set / ro false`; it reverts on system update.
- **A mouse can expose buttons and movement on separate evdev nodes** (the Razer ProClickM does: buttons on one node, REL on another). `MouseReader` opens *all* matching devices for this reason. When hunting input bugs, check `sudo timeout 5 cat /dev/input/eventN | xxd` (is the hardware emitting?) and `/sys/class/input/inputN/capabilities/key` (hex word bit N = key N; 7 = BTN_EXTRA, 8 = BTN_SIDE) before suspecting the driver.
- Device selection: `readdir("/dev/input")` order is arbitrary (hash order), never numeric. If two mice match, `deviceNameFilter` in `~/.config/openvr/config/steamvr.vrsettings` under `driver_mouselaser` picks the right one.

## Ground rules from the owner
- **Never commit, stage or push.** Git is the owner's job.
- **Warn before doing anything that touches the system**: registering or unregistering the driver, editing `steamvr.vrsettings`, rebooting, or restarting services. Wait for approval. Editing and building inside this repo is fine.
- Keep the "vibecoded / experimental" warning in the README accurate. Don't oversell what has been tested.
- Record new findings and behaviour changes in `docs/`. Add an entry to `docs/development-log.md` for each meaningful change.

## Build
```sh
cmake -S . -B build -G Ninja && cmake --build build
```
Output: `driver/mouselaser/bin/linuxarm64/driver_mouselaser.so` (gitignored). The only warnings expected come from SteamVR's own `openvr_driver.h` (unused parameters). `build/compile_commands.json` is exported, so clangd can resolve the header.

Wheel logic can be checked without a headset: `cmake -S . -B build -DMOUSELASER_TESTS=ON && cmake --build build && ./build/wheel_sim`. Bump `k_version` in `src/driver.cpp` on every behaviour change, so the log (`mouselaser: version ...`) shows which build is running.
