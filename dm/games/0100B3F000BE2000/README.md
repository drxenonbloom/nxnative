# POKKÉN TOURNAMENT DX (`0100B3F000BE2000`)

Update 1.3.3 (v458752) over the base game, SDK 4.4.1 (its own `sdk` build). Bandai Namco Studios, in-house engine, NVN, Bandai Namco `.drp` archives, CRI Sofdec2 / VP9 movies through `subsdk1` (`NintendoSDK_movie`). Modules `main`, `sdk`, `subsdk0` (GLSLC) and `subsdk1` (`NintendoSDK_movie`). Plays on the Mac (`PokkenTournamentDX.app`); 60 fps, as on the Switch.

Status: on the Mac, the boot, company logos (Bandai Namco, The Pokémon Company), the intro movie (`opening01.webm`), title screen (L+R), main menu, Practice / Single Battle, Pokémon character selection, battle intro cutscenes, and full 3D battle gameplay; 60 fps as on the Switch, audio via AudioToolbox / CoreAudio, about 2.2–2.5 GB resident and 1.2 GB of Metal allocations, no red line. The title is up about 12 s after launch.

## What it needs

| Setting | Where | Why |
|---|---|---|
| Multi-output transform feedback GPU skinning | [`dmsh_build.py`](../../gfx/tools/dmsh_build.py) `tf_layout` & `vs_tf` | character meshes capture 1–4 interleaved generic outputs (`locn0`–`locn3`, 16–64 byte strides) into feedback buffer 0; see [Graphics and shaders](#graphics-and-shaders) |
| Bandai Namco `.drp` container extraction | [`shader_split.py`](../../gfx/tools/shader_split.py) `drp_shaders` | `.drp` archives in `NX/shader/` hold the NVN shaders (encrypted with Xoroshiro128+ in v0 and Xorshift128 in v458752); see [Files and shaders](#files-and-shaders) |
| Standard 4 GB block: `mem_reserve 0x100000000`, `mem_alias 0x80000000`, `mem_stack 0x10000000`, `app_pool 0x70000000` | [`game.tsv`](game.tsv) | default 4 GB address space layout; no system resource declared in NPDM |
| `tex_flush_upload on` & `tex_dirty_upload on` | `game.tsv` | dynamically generated and CPU-written UI and character textures published without explicit driver writes |
| `pad_layout full` | `game.tsv` | fighting game requiring full D-pad, face buttons (A/B/X/Y), shoulder triggers (L/R, ZL/ZR), and stick clicks |

## Frame rate

- **Current:** 60 fps, as on the Switch. The engine paces by present interval 1 during menus and 3D battles, with 30 fps playback for prerendered video sequences.

## Memory

- Standard 4 GB layout without system resource: 2 GB alias region (`0x80000000`), 256 MB stack (`0x10000000`), 1.75 GB application pool (`0x70000000`).
- Resident memory stays around 2.2–2.5 GB with ~1.2 GB Metal allocation across battle stages.

## Files and shaders

- The RomFS holds 15,507 files (3.42 GB) organized under `NX/`:
  - `NX/packdata/`: 3,843 `.drp` resource archives holding stages, character models, and textures.
  - `NX/movie/`: 10 video files (MP4 logos and WebM opening/character cutscenes).
  - `NX/sound/`: 226 CRI `.nus3bank` sound banks.
  - `NX/shader/`: 4 `.drp` archives (`shader.drp`, `effect.drp`, `main.drp`, `font.drp`) holding all 381 compiled Maxwell NVN shaders (`shader 284`, `effect 50`, `main 35`, `font 12`).
- The update (v458752, 0.77 GB) is a delta over the base game.
- Encryption:
  - `v0` (1.0.0): compressed `.drp` files use Xoroshiro128+ (mode 5).
  - `v458752` (1.3.3): compressed `.drp` files use Xorshift128 (mode 4).
  - `shader_split.py` implements `drp_shaders` and `_drp_inflate5` to decompress and extract both formats directly.

## Graphics and shaders

- **Transform feedback GPU skinning:**
  - *Pokkén Tournament DX* skins Pokémon fighter and support meshes using transform feedback shaders in `main` (`vs_main.bin#0..3` in `v0` and `vs_main.bin#9..12` in `v458752`), reading the output as a vertex stream for the rasterization pass.
  - Unlike *Pokémon Legends: Z-A*, feedback buffer 0 captures up to 4 interleaved generic outputs:
    - `vs_main.bin#0` & `#9`: 1 output (`locn0`), component count 4, stride 16 bytes.
    - `vs_main.bin#1` & `#10`: 2 outputs (`locn1`, `locn0`), component counts 4, stride 32 bytes.
    - `vs_main.bin#2` & `#11`: 3 outputs (`locn2`, `locn1`, `locn0`), component counts 4, stride 48 bytes.
    - `vs_main.bin#3` & `#12` (`79966d1954052bc6`): 4 outputs (`locn2` position, `locn1` normal, `locn0` tangent, `locn3` binormal), component counts 4, stride 64 bytes.
  - In `dmsh_build.py`:
    - `ctl[0x6e8 + b]` is the output varyings count `nv` for buffer `b` (not a boolean enable).
    - Varyings are indexed at `ctl[0xdc + 0x80 * b : +nv]` and component counts at `ctl[0x4dc + 0x80 * b : +nv]`.
    - `vs_tf` generates `main0_tf` storing each output word sequentially into `dm_tf{b}[vid * wstride + i]`.
    - Total stride `4 * sum(comps)` is recorded in `tf_strides` in the reflection json.

## Mac routes

- The game boots into the Bandai Namco and Pokémon Company video logos, followed by `opening01.webm`.
- Pressing A skips the video to the title screen.
- At the title screen, pressing **L + R** enters the main menu.
- From the main menu, selecting Single Battle or Practice enters character selection and loads into the 3D arena.
- Automated boot and menu navigation script: `DM_KEYS="5:A,8.5:A,12:L+R~0.3,16:A,19:A,22:A,25:A,28:A,31:A"`.

## Found on this game

General engine and toolchain improvements first needed here:

- **Bandai Namco `.drp` archive extraction** (`shader_split.py`): support for both Xoroshiro128+ and Xorshift128 container formats.
- **Multi-output interleaved transform feedback** (`dmsh_build.py`): support for vertex shaders capturing multiple interleaved outputs into a single transform feedback buffer.
- **Standalone ad-hoc bundle signing** (`pack_mac_app.py`): signing the Mach-O binary independently before bundle assembly to avoid recursive directory sealing timeouts.

General engine and toolchain improvements first needed here:

- **Bandai Namco `.drp` archive extraction** (`shader_split.py`): support for both Xoroshiro128+ and Xorshift128 container formats.
- **Multi-output interleaved transform feedback** (`dmsh_build.py`): support for vertex shaders capturing multiple interleaved outputs into a single transform feedback buffer.
- **Standalone ad-hoc bundle signing** (`pack_mac_app.py`): signing the Mach-O binary independently before bundle assembly to avoid recursive directory sealing timeouts.

