## **1. channel长啥样**

### **1.1 结构定义**

#### 1.1.1 hchan

```go
type hchan struct {
   qcount   uint           // 循环队列中的数据总数
   dataqsiz uint           // 循环队列大小
   buf      unsafe.Pointer // 指向循环队列的指针
   elemsize uint16         // 循环队列中的每个元素的大小
   closed   uint32         // 标记位，标记channel是否关闭
   elemtype *_type         // 循环队列中的元素类型
   sendx    uint           // 已发送元素在循环队列中的索引位置
   recvx    uint           // 已接收元素在循环队列中的索引位置
   recvq    waitq          // 等待从channel接收消息的sudog队列
   sendq    waitq          // 等待向channel写入消息的sudog队列
   lock mutex              // 互斥锁，对channel的数据读写操作加锁，保证并发安全
}
```

-   链表节点是 `sudog`（G + 数据地址 + ticket 号）
    
-   元素无指针 → 整块 GC 标记 `noscan`，直接跳过扫描
    
-   通过环形回绕的方式使用，环形回绕一句：`(idx+1)%dataqsiz`
    

#### 1.1.2 wqitq

```go
type waitq struct {
    first *sudog              // sudog队列的队头指针
    last  *sudog              // sudog队列的队尾指针
}
```

-   waitq是对一个sudog链表进行封装之后的一个结构，其字段为这个sudog队列的首位指针，链表中所有的元素都是 sudog 结构
    
-   **出队：** 直接取 `first`，移动 `first = first.next`
    
-   **入队：** 把新节点挂到 `last.next`，再移动 `last = newNode`
    
-   channel 只在队头/队尾操作，**从不中间删除**，单向链表已满足需求；少一个 `prev` 指针，省 8 字节/节点，GC 扫描也更快
    

#### 1.1.3 sudog

```go
type sudog struct {
   g *g                  // 绑定的goroutine
   next *sudog           // 指向sudog链表中的下一个节点
   prev *sudog           // 指向sudog链表中的下前一个节点
   elem unsafe.Pointer   // 数据对象
   acquiretime int64     
   releasetime int64
   ticket      uint32
   isSelect bool
   success bool
   parent   *sudog // semaRoot binary tree
   waitlink *sudog // g.waiting list or semaRoot
   waittail *sudog // semaRoot
   c        *hchan // channel
}
```

-   注意elem字段，当向channel发送数据时，elem代表将要保存进channel的数据，当从channel读取数据时，elem代表从channel接受的数据
    

### **1.2 buf缓冲区**

#### 1.2.1 大小怎么定？

-   `make(chan T, N)` 里 `N` 直接作为 `dataqsiz` 存入 `hchan`
    
-   若 `N == 0` → 无缓冲，buf 指针为 nil。
    
-   若 `N > 0` → 运行时一次性 `mallocgc(N * elemsize, elemtype, 0)` 分配整块内存。
    

#### **1.2.2 为什么用环形数组，好处在哪里？**

-   只要维护头尾两个下标 `sendx / recvx`，就能在常数时间完成入队/出队；不需要像链表那样遍历或动态扩容。
    
-   **CPU 缓存友好：** 整块内存连续，访问模式固定，cache line 命中率高；链表节点分散，易造成 cache miss。
    
-   **内存一次性分配：** channel 创建时 `make(chan T, N)` 就知道容量，一次性 malloc 一块固定内存；后续不再扩容，避免运行时额外开销。
    
-   **零拷贝与回绕简单：** 环形回绕只需一次取模运算：`next := (idx + 1) % dataqsiz` ，相比链表指针跳转，指令更少。
    

### **1.3 channel的三种形态**

-   **无缓冲**：容量 0，必须握手
    
-   **有缓冲**：容量 > 0，环形队列
    
-   **nil ch**：读、写、关闭全阻塞（还会 panic）
    

## **2. ch <- v 怎么写的？**

1.  **加锁：**`ch.lock`全局互斥，保证并发安全。
    
2.  **优先直接交付**
    
    无缓冲或有缓冲但有人等读( `recvq` 非空) → **零拷贝** 把数据直接搬到对方 goroutine 栈，唤醒 G
    
