#### AtomicLong与LongAdder
##### AtomicLong简要介绍
AtomicLong是作用是对长整形进行原子操作，显而易见，在java1.8中新加入了一个新的原子类LongAdder，该类也可以保证Long类型操作的原子性，相对于AtomicLong，LongAdder有着更高的性能和更好的表现，可以完全替代AtomicLong的来进行原子操作。
在32位操作系统中，64位的long 和 double 变量由于会被JVM当作两个分离的32位来进行操作，所以不具有原子性。而使用AtomicLong能让long的操作保持原子型。
####  LongAdder https://www.cnblogs.com/rickzhai/p/7936758.html
#### 并发与并行

```
并发性指的是同一时刻只能有一条指令执行，但多个进程指令被快速的轮换执行，使得在宏观上具有多个进程同时执行的效果。

并行性指同一时刻，有多条指令在多个处理器上同时执行。
```
