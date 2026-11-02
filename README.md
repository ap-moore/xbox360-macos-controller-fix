# Wired Xbox 360 controllers on macOS — Stardew Valley SDL2 fix

Fix inverted vertical-stick movement when using Microsoft wired Xbox 360 controllers in Stardew Valley on macOS. This repository provides a patch for **SDL2 2.30.4**, the library Stardew Valley uses for controller input.

The patched library supports two controllers controlling separate players, with correct stick directions and Player 1 / Player 2 LED indicators. No SDL environment-variable configuration is required.

**Tested hardware:** USB vendor ID `045e`, product ID `028e` (Microsoft wired Xbox 360 controller). Tested on an Apple Silicon Mac with the Steam version of Stardew Valley. Other hardware and games are untested.

## Requirements

- macOS and Apple Command Line Tools (`xcode-select --install` if needed)
- Git
- CMake (`brew install cmake` if using Homebrew)
- Steam installation of Stardew Valley for macOS
- Two Microsoft wired Xbox 360 controllers (`045e:028e`)

The instructions below build an **x86_64** SDL2 library for Stardew Valley's Intel macOS executable, which runs through Rosetta on Apple Silicon.

## Build from source

From a working directory of your choice:

```bash
git clone https://github.com/ap-moore/xbox360-macos-controller-fix.git
git clone --depth 1 --branch release-2.30.4 https://github.com/libsdl-org/SDL.git SDL
cd SDL

git apply --check ../xbox360-macos-controller-fix/patches/xbox360-controller-fix.patch
git apply ../xbox360-macos-controller-fix/patches/xbox360-controller-fix.patch

cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DSDL_SHARED=ON \
  -DSDL_STATIC=OFF \
  -DHAVE_FRAMEWORK_GAMECONTROLLER=FALSE

cmake --build build --parallel 4
file build/libSDL2-2.0.0.dylib
cd ..
```

The output is `SDL/build/libSDL2-2.0.0.dylib`. The `file` command should report an `x86_64` library. Keep `HAVE_FRAMEWORK_GAMECONTROLLER=FALSE` to disable Apple's GameController/MFi backend; this is part of the tested configuration.

## Install in Stardew Valley

**Quit Stardew Valley and Steam completely first.** From the directory containing the `SDL` checkout, back up the game's original library and install the patched one. Adjust `GAME` if your Steam library is installed elsewhere.

Keep the original backup when reinstalling the patch; do not overwrite it with a patched library.

```bash
GAME="$HOME/Library/Application Support/Steam/steamapps/common/Stardew Valley/Contents/MacOS"

cp -n "$GAME/libSDL2-2.0.0.dylib" \
  "$GAME/libSDL2-2.0.0.dylib.backup"

cp SDL/build/libSDL2-2.0.0.dylib \
  "$GAME/libSDL2-2.0.0.dylib"
```

Connect both controllers, restart Steam, and launch Stardew Valley. Start local split-screen co-op and check that:

- Each controller controls one separate character.
- Moving either stick up or down produces the expected direction.
- The LEDs indicate Player 1 and Player 2.

Steam updates or game-file verification may replace the patched library.

## Restore the original library

Quit Steam and Stardew Valley. If you created the backup above:

```bash
GAME="$HOME/Library/Application Support/Steam/steamapps/common/Stardew Valley/Contents/MacOS"
cp "$GAME/libSDL2-2.0.0.dylib.backup" \
  "$GAME/libSDL2-2.0.0.dylib"
```

Alternatively, use **Steam → Stardew Valley → Properties → Installed Files → Verify integrity of game files** to restore the library distributed with the game.

## How it works

The build disables Apple's GameController/MFi backend and retains SDL2's IOKit and HIDAPI controller drivers. The patch corrects vertical input for Microsoft wired Xbox 360 controllers (`045e:028e`) in both drivers:

- **HIDAPI:** corrects the left and right sticks' Y-axis orientation on macOS.
- **IOKit:** corrects the left-stick Y-axis when this driver handles input.

The axis corrections are restricted to this vendor and product ID. Other controller models retain SDL2's existing axis behavior.

## Compatibility and limitations

- Verified: two wired Microsoft Xbox 360 controllers (`045e:028e`), Apple Silicon Mac, Steam Stardew Valley, SDL2 2.30.4 built for `x86_64`.
- Not verified: other controller models, SDL releases, games, or macOS versions.
- This replaces a game-bundled dynamic library; keep a backup and expect Steam verification or updates to restore the original.

## Credits and licensing

**Thanks to [Tom Grimwood](https://github.com/TomGrimwood)** for his earlier Xbox 360 controller work in his [SDL fork](https://github.com/TomGrimwood/SDL). His Xbox 360 Y-axis work informed this project. The build instructions use the official SDL2 source. See Tom's repository for his implementation and any applicable attribution or licensing notices.

[SDL2](https://github.com/libsdl-org/SDL) is developed by the SDL project and distributed under the zlib license. This project is not affiliated with SDL, Steam, or ConcernedApe.
