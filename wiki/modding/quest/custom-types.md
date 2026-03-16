---
prev: Decompiling
next: Il2Cpp and C++
description: Create your own C# types from C++.
---

# Custom Types

`custom-types` is a library that allows you to create C# types using macros. These types can extend classes such
as `MonoBehaviour` and more, allowing you to create custom Unity components or implement interfaces used by the game.
They have a fairly significant overhead, however, and should generally be avoided when not necessary for interacting
with the game or Unity.

It also allows you to create [coroutines](https://docs.unity3d.com/Manual/Coroutines.html) and [delegates](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/).

Custom types require knowledge of basic C#.

## Basics

First, install `custom-types` by running `qpm dependency add custom-types` in your project directory. Make sure to restore
after adding the dependency.

To create a custom type, create a header file for your type. In this example, we'll make a type called `Counter`
that extends `MonoBehavior`.

In your header file, include the macros file.

```cpp
#include "custom-types/shared/macros.hpp"
```

Since our `Counter` Custom Type will be extending `MonoBehaviour`, we need to include this too.

```cpp
#include "UnityEngine/MonoBehaviour.hpp"
```

### Declaring the Type

With those files included, we can now declare our `Counter` type. Types are declared using macros, similarly to hooks.

```cpp
// Equivalent to an include guard - should be in all your header files
#pragma once

#include "custom-types/shared/macros.hpp"
#include "UnityEngine/MonoBehaviour.hpp"

// Parameters: (namespace, class name, parent class)
DECLARE_CLASS_CODEGEN(MyNamespace, Counter, UnityEngine::MonoBehaviour) {
    // Should generally be included in your custom type
    // See the Constructors section for more details
    DECLARE_DEFAULT_CTOR();

    // Parameters: (type, name)
    DECLARE_INSTANCE_FIELD(int, counts);

    // Parameters: (return type, name, arguments...)
    DECLARE_INSTANCE_METHOD(void, Update);

    // Additional macros include:
    // DECLARE_INSTANCE_FIELD_DEFAULT
    // DECLARE_INSTANCE_FIELD_PRIVATE
    // DECLARE_INSTANCE_FIELD_PRIVATE_DEFAULT
    // DECLARE_STATIC_METHOD
};
```

In C#, this would translate to the following:

```csharp
namespace MyNamespace
{
    public class Counter : MonoBehaviour
    {
        public int counts;

        public void Update() { }
    }
}
```

Note that only basic types, such as `int`, `bool`, etc, and C# types can be used as instance fields and method parameters
declared with these macros. If you need a C++ specific type, such as a `std::vector` or your own struct, you can declare
it in the class like a normal field or method, without the macros.

::: warning
Custom types do not support overloaded methods, unless none of the overloads are used in any of the macros. Instead,
simply give the methods different names.
:::

::: danger
The `DECLARE_STATIC_FIELD` macro exists, but should not be used. Instead, use C++ static fields without macros, as you
would in a normal C++ class or struct. If you also need the GC to be aware of them, use a `SafePtr`.
:::

### Defining the Type

To define the type, include the header file and use the `DEFINE_TYPE(namespace, class)` macro in a source file:

```cpp
#include "Counter.hpp"

DEFINE_TYPE(MyNamespace, Counter);
```

We can now define the `Update` method that we declared. Our `Counter.cpp` file now looks like this:

```cpp
#include "Counter.hpp"

DEFINE_TYPE(MyNamespace, Counter);

void MyNamespace::Counter::Update() {
    counter += 1;
}
```

Since our custom type inherits `MonoBehaviour`, this method will be run every frame when the component is enabled, just
like an equivalent C# type. Note that this would not happen if the `Update` method was not declared with the macro,
even in a custom type.

### Registering

You can register all the custom types in your project using the `custom_types::Register::AutoRegister()` method.

This method should be put in your `load()` or `late_load()` like so:

```cpp
#include "custom-types/shared/register.hpp"
// Counter.hpp does not have to be included

extern "C" void late_load() {
    // Make sure to run this first
    il2cpp_functions::Init();
    custom_types::Register::AutoRegister();
    // Install hooks after running AutoRegister
}
```

### Using the Type

Custom types can be used as if they were the equivalent C# type. For our `Counter` type, since it inherits `MonoBehaviour`,
we can add it as a component to a `GameObject`.

```cpp
#include "Counter.hpp"
#include "UnityEngine/GameObject.hpp"

void CreateCounter() {
    UnityEngine::GameObject* gameObject = UnityEngine::GameObject::New_ctor("CounterObject");
    gameObject->AddComponent<MyNamespace::Counter*>();
}
```

This is because Unity requires `MonoBehaviour`s to only be created this way. If we instead have a class that inherits
from a more standard C# type, it can be created with the `New_ctor` method.

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"

// Il2CppObject or System::Object can be used as an equivalent to a plain object with no parent
DECLARE_CLASS_CODEGEN(MyNamespace, SimpleExample, Il2CppObject) {
    DECLARE_DEFAULT_CTOR();
};
```

```cpp
#include "SimpleExample.hpp"

DEFINE_TYPE(MyNamespace, SimpleExample);

void CreateSimpleExample() {
    MyNamespace::SimpleExample* created = MyNamespace::SimpleExample::New_ctor();
}
```

::: danger
Do not run `ctor` yourself or use the `new` keyword on custom types. Only create them with `New_ctor`, `il2cpp_utils::New`,
or a C# method that creates an object such as `AddComponent`.
:::

## Overriding

We can also define methods that override those on parent types or interfaces, but like in C# we are limited to only
overriding methods explicitly defined as `virtual` or `abstract`. You can check this in a decompiler or in the bs-cordl
headers. An example of a virtual method that is commonly overriden is `HMUI::ViewController::DidActivate`:

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"
#include "HMUI/ViewController.hpp"

DECLARE_CLASS_CODEGEN(MyNamespace, CustomMenu, HMUI::ViewController) {
    DECLARE_DEFAULT_CTOR();

    // DECLARE_OVERRIDE_METHOD_MATCH overrides a method in a parent type or interface
    // Parameters: (return type, method name, base method, arguments...)
    // Note that the method name does not have to be the same as the overrided method name
    DECLARE_OVERRIDE_METHOD_MATCH(
        void, DidActivate, &HMUI::ViewController::DidActivate, bool firstActivation, bool addedToHierarchy, bool screenSystemEnabling
    );
};
```

### Using Interfaces

Sometimes you will need to have your custom type implement interfaces. Putting them as the parent type will not work, of
course. Instead, there is another macro for it:

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"
// Nested types, such as the interface we will be implementing, can be included by including the main type
#include "HMUI/TableView.hpp"

DECLARE_CLASS_CODEGEN_INTERFACES(MyNamespace, TableDataSource, Il2CppObject, HMUI::TableView::IDataSource*) {
    DECLARE_DEFAULT_CTOR();

    // Like in C#, all members of the interface must be implemented
    DECLARE_OVERRIDE_METHOD_MATCH(float, CellSize, &HMUI::TableView::IDataSource::CellSize);
    DECLARE_OVERRIDE_METHOD_MATCH(int, NumberOfCells, &HMUI::TableView::IDataSource::NumberOfCells);
    DECLARE_OVERRIDE_METHOD_MATCH(
        HMUI::TableCell*, CellForIdx, &HMUI::TableView::IDataSource::CellForIdx, HMUI::TableView* tableView, int idx
    );
};
```

### Custom Base Types

If you need to make a custom type that inherits from a custom type, instead of regular C# type, use the
`DECLARE_CLASS_CUSTOM` macro instead of `DECLARE_CLASS_CODEGEN`.

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"

DECLARE_CLASS_CODEGEN(MyNamespace, Parent, Il2CppObject) {
    // Implementation as usual
};

DECLARE_CLASS_CUSTOM(MyNamespace, Child, MyNamespace::Parent) {
    // Implementation as usual
};
```

`DECLARE_CLASS_CUSTOM_INTERFACES` exists as well.

::: danger
Do not use C++ `virtual` methods in custom types. Currently, there is no way to create your own virtual methods in
custom types, so if you need custom polymorphism you will have to work around that.
:::

## Constructors

For most custom types, the `DECLARE_DEFAULT_CTOR` macro is enough. However, sometimes you may want to have a custom constructor.
You can do so with the `DECLARE_CTOR` macro:

```cpp
#pragma once

#include "custom-types/shared/macros.hpp"

DECLARE_CLASS_CODEGEN(MyNamespace, ConstructorExample, Il2CppObject) {
    // Parameters: (method name, arguments...)
    DECLARE_CTOR(ctor);
};
```

Then, in a source file, define it just like any other method. However, in that definition, you should make sure to run
the `INVOKE_CTOR` and `INVOKE_BASE_CTOR` macros:

```cpp
void MyNamespace::ConstructorExample::ctor() {
    // Initializes C++ fields
    INVOKE_CTOR();
    // Runs base class constructor
    INVOKE_BASE_CTOR(classof(parent type*), ...arguments);
}
```

`INVOKE_CTOR` runs the C++ constructor on your class. This is necessary if your class has any fields with default values
or non-trivial constructors themselves, such as `std::vector`.

`INVOKE_BASE_CTOR` runs the constructor of your class's parent type. In the case of `Il2CppObject` or something that
doesn't do anything in its constructor such as `MonoBehaviour`, this isn't necessary.

::: tip
While both of these can be omitted in many cases, it's best to just include them unless you have a good reason not to. Not
doing so can cause hard to track down bugs.

The `DECLARE_DEFAULT_CTOR` macro used in the examples creates a constructor with `INVOKE_CTOR` and `INVOKE_BASE_CTOR` and
nothing else, so it's an easy thing to paste in all your custom types.
:::

Destructors can be defined custom similarly to contructors with `DECLARE_DTOR`, or `DECLARE_SIMPLE_DTOR` to run
the destructor for any C++ fields that need to have special behavior when being destroyed. You don't need to worry
about running the base class destructor, though.

<!-- TODO investigate dtor potential bugs -->

## Coroutines

In Unity, a [coroutine](https://docs.unity3d.com/Manual/Coroutines.html) is a method that can pause execution and return
control to Unity, then continue where it left off on the following frame.

### Creating a Coroutine

Using `custom-types`, coroutines are pretty much the same as their C# counterparts. Take a look at this example:

```cpp
#include "custom-types/shared/coroutine.hpp"
#include "UnityEngine/WaitForSeconds.hpp"
#include "System/Collections/IEnumerator.hpp"

custom_types::Helpers::Coroutine CounterCoroutine() {
    int secondsPassed = 0;

    for (int i = 0; i < 30; i++) {
        secondsPassed++;

        // Arguments passed to co_yield must be cast to this type
        // You can also wait a single frame with: co_yield nullptr
        co_yield reinterpret_cast<System::Collections::IEnumerator*>(UnityEngine::WaitForSeconds::New_ctor(1));
    }
    co_return;
}
```

| C#             | C++         |
| -------------- | ----------- |
| `yield return` | `co_yield`  |
| `yield`        | `co_yield`  |
| `yield break`  | `co_return` |

`co_return` is used to end a coroutine. C# automatically handles this during compilation, but C++ does
not, so make sure you have one at the end of all your coroutines.

You can also use `co_return` to exit a coroutine early, just like `return` would in a typical function.

Using normal `return` in a coroutine will not work.

### Using the Coroutine

You can start a coroutine on any `MonoBehaviour` using the `StartCoroutine` method just like in C#. However, to turn
your function into an actual C# coroutine, you need to use `custom_types::Helpers::CoroutineHelper::New`.

```cpp
#include "custom-types/shared/coroutine.hpp"
#include "UnityEngine/GameObject.hpp"

void CreateCoroutine() {
    auto gameObject = UnityEngine::GameObject::New_ctor("MyCoroutineRunner");
    // This is the example custom type from earlier, but anything inheriting from a MonoBehaviour will work, custom or not
    auto component = gameObject->AddComponent<MyNamespace::Counter*>();
    // This creates the actual coroutine object that can be passed to StartCoroutine
    auto coroutine = custom_types::Helpers::CoroutineHelper::New(CounterCoroutine());
    component->StartCoroutine(coroutine);
}
```

## Delegates

[Delegates](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/) are C#'s mechanism for functional
programming, and are very commonly used in Beat Saber.

::: tip
The easiest way to create many types of delegates is with `metacore`.

Include `metacore/shared/delegates.hpp`, then use `CreateSystemAction` or `CreateUnityAction`. They take either a `std::function`
or a lambda, and the argument types will be automatically inferred.

This method still uses `custom-types` internally, of course, and does not cover every delegate type.
:::

To create a delegate, you'll need to include the header file, then use `custom_types::MakeDelegate`.

```cpp
#include "custom-types/shared/delegate.hpp"

using namespace UnityEngine;

void OnSceneLoad(SceneManagement::Scene scene, SceneManagement::LoadSceneMode mode) {
    logger.info("Scene loaded!");
}

void RegisterDelegate() {
    // List the delegate type, with its parameters, for conciseness in the next line
    using DelegateType = Events::UnityAction_2<SceneManagement::Scene, SceneManagement::LoadSceneMode>*;

    // Provide the delegate type as the template parameter, and cast our callback to a std::function
    auto delegate = custom_types::MakeDelegate<DelegateType>(
        std::function<void(SceneManagement::Scene scene, SceneManagement::LoadSceneMode)>(OnSceneLoad)
    );

    // Alternatively, with metacore:
    // auto delegate = MetaCore::Delegates::CreateUnityAction(OnSceneLoad);

    // Add our delegate; += does not work in C++
    SceneManagement::SceneManager::add_sceneLoaded(delegate);
}
```