3.  **写数据**
    
    -   **缓冲区已满：** 构造 `sudog`，挂入 `sendq`，调用 `gopark` 休眠；被唤醒后继续完成写入
        
    -   **缓冲区未满：** 把数据写入 `buf[sendx]`，`sendx = (sendx+1)%dataqsiz`，`qcount++`
        
4.  **解锁：** 释放全局互斥锁
    

> 一句话：写路径永远先锁后交付，缓冲区未满就落盘，满了就挂链表睡觉。

## **3. <- ch 怎么读的？**

1.  **加锁：**`ch.lock`全局互斥，保证并发安全。
    
2.  **优先直接交付**
    
    有缓冲且有人等写( `sendq` 非空) → 直接从等待者拿数据，把对方数据搬到自己栈，唤醒 G
    
3.  **读数据**
    
    -   **缓冲区有数据：** 从 `buf[recvx]` 拷贝出数据，`recvx = (recvx+1)%dataqsiz`，`qcount--`，解锁返回
        
    -   **缓冲区为空：** 构造 `sudog`，挂入 `recvq`，调用 `gopark` 休眠；被唤醒后继续完成读取
        
4.  **解锁：** 释放全局互斥锁
    

> 一句话总结：读路径永远先锁后交付，缓冲区有货就取，空了挂链表睡觉。

## **4. 非缓冲 & nil ch**

-   非缓冲（`make(chan T)`）：`dataqsiz = 0`，`buf = nil`，读写必须 **握手**，否则直接挂起。
    
-   nil ch：`hchan = nil`，所有操作立即返回 **永久阻塞**（读/写/关闭都会 panic 或挂死）。
    

## **5. 关闭 close(ch)**

1.  **设置关闭标志**
    
    -   通过原子操作，`atomic.Store(&hchan.closed, 1)`
        
    -   对后续产生的写操作立即触发panic
        
2.  **清缓冲区**
    
    -   如果存在缓冲区，根据FIFO的顺序，把剩余数据全部交付给 `recvq` 里的等待者
        
3.  **广播recvq**
    
    -   缓冲区清空后，剩余 `recvq` 中的 G 全部收到 `(零值, false)`，一次性 `goready`
        
4.  **杀sendq**
    
    -   遍历 `sendq`，每个等待写 G 直接 `panic("send on closed channel")`
        

> 一句话总结：关闭先置位，再顺序清缓冲区，广播 recvq，最后杀 sendq。

## **6. select和channel如何搭配的**

1.  **编译期：** case → scase
    

-   每个 `case` 被编译器翻译成 `scase`
    
    ```
    type scase struct {
        c    *hchan         // 目标 channel
        elem unsafe.Pointer // 数据地址（读/写）
        kind uint16        // 类型：caseRecv / caseSend / caseDefault / caseNil
        pc   uintptr       // 调试用的 PC
        releasetime int64  // 竞态检测
    }
    ```

-   `default` 也会被转成 `scase`，只是 `kind = caseDefault`
    
-   所有 case 会被塞进一个数组 `scases []scase`，并 **随机洗牌**
    

2.  **运行时：**`selectgo` **的两轮扫描**
    
    1.  第一轮：**立即执行**
        
        1.  随机洗牌：使用 Fisher-Yates 算法把 `scases` 乱序，防止固定顺序导致饥饿。
            
            -   Fisher-Yates：经典“洗牌算法”。给定一个数组，从最后一个元素开始，向前依次把当前元素与随机下标（0 ≤ j ≤ i）交换，时间复杂度 O(n)，保证每个排列概率均等。
                
        2.  顺序检查
            
            -   对每个 case 调用对应“快速探测”函数：
                
                -   `caseRecv` → `chanrecv` 的 non-blocking 路径
                    
                    -   如果 `block=false`，立即探测能否读；成功返回 `(true, true)`，失败返回 `(false, false)`
                        
                -   `caseSend` → `chansend` 的 non-blocking 路径
                    
                    -   如果 `block=false`，立即探测能否写；成功返回 `true`，失败返回 `false`
                        
                -   select 第一轮扫描时把 `block` 传 `false`，拿到布尔返回值即可判断该 case 是否立即可执行。只要有一个 case 就绪，立刻返回索引，整个 select 结束。
                    
    2.  第二轮：**全部挂起**
        
        1.  如果第一轮没有 case 就绪：
            
            1.  构造 sudog 链表
                
                -   为每个 scase 生成 sudog，挂到对应 channel 的 `recvq` 或 `sendq`。
                    
                -   sudog 里带上 **ticket 号**（洗牌后的序号），用于后续“公平唤醒”。
                    
            2.  park 当前 G
                
                -   调用 `gopark` 把当前 goroutine 挂起，**一次性**释放所有 channel 的锁。
                    
            3.  被唤醒后的清理
                
                -   当某个 channel 变为就绪：
                    
                    -   调度器找到最先挂入的 sudog（ticket 最小）；
                        
                    -   从 **其他 channel 的等待队列** 里把自己摘掉，避免惊群；
                        
                    -   返回被选中的 case 索引，继续执行。
                        
