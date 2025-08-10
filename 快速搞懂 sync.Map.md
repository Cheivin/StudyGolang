## **1. sync.Map长啥样**

### **1.1 结构定义**

#### **1.1.1 Map顶层结构**

```go
type Map struct {
    mu     sync.Mutex                    // 一把大锁，只用于 dirty 晋升、miss 清零等低频动作
    read   atomic.Pointer[readOnly]      // 读热路径，CAS 无锁
    dirty  map[any]*entry                // 写操作的主战场，需要 mu 保护
    misses int                          // read 未命中次数，达到 len(dirty) 触发晋升
}
```

- `read` 是一个原子指针，指向**只读结构 readOnly**
- `dirty` 是普通 `map`，由锁 `mu` 保护
- `misses` 用于 `dirty` 晋升的计数器，用来**决定何时把 dirty 整体搬进 read**

#### **1.1.2 readOnly**

```go
type readOnly struct {
    m       map[any]*entry  // 只读 map
    amended bool            // true ⇒ dirty 里还有 read 中没有的 key
}
```

- `readOnly.m` 与 `dirty` 中的 `key` 可能同时存在，但值指针指向同一个 `entry`

- `amended`的一些细节

    - `amended`的影响效果
        - 为 `false` 时，读操作可以完全不走锁
        - 为 `true` 时，读未命中需要 加锁再到 `dirty` 里找
    - 初始状态：`read.amended == false`同时`dirty == nil`
    - 变更时机：在 `Store` 时：

        1. `key` 既不在 `read.m` 也不在 `dirty`(因为 dirty 可能是 nil)
        2. 于是需要先把 `read` 中未被删除的 `key` 复制出来初始化 dirty(dirtyLocked) 
        3. 紧接着执行`m.read.Store(readOnly{m: read.m, amended: true})`把 `amended` 设为 `true`
    - 只要 `dirty` 里还有 `read` 中没有的新 `key`，`amended` 就一直是 `true`
    - 当 `misses == len(dirty)` 触发**晋升**，或**手动调用 Range 强制晋升**时，`dirty` 被整体提升为 `read`，`dirty` 置为 `nil`，`amended` 重新变回 `false`

    
#### **1.1.3 entry**

```go
type entry struct {
    p atomic.Pointer[any] // 实际存储 *value 的指针
}
```

- `p`有三种形态

    - 正常指针：存储 `*value` 的指针
    - `nil`：已删除(key仍在，但值已经删除，lazy清理)
    - `expunged`：`晋升`时，由源码统一把当前**p为nil**的`entry`标记为`expunged`，从而告诉后续写入“这条 key 在 dirty 里已不存在”

- `entry`为`expunged`的一些细节：

    - 晋升时，只把未删除 **(非 expunged)** 的 `key/entry` 复制到新 read，而 `expunged` 的 key 被自然丢弃 **(不再放进新的 readOnly.m)**
    - 晋升完成后，旧 `readOnly` 对象成为无人引用的垃圾，整个 map，连同里面的 `expunged entry`一起被 GC 回收
    - 如果 key 只在 read 里且已 expunged，而 dirty 为 nil，后续也没有任何 Store 再出现这个 key，那么：

        - `read` 里的这条记录会一直存在
        - 直到下一次晋升，或整个 Map 被释放时，才会**随旧 read 一起被 GC**

### **1.2 Map的三种场景**

| 场景       | read.amended | dirty | 读写特征                 |
| -------- | ------------ | ----- | -------------------- |
| 只读       | false        | nil   | 读≈无锁，写需复制 read→dirty |
| 读写交替(常态) | true         | ≠nil  | 读少量锁，写批量             |
| 写爆炸      | 频繁切换         | 频繁重建  | 退化为类似 map + Mutex      |

> 一句话：读路径只用原子指令，写路径先 dirty，再定期把 dirty 整体搬进 read，实现 **无锁读 + 批量写**。

## **2. Load怎么读的？**

1. 无锁

    直接 `CAS` 读 `read.m[key]`，找到`entry`且 `p != nil && p != expunged` → 立即返回
    
