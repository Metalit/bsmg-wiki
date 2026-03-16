---
prev: Il2Cpp and C++
next: Configs and UI
description: Learn how to run, test, and debug your mod.
---

# Testing

Quest mods can be much stricter than PC ones. If an exception is uncaught, or a null reference is used, the game will
immediately crash instead of simply logging an error. Therefore, you need to know how to debug these crashes, and be
extra vigilant in testing.

## Checklist

- Make sure your mod works in Solo, Party, Multiplayer, and Tutorial game modes, if it affects anything to do with level
  selection or gameplay.
- Run the game through a "soft" (or "internal") restart, as that may destroy objects that you use. You can easily do this
  by going to the vanilla settings and pressing OK.
- Test with as many other mods as you can, to reveal potential incompatibilities.
- Give your mod to other people to test, as they may do things you haven't thought of.

## Logs

Logs are a convenient way to give yourself information about what your mod is doing. In Quest mods, we use the `paperlog`
library, which allows for us to send messages to logcat and per-mod files with [fmt](https://fmt.dev/).

If you started with the template, the logger is already set up for you, using your mod ID and logging to a file in addition
to logcat. To use it, you can just make sure `main.hpp` is included and write:

```cpp
logger.info("this is a log message");
```

And whenever that code is run, the message will appear in logcat and your mod-specific file, with metadata such as the
log level, time, and mod ID. You can access the file using `qpm s log`, or view it as it comes through logcat with
`qpm s log -l`.

::: tip
The available log levels are `debug`, `info`, `warn`, `error`, and `critical`. You can filter them in the log script - run
`qpm s log -h` for its full set of options.
:::

### Formatting

As mentioned, the logger uses the formatting library `fmt`. You can read [its documentation](https://fmt.dev/) for more
details, but for a simple overview, you can place `{}` in your string wherever a value should be inserted then
provide the values as arguments in order.

```cpp
void LogMessage() {
    int x = 1;
    int y = 9;
    logger.info("x: {}, y: {}", x, y);
    // Logs "x: 1, y: 9"
}
```

::: tip
You can also use the `fmt::format` function to format a string instead of logging a message.
:::

The wrapper types provided by `beatsaber-hook`, such as `StringW`, have custom formatters. This allows them to be
used in log messages without needing to be converted to a `std::string` (for example). You can also make custom formatters
for your own types using a simple `fmt` API.

```cpp
enum class Number { Zero, One, Two };

// Provides formatting for Number
auto format_as(Number num) {
    return (int) num;
}

void LogMessage() {
    Number num = Number::Two;
    logger.info("enum value: {}", num);
    // Logs "enum value: 2"
}
```

::: warning
In some cases, `fmt` may attempt to format your type as a range, preventing `format_as` from working. If this happens,
you can fix it with the following specialization:

```cpp
template <typename Char>
struct fmt::is_range<MyType, Char> {
    static constexpr bool value = false;
};
```

:::

## Crashes

<!-- TODO -->

The majority of crashes will print out detailed info in one or both of 2 locations. The first possibility is a native
tombstone in the app's data directory, and the second is a backtrace in the global log file.

## Quest RUE

[Quest Runtime Unity Editor](https://github.com/Metalit/quest-rue/), sometimes abbreviated to QRUE, is a reimagined port
of RUE/UnityExplorer on PC. It allows you to find objects in game, list their components, tweak properties, run methods,
and more, making it an invaluable tool for figuring out the game and iterating on your mod.

After installing the QMOD on your device, you need to connect to it from the client either through entering its IP address,
or if ADB is on your PATH, plugging the device in with a cable and selecting it in the list of devices.

::: tip
You can also use [RUE on the PC version](../pc/rue.md). Since the game is the same, it can be equally as useful for some
purposes, but you obviously won't be able to have your mod running.
:::

## Emulators

Setting up an emulator is a relatively new devlopment, and as such remains a somewhat involved process. However, it can
be incredibly useful, since testing mods on the physical device only can cause many inconveniences, such as having to put
on the headset every time you make a change in QRUE.

### Installing

Utilities to setup an emulator are included in QPM. To start, run this command to install the Android SDK and emulator:

```sh
qpm emu setup --install-sdk --install-emulator
```

If you already have the SDK installed but not the emulator, you can add `--sdk-manager-path <sdk manager root>`, or set
the `ANDROID_SDK_ROOT` environment variable. If you have both, you can skip that command, but make sure to provide the argument
or environment variable when running future commands as well.

::: warning
Installing the emulator will also install the `android-33;android-desktop;x86_64` system image. You can configure the
`create` and `start` commands to use a different image, but most others either fail to work with Beat Saber or don't
have CPU architecture translation.
:::

Next, run this command to create the emulator image:

```sh
qpm emu create --avd --name <emulator name>
```

You can run `qpm emu create --help` to see additional configuration options.

::: tip
By default, the AVD (Android Virtual Device) will be created in `~/.android/avd`. To move this elsewhere, you can set the
`ANDROID_AVD_HOME` environment variable to the containing folder.
:::

Finally, you need to start the emulator to install the necessary apps.

```sh
qpm emu start --name <emulator name>
```

### Modding

::: danger
Do not share the APK file, as that would be piracy.
:::

<!-- TODO double check all the commands here -->

If you have the unmodded APK and OBB of the version you want to test already, or installed on your quest, then you can
use this command to patch it in place:

```sh
qpm emu apk patch <beat saber apk>
```

Then install the APK and copy the OBB file.

```sh
adb install <beat saber apk>
adb push <beat saber obb> /sdcard/Android/obb/com.beatgames.beatsaber/
```

Otherwise, you can use the setup tool.

First, you need to get your Oculus token. Log in to [secure.oculus.com](https://secure.oculus.com/) and open the developer
tools. Then, navigate to the "Application" tab, expand "Cookies", and select `https://secure.oculus.com`. The value of
the `oc_ac_at` cookie is your token. (It should start with `OC`.)

::: danger
Your token is a confidential piece of information. Possession of this token allows individuals to download applications,
send messages, and more, under your identity. Do not share it with anyone else.
:::

Now, run this command to download, patch, and install the APK:

```sh
qpm emu apk download --token <token> <game version> --patch --install
```

<!-- TODO probably change file hosting -->

Currently, even with the correct mods installed the game will crash without the Meta Horizon app installed. For the emulator,
you need to use an older version. You can [download it here](https://www.mediafire.com/file/t6w909bnp4vfefd/Horizon.apk-signed-aligned.apk/file),
or extract the unsigned APK from the system partition of [older OS versions](https://cocaine.trade/Quest_2_firmware).
The download is from version `51062580065800150` and has been signed.

```sh
adb install <horizon apk>
```

<!-- TODO update mbf cli -->

The other extra piece needed for the game to run on an emulator is the [AndroidHelper mod](https://github.com/kodenamekrak/beat-saber-emulator-guide/blob/main/AndroidHelper_1.40.6.qmod),
which disables some of the checks the game does for VR support on the device.

Another mod you might find useful for working on an emulator is [Emulator FPFC](https://github.com/kodenamekrak/Emulator-FPFC/).
It allows you to move with WASD + QE, click to click on things, drag to rotate the camera, type in input fields, and more.
