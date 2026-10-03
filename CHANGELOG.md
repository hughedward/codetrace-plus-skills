# Changelog

## 0.1.0（2026-10-03）

首个固化版本。skill 标识 `codetrace-plus`，英文名 **Code Trace Plus**，中文名**代码追踪小能手**。

仓库布局按 [skills.sh](https://www.skills.sh/) 收录约定：skill 位于 `skills/codetrace-plus/`，可用 `npx skills add <owner>/<repo>` 安装。README 按 AI 可发现性最佳实践编写（skills.sh badge / 30 秒安装 / 触发词表 / For AI Agents 段），并附 [llms.txt](llms.txt)（llmstxt.org 标准，供 AI 爬虫与 Agent 读取）与 GitHub Pages landing page（`docs/index.html`）。

### 核心能力

- 给一个入口/问题，在真实源码里逐跳验证，产出中文"IDE 阅读剧本"（markdown 落盘到 `trace/<主题>.md`）
- 用户拿着剧本在 IDEA/VS Code 里逐步操作：它说点哪就 ctrl+点击哪

### 格式规范（经 4 轮用户反馈迭代定稿）

- 编号三位数字（001 起）进标题：`#### ↓NNN、File.java:行号 一句话描述`，正文直接引用编号
- 顺序固定：标题 → 代码块 → 说明（任何说明不得插在标题和代码块之间）
- 跳转指令只写代码行内注释：`【点击】符号 跳转 本文件:行 / 文件.java:行`，线性调用禁止"为什么能走到"式废话
- 关键点击带 `pNNNN` 编号（全文递增），跨步骤精确回指（专治"跟久了忘了谁在调谁"）
- **非线性 ≠ 断链**（灵魂规则）：定时器/回调/重写/换库/换线程也追成连续代码链，每句话落到 file:line；跨库用同版本 sources（示例中追进 Netty HashedWheelTimer）；JDK 内部用"语义一句话 + 断点验证线程名"落地
- **不留半吊子结论**：结论式的话必须说完整（机制怎么实现、和什么对比、代码锚点），要么说完整要么删掉
- 主线筛选以"面试官会问什么"为标尺；支持 `.....` 折叠、`{疑问？}` 处设问并当步回答
- 收尾三件套：全程一条线总结（ASCII，多线程接力，节点用编号）+ 支线速览 + 面试问答 QA（答案回引编号/行号，不引入主线外内容）

### 硬性规则（节选）

- 只写真实读过的代码，行号必须实际 grep/read 验证，禁止凭记忆编写
- 文档头声明依赖版本，行号以该版本为准
- 闭环检测：回到已出现编号即标注"闭环"并终止主线
- 禁止悄悄换线程：线程跳变只在来历说明和总结图中体现

### 附带资产

- `references/example-redisson.md`：22 步 golden sample（Redisson 4.7.0 + Netty 4.2.16.Final，从 `redissonLock.lock()` 到看门狗批量续期闭环，跨 Netty 追踪，9 条 QA）
- `references/format.md`：格式细则与正反例
