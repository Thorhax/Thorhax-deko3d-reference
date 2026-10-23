# Shipping a recompiled port without game data

How usf4-nx publishes a playable Switch build while publishing nothing of the game: the NRO is recompiled code, and
everything made from the game (its files, the translated shaders) is made on the player's own computer from their own
disc, by a web page. This follows [sa2-nx-installer](https://github.com/ChanseyIsTheBest/sa2-nx-installer), which
built on [nfsmw-nx-installer](https://github.com/StevensND/nfsmw-nx-installer) (page structure, service worker) and
[peggle2-nx-installer](https://github.com/ChanseyIsTheBest/peggle2-nx-installer) (shaders for deko3d in the browser).
usf4-nx's page: [usf4-nx-installer](https://github.com/Thorhax/usf4-nx-installer), live at
https://thorhax.github.io/usf4-nx-installer/.

## 1. What is published and what is not

| Published | Not published, made by the page from the player's disc |
|---|---|
| The NRO, as a GitHub release asset of the port's repository | The game's files (`game_root/`) |
| The page, its WebAssembly tools, a recipe and a manifest | The translated shaders (`deko3d_glsl.bin` for Renut on deko3d; `sa2_shaders.mesp` for the native renderer before) |
| The licences of everything linked into the NRO | Shader caches (`deko3d_shaders.bin`), which the game writes on the console |

The repository never holds game data, generated code or the game's artwork; a personal icon (box art) is git-ignored
and the published NRO is built with an original icon (check the icon's bytes are inside the release NRO, and that the
personal one's are not).

## 2. How the page works

1. **Finds the game** in what the player chose: a plain `.iso` read in place in the browser (an XDVDFS reader; nothing
   is extracted to disk) or a folder with the extracted disc.
2. **Identifies the build** by the SHA-256 of `default.xex` (or of its decompressed image) in `release/manifest.json`,
   and checks the game's file count and size for a full package.
3. **Downloads the NRO** from the Pages site (the deploy workflow copies it from the port's latest release and checks
   its SHA-256 against the manifest before publishing).
4. **Makes the shaders** (section 3) and checks them against the manifest's SHA-256.
5. **Streams the zip** to disk through a service worker, so an 8 GB package never has to fit in memory. Everything runs
   in the browser; the game's files never leave the player's computer.

Two buttons: **Create usf4-nx.zip** (a first install: NRO, licences, shaders, game files) and **Create
usf4-nx-update.zip** (NRO, licences and shaders only; never the player's settings file, which the game writes at its
first start).

## 3. Shaders from the player's disc

- **A recipe says where each shader container is** in the player's files (`release/recipe-<version>.json`, written by
  the port's `tools/installer_recipe.py`): a slice of an EMZ stream of the game's archives, a slice of a game file, or
  a slice of `default.xex`'s decompressed image, with patch runs for the few bytes the game or Direct3D change at run
  time; the 15 shaders Direct3D builds itself at run time exist on no disc and the recipe carries them (about 5 KB).
  The recipe's containers are fingerprinted (the SHA-256 of their sorted SHA-256s) before the long compile.
- **For Renut on deko3d** (`lib/deko3d.js`): Renut's translator (`wasm/renut_hlsl`, built from the port's own copy of
  the translator so its game fixes are in it), DXC (`wasm/dxc_web`, v2025.1), SPIRV-Cross (`wasm/spirv_cross`,
  `22b22f56`), the GLSL conversion and passes (`lib/deko3d_glsl.js`) and the package writer (`lib/deko3d_package.js`).
  The WebAssembly modules are built in the `emscripten/emsdk` Docker image (`wasm/src/deko3d/build_wasm_deko3d.sh`).
- **Pin the result:** the build's manifest entry has the pass list, the shader list, and the package's SHA-256 and
  size; the page refuses anything else.
- **Prove the page's package before publishing:** run the page's code in Node on your own copy
  (`test/deko3d_node.mjs`), compare with your tested native package, compile every differing shader with uam on the PC
  (the page cannot), and run the release NRO on the console with exactly the page's package from a fresh state (no
  shader cache). For USF4: 148 of 874 shaders identical to the native chain's, the rest differing only through the DXC
  version, all 726 compiling with uam, and the console run right.

## 4. Browser pitfalls that were hit

- **Chrome's `DecompressionStream` rejects bytes after the end of a deflate stream** ("Junk found after end of
  compressed data"), sometimes before it has handed out the stream's last bytes. The recipe now carries each EMZ
  stream's exact compressed length (`emz_lengths`), and the page gives the decompressor exactly the stream.
- **A 64 MB download breaks on bad connections** ("Failed to fetch", reported by a player). The page retries the NRO
  in 4 MB parts with HTTP range requests (GitHub Pages answers 206), each part up to six times with growing waits,
  then checks the SHA-256; if it still fails it says what to try (connection, VPN, download blockers, antivirus web
  filters, another browser). A notice at the top of the page says the same in English, Portuguese, Spanish and
  Simplified Chinese.
- **Range requests bypass the service worker** (it keeps whole files), which is what lets the resume work.
- **A deploy that copies a fixed file list** misses new files: update the list with the page.

## 5. Releasing a version

1. Build the release NRO (for deko3d: release shaders on, Vulkan driver off, the original icon), and archive its
   unstripped ELF privately for reading crash reports.
2. Make the page's shader package from your copy, prove it (section 3), and fill the build's manifest entry.
3. Update `LICENSES.txt` to what the NRO links (for 0.8.0: deko3d, uam with its patches' MPL-2.0, plume, reNut with its
   author's permission, Mesa's nouveau code through uam; NVK and Expat removed).
4. Commit and push the port; create the GitHub release with the NRO (the release API wants a full commit SHA or a
   branch name as the target, not a short SHA).
5. Switch the page to the new manifest, recipe and licences, and push: the deploy workflow takes the latest release's
   NRO, checks it, and publishes. Check the live manifest, the NRO's SHA-256 and the new files on the live site.

## 6. Telling players how to update

When a version changes what is beside the NRO, replacing the NRO alone breaks the game. For usf4-nx 0.8.0 (the first
deko3d version) the release page and the installer page both say:

1. On the installer page, choose the disc image and press **Create usf4-nx-update.zip**.
2. Extract it into `sdmc:/switch/` (it replaces `usf4.nro` and `LICENSES.txt` and adds `deko3d_glsl.bin`).
3. Delete the files the new version no longer uses: `sa2_shaders.mesp`, `sa2_shaders.mesp.idx`,
   `cache/sa2_native_pipelines.bin`.
4. Keep `game_root/`, the saves (`usf4/`), the settings file (`usf4.toml`), `logs/` and
   `cache/vfs_index_game_root.bin`.

Also say what the first session looks like (shaders compile once: the first fight's loading screen can take several
seconds longer) and what happens to settings that no longer apply (a 540p render height draws at 720p on Renut).

## 7. Licences and credits

Every project whose code is in the NRO or the page gets a line in `THIRD_PARTY_NOTICES.md` / `NOTICES.md` and its full
licence text in the release's `LICENSES.txt`. Code used by permission rather than by licence (reNut) is credited and
the permission stated. Every change to third-party code is kept as a diff in `patches/`.
