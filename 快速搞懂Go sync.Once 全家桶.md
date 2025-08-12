# 快速搞懂Go sync.Once 全家桶

## 1. 它们长啥样

### 1.1 公共底层 sync.Once

```go
type Once struct {
	_ noCopy
	done atomic.Uint32 // 是否已执行过（0/1）
	m    Mutex // 保护慢路径
}
```
- 所有衍生 API 最终都落到这个结构。
- done 用 原子+双重检查 保证“恰好一次”语义。

### 1.2 衍生(Go1.21+)

| 名称             | 签名                                                    | 作用                   |
| -------------- | ----------------------------------------------------- | -------------------- |
| **OnceFunc**   | `func(f func()) func()`                               | 返回一个 **只会执行一次的函数**   |
| **OnceValue**  | `func[T any](f func() T) func() T`                    | 返回一个 **只会计算一次的返回值**  |
| **OnceValues** | `func[T1, T2 any](f func() (T1, T2)) func() (T1, T2)` | 返回一个 **只会计算一次的二返回值** |

三者内部都包了一个 `Once`，把 `f` 包成闭包塞进去。

## 2. 实现原理

### 2.1 Once.Do 的核心流程
```go
func (o *Once) Do(f func()) {
	if o.done.Load() == 0 {
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()
	if o.done.Load() == 0 {
		defer o.done.Store(1)
		f()
	}
}
```

1. **快路径**：原子读 done==1 直接返回；无锁。
2. **慢路径**：只让第一个抢到锁的 `goroutine` 执行 `f`，其余阻塞等待。

- **panic 行为**：若 `f` panic，`done` 仍会被置 `1`；后续任何 `goroutine` 再次调用 `Once.Do` 都会 直接跳过 `f`，**不会重试**。


## 3. OnceFunc / OnceValue / OnceValues 的糖衣

### 3.1 OnceFunc

```go

func OnceFunc(f func()) func() {
	var (
		once  Once
		valid bool // 标记 f 是否“真正成功”跑完，没 panic
		p     any  // 缓存 panic 的值
	)
	// g 只注册给 once.Do，保证全局只跑一次
	g := func() {
		defer func() {
			p = recover()  // 捕获 f 的 panic
			if !valid {    // 如果 f 没跑完需要抛出panic
				panic(p)   // 立刻把 panic 原样抛出，让用户看到完整栈
			}
		}()
		f()
		f = nil      // 把 f 置 nil，断引用，帮助 GC
		valid = true // 只有 f 正常结束才会走到这里
	}
	return func() {  // 返回给调用者的闭包
		once.Do(g)   // 第一次真正跑 g，之后直接跳过
		if !valid {  // 如果第一次 panic 了
			panic(p) // 以后每次调用都复现同一份 panic
		}
	}
}
```

### 3.2 OnceValue

- 大体同`OnceFunc`，但是缓存了返回值

```go
func OnceValue[T any](f func() T) func() T {
	var (
		once   Once
		valid  bool
		p      any
		result T  // 用来缓存 f 的返回值
	)
	g := func() {
		defer func() {
			p = recover()
			if !valid {
				panic(p)
			}
		}()
		result = f()   // 真正计算
		f = nil
		valid = true
	}
	return func() T {
		once.Do(g)
		if !valid {
			panic(p)   // 复现 panic
		}
		return result  // 正常分支：直接返回缓存结果
	}
}
```

### 3.3 OnceValues

- 大体同`OnceValue`，但是缓存了两个返回值

```go
func OnceValues[T1, T2 any](f func() (T1, T2)) func() (T1, T2) {
	var (
		once  Once
		valid bool
		p     any
		r1    T1
		r2    T2
	)
	g := func() {
		defer func() {
			p = recover()
			if !valid {
				panic(p)
			}
		}()
		r1, r2 = f()    // 两个返回值
		f = nil
		valid = true
	}
	return func() (T1, T2) {
		once.Do(g)
		if !valid {
			panic(p)
		}
		return r1, r2
	}
}
```

### 3.4 总结一下

1. **没有重试**：无论 `panic` 还是正常完成，都只执行一次。
2. **panic 缓存**：第一次 `panic` 值被永久保存；后续每次调用都会 复现同一份 panic。

- 速记
| 需求             | 选谁           |
| -------------- | ------------ |
| 只执行一次副作用       | `Once.Do`    |
| 把函数包成“只跑一次”的闭包 | `OnceFunc`   |
| 把函数结果缓存成“只算一次” | `OnceValue`  |
| 同上，但缓存两个返回值    | `OnceValues` |


## 4. 面试题速查

#### 4.1 Once 如何保证并发安全？

原子读 done 做快速路径；慢路径加锁双重检查，确保只有一个 goroutine 执行。

#### 4.2 如果 `f` `panic`，后续调用还会执行 `f` 吗？

不会。无论裸用 `Once.Do` 还是 `OnceFunc` 等包装器，`done` 都会被置 `1`，`f` 只会**执行一次**；后续直接跳过。

#### 4.3 OnceValues 能缓存错误吗？

能，但**不是重试**。函数返回的 `error` 作为第二值被缓存，之后每次调用直接返回该值；如果函数 `panic`，则缓存 `panic` 值并复现。

## 5. 补充

| 场景     | 写法                                        |
| ------ | ----------------------------------------- |
| 单例初始化  | `var init = sync.OnceFunc(singletonInit)`     |
| 全局配置   | `var cfg = sync.OnceValue(loadConfig)`    |
| 双返回值缓存 | `token, err := sync.OnceValues(getToken)` |


