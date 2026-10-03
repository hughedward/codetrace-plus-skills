## 追踪示例：`redissonLock.lock()` —— 从业务入口到看门狗续期闭环

> - **入口**：`redis-lock` 项目 IndexController.java:42
> - **版本**：Redisson 4.7.0 + Netty 4.2.16.Final（行号以此为准，IDEA ctrl+点击可验证；追进依赖源码时用同版本 sources）
> - **主线一句话**：加锁（Lua #1）→ 看门狗登记 → 时间轮入队 → 10 秒后到期回调（Netty 侧逐步点击）→ 续期（Lua #2）→ complete → 再上闹钟，**闭环**
> - **约定**：
>   - 编号三位数字（001 起），写在标题里；正文直接引用编号
>   - 跳转指令写在代码注释里：`【点击】符号 跳转 本文件:行` / `【点击】符号 跳转 文件.java:行`
>   - 关键点击带 `pNNNN` 编号（全文递增），后文用它回指
>   - **非线性 ≠ 断链**：换库、换线程、回调，也要追成连续的代码链（每句话都落到 file:line）
>   - 省略号 = 与主线无关的代码

---

#### ↓001、IndexController.java:42（redis-lock 项目）业务代码里的 lock()

```java
40:  RLock redissonLock = redisson.getLock(lockKey);  // ← getLock 就一行：new RedissonLock(...)（Redisson.java:646）
42:  redissonLock.lock();                             // p0001：【点击】lock 跳转 RedissonLock.java:75
```

（声明类型是接口 RLock，运行时对象是 RedissonLock——ctrl+点击先落接口，ctrl+alt+B 进实现。）

#### ↓002、RedissonLock.java:75 无参 lock() 只是转发

```java
75:  public void lock() {
76:      try {
77:          lock(-1, null, false);   // ← 【点击】lock 跳转 本文件:103（三参重载）
78:      } catch (InterruptedException e) {
79:          throw new IllegalStateException();
80:      }
81:  }
```

> **⚠️**：`-1` 传给 leaseTime：`= -1` 才启用看门狗；`> 0` 时到期自动放、不续期（分支见本文件:187-192）。

#### ↓003、RedissonLock.java:103 发第一枪（失败分支带过）

```java
103: private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
104:     long threadId = Thread.currentThread().getId();
105:     Long ttl = tryAcquire(-1, leaseTime, unit, threadId);  // p0002：【点击】tryAcquire 跳转 本文件:152
                                                             //    null=抢到；数字=别人的锁还剩几毫秒
106:     // lock acquired
107:     if (ttl == null) {
108:         return;   // ← 无竞争：到这直接返回。但"看门狗登记"藏在 tryAcquire 内部回调里（见 005）
109:     }
    .....
111:     CompletableFuture<RedissonLockEntry> future = subscribe(threadId);  // 支线：失败才走——订阅"已解锁"频道
    .....
121:     while (true) {
122:         ttl = tryAcquire(-1, leaseTime, unit, threadId);   // 再抢一次
124:         if (ttl == null) { break; }
128:         // waiting for message
129:         if (ttl >= 0) {
131:             entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);  // ← 在这等待：{会不会一直等？}
                                                                          //    不会——最多等持有者剩余 ttl 毫秒，
                                                                          //    等不到自动醒，回 :122 再抢
    .....
```

#### ↓004、RedissonLock.java:152 两跳纯转发

```java
152: private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
153:     return get(tryAcquireAsync0(waitTime, leaseTime, unit, threadId));  // ← get() 同步等结果，点击 `tryAcquireAsync0`
155: }
156: private RFuture<Long> tryAcquireAsync0(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
157:     return getServiceManager().execute(() -> tryAcquireAsync(waitTime, leaseTime, unit, threadId));
                                    // p0003：【点击】tryAcquireAsync 跳转 本文件:185
```

#### ↓005、RedissonLock.java:185 一条 Lua 抢锁 + 成功回调