3.  **公平性与饥饿**
    
    -   **随机洗牌** 保证每个 case 有相同概率在第一轮被选中；
        
    -   **ticket 机制** 保证第二轮唤醒时“先到先服务”，防止新加入 case 插队
        

> 一句话总结：编译期转 scase → 运行时洗牌 → 两轮扫描 → 挂 sudog → 公平唤醒；洗牌 + ticket 双保险，既防饥饿，又避免惊群。

## 7\. 面试题速查

### **7.1 内存与 GC**

#### **7.1.1 缓冲区元素何时被 GC 扫描？**

元素含指针则整块缓冲区被扫描；无指针标 `noscan`，GC 跳过。

#### **7.1.2 channel 会内存泄漏吗？**

不会，链表随 channel 生命周期一起消失。

#### **7.1.3 关闭后再写会怎样？**

写入前有检查 `if closed != 0 { panic(...) }`，因此会立即 `panic("send on closed channel")`。

* * *

### **7.2 并发与调度**

#### **7.2.1 send/recv 怎么无锁唤醒？**

链表节点直接放 goroutine 指针，调度器`goready(g)`一次唤醒，无需二次加锁。

#### **7.2.2 select 随机性？**

`selectgo` 先 Fisher-Yates 洗牌，再两轮扫描。for中每轮循环重新洗牌，顺序不可预测

#### **7.2.3 nil channel 行为？**

读写关闭均永远阻塞或 panic

* * *

### **7.3 性能陷阱**

#### **7.3.1 无缓冲 vs 有缓冲性能？**

-   无缓冲需一次 G 切换；有缓冲未满时零切换。
    
    -   无缓冲时 **发送方 goroutine** 与 **读取方 goroutine** 必须 **直接交接**数据：`发送方 park → 读取方 goready → 读取运行`，产生切换成本。
        
    -   **有缓冲且未满** → 发送方把数据写进 `buf`，**不触发任何 goroutine 调度**，直接返回。
        
    -   **有缓冲且为空,存在读取方** → 直接交付，`接收方 goready → 接收方运行`，产生切换成本。
        
    -   **缓冲区满** → 回到“挂起-唤醒”老路，产生切换成本。
        
-   无缓冲延迟低，有缓冲吞吐高；高并发锁竞争会退化。
    

#### **7.3.2 大结构体传指针还是值？**

大于128 B 传指针，减少拷贝 + cache miss。

#### **7.3.3 缓冲区容量 1 为什么容易死锁？**

缓冲区一满立即 `gopark`，接收者若迟到 → 双方挂死。单槽位极易形成“写等读、读等写”循环。

-   **秒记**：一床被子俩人抢，谁先谁后？
    

#### **7.3.4 goroutine 泄漏排查？**

`pprof goroutine` → 查看 `chan receive` / `chan send` 栈，看到大量 G 卡在 `recvq/sendq` 即可定位泄漏根因。

* * *

### **7.4 关闭语义**

#### **7.4.1 关闭后还能读吗？**

读到空返回零值 + ok=false；缓冲区还有数据先读完。

#### **7.4.2 关闭两次会怎样？**

第二次close，会遇到 `if closed != 0 { panic(...) }`，立即 `panic("close of closed channel")`。

#### **7.4.3 多生产者优雅关闭？**

1.  额外 `done chan struct{}` 通知所有写者退出
    
2.  写者退出后 `sync.WaitGroup` 计数归零
    
3.  主 goroutine `close(mainCh)`
    

-   **核心**：确保所有写者退出再关闭，避免 panic。
