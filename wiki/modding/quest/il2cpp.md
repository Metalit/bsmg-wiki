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

A `SafePtr` is a wrapper type that provides a reference to an object to the GC. Effectively, using a `SafePtr` allows you
to store and keep around a C# object for as long as you want.

```cpp
SafePtr<Array<int>> cSharpArray;

void GetArray() {
    // SafePtr requires a pointer type, so ArrayW can't be used directly in it
    cSharpArray = (Array<int>*) ArrayW<int>({1, 2, 3, 4});
}

void UseArrayLater() {
    // We can access the raw pointer again with ptr(), or call methods with -> like normal
    logger.info("Array length: {}", cSharpArray->get_Length());
    for (int value : ArrayW<int>(cSharpArray.ptr()))
        logger.info("Array value: {}", value);
}
```

As mentioned, Unity objects do not use the GC, and therefore should not be used in a `SafePtr`. However, there is a subtlety
that allows for the existence of a `SafePtrUnity`. _This type does not prevent the object from being destroyed._ Instead,
all it does is keep specifically the C# wrapper of the Unity object alive, allowing you to do null checks on it safely.

```cpp
SafePtrUnity<UnityEngine::GameObject> toggleObject;

void CreateToggleObject() {
    toggleObject = UnityEngine::GameObject::New_ctor("ToggleObject");
}

void UseToggleObjectLater() {
    if (toggleObject)
        toggleObject->SetActive(!toggleObject->activeSelf);
    else
        logger.warn("ToggleObject is destroyed, cannot use it");
}
```

::: tip
Part of the features of `SafePtrUnity` are also provided by the wrapper type `UnityW`. It provides the same null check,
but does not keep the C# reference to the object alive.
:::

::: warning
If you have a `SafePtrUnity` or `UnityW` that may be null, and you want to get the raw pointer anyway, make sure to use
`unsafePtr()` instead of `ptr()`. Otherwise, an exception will be thrown if the object is null.
:::

### Custom Types

The other way of making the GC aware of object references is through [custom types](./custom-types.md). All fields
declared with `DECLARE_INSTANCE_FIELD` in a custom type will be safe for the lifetime of that custom type, allowing the
easy use of C# objects in them. They could also potentially be used as a way of keeping a large number of objects alive
more efficiently than having a `SafePtr` for each one.

## Internal Calls

The other significant effect of Il2Cpp on the game is [managed code stripping](https://docs.unity3d.com/Manual/managed-code-stripping.html).
Effectively, this is the removal of unused classes and methods from the compiled code, both from the game's
scripts but also from Unity.

In some cases, you can easily implement these stripped methods yourself, by copying the implementation from a [decompiler](./decompiling.md).
However, in other cases, part of the method may involve something known as an icall, or internal call.

<!-- TODO picture of icall in ilspy -->

An icall is a call to the core Unity runtime, closed source and written in C++, so effectively non-decompilable. However,
as part of the mod install process on quest, the core Unity runtime is replaced with an "unstripped" version. This only
restores the methods in Unity's internals, not the C# scripts, but it's enough for mods to run those internal calls themselves.

For this example, we'll use the AssetBundle method from above. To run it, we'll need to find use the namespace, class,
and method names to find its identifier, then use the `resolve_icall` method from `beatsaber-hook` to get a
reference to the method before actually running it.

```cpp
// Header for the resolve_icall function. Other used classes must also be included like always
#include "beatsaber-hook/shared/utils/il2cpp-utils.hpp"

UnityEngine::AssetBundleCreateRequest* LoadAsset(std::string file) {
    // References to icalls are valid for the runtime of the game, so we can cache it as a static variable
    // The template parameters are the return type followed by the arguments, all as C# types
    // If the method is not static, an extra first parameter of the object pointer is added, just like in hooks
    // The identifier is the namespace and class name separated by a period, followed by :: then the method name
    static auto LoadFromFileAsync =
        il2cpp_utils::resolve_icall<UnityEngine::AssetBundleCreateRequest*, StringW, uint, u_long>("UnityEngine.AssetBundle::LoadFromFileAsync_Internal");
    // Now we can just run the method
    return LoadFromFileAsync(file, 0, 0);
}
```

::: warning
Remember that only the internal methods are restored, not the full C# scripts. Only methods marked with the `InternalCall`
attribute can be used with `resolve_icall`.
:::
