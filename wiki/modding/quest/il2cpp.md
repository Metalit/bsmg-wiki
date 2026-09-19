---
prev: Custom Types
next: Testing
description: Details on the backend of the Quest version of the game, and how it affects your mod.
---

# Il2Cpp and C++

[Il2Cpp](https://docs.unity3d.com/Manual/scripting-backends-il2cpp.html) is the backend all the game's scripts use on
the Quest version. It's the reason Quest mods are so different from PC mods, including being written in C++.

Working in regular C++ poses some challenges that have to be considered when interacting with the Il2Cpp backend (so
any of Unity or the game's code).

## The Garbage Collector

All C# objects are garbage collected, even with Il2Cpp. At a high level, this works by periodically scanning memory for
references to every allocated object, and if none are found, freeing it. This matters because our C++ code and local variables
are not included in the memory scanned. Additionally, when objects are freed, it does not update our references to them,
so there's no easy null check that can be done.

It's also worth mentioning that references to Unity objects are not garbage collected the same way, and instead have
lifetimes tied to their actual existence in the game. So a `GameObject` pointer will be valid as long as that object
exists, even if there are no references to it in scripts.

There are solutions to use and store C# objects as one would in a C# program, when necessary.

::: tip
The use of the following methods is not always necessary, and can increase the complexity of your mod. There are some easy
workarounds that should be considered first:

- By not using C# objects where not necessary, you can avoid having to think
  about the GC in the majority of your code.
- Knowing that Unity objects remain valid for their lifetimes can allow you to logically deduce when they are valid, in
  many cases.
  - For example, a reference to a UI element in a button callback on the same menu can be used safely, as the button and
    that UI element would be destroyed at the same time.
- C# references can still be used immediately, as the GC is not exactly likely to free an object within a few CPU cycles
  of its creation.

:::

<!-- TODO double check last point, and if the stack is tracked -->

### SafePtr Types

A `safe_ptr` is a wrapper type that provides a reference to an object to the GC. Effectively, using a `safe_ptr` allows you
to store and keep around a C# object for as long as you want.

```cpp
#include "beatsaber-hook/shared/arrayw.hpp"
#include "beatsaber-hook/shared/safeptr.hpp"

safe_ptr<ArrayW<int>> cSharpArray;

void GetArray() {
    cSharpArray = ArrayW<int>({1, 2, 3, 4});
}

void UseArrayLater() {
    // We can access the raw value again with ptr(), or access methods with ->
    logger.info("Array length: {}", cSharpArray->size());
    for (int value : cSharpArray.ptr())
        logger.info("Array value: {}", value);
}
```

As mentioned, Unity objects do not use the GC, and therefore cannot be used the same way in a `safe_ptr`. However, there is a
subtlety that allows the type to be somewhat useful. _`safe_ptr`s do not prevent Unity objects from being destroyed._ Instead,
all they do is keep specifically the C# wrapper of the Unity object alive, allowing you to do null checks on it safely.

```cpp
safe_ptr<UnityEngine::GameObject*> toggleObject;

void CreateToggleObject() {
    toggleObject = UnityEngine::GameObject::New_ctor("ToggleObject");
}

void UseToggleObjectLater() {
    if (toggleObject)
        toggleObject.ptr()->SetActive(!toggleObject->activeSelf);
    else
        logger.warn("ToggleObject is destroyed, cannot use it");
}
```

::: tip
The wrapper type `UnityW` can also be used to null check Unity objects, but it does not keep the C# reference to the
object alive.
:::

::: warning
If you have a `safe_ptr` or `UnityW` that may be null, and you want to get the raw pointer anyway, make sure to use
`unchecked_ptr()` and `unsafe_ptr()` respectively instead of `ptr()`. Otherwise, an exception will be thrown if the object
is null.
:::

### Custom Types

The other way of making the GC aware of object references is through [custom types](./custom-types.md). All fields
declared with `DECLARE_INSTANCE_FIELD` in a custom type will be safe for the lifetime of that custom type, allowing the
easy use of C# objects in them. They could also potentially be used as a way of keeping a large number of objects alive
more efficiently than having a `safe_ptr` for each one.

## Internal Calls

The other significant effect of Il2Cpp on the game is [managed code stripping](https://docs.unity3d.com/Manual/managed-code-stripping.html).
Effectively, this is the removal of unused classes and methods from the compiled code, both from the game's
scripts but also from Unity.

In some cases, you can easily implement these stripped methods yourself, by copying the implementation from a [decompiler](./decompiling.md).
However, in other cases, part of the method may involve something known as an icall, or internal call.

![Templatr Example](/.assets/images/modding/quest-mod-icall-dnspy.png)

An icall is a call to the core Unity runtime, closed source and written in C++, so effectively non-decompilable. However,
as part of the mod install process on quest, the core Unity runtime is replaced with an "unstripped" version. This only
restores the methods in Unity's internals, not the C# scripts, but it's enough for mods to run those internal calls themselves.

For this example, we'll use the Microphone method from above. To run it, we'll need to find use the namespace, class,
and method names to find its identifier, then use the `resolve_icall` method from `beatsaber-hook` to get a
reference to the method before actually running it.

```cpp
// Header for the resolve_icall function
#include "beatsaber-hook/shared/api.hpp"
// Other used classes must also be included like always
#include "beatsaber-hook/shared/byref.hpp"
#include "beatsaber-hook/shared/stringw.hpp"

#include "UnityEngine/Bindings/ManagedSpanWrapper.hpp"

int GetMicrophoneDeviceID(std::string name) {
    // References to icalls are valid for the runtime of the game, so we can cache it as a static variable
    // The template parameters are the return type followed by the arguments, all as C# types
    // If the method is not static, an extra first parameter of the object pointer is added, just like in hooks
    // The identifier is the namespace and class name separated by a period, followed by :: then the method name
    static auto GetMicrophoneDeviceIDFromName =
        i2c::resolve_icall<int, by_ref<UnityEngine::Bindings::ManagedSpanWrapper>>(
            "UnityEngine.Microphone::GetMicrophoneDeviceIDFromName_Injected"
        ).value();
    // Now we can just run the method (with a little annoyance from it taking a ref ManagedSpanWrapper)
    StringW cSharpName = name;
    UnityEngine::Bindings::ManagedSpanWrapper wrapper = {cSharpName.begin(), static_cast<int>(cSharpName.size())};
    return GetMicrophoneDeviceIDFromName(by_ref(wrapper));
}
```

::: warning
Remember that only the internal methods are restored, not the full C# scripts. Only methods marked with the `InternalCall`
attribute can be used with `resolve_icall`.
:::
