# Sticky Grenade Handles v1.0

The G-123 Thermite sticks when its handle hits first, instead of bouncing off. The sticky stun grenade, which is
built the same way, gets the same fix.

**Status:** played in missions; the change stays in place through them (checked at every shutdown). A
handle-first hit has not been captured in a log yet. See [docs/TECHNICAL.md](docs/TECHNICAL.md).

## Why the handle bounces

The thermite's collision is two capsules: the canister and the handle. In the base game the handle has its own
physics material, `sticky_grenade_handle`, while the canister uses `throwable`. When a throwable touches
something, the game sticks it only if the touching part's material is the one its settings name, and for every
sticky grenade that is `throwable`. A handle-first hit therefore never sticks, and the physics bounce plays out.
Only the thermite and the sticky stun grenade use the handle material.

## What the mod changes

One value, once per game session: the handle material's tag (the `userData` the game copies into each contact)
becomes the `throwable` tag. A handle-first hit then sticks exactly like a canister hit. The handle's friction,
bounciness and everything else stay as they are, and nothing else in the game uses that material.

By the game's own sticking code, a grenade that sticks to an enemy attaches upright at the contact point either
way. On a wall or the ground it keeps the angle it hit with, so a handle-first stick is expected to leave the
canister standing out of the surface.

## Requirements and install

- Helldivers 2 Steam build **25480438** (EXE 1.8.46015.0). The mod checks the EXE and game.dll hashes and does
  nothing on any other build.
- [Bingus Shared Loader v18 or newer](https://github.com/CowboyBingus/BingusSharedLoader/releases/latest).

Import the release ZIP into Arsenal or HD2MM together with the loader, enable and deploy both, then restart the
game. The log is `%LOCALAPPDATA%\CowboyBingus\Helldivers2\Logs\StickyGrenadeHandles.log`.

Before writing, the mod checks that the material library is the game's `hknpMaterialLibrary`, that exactly one
material is named `sticky_grenade_handle` and one `throwable`, and that the thermite and the sticky stun grenade
still stick by the `throwable` material. It writes 4 bytes into committed, private, read-write memory and reads
them back. On anything unexpected it logs the reason and changes nothing. At shutdown it logs whether the value
still held.

## Multiplayer

The game decides a throwable's sticking on the machine that owns it, which is expected to be the thrower's.
Other players see your grenades stick, and theirs follow their own install.

## Cost

- No work per frame once the change is made.
- At startup: the game loads its entity settings a few seconds after the mod starts, so until then the mod reads
  two memory values per frame at the splash screen. When they are loaded: 13 memory reads, one memory-protection
  check and one 4-byte write; then the mod removes its update hook. One memory read at shutdown.
- Measured in live play: 0.001 ms per frame in missions and on the ship.

## Build and test

Clone [Bingus Shared Loader](https://github.com/CowboyBingus/BingusSharedLoader) beside this repository (or set
`BINGUS_SHARED_LOADER` to its path). The tests need LuaJIT (`HD2_LUAJIT`, or `luajit` on `PATH`) and the installed
game's `bin/lua51.dll` (set `HD2_LUA51_DLL` if the game is not in the default Steam folder).

```
python scripts/build.py
```

It runs the tests in LuaJIT and in the game's own `lua51.dll` (the game is never started), checks that the
vendored [Bingus Shared Runtime](https://github.com/CowboyBingus/BingusSharedRuntime) files are byte-identical to
the canonical ones when the workspace has them, and writes `releases/Sticky-Grenade-Handles-v1.0.zip`.

## License

Zero-Clause BSD (0BSD), see `LICENSE`.

**AI disclosure:** Claude Opus 5.5 assisted with research, implementation, tests and documentation.
