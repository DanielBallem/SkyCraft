# Porting SkyCraft to another host game

This explains what the Skyrim SKSE plugin and the Minecraft Fabric mod say to each other, and what
another game (the "host") has to do to drive the **unchanged** Fabric mod the way Skyrim does.

The source of truth is always `protocol/skycraft_protocol.h` (mirrored in
`fabric/src/main/java/dev/skycraft/link/Proto.java`). This document is a map of it, not a copy.

---

## 1. The model in one paragraph

Minecraft runs hidden, as a **simulation and rendering service**. It owns the player's body
(movement physics, health, inventory, items, blocks, combat math). The host game owns the
**world** (terrain, buildings, NPCs, AI, quests, saves), the **camera**, the **window and input**,
and **draws everything**. The host tells Minecraft what the world is shaped like, where the NPCs
are and what keys are pressed. Minecraft tells the host where the player is, what the player hit,
and gives it meshes, textures and a HUD image to draw. Neither game's rules are re-implemented
in the other; the two mods only translate.

## 2. Transport

| Thing | Value |
|---|---|
| Mechanism | One Windows named file mapping, `Local\SkyCraft_v1` (Java side overridable with `-Dskycraft.link=...`) |
| Size | ~263 MB (`kMappingBytes`): mostly the overlay pixel slots and the two big byte rings |
| Who creates it | **The host** (`CreateFileMappingW`), with a DACL that lets a non-elevated Minecraft open it (see `SharedWithThisUser` in `skse/src/Link.cpp`) |
| Who opens it | Minecraft, polling `OpenFileMappingW` once a second until it exists |
| Handshake | `magic = 0x43594B53` ("SKYC") and `version = 11` in the header. Minecraft refuses a mismatch |
| Liveness | Each side writes `GetTickCount64()` into its heartbeat field every frame. Minecraft treats the host as gone after 8 s; Skyrim treats Minecraft as gone after 3 s |
| Restart detection | Minecraft watches `header.skyrimPid`; a change bumps its `generation`, and it **resends everything it cached** (atlas, textures, every block section) |
| Byte order / layout | Little-endian, fixed-size `#pragma`-free structs with `static_assert`ed sizes. No serialization library |
| "Is Minecraft already running?" | Minecraft holds a named mutex `<mapping name>_minecraft`. The host checks it before launching a second one |

There are only three kinds of channel inside the mapping:

- **Seqlock slots**: latest-value-wins state. The writer makes `seq` odd, writes, makes it even.
  The reader retries until it reads the same even `seq` before and after. Used for per-frame state.
- **SPSC rings**: `u64 head` (producer) at `+0x00`, `u64 tail` (consumer) at `+0x40` (separate cache
  lines), data at `+0x80`. Either fixed-size entries (input, events) or 8-byte-aligned
  `{u32 type, u32 bytes}` messages with a `kPad` "wrap to start" message (collision, render).
- **Triple buffer**: the overlay image. One atomic `state` word holds the middle slot index plus a
  dirty bit; writer and reader each `xchg` it.

## 3. Coordinates

**Everything in the protocol is Minecraft space**: blocks, Y up, +Z south, yaw in degrees with
0 = south. The host converts on its side.

For Skyrim (`skse/src/Link.h`): `1 block = 70 Skyrim units`, `mc = (sky.x, sky.z, -sky.y) / 70`,
`mcYaw = heading * 180/π + 180`. Pick your scale so a 1.8-block-tall Minecraft player matches
your game's player height.

`worldId` (any 32-bit id of the current map/level/cell) and `collisionEpoch` (bump it whenever the
world changes) tell Minecraft when to drop everything it knows about the old world.

## 4. Every channel, by direction

### Host → Minecraft

