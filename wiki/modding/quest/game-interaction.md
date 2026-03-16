---
prev: Setup Guide
next: Decompiling
description: Learn how to hook and run methods in the game.
---

# Game Interaction

## BS-Cordl

`cordl` is a tool used in quest modding to provide easy access to the game's code and functions by generating header files
for every class in the game. The Beat Saber generated code, `bs-cordl`, will already be in your dependencies if you started
with the template, and each of its versions match a single game version.

To use it, you'll need to know the namespace and name of the class you want to use, then write `#include "namespace/name.hpp"`.
For example, to use methods on the `StandardLevelDetailViewController` class in `GlobalNamespace`, you would write
`#include "GlobalNamespace/StandardLevelDetailViewController.hpp"` at the top of the file.

To browse and find the classes and methods in the game, it is possible to browse `bs-cordl`'s header files. However, much
better ways exist, which are covered in the [next page](./decompiling.md).

::: warning
While your IDE may be able to autocomplete methods and fields even without the files included, it will cause errors when
you build your mod. The rule is, include what you use!
:::

::: tip
`bs-cordl` does not generate C++ operators, such as `+` for vectors, instead requiring you to use
`Vector2::op_Addition(a, b)`. This is to avoid making calls to C# functions without being aware of it, but it can be
very annoying when doing more significant amounts of math.

To make the operators more convenient, you can add `metacore` to your dependencies, then simply write `#include "metacore/shared/operators.hpp"`
in your file. Now, you can write `a + b` for two vectors in that file, like you could in C#.
:::

### Wrapper Types

While `bs-cordl` allows us to easily access fields and run methods on objects, a significant amount of convenience available
in C# can be missing for some types. For commonly used C# types, C++ wrappers are provided that allow for more natural
syntax and use in C++ are available: `StringW`, `ArrayW`, and `ListW`.

`StringW` mainly makes conversions between C++ strings and C# strings easy. To convert to a `std::string`, you can use
`static_cast<std::string>(myStringW)`. To convert to a C# string, you can use `StringW(myStdString)`.

Both of those conversions can be done implicitly as well. Since `bs-cordl` generates StringW in its methods, this
means you can pass C++ strings or string literals into them, and they will automatically be converted to C# strings
for you.

`ArrayW` and `ListW` provide C++ iterators, size access, indexing, and for lists, addition and removal. C# methods can also
be run with `->` (C++ methods such as `size()` would be run with `.size()`). These have less trivial conversions to and
from `std::vector`, but can be constructed from one, and converted to one using their `begin()` and `end()` iterators.

The `ConstString` type is also available. It can be used like a `StringW`, but allocates its storage statically at compile
time, for slightly better performance in critical situations. You can use it in a function with the `static` keyword.

```cpp
void PerformanceCritical() {
    static ConstString name("MyObject");  // Only allocated once!
    UnityEngine::GameObject::New_ctor(name);
}
```

::: tip
The `metacore` package also provides `DictionaryW` and `ConstArray` types in `metacore/shared/il2cpp.hpp`.
:::

## Hooking

Hooking is core to modding. Fundamentally, it provides us a way of running our own code whenever the game runs a method.
If you're coming from PC, you can think of it as an equivalent of Harmony Patches. `beatsaber-hook` provides a simple way
of hooking methods and static functions.

