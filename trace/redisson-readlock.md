# 追踪示例：`rLock.lock()`（读写锁之读锁）—— 读读共享、看门狗 ReadLockTask 续期闭环

> - **入口**：`redis-lock` 项目 ProductService.java:89（`get()` 方法里的 DCL 双层锁内层）
> - **版本**：Redisson 4.7.0（行号以此版本为准，IDEA ctrl+点击可验证）
> - **主线一句话**：读锁 Lua（mode-hash + 读者序号 key）→ 看门狗登记（全局 ReadLockTask）→ 10 秒后 Netty 回调 → 读锁续期 Lua → 再上闹钟，**闭环**
> - **约定**：跳转写代码注释里（【点击】）；关键点击带 pNNNN，后文回指；非线性在代码块下方交代来历；`.....` = 无关代码
> - **与前剧的关系**：写锁剧本（example-redisson.md）已详展的公共链路（三参 lock / tryAcquire 转发 / Netty 时间轮五连 / RedisExecutor 完成链）本剧收拢带过，行号同版本通用

---

#### ↓001、ProductService.java:89（redis-lock 项目）业务入口：读锁上锁

```java
87:  RReadWriteLock readWriteLock = redisson.getReadWriteLock(LOCK_PRODUCT_UPDATE_PREFIX + productId);
                    // ↑ getReadWriteLock 就一行：new RedissonReadWriteLock(...)（Redisson.java:730）
88:  RLock rLock = readWriteLock.readLock();
                    // ↑ readLock() 也就一行：new RedissonReadLock(...)（RedissonReadWriteLock.java:45）
                    //   —— 对象类型在这两行就定了：RedissonReadLock（后文 p0003/p0004 分派的伏笔）
89:  rLock.lock();  // p0001：【点击】lock 跳转 RedissonLock.java:75
```

（场景：:77-78 先拿 hot_cache 单飞锁、查缓存未命中，再拿读锁去库里取数——与 update() 的写锁互斥。）

#### ↓002、RedissonLock.java:75 lock()：读锁把"锁壳"整个借走了

```java
75:  public void lock() {
77:          lock(-1, null, false);   // ← 【点击】lock 跳转 本文件:103（三参重载）
78:      } catch (InterruptedException e) {
```

（RedissonReadLock **没有** override `lock()`（RedissonReadLock.java 全文 191 行里没有它）——壳全部复用写锁的：加锁流程、自旋重试、订阅唤醒一模一样。**不一样的只有两颗心脏**：抢锁 Lua 和续期，见 p0003/p0004。）

#### ↓003、RedissonLock.java:103 三参 lock：发第一枪（失败自旋用省略号带过）

```java
103: private void lock(long leaseTime, TimeUnit unit, boolean interruptibly) throws InterruptedException {
105:     Long ttl = tryAcquire(-1, leaseTime, unit, threadId);  // p0002：【点击】tryAcquire 跳转 本文件:152
                                                             //    null=抢到；数字=还得等多久
107:     if (ttl == null) { return; }   // ← 读模式下几乎必返回 nil（读读共享）
    .....
121:     while (true) {                       // 支线：被写者挡住才走——订阅 + 自旋重试
131:         entry.getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);  // ← 最多等写锁剩余 ttl，不会死等
    .....
```

#### ↓004、RedissonLock.java:152 两跳纯转发

```java
152: private Long tryAcquire(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
153:     return get(tryAcquireAsync0(waitTime, leaseTime, unit, threadId));  // ← get() 同步等结果
156: private RFuture<Long> tryAcquireAsync0(...) {
157:     return getServiceManager().execute(() -> tryAcquireAsync(...));   // 【点击】tryAcquireAsync 跳转 本文件:185
```

#### ↓005、RedissonLock.java:185 —— 非线性①：同一次点击，进的是读锁的 Lua

