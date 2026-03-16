---
prev: Configs and UI
next: Advanced UI
description: Store whatever data you want, local to your mod.
---

# Advanced Data Storage

## Rapidjson-Macros

::: warning
This library may be deprecated in future game versions, in favor of [reflect-cpp](https://github.com/getml/reflect-cpp).
It will remain available for use on QPM.
:::

For storing more complicated values and structures, the `rapidjson-macros` package allows you to easily read and write
from JSON, and can integrated directly into `config-utils` configs if desired.

To start, you can define an object with the `DECLARE_JSON_STRUCT` and `VALUE` macros.

```cpp
#include "rapidjson-macros/shared/macros.hpp"

DECLARE_JSON_STRUCT(ControllerButton) {
    VALUE(int, Button);
    VALUE(int, Controller);
};
```

This creates a struct that represents a JSON object, with two required values, `Button` and `Controller`. It can be
converted to and from strings with `ReadFromString` and `WriteToString`.

```cpp
void ConvertJSON() {
    std::string json = "{\"Button\": 0, \"Controller\": 0}";
    ControllerButton value = ReadFromString<ControllerButton>(json);
    value.Button = 1;
    json = WriteToString(value);
}
```

When reading from a string, if required fields are missing or of the incorrect type, a `JSONException` will be thrown,
with a message describing the error and its location in the parsed object. You can also use the `VALUE_DEFAULT` or
`VALUE_OPTIONAL` macros to make a field non-required, so it will not throw an exception.

::: tip
While you can use normal C++ (or C) methods of reading and writing files, `beatsaber-hook` also provides the simple
functions `readfile`, `writefile`, `fileexists`, and a few more in `beatsaber-hook/shared/utils/utils-functions.h`.
:::

Types can also be nested, allowing for more complicated JSON structures and reuse.

```cpp
#include "rapidjson-macros/shared/macros.hpp"

DECLARE_JSON_STRUCT(ControllerButton) {
    VALUE(int, Button);
    VALUE(int, Controller);
};

DECLARE_JSON_STRUCT(ButtonSettings) {
    VALUE(ControllerButton, MainActionButton);
    VALUE(std::vector<ControllerButton>, OtherActionButtons);
};

DECLARE_JSON_STRUCT(ExampleConfig) {
    VALUE(int, Version);
    VALUE(ButtonSettings, Buttons);
    VALUE(std::string, OtherSetting);
};
```

::: tip
There are a number of other value macros in the `macros.hpp` header, with conveniences such as custom JSON names, and
vectors and maps without having to type out `std::vector`.

There are also `TypeOptions` and `UnparsedJSON` classes available, which can be used in the same ways as types created
with `DECLARE_JSON_STRUCT`, but have custom behavior.
:::

## File Storage

While mods can access most non-restricted folders and files on the Quest, the convention for storing configs and data
files is to use the `ModData` folder. This is used over the standard application directory because files in it will
persist across game reinstalls for updates (allowing for mods such as PlayerDataKeeper), and are also accessible from
file managers.

`config-utils`, as covered in the [last page](./configs-ui.md), will handle reading and writing to the config file path
for you. If you need to access the path yourself anyway, you can use `Configuration::getConfigFilePath` with
your mod info.

To access the data directory, the function `getDataDir` will use your mod info or mod ID to generate the path for your mod.
It will not create the directory for you, so you may need to check if it exists.

```cpp
#include "beatsaber-hook/shared/config/config-utils.hpp"

std::string GetConfigPath() {
    return Configuration::getConfigFilePath(modInfo);
}

std::string GetDataPath() {
    return getDataDir(modInfo);
}
```

## Copy Extensions

<!-- TODO -->

## Assets

Some mods will need to provide constant data that isn't best expressed through code, such as images, BSML files, or
unity asset bundles. There are two ways of shipping arbitrary files with your mod - embedding them into the C++ library
itself, or placing them inside the QMOD file.

### Asset Include

For many assets, it's easiest to embed them into the C++ library. This is specifically best if you don't need them as an
actual file, but just their data, and they aren't too large.

To do this, add `metacore` as a dependency, and restore. Then, inside your `CMakeLists.txt`, add this line:

```cmake
include(extern/includes/metacore/shared/assets.cmake)
```

Finally, create a directory named `assets` in your project folder, and place any files you want in that directory.
Every time you build your mod, a file named `assets.hpp` will be automatically generated or updated in your `include`
folder, based on the contents of the `assets` folder.

::: tip
You can also make subdirectories inside of your `assets` folder, and the generated C++ file will have matching
namespaces for each file.
:::

Now, you can include the generated `assets.hpp` file in any other file, and use the asset names in the `IncludedAssets`
namespace. For example, if you have the image `button.png` in your assets folder, you could use it with this code.

```cpp
#include "assets.hpp"

UnityEngine::Sprite* GetButtonImage() {
    // Can access size or raw data, or convert to an ArrayW, std::string, or std::span
    logger.info("button image size: {}", IncludedAssets::button_png.size());
    // Will only be defined if BSML is a dependency
    // Does not cache the sprite - make sure to cache it if used more than once
    return PNG_SPRITE(button);
}
```

::: warning
Non-alphanumeric characters in file names and directories will be replaced with underscores, so make sure no asset files
or folders are distinguished with only those characters, to avoid build errors from duplicate symbols.
:::

### File Copies

Alternatively, files can be included in the QMOD file, and copied into a desired location by the mod installer. (QMODs
are just zip files.) This is generally only better if your mod requires them to be stored in the filesystem.

To do this, you'll need to modify your `mod.template.json` for each file you want to copy.

<!-- TODO -->
