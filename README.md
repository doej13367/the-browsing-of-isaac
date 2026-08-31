# The Browsing of Isaac

Native/Wasm decompilation of **The Binding of Isaac: Repentance+**.

Does not ship the game, the PE, assets, or generated Wasm.

![Decompilation progress](docs/progress.svg)

## Status

| | |
| --- | --- |
| `Game::Update` | Instruction-complete (`ret` @ `0x006fbbba`) |
| Opaque records | **27 / 51** removed · **24** remaining |
| Update ABI | **99** · state 524 · runtime 23696 · events 1260 |
| Live frame path | Native Update tick + typed residual hosts |
| x86 emulation | Removed |

Family ABIs: Lua 103 · Room 87 · ANM2 77 · render-shell 74 · render root 73 · frame-opaque 64 · ProcessInput 52 · frame-effect 51 · Exit 48 · PM 45 · PlayerHUD 44 · SFX 41 · HUD 39 · PGD 35 · room-transition 30 · log 25 · game-state 20 · alloc 4

## Remaining

- **14** Update records narrowed but still host
- **7** untranslated host bodies (menu, walks, rain, mid-tail)
- **3** assessed / declined (`0x40c7f0`, room-transition body, `lua_pcallk`)
- Render, Input, Exit, and Lua are not live PE-free roots yet

Ledger: [`docs/PROGRESS.md`](docs/PROGRESS.md)

## Layout

| Path | |
| --- | --- |
| `native/decomp/` | C++ translations |
| `scripts/decomp/` | JS oracles, build, verify |
| `decomp/` | ABI contracts |
| `web/js/` | Native Update / Render bridge |

## Build

Node 18+ and Emscripten 6.x. The PE is not required.

```bat
call %USERPROFILE%\emsdk\emsdk_env.bat
npm run decomp:build-slice
npm run decomp:build-render-slice
```

You must own the game. Local PE: `tools/isaac-ng.unpacked.exe` (SHA-256 `5129DF723E645DAAEA59514394195F3EA1DCE1671BB0433D724648A845017200`).