```java
185: private RFuture<Long> tryAcquireAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
187:     if (leaseTime > 0) { ..... } else {
190:         ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
191:                 TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_LONG);  // p0003：★ 点击 tryLockInnerAsync
                                                                            //   不会落在下面的 :214 默认实现！
    .....
195:     CompletionStage<Long> f = ttlRemainingFuture.thenApply(ttlRemaining -> {
197:         if (ttlRemaining == null) {
201:             scheduleExpirationRenewal(threadId);  // p0004：【点击】scheduleExpirationRenewal
                                                     //   也不会落在父类！见 007
```

```
来历：001 的 :88 new RedissonReadLock(...)——对象从出生就是读锁。
身份：tryLockInnerAsync 在 RedissonLock.java:214 有默认实现（写锁 Lua），但 RedissonReadLock
     重写了它（RedissonReadLock.java:56，@Override 在 :55）。
分派：:190 的调用隐含 this. → 动态绑定按运行时类型（RedissonReadLock）→ 进 :56。
IDEA 操作：ctrl+点击 tryLockInnerAsync 会给出多个候选，选 RedissonReadLock 那个；
          或在 :190 打断点，step-into 直接落进读锁的 Lua。
```

#### ↓006、RedissonReadLock.java:56 Lua #1：读锁抢锁——mode-hash + 读者序号 key

```java
56:  <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, ...) {
57:      return commandExecutor.syncedEvalNoRetry(getRawName(), LongCodec.INSTANCE, command,
58:          "local mode = redis.call('hget', KEYS[1], 'mode'); " +        // 主 hash 里存着模式：read/write
59:          "if (mode == false) then " +                                  // 没人持锁？
60:              "redis.call('hset', KEYS[1], 'mode', 'read'); " +         // 置为读模式
61:              "redis.call('hset', KEYS[1], ARGV[2], 1); " +             // 我这个读者计数 1
62:              "redis.call('set', KEYS[2] .. ':1', 1); " +               // ★ 独立 key：…:rwlock_timeout:1
63:              "redis.call('pexpire', KEYS[2] .. ':1', ARGV[1]); " +
64:              "redis.call('pexpire', KEYS[1], ARGV[1]); " +             // 主 hash 也 30s
65:              "return nil; " +                                          // 抢到
66:          "end; " +
67:          "if (mode == 'read') or (mode == 'write' and redis.call('hexists', KEYS[1], ARGV[3]) == 1) then " +
68:              "local ind = redis.call('hincrby', KEYS[1], ARGV[2], 1); " +   // 我是第 ind 个重入读者
69:              "local key = KEYS[2] .. ':' .. ind;" +
70:              "redis.call('set', key, 1); " +                                // 每个序号一个 timeout key
72:              "local remainTime = redis.call('pttl', KEYS[1]); " +
73:              "redis.call('pexpire', KEYS[1], math.max(remainTime, ARGV[1])); " + // 主锁取最大剩余
74:              "return nil; " +
75:          "end;" +
76:          "return redis.call('pttl', KEYS[1]);",                        // 被写者挡住：返回剩余时间
77:          Arrays.asList(getRawName(), getReadWriteTimeoutNamePrefix(threadId)),
78:          unit.toMillis(leaseTime), getLockName(threadId), getWriteLockName(threadId));
            // ARGV[1]=30000, ARGV[2]=UUID:threadId（读者名）, ARGV[3]=UUID:threadId:write（写者名）
```

> **⚠️**：网上老八股说 Redisson 读锁用 zset 按时间戳排序淘汰读者——**4.7.0 不是**！是 mode 字段 + 读者计数字段 + 每序号独立 timeout key。追代码，别背旧文。
>
> **⚠️**：:67 的 `mode == 'write' and hexists(ARGV[3])`——**持有写锁的人可以再加读锁**（锁降级的入口）；纯读之间（mode=='read'）永不互斥。

