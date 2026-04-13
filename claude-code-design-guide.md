# Claude Code Design Guide

なぜその機能を選ぶのか——確定的な制御からエージェント的な推論まで

---

## このガイドの目的

公式ドキュメントは各機能の「設定方法」を教えてくれる。このガイドは「**なぜその機能を選ぶのか**」を教える。

Claude Code には CLAUDE.md、Rules、settings.json、Hooks、Skills、Subagents、MCP サーバーといった機能がある。どれも Claude の動作を変えるものだが、性質が異なる。このガイドでは3つの設計軸——**確定性**、**コンテキストコスト**、**強制度**——で全機能を位置づけ、「いつ何を使うか」の設計判断の基準を示す。

---

## 3つの設計軸

### 軸1: 確定的 ↔ 推論的

Claude Code のカスタマイズ機能は、**「確定的（deterministic）」** と **「推論的（agentic）」** という連続的なグラデーションの上に位置している。

> Hooks provide **deterministic control** over Claude Code's behavior, ensuring certain actions **always happen** rather than relying on the LLM to choose to run them.
> — [Hooks Guide](https://code.claude.com/docs/en/hooks-guide)
>
> Hooks は Claude Code の動作を確定的に制御する。LLM に実行するかどうかを委ねるのではなく、特定のアクションが必ず実行されることを保証する。

> Claude **decides what each step requires** based on what it learned from the previous step, chaining dozens of actions together and **course-correcting along the way**.
> — [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works)
>
> Claude は前のステップで学んだことをもとに、次に何が必要かを自分で判断する。数十のアクションを連鎖させながら、途中で軌道修正していく。

つまり「必ずこう動く」仕組みと「Claude が考えて判断する」仕組みの両方がある。

| 機能 | 性質 | 実行主体 |
|---|---|---|
| settings.json | 確定的 | ハーネス（※） |
| Hooks | 確定的 | シェル |
| CLAUDE.md / Rules | 中間 | Claude（指示に従う） |
| MCP サーバー | 中間 | Claude（ツールとして使用） |
| Skills | 中間 | Claude（手順に従う） |
| Subagents | 推論的 | Claude（独立判断） |

> ※ **ハーネス**: AI モデルを包んで動かすランタイム（実行基盤）。ユーザー入力の受け取り、ツール呼び出し、権限チェックなど、Claude（LLM）の外側で動くプログラム部分。

**確定的**な機能は、条件にマッチすれば毎回同じように実行される。Claude の判断は介在しない。**推論的**な機能は、Claude がコンテキストを読んで何をすべきか自分で判断する。**中間**にある CLAUDE.md や Skills は、Claude への指示だが実行は Claude の判断に委ねられる。指示を100%守る保証はない。

### 軸2: コンテキストコスト

各機能がコンテキストウィンドウ（1M トークン）をどれだけ消費するか。この違いが使い分けの重要な判断基準になる。

| 機能 | コンテキストへの影響 |
|---|---|
| settings.json / Hooks | **消費ゼロ**。ハーネス側で処理される |
| CLAUDE.md + paths なし Rules | **常時消費**。セッション中ずっと占有する |
| paths あり Rules | **条件付き**。マッチするファイルを読んだ時だけロード |
| Skills の description | **常時消費（微量）**。250文字以内 |
| Skills の本文 | **起動時に消費**。コンテキスト圧縮時に古いものから削除されうる |
| Subagents | **メインには消費なし**。別コンテキストで動く |

ここから導かれる設計原則:

- **常時ロードは最小限に**: CLAUDE.md + paths なし Rules は短く保つ（推奨200行以下/ファイル）
- **詳細は条件ロードに寄せる**: paths 付き Rules で必要時だけロード
- **大きな参考資料は Skills に**: 使う時だけ読み込まれる
- **重い調査・分析は Subagents に**: メインのコンテキストを消費しない

> "Longer files consume more context and **reduce adherence**."
> — [Memory](https://code.claude.com/docs/en/memory)

コンテキスト消費の問題は容量ではなく**指示追従度の低下**。100ページのルールブックより1ページの要点の方が守られる。

### 軸3: 強制度のエスカレーション

同じ目的（例: `.env` を守る）でも、求める強制度によって使う機能が変わる:

| 強制度 | 手段 | 効果 | 限界 |
|---|---|---|---|
| 低 | CLAUDE.md に「`.env` を編集しない」と書く | ほぼ守られる | 100%の保証はない |
| 中 | settings.json の `deny: ["Edit(.env)"]` | Edit ツール経由は確実にブロック | `Bash` で `cat .env` は通る |
| 高 | サンドボックスの `denyRead` | OS レベルで遮断。どのツール経由でも読めない | エスケープハッチの無効化が必要 |

この段階を意識すると「CLAUDE.md に書くべきか、settings.json に書くべきか」が明確になる。

---

## 機能の合成パターン

単体の機能だけでなく、組み合わせることで単体では得られない効果が生まれる。

### 確定的 × 推論的: Hooks が推論的機能をトリガーする

Hooks（確定実行）で Skills や Subagents（推論的）を起動する。たとえば Stop フックで「Claude の応答完了時に品質チェックスキルを実行させる」。確定的なトリガーで推論的な処理を呼ぶのが基本の合成形。SubagentStop フックで通知を送るのも同じパターン。

### コンテキスト隔離 × 呼び出しの手軽さ: Skills の context: fork

普通の Skill はメインのコンテキスト内で実行される。大量のファイルを読む分析タスクでは、途中の Read 結果もすべてメインに残りコンテキストが圧迫される。一方 Subagent は別コンテキストで動くのでメインを汚さないが、`/skill` のように手軽に呼べない。

`context: fork` はこのトレードオフを解消する。`/analyze src/` のようにスラッシュコマンドで起動でき、引数も事前承認も使える（Skill の利点）。しかし実行は別のコンテキストウィンドウで行われ、結果は要約されてメインの会話に返る（Subagent の利点）。途中の Read 結果などはメインに残らない。

**要するに「呼び方は Skill、実行場所は Subagent」。**

### 蓄積 × 共有: Subagents の Memory

`memory: project` を設定したレビューエージェントが、レビューのたびにプロジェクト固有のパターンを学習する。メモリは `.claude/agent-memory/` に Git 管理可能な形で保存されるため、チーム全体で共有できる。回を重ねるごとにレビュー品質が向上する。

---

## 判断フローチャート

新しいカスタマイズを追加するとき、以下の順で考える:

```
1. 100%の強制が必要か？
   ├─ Yes → settings.json (deny) またはサンドボックス
   └─ No ↓

2. イベント駆動で毎回同じ処理を実行したいか？
   ├─ Yes → Hooks
   └─ No ↓

3. 外部サービスへのアクセス手段が必要か？
   ├─ Yes → MCP サーバー
   └─ No ↓

4. 再利用可能な手順で、引数を受け取りたいか？
   ├─ Yes → Skills（重い処理なら context: fork）
   └─ No ↓

5. 独立したコンテキストで専門的に処理させたいか？
   ├─ Yes → Subagents
   └─ No ↓

6. Claude の判断を方向づけたいだけか？
   └─ Yes → CLAUDE.md / Rules
```

---

## 各機能の要点

ここからは各機能を「確定的↔推論的のどこにいるか」「他の機能との境界は何か」の観点で説明する。設定方法やフィールドの詳細は各出典リンク先の公式ドキュメントを参照。

### CLAUDE.md / Rules — 指示が届く範囲と限界

セッション開始時に自動ロードされる Markdown ファイル。Claude への指示・制約を書く。

**確定↔推論の位置**: 中間。Claude が読んで判断する。守られる可能性は高いが100%ではない。

**他機能との境界**:

- 「テストを先に書け」のような**方向づけ**にはこれで十分
- 「`.env` を絶対に編集するな」のような**強制**には settings.json の deny やサンドボックスを使う
- 指示が多すぎると追従度が落ちる。詳細なルールは paths 付き Rules に分割し、常時ロードを最小限にする

**CLAUDE.md と Rules の使い分け**: CLAUDE.md はプロジェクト全体に常に必要な要点だけ。トピック別の詳細は `.claude/rules/` に分割し、`paths` フロントマターで必要時だけロードさせる。分割しても paths なしなら常時ロード量は変わらない。

```
常時ロード量 = CLAUDE.md + paths なし Rules の合計
```

この合計を短く保つことが重要。

**ユースケース**:

| やりたいこと | 選ぶ機能 | なぜ他ではないか |
|---|---|---|
| ビルドコマンドを毎回伝えたくない | CLAUDE.md | Skills にするほどの手順ではない。一行で済む指示 |
| コーディング規約をチーム共有したい | Rules（Git 管理） | CLAUDE.md に全部書くと肥大化する。トピック別に分割 |
| API 開発時だけバリデーションルールを適用 | paths 付き Rules | paths なしだと常時コンテキストを消費する |
| 「`.env` を編集するな」と伝えたい | CLAUDE.md でも可 | ただし100%の保証が要るなら settings.json の deny へ |

> 出典: [Memory](https://code.claude.com/docs/en/memory)、[.claude directory](https://code.claude.com/docs/en/claude-directory)

### settings.json — Claude ではなくハーネスを動かす

Claude Code というハーネスの動作設定。権限、サンドボックス、Hooks、環境変数など。MCP サーバーの設定は主に `.mcp.json` で行う（ツールのセクション参照）。

**確定↔推論の位置**: 確定的。プログラムとして実行される。Claude の判断は介在しない。

**CLAUDE.md との決定的な違い**: CLAUDE.md は「Claude に読ませて守らせる」、settings.json は「ハーネスが強制する」。同じ「禁止」でも強制度がまったく異なる。

**deny の限界とサンドボックス**: `deny: ["Read(.env)"]` は Read ツールをブロックするが、`Bash` で `cat .env` すれば読めてしまう。deny はツール呼び出し時点のフィルタでしかない。OS レベルで遮断したいならサンドボックスを使う。サンドボックスにはエスケープハッチ（`dangerouslyDisableSandbox`）があるので、完全な遮断には `allowUnsandboxedCommands: false` も必要。

**全機能の配置場所**: Claude Code の設定ファイルは4層スコープ（managed > local > project > user）に従う。managed（組織）が最優先で下位はオーバーライドできない。配列フィールドは全スコープから結合、非配列フィールドは高優先度が勝つ。なお CLI 引数（`--model`, `--permission-mode` 等）はファイル設定より優先されるが、永続的な設定ではなく実行時の一時的なオーバーライド。

**ユースケース**:

| やりたいこと | 選ぶ機能 | なぜ他ではないか |
|---|---|---|
| `rm -rf` を絶対に実行させない | deny ルール | CLAUDE.md だと100%の保証がない。確定的にブロック |
| `.env` を Bash 経由でも読ませない | サンドボックス denyRead | deny ルールだと `Bash(cat .env)` を防げない |
| コード修正の反復で毎回の確認を減らしたい | acceptEdits モード | allow に個別追加するより、モード切替の方が手軽 |
| CI/CD パイプラインで無人実行したい | dontAsk モード | 権限確認が必要なモードでは CI が止まる |

> 出典: [Settings](https://code.claude.com/docs/en/settings)、[Permissions](https://code.claude.com/docs/en/permissions)、[Permission Modes](https://code.claude.com/docs/en/permission-modes)、[Sandboxing](https://code.claude.com/docs/en/sandboxing)

### ツール（組み込み + MCP） — 全機能を貫く共通言語

Claude が操作に使うツール群。Read、Edit、Bash などの組み込みツールと、MCP サーバーで追加する外部ツール。

**なぜここで理解しておくべきか**: ツール名の命名規約（`mcp__<サーバー名>__<ツール名>`）が**全機能に波及する共通言語**だから。

- settings.json の権限ルール: `"allow": ["mcp__github__*"]`
- Hooks の matcher: `"matcher": "Edit|Write"`
- Skills の allowed-tools: `allowed-tools: Bash(npm run build)`
- Subagents の tools: `tools: Read, Grep, Glob`

すべて同じツール名体系で指定する。ここを理解すると、他の全セクションの設定が一貫して読める。

**ツールと権限の関係**: 組み込み・MCP 問わず、すべてのツールは settings.json の権限ルール（`deny → ask → allow` の順に評価）で制御される。読み取り専用ツール（Read, Glob, Grep）は権限不要、書き込み系（Edit, Bash 等）は権限が必要。

**ユースケース**:

| やりたいこと | 選ぶ機能 | なぜ他ではないか |
|---|---|---|
| GitHub の PR を操作させたい | MCP サーバー追加 | 組み込みツールに GitHub 連携はない。MCP で拡張する |
| MCP ツールの一部だけ許可したい | 権限ルールで個別指定 | ツール名体系が共通なので `mcp__github__list_prs` のように絞れる |
| ブラウザの E2E テストを自動化したい | Playwright MCP（stdio） | リモートではなくローカルプロセスとして起動する用途 |

> 出典: [Tools Reference](https://code.claude.com/docs/en/tools-reference)、[MCP](https://code.claude.com/docs/en/mcp)

### Hooks — 確定実行されるシェルコマンド

Claude のライフサイクルイベント（ツール実行前後、応答完了時、セッション開始時など[多数](https://code.claude.com/docs/en/hooks)）にシェルコマンドをフックさせる仕組み。

**確定↔推論の位置**: 確定的。条件にマッチすれば必ず実行される。Claude の判断は介在しない。

**CLAUDE.md との決定的な違い**: 「Edit 後に Prettier を実行」を CLAUDE.md に書くと、Claude が忘れたり判断で省略する可能性がある。Hooks なら毎回確実に実行される。**「毎回同じ動作を保証したい」なら Hooks、「方向づけたい」なら CLAUDE.md**。

**合成パターンでの役割**: Hooks は他の推論的機能のトリガーとして機能する。Stop フックで Skills を呼び出す、SubagentStop フックで通知を送る——確定的なイベントで推論的な処理を起動するのが基本の合成パターン。

**重要な仕組み**: PreToolUse フックは exit code 2 でツール実行をブロックできる。stdin でツール名・引数を JSON で受け取り、スクリプトで判定する。これにより settings.json の deny より柔軟な条件分岐が可能。

**Hooks を定義できる3つの場所**: Hooks は settings.json だけのものではない。Skills と Subagents のフロントマターにも `hooks` を定義でき、**その機能が有効な間だけ作用するスコープ付きフック**になる。たとえばレビュー用 Subagent に PreToolUse フックを定義して、そのエージェントだけ書き込みツールをブロックする、といった使い方ができる。

| 定義場所 | スコープ |
|---|---|
| settings.json | グローバル（常時作用） |
| Skills のフロントマター | そのスキル実行中のみ |
| Subagents のフロントマター | そのサブエージェント実行中のみ |

**ユースケース**:

| やりたいこと | 選ぶ機能 | なぜ他ではないか |
|---|---|---|
| Edit 後に必ず Prettier を実行 | PostToolUse Hook | CLAUDE.md だと忘れる可能性がある。「必ず」なら確定実行 |
| 保護ファイルの編集を条件付きでブロック | PreToolUse Hook | deny は単純パターンマッチだけ。Hook ならスクリプトで柔軟に判定 |
| Claude 完了時にデスクトップ通知 | Stop Hook | Claude に「通知して」と頼む話ではない。外部連携は確定実行 |
| Subagent 完了時に Slack 通知 | SubagentStop Hook | 確定的トリガーで外部通知。Hooks + 推論的機能の合成パターン |

> 出典: [Hooks Guide](https://code.claude.com/docs/en/hooks-guide)、[Hooks Reference](https://code.claude.com/docs/en/hooks)

### Skills — 再利用可能なワークフロー

「この手順を実行しろ」という再利用可能なアクション。`.claude/skills/<name>/SKILL.md` に定義し、`/skill-name` で呼び出せる。

**確定↔推論の位置**: 中間。Claude が手順に従って実行する。

**CLAUDE.md / Rules との根本的な違い**: CLAUDE.md は「常にこうしろ」という制約、Skills は「呼ばれたらこれをやれ」というワークフロー。CLAUDE.md は常時ロードされるが、Skills は description だけ常時ロードされ、本文は起動時に初めて読み込まれる。この違いがコンテキスト管理に直結する。

**Subagents との境界**: 両方とも Claude が実行するが、通常の Skills はメインのコンテキスト内で動く。大きな分析タスクでメインのコンテキストを汚したくなければ、`context: fork` でサブエージェントとして実行するか、Subagents を使う。

**context: fork** を使うと「呼び方は Skill（`/skill` で呼べる、引数を受け取れる、`allowed-tools` で事前承認できる）、実行場所は Subagent（別コンテキスト、結果は要約されてメインに返る）」になる。

**前処理でシェルコマンドを実行できる**: Skill の本文に `` !`command` `` と書くと、Claude が読む前にシェルコマンドが実行され、結果が本文に埋め込まれる。たとえば PR のレビュースキルで `` !`gh pr diff` `` と書けば、起動時に自動で差分を取得してコンテキストに注入できる。静的な手順書ではなく、動的にコンテキストを組み立てるワークフローが作れる。

**コンテキスト管理上の注意**: 一度起動された Skill の本文（前処理の結果を含む）はコンテキストに残り続ける。ただしコンテキスト圧縮時には古いスキルから削除される可能性がある。再添付は各スキル先頭 5,000 トークン、合計 25,000 トークンの予算内で最新起動順に充填される。つまり1セッションで多くのスキルを起動すると、古いものは消える。スキルは軽く保ち、必要なものだけ起動するのが原則。

**自動起動の制御**: デフォルトでは Claude が description を見て関連があると判断したら自動でスキルを起動する。`disable-model-invocation: true` にすると自動起動しなくなり、ユーザーが `/skill` で明示的に呼んだ時だけ動く。デプロイやメッセージ送信など副作用があるスキルは自動起動を切っておく。

**ユースケース**:

| やりたいこと | 選ぶ機能 | なぜ他ではないか |
|---|---|---|
| デプロイ手順を `/deploy` で呼びたい | Skills | CLAUDE.md は呼び出せない。手順をコマンド化するのが Skill |
| 大きなコード分析を手軽に起動したい | Skills（context: fork） | Subagent は `/skill` で呼べない。fork なら手軽さと隔離を両立 |
| 大きな参考資料を必要時だけ読ませたい | Skills | CLAUDE.md / Rules は常時ロード。Skill は使う時だけ |
| デプロイスキルが勝手に起動するのを防ぎたい | disable-model-invocation | 副作用があるスキルはユーザー明示のみにする |

> 出典: [Skills](https://code.claude.com/docs/en/skills)

### Subagents — 独立した特化エージェント

メインの会話とは**別のコンテキストウィンドウ**で動く特化型エージェント。`.claude/agents/<name>.md` に定義する。

**確定↔推論の位置**: 推論的。独立したコンテキストで、何をすべきか自分で判断して行動する。

**Skills との根本的な違い**: コンテキストの隔離。Skills はメインのコンテキスト内で動き会話履歴が見えるが、Subagents は別コンテキストで動き親の会話履歴を知らない。メインのコンテキストを消費しないのが最大の利点。

**Skills ではなく Subagents を選ぶとき**:
- メインのコンテキストを消費したくない（大量の調査・分析）
- ツールを**制限**したい（Skills の `allowed-tools` は事前承認だけ、Subagents の `tools` はホワイトリストで制限できる）
- セッションを跨いで知識を蓄積したい（`memory` フィールド）
- 安全な隔離環境で作業させたい（`isolation: worktree` で Git worktree に隔離）
- 軽い調査はコストを抑えたい（`model: haiku` 等でモデルを変更できる）

**Skills の context: fork で足りる場合**: `/skill` で手軽に呼びたい、引数を渡したい、`allowed-tools` で事前承認したい——これらが必要なら Skills の `context: fork` で十分。永続メモリやツール制限が必要な場合にだけ Subagents を使う。

**合成パターンでの役割**: Hooks と組み合わせて SubagentStop で通知を送る、`memory: project` で知見をチームに蓄積する。独立した専門家として他の機能と組み合わさる。

**ユースケース**:

| やりたいこと | 選ぶ機能 | なぜ他ではないか |
|---|---|---|
| コードレビューを読み取り専用で任せたい | Subagents（tools 制限） | Skills の allowed-tools は事前承認だけで制限はできない |
| 大量のファイルを調査させたい | Subagents | Skills だとメインのコンテキストを消費する |
| レビュー知見をチームに蓄積したい | Subagents（memory: project） | Skills にはメモリがない |
| 軽い調査をコスト抑えて実行したい | Subagents（model: haiku） | Skills でも model 指定は可能だが、Subagent ならコンテキスト隔離も同時に得られる |
| 安全な隔離環境で作業させたい | Subagents（isolation: worktree） | Skills にはワークツリー隔離がない |

> 出典: [Subagents](https://code.claude.com/docs/en/sub-agents)
