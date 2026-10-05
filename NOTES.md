# Ragdoll On Command: research notes

Goal (mod-ideas.txt item 1): ragdoll your own Helldiver with a key press, the key set in Mod Bindings Menu.

All addresses are game.dll RVAs for build 25480438 (Ghidra's session base is 0x7ffcea020000: Ghidra `FUN_7ffceaa95fa0` = RVA 0xa75fa0). Decompiles: `_research/ragdoll1.c` to `ragdoll7.c`.

## How the game ragdolls an avatar (offline, build 25480438)

Found from the string `avatar_ragdoll_active` (an animation variable the ragdoll code sets) and the avatar flag writes. The avatar manager is the global 0x3326d20 (the one Shallow Water Diving reads):

- **Avatar index:** the manager's hash map at +0xf8 (data), +0x100 (capacity), +0x104 (empty key), +0x108 (multiplier) maps an avatar entity id to its index. Rows are (key, value) u32 pairs, probed linearly from `key * mult` (low 32 bits), masked by `capacity - 1`. The entity's record pointer is at +0x110 + index * 8.
- **Per avatar,** at manager + 0x53d900 + index * 0x1238 (the avatar controller):
  - +0xf80, +0xf88 and +0xf90 are a 192-bit flag set. Every change goes through the dispatcher 0xa3f220 (controller, mask, on/off).
  - +0xf88 bit 36 (0x1000000000) is **ragdolled**; bit 44 is checked when the ragdoll starts and ends.
  - +0x448 (manager + 0x53dd48 + index * 0x1238) is the **ragdoll state**, with the avatar's entity id at +0x398.
- **0xa74700 `is_ragdolled(state)`:** looks up the state's entity and returns flag bit 36. It has sibling checks with identical code that test other bits; its signature runs through the `shr rax, 0x24`.
- **0xa74850 `can_ragdoll(state)`:** true when `flags & 0x184080980000040 == 0` and byte +0xf80 bit 2 is set. The meaning of those bits is unknown; it is probably what refuses diving, swimming, climbing and so on.
- **0xa75fa0 `start_ragdoll(state, mode)`:**
  - Sets flag bit 36 through the dispatcher and clears flag bit 1.
  - Sets the ragdoll mode (0xa77800: 2, or 4 when bit 44 is set; or the mode from the optional `mode` record).
  - Resets the state's timers (+0x34c, +0x350 = 0.1, +0x354 = 0.5) and stores the position.
  - Posts a network event (0xabfe430, id 0x112a8a3b).
- **0xa747d0 `try_ragdoll(state, mode)`:** returns 1 if `!is_ragdolled && can_ragdoll`, and calls start. The knockdown at 0x833540 (avatar manager, entity) inlines the same three calls with mode NULL, after 0x9415a0, a physics or audio call whose purpose is still unknown.
- **0xa76630 `end_ragdoll(state)`:**
  - Clears bit 36 and sets mode 0.
  - Posts event 0x85d82942.
  - Restores the stance. The game's per-frame ragdoll update (0xa74ce0, state, dt) decides when to call it, so the Helldiver gets up without the mod.
- **0xa76fc0 (physics impact):** it accumulates impact force, and when the impact is strong enough it spawns the "ragdoll collision" body and sets the animation variables. Its +0xff0 cooldown (1.5 s) is only read there. This is the explosion path, not a trigger the mod can use.

**Native calls are only safe for an avatar in the map.** If an entity is not in the avatar map, the lookup in these functions returns index 0xffffffff, and they read manager + 0xffffffff * 0x1238, which crashes. The mod calls them only after it has read the whole chain itself in the same frame and the state names the same entity.

## The local Helldiver

The game's "is this the local avatar" check at 0xa40070 shows the chain:

1. Players: the global 0x3326468. The local player is active when +0x88 is non-zero; its unit is at +0x3a8 (0x7fff = none).
2. Unit to entity (0xfd9ba0): the entity owner global 0x346bf98 has a unit map at +0xf22ec8 (data), +0xf22ed0 (capacity), +0xf22ed4 (empty key) and +0xf22ed8 (multiplier). The entity id is at owner + 0x1e65e4 * 8 + index * 24, which is the 24-byte entity record Shallow Water Diving reads at +0xf32f18, with the id at +8.
3. The entity id is checked against the "no entity" id (the global 0x3483c20), then looked up in the avatar map.

This matches Shallow Water Diving's live-proven chain for this build.

## Signatures (research/signatures.py)

| Signature | Code | What the mod reads |
|---|---|---|
| `local_avatar` | 0xa40076, through the function's tail (the shorter span also matches 0x18ba436) | players, local flag and unit, the unit lookup's call, no-entity id, avatar map |
| `unit_lookup` | 0xfd9ba0 | owner, unit map, entity id array |
| `knockdown` | 0x833680 | state offset and stride, and the calls to the three functions |
| `ragdoll_check` | 0xa74700 | state entity offset, flag word, the same avatar map |
| `ragdoll_flags` (optional) | 0xa748fa | the +0xf80 word, for refusal logs |

Fields shared between signatures must agree. The mod also checks that the knockdown's `is_ragdolled` call and the local-avatar check's unit-lookup call go to the functions found by their own bodies.

## 1-recon-1 (built 2026-10-03, untested)

When the key is pressed, the mod resolves the local state, then calls `is_ragdolled`, then `can_ragdoll`, then `start_ragdoll(state, 0)`, then `is_ragdolled` again.

`RagdollOnCommand.log` records:
- each press and why nothing happened (no Helldiver, already ragdolled, refused, with both flag words);
- a breadcrumb line before the start call;
- how many frames the ragdoll lasted, read from the flag in memory.

Open questions for the live test:
- Is calling these gameplay functions from the Lua `update` hook safe (thread and frame phase)? A crash right after the "starting" line would mean no.
- Does the Helldiver ragdoll and get up normally, and how long does it last?
- Is it allowed on the ship?
- In a squad as a client, do others see it? `start_ragdoll` posts a network event; whether a client's call reaches the host is unknown.
- Does Mod Bindings Menu's key fire while typing in chat?

## Live results and release (2026-10-04)

- 1-recon-1/1-recon-2: the key ragdolls the Helldiver and the game stands them up; calling the knockdown from the Lua update is fine. The binding name shows translated (bp: 2 of 2 texts).
- Released as V1 (version number only changed). Still unverified: whether squadmates see a client's ragdoll.
