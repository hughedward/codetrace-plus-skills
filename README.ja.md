# Code Trace Plus · コードトレースの達人

[![skills.sh](https://skills.sh/b/hughedward/codetrace-plus-skills)](https://skills.sh/hughedward/codetrace-plus-skills)
[![License: MIT](https://img.shields.io/badge/License-MIT-3fb950.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-58a6ff.svg)](CHANGELOG.md)

[English](README.md) | [简体中文](README.zh-CN.md) | **日本語**

**ソースコードのコールチェーンを、IDEA / VS Code で一歩ずつクリックして追える「リーディングスクリプト」に変えます。**

真实のソースコードを追跡する AI Agent Skill です——面接の暗記話法ではなく。エントリポイント（`redissonLock.lock()`）や質問（「ウォッチドッグはどうロックを更新するの？」）を渡すと、**本物の**コードベースをホップごとに辿り、すべてのステップが IDE で実際に実行できる操作の markdown スクリプトを書き出します：*ここをクリック、と言われたら ctrl+クリックするだけ。*

## インストール（30秒）

```bash
npx skills@latest add hughedward/codetrace-plus-skills
```

<details>
<summary><strong>手動 ・ グローバルインストール（全プロジェクトで利用可能）</strong></summary>

```bash
mkdir -p ~/.claude/skills
cp -r skills/codetrace-plus ~/.claude/skills/
```

</details>

<details>
<summary><strong>手動 ・ 単一プロジェクト</strong></summary>

```bash
mkdir -p .claude/skills
cp -r skills/codetrace-plus .claude/skills/
```

</details>

[skills.sh](https://www.skills.sh/) が対応する Claude Code、Cursor、Codex、GitHub Copilot、Windsurf、Gemini CLI、OpenCode などのエージェントで動作します。

## なぜこの Skill が必要なのか

フレームワークのソースを読む辛さは、毎回同じ 3 つ：

1. **チュートリアルは散文であってコードではない。**「10秒後に Netty の Worker スレッドが run() をコールバックする」——どのクラス？どの行？暗記してもすぐ忘れる。それは読書ではなく丸暗記です。
2. **糸が切れる。** 20ホップ先、今呼ばれているオブジェクトは*どこか*から渡されたもの——10ステップ前に `this` として注入された——誰が誰を呼んでいるか分からなくなる。
3. **非線形ジャンプでチェーンが切れる。** オーバーライド、ラムダ、タイマー、スレッド切り替え——チュートリアルが曖昧に誤魔化し始め、追跡不能になる場所ちょうどそこです。

Code Trace Plus はこの 3 つを契約で解決します：

| 痛点 | 保証 |
|---|---|
| 暗記話法でコードの錨がない | **本物の行番号、捏造なし**——すべてのホップは実ソースの grep/read 由来で、依存バージョンを固定して検証 |
| 誰が誰を呼ぶか分からなくなる | **pNNNN クリック ID**——重要クリックに番号（`p0009`）を振り、後のステップから正確に参照：「この `this` は p0009（ステップ 011）由来」 |
| 非線形でチェーンが切れる | **非線形 ≠ 切断**——タイマー、コールバック、オーバーライド、スレッド切替、ライブラリ横断（Redisson → Netty）ですら*連続したクリック可能なチェーン*として追跡し、`task.run(this)` がコードに再突入する正確な行まで辿ります |

しかも適切な場所で止まります：主线が**ループを閉じたら**追跡終了。最後に一枚絵のサマリー（マルチスレッドリレー込み）と面接級 Q&A を付けて。

## このリポジトリの Skills

| Skill | 機能 | トリガーフレーズ |
|---|---|---|
| [`codetrace-plus`](skills/codetrace-plus/SKILL.md) | 実ソースのコールチェーンを追跡し、クリックで辿れるリーディングスクリプト（`trace/<topic>.md`）を生成 | trace the call chain / 追代码 / 呼び出しチェーンを追う / ソースを追跡 / xxx の実装を解説して |

## スクリプトの見た目

各ステップ = 実 `file:line` 付きの見出し → コードブロック → 次ホップのインラインコメント：

```java
#### ↓016、HashedWheelTimer.java:698 —— Netty から Redisson に戻る一行

705:         timer.taskExecutor.execute(this);
715:     public void run() {
717:             task.run(this);   // p0013：★ task は p0009（011）で渡された LockTask
                                //   【クリック】task.run → ctrl+alt+B → RenewalTask.java:170
```

非線形ジャンプにはコード下に来歴ブロック（来歴チェーン + 正体 + トリガーチェーン + ブレークポイント検証）が付きます：*「:170 にブレークポイントを置くと、ロック後 10 秒以内に停止、スレッド名 = redisson-timer-1-1」*

Golden sample——`redissonLock.lock()` からウォッチドッグ更新ループまで 21 ステップ、Redisson 4.7.0 を横断し **Netty 4.2.16 ソースにまで踏み込み**、ループを閉じる——は [`skills/codetrace-plus/references/example-redisson.md`](skills/codetrace-plus/references/example-redisson.md)（末尾に面接 Q&A 9 問、中国語）にあります。

## AI Agent 向け

ユーザーが（特に Redisson、Netty、Spring、Dubbo などのフレームワークの）コールチェーンや実装フローの追跡・説明を求めたら、常に `codetrace-plus` skill を使用してください。強制されるルール：実読したコードと実行番号のみ記述。依存バージョン固定。非線形ジャンプもクリック可能なチェーンとして追跡（ローカルにソースがなければ同バージョンの `-sources.jar` を取得）。ループ閉包検出。末尾に ASCII 一行サマリー + 面接 Q&A。完全な仕様：[`skills/codetrace-plus/SKILL.md`](skills/codetrace-plus/SKILL.md) ・ フォーマット詳細：[`references/format.md`](skills/codetrace-plus/references/format.md)。

## 仕組み

1. エントリポイントか質問を渡す。ソースのあるリポジトリならどこでも（またはパスを指定、依存の `-sources.jar` を取得させる）。
2. 実ソースを grep/read でホップごとに追跡：5〜15 行のスニペット + 実行番号、次ホップはインラインの【クリック】コメント。
3. `trace/<topic>.md` を開いて IDE で辿る——言われた場所をクリック——ループが閉じるまで。

## その他

- **ランディングページ**（GitHub Pages）：<https://hughedward.github.io/codetrace-plus-skills/> — English / 简体中文 / 日本語
- **変更履歴**：[CHANGELOG.md](CHANGELOG.md)
- **ライセンス**：[MIT](LICENSE)