（**电商例子看 Redis 里到底存了什么**：商品 101，客户端 UUID 简写 `c604`，线程 88/99。`KEYS[1]`=`lock:product:update:101`（你的前缀+商品ID）；`KEYS[2]`=`{lock:product:update:101}:{c604:88}:rwlock_timeout`（`suffixName` 拼接，RedissonObject.java:146-150），`:62` 拼 `':1'` 后即完整沙漏 key。第一次 lock 后：主 hash 两笔 `HSET mode→read`、`HSET c604:88→1` + 整体 30s；外加 `SET ...:rwlock_timeout:1` 独立 30s。线程 99 也来读：hash 加 field `c604:99→1` + 新沙漏 `...:99:rwlock_timeout:1`；线程 88 重入：`HINCRBY c604:88→2` + 新沙漏 `...:88:rwlock_timeout:2`。**为什么两套结构**：Redis 的 hash 不能给单个 field 设 TTL——hash 当登记簿（加锁判定 :67 查它），string key 当每个读者序号的沙漏（unlock :107 按序号 DEL、:116 逐沙漏 Pttl 取 max 收敛主锁；看门狗 ReadLockTask.java:104 逐序号续、:110 任一活着才续主 hash）。）

线索回收：返回 nil → 005 的 thenApply（:195-201）→ 调 p0004 指向的 scheduleExpirationRenewal。

#### ↓007、RedissonReadLock.java:147 —— 非线性②：看门狗登记也被读锁重写了

```java
147: protected void scheduleExpirationRenewal(long threadId) {
148:     String timeoutPrefix = getReadWriteTimeoutNamePrefix(threadId);   // …:rwlock_timeout 前缀
149:     String keyPrefix = getKeyPrefix(threadId, timeoutPrefix);         // 锁名前缀（去掉 UUID 段）
150:     renewalScheduler.renewReadLock(getRawName(), threadId, getLockName(threadId), keyPrefix);
                                        // p0005：【点击】renewReadLock 跳转 LockRenewalScheduler.java:44
151: }
```

```
来历：RedissonLock.java:201 调用的 scheduleExpirationRenewal 在父类 RedissonBaseLock.java:72 有默认实现
     （写锁版，前剧 007）；但对象是 001 :88 的 RedissonReadLock，它重写了本方法（@Override :146）。
分派：:201 隐含 this. → 进读锁版。差异：多传 keyPrefix（续期 Lua 要拼 timeout key 用）。
```

#### ↓008、LockRenewalScheduler.java:44 读锁有自己的懒汉单例：ReadLockTask

```java
31:  private final AtomicReference<FastMultilockTask> multilockReference = new AtomicReference<>();
32:  private final AtomicReference<ReadLockTask> readLockReference = new AtomicReference<>();   // ← 读锁专用的引用
    .....
44:  public void renewReadLock(String name, Long threadId, String lockName, String keyPrefix) {
45:      readLockReference.compareAndSet(null, new ReadLockTask(internalLockLeaseTime, executor, batchSize));
                                        // ← 记住这个 new ReadLockTask：014 分派的伏笔；与写锁的
                                        //   reference(:33)/LockTask 互不相干，两套全局单例并存
46:      ReadLockTask task = readLockReference.get();
47:      task.add(name, lockName, threadId, keyPrefix);   // p0006：【点击】add 跳转 ReadLockTask.java:134
                                                          //    （四参版，选 ReadLockTask 的）
48:  }
```

> **⚠️**：看门狗不是"一个客户端一个任务"，而是**一类锁一个全局任务**：写锁 LockTask（reference）、读锁 ReadLockTask（readLockReference）、联锁 FastMultilockTask（multilockReference）——三套 AtomicReference 各管各的（LockRenewalScheduler.java:31-33）。

#### ↓009、ReadLockTask.java:134 四参 add：登记进读锁登记簿

