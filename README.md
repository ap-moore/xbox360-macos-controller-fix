# Xbox 360 Controllers on macOS — SDL2 Fix

Fix for using multiple wired Xbox 360 controllers simultaneously on macOS, including inverted analog-stick controls.

## Build prerequisites

Building the patched SDL2 library requires:

- Git
- Apple Command Line Tools (`xcode-select --install`)
- CMake (`brew install cmake` if using Homebrew)

Homebrew is only needed if CMake is not already installed.

## The problem

On an Apple Silicon Mac running macOS 27, two wired Xbox 360 controllers were connected over USB, but Stardew Valley only recognized one at a time.

Both controllers appeared in macOS's USB device information, but the last controller connected would take precedence.

## The solution

Two changes were required:

1. Disable SDL2's MFi backend so SDL uses IOKit.
2. Patch SDL2's IOKit backend to correct the inverted left-stick Y-axis.

Tested successfully with two wired Xbox 360 controllers in Stardew Valley.

## Step 1: Enable SDL's IOKit backend

Run:

```bash
launchctl setenv SDL_JOYSTICK_MFI 0
```

Completely quit and restart Steam.

Make sure:
- Steam Input is disabled for Stardew Valley.
- Steam launch options are blank.
- No custom `SDL_GAMECONTROLLERCONFIG` mapping is active.

At this point, both controllers should be detected, although vertical movement may be inverted.

## Step 2: Build the patched SDL2 library

Clone the custom SDL repository:

```bash
git clone --branch SDL2 https://github.com/TomGrimwood/SDL.git SDL-custom
cd SDL-custom
git checkout 4f623319d
```

Apply the patch from this repository:

```bash
git apply /path/to/xbox360-macos-controller-fix/patches/xbox360-iokit-y-axis.patch
```

Configure and build SDL2:

```bash
cmake -S . -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_OSX_ARCHITECTURES=x86_64 \
  -DCMAKE_OSX_DEPLOYMENT_TARGET=11.0 \
  -DSDL_SHARED=ON \
  -DSDL_STATIC=OFF \
  -DSDL_TESTS=OFF

cmake --build build -j 4
```

The patch specifically targets wired Xbox 360 controllers with vendor ID `0x045e` and product ID `0x028e`.

## Step 3: Install in Stardew Valley

Quit Stardew Valley and Steam first.

```bash
GAME="$HOME/Library/Application Support/Steam/steamapps/common/Stardew Valley/Contents/MacOS"

cp "$GAME/libSDL2-2.0.0.dylib" \
   "$GAME/libSDL2-2.0.0.dylib.backup"

cp build/libSDL2-2.0.dylib \
   "$GAME/libSDL2-2.0.0.dylib"
```

Restart Steam and launch Stardew Valley.

Both controllers should now work simultaneously with correctly oriented analog sticks.

## Optional: Persist the environment variable

The `launchctl setenv` command may not survive a logout or restart.

To restore the setting automatically at login, create:

`~/Library/LaunchAgents/com.local.stardew-controller.plist`

With the following contents:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.local.stardew-controller</string>
    <key>ProgramArguments</key>
    <array>
        <string>/bin/launchctl</string>
        <string>setenv</string>
        <string>SDL_JOYSTICK_MFI</string>
        <string>0</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
</dict>
</plist>
```

Then load it:

```bash
launchctl bootstrap "gui/$(id -u)" \
  "$HOME/Library/LaunchAgents/com.local.stardew-controller.plist"
```

### How the LaunchAgent works

The LaunchAgent runs `launchctl setenv SDL_JOYSTICK_MFI 0` once when you log in.

- It does not install a driver or run continuously.
- It does not require administrator privileges.
- **Steam must launch after the LaunchAgent runs** to inherit the setting.
- The LaunchAgent is optional. You can run the command manually instead.

A ready-to-use configuration is included at
[`launchagents/com.local.stardew-controller.plist`](launchagents/com.local.stardew-controller.plist).

To install it, copy the file into `~/Library/LaunchAgents/` and load it
using the `launchctl bootstrap` command shown above.

## Reverting the changes

Restore the original SDL2 library:

```bash
GAME="$HOME/Library/Application Support/Steam/steamapps/common/Stardew Valley/Contents/MacOS"

cp "$GAME/libSDL2-2.0.0.dylib.backup" \
   "$GAME/libSDL2-2.0.0.dylib"
```

To restore the default SDL controller backend:

```bash
launchctl unsetenv SDL_JOYSTICK_MFI
```

If you installed the optional LaunchAgent, unload and remove it first:

```bash
launchctl bootout "gui/$(id -u)" \
  "$HOME/Library/LaunchAgents/com.local.stardew-controller.plist"

rm "$HOME/Library/LaunchAgents/com.local.stardew-controller.plist"
```

## Technical explanation

Apple's GameController framework appeared to consolidate two identical wired Xbox 360 controllers into one logical device.

Disabling SDL's MFi backend allowed its IOKit backend to recognize both independently.

However, the IOKit backend reported the left stick's vertical axis in the opposite direction expected by Stardew Valley.

The included patch modifies `SDL_iokitjoystick.c` and `SDL_iokitjoystick_c.h` to invert axis 1 only for wired Xbox 360 controllers (`045e:028e`).

No third-party controller driver or changes to macOS security protections are required.

## Compatibility

Verified on:
- Apple Silicon Mac
- macOS 27
- Two wired Xbox 360 controllers (`045e:028e`)
- Stardew Valley using SDL2 (x86_64)

Other macOS versions, games, and controller models have not been tested.
