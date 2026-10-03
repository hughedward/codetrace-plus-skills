# Code Trace Plus

-->
[![License: MIT](https://img.shields.io/badge/License-MIT-3fb950.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-58a6ff.svg)](CHANGELOG.md)

**English** | [简体中文](README.zh-CN.md) | [日本語](README.ja.md)

**Turn source-code call chains into "reading scripts" you can click through, step by step, in IDEA / VS Code.**

An AI Agent Skill for **tracing real source code** — not reciting interview answers. Give it an entry point (`redissonLock.lock()`) or a question ("how does the watchdog renew the lock?"), it walks the **real** codebase hop by hop, and writes a markdown script where every step is an IDE action you can physically perform: *it says click here, you ctrl+click here.*

## Installation (30 seconds)

```bash
npx skills@latest add hughedward/codetrace-plus-skills
```

<details>
<summary><strong>Manual · global (all projects)</strong></summary>

```bash
mkdir -p ~/.claude/skills
cp -r skills/codetrace-plus ~/.claude/skills/
```

</details>

<details>
<summary><strong>Manual · single project</strong></summary>

```bash
mkdir -p .claude/skills
cp -r skills/codetrace-plus .claude/skills/
```

</details>

Works with Claude Code, Cursor, Codex, GitHub Copilot, Windsurf, Gemini CLI, OpenCode and other agents supported by [skills.sh](https://www.skills.sh/).

## Why This Skill Exists

Reading framework source is painful for the same three reasons, every time:

1. **Tutorials are prose, not code.** "10 seconds later, Netty's Worker thread calls back into run()" — which class? which line? You memorize it, then forget it. That's recited lore, not reading code.
2. **You lose the thread.** Twenty jumps in, the object being called came from *somewhere* — injected as `this` ten steps ago — and you can't remember who is calling whom.
3. **Non-linear jumps break the chain.** Overrides, lambdas, timers, thread handoffs: exactly where tutorials wave their hands and stop being followable.

Code Trace Plus fixes all three by contract:

| Pain | Guarantee |
|---|---|
| Recited lore, no code anchors | **Real line numbers, never invented** — every hop comes from grep/read of real source, verified against a pinned dependency version |
| Lost track of who calls whom | **pNNNN click IDs** — key clicks are numbered (`p0009`); later steps point back: "this `this` came from p0009 (step 011)" |
| Non-linear chains break | **Non-linear ≠ broken chain** — timers, callbacks, overrides, thread switches, even cross-library hops (Redisson → Netty) are traced as a *continuous clickable chain*, down to the exact line where `task.run(this)` re-enters your code |

And it stops where it should: the trace terminates when the main line **closes the loop**, ending with a one-picture summary (multi-thread relay included) and interview-grade Q&A.

## Skills in This Repo

| Skill | What it does | Triggers when you say |
|---|---|---|
| [`codetrace-plus`](skills/codetrace-plus/SKILL.md) | Trace a call chain through real source code and write a clickable reading script (`trace/<topic>.md`) | trace the call chain / how is xxx implemented / follow this call / 追代码 / 追一下调用链 / 源码追踪 / 帮我理清源码思路 / 讲讲 xxx 是怎么实现的 |

## What a Script Looks Like

Every step is: a heading with real `file:line` → a code block → the next hop as an inline comment:

```java
#### ↓016、HashedWheelTimer.java:698 —— the line where we jump back from Netty into Redisson

705:         timer.taskExecutor.execute(this);
715:     public void run() {
717:             task.run(this);   // p0013: ★ task is the LockTask passed out at p0009 (step 011)
                                //   [click] task.run → ctrl+alt+B → RenewalTask.java:170
```

Non-linear hops carry a provenance block under the code (origin chain + identity + trigger chain + breakpoint proof): *"breakpoint at :170 fires within 10s of locking, thread name = redisson-timer-1-1."*

The golden sample — 21 steps from `redissonLock.lock()` to the watchdog renewal loop, across Redisson 4.7.0 **and into Netty 4.2.16 source**, closing the loop — lives at [`skills/codetrace-plus/references/example-redisson.md`](skills/codetrace-plus/references/example-redisson.md) (ends with 9 interview Q&As, in Chinese).

## For AI Agents

Use the `codetrace-plus` skill whenever the user asks to trace, follow, or explain a call chain / implementation flow through real source code — especially for frameworks (Redisson, Netty, Spring, Dubbo…). Hard rules it enforces: only real read code with real line numbers; pinned versions; non-linear jumps must also be traced as clickable chains (download same-version `-sources.jar` when needed); loop-closure detection; final one-line ASCII summary + interview QA. Full spec: [`skills/codetrace-plus/SKILL.md`](skills/codetrace-plus/SKILL.md) · Format reference: [`references/format.md`](skills/codetrace-plus/references/format.md).

## How It Works

1. You give an entry point or a question, in any repo with source (or point it at one, or let it fetch the dependency's `-sources.jar`).
2. It greps/reads the real source, hop by hop: 5–15-line snippets with real line numbers, next hop as an inline `[click]` comment.
3. You open `trace/<topic>.md` and follow along in the IDE — it says click where, you ctrl+click there — until the loop closes.

## More

- **Landing page** (GitHub Pages): <https://hughedward.github.io/codetrace-plus-skills/> — English / 简体中文 / 日本語
- **Changelog**: [CHANGELOG.md](CHANGELOG.md)
- **License**: [MIT](LICENSE)