```java
185: private RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
187:     if (leaseTime > 0) { ..... } else {
190:         ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
191:                 TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);  // p0004：【点击】tryLockInnerAsync 跳转 本文件:214
192:     }
    .....
195:     CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
197:         if (ttlRemaining == null) {          // ← null = 抢到了
201:             scheduleExpirationRenewal(threadId);  // p0005：【点击】scheduleExpirationRenewal
                                                     //    跳转 RedissonBaseLock.java:72（在父类里）
```

> **⚠️**：`internalLockLeaseTime` 来自构造器（本文件:61）= lockWatchdogTimeout，默认 30000ms（Config.java:81）。

#### ↓006、RedissonLock.java:214 Lua #1：抢锁

```java
214: <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, ...) {
215:     return evalWriteSyncedNoRetryAsync(getRawName(), LongCodec.INSTANCE, command,
216:         "if ((redis.call('exists', KEYS[1]) == 0) " +                      // 锁不存在？
217:                     "or (redis.call('hexists', KEYS[1], ARGV[2]) == 1)) then " +  // 或：本来就是我持有的（可重入）
218:             "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +               // 重入计数 +1
219:             "redis.call('pexpire', KEYS[1], ARGV[1]); " +                  // ★ 设 30s 过期
220:             "return nil; " +                                               // nil = 抢到
221:         "end; " +
222:         "return redis.call('pttl', KEYS[1]);",                             // 没抢到：返回剩余毫秒
223:         Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
```

（Lua = 判断+写入原子。hash 结构怎么支持可重入：字段名 = `UUID:threadId`，value = 重入计数——同线程再 lock 就 hincrby +1（:218），unlock 时 -1，减到 0 才删锁。ARGV[1]=30000，ARGV[2]=持有者。）

线索回收：返回 nil → 005 的 thenApply（:195-201）拿到 null → 调 p0005 指向的 scheduleExpirationRenewal。

#### ↓007、RedissonBaseLock.java:72 看门狗登记：锁对象 → 全局调度器

```java
72: protected void scheduleExpirationRenewal(long threadId) {
73:     renewalScheduler.renewLock(getRawName(), threadId, getLockName(threadId));
                                     // p0006：【点击】renewLock 跳转 LockRenewalScheduler.java:56
74: }
```

（renewalScheduler 是 RedissonBaseLock.java:55 的字段，客户端启动时注册的全局单例（Redisson.java:83）。**"一个客户端一个 LockTask，不是每把锁一个定时器"是怎么做到的**：任何一把锁来登记，走的都是 008 :57 的 `compareAndSet(null, new LockTask(...))`——只有第一次会 new，之后直接复用 reference 里的对象；每把锁只是往这个 LockTask 的 `name2entry` 登记簿（RenewalTask.java:44 的 Map）里加一条"锁名 → 持有者"。10 秒后 LockTask 醒来，把登记簿里**所有**锁名拼进同一条 Lua 批量续期（020 的 KEYS 就是这个登记簿）。对比"每把锁一个定时器"：1000 把锁同时持有，也只有 1 个 Worker 线程、每轮 1 次 Redis 往返，而不是 1000 个 Timeout 定时器。）

#### ↓008、LockRenewalScheduler.java:56 懒汉式单例：拿到/创建全局唯一的 LockTask

```java
56: public void renewLock(String name, Long threadId, String lockName) {
57:     reference.compareAndSet(null, new LockTask(internalLockLeaseTime, executor, batchSize));  // ← 记住这个 new LockTask：
                                                                                                //    019/020 的主角
58:     LockTask task = reference.get();
59:     task.add(name, lockName, threadId);  // p0007：【点击】add 跳转 LockTask.java:105
                                            //    （add 有两个重载，选 LockTask 自己的）
60: }
```

#### ↓009、LockTask.java:105 包一个 LockEntry 再交给父类

```java
105: public void add(String rawName, String lockName, long threadId) {
106:     LockEntry entry = new LockEntry();
107:     entry.addThreadId(threadId, lockName);   // 记下持有者标识（UUID:threadId）
109:     add(rawName, lockName, threadId, entry);  // ← 【点击】add 跳转父类 RenewalTask.java:136
110: }
```

#### ↓010、RenewalTask.java:136 第一次登记 → 上第一个闹钟

