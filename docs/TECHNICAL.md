# Sticky Grenade Handles: technical notes

Steam build 25480438 (EXE 1.8.46015.0). Offsets below are for that build; the mod checks both module hashes and
does nothing on any other build.

## Why a handle-first hit bounces

The thermite (`sticky_grenade`, resource `0xc5c05fcb5747c799`) is one dynamic body with a compound shape of two
capsules:

| Child | Mesh node | Radius | Material |
| --- | --- | --- | --- |
| 0, the canister | `c_collision` | 0.0489 m | `throwable` `0xc4692706` (friction 3.5, restitution 0.05) |
| 1, the handle | `c_collision_shaft` | 0.0214 m | `sticky_grenade_handle` `0x9a056e1c` (friction 1, restitution 0) |

Along the grenade's axis the handle spans -0.148 to +0.138 m and the canister +0.124 to +0.267 m; the origin is
the middle of the handle. Of all 4,821 physics resources, only the thermite and `sticky_stun_grenade`
(`0x3fa94f58f596bc0b`) use `sticky_grenade_handle`.

The engine's contact callback (helldivers2.exe `0x78BFC0`) posts one event per side of a contact. Its payload
carries, at `+0x14`, the `userData` (`+0x48`) of the contacted shape's material in the material library
`*(exe+0x27C5CF0)` (an `hknpMaterialLibrary`: entries of 80 bytes at `+0x48`, count and capacity at `+0x50` and
`+0x54`).

The throwable handler (game.dll `0x6C7C50`) acts on a locally owned throwable that is not yet attached. Unless
the throwable is an impact trigger, it calls the sticking code only when payload `+0x14` equals the
ThrowableComponent's `+0x164`, which is `throwable` for every sticky grenade (the shark mine has its own). A
handle contact carries `sticky_grenade_handle`, fails that comparison, and the physics bounce plays out.

How a stuck grenade sits (from the sticking code, game.dll `0x816F30`): on a unit it attaches upright in world
space at the contact point whichever part touched first; on the world it keeps its rotation and moves its origin
to the contact point, so a canister hit sinks the canister into the surface and a handle-first hit is expected to
leave the canister out.

## The change

The low dword of `userData` of the library entry named `sticky_grenade_handle` becomes `0xc4692706`
(`throwable`). Handle contacts then pass the comparison and stick like canister contacts. Friction, restitution
and every other material field stay as they are.

Alternatives that were not used: rebinding the handle in `sticky_grenade.physics` (also changes friction and
restitution, and overrides the resource), the ThrowableComponent's `+0x164` (one value per throwable, so it cannot
accept both materials; read-only pages), retagging the loaded shape (also changes friction and restitution, and
has to be found on every load of the grenade's package).

## Order of work (once per session)

1. Loader API 1 with a log; a second deployed copy does nothing (`StickyGrenadeHandlesInstalled`).
2. Build gate: SHA-256 of `helldivers2.exe` and `game.dll` (Bingus Shared Runtime's `verify_build`).
3. Material library: `0 < count <= capacity <= 1024`, one bulk read of the entries.
4. Exactly one entry tagged `0x9a056e1c` named `sticky_grenade_handle` and one tagged `0xc4692706` named
   `throwable` (names through `hkStringPtr`, bit 0 masked; entries are found by name, never by index, since the
   order differs between the file and the running game). A handle entry that already carries the throwable tag
   counts as applied.
5. Data check, because a data-only game patch can change these without changing the binaries: the
   ThrowableComponent records of the thermite and the sticky stun grenade (table `*(*(game+0x346BF98)+0xF12C90)`,
   92 slots, 360-byte records at `+1472`) have `sticky` (`+0x54`) = 1 and `+0x164` = `0xc4692706`.
6. One checked write (a memory-protection check, then `WriteProcessMemory`) of the 4 bytes, then a read back.
7. At shutdown, one read: the log says whether the value still held.

The material library and both materials exist when addons start, but the entity settings load a few seconds
later (at the splash screen). So the normal path is a self-removing update hook: each frame it checks the settings
root pointer and its table pointer first, and only when both exist runs the full check and the write. It gives up
after 600 s of game time. When done, it puts the previous update back if it is still the outermost one; if another
mod wrapped update later, it stays as a plain pass-through with no memory calls.

## Cost per frame

| Path | Calls | In-game cost |
| --- | --- | --- |
| Waiting at the splash screen (a few seconds) | 2 memory reads per frame (1 while the settings root is unset) | a few microseconds per frame, startup only |
| The applying frame | 13 memory reads, 1 memory-protection check, 1 write | about 0.3 ms once |
| Every frame afterwards | none: the hook removed itself | 0 |
| Shutdown | 1 memory read | once |

Measured in live play with the frame probe of Bingus Shared Loader's development build: 0.001 ms per frame in
missions and on the ship (the probe's own wrapper around the removed hook).

## Live checks

- On the title screen and the ship: the write applied 1-6 s after the mod started (about 1-2 s in normal
  launches), the handle material read back as `0xc4692706` from outside, and the shutdown line said it still held.
- In play sessions with joined missions, the shutdown line said the change still held after every launch that
  ended normally.
- Not yet captured in a log: a handle-first hit itself.

## Tests

`python scripts/build.py` runs, in LuaJIT and in the game's own `lua51.dll`: the mod against synthetic engine
structures in FFI memory with exact call budgets per scenario (`tests/frame_budget.lua`), FFI name clashes,
hostile update neighbours (below, above, re-hooking) and no C type growth per waiting frame; then the assembled
entry compiles as one chunk, refuses outside the game and runs only once.
