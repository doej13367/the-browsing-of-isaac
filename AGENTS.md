# Isaac native/Wasm decomp port

This repository is replacing the per-frame x86-emulation path with a verified
native WebAssembly port. Continue that real objective. Do not redefine success
as a demo, a thin wrapper around the emulator, or a small slice that merely has
green tests. The port is complete only when the shipped frame path no longer
depends on x86 emulation and runtime behavior is verified against the original.

## Worktree and artifact safety

- Work on `codex/decomp` unless the user explicitly selects another branch.
- The worktree may contain unrelated user changes. Inspect `git status`, edit
  only files required for the current boundary, and never reset or discard work.
- The executable and all binary-derived output are local evidence, not source.
  Never commit game executables, assets, Ghidra projects, decompiler listings,
  disassembly dumps, runtime captures, or generated Wasm/native objects.
- Local analysis roots must remain ignored: `/tools/`, `/output/`, and
  `/re/ghidra_project/`. Confirm important generated files with
  `git check-ignore -v` before finishing a work unit.
- The canonical local analysis input is `tools/isaac-ng.unpacked.exe`, currently
  SHA-256 `5129DF723E645DAAEA59514394195F3EA1DCE1671BB0433D724648A845017200`.
  Treat every RVA, VA, signature, and decompile as version-bound. Re-inventory
  and re-verify roots if the hash changes.

## Authoritative tracked sources

- `decomp/port-roots.json`: signature-backed root requests.
- `decomp/game-update-slice.json`: recovered offsets, predicates, boundaries,
  call receivers, evidence, runtime inputs, and host-event ordering.
- `native/decomp/game_update_slice.{h,cpp}`: freestanding C++20 translation and
  sparse Game-object capture/apply ABI.
- `scripts/decomp/game-update-model.mjs`: independent JavaScript behavioral
  oracle. Do not derive it mechanically from the C++ implementation.
- `scripts/decomp/verify-game-update-slice.mjs`: deterministic native-vs-Wasm
  differential corpus and recapture/resume driver.
- `scripts/decomp/build-game-update-slice.mjs`: warning-clean native build,
  explicit Wasm exports, and zero-import ABI report.
- `scripts/decomp/frame-path.mjs`: shipped frame-path selector, hybrid
  capture/apply (`runHybridGameUpdateTick`), residual `HostHandler`
  (`createResidualHostHandler` / `createLoggingHostHandler`), and multi-frame
  session (`createNativeUpdateSession`).
- `web/js/native-update-bridge.js`: platform-host boot of the native Update
  session (no PE on the Update tick when the slice loads).
- `docs/decomp-port.md`: chronological ABI checkpoint, hybrid path lifecycle,
  and exact next boundary.

Generated analysis belongs under `output/decomp/<first-12-sha256>/`. The cached
Ghidra project belongs under `re/ghidra_project/`. Both are intentionally ignored.

## Evidence rules

1. Start from exact machine control flow. Inspect instructions at the boundary,
   the complete decompiler body, call receivers/arguments, and every branch that
   determines externally visible behavior.
2. Run `node scripts/decomp/identify-zhl-address.mjs <VA>...` for each recovered
   call target. Use a source-level name only for an exact matching signature.
   Otherwise retain an address-stable name such as `opaque_call_009b7680`.
3. Never name a function from proximity, a string reference, one caller, or a
   plausible decompiler type. Record such evidence without promoting a guess.
4. Translate complete small helpers when all paths are understood. Keep larger
   or stateful functions as typed host events until their bodies are ported.
5. If an opaque call can mutate a later predicate, stop at a continuation,
   execute the host action, recapture the sparse fields, and resume. Do not use
   stale pre-call state to make a post-call decision.
6. Preserve x86 and float semantics explicitly: 32-bit wrap, low-byte tests,
   signed comparisons, raw bit copies, float32 rounding, NaN behavior where
   relevant, call ordering, and exact early-return boundaries.

Decompiler output is evidence only. Do not paste it into tracked source and do
not treat decompiler-generated types as authoritative.

## Port workflow

1. Record the current sequential VA and disassemble enough instructions to see
   the whole bounded block, including all exits and the next safe handoff.
2. Export/decompile unresolved callees from the hash-keyed Ghidra project. Keep
   custom seed TSVs, manifests, and C output under ignored `output/decomp/`.
