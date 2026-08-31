# Remaining work — detailed ledger

Authoritative counts come from `decomp/game-update-slice.json` and
`grep ABI_VERSION scripts/decomp/*-model.mjs`. Refreshed **2026-08-31**.

If this file and a header disagree, the header wins.

## Scoreboard

```
Game::Update body          ████████████████████  100%   ret @ 0x006fbbba
Opaque records removed     ██████████░░░░░░░░░░   53%   27 removed, 24 remain
  of the 24 remaining      ██████████████▒▒▒░░░        14 narrowed / 7 host / 3 declined
Live PE-free frame path    ██████░░░░░░░░░░░░░░   ~30%  native Update, hybrid residuals
```

| Metric | Value |
| --- | --- |
| Update ABI | 99 |
| State / runtime / events / constants | 524 / 23696 / 1260 / 32 bytes |
| Opaque boundaries remaining | **24** |
| Resolved (removed) | **27** |
| Original ledger size | 51 |
| `usesX86Emulation` on the Update tick | `false` when the slice loads |

## Family ABI (depth, not percent-complete)

| Family | ABI | Files | Still owed to the frame path |
| --- | ---: | --- | --- |
| Update | 99 | `game_update_slice.{h,cpp}`, `game-update-model.mjs` | 24 residuals below |
| Lua | 103 | `lua_engine_pure_helpers.*` | `lua_pcallk` of registry mods (record 48); live Lua root unpinned |
| Room | 87 | `room_pure_helpers.*` | spawn/search/bind, water-band type-7, rain loop |
| ANM2 | 77 | `anm2_pure_helpers.*` | Load/Reset/Rewind/overlay hosts on records 4, 8, 22 |
| Render shell | 74 | `render_shell_pure_helpers.*` | consumed by render root |
| Render root | 73 | `game_render_slice.*` | tail bodies; not a PE-free live root yet |
| Frame-opaque | 64 | `frame_opaque_pure_helpers.*` | 4212c0 removed; list helpers still typed-host at some sites |
| ProcessInput | 52 | `process_input_pure_helpers.*` | live rAF uses a host-side key bridge instead |
| Frame-effect | 51 | `frame_effect_pure_helpers.*` | nested `0x006fdc10` (record 11) still host |
| Exit | 48 | `exit_pure_helpers.*` | live Exit root unpinned; `0x40c7f0` declined |
| Player manager | 45 | `player_manager_update_pure_helpers.*` | death walk cannot expand TriggerDeath |
| PlayerHUD | 44 | `playerhud_post_update_pure_helpers.*` | slot virtuals inside record 0 |
| SFX | 41 | `sfx_pure_helpers.*` | device virtuals stay platform audio |
| HUD | 39 | `hud_post_update_pure_helpers.*` | records 3–4 mode bodies |
| PGD | 35 | `pgd_pure_helpers.*` | not on the hot Update residual list |
| Room-transition | 30 | `room_transition_engine_pure_helpers.*` | engine *body* declined (record 22); prefix removed |
| Log | 25 | `log_pure_helpers.*` | vsnprintf/OutputDebugString/file tail is platform |
| Game state | 20 | `game_state_pure_helpers.*` | snapshot seeding of the live buffers |
| Alloc | 4 | `alloc_pure_helpers.*` | raw `new`/`free` stay host by policy |

## Remaining Update records

Copied from `opaqueBoundaries` in `decomp/game-update-slice.json`.

### Narrowed (14) — peel or capture, do not re-census from zero

| Idx | Name | VA | Next fact |
| ---: | --- | --- | --- |
| 0 | `opaqueCall009a2b30` | `0x009a2b30` | **Will not remove.** `0x83b850` RecomputeAll is irreducible. Do not re-open. |
| 3 | `opaqueCall0092f1c0` | `0x0092f1c0` | Fold or type Room `0x4186c0` on always-host cases 0 and 3. |
| 4 | `opaqueCall008318a0` | `0x008318a0` | Modes 1–4: ANM2 pair + per-player walks. |
| 9 | `opaqueState24eccUpdate` | `0x008ef990` | Virtual Notify on global listener registry. Near maximum narrowing. |
| 13 | `opaqueRoomUpdatePrefixB1` | B1 span | Search `0x813520` / spawn `0x428b20` / bind `0x7ef420`. |
| 14 | `opaqueRoomUpdatePrefixB2` | `0x00802e20..0x00803321` | Remaining item-pool/spawn special. |
| 15 | `opaqueRoomUpdatePrefixB3B7` | B3–B7 | Capture-capped residual plan. |
| 17 | `opaqueRoomUpdatePrefixB9B11` | B9–B11 | Challenge result + death-spawn fatal. |
| 18 | `roomTriggerOutput` | `0x00823080` | Three host call classes after pure PH1–PH3 scan. |
| 31 | `roomTriggerClearAwardsTail` | `0x008068f0` | Removal declined v69 (award spawner + music). Narrow only. |
| 32 | `opaqueCall0098dba0PlayerWalk` | `0x0098dba0` | Virtual `[[player+0x370]+0x14]`. Entity-surface capture exists; dispatch stays host. |
| 34 | `opaqueRoomUpdateAmbient` | ambient | Gates H3/H5 pure; residual host. |
| 36 | `opaqueRoomUpdateTailWaterB16` | `0x00823540` | Type-7 / lava spawn grid walks lack per-cell capture rows. |
| 43 | `playerManagerUpdateDeath` | `0x009bb5d0` | Do **not** pre-expand TriggerDeath (defect class 1: mid-walk mutation). |