| Channel | Kind | Rate | What it carries | Why Minecraft needs it |
|---|---|---|---|---|
| `Header` | fields | per frame | magic, version, host PID, heartbeat | link up / down / restarted |
| `SkyState` @0x100 | seqlock | **every host frame** | flags (`InGame`, `MenuOpen`, `Loading`), `worldId`, `collisionEpoch`, host player feet pos, **yaw/pitch**, `teleportSeq`, viewport size, game hour | `seq` advancing is **the frame clock Minecraft paces itself to**. Look direction is computed by the host (from mouse deltas using Minecraft's sensitivity curve) and is authoritative. Viewport sets Minecraft's render size. Game hour drives Minecraft's daylight |
| Input ring @0x1000 | 16-byte entries ×4096 | per input event | `kInKey` (**SDL scancodes**), `kInMouseButton`, `kInScroll` (notches×120), `kInCursor` (overlay pixels, for GUI screens), `kInText` (code points), `kInReleaseAll`, `kInOpenMenu`, **`kInHurt`** | Minecraft's window is hidden and never focused; this *is* its keyboard and mouse. `kInHurt` is how the host's NPCs damage the player |
| Collision ring @0x20000 | 32 MB byte ring | streamed | `kColClear{epoch}`; `kColTris` (exact triangles + flags: stair helper, diggable+material, ghost, terrain); `kColRegion` (8×8×8 sub-voxel occupancy per block over a box; replaces all collision in that box) | The world's shape. Feeds Minecraft's own collision, ray casts (arrows, crosshair), the smooth triangle collider, digging, and explosion craters |
| Actor table @0x12000 | seqlock, ≤256 × 64 B | every frame | id, flags (hostile, dead, essential, in combat), feet pos, yaw, width, height, health fraction, level, name | Minecraft spawns an invisible hittable `SkyrimActorEntity` per record, so vanilla swords, bows, enchantments, sweeps all hit NPCs |
| Water grid @0x400 | seqlock | when it changes | 16×16 column grid of water surface heights around the player | Minecraft treats air under that surface as water for entity physics (swim, float, drown) |

### Minecraft → Host

| Channel | Kind | Rate | What it carries | What the host does with it |
|---|---|---|---|---|
| `Header` | fields | per frame | Minecraft PID, heartbeat | link up / down |
| `McState` @0x200 | seqlock | every MC frame | flags (in world, screen open, on ground, sneaking, sprinting, dead, swimming, flying), interpolated feet + eye pos, eye height, sensitivity, **`teleportAck`**, GUI scale, FOV, view bob, **raw 20 Hz tick positions + QPC timestamp**, camera mode (F5) and distance | Puppets its own player to this position every frame, drives its camera from it (including bob/FOV/F5), and plays matching animations. The tick data lets the host interpolate on its own clock without judder |
| Event ring @0x17000 | 32-byte entries ×512 | events | `kEvHitActor` (actor id, damage after MC modifiers, knockback dir/strength, crit/projectile/sweep/fire, weapon class), `kEvPlayerDied`, `kEvExplosion` (centre, radius), `kEvArrowStuck`, `kEvSkillUse` | Applies damage through its own hit pipeline (reactions, aggro, crime), kills its player, shakes/damages around explosions, attaches arrow meshes, levels skills |
| Render ring | 64 MB byte ring | streamed | `kRenAtlas` (block+item atlas RGBA), `kRenAtlasRegion` (animated sprite frames), `kRenSection` (Minecraft-built block meshes per 16³ section, 32-byte vertices with UV, tint/AO colour, block+sky light, face normal, cutout/translucent flags), `kRenLights` (light-emitting blocks with colour and hazard kind), `kRenSolids` (block bitset NPCs should collide with), `kRenDug` (cells dug out of the host world), `kRenTexture` (skins, armour), `kRenAvatar` (posed player model in third person), `kRenScene` (all other entities, block entities, particles), `kRenRagdoll` (player body parts for a death ragdoll), `kRenClearAll` | Draws Minecraft's world **inside its own frame** with its own lighting, shadows and fog, depth-tested against its geometry. Adds collision for placed blocks so NPCs can't walk through builds. Hides its own geometry where the player dug |
| World entities @0x1C000 | seqlock, ≤160 × 96 B | every MC frame | arrows, dropped items, tridents, dropped blocks, break cracks, contact shadows, plus the targeted-block outline | Lightweight per-frame sprites/quads the host draws itself |
| Overlay @0x300 + pixels | triple buffer, 3 × 3840×2160 RGBA | every MC frame | Minecraft's own framebuffer with a transparent background: first-person hand and held item, hotbar, hearts, crosshair, every open screen (inventory, chat, pause) | Composited on top of the host's frame |

### Handshakes worth knowing

- **Frame pacing.** Minecraft renders at most one frame per host frame: it spins (≤25 ms) until
  `SkyState.seq >> 1` changes. If the host stops writing (menu, alt-tab), Minecraft stops waiting.
  So the host must bump `SkyState.seq` **exactly once per rendered frame**.
- **Teleport.** The host is authoritative for position only at discontinuities (load, fast
  travel, cutscene hand-back). It sets `pos` and increments `teleportSeq`. Minecraft moves the
  player there, then **holds it frozen until collision under it has arrived** (or 6 s), and only
  then reports `teleportAck == teleportSeq`. The host must not puppet its player from
  `McState` until the ack matches, and should re-teleport if Minecraft waits far from the host player.
- **Menus.** While `kSkyMenuOpen`, Minecraft drops held keys; the host keeps input for itself.
  While `kMcScreenOpen`, the host shows Minecraft's cursor and sends `kInCursor`.
- **Ownership of the player.** Normally Minecraft drives. When the host needs its own animation
  (furniture, mounts, scenes), it simply stops puppeting and later teleports Minecraft to where its
  player ended up (`SkyrimTakeover` in `skse/src/Game.cpp`).

## 5. What a new host must implement

Skyrim-specific names in the protocol (`formId`, `DigMaterial`, `kEvSkillUse`'s ActorValue
numbers) are just ids; any game can fill them.

### Tier 0: link (nothing works without this)
1. Create the mapping with the exact layout and size; zero the host-owned regions; write version,
   PID, then magic (release store).
2. Heartbeat every frame.
3. Write `SkyState` once per frame (seqlock, `seq += 2`), including viewport and flags.
4. Launch Minecraft if the `_minecraft` mutex is absent (bundled Prism: `--launch SkyCraft`, with
   `-Dskycraft.startHidden=true`). Minecraft opens the void "mirror" world on its own once linked.

### Tier 1: walk around (the first playable milestone)
5. **Collision export.** Stream the world around the player (Skyrim streams 8-block regions, 5 regions out horizontally, 3 below, 2 above, and re-sends the nearest ones every second) as
   `kColTris` + `kColRegion`. Re-send regions when the world changes (doors open, objects move).
   `kColClear` on world change. This is the hardest and most game-specific part: reading your
   engine's physics shapes (Skyrim reads Havok: heightfields, compressed meshes, convex shapes).
   Mark stairs/ramps `kTriStairHelper`, land `kTriTerrain`.
6. **Teleport handshake** as above.
7. **Player puppet.** Disable the host's own player movement; set its position (and physics
   capsule) from `McState` every frame, interpolating from the tick fields.