2. 加锁

    - `amended == true` 且 `read.m` 没找到 → 加锁再到 `dirty` 里找
    - 无论找没找到，`misses++`，当 `misses == len(dirty)` 触发 **dirty 晋升**

3. dirty 晋升

    - 将 dirty 整体提升为新的 read`(readOnly.m = dirty)`，重建 dirty 为 nil，并重置 `misses = 0`
    - 晋升时会把所有 `nil entry`的`p`置为 `expunged`，后续插入走`dirty`

> 一句话：读尽可能无锁，miss 累积到一定程度，把 dirty 整体搬到 read，摊销锁开销

## **3. Store怎么写的？**

1. 无锁

    如果 `key` 在 `read.m` 已**存在**且 `entry.p != expunged` → `CAS`更新，全程无锁

2. 加锁

    - 先加锁
    - 如果 `dirty`为`nil` → 先复制 read 中所有未删除的 `key` 到 dirty，再插入新值
        - 一次性复制可以把本来每次写都可能发生的复制成本，集中在一小段时间里一次性完成
        - 只有当 `dirty==nil` 且 需要写新 `key` 时才会触发，频率远低于每次写都复制
    - 若 `key` 在 `dirty` 已存在 → 更新 `entry.p`
    - 若 `key` 不存在 → 先检查 `key` 在 `read` 中是否被标记 `expunged`，是的话重新放进 `dirty`，再更新指针

> 一句话：写优先CAS更新旧值，无旧值则加锁走dirty，无dirty还得把read中有效都key刷到dirty

- LoadOrStore

    Load + Store 的组合，但把两个操作放在同一次锁临界区内，**避免并发双写，保证原子性**
    
## **4. Delete怎么删？**

1. 无锁
    如果 `key` 在 `read.m`，则直接 `CAS` 把 `entry.p` 置 `nil`

2. 加锁
    加锁后在 `dirty`中`delete(dirty, key)`真正删除`key`

3. 何时清理
    `dirty`晋升时不复制`expunged key`，`read` 中所有 `nil key`会置为`expunged`并在下一轮晋升被自然淘汰
    
## **5. Range的便利过程**

1. 加锁 mu，然后`dirty`晋升（保证一致性）
2. 遍历`readOnly.m`，对每个**非 nil 非 expunged entry** 回调`f(k, v)`
> Range 自身**不会**修改 map，但会触发**强制晋升**

## **6. 面试题速查**

### **6.1 内存与 GC**

#### **6.1.1 entry 被 GC 扫描吗？**

- `entry` 本身是指针，`readOnly.m` 和 `dirty` 都是普通 map，`key/value` 按指针规则扫描
- `p` 为 `nil/expunged` 时，`value` 目标的引用已清楚，GC 不再追踪

#### **6.1.2 sync.Map 会内存泄漏吗？**

不会。晋升时把已删除 key 自然清理；无额外链表，生命周期随 Map 实例

### **6.2 并发与调度**

#### **6.2.1 为什么读能无锁？**

- `read`字段用 `atomic.Pointer`，整个`readOnly`不可变，读操作只读 `map + entry`，无数据竞争
- 写操作通过**新建 readOnly**原子替换指针，实现 RCU(Read-Copy-Update)思想

#### **6.2.2 写操作何时会阻塞？**

仅当需要创建/重建 dirty 或执行晋升时会持 mu，其他并发写排队，**短暂阻塞**

### **6.3 性能陷阱**

#### **6.3.1 为什么 sync.Map 不适合写远大于读？**

- 每次写大概率触发 dirty 重建 → 全表复制 + 全 map 扫描，O(n) 开销
- 高并发写会导致 **dirty 频繁晋升**，退化为 map + Mutex

#### **6.3.2 Range 会阻塞写吗？**

- 会持 mu 晋升 dirty，**短暂阻塞**所有写，但读仍可走旧 read
- 大量 Range 建议异步或合并，避免抖动

#### **6.3.3 value存指针还是值？**

- 与 `channel` 类似，大于**128 B**建议存指针，减少晋升时复制成本
- `value` 含锁或通道时，一律存指针，避免复制语义导致状态分裂

> 影响sync.Map性能的核心就是**晋升**，它会持有锁，并且产生数据迁移成本