```java
136: final void add(String rawName, String lockName, long threadId, LockEntry entry) {
137:     name2entry.compute(rawName, (k, oldEntry) -> {   // ← 登记簿：锁名 → 持有者集合（本文件:44）
    .....
144:         } else {                                     // 这把锁第一次登记
145:             if (tryRun()) {   // ← CAS(false→true)：并发加锁时只有一个线程成功去定闹钟
146:                 schedule();   // p0008：【点击】schedule 跳转 本文件:62
147:             }
148:         }
```

#### ↓011、RenewalTask.java:62 把 this 交给时间轮，你的线程收工

```java
62: public void schedule() {
63:     if (!running.get()) {   // 已 stop（没有锁要续了）→ 不再上闹钟
64:         return;
65:     }
67:     long internalLockLeaseTime = ...getLockWatchdogTimeout();  // 30000
68:     executor.getServiceManager().newTimeout(this, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
                                  // p0009：★ this = LockTask 自己；30000/3 = 10 秒后触发
                                  //    【点击】newTimeout 跳转 ServiceManager.java:299（跨包，见 012）
```

> **⚠️**：为什么 /3——30s 有效期每 10s 续一次，某次失败还剩 2 次补救机会。

#### ↓012、ServiceManager.java:299（org.redisson.connection 包）Redisson 转交给 Netty 时间轮

```java
299: public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
301:         return timer.newTimeout(task, delay, unit);  // p0010：【点击】newTimeout 跳转 HashedWheelTimer.java:432
                                                          //    （io.netty.util 包，Netty 源码）
    .....
138: private HashedWheelTimer timer;   // ← 字段声明：就是 Netty 的时间轮
293: timer = new HashedWheelTimer(new DefaultThreadFactory("redisson-timer"),
294:         minTimeout, TimeUnit.MILLISECONDS, 1024, false);
        // ↑ 时间轮是 Redisson 启动时自己 new 的：Worker 线程名前缀 "redisson-timer"（断点验证用），
        //   tickDuration = minTimeout（50 或 100ms，本文件:289-292），1024 个槽
```

#### ↓013、HashedWheelTimer.java:432（io.netty.util，Netty 4.2.16.Final）包成 HashedWheelTimeout，入队

```java
432: public Timeout newTimeout(TimerTask task, long delay, TimeUnit unit) {
    .....
445:     start();   // ← 第一次调用才启动 Worker 线程（懒启动，线程名 redisson-timer-1-1）；
                 //   启动后它就进入 014 的 run() 循环，直到客户端 shutdown
    .....
456:     HashedWheelTimeout timeout = new HashedWheelTimeout(this, task, deadline);
        //                          ↑ 把 p0009 传进来的 LockTask（this）包进 HashedWheelTimeout
457:     timeouts.add(timeout);   // ← 先进队列，下一个 tick 才被搬进轮槽（见 014 的 transferTimeoutsToBuckets）
458:     return timeout;
```

你的业务线程到这就返回了。**接下来 10 秒发生的事，全部能点击跟踪：**

#### ↓014、HashedWheelTimer.java:482 Worker 线程的巡检循环——"到期"就发生在这里

```java
476: private final class Worker implements Runnable {
    .....
482:     public void run() {          // ← 012 的 :293 DefaultThreadFactory 创建的线程跑的就是它
    .....
494:             final long deadline = waitForNextTick();   // ← 每 tick（50~100ms）醒一次，这就是"时间轮在走"
    .....
500:                     transferTimeoutsToBuckets();       // ← 把 :457 队列里的任务搬进轮槽
501:                     bucket.expireTimeouts(deadline);    // p0011：【点击】expireTimeouts 跳转 本文件:783
    .....
        } while (WORKER_STATE_UPDATER.get(HashedWheelTimer.this) == WORKER_STATE_STARTED);
```

#### ↓015、HashedWheelTimer.java:783 巡检发现 deadline 到了 → expire()

```java
783: public void expireTimeouts(long deadline) {
         HashedWheelTimeout timeout = head;
         while (timeout != null) {
    .....
789:             if (timeout.remainingRounds <= 0) {
790:                 if (timeout.deadline <= deadline) {
791:                     timeout.expire();     // p0012：【点击】expire 跳转 本文件:698（HashedWheelTimeout 内部类）
    .....
```

#### ↓016、HashedWheelTimer.java:698 expire() → taskExecutor → task.run(this)：从 Netty 跳回 Redisson 的那一行

```java
698:         public void expire() {
699:             if (!compareAndSetState(ST_INIT, ST_EXPIRED)) { return; }
704:             remove();
705:             timer.taskExecutor.execute(this);
                // ↑ 注意：不是直接调 task！是把 this（HashedWheelTimeout，它也是 Runnable）交给 taskExecutor
                //   Redisson 用 5 参构造器（ServiceManager.java:293），默认 taskExecutor = ImmediateExecutor.INSTANCE
                //   （本文件:252），即"同线程直接 run"——所以下一步还在 redisson-timer 线程
    .....
715:         public void run() {
717:                 task.run(this);   // p0013：★ task 就是 p0009（011）传进来、:456 被包了一层的 LockTask
                                    //   【点击】task.run 落在接口 TimerTask.run 上，ctrl+alt+B 找实现 → RenewalTask.java:170
```

> **⚠️**：老版本 Netty 是 expire() 直接调 task.run；4.2 中间隔了一层 taskExecutor——这就是"落实到代码、不背八股"的意义。

#### ↓017、RenewalTask.java:170 —— 非线性：从 Netty 跳回 Redisson（当前线程：redisson-timer-1-1）

```java
169: @Override
170: public void run(Timeout timeout) {
171:     if (executor.getServiceManager().isShuttingDown()) {
172:         return;
175:     CompletionStage<Void> future = execute();          // p0014：【点击】execute 跳转 本文件:78
176:     future.whenComplete((result, e) -> {               // ← 谁来调这个回调？见 021（不是白等，能追到代码）
183:         schedule();   // ← 再上闹钟 → 回到 011（p0008）：闭环的后半段
184:     });
185: }
```

```
完整来历链（每一跳都能点）：p0009（011）newTimeout(this, 10s)
→ p0010（012）ServiceManager.newTimeout → timer.newTimeout
→ 013 HashedWheelTimer.java:456 包成 HashedWheelTimeout、:457 入队
→ 014 Worker.run():501 每 tick 巡检 → 015 :791 发现到期 → expire()
→ 016 :705 taskExecutor.execute（ImmediateExecutor 同线程）→ :717 task.run(this)
→ 本方法。task 的类型是 io.netty.util.TimerTask（本文件:35 implements），run 是 @Override（:169）。
验证：在 :170 打断点，加锁后 10 秒内必停，线程名 = redisson-timer-1-1。
```

#### ↓018、RenewalTask.java:78 非集群模式：登记簿里的锁名全交给 renew

```java
78: final CompletionStage<Void> execute() {
79:     if (name2entry.isEmpty()) {                          // 登记簿空了（都解锁了）
80:         return CompletableFuture.completedFuture(null);  // → 不续也不再 schedule，循环自然停
83:     if (!executor.getServiceManager().isClusterSetup()) {
84:         return renew(name2entry.keySet().iterator(), chunkSize);
                     // ← renew 是本类声明的抽象方法（本文件:95），实现在子类 LockTask.java:41
                     //   ——就是 p0007（008）那个 new LockTask
```

#### ↓019、LockTask.java:41 renew → buildChunk：批量切分

```java
40: @Override
41: CompletionStage<Void> renew(Iterator<String> iter, int chunkSize) {
42:     return AsyncChunkProcessor.processAll(iter, chunkSize, this::buildChunk);
                                 // ← 【点击】buildChunk 跳转 本文件:45（每切一批锁名回调一次）
43: }
```

#### ↓020、LockTask.java:82 Lua #2：续期本体