```java
134: public void add(String rawName, String lockName, long threadId, String keyPrefix) {
136:     addSlotName(rawName);
138:     ReadLockEntry entry = new ReadLockEntry();          // ← 读锁版条目（比 LockEntry 多存 keyPrefix）
139:     entry.addThreadId(threadId, lockName, keyPrefix);
141:     ReadLockEntry oldEntry = (ReadLockEntry) name2entry.putIfAbsent(rawName, entry);
142:     if (oldEntry != null) { oldEntry.addThreadId(threadId, lockName, keyPrefix); }
143:     } else {
144:         if (tryRun()) {          // ← CAS：这把锁第一个登记的人才定闹钟
145:             schedule();           // 【点击】schedule 跳转父类 RenewalTask.java:62
```

#### ↓010、RenewalTask.java:62 schedule()：把 this 交给时间轮（与写锁一字不差）

```java
62: public void schedule() {
67:     long internalLockLeaseTime = ...getLockWatchdogTimeout();   // 30000
68:     executor.getServiceManager().newTimeout(this, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
                                  // p0007：★ this = ReadLockTask 自己；30000/3 = 10 秒后触发
                                  //    业务线程到此返回
```

#### ↓011、HashedWheelTimer.java:432 —— 非线性③：10 秒后时间轮到期回调（前剧已详展，此处收拢）

```
完整来历链（每跳可点，同版本行号）：p0007（010）newTimeout(this, 10s)
→ ServiceManager.java:299 → timer.newTimeout
→ HashedWheelTimer.java:456 包成 HashedWheelTimeout、:457 入队
→ :482 Worker.run() 每 tick（50~100ms）巡检 → :501 expireTimeouts → :791 到期 → :698 expire()
→ :705 taskExecutor.execute（ImmediateExecutor 同线程）→ :717 task.run(this)
→ 下一方法。task 类型 io.netty.util.TimerTask，ReadLockTask 经继承链最终实现它。
验证：在 RenewalTask.java:170 打断点，加锁后 10 秒内必停，线程名 = redisson-timer-1-1。
```

#### ↓012、RenewalTask.java:170 run()：续期循环入口（当前线程：redisson-timer-1-1）

```java
169: @Override
170: public void run(Timeout timeout) {
175:     CompletionStage<Void> future = execute();       // p0008：【点击】execute 跳转 本文件:78
176:     future.whenComplete((result, e) -> {
183:         schedule();   // ← 闭环后半段：再上 10s 闹钟 → 回到 010
```

#### ↓013、RenewalTask.java:78 execute()：登记簿交给 renew——这次进的是读锁实现

```java
78: final CompletionStage<Void> execute() {
83:     if (!executor.getServiceManager().isClusterSetup()) {
84:         return renew(name2entry.keySet().iterator(), chunkSize);
                     // ← renew 抽象（本文件:95）→ 实现在 ReadLockTask.java:39
                     //   ——008 :45 new 的那个 ReadLockTask（同 018 的分派道理：this 决定一切）
```

#### ↓014、ReadLockTask.java:39 renew → buildChunk：批量切分

```java
39:  @Override
40:  CompletionStage<Void> renew(Iterator<String> iter, int chunkSize) {
41:      return AsyncChunkProcessor.processAll(iter, chunkSize, this::buildChunk);
                                     // ← 【点击】buildChunk 跳转 本文件:44（每批锁名回调一次）
```

#### ↓015、ReadLockTask.java:91 Lua #2：读锁续期——anyAlive 判定 + 逐序号续 key

