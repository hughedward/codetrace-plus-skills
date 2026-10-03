# CodeTrace 输出格式规范

完整示例见 `example-redisson.md`（Redisson 看门狗续期，15 步闭环）。本文是细则和正反例。

## 一、文档头

```markdown
# 追踪示例：`xxx` —— 一句话主线

> - **入口**：项目名 File.java:行
> - **版本**：依赖名 版本号（行号以此版本为准，IDEA ctrl+点击可验证）
> - **主线一句话**：A → B → C → 闭环
> - **约定**：跳转写代码注释里 / pNNNN 编号 / 非线性下方说明来历 / 省略号折叠无关代码
```

## 二、step 结构

- 标题一行（最多两行），格式：`#### ↓NNN、File.java:行号 一句话描述`
  - 例：`#### ↓005、RedissonLock.java:185 一条 Lua 抢锁 + 成功回调`
  - 编号三位数字（001 起），不用 `step_` 前缀；文件名:行号后面可以直接带描述
  - 非线性入口可用第二行副标题：`#### ↓017、RenewalTask.java:170 —— 非线性：从 Netty 跳回 Redisson（当前线程：redisson-timer-1-1）`
- 标题下**不放单独的标题行**，直接上代码块
- 解说一律放代码块下方，不放标题和代码块之间

## 三、跳转注释（唯一的跳转表达方式）

写在代码行内注释里，紧贴那一行，向左的箭头指向代码：

```java
77:          lock(-1, null, false);            // ← 【点击】lock 跳转 本文件:103
105:     Long ttl = tryAcquire(...);           // p0002：【点击】tryAcquire 跳转 本文件:152
73:     renewalScheduler.renewLock(...);       // p0006：【点击】renewLock 跳转 LockRenewalScheduler.java:56
42:     redissonLock.lock();                   // p0001：【点击】lock 跳转 RedissonLock.java:75（先落接口，ctrl+alt+B 进实现）
59:     task.add(name, lockName, threadId);    // p0007：【点击】add 跳转 LockTask.java:105（有两个重载，选 LockTask 自己的）
```

### 反例（禁止）

```markdown
↓【向下】RedissonLock.java:185 的 tryAcquireAsync（同文件）
// 为什么能走到：:157 的 lambda 直接调用了它。
```

线性调用不需要任何"为什么能走到"的解释——点进去就是了。这类话全部删掉，换成紧贴代码的一行注释。

## 四、pNNNN 点击编号

- 只给**关键点击**编号，p0001 起全文递增，写法：`// p0009：★ this = LockTask 自己；10 秒后触发`
- 用途：跨步骤精确回指"哪一次点击/哪一行传出去的东西"，例：

```markdown
来历：p0009（011）传出去的 this 就是这个 LockTask 对象。
```

- 编号可和步骤编号同时出现：`p0009（011）`

## 五、非线性来历块（技能的灵魂）

**原则：非线性 ≠ 断链。** lambda / 回调 / 方法引用 / @Override / 抽象方法实现 / 定时器触发——这些"点不进去"的跳转，必须追成**连续的代码链**：来历说明里的每句话都落到 file:line，能点击的都给【点击】注释，跨库（Netty、JDK 之外）也要用同版本 sources 追进去。

**位置：代码块下方。** 顺序固定为 标题 → 代码块 → 来历块，来历块不得插在标题和代码块之间。

### 反例（禁止）

```markdown
谁调：10 秒后，Netty 时间轮 Worker 线程到期，经 HashedWheelTimeout.expire() → task.run(this) 进来。
```

问题：纯文字描述。"Worker 线程到期"是什么代码？HashedWheelTimeout.expire() 在哪一行？没有锚点 = 背八股。

### 正例（每一跳都有真实代码）

拆成多个 step，逐步点击穿过非线区域：

```markdown
#### ↓013、HashedWheelTimer.java:432 Netty newTimeout：包成 HashedWheelTimeout，入队
（:445 start() 首次懒启动 Worker 线程；:456 new HashedWheelTimeout(this, task, deadline)；:457 timeouts.add 入队）

#### ↓014、HashedWheelTimer.java:482 Worker 线程的巡检循环——"到期"就发生在这里
（:494 waitForNextTick 每 tick 醒一次；:501 bucket.expireTimeouts(deadline) → 【点击】跳 :783）

#### ↓016、HashedWheelTimer.java:698 expire() → taskExecutor → task.run(this)：从 Netty 跳回 Redisson 的那一行
（:705 taskExecutor.execute(this)，Redisson 用 5 参构造器 → 默认 ImmediateExecutor = 同线程；
 :717 task.run(this) ← task 就是 p0009 传进来的 LockTask，ctrl+alt+B → RenewalTask.java:170）
```

### 来历块的三个要素 + 验证锚点

跨步骤回指时在代码块下方给一小段文本：

