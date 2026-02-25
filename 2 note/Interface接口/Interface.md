https://zhuanlan.zhihu.com/p/609360213

一个接口之所以分为 `UInterface`（U 前缀）和 `IInterface`（I 前缀）两个类，是因为它们承担了**完全不同的职责**：**`UInterface` 负责接入引擎的反射系统，而 `IInterface` 负责提供真正的功能接口**。这种巧妙的设计是为了在 C++ 单一继承的限制下，让接口也能获得 UObject 体系的强大能力。

简单来说，`UInterface` 类是一个“空壳子”，它的唯一使命就是**让这个接口能被 UE 的反射系统（如类型识别、序列化、蓝图等）所识别和处理。它本身几乎不包含任何逻辑代码。

真正干活的是 `IInterface` 类，它就是一个标准的 C++ 纯虚接口类，包含了所有需要被实现的**函数声明**。当你想让一个类具备某个接口的能力时，你需要继承的正是这个 `I` 前缀的类

# 为什么要这么设计？
- **UObject 的单继承限制与接口的多继承需求**：在 C++ 中，一个类可以多继承。但在 UE 的 UObject 体系中，一个 UObject 派生类（如 `AActor`）只能继承一个 `UObject` 基类。如果接口本身也是一个 `UObject`，那么实现它的类就会面临继承两个 `UObject` 的窘境，这在 UE 的反射系统中是不允许的[](https://forums.unrealengine.com/t/lyra-c-multiple-inheritance-for-gameinstance-derived-class-diamond-problem/2445108/1)[](https://www.oreilly.com/library/view/unreal-engine-4-x/9781789809503/14d3f554-6970-4024-86df-12d8759bf4b9.xhtml)。
    
- **`UInterface`：为接口赢得“反射特权”**：通过让 `UInterface` 继承自 `UObject`，这个接口就在 UObject 体系中注册成功了。现在，UE 的反射系统（如垃圾回收、编辑器属性窗口）就可以识别它，知道它是一个“接口类型”。
    
- **`IInterface`：让类能“实现”多个功能**：由于 `IInterface` 只是一个普通的 C++ 类，你可以在继承 `UObject` 派生类（如 `AActor`）的同时，**以 C++ 多继承的方式**继承任意多个 `IInterface`。这就绕开了 `UObject` 的单继承限制，让不同的类可以共享同一套接口功能[](https://dev.epicgames.com/documentation/tr-tr/unreal-engine/interfaces-in-unreal-engine?application_version=5.0)[](https://dev.epicgames.com/documentation/zh-cn/unreal-engine/interfaces-in-unreal-engine?application_version=5.4)[](https://www.cnblogs.com/scyrc/p/17499221.html)。