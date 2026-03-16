---
prev: Advanced Data Storage
next: Mod APIs
description: Create advanced user interfaces with BSML.
---

# Advanced UI

In [Configuration and UI](./configs-ui.md), we covered the basics of creating a simple mod settings UI, using helper
methods from `config-utils`. This page covers in detail how to make more advanced custom UI, whether for more
complicated settings menus, or fully custom menus and screens in the game.

`BSML` is the primary way of creating fully customized UI in Beat Saber. It allows for two modes of creation, a
tag-based language that mimics Unity's object hierarchy, and an API that can be used to create objects directly from
C++ code. Either way, to start, add the package by running `qpm dependency add bsml` and `qpm dependency add custom-types`
in your project directory. Make sure to restore after adding the dependencies.

## Vanilla UI

Since modded UI almost always imitates, modifies, or expands on the base game UI, it's important to know the terms and
system Beat Saber uses. To structure its hierarchy of menus, like going from the main menu to the settings or
displaying the song info when you select one in a list, it uses a custom system on top of Unity called HMUI. It has two
important base classes: `ViewController` and `FlowCoordinator`.

A `FlowCoordinator` purely provides a position in the hierarchy, as opposed to directly holding UI
elements. It is used to hold one or more `ViewController`s, and can be presented as a child of another
`FlowCoordinator`, or gain a child itself.

A `ViewController` is generally owned by a `FlowCoordinator`, and can be presented or dismissed in one to show the UI
elements it contains (through regular Unity parenting).

For example, the `MainMenuFlowCoordinator` provides `ViewController`s for the main menu mod list, and the solo/party
buttons. When you click the solo button, it presents the `SoloFreePlayFlowCoordinator`, which provides
`ViewController`s for the player settings on the left, the song selection in the center, and the leaderboards on the right.

<!-- TODO: Presentation gif -->

The other notable term in Beat Saber UI is the modal, which displays a smaller popup menu above everything else. In the
base game, this is used in dropdown selectors and the keyboard, but mods can and do use it in many other places.

<!-- TODO: Image of modal -->

::: tip
Although Beat Saber's UI appears to be 3-dimensional, the curved effect is achieved through a custom material,
instead of actual 3D coordinates. This means that, when creating your UI layout, you can treat it as a regular 2D screen.
:::

### Registered Menus

