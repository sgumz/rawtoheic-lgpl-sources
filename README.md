# LGPL sources used by the Android app "RAW to HEIC"

The Android app **RAW to HEIC: Photo Converter** (`com.sgumz.rawtoheic`) uses two libraries licensed under the
GNU Lesser General Public License v3.0. As the LGPL requires, this repository provides their complete source, the
changes made to them, and how to build and replace them. It contains only these two libraries; the rest of the app
is not covered by the LGPL and is not published here.

| Library | Version | Upstream | Licence |
|---|---|---|---|
| libheif | 1.19.7 at commit `4a3f74bc593ebfc29becc1ed5dd0a61cc66d40e1`, modified (see below) | https://github.com/strukturag/libheif | LGPL-3.0 (`libheif/COPYING`) |
| libde265 | 1.0.19, unmodified | https://github.com/strukturag/libde265 | LGPL-3.0 (`libde265/COPYING`) |

Upstream archive checksums (SHA-256):
- libheif `4a3f74bc….tar.gz`: `b497316d557b8a5683066b446dc0df500a6d19da36ecf03753f40f22ac4c05d3`
- libde265 `libde265-1.0.19.tar.gz`: `bb19a0b485d2643e0eeb7e91f3ab32d1ad617e7c487dbedc91214ca3dbd8d7eb`

Tests, fuzzers, documentation, examples and sample files were removed from both source trees to keep them small;
no library code was removed, and every licence file is kept.

## Changes to libheif

`libheif/` is the exact tree that ships in the app, with these patches **already applied**. They are also in
`patches/`, in the order applied, for reference:

1. `00-libheif_pr1503.patch`: the ISO 21496-1 gain-map API (libheif PR #1503), as applied by libultrahdr.
2. `libheif-tmap-brand-only-with-tmap-item.patch`: writes the `tmap` compatible brand only for files with a `tmap` item.
3. `libheif-add-gain-map-and-auxiliary-items.patch`: adds `heif_context_add_gain_map_item` and
   `heif_context_add_auxiliary_image_item` and makes the item-ID allocator skip entity-group IDs.
4. `libheif-nclx-colr-before-icc.patch`: associates the nclx `colr` box before the ICC one.

libde265 is unmodified.

## How the app uses them

Both are built as shared libraries and packaged separately in the app as `lib/<abi>/libheif.so` and
`lib/<abi>/libde265.so`, so they can be replaced.

## Building for Android

Tools: Android NDK r30 (`30.0.16248370`), CMake 4.1.2. ABIs: `arm64-v8a`, `x86_64`. Minimum API 31. Linker flag for
16 KB pages: `-Wl,-z,max-page-size=16384`.

```sh
# libde265
cmake -S libde265 -B build/de265-arm64 -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-31 -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_SHARED_LINKER_FLAGS="-Wl,-z,max-page-size=16384" \
  -DBUILD_SHARED_LIBS=ON -DENABLE_SDL=OFF -DENABLE_DECODER=OFF -DENABLE_ENCODER=OFF -DENABLE_SHERLOCK265=OFF
cmake --build build/de265-arm64
cmake --install build/de265-arm64 --prefix build/de265-arm64/install

# libheif (point it at the libde265 installed above)
cmake -S libheif -B build/heif-arm64 -G Ninja \
  -DCMAKE_TOOLCHAIN_FILE="$ANDROID_NDK/build/cmake/android.toolchain.cmake" \
  -DANDROID_ABI=arm64-v8a -DANDROID_PLATFORM=android-31 -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_SHARED_LINKER_FLAGS="-Wl,-z,max-page-size=16384" \
  -DBUILD_SHARED_LIBS=ON -DENABLE_PLUGIN_LOADING=OFF -DWITH_EXPERIMENTAL_GAIN_MAP=ON \
  -DWITH_LIBDE265=ON -DWITH_LIBDE265_PLUGIN=OFF \
  -DLIBDE265_INCLUDE_DIR="$PWD/build/de265-arm64/install/include" \
  -DLIBDE265_LIBRARY="$PWD/build/de265-arm64/install/lib/libde265.so" \
  -DWITH_X265=OFF -DWITH_AOM_DECODER=OFF -DWITH_AOM_ENCODER=OFF -DWITH_DAV1D=OFF -DWITH_RAV1E=OFF \
  -DWITH_SvtEnc=OFF -DWITH_KVAZAAR=OFF -DWITH_OpenH264_DECODER=OFF -DWITH_FFMPEG_DECODER=OFF \
  -DWITH_JPEG_DECODER=OFF -DWITH_JPEG_ENCODER=OFF -DWITH_OpenJPEG_DECODER=OFF -DWITH_OpenJPEG_ENCODER=OFF \
  -DWITH_OPENJPH_ENCODER=OFF -DWITH_UNCOMPRESSED_CODEC=OFF -DWITH_LIBSHARPYUV=OFF -DWITH_EXAMPLES=OFF \
  -DWITH_GDK_PIXBUF=OFF -DWITH_FUZZERS=OFF -DBUILD_TESTING=OFF
cmake --build build/heif-arm64
```

Repeat with `-DANDROID_ABI=x86_64` for that architecture.

## Using a modified library with the app

1. Obtain the app's split APKs from your device (`adb shell pm path com.sgumz.rawtoheic`, `adb pull`), or a universal APK.
2. Replace `lib/<abi>/libheif.so` and/or `lib/<abi>/libde265.so` in the APK that contains them with your build. Your
   library must stay compatible with the libheif API used by the app (the headers in `libheif/libheif/api`).
3. Align and re-sign every APK of the app with your own key (`zipalign -P 16 -f 4`, `apksigner sign`), then install them
   together (`adb install-multiple`). You must first uninstall the Play version, because your signature differs from it.

## Contact

Questions: support details are in the app under Settings → About → Support, or on https://rawtoheic.app.