### Host bodies (7) — translate or split, then maybe remove

| Idx | Name | VA | Size / shape |
| ---: | --- | --- | --- |
| 2 | `opaqueMenuGateUpdate` | `0x009b6840` | 645 insns, 46 E8 + 3 indirect, every frame via `0x006fb47b`. |
| 5 | `opaqueCall0098dba0` | `0x0098dba0` | Manager-entry player walk. Timer gate already folded. |
| 6 | `opaqueCall00746560` | `0x00746560` | String build + ANM2 widget. Gate typed at v65. |
| 8 | `opaqueMenuGateOpen` | `0x009b7680` | ~1979 insns / 128 E8 menu cluster. |
| 11 | `opaqueCall006fdc10` | `0x006fdc10` | Frame-effect nested stage. |
| 35 | `opaqueRoomUpdateTailRain` | `0x00802a80` | Rain spawn loop (double-booked with B1). |
| 37 | `opaqueRoomUpdateTailMid` | mid tail | Curse + `0x801ee0` + `0x800500`, always-path. |

### Declined (3) — closed unless new evidence appears

| Idx | Name | VA | Closed because |
| ---: | --- | --- | --- |
| 7 | `opaqueCall0040c7f0` | `0x0040c7f0` | Always-path Win32 CS + `operator new` + virtual dtor + second lock. |
| 22 | `opaqueRoomTransitionEngineBody` | engine body | ANM2 Reset/Rewind stores outside the Game buffer. |
| 48 | `opaqueCall008607a0` | `0x008607a0` | `lua_pcallk` of mod bytecode. Census complete; one reach site. |

## Resolved records (27) — do not re-open

`0x4257b0`, `opaqueGlobal4aba0Refresh`, `opaqueRoomTransitionEnginePrefix`,
`opaqueCall004212c0`, `opaqueCall0092e300`, `opaqueRoomUpdateTailMidRestock`,
`opaqueCall006fd7c0`, `opaqueRoomUpdatePrefixB8`, `opaqueRoomUpdateTailPath`,
`opaqueRoomUpdateClearPath`, `opaqueRoomUpdateClearDoorSlots`,
`opaqueRoomUpdateClearDoors`, `opaqueRoomUpdateTailEntity`,
`playerManagerUpdateHeartbeatSfxStop`, `opaqueCall006fd7c0Mode4Sfx`,
`playerManagerUpdateHeartbeatSfxPlay`, `roomTriggerClearStats`,
`opaqueRoomUpdateTailMid7230Spawn`, `opaqueRoomUpdateTailMid706cExpire`,
`playerManagerUpdateHeartbeatSfxUpdate`, `opaqueFrameEffect6fd7c0Shell`,
`playerManagerUpdateHeartbeat`, `roomTriggerClearAwardsGreed`,
`roomTriggerClearAwardsNonGreed`, `roomTriggerClearAudioNonGreed`,
`roomTriggerClearAudio`, `roomTriggerClearAwardsBit7`.

## Integration remaining (outside the opaque count)

1. **Bridge capture completeness.** Blob-gated pures (B8, B16, rain rows, greed probe, audio packs, entity-surface) fall back to the monolithic residual when the live bridge does not fill the voucher. Filling those lanes is what makes removals visible in the browser, not only in the differential.
2. **Render as a live root.** The render slice is built and wired; the GL host still models some guest memory. Tail VAs `0x00817b53` / `0x0081d20a`.
3. **Input as a live root.** `frame-input-bridge.mjs` is a host-side stand-in. Recovered `ProcessInput` helpers are not the rAF source.
4. **Exit / Lua VM as live roots.** Unpinned.
5. **No FPS claim** until the hybrid residual set is measured in the actual Chromium rAF loop.

## How to pick the next unit

1. Confirm the assigned target is still in `opaqueBoundaries` (docs lag ABI).
2. Prefer a narrowed record whose peel drops a host edge the live path actually takes.
3. Skip declined rows and record 0’s `0x83b850`.
4. Hunt a repeated structure (byte-diff siblings) before translating instruction-by-instruction.
5. After the landing: bump ABI only if layout/event/continuation changed; keep C++, JS oracle, JSON, static_asserts, and this file in agreement.