8. **Camera.** Drive the host camera from `McState` eye pos + host-computed yaw/pitch, Minecraft
   FOV (vertical) converted to the host's convention, view bob, F5 mode/distance.
9. **Input.** Capture raw keyboard/mouse, convert to SDL scancodes/buttons, push to the input ring,
   swallow them from the host's controls, except a small allow-list (Skyrim keeps Esc, G activate,
   J, M, H, F9, console). Compute yaw/pitch from mouse deltas with Minecraft's sensitivity curve
   (`s = sens*0.6+0.2; factor = s³·8·0.15`). Send `kInReleaseAll` whenever focus leaves Minecraft.
10. **Overlay.** Acquire the newest triple-buffer slot and draw it full-screen on top (alpha
    blended; honour the bottom-up flag). Hide the host's own HUD and first-person arms.

### Tier 2: blocks
11. **Render ring consumer.** Upload the atlas; keep a mesh per section; draw them in the host's
    world pass (depth-tested), applying host sun/shadow/fog plus the vertex block/sky light.
    Translucent after opaque. Update atlas regions for animated textures.
12. `kRenLights` → host point lights (flicker for flames). `kRenSolids` → invisible collision so
    NPCs and items respect builds; also feed NPC pathing. `kRenDug` → hide/cut the host's
    geometry and collision in dug cells.
13. **World entities** (arrows, items, cracks, block outline) and `kRenScene`/`kRenAvatar`
    (entities, particles, third-person player).

### Tier 3: combat and the rest
14. **Actor table**: every nearby NPC, every frame.
15. **Event ring**: apply `kEvHitActor` through the host's real hit pipeline with a level-based
    multiplier (Skyrim: `5 + 0.25 × level`); `kEvPlayerDied` → kill the host player.
16. **Hurt**: intercept damage to the host player, **cancel it**, and send `kInHurt` (Skyrim
    divides by 5). Minecraft health is authoritative; armour, shields and totems are vanilla.
17. Optional: water grid, explosions, arrows stuck in actors, ragdoll, skill use, game hour.

## 6. Things to keep in mind

- **Translate, don't re-implement.** If you are writing Minecraft movement in the host or host AI
  in Java, stop. That is the design's core rule.
- **Bump nothing silently.** Any layout change: edit the header, mirror it in `Proto.java`, bump
  `kVersion` on both sides, and update `tools/fake_skyrim.py`. A mismatch is refused, not tolerated.
- **Threads.** Each ring has exactly one producer thread and one consumer thread. Skyrim writes
  collision from one thread, drains events on the main thread, and drains render data on the
  render thread with a per-frame byte budget so a big burst can't stall a frame.
- **Ring full is normal.** Collision writes return false when full: retry next frame. Input drops
  if Minecraft is a whole ring behind.
- **Loading screens stall heartbeats.** Minecraft tolerates 8 s. Set `kSkyLoading` and keep the
  heartbeat going where you can.
- **Restarts on either side.** The host must re-zero its regions on create (a stale mapping can
  survive if Minecraft kept it open). Minecraft resends its caches when the host PID changes. When
  Minecraft (re)connects (heartbeat back, or a new `mcPid`), the host resets its overlay swap,
  bumps `collisionEpoch` and resends all collision, and teleports Minecraft again (`Game.cpp`).
- **Elevation.** If the host runs as administrator the default DACL locks Minecraft out
  (`ERROR_ACCESS_DENIED`). Build an explicit DACL for the current user at medium integrity.
- **Scale and world height.** The mirror world uses a custom dimension type with min_y −1024,
  height 2048 (`fabric/src/main/resources/data/skycraft/dimension_type/mirror.json`). Your world (after scaling) must fit, or you need to offset it.
- **Interiors share one world.** Minecraft has a single mirror world, so separate host levels at
  the same coordinates see each other's blocks unless the host offsets them.
- **Windows only.** The transport uses Win32 APIs (and QPC for tick timestamps on both sides). A
  Linux/Proton host would need the same names visible to both processes.
- **Test without the host.** `tools/fake_skyrim.py` creates the mapping, streams a floor and
  stairs, holds keys, prints `McState`, drains events and saves the overlay as a PNG. Start a new
  port by making your host behave like that script.

---

## 7. A prompt for porting to another game

Fill in the bracketed parts and give it to an agent working in a fork of this repo with the host
game's modding SDK available.