3. Update `decomp/game-update-slice.json` before or with code: fields, runtime
   inputs, translated predicates, opaque boundaries, receivers, arguments,
   ordering, continuations, and evidence must remain explicit.
4. Increment the ABI version whenever a struct layout, field meaning,
   continuation, event, or exported contract changes. Keep C++ static assertions,
   JS layouts, verifier expectations, exports, tests, and docs synchronized.
5. Implement the C++ translation and a separately reasoned JS oracle. Add fixed
   edge cases first, then deterministic randomized differential coverage.
6. Advance `docs/decomp-port.md` to the exact next untranslated instruction.
   Never claim an FPS improvement until the slice is integrated and measured in
   the actual browser frame loop.

## Throughput: measured ways to decompile faster

Every rule below is backed by a measured outcome in this repository, not theory.
The unit of progress is boundaries removed and sites covered - report those.

1. **Hunt repeated structure before translating instructions.** Byte-diff
   candidate sibling functions first. One proven template + per-index law covers
   N sites in one unit: the 8 Lua class-create helpers are one 528-byte template
   (25 differing bytes) with exact laws over the registration index; one unit
   covered all 8. The install-record model covered 95 sites, the builder model
   413. Instruction-by-instruction translation of the same surface would have
   taken ~20 units.
2. **Validate with whole-body census, not hand-picked cases.** Scanning the
   entire body and asserting the model reproduces 100% of matching sites
   (0 mismatches over 3377 pushes; 0 over 1824 rel32 calls; 82/82 strides) is
   both stronger and faster than case-by-case reasoning.
3. **Peel the probe, collapse the cascade.** Prefer boundaries whose removal
   converts many host edges into one pure decision: translating two collectible
   probes turned a ten-stage revive cascade into a pure decision tree in one
   unit. A leaf peel of off-frame-path code (device hot-plug, menu-only paths)
   costs the same effort and shrinks nothing - check reachability from the
   frame path before choosing a target.
4. **Escalate cross-cutting blockers to a standing decision or a new family;
   never stop at them repeatedly.** Sections stalled at the allocator in four
   places until one policy decision (wrappers = game logic, raw alloc = platform
   primitive) unblocked all of them at once and re-opened `_Tree::_Erase` as
   translatable. ANM2 and SFX became families because three sections each kept
   stopping at the same bodies.
5. **When two corrections share a defect class, run a class-wide audit
   immediately.** A single read-only stale-state audit found 11 instances
   (5 Update-wired) in one sweep - far cheaper per defect than the incidental
   discovery that found the first four. Audit findings route to file owners.
6. **Write capture contracts at section boundaries.** Room recovered the RNG and
   published exact field addresses/sizes/direction; Update wired it without
   re-deriving anything. Frozen byte-for-byte helper contracts are what make
   parallel sections safe.
7. **Prefer exact-ZHL targets and weigh pattern length.** Units on exact-ZHL
   roots (known signature and receiver) land faster with fewer corrections than
   deep address-stable bodies. But: an exact match is necessary, not sufficient
   (a 24-byte exact match at `0x0095b310` is disproven at its callsite), a miss
   does not mean absent (`Game::Update` misses its own pattern), and a 7-byte
   pattern is weak evidence. Staleness is per-function in this build.
8. **Batch the evidence pass.** Run `identify-zhl-address.mjs` on all VAs of a
   unit in one invocation, dump all disasm spans to the section-notes dir once,
   then work from files. Re-running tools per instruction is pure overhead.
9. **Translate right the first time - the known defect classes.** Every
   correction to a landed peel was one of: post-call value folded to a pre-call
   snapshot; recapture taken **unconditionally** where the PE recaptures only on
   one path (both skip paths keep the pre-call registers - "always after" is as
   wrong as "always before"); strict/non-strict or signed/unsigned compare
   misread; full-word test where the PE tests a low byte; loop bound folded
   once where the PE re-derives it per iteration; NaN direction of
   `comiss`/`cmov` inverted; constant assumed where the callee rewrites through
   a pointer. Check each at translation time; mutation-check each at test time.
   A green differential with an oracle derived from your own C++ catches none
   of them - and a wrong assertion actively **locks the bug in**: one confirmed
   defect had two pre-existing assertions pinning the wrong NaN result, one of
   them directly under a comment rationalising it. When two helpers model the
   same instruction range, assert they agree - a cross-helper differential
   catches what per-helper expectations cannot.
