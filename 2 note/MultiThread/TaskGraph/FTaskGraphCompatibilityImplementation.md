# 创建

```cpp
1 FTaskGraphCompatibilityImplementation是一个静态全局的对象，继承FTaskGraphInterface
2 在引擎的初始化PreInitPreStartupScreen方法中，创建出来
3 这个类构造函数中会通过FScheduler来创建出工作线程，紧接着就会创建出3个FNamedTaskThread，分别是RHIThread，GameThread，RenderThread。其中NamedTaskThread是继承FRunnable的
```

# QueueTask
```cpp

```