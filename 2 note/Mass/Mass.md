UE5的ECS：MASS框架(一)
https://zhuanlan.zhihu.com/p/441773595

# 1 FMassEntityHandle
Entity就相当于是 FMassEntityHandle，其中FMassEntityHandle的结构如下：
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250716234618.png)
其中只有两个成员变量，显而易见的就是在一个大数组里存储的索引和一个唯一标识，和FWeakObjectPtr原理是一样的。那么这个大数组存储在哪呢？下面这张图就是创建 FMassEntityHandle 的方法。
![image.png](https://gitee.com/lurenjia399/image/raw/master/image/20250716235838.png)
图中可以看到大数组索引就是 EntityIndex。我们第一次走到这个Acquire方法会执行AddPage来添加页，也就是分配一大块内存（内存的大小 = sizeof(FEntityData) * countprepage）,这个页中会有count数量的FEntityData，所有页中的FEntityData索引都是递增的。==所以呢 EntityIndex 代表的就是页中FEntityData的索引。而我们的大数组呢，就是所有页的集合。==

# 2 FMassEntityTemplate

FMassEntityTemplate 是用来存放 Archetype 的，那什么是 Archetype 呢？类是对象的原型，UObject的原型是CDO，那么Entity的原型就是Archetype，可以这么理解吧。回过头来，FMassEntityTemplate的创建是由

# 1 MassSample学习
## 1 # CrowdGym 场景