`BSML` provides a few entry points in the vanilla menus for your mod to easily add custom UI to. The example in the
[configs page](./configs-ui.md#creating-ui) involved registering to the mod settings, but there are two other locations
also available.

```cpp
#include "bsml/shared/BSML.hpp"

void DidActivate(HMUI::ViewController* self, bool firstActivation, bool, bool) {
    if (firstActivation)
        BSML::Lite::CreateText(self, "Hello World!");
}

extern "C" void late_load() {
    // Parameters: (menu title, button text, button hover hint, method)
    BSML::Register::RegisterMainMenu("My Mod Settings", "Click Me", "Opens my mod settings", DidActivate);
    // Can also use a custom ViewController or FlowCoordinator type
    // BSML::Register::RegisterMainMenu<MyViewController*>("My Mod Settings", "Click Me", "Opens my mod settings");
    // BSML::Register::RegisterMainMenu<MyFlowCoordinator*>("Click Me", "Opens my mod settings");
}
```

<!-- TODO gif -->

```cpp
#include "bsml/shared/BSML.hpp"

void DidActivate(UnityEngine::GameObject* self, bool firstActivation) {
    if (firstActivation)
        BSML::Lite::CreateText(self, "Hello World!");
}

extern "C" void late_load() {
    // Parameters: (title, method, optional menu type)
    BSML::Register::RegisterGameplaySetupTab("My Mod", DidActivate, BSML::MenuType::All);
}
```

<!-- TODO picture -->

## Creating UI

At its core, `BSML` simply creates copies of the base game UI elements, allowing them to be modified with custom
callbacks, images, and layouts. As mentioned at the start of the page, it has two main APIs.

The [tag-based language](#xml) uses XML to write UI documents, and has the advantages of hot reload and increased readability.
Its disadvantages are that it forces you to write verbose custom types for dynamic UI, and is not quite capable enough
to write everything in XML, often requiring post-creation modification in your C++ code anyway.

The [C++ API](#bsml-lite) is called BSML Lite. Its advantages are easier beginner use, less overhead, and easier
extensibility, while its disadvantages are slower iteration time especially for significant layout changes,
and often extremely long and hard to read functions.

::: tip
You can mix XML and BSML Lite in your project as much as you want.

Custom source lists, string settings, and floating progress bars can only be created with BSML Lite. Leaderboards,
gradient text, and loading indicators can only be created with XML (using `BSML` APIs).

For simple elements that need XML, you can even use simple string literals in your code, instead of assets and custom
types. For example, if you prefer BSML Lite but want a loading spinner, you can simply write
`BSML::parse_and_construct("<loading pref-width='20' pref-height='20' preserve-aspect='true' />", parent)`.
:::

### BSML Lite

All the examples so far have used BSML Lite to create their UI. (Even the helper functions in `config-utils` use it to
create their elements.) Here's a more complicated function that creates some buttons, settings, and a modal.

```cpp
void CreateUI(UnityEngine::Transform* parent) {
    auto modal = BSML::Lite::CreateModal(parent, {0, 0}, {100, 60});
    auto layout = BSML::Lite::CreateVerticalLayoutGroup(modal);

    BSML::Lite::CreateToggle(layout, "Toggle", false, [](bool value) { logger.debug("Toggle changed to {}", value); });
    BSML::Lite::CreateIncrementSetting(
        layout,
        "Increment",
        0, // Decimals
        1, // Increment amount
        0, // Initial value
        0, // Minimum value
        10, // Maximum value
        [](float value) {
            logger.debug("Increment changed to {}", value)
        }
    );

    auto container = BSML::Lite::CreateScrollableSettingsContainer(parent);

    BSML::Lite::CreateUIButton(container, "Open Modal", [modal]() { modal->Show(); });
    BSML::Lite::CreateColorPicker(
        container,
        "Color",
        UnityEngine::Color::get_white(),
        [](UnityEngine::Color) { logger.debug("color picker finished"); }
        []() { logger.debug("color picker cancelled"); }
        [](UnityEngine::Color) { logger.debug("color picker changed"); }
    );
    BSML::Lite::CreateSliderSetting(
        container,
        "Slider",
        0.1, // Increment
        0, // Initial value
        -5, // Mininum value
        5, // Maximum value
        0.1, // Time without interaction to call callback
        [](float value) { logger.debug("slider set to {}", value); }
    );
}
```

<!-- TODO picture -->

::: tip
While BSML Lite doesn't have any sort of hot reload feature, [Quest RUE](./testing.md#quest-rue)
can be very effectively used to modify positions or debug layout issues. It's not very useful for significant hierarchy
changes or the creation of new elements, however.
:::

::: tip
Since BSML Lite gives immediate access to the actual UI objects in code, it's easy to make custom functions that do more
with them. `metacore` has a number of such utility functions in its `ui.hpp` header, including extra slider and keyboard
callbacks, icon buttons, easier value setting, and more.
:::

#### Floating Screens

To create interactable UI anywhere in 3D space, instead of in the `ViewController` system, it's a little more
complicated than just creating elements. `BSML` has two options for it: canvases and floating screens.

Canvases, made with `BSML::Lite::CreateCanvas()`, are just `GameObject`s with the required components and fields to host
UI. After creating one, you can set its size and position and use it as a parent for UI elements.

Floating screens, made with `BSML::Lite::CreateFloatingScreen(...)` are also effectively canvases, but with the inbuilt
option to have a handle to move the screen's position and rotation with your controllers, and a background.

<!-- TODO pictures -->

::: tip
Using canvases or floating screens is also the easiest way to create UI inside the gameplay scene, since it doesn't
have the menu's `ViewController` system.
:::

#### Custom Lists

There are two types of lists in BSML, with a potentially unclear distinction. A custom list is simply a list, such as
the level selection or items in a dropdown, with a variable amount of content. It has a fixed set of options to display
its content - either plain text, square images, or a song list (with custom titles, subtitles,
and images). If none of those options are acceptible to display the content your mod needs, then you need to use a
custom source list, which uses a more complicated API to control everything about the cells created.

<!-- TODO base list type pictures -->

To create a custom source list, you need to first create a custom type that inherits `HMUI::TableView::IDataSource`. See
the [custom types page](./custom-types.md) if you need an introduction to them.

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"
// Nested types, such as the interface we will be implementing, can be included by including the main type
#include "HMUI/TableView.hpp"

DECLARE_CLASS_CODEGEN_INTERFACES(MyNamespace, TableDataSource, Il2CppObject, HMUI::TableView::IDataSource*) {
    DECLARE_DEFAULT_CTOR();

    DECLARE_OVERRIDE_METHOD_MATCH(float, CellSize, &HMUI::TableView::IDataSource::CellSize);
    DECLARE_OVERRIDE_METHOD_MATCH(int, NumberOfCells, &HMUI::TableView::IDataSource::NumberOfCells);
    DECLARE_OVERRIDE_METHOD_MATCH(
        HMUI::TableCell*, CellForIdx, &HMUI::TableView::IDataSource::CellForIdx, HMUI::TableView* tableView, int idx
    );

   public:
    std::vector<int> data;
};
```

```cpp
#include "TableDataSource.hpp"

DEFINE_TYPE(MyNamespace, TableDataSource);

using namespace MyNamespace;

float TableDataSource::CellSize() {
    // Generally a hardcoded constant, representing the height of your cells
    return 8;
}

int TableDataSource::NumberOfCells() {
    return data.size();
}

HMUI::TableCell* TableDataSource::CellForIdx(HMUI::TableView* tableView, int idx) {
    // For performance, lists remove cells outside of the visible range
    // So in order to not create an endless number of cells when scrolling,
    // your mod needs to provide a unique identifier to find and reuse cells
    auto cell = tableView->DequeueReusableCellForIdentifier(MOD_ID "TableDataSource");

    // Since cells might be reused, you should only create them if none are available,
    // but set or update their contents either way outside of this null check
    if (!cell) {
        cell = // TODO
        cell->reuseIdentifier = MOD_ID "TableDataSource";
    }
    // TODO

    return cell;
}
```

<!-- TODO pictures -->

### XML

XML is the namesake of Beat Saber Markup Language, and the most common way PC mods are written, if you want to port them
or use them as a reference. To use it, you'll want to make a new `.bsml` file that your mod can access (see the
[assets guide](./mod-data.md#assets) for ways to do this).

To start your file, you'll likely want to include a schema, which can be put inside a background tab without affecting
the UI. This will provide autocomplete for tags and their properties while writing the UI.

```xml
<bg xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/RedBrumbler/Quest-BSML-Docs/gh-pages/schema.xsd">
</bg>
```

::: tip
For a proper editing experience in VSCode, add an extension for xml support, and add `*.bsml` to xml to the file
associations in settings.
:::

::: warning
The Quest BSML schema has a small number of missing tags and property options. You can find a more accurate reference
on its [docs](https://redbrumbler.github.io/Quest-BSML-Docs/#/tags).
:::

Next, go ahead and start creating your UI. The language mirrors Unity's hierarchy, so child elements of tags will be
children inside Unity as well. This example creates a simple settings UI, similar to the BSML Lite example.

```xml
<bg xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:noNamespaceSchemaLocation="https://raw.githubusercontent.com/RedBrumbler/Quest-BSML-Docs/gh-pages/schema.xsd">
  <vertical child-expand-height="false">
    <toggle-setting id="toggle" text="Toggle" on-change="ToggleChanged" />
    <color-setting text="Color" on-change="ColorChanged" on-cancel="ColorCancelled" />
    <slider-setting id="slider" text="Slider" on-change="SliderChanged" />
  </vertical>
</bg>
```

While this might be enough info for a completely static UI, you're probably wondering how to actually run code when these
settings are changed, or update their values. Of course, you could always traverse the transform hierarchy in your
code after creating the UI and use `GetComponent`, but that's not very ergonomic. To easily interact with your UI
from code, you'll need to create a [custom type](./custom-types.md).

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"
// Remember, include what you use!
#include "UnityEngine/Color.hpp"
#include "UnityEngine/GameObject.hpp"
#include "UnityEngine/MonoBehaviour.hpp"
#include "bsml/shared/BSML/Components/Settings/ToggleSetting.hpp"

// Can inherit from MonoBehaviour, System::Object, or really whatever you want
DECLARE_CLASS_CODEGEN(MyNamespace, MyCustomUI, UnityEngine::MonoBehaviour) {
    DECLARE_DEFAULT_CTOR();

    // BSML matches instance field names (created with the macro) to the ID of the XML element
    // You may have to browse around in its headers to find the type, or check what Lite returns
    DECLARE_INSTANCE_FIELD(BSML::ToggleSetting*, toggle);
    // You can also use GameObject or another component on the created object, like RectTransform
    DECLARE_INSTANCE_FIELD(UnityEngine::GameObject*, slider);
    // Some tags, such as <list> or <text-segments>, use a data field that references a list
    // Those are also done with macro-declared instance fields with matching names,
    // but need to be populated before constructing the UI instead of getting filled in after

    // Callback types match the Lite API, or can be checked in the BSML docs (linked above)
    // Again, the names must exactly match those in the XML, and must be created with the macro
    DECLARE_INSTANCE_METHOD(void, ToggleChanged, float value);
    DECLARE_INSTANCE_METHOD(void, ColorChanged, UnityEngine::Color value);
    DECLARE_INSTANCE_METHOD(void, ColorCancelled);
    DECLARE_INSTANCE_METHOD(void, SliderChanged, float value);

    // When declared exactly like this, will be automatically run after the UI is created
    DECLARE_INSTANCE_METHOD(void, PostParse);
};
```

Finally, to actually create the UI from the XML and your custom type instance, simply run `BSML::parse_and_construct`.

```cpp
#include "MyCustomUI.hpp"
#include "BSML.hpp"

void CreateUI(UnityEngine::Transform* parent) {
    // Example if reading from a file, using asset include is generally recommended though
    std::string xmlText = readfile("my_custom_ui.bsml");
    // The "host" is the custom type where fields are populated and callbacks are called
    auto host = parent->gameObject->AddComponent<MyNamespace::MyCustomUI*>();
    // You can have a completely different host and Unity parent, if you want,
    // but make sure it stays alive for any callbacks if you do
    BSML::parse_and_construct(xmlText, parent, host);
}
```

::: warning
BSML has `modal-keyboard` and `string-setting` tags, which use a custom, deprecated large modal keyboard with less
features and integration (and aesthetics) compared to the normal one.

Instead, use the BSML Lite `CreateStringSetting` API in a `PostParse` method for text inputs.
:::

#### Hot Reload

<!-- TODO add helpers and write -->
<!-- also note that this is why post parse is useful -->

::: warning
When using hot reload, the entire content of your UI is destroyed and recreated every time it changes. Make sure
to handle this, null checks for potentially removed elements, and other potential logic errors from creating
the UI multiple times, or your mod will crash the game and you won't save any time from the feature.
:::

#### Images

Like with the BSML files, to use custom images in your XML UI, you'll need to [include them in your mod](./mod-data.md#assets).
Once you have access to them, you need to add a little code to your mod to make `BSML` aware of their existence, using
the `BSML_DATACACHE` macro.

```cpp
#include "bsml/shared/BSMLDataCache.hpp"

// This will be automatically registered on dlopen, so it can go in any source file
// It must return an ArrayW<uint8_t> with the image data, and have a unique name in your mod
BSML_DATACACHE(ButtonIcon) {
    // Again, reading from a file is an example, using asset include is generally recommended
    auto data = readbytes("button_icon.png");
    return ArrayW<uint8_t>({(uint8_t*) data.data(), data.size()});
}
```

<!-- TODO double check arrayw code -->

<!-- TODO add datacache helper to metacore assets -->

Now, you can use the name you provided, combined with your mod ID with an underscore, to directly reference the
image in a BSML file.

```xml
<img src='MyModID_ButtonIcon' preserve-aspect='true' pref-width='20' />
```

#### Extra Actions

<!-- TODO go through bsml code to see if there is anything other than modal here :( -->
