# Flutter engine binaries for Linux x64

This repo builds Flutter engine binaries (`libflutter_engine.so`) for Linux x64,
with fontconfig support, for use with custom embedders.

# 📦 Downloads

See [Releases](https://github.com/ardera/flutter_embedded/releases).

The release contains one archive per engine variant:

| asset                       | variant         | runtime            | optimization                 | debug symbols                  |
| --------------------------- | --------------- | ------------------ | ---------------------------- | ------------------------------ |
| `engine.release-fast.tar.xz`  | `release-fast`  | release            | speed                        | yes, in `libflutter_engine.so` |
| `engine.release-small.tar.xz` | `release-small` | release            | size (`--optimize-for-size`) | no, stripped                   |
| `engine.debug.tar.xz`         | `debug`         | debug, unoptimized | none                         | yes, in `libflutter_engine.so` |

Each engine archive contains only `libflutter_engine.so`.

The shared data is in `common.tar.xz`:

- `flutter_embedder.h`
- `gen_snapshot` (release build, shared by all engines)
- `icudtl.dat`
- `flutter.version`
- `dart-sdk.version`

# 🛠️ Build Config

From `flutter/engine/src`, the engine is configured with:

```sh
$ ./flutter/tools/gn \
  --runtime-mode <release / debug> \
  [--optimize-for-size] \
  [--unoptimized] \
  --target-os linux \
  --linux-cpu x64 \
  --target-dir build \
  --embedder-for-target \
  --disable-desktop-embeddings \
  --no-build-glfw-shell \
  --no-build-embedder-examples \
  --no-enable-unittests \
  --enable-minimal-linux \
  --enable-fontconfig \
  --no-goma \
  [--stripped / --no-stripped]
```

The variants are:

| variant         | flags                                                   |
| --------------- | ------------------------------------------------------- |
| `release-fast`  | `--runtime-mode release --no-stripped`                  |
| `release-small` | `--runtime-mode release --optimize-for-size --stripped` |
| `debug`         | `--runtime-mode debug --unoptimized --no-stripped`      |

`--enable-fontconfig` sets both `skia_use_fontconfig` and
`flutter_use_fontconfig`. This makes the engine use
`SkFontMgr_New_FontConfig` instead of scanning `/usr/share/fonts` directly, and
links `libfontconfig.so`.

Then build the targets:

```sh
ninja -C out/build libflutter_engine.so gen_snapshot flutter_embedder.h icudtl.dat
```

The engine is built against the Debian sid sysroot provided by the engine build
scripts. Only the `amd64` sysroot is needed.

## Rpath

`libflutter_engine.so` is patched with `patchelf` so it can load shared
libraries placed next to it:

## Debug symbols

`release-fast` and `debug` keep debug symbols in `libflutter_engine.so`.
`release-small` is stripped.

The engine patches make every build emit `-ggdb -fdebug-default-version=4`
regardless of the selected symbol level. The final strip is controlled by
`--stripped` / `--no-stripped`.

# 🩹 Patches

Only three engine patches are applied:

- `0002-always-include-debug-symbols.patch` — always emit debug symbols.
- `0007-don-t-use-vulkan-xcb-wayland-wsi-integrations.patch` — disable Vulkan
  XCB/Wayland WSI.
- `0009-disable-angle-x11-wayland-wsi-for-linux-target-builds.patch` — disable
  ANGLE X11/Wayland WSI for Linux target builds.

The X11/Wayland WSI integrations are not needed for a custom embedder: the
embedder owns the window and the `VkSurfaceKHR`, and the engine only renders to
it. Disabling them avoids X11/XCB/Wayland build and runtime dependencies and
avoids GN failures caused by undefined Wayland configuration.

# 🖋️ Fonts / fontconfig

The engine links `libfontconfig.so.1`. The target system needs fontconfig and at
least one installed font:

```sh
sudo apt install fontconfig fonts-dejavu-core
sudo fc-cache -f
```

Verify the setup:

```sh
fc-match sans
ldd libflutter_engine.so
readelf -d libflutter_engine.so | grep fontconfig
```

The engine's default font families are:

```text
Ubuntu, Adwaita Sans, Cantarell, DejaVu Sans, Liberation Sans, Arial
```

At least one of these (or a fontconfig alias to one) should resolve.

# ✅ Verifying the build

```sh
# fontconfig dependency
readelf -d libflutter_engine.so | grep fontconfig

# symbols
file libflutter_engine.so
readelf -S libflutter_engine.so | grep '\.debug_'
```

Expected:

- `release-fast`: `with debug_info, not stripped`, `.debug_*` sections present
- `release-small`: `stripped`, no `.debug_*` sections
- `debug`: `with debug_info, not stripped`, `.debug_*` sections present
