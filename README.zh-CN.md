# Code Trace Plus · 代码追踪小能手

<!-- skills.sh badge: 收录后开启
[![skills.sh](https://skills.sh/b/hughedward/codetrace-plus-skills)](https://skills.sh/hughedward/codetrace-plus-skills)
-->
[![License: MIT](https://img.shields.io/badge/License-MIT-3fb950.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-58a6ff.svg)](CHANGELOG.md)

[English](README.md) | **简体中文** | [日本語](README.ja.md)

**把源码调用链，变成你能在 IDEA / VS Code 里一步步跟着点击的「阅读剧本」。**

一个**追踪真实源码**的 AI Agent Skill——不是背面试八股。给它一个入口（`redissonLock.lock()`）或一个问题（"看门狗怎么续期？"），它沿着**真实**代码库逐跳追踪，写一份 markdown 剧本，每一步都是你在 IDE 里能亲手执行的操作：*它说点哪，你就 ctrl+点击哪。*

## 安装（30 秒）

```bash
npx skills@latest add hughedward/codetrace-plus-skills
```

<details>
<summary><strong>手动 · 全局安装（所有项目可用）</strong></summary>

```bash
mkdir -p ~/.claude/skills
cp -r skills/codetrace-plus ~/.claude/skills/
```

</details>

<details>
<summary><strong>手动 · 单项目安装</strong></summary>

```bash
mkdir -p .claude/skills
cp -r skills/codetrace-plus .claude/skills/
```

</details>

兼容 Claude Code、Cursor、Codex、GitHub Copilot、Windsurf、Gemini CLI、OpenCode 等 [skills.sh](https://www.skills.sh/) 支持的 Agent。

## 为什么需要这个 Skill

读框架源码的痛，每次都是同样三个：

1. **教程是散文，不是代码。**"10 秒后 Netty 的 Worker 线程会回调 run()"——哪个类？哪一行？背完就忘。这是八股文，不是读代码。
2. **线索跟丢了。** 追了二十跳，正在被调的这个对象是*某处*传进来的——十步之前作为 `this` 注入的——你已经想不起谁在调谁。
3. **非线性跳转断链。** 重写、lambda、定时器、线程切换：恰好是教程开始挥手含糊、无法跟随的地方。

Code Trace Plus 用契约解决这三个问题：

| 痛点 | 保证 |
|---|---|
| 背八股，没有代码锚点 | **真实行号，绝不编造**——每一跳都来自对真实源码的 grep/read，并与锁定的依赖版本核对 |
| 跟久了忘了谁在调谁 | **pNNNN 点击编号**——关键点击有编号（`p0009`），后文精确回指："这个 `this` 来自 p0009（011 步）" |
| 非线性断链 | **非线性 ≠ 断链**——定时器、回调、重写、线程切换，甚至跨库跳转（Redisson → Netty）都追成*连续可点击的链*，精确到 `task.run(this)` 重新进入你代码的那一行 |

该停就停：主线**闭环**即终止，结尾附一张图总结（含多线程接力）和面试级问答。

## 本仓库的 Skills

| Skill | 作用 | 触发词 |
|---|---|---|
| [`codetrace-plus`](skills/codetrace-plus/SKILL.md) | 在真实源码中追调用链，生成可点击的阅读剧本（`trace/<主题>.md`） | 追代码 / 追一下调用链 / 源码追踪 / 帮我理清源码思路 / 讲讲 xxx 是怎么实现的 / trace the call chain |

## 剧本长什么样

每一步 = 带真实 `file:line` 的标题 → 代码块 → 下一跳的行内注释：

```java
#### ↓016、HashedWheelTimer.java:698 —— 从 Netty 跳回 Redisson 的那一行

705:         timer.taskExecutor.execute(this);
715:     public void run() {
717:             task.run(this);   // p0013：★ task 就是 p0009（011）传进来的 LockTask
                                //   【点击】task.run → ctrl+alt+B → RenewalTask.java:170
```

非线性跳转在代码块下方带来历块（来历链 + 身份 + 触发链 + 断点验证）：*"在 :170 打断点，加锁后 10 秒内必停，线程名 = redisson-timer-1-1。"*

Golden sample——从 `redissonLock.lock()` 到看门狗续期循环共 21 步，横跨 Redisson 4.7.0 **并追进 Netty 4.2.16 源码**、最终闭环——见 [`skills/codetrace-plus/references/example-redisson.md`](skills/codetrace-plus/references/example-redisson.md)（结尾含 9 条面试问答）。

## 给 AI Agent

当用户要求追踪、跟随或解释某条调用链 / 实现流程（尤其是 Redisson、Netty、Spring、Dubbo 等框架）时，请使用 `codetrace-plus` skill。它强制执行的硬规则：只写真实读过的代码与真实行号；锁定依赖版本；非线性跳转同样追成可点击的链（依赖源码不在本地时下载同版本 `-sources.jar`）；闭环检测；结尾一条线 ASCII 总结 + 面试问答。完整规范：[`skills/codetrace-plus/SKILL.md`](skills/codetrace-plus/SKILL.md) · 格式细则：[`references/format.md`](skills/codetrace-plus/references/format.md)。

## 工作方式

1. 你给一个入口或问题，任何有源码的仓库（或指定路径，或让它拉取依赖的 `-sources.jar`）。
2. 它 grep/read 真实源码逐跳追踪：5~15 行片段 + 真实行号，下一跳是行内【点击】注释。
3. 你打开 `trace/<主题>.md` 在 IDE 里跟着走——它说点哪就点哪——直到闭环。

## 更多

- **落地页**（GitHub Pages）：<https://hughedward.github.io/codetrace-plus-skills/> — English / 简体中文 / 日本語
- **变更记录**：[CHANGELOG.md](CHANGELOG.md)
- **许可**：[MIT](LICENSE)