```java
91:  CompletionStage<List<Object>> f = executor.syncedEval(firstName, LongCodec.INSTANCE, ...,
93:    "local result = {} " +
94:      "local argIdx = 2 " +
95:      "for i = 1, #KEYS, 2 do " +                       // 每锁两个 KEY：锁名 + keyPrefix（:89 keysArgs 成对加入）
96:          "local anyAlive = false; " +
98:          "for k = 1, lockNamesCount do " +             // 该锁的每个持有者
99:              "local counter = redis.call('hget', KEYS[i], ARGV[argIdx]); " +  // 这读者还登记着？
101:                 "anyAlive = true; " +
102:                 "for c=counter, 1, -1 do " +          // 它的每个重入序号
104:                     "redis.call('pexpire', KEYS[i+1] .. ':' .. ARGV[argIdx] .. ':rwlock_timeout:' .. c, ARGV[1]); " +
                             // ★ 逐序号续 timeout key 30s
105:                 "end; " +
    .....
109:          "if (anyAlive) then " +
110:              "redis.call('pexpire', KEYS[i], ARGV[1]); " +   // ★ 任一读者活着 → 主 hash 也续 30s
111:              "table.insert(result, 1); " +
112:          "else ... table.insert(result, 0); ... " +           // 全走了 → 记 0，回头注销
```

（与写锁续期（前剧 020）的差别：写锁只查一个 hexists、续一个 key；读锁要**遍历所有读者 × 所有重入序号**逐个续，且主锁"只要还有一个读者活着就续"。返回 0 的锁在回调里 cancelExpirationRenewal 注销，ReadLockTask.java:120-131。）

续期完成 → 012 :176 的 whenComplete → :183 schedule()（=010）→ 10 秒后一切重演。

#### ↓016、RedisExecutor.java:713 —— 非线性④：闭环点（与写锁同一条完成链）

```
来历链（同版本行号，前剧已逐跳展开）：:201 sendCommand 异步发出 → 不等待
→ Redis 响应回到连接 → redisson-netty 线程 complete attemptPromise（:141）
→ :215 预注册 whenComplete → :574 checkAttemptPromise → :673 handleResult
→ :711 handleSuccess → :713 promise.complete(res)
→ 触发 012 :176 的 whenComplete → :183 schedule() → 回 010 再上闹钟。
验证：ReadLockTask 的断点（:91 syncedEval 行）停第一次是 redisson-timer 线程；
     在 RenewalTask.java:183 打断点，续期完成后停下的是 redisson-netty-x。
```

**闭环达成：010 →（时间轮五跳）→ 012 → 015 → 016 → 回到 010。**

---

#### 全程一条线总结

```
你的线程（HTTP 请求线程）：
001 rLock.lock() → 002/003 借用写锁的壳 lock(-1,null,false) → 004 tryAcquire
→ 005 tryLockInnerAsync（分派！）→ 006 读锁Lua(mode=read, hincrby, timeout:1, 30s) → nil
→ 005 thenApply → 007 scheduleExpirationRenewal（分派！）→ 008 renewReadLock
→ 009 登记进全局 ReadLockTask → 010 newTimeout(this,10s) ─┐（返回，去读库写缓存）
                                                            │ 10 秒
redisson-timer-1-1（Netty 时间轮）：─────────────────────────┘
→ 011 巡检到期 task.run(this) → 012 run() → 013 execute() → 014/015 读锁续期Lua
   （anyAlive → 逐序号续 timeout key + 主 hash 30s）→ 发出后不等待
                                                            │ Redis 响应
redisson-netty（连接 eventLoop）：────────────────────────────┘
→ 016 promise.complete → whenComplete 同步执行 → schedule()（=010 再上闹钟）
        ▲                                                    │
        └──── 每 10 秒一轮，直到 rLock.unlock() 停表 ────────┘
```

#### 支线速览（面试追问才展开）

- **unlock 做了什么**：RedissonReadLock.java:82-140——hincrby -1，减到 0 删读者字段；del 对应 `:rwlock_timeout:{n}` key；还有别的读者（hlen>1）时算 maxRemainTime 续主锁、发解锁消息；最后一个读者 del 整个主锁
- **forceUnlock（读锁版）**：RedissonReadLock.java:165-175——mode==read 直接 del 主锁 + publish，不看还有没有读者
- **isLocked 的细节**：RedissonReadLock.java:179-190——mode==read 算锁住；mode==write 要 hlen>2（mode 字段 + 写者字段之外还有别的字段）才算，写锁持有者自己降级的读锁不算"锁住"
- **抢锁失败自旋**：003 已展示——订阅 `redisson_rwlock:{锁名}` 频道（:44）+ while 自旋，与写锁共用（RedissonLock.java:111-148）
- **写锁链路**：见 `skills/codetrace-plus/references/example-redisson.md`（同版本 22 步，Lua/Task/两处 override 都不同）