> In computer programming, the term hooking covers a range of techniques used to alter or augment the behavior of an
> operating system, applications, or other software components by intercepting function calls or messages or events
> passed between software components. Code that handles such intercepted events is called a hook.
> [Wikipedia](https://en.wikipedia.org/wiki/Hooking#:~:text=In%20computer%20programming%2C%20the%20term,events%20passed%20between%20software%20components.&text=Hooking%20can%20also%20be%20used%20by%20malicious%20code.)

In this example, we will hook onto the initialization of the level screen and change the text on the play button to
something funny.

The level screen runs the event `DidActivate` when it is fully initialized. This is useful for us because we can hook
this event and add our own functionality.

First, create your hook using the `MAKE_HOOK_MATCH` macro:

```cpp
// These files are included from the bs-cordl dependency, as covered earlier.
#include "GlobalNamespace/StandardLevelDetailView.hpp"
#include "GlobalNamespace/StandardLevelDetailViewController.hpp"
#include "UnityEngine/UI/Button.hpp"
#include "UnityEngine/GameObject.hpp"
#include "HMUI/CurvedTextMeshPro.hpp"

// Here, we create a hook struct named StandardLevelDetailViewController_DidActivate
// targeting the method "StandardLevelDetailViewController::DidActivate".
// The method takes the following arguments:
// bool firstActivation, bool addedToHierarchy, bool screenSystemEnabling
// and returns void.

// The name can be any valid, unique C++ identifier. Naming it after the class and method is just a convention.

// Macro format: MAKE_HOOK_MATCH(hook name, &hooked method, method return type, self pointer, arguments...) {
//     Code for before the method runs
//     HookName(self, arguments...);
//     Code for after the method runs
// }

MAKE_HOOK_MATCH(
    StandardLevelDetailViewController_DidActivate,
    &GlobalNamespace::StandardLevelDetailViewController::DidActivate,
    void,
    GlobalNamespace::StandardLevelDetailViewController* self,
    bool firstActivation,
    bool addedToHierarchy,
    bool screenSystemEnabling
) {
    // Run the original method before our code, often abbreviated to the "orig".
    // This could be after our code or in the middle if we wanted to change argument values
    // or do something immediately before it runs, or it could even be omitted entirely
    // to prevent the default method from running at all. (See the Orig Hooks section.)
    StandardLevelDetailViewController_DidActivate(self, firstActivation, addedToHierarchy, screenSystemEnabling);

    // Get the actionButton text object by accessing the actionButton field and some standard Unity methods.
    // Note that auto can be used instead of declaring the full type in many cases.
    GlobalNamespace::StandardLevelDetailView* standardLevelDetailView = self->_standardLevelDetailView;
    UnityEngine::UI::Button* actionButton = standardLevelDetailView->actionButton;
    UnityEngine::GameObject* gameObject = actionButton->gameObject;
    HMUI::CurvedTextMeshPro* actionButtonText = gameObject->GetComponentInChildren<HMUI::CurvedTextMeshPro*>();

    // Set the text to "Skill Issue"
    actionButtonText->text = "Skill Issue";
}
```

Next, you have to install your hook. Usually, hooks are installed in `load()` or `late_load()` in `main.cpp`:

```cpp
extern "C" void late_load() {
    il2cpp_functions::Init();

    logger.info("Installing hooks...");
    INSTALL_HOOK(logger, StandardLevelDetailViewController_DidActivate);
    logger.info("Installed all hooks!");
}
```

::: tip
The first method parameter to a hook is almost always a self pointer - a pointer to an instance of the class that the
hooked method belongs to. However, when hooking static methods, the self pointer should be omitted, as there is
no instance associated with the call.
:::

### Auto Hooks

<!-- TODO actually make hooking package -->

To avoid having to remember to install every hook you make, and put them in the same file, a simple wrapper dependency can
be added to your project by running `qpm dependency add meta-hooks`. Make sure to restore after adding the dependency.

To use it, include its header file in the file you want to create your hook, then simply replace the `MAKE_HOOK_MATCH` macro
with `MAKE_AUTO_HOOK_MATCH`.

```cpp
// Include the header file from the dependency.
// You still need to include everything you use from bs-cordl.
#include "meta-hooks/shared/hooks.hpp"

// Change MAKE_HOOK_MATCH to MAKE_AUTO_HOOK_MATCH.
MAKE_AUTO_HOOK_MATCH(
    StandardLevelDetailViewController_DidActivate,
    &GlobalNamespace::StandardLevelDetailViewController::DidActivate,
    void,
    GlobalNamespace::StandardLevelDetailViewController* self,
    bool firstActivation,
    bool addedToHierarchy,
    bool screenSystemEnabling
) {
    // Unchanged from the example.
}
```

Next, in `main.cpp`, you'll also need to include the header file, and then replace your individual hook installs in `load`
or `late_load` with a single function.

```cpp
#include "meta-hooks/shared/hooks.hpp"

extern "C" void late_load() {
    il2cpp_functions::Init();

    logger.info("Installing hooks...");
    Hooks::Install();
    logger.info("Installed all hooks!");
}
```

Now, any hook you make with `MAKE_AUTO_HOOK_MATCH` will automatically be installed.

### Orig Hooks

While browsing other mods' source code, you may encounter `MAKE_AUTO_ORIG_HOOK_MATCH` or `INSTALL_HOOK_ORIG`. These are
called orig hooks, and they are used whenever the hook will always replace the original method instead of running it.

You aren't required to use these when overriding a method with a hook, but they are useful to help ensure multiple mods
aren't trying to override the same method.