```java
82: CompletionStage<List<String>> f = executor.syncedEval(firstName, LongCodec.INSTANCE, ...,
84:     "local result = {} " +
85:         "for i = 1, #KEYS, 1 do " +
86:             "if (redis.call('hexists', KEYS[i], ARGV[i + 1]) == 1) then " +  // 锁还归这个线程？
87:                 "redis.call('pexpire', KEYS[i], ARGV[1]); " +                // ★ 还归你 → 重置 30s
88:                 "table.insert(result, 1); " +
89:             "else ... table.insert(result, 0); ... " +                       // 不归你 → 记 0，回头注销
92:         "end; " +
93:     "return result;",
94:     new ArrayList<>(keys),   // ← KEYS = 登记簿里的锁名们
95:     args.toArray());         // ← ARGV[1]=30000，ARGV[2..]=各锁持有者标识
```

（续期前必须 hexists 校验——锁被误删后别人加了新锁，绝不能把**别人的锁**续活；返回 0 的锁在回调里注销，本文件:97-102。）

Lua 发出后当前线程**不阻塞等待**，run() 返回。剩下的疑问：**017 :176 的 whenComplete 回调，到底谁在哪个线程调？**——继续追，不断链：

#### ↓021、RedisExecutor.java:711（org.redisson.command 包）—— 非线性：闭环点，complete → whenComplete → schedule()

```java
711: protected void handleSuccess(CompletableFuture<R> promise, CompletableFuture<RedisConnection> connectionFuture, R res) {
713:             promise.complete(res);   // ★ future 在这被完成——当前线程是 Redis 连接的 Netty eventLoop（redisson-netty-x）
    .....
```

```
来历链：syncedEval 发出 Lua 后返回的 future，在 Redis 响应回到连接时被 complete：
Redis 连接的 Netty eventLoop 线程（redisson-netty-x）读到响应、解码后 complete attemptPromise
→ attemptPromise.whenComplete（RedisExecutor.java:215）→ checkAttemptPromise（:574）
→ handleResult（:673）→ handleSuccess（:711）→ 上面那行。
```

JDK `CompletableFuture.whenComplete` 的语义：**完成时，回调同步在调用 complete() 的线程里执行**。所以 017 :176 的 whenComplete → :183 `schedule()` 跑在 redisson-netty 线程——再上 10 秒的闹钟，10 秒后 redisson-timer-1-1 再次到期，一切重演。

> 验证：在 RenewalTask.java:183 `schedule()` 打断点，续期完成后停下的线程名是 redisson-netty-x，不是 redisson-timer。

**闭环达成：011 →（Netty 五步）→ 017 → 020 → 021 → 回到 011。**

---

#### 全程一条线总结

```
你的线程（HTTP 请求线程）：
001 lock() → 002/003 lock(-1,null,false) → 004 tryAcquire
→ 005/006 Lua抢锁(exists+hincrby+pexpire 30s) → 成功返回 nil
→ 007 scheduleExpirationRenewal → 008/009 登记进全局 LockTask
→ 010 首次登记 tryRun() 成功 → 011 newTimeout(this,10s)
→ 012 ServiceManager.newTimeout ─┐（入队即返回，去扣库存）
                                   │
redisson-timer-1-1（Netty 时间轮 Worker）：│ 10 秒（每 50~100ms 巡检一次）
→ 013 :457 入队 → 014 Worker.run() 巡检 ─┘
→ 015 expireTimeouts → expire() → 016 task.run(this)
→ 017 RenewalTask.run() → 018 execute() → 019 renew/buildChunk
→ 020 Lua续期(hexists+pexpire 重置30s) → 发出后不等待，run() 返回
                                   │ Redis 响应回到连接
redisson-netty（Redis 连接的 eventLoop 线程）：│
→ 021 handleSuccess → promise.complete(...) ─┘
→ whenComplete 同步执行 → schedule()（=011 再上 10s 闹钟）
        ▲                                                      │
        └──────────── 每 10 秒循环一轮，直到 unlock 停表 ────────┘
```

#### 支线速览（面试追问才展开）

- **抢锁失败**：003 已展示——subscribe 订阅解锁消息 + while 自旋重试（RedissonLock.java:111-148）
- **unlock 停表**：RedissonBaseLock.java:185 `unlock()` → cancelExpirationRenewal（:76）把锁移出登记簿；最后一个锁移除后 stop()（RenewalTask.java:58），011 的 :63 判断让闹钟不再上，循环自然终止
- **可重入**：006 的 hincrby +1，对应 unlock 时 hincrby -1，减到 0 才删锁
- **时间轮原理**：tick/槽位/remainingRounds 细节（HashedWheelTimer.java:494-540），主线只用了"入队 → 巡检 → 到期"