#### 面试问答（QA）

**Q1：Redisson 读写锁在 Redis 里长什么样？**
A：一个主 hash：`mode` 字段存 read/write，每个读者一个字段 `UUID:threadId` 存重入计数（006 :58-61）；再加每个重入序号一个独立 key `{锁}:{UUID:threadId}:rwlock_timeout:{n}`（:62）。注意 4.7.0 不是老文章说的 zset 结构——追到 :56 的 Lua 才算数。

**Q2：读锁之间会互斥吗？**
A：不会。:67 `mode == 'read'` 直接 hincrby 进门返回 nil；互斥只发生在 mode=='write' 且申请者不是写锁持有人时（:76 返回 pttl 进自旋）。

**Q3：同一个线程先写锁再读锁行吗？**
A：行，这是锁降级的入口：:67 后半 `mode == 'write' and hexists(KEYS[1], ARGV[3])`——写锁持有者（ARGV[3]=`UUID:threadId:write`）可以直接加读锁。反过来读→写不行，会 pttl 自旋。

**Q4：读锁也有看门狗吗？和写锁共用吗？**
A：有，但不共用。登记走独立的 `renewReadLock`（LockRenewalScheduler.java:44），全局单例是 `readLockReference` 里的 **ReadLockTask**（:45），续期 Lua 也不同（anyAlive 逐读者逐序号续，015）。三类锁三套 Task 并存（:31-33）。

**Q5：100 个线程同时读，Redis 里几个 key？多久续一次？**
A：1 个主 hash（含 mode + 100 个读者字段）+ 100 个 `:rwlock_timeout:1` key（有重入则更多）。ReadLockTask 每 10 秒一轮把活着的读者 key 和主 hash 全部续回 30s（批量一条 Lua）。

**Q6：某个读者线程崩了，读锁会永远占着吗？**
A：不会两重保险：它的 timeout key 30s 自然过期；unlock 侧 :109-120 hlen>1 时逐读者取 maxRemainTime 收敛主锁 TTL。但注意：**其它读者的看门狗会把主 hash 一直续下去**（015 anyAlive 只要一个活着就 pexpire 主锁）——所以"崩掉的读者"靠 key 过期退出，主锁由活人维护。

**Q7：为什么 ProductService.get() 要两层锁（hot_cache 锁 + 读锁）？**
A：001 场景：外层 hot_cache 是"缓存重建单飞锁"（防缓存击穿，一个线程去查库）；内层读锁保证查库+写缓存期间不被 update() 的写锁并发改数据（:54-56 update 拿同一把读写锁的写锁）。经典 DCL 双检 + 读写分离。

**Q8：读锁这条链和写锁差在哪？**
A：壳完全相同（002-004 一字不差，继承复用）；差两颗心脏：抢锁 Lua（RedissonReadLock:56 vs RedissonLock:214）和续期体系（ReadLockTask vs LockTask）。实现方式就是两次 @Override + 动态分派（p0003/p0004）——模板方法模式的教科书现场。

**Q9：`lock()` 和 `tryLock(-1)` 的读锁有 leaseTime 吗？能指定吗？**
A：不传 leaseTime → 内部用 30000ms 且挂看门狗；传了 leaseTime（如 tryLock(3, 30, SECONDS)）→ :187 分支用你的值且**不登记看门狗**（thenApply 里不会调 scheduleExpirationRenewal），到期自动放——与写锁行为一致。