```text
You are porting SkyCraft to a new host game: [GAME], modded with [SDK / framework, e.g. "UE4SS C++
mods", "BepInEx + Harmony", "Script Hook V", "a DLL injected with ..."], on Windows.

Read first, in this order: docs/DESIGN.md, docs/PORTING.md, protocol/skycraft_protocol.h,
tools/fake_skyrim.py, then skse/src/Link.cpp, skse/src/Game.cpp, skse/src/Input.cpp,
skse/src/Overlay.cpp, skse/src/Collision.cpp, skse/src/WorldRender.cpp and skse/src/Combat.cpp as
the reference host implementation.

Goal: a [GAME] mod that plays the role skse/ plays for Skyrim, driving the existing Fabric mod in
fabric/ UNCHANGED over the shared-memory protocol (version 11, mapping Local\SkyCraft_v1). Do not
change protocol/ or fabric/ unless a step below truly cannot work without it; if so, stop and
explain the change you need before making it, and keep Proto.java, the header, kVersion and
tools/fake_skyrim.py in sync.

Ground rules:
- Minecraft owns the player's body: position, physics, health, inventory, blocks, combat math.
  [GAME] owns the world, NPCs, camera, window, input capture and all drawing. Translate between
  them; never re-implement one game's mechanics in the other.
- All protocol coordinates are Minecraft space (blocks, Y up, +Z south, yaw 0 = south). Choose a
  units-per-block scale so [GAME]'s player height ≈ 1.8 blocks, write SkyToMc/McToSky and
  heading<->yaw helpers, and unit-test them first. [GAME] axes: [describe: up axis, handedness,
  units, rotation convention]. World height after scaling must fit y -1024 .. 1023.
- Seqlocks: seq odd while writing, +2 per write. Rings: one producer thread, one consumer
  thread, release/acquire on head/tail, 8-byte aligned messages with a pad-to-start message.
- Bump SkyState.seq exactly once per rendered [GAME] frame: Minecraft paces its frames on it.

Build in tiers, each ending in something testable in game. Report what works after each tier.
1. Link: create the mapping (with a DACL for the current user, medium integrity), zero host-owned
   regions, write version/pid/magic, heartbeat, SkyState every frame. Detect a running Minecraft
   via the "Local\SkyCraft_v1_minecraft" mutex, else launch it (Prism "--launch SkyCraft"). Log
   when Minecraft links (header.mcPid / heartbeat) and when it goes away (3 s).
2. Walk: stream [GAME]'s collision around the player (~40 blocks) as kColTris + kColRegion, with
   kColClear + a new collisionEpoch on level change. Implement the teleportSeq/teleportAck
   handshake. Disable [GAME]'s own player movement and puppet the player from McState
   (interpolate the 20 Hz tick fields on [GAME]'s clock with QPC). Drive [GAME]'s camera from
   McState (eye, FOV converted from vertical, bob, F5 mode). Capture raw input, convert to SDL
   scancodes/buttons, push to the input ring and swallow it, except an allow-list of [GAME] keys:
   [list, e.g. pause menu, interact, map]. Compute yaw/pitch from mouse deltas with Minecraft's
   sensitivity curve. Send kInReleaseAll on focus loss and set kSkyMenuOpen while a [GAME] menu
   owns input. Composite the overlay triple buffer full-screen on top; hide [GAME]'s HUD and
   first-person arms.
3. Blocks: consume the render ring on the render thread with a per-frame byte budget. Upload the
   atlas (and atlas regions), keep a mesh per section, draw them depth-tested in [GAME]'s world
   pass with its lighting/shadows/fog and the vertex block/sky light. Turn kRenLights into point
   lights, kRenSolids into invisible collision NPCs respect, kRenDug into hidden [GAME] geometry.
   Draw world entities, kRenScene and kRenAvatar. Clear everything when Minecraft's pid changes.
4. Combat: fill the actor table every frame with nearby NPCs. Drain the event ring on the main
   thread: apply kEvHitActor through [GAME]'s real damage pipeline (scaled by NPC level),
   kEvPlayerDied kills [GAME]'s player. Intercept damage to [GAME]'s player, cancel it, and send
   kInHurt (scale down so a typical hit costs a heart or two).
5. Optional: water grid, explosions, stuck arrows, ragdoll, skill use, time of day.

Test each tier first against a real Minecraft started with fabric\gradlew runClient, and compare
behaviour with tools/fake_skyrim.py. Log every link state change and protocol mismatch clearly to
[GAME]'s log file. Keep the [GAME] plugin in its own top-level folder (e.g. [game]/) beside skse/.
```

### What changes most between games

| Piece | Effort | Notes |
|---|---|---|
| Link, heartbeat, SkyState, input ring | Small | Mostly a port of `Link.cpp` and `Input.cpp` |
| Overlay compositing | Small–medium | Needs a hook at the end of the host's frame (D3D11/12/Vulkan Present) |
| Player puppet + camera | Medium | Needs a way to switch off the host's character controller but keep the player "in the world" for AI and triggers |
| Collision export | **Large** | Reading the host's physics engine (Havok, PhysX, Bullet, Chaos...) near the player |
| Drawing Minecraft's meshes in the host's frame | **Large** | Needs a mid-frame render hook with access to the depth buffer and host lighting |
| Combat bridge | Medium | Needs the host's "apply hit" function and a hook on damage to the player |
