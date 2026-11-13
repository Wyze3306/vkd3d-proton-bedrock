# vkd3d-proton for BedrockOnLinux

This is vkd3d-proton 3.0.1 as BedrockOnLinux ships it, plus one change aimed at
Minecraft: Bedrock Edition on NVIDIA. It is a fork for that one game, kept
small enough to rebase, not a general replacement for upstream.

## Branch layout

`main` is a straight line on top of upstream `v3.0.1`
(`3b10bd7a7ec6a7347e616cf8bea59333afec2255`):

1. `Revert "vkd3d: Get rid of obsolete NV_dgc paths."` restores
   `VK_NV_device_generated_commands`. NVIDIA's 550 driver exposes only the NV
   extension, so without it Minecraft's command signatures cannot be built at
   all. The tree after this commit is byte-identical to the one BedrockOnLinux
   builds from `restore-nv-dgc.patch`.
2. `keep the frame latency object ticking while occluded` (BedrockOnLinux #50).
3. `keep the frame rate limiter's busy-wait short under Wine` (BedrockOnLinux #202).
4. `tests: Accept NV_device_generated_commands as a DGC implementation.`
5. `vkd3d: Build command signatures that update root CBVs against a raw VA twin.`

Commits 1 to 3 are exactly the payload of BedrockOnLinux engine
`wow64-archs-native23`. Commits 4 and 5 are new here.

## Why: root CBVs on NVIDIA

vkd3d-proton passes root constant buffer views (root CBVs) to shaders in one of
two ways:

- **push descriptors**: real uniform buffer bindings, which NVIDIA reads
  through its constant cache. This is upstream's default on NVIDIA because, in
  upstream's words, "the difference in performance is profound (~15% in some
  cases)";
- **raw VAs**: 64-bit addresses in push constants, which shaders read as
  buffer device addresses. This is the default everywhere else.

A command signature that updates a root CBV from the GPU, through
`ExecuteIndirect`, can only be expressed with raw VAs, because device generated
commands can write push constants but not push descriptors. On NVIDIA,
upstream therefore refuses such signatures with `E_NOTIMPL`.

Minecraft creates 20 of them at start-up, against the one root signature that
all 224 of its pipelines use, before it creates any pipeline. Refused, the game
freezes on its first menu (BedrockOnLinux #27, #29, #30). BedrockOnLinux has
worked around that since then with `VKD3D_CONFIG=force_raw_va_cbv`, which makes
every root CBV in the game a raw VA, for every draw. Yet tracing the game's
D3D12 calls in menus and in a world showed that it **never calls
`ExecuteIndirect`** on that path. It pays the raw VA cost on every draw for a
feature it creates and then leaves unused.

Measured on 2026-11-12 (RTX 4060, Minecraft 1.26.60.5, uncapped, in a world,
gamescope headless at 1600x900), with a measurement-only build that kept push
descriptors and dropped the state template:

| root CBVs | FPS (three runs) | p99 frame time |
|---|---|---|
| raw VA (`force_raw_va_cbv`, what ships) | 502 / 548 / 617 | 5.5 / 5.2 / 4.4 ms |
| push descriptors | 785 / 785 / 778 | 2.6 to 2.8 ms |

That build could not ship, because an executed signature would have ignored
its root CBV updates. This change keeps that speed and stays correct when a
signature does run.

## What the change does

Push descriptors stay the default on NVIDIA. When a command signature updates
a root CBV that its root signature made a push descriptor:

- **Root signature twin.** A root signature that has push-descriptor root CBVs
  keeps a copy of its serialized blob. The first signature that needs it builds
  a *raw VA twin* from that blob: the same root signature with raw VA root
  CBVs. The state template is built against the twin. The twin's compatibility
  hash is salted so that its pipelines never share a cache entry with the
  originals.
- **Pipeline twin.** A pipeline created with such a root signature keeps a
  self-contained copy of its creation desc: shader bytecode, input layout,
  stream output and view instancing, with every string and array duplicated.
  The first `ExecuteIndirect` that needs the pipeline compiles it once against
  the twin root signature.
- **Swapping around ExecuteIndirect.** Around such a call, the command list
  binds the twin pipeline and twin root signature, then restores the
  application's own afterwards. Root constants are moved to the twin's push
  constant layout, which places root CBV addresses first, and root descriptors
  are set again from their saved addresses.
- **Everything else is unchanged.** Draws and dispatches outside such calls
  keep the push descriptor layout. AMD, Intel and any device already using raw
  VA root CBVs take the old code path, because none of their root signatures
  qualifies for a twin.

`VKD3D_CONFIG=force_raw_va_cbv` still behaves as before. To get the faster
path, stop setting it.

## Building

The same as upstream. For the PE DLLs that Wine loads:

```bash
meson setup build.64 --cross-file build-win64.txt --buildtype release -Dstrip=true
ninja -C build.64
```

For the native test suite, which runs on the host Vulkan driver:

```bash
meson setup build.native --buildtype release -Denable_tests=true
ninja -C build.native
VKD3D_TEST_FILTER=execute_indirect build.native/tests/d3d12
```

## Validation

On an RTX 4060 with NVIDIA 550.163.01, which exposes `VK_NV_device_generated_commands` only:

| build, `VKD3D_CONFIG` | `execute_indirect` tests |
|---|---|
| native23, unset | 2160 checks, 0 failures, cases 4, 5, 7 skipped: "Root parameter 1 is not a raw VA" |
| native23, `force_raw_va_cbv` | 2226 checks, 0 failures |
| this branch, unset | 2226 checks, 0 failures |
| this branch, `force_raw_va_cbv` | 2226 checks, 0 failures |

The full suite and an in-game measurement of this branch have not been run
yet; until they have, treat the speed-up above as measured on the
measurement-only build, not on this one.