10. **A census is only as complete as its reach-site enumeration - prove the
    channel, and anchor the decode.** Three measured failures in one session:
    a byte-diff census that enumerated only direct `FF15` calls in a fixed
    window structurally could not see **60 register-held** call sites
    (`mov reg,[imm]` + `call reg`) and under-counted a surface by 36 bodies; a
    back-scan for displacements reported **44 phantom** sites that a linear
    decode showed do not exist; and three "extra callsites" turned out to be
    misdecodes of a preceding jump-address table. So: enumerate every call form
    (direct, register-held with register-lifetime tracking, thunk), decode
    linearly from a known anchor (function start or an int3 run), re-derive any
    surprising count from a second anchor, and close the argument by proving
    your channel is the ONLY one (that surface was provable at 383 exactly
    because `luaL_setfuncs` is not imported, so `pushcclosure` is the sole
    install path). Report the censused number, never a `~`.

    **The worst variant produces a clean-looking EMPTY result.** Capstone's
    instruction generator HALTS at the first undecodable byte. One census
    silently covered 35,730 of ~2.09M instructions (1.7%) and reported *zero*
    writers for every tracked global - a census that looks tidy and is
    meaningless. A linear pass is only linear with a resync loop that skips the
    bad byte and continues. If a writer census comes back empty, assume the
    decode died before assuming the address is constant. (Instruction totals
    reported by different units - 2,094,319 / 2,094,686 / 2,094,788 - differ by
    decode configuration; state which you used.)

    Related trap when scripting restores: Python's `open(..., encoding=...)`
    normalizes newlines on READ, so a text-mode round-trip can silently CRLF-ize
    a file **and** its own hash check will not see it. Hash bytes, not decoded
    text, and prefer the test runner's own restore path.
11. **Verify the assigned target is still open before doing anything else.**
    Measured: three units in one session opened with a target that was already
    landed, because docs lagged the tree by 3-6 ABI versions. Read the family
    header/model/tests and the instruction stream FIRST; if it is done, say so
    and move to the family's real frontier from its own recorded handoff. Count
    **assertions per export**, not tests per file - a family can be "at v8" with
    the target implemented and never behaviourally asserted.
12. **Keep the honest counter.** Narrowed is not removed. A unit that translates
    a whole function but leaves the boundary (sentinel self-links, one aliasing
    question) reports the count unchanged and names the exact blocker - that
    names the next unit's target.

## Read the PE through its section table, never a fixed offset

`VA -> file offset` must be resolved from the section headers. A `.text`-only
formula (`0x400 + (va - 0x401000)`) is correct for code and **wrong by `0xE00`
for `.rdata`** - and because the bad offset still lands inside `.rdata`, it
returns plausible garbage instead of failing. One section recorded a default
string as `"stage 8"` when the PE says `"unknown"`, and `"%s%s"` as `"htBegin"`.

| Section | VA | virtual size | raw ptr | raw size |
|---------|-----|--------------|---------|----------|
| `.text` | `0x00401000` | `0x00716134` | `0x00000400` | `0x00716200` |
| `.rdata` | `0x00b18000` | `0x000df948` | `0x00716600` | `0x000dfa00` |
| `.data` | `0x00bf8000` | `0x000a4aa4` | `0x007f6000` | `0x00069e00` |

`.data`'s raw size is much smaller than its virtual size: addresses past the raw
end are **zero at load** and are not file-backed. A VA-to-offset helper should
raise on those rather than return bytes.

A measured instance, found six ABI versions late: two "float constants" at
`DAT_00c7b640`/`0x00c7b644` were pinned as `0x3760371c`/`0x37873770` in a
header, a `.cpp`, a JS oracle **and** its tests - all four wrong together. The
naive `.data` formula put them at file offset `0x00879640`, inside `.reloc`;
the bytes were base-relocation HIGHLOW entries that bit-cast to plausible tiny
floats (1.34e-5), so nothing looked wrong. The address is `0x19840` bytes past
`.data`'s raw end, i.e. **zero at load**, and a whole-`.text` census found 5023
readers and ZERO writers - so the true value is `+0.0f` for the process
lifetime. Cross-check a suspicious constant against its writers: a
never-written zero-at-load address cannot hold a nonzero constant.

