# ConcurrentMemoryPool
参考Google TCmalloc源码实现的高并发内存池项目

本项目实现了一个高效的多线程内存分配器，采用三级缓存结构优化性能，核心模块包括：

1. **ThreadCache** - 线程本地缓存，无锁分配
2. **CentralCache** - 中心缓存，全局共享，桶锁同步
3. **PageCache** - 页缓存，全局唯一，管理大块内存
4. **ObjectPool** - 对象池优化高频小对象分配

------

### 核心模块解析

#### 1. ThreadCache（线程缓存）

- **线程局部存储**：`_declspec(thread)`实现无锁访问

- **自由链表管理**：16B-256KB内存分配，按SizeClass对齐

- **批量获取/归还**：慢启动算法动态调整批量大小

- 关键方法：

  ```
  void* Allocate(size_t size);       // 申请内存
  void Deallocate(void* ptr, size_t size); // 释放内存
  ```

#### 2. CentralCache（中心缓存）

- **哈希桶结构**：240个桶管理不同规格内存块

- **Span切分管理**：将PageCache分配的Span切分为小块

- **交叉锁设计**：桶锁+全局页锁避免死锁

- 关键方法

  ```
  size_t FetchRangeObj(...);    // 批量获取对象
  void ReleaseListToSpans(...); // 回收内存到Span
  ```

#### 3. PageCache（页缓存）

- **全局单例**：管理最大128页的Span

- 伙伴系统：

  - 页合并：`ReleaseSpanToPageCache`时前后合并
  - 页分割：`NewSpan`时拆分大Span

- **基数树映射**：TCMalloc_PageMap1实现页号到Span的O(1)查找

- 关键方法：

  ```
  Span* NewSpan(size_t k);             // 获取K页Span
  void ReleaseSpanToPageCache(Span* span); // 释放Span
  ```

#### 4. 对象池（ObjectPool）

- **内置优化**：用于Span和ThreadCache对象分配
- **自由链表管理**：避免频繁调用系统malloc
- **内存对齐**：保证至少分配指针大小的内存块

------

### 关键技术点

#### 1. 内存对齐策略

```
// Common.h
static inline size_t RoundUp(size_t size) {
    if (size <= 128) return _RoundUp(size, 8);
    else if (...) // 多级对齐策略
}
```

- 小对象按8/16/...字节对齐
- 大对象按页对齐

#### 2. 锁粒度优化

- **线程级**：ThreadCache无锁
- **桶级锁**：CentralCache每个桶独立锁
- **全局页锁**：PageCache使用单独锁

#### 3. 高效映射设计

```
// PageMap.h
TCMalloc_PageMap1<32-PAGE_SHIFT> _idSpanMap; 
```

- 使用单层基数树实现页号到Span的快速映射
- 每个页号直接对应数组下标

#### 4. 批量操作优化

- **慢启动算法**：动态调整批量大小（16→MaxNum）
- **链表批处理**：`PushRange/PopRange`减少锁竞争

#### 5. 内存碎片控制

- **Span合并**：释放时检测前后页是否空闲
- **统一页管理**：超过128页直接使用系统API

------

### 性能对比

通过Benchmark测试（4线程/10000次操作）：
![image](https://github.com/user-attachments/assets/fe75c467-c2fe-4b48-9497-47e783cc5375)

优势体现：

1. 减少系统调用次数
2. 降低锁竞争概率
3. 缓存局部性优化