#### 面试问答（QA）

**Q1：`lock()` 之后锁的默认有效期是多久？业务执行超过这个时间，锁会被别人抢走吗？**
A：默认 30s（Config.java:81 lockWatchdogTimeout）。不会被抢：不传 leaseTime 时看门狗每 10s 把它续回 30s（011 的 /3，020 的 pexpire），直到 unlock。只有客户端进程宕机（续不了期）才会等 30s 自然过期释放。

**Q2：那 `lock(10, TimeUnit.SECONDS)` 这种指定了 leaseTime 的，还有看门狗吗？**
A：没有。005 的 :187 分支：leaseTime > 0 就用你给的值，且 :195-201 的 thenApply 里**不会**调 scheduleExpirationRenewal——到期自动放，不续期。所以"指定 leaseTime = 主动放弃看门狗"。

**Q3：看门狗是每把锁一个定时器吗？**
A：不是。一个客户端一个全局 LockTask：008 的 `compareAndSet(null, new LockTask(...))` 只在第一次创建，之后所有锁都往它的 name2entry 登记簿（RenewalTask.java:44）加条目；10 秒醒来一轮，把登记簿里**所有**锁名拼进同一条 Lua 批量续期（020 的 KEYS）。1000 把锁同时持有 = 1 个 Worker 线程 + 每轮 1 次 Redis 往返。

**Q4：看门狗什么时候启动、什么时候停？**
A：启动是懒的：第一把锁首次登记时 010 :145 `tryRun()` CAS 成功才 `schedule()` 上第一个闹钟（时间轮 Worker 也是第一次 newTimeout 时 :445 才 start）。停止：unlock 把锁移出登记簿，最后一把移除后 `stop()`（RenewalTask.java:58）把 running 置 false，011 :63 的判断让闹钟不再上，循环自然终止；登记簿空了 018 :79-80 也直接短路。

**Q5：续期前为什么必须 hexists 校验？**
A：020 的 :86——如果业务没执行完、锁却被误删（比如别的客户端误操作 DEL）后别人加上了同名的锁，hexists 发现字段不归你，返回 0、不 pexpire，回调里把这把锁从登记簿注销（:97-102）。绝不能把**别人的锁**续活。

**Q6：可重入是怎么实现的？**
A：锁的 Redis 结构是 hash：字段 = `UUID:threadId`，value = 重入计数。同线程再 lock：006 :218 hincrby +1；unlock：hincrby -1，减到 0 才 DEL 锁+发解锁消息。不同线程/客户端字段名不同，互斥。

**Q7：抢锁失败后线程在干嘛？会一直等吗？**
A：003 的 :111 先订阅"已解锁"频道，然后 while 自旋：再抢一次，没抢到就在 Semaphore 上 `tryAcquire(ttl)` 睡**最多持有者剩余的 ttl 毫秒**（:131），等不到自动醒回 :122 再抢；持有者 unlock 时 publish 解锁消息会立刻唤醒它。不会无限等。

**Q8：执行续期的到底是哪个线程？**
A：两段：到期回调（017 run()）跑在 **redisson-timer-1-1**（Netty 时间轮 Worker 线程，012 :293 起的名字，ImmediateExecutor 同线程执行，见 016）；续期 Lua 的响应回来后，whenComplete 里的 schedule()（017 :183）跑在 **redisson-netty-x**（Redis 连接的 eventLoop，JDK whenComplete 语义：同步在 complete 线程执行，见 021）。两个断点都能亲手验证。

**Q9：为什么加锁和续期都必须用 Lua？**
A：加锁 006：`exists 判断 + hincrby + pexpire` 三步必须原子，拆开就是两个客户端都能通过的竞态；续期 020：`hexists 校验 + pexpire` 同理。Redis 单线程执行 Lua 期间不会插入其他命令。
