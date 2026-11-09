# hookshot-bc250

> **Disclaimer: this project does not support or condone piracy.** No game data, keys,
> firmware or other copyrighted console assets are included, provided or linked here.
> Hookshot builds the game **from your own copy, on your own PC**; everything the build
> creates (the executable, the generated code, the shader caches, your saves) comes from
> your game and stays on your machine. Do not open issues about obtaining games, keys or
> firmware — they will be closed. Never share files generated from your game.

A [Hookshot](https://github.com/NotRexed/Hookshot) fork tracking **The Legend of Zelda:
Ocarina of Time (Nintendo Switch 2)** running natively on the **AMD BC-250 console**
(GFX1013-class iGPU, RADV, 1080p TV output, 16 GB shared memory).

All credit for the recompilation itself belongs to NotRexed and the Hookshot contributors
(see upstream's readme); this repo only carries the BC-250-specific deltas and validation
notes on top. GPLv3, like upstream.

## Why this fork

The BC-250's GPU has **no mesh shaders**, and OoT draws its foliage, grass and many effects
with task/mesh programs. Upstream Hookshot ships a compute fallback for such GPUs
(issue 640), re-landed in v2.0.7 — the same approach our
[oot-bc250](https://github.com/lethevimlet/oot-bc250) emulator fork validated on this
hardware. This fork exists to:

1. Verify and report the fallback's behavior on GFX1013/RADV, where it matters most.
2. Port the lessons learned from the emulator fork back into the native port — starting
   with the empty-draw skip described below.
3. Carry any BC-250-specific deltas that don't belong upstream.

## BC-250 status (verified on hardware)

- The game builds from the `.nup` with the official Linux AppImage builder
  (`v2.0.7`, ~35 minutes on this 8-core console) and **boots to a steady 60 fps** on the
  title/intro path with the mesh fallback active:
  `vk: no mesh shaders: task and mesh programs run as compute shaders (issue 640 fallback)`.
- Requirements met on this box: Vulkan 1.4 via Mesa RADV (1.3 required), `bufferDeviceAddress`
  (the fallback needs it), AVX2 CPU, ~10 GB of RAM free for a run, ~60 GB free disk to build.
- Saves live in `~/.local/share/hookshot`, separate from any emulator's saves.

### First launch: a long black screen is normal

The **first start stays black for several minutes** while the shaders are translated and
the driver compiles the pipelines (measured here: ~100 s to translate ~39k shader programs,
plus per-pipeline compiles on top). The window is up and the game is running the whole
time — check the log (`hookshot_log.txt` in the game folder) rather than restarting. Later
starts are much faster, and the first visit to each new area carries a one-time hitch for
the same reason. This is the single most common "it's broken" report — it isn't; wait it
out the first time.

### Installing as a Steam shortcut (Gaming Mode)

The game folder's `hookshot` launcher expects a desktop; for Big Picture, point a
non-Steam shortcut at the game binary with the update prompt disabled, e.g. a small
wrapper:

```sh
#!/usr/bin/env bash
export HOOKSHOT_NO_UPDATE_CHECK=1
exec /path/to/Games/Hookshot/bin/hookshot
```

Add it to Steam (Steam must be stopped while `shortcuts.vdf` is edited), and copy the
artwork of your existing OoT tile to the new shortcut's app id in
`~/.local/share/Steam/userdata/<uid>/config/grid/` if you want it to look the same.

## Changes on top of upstream

None merged yet — this fork currently tracks upstream `main` unchanged except for this
readme, while the fallback is being validated on hardware. The first planned delta, from
a measured finding in our emulator fork of the same game:

- **Skip empty mesh draws before the fallback's fixed overhead.** In the emulator, ~12% of
  OoT's emulated mesh draws arrive culled to zero workgroups (the game culls on the GPU),
  and skipping them *before* any staging, pass break or barrier work was a verified win.
  Hookshot's fallback still pays `end_pass`, two whole-buffer fills and two full barriers
  before discovering a dispatch is empty (`gpu/vk/vk_draw.cpp`, `fallback_record`) — the
  same skip should apply, with the game issuing the same draws natively.

Anything merged here gets validated on the console first (frame times, visuals, memory),
and fixes of general interest go upstream as pull requests.

## Updating from upstream

```sh
git fetch upstream
git merge upstream/main
```

The game itself updates with the Hookshot AppImage (`--update <game folder>`); the build
is incremental and keeps saves and settings.

## License

GPL-3.0-or-later, as Hookshot. See `LICENSE` and upstream's readme for the full credits.
