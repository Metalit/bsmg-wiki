---
prev: Testing
next: Advanced Data Storage
description: Create a simple configuration and settings menu in your Quest mod.
---

# Configs and UI

Most mods require a configuration to allow users to change the functionality of the mod. This section will guide you through
the basics of using `config-utils` to create a settings menu for your mod.

## Defining the Config

First, install `config-utils` by running `qpm dependency add config-utils` in your project directory. Make sure to restore
after adding the dependency.

To create the config, make a header file with the following contents.

```cpp
#pragma once

#include "config-utils/shared/config-utils.hpp"

// Declare the mod config with the name Config, which determines the get() function name
// This would generate getConfig(), DECLARE_CONFIG(ModCfg) would generate getModCfg()
DECLARE_CONFIG(Config) {
    // Create a config value
    // Parameters: (C++ name, type, title, default value, optional description)
    CONFIG_VALUE(SettingOne, int, "Setting Name", 0);

    // Like with custom types, you can also have regular C++ struct or class contents here
};
```

Supported types include C++ primitives (`int`, `float`, etc), `std::string`, all Unity vector types, `UnityEngine::Color`,
C++ vectors of any supported type, and C++ maps from string to any supported type. Custom JSON types can also be created
and used, which are covered in the [next page](./mod-data.md).

Next, you need to make sure to initialize your config. This is generally done in `setup`.

```cpp
#include "config.hpp"

extern "C" void setup(CModInfo* info) {
    // Uses modInfo to determine the config file, either creating it or loading the contents
    // The file will be in ModData/com.beatgames.beatsaber/Configs/, named by the mod ID
    getConfig().Init(modInfo);
}
```

::: danger
If you try to access values in your config before running `Init`, the game will crash.
:::

You can now use your config in your mod. All values are accessed with `GetValue` and updated with `SetValue`.

```cpp
#include "config.hpp"

void UpdateConfig() {
    int value = getConfig().SettingOne.GetValue();
    value += 1;
    getConfig().SettingOne.SetValue(value);
    // You can also not save the config file with SetValue(value, false),
    // then save it later with getConfig().Save()
}
```

## Creating UI

`config-utils` does not require UI, but is generally much more convenient for the user with it. To create UI, first
install BSML by running `qpm dependency add bsml` and `qpm dependency add custom-types` in your project directory. Make
sure to restore after adding the dependencies.

To make a simple mod settings page, you can create and register a single function.

```cpp
#include "bsml/shared/BSML.hpp"

void DidActivate(HMUI::ViewController* self, bool firstActivation, bool addedToHierarchy, bool screenSystemEnabling) {
    // Only create the elements once per menu
    if (!firstActivation)
        return;

    BSML::Lite::CreateText(self, "Hello World!");
}

extern "C" void late_load() {
    il2cpp_functions::Init();
    BSML::Init();
    // Parameters: (title, method, show ok/cancel buttons)
    BSML::Register::RegisterSettingsMenu("My Mod", DidActivate, false);
}
```

<!-- TODO picture -->

You can also register to the gameplay setup or main menu with other methods in `BSML::Register`.

To create UI for your config specifically, `config-utils` provides specialized functions that will create simple UI elements
for its settings. These will be labelled with the name you provided when making the config, and the description will be
a hover hint, if also specified in the config definition.

```cpp
#include "bsml/shared/BSML.hpp"
#include "config.hpp"

// addedToHierarchy and screenSystemEnabling are generally not needed
void DidActivate(HMUI::ViewController* self, bool firstActivation, bool, bool) {
    if (!firstActivation)
        return;

    auto container = BSML::Lite::CreateScrollableSettingsContainer(self);

    // Parameters: (Config setting, increment amount, min value, max value)
    AddConfigValueIncrementInt(getConfig().SettingOne, 1, 0, 10);
}
```

<!-- TODO picture -->

For creating more advanced UI, visit [its page](./bsml-ui.md).

### Config UI Functions

Below are all the functions for creating UI with `config-utils`, for supported types of values. Types not in this table
can still have UI, of course, but it will have to be created manually.

<!-- markdownlint-disable MD013 -->

| Type                   | Functions                                                                                                                                                      |
| ---------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `bool`                 | `AddConfigValueToggle`<br/>`AddConfigValueModifierButton`                                                                                                      |
| `int`                  | `AddConfigValueIncrementInt`<br/>`AddConfigValueIncrementEnum`<br/>`AddConfigValueDropdownEnum`<br/>`AddConfigValueSlider`<br/>`AddConfigValueSliderIncrement` |
| `float`                | `AddConfigValueIncrementFloat`<br/>`AddConfigValueSlider`<br/>`AddConfigValueSliderIncrement`                                                                  |
| `double`               | `AddConfigValueIncrementDouble`<br/>`AddConfigValueSlider`<br/>`AddConfigValueSliderIncrement`                                                                 |
| `std::string`          | `AddConfigValueInputString`<br/>`AddConfigValueDropdownString`                                                                                                 |
| `UnityEngine::Color`   | `AddConfigValueColorPicker`                                                                                                                                    |
| `UnityEngine::Vector2` | `AddConfigValueIncrementVector2`                                                                                                                               |
| `UnityEngine::Vector3` | `AddConfigValueIncrementVector3`                                                                                                                               |
| `UnityEngine::Vector4` | `AddConfigValueIncrementVector4`                                                                                                                               |

<!-- markdownlint-enable MD013 -->

:::tip
`AddConfigValueModifierButton` creates a toggle button in the style of the vanilla modifiers.

`AddConfigValueSliderIncrement` creates a slider with increment buttons on its sides, while `AddConfigValueSlider` only
creates the slider.

`AddConfigValueIncrementEnum` and `AddConfigValueDropdownEnum` allow for your UI to display names for options, while
storing the choice as an integer.

`AddConfigValueInputString` allows free input, while `AddConfigValueDropdownString` gives a list of options.
:::