```markdown
完整来历链（每一跳都能点）：p0009（011）newTimeout(this, 10s)
→ p0010（012）ServiceManager.newTimeout → timer.newTimeout
→ 013 HashedWheelTimer.java:456 包成 HashedWheelTimeout、:457 入队
→ 014 Worker.run():501 每 tick 巡检 → 015 :791 发现到期 → expire()
→ 016 :705 taskExecutor.execute（ImmediateExecutor 同线程）→ :717 task.run(this)
→ 本方法。
验证：在 :170 打断点，加锁后 10 秒内必停，线程名 = redisson-timer-1-1。
```

- **来历**：对象从哪一步传出（pNNNN/步骤编号回指），传递路径每一站有 file:line
- **身份**：实现/重写了谁的什么方法（接口或父类 file:line）
- **触发链**：事件 → 调进目标方法的逐行路径
- **验证**：断点位置 + 预期线程名（如 redisson-timer-1-1、redisson-netty-x）

JDK 内部无法继续下钻时（如 CompletableFuture.whenComplete 同步执行语义），用"语义一句话 + 断点验证线程名"落地，不编造 JDK 行号。

## 六、代码块内部

- 每行带真实行号前缀（`:103` 或 `103:` 风格全文统一）
- 无关代码折叠：`.....`（或保留行号占位），但折叠不能吞掉主线出口行
- 适度补充上下文：如抢锁失败分支，可以在主线 step 里"带一眼"while 循环，配省略号
- 关键处设问并用 `{}` 标注，紧跟答案：

```java
131:     entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);  // ← 在这等待：{会不会一直等？}
                                                                    //    不会——最多等持有者剩余 ttl 毫秒，
                                                                    //    等不到自动醒，回 :122 再抢
```

## 七、解说（代码块下方）

- 只写一两句，服务于追踪主线；太基础的内容（如动态绑定常识）一句带过或不写
- 重要提醒用引用块：`> **⚠️**：一句话 + 行号引用（可在 xxx.java:81 里面看到）`
- 交代关键对象的来龙去脉（如"renewalScheduler 是全局单例，Redisson.java:83"）
- **不留半吊子结论**：结论式的话必须说完整——机制怎么实现、和什么方案对比、代码锚点。要么说完整，要么删掉。

### 反例（半吊子，禁止）

```markdown
（renewalScheduler 是全局单例——看门狗是"一个客户端一个 LockTask"，不是每把锁一个定时器。）
```

问题：说完就完。"怎么做到一个的？"没交代，读者只能背结论。

### 正例（说完整）

```markdown
（"一个客户端一个 LockTask，不是每把锁一个定时器"是怎么做到的：任何一把锁来登记，走的都是
008 :57 的 compareAndSet(null, new LockTask(...))——只有第一次会 new，之后直接复用 reference
里的对象；每把锁只是往这个 LockTask 的 name2entry 登记簿（RenewalTask.java:44 的 Map）里加一条
"锁名 → 持有者"。10 秒后 LockTask 醒来，把登记簿里所有锁名拼进同一条 Lua 批量续期（020 的 KEYS
就是这个登记簿）。对比"每把锁一个定时器"：1000 把锁同时持有，也只有 1 个 Worker 线程、每轮 1 次
Redis 往返，而不是 1000 个 Timeout 定时器。）
```

## 八、收尾三件套

### 全程一条线总结

ASCII 流程图，节点用 step 号，线程接力显式画，闭环用箭头：

```
你的线程（HTTP 请求线程）：
001 lock() → 002/003 ... → 011 newTimeout(this, 10s) ─┐（你的线程返回，去扣库存）
                                                                      │ 10 秒
Netty 时间轮 Worker 线程：────────────────────────────────────────────┘
→ 012 ... → Lua续期 → 回到 011 再上闹钟
        ▲                                              │
        └────────── 每 10 秒循环，直到 unlock 停表 ─────┘
```

### 支线速览

一行一条、带 file:line、注明"面试追问才展开"：

```markdown
#### 支线速览（面试追问才展开）

- **抢锁失败**：subscribe 订阅解锁消息 + while 自旋重试（RedissonLock.java:111-148）
- **unlock 停表**：RedissonBaseLock.java:185 → cancelExpirationRenewal（:76）移出登记簿，最后一个移除后循环自然停
```

### 面试问答（QA）

从主线提炼 6~9 个面试官真会问的问题。答案 2~4 句话、回引编号/行号，**不引入主线没追过的内容**：

```markdown
#### 面试问答（QA）

**Q：`lock(10, TimeUnit.SECONDS)` 这种指定了 leaseTime 的，还有看门狗吗？**
A：没有。005 的 :187 分支：leaseTime > 0 就用你给的值，且 :195-201 的 thenApply 里不会调
scheduleExpirationRenewal——到期自动放，不续期。所以"指定 leaseTime = 主动放弃看门狗"。

**Q：执行续期的到底是哪个线程？**
A：两段：到期回调（017）跑在 redisson-timer-1-1（时间轮 Worker）；续期响应回来后 whenComplete
里的 schedule() 跑在 redisson-netty-x（021）。两个断点都能亲手验证。
```