A zero-at-load BSS address is only a constant if you have censused its
**writers**, and the claim must be bounded to the exact range you censused.
Measured, both directions, by a linear decode of all 2,094,319 `.text`
instructions (a back-scan for displacements MISDECODES - one run reported 44
phantom `add eax,0xc79100` sites that a linear pass showed do not exist):

- `0x00c7169c` is **runtime state, not a constant** - 4421 reads and **2 real
  writers** (`0x009aaab0`, `0x009ab8cc`). Only its field offsets are constants.
- `0x00c79100` **is** safe to treat as constant zero - 55 reads, 0 writes, and
  its one address escape is a provably read-only predicate - but ONLY over
  `0xc79100..0xc7910f`. `0xc79110`/`0xc79114` ARE written (`0x004012ce`,
  `0x0095426a`), so the claim does not extend past `+0x10`.

Any constant, string, or table recorded from a raw offset read of a
non-`.text` address is suspect and must be re-checked. Numeric constants that
were independently cross-validated (against a second consumer, a serializer, or
an arithmetic identity) are unaffected - that redundancy is exactly why it is
required.

## A green suite can mean "nothing was asserted"

One family shipped an ABI version in which thirteen new helpers were **imported
and listed in the exports table but never asserted on** - no test touched their
behaviour, the suite was green, and the header even named a helper that did not
exist. Implementation-without-verification passes every check this repo runs.

So a work unit is not complete when the tests pass; it is complete when each new
helper has at least one behavioural assertion and a mutant proving that
assertion discriminates. When you inherit a version you did not land, verify its
coverage before building on it - count assertions per new export, not tests per
file.

**A sixth way: the HARNESS silently neuters the input.** Three measured
instances, all found by mutants rather than by review:
1. the adapter round-trip's `?? 0` expectation coincided with pre-zeroed
   scratch, so a nonzero row would have failed but no row was ever nonzero;
2. `resumeGameUpdateRoomTransitionEffect` was called with **no runtime inputs**
   while the Wasm export forwards them - the two sides driven with different
   inputs, and no corpus case reached the block by that route for many versions;
3. `verify-game-update-slice.mjs` copies runtime keys through an explicit
   **per-key whitelist**, so a unit's four NEW corpus keys arrived as zero and
   every new differential case was a silent no-op.
When you add a runtime input, prove the harness actually delivers it - assert a
case where the new input CHANGES the result, not merely that the suite is green.

**A fifth way, measured 2026-08-08: green in one harness reported as green in
another.** A v70 unit reported "Differential passes - 5191 cases at ABI 70" as
evidence its work was verified. That was true of `verify-slice`, a SEPARATE
script - meanwhile six of its own slice-level behavioural tests were throwing
`ReferenceError: walk is not defined` before their first assertion, because a
module-scope helper called a symbol that only existed as a `const` inside two
other tests' callbacks. The translation shipped with **zero executed
slice-level behavioural assertions**. Two lessons: name the harness when you
claim a pass, and when you inherit assertions that have never actually
executed, mutation-check them before trusting them (the recovery unit did, and
killed two).

**The four ways green has lied in this port**, in ascending order of nastiness:

1. **Nothing was asserted** - helpers imported and export-listed, never exercised.
2. **A masked differential** - the oracle was right, but every Wasm-side draw was
   pre-masked `& 0xff`, so the divergent input was never presented.
3. **A wrong assertion** - a fixed expectation pinning the incorrect answer,
   sometimes with a comment rationalising it. This actively locks the defect in.
4. **A wrong "PE truth" reference** - one section's brute-force reference model,
   written specifically to be the ground truth, repeated the *same* misreading as
   both the C++ and the JS oracle, and a fixed assertion pinned the wrong result.
   Three independent-looking implementations agreed and all three were wrong.

The defence against (4) is the only one that is not just "add coverage":
**re-transcribe the reference branch-by-branch from the instruction stream**,
not from your understanding of what the function does. If your PE-truth
reference contains a tidy `if (n <= 0) return 0` where the machine spells
`test eax,eax ; je` plus an unsigned `jb` loop-back, it is not a reference.

## Known toolchain defect: `uint8_t` helper parameters are silently wrong

A helper that models an x86 byte test (`test al,al`, `cmp byte ptr …`) must NOT
take a `uint8_t` parameter. The Wasm ABI does not narrow i32 arguments, and
`-O2` deletes the in-body mask because it can prove a `uint8_t` parameter is
already in range. The shipped module then disagrees with the PE for any argument
above `0xff`.

This is **not hypothetical and not section-specific**. One family found nine
exports divergent on the shipped module by direct probe - e.g.
`isaac_exit_415800_virtual_ok(0x100)` returned `1` where the PE's `test al,al`
sees `AL == 0` and does not take the gate. It had survived 28 ABI versions
because **the oracle was right and only the oracle side ever saw `0x100`**: every
Wasm-side draw was masked `& 0xff` before the call, so the differential could
never expose it. A census at the time of writing found ~181 at-risk parameters
across six families.

Required, in every family:

- Take `uint32_t` (or wider) and re-narrow explicitly in the body.
- Add a static assertion that no `uint8_t` scalar parameter remains.
- Drive every byte-gate export across the boundary with wide values (`0x100`,
  `0x1ff`, `0xffffffff`, …) - **do not pre-mask the Wasm-side argument**, or you
  will reproduce exactly the blind spot that hid this for 28 versions.
- Mutation-check it **behaviourally**: a mutant that drops the mask while keeping
  the `uint32_t` signature must fail a test. A static/grep check alone does not
  prove the test discriminates.

## Required verification for every ABI work unit

Run all of these before reporting a boundary complete:

```powershell
node --test tests/decomp-game-update-slice.test.js tests/decomp-pipeline.test.js
npm run decomp:verify-slice
npm test
node scripts/check-repo-safety.mjs
git diff --check
```

Also verify:

- `output/decomp/wasm-slice/abi.json` reports zero Wasm imports and lists every
  new direct and scratch export.
- `output/decomp/wasm-slice/verification.json` reports `result: "pass"` and
  includes meaningful cases for each new continuation.
- `decomp/game-update-slice.json` parses as JSON.
- No stale ABI version, renamed field, or old struct size remains in tracked
  decomp sources/tests/docs.
- Proprietary and generated outputs are still ignored.

## Current checkpoint

Update ABI **99** (state 524 / runtime 23696 / events 1260 / constants 32).
`Game::Update` is instruction-complete (`ret` @ `0x006fbbba`). All remaining
Update work is depth: **24** opaque host residuals, **27** removed (of 51).
The Update tick runs native Wasm (`usesX86Emulation = false`). Live rAF is
still a hybrid: typed residual hosts plus unpinned Render / Input / Exit / Lua
roots.

Family ABI:

| Family | ABI |
| --- | ---: |
| Update | 99 |
| Lua | 103 |
| Room | 87 |
| ANM2 | 77 |
| Render shell | 74 |
| Render root | 73 |
| Frame-opaque | 64 |
| ProcessInput | 52 |
| Frame-effect | 51 |
| Exit | 48 |
| Player manager | 45 |
| PlayerHUD | 44 |
| SFX | 41 |
| HUD | 39 |
| PGD | 35 |
| Room-transition | 30 |
| Log | 25 |
| Game state | 20 |
| Alloc | 4 |

Remaining Update residuals (full ledger in `docs/PROGRESS.md`):

- **14 narrowed** - still emit a host event (idx 0, 3, 4, 9, 13, 14, 15, 17, 18, 31, 32, 34, 36, 43). Record 0 stays: `0x83b850` RecomputeAll is irreducible.
- **7 host bodies** - idx 2, 5, 6, 8, 11, 35, 37 (menu update/open, `0x98dba0` walk, notify widget, frame-effect nested stage, rain spawn, mid tail).
- **3 declined** - idx 7 `0x40c7f0` (Win32 CS + alloc + virtual dtor), idx 22 room-transition body (`ANM2::Reset` / Rewind outside the Game buffer), idx 48 `lua_pcallk` of mod bytecode.

Next: peel a reachable narrowed record that is not declined. Skip record 0.

## Tidy handoff

At the end of each work unit, leave one exact next VA, one passing verification
report, synchronized ABI metadata, and no tracked binary-derived material.
Report what is translated, what remains an address-stable host action, and
whether the code is integrated into the live frame loop. A verified slice is
progress; it is not proof that the full decomp or FPS objective is complete.
