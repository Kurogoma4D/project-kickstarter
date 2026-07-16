# project-kickstarter

Claude Code と Codex 両対応のエージェント定義テンプレート集。要求のヒアリングから Issue の起票・自動実装・レビュー・マージまでを一貫して行うエージェントとスキルを提供します。

## Disclaimer

このテンプレートはGitHubを利用することを前提としたものです。ご自身の開発スタイルに合わない場合は、適宜お手元のエージェントにご相談の上Skills/Subagentsを書き換えてご利用ください。

## 対応エージェント

Claude Code と Codex の両方が読める形でエージェント定義を提供します。

- スキルは共有。Codex 用ディレクトリ `.agents/skills` は `.claude/skills` へのシンボリックリンク。Skill は Agent Skills オープン標準（`name` / `description` + 本文）に両者とも準拠するため、同じファイルをそのまま使う。
- サブエージェントはエージェントごとに定義。Claude 用 `.claude/agents/*.md` と Codex 用 `.codex/agents/*.toml` をそれぞれソースとして持つ。Markdown と TOML で形式が異なるため、片方を編集したらもう片方も合わせる。

## 含まれるファイル

```
.claude/                       # 正本（Claude Code が読む）
├── agents/
│   ├── code-reviewer.md       # PR コードレビューサブエージェント
│   └── issue-implementer.md   # GitHub Issue 実装サブエージェント
└── skills/
    ├── kickstart/SKILL.md             # ワークフロー一括実行
    ├── template-setup/SKILL.md        # プレースホルダ一括置換（セットアップ）
    ├── template-update/SKILL.md       # テンプレート更新を作業リポジトリへ反映（3-wayマージ）
    ├── spec-builder/SKILL.md          # 要求ヒアリング → spec.md 作成
    ├── spec-to-issues/SKILL.md        # spec.md → タスク分解 → Issue 起票
    ├── task-to-issue/SKILL.md         # 単発タスクを1件の Issue として起票
    ├── supply-chain-guard/SKILL.md    # サプライチェーン対策（ガードレール構築 → 監査 → Issue 起票）
    ├── auto-issue-worker/SKILL.md     # Issue 自動消化
    ├── issue-worker/SKILL.md          # 単一 Issue を実装・1回レビューして PR を残す
    └── pentest/SKILL.md               # ペネトレーションテスト → 脆弱性 Issue 起票

.agents/
└── skills -> ../.claude/skills   # Codex 用スキル（.claude/skills へのシンボリックリンク）

.codex/
└── agents/                       # Codex 用サブエージェント（ソース）
    ├── code-reviewer.toml
    └── issue-implementer.toml
```

## セットアップ

1. このリポジトリをテンプレートとしてリポジトリを作成する
   1. もしくは `.claude/`・`.agents/`・`.codex/` をプロジェクトルートにコピー（`.agents/skills` はシンボリックリンクなので、リンクを保ったままコピーする）
2. `/kickstart` を実行すると、以下のワークフロー（spec 作成 → プレースホルダ置換 → Issue 起票）を順に実行する
   - 個別に実行したい場合は各スキルを直接呼び出す（手動でプレースホルダを置換する場合は下記の一覧を参照）
   - `/template-setup` は `.claude/agents/*.md` と `.codex/agents/*.toml` の両方のプレースホルダを埋める

## テンプレート更新の反映

このテンプレートを更新した後、テンプレートから作成済みの作業リポジトリへ更新を取り込むには `/template-update` を実行する。

- GitHub のテンプレートリポジトリは fork と異なり元リポジトリとの履歴リンクを持たないため、作業リポジトリ側で同期済みコミットを `.claude/.template-version` に記録する。`/template-setup` 実行時に自動生成される。
- `/template-update` は記録した基準コミット（base）と最新テンプレート（theirs）の差分を、置換済みの作業ファイル（ours）へファイル単位の 3-way マージで適用する。テンプレートの変更だけが反映され、置換済みのプレースホルダ値は保持される。
- プレースホルダを埋めた行とテンプレートの変更行が隣接している（同一ハンクになる）場合はコンフリクトが発生する。解決は機械的で、作業リポジトリの値を残しつつテンプレートの周辺変更を採り、`/template-update` が対話的に処理する。
- 新規追加されたテンプレートファイルはプレースホルダを含むため、反映後に `/template-setup` を実行して埋める。
- マージ対象はテンプレート所有パス（`.claude/`・`.agents/`・`.codex/`）。`.codex/agents/*.toml` も同じプレースホルダを持つソースなので 3-way マージで反映する。

```
/template-update 実行
  ↓
.claude/.template-version から base コミットを取得（無ければブートストラップ）
  ↓
テンプレートを FETCH_HEAD へ取得（origin は変更しない）
  ↓
.claude/・.agents/・.codex/ の変更ファイルを base→最新で抽出
  ↓
変更: 3-way マージ / 追加: そのまま配置 / 削除: 確認の上で削除
  ↓
コンフリクトを解決 → git diff で確認 → base SHA を更新
```

## プレースホルダ一覧

| プレースホルダ | 説明 | 記入例 |
|---|---|---|
| `{{PROJECT_NAME}}` | プロジェクト名 | `my-app` |
| `{{PROJECT_DESCRIPTION}}` | プロジェクトの簡潔な説明 | `a web application built with Next.js` |
| `{{PROJECT_SHORT_DESCRIPTION}}` | 1行のプロジェクト概要（auto-issue-worker 用） | `my-app is a Next.js web application for task management.` |
| `{{GITHUB_OWNER}}` | GitHub オーナー名（ユーザー or 組織） | `octocat` |
| `{{GITHUB_REPO}}` | GitHub リポジトリ名 | `my-app` |
| `{{PROJECT_STRUCTURE}}` | プロジェクトのディレクトリ構造やモジュール構成の説明 | 下記参照 |
| `{{KEY_DEPENDENCIES}}` | 主要な依存ライブラリのリスト | `React, Next.js, Prisma, Tailwind CSS` |
| `{{LANGUAGE_VERSION_NOTE}}` | 言語バージョンや特筆すべき言語機能の説明 | `**TypeScript**: 5.4+ with strict mode enabled.` |
| `{{TECH_STACK}}` | 技術スタックの箇条書き（issue-implementer 用） | 下記参照 |
| `{{LANGUAGE_SPECIFIC_REVIEW_CRITERIA}}` | 言語固有のレビュー観点（code-reviewer 用） | 下記参照 |
| `{{LANGUAGE_SPECIFIC_REVIEW_RULES}}` | 言語固有のレビュールール（code-reviewer 用） | 下記参照 |
| `{{LANGUAGE_SPECIFIC_IMPLEMENTATION_GUIDELINES}}` | 言語固有の実装ガイドライン（issue-implementer 用） | 下記参照 |
| `{{QA_COMMANDS}}` | QA で実行するコマンド一覧（issue-implementer 用） | 下記参照 |

## プレースホルダの記入例

### `{{PROJECT_STRUCTURE}}`

```markdown
Cargo workspace project with the following crate structure:
- `crates/core/` — Core business logic
- `crates/api/` — REST API server
- `crates/db/` — Database access layer
- `src/` — Binary entry point
```

### `{{TECH_STACK}}`

```markdown
- **Language**: TypeScript 5.4+
- **Framework**: Next.js 14 (App Router)
- **Database**: PostgreSQL + Prisma ORM
- **Styling**: Tailwind CSS
- **Testing**: Vitest + Playwright
- **Package manager**: pnpm
```

### `{{LANGUAGE_SPECIFIC_REVIEW_CRITERIA}}`

TypeScript の場合:

```markdown
- **Type safety**: Are types properly defined? No `any` unless justified. Prefer `unknown` over `any`.
- **Null handling**: Are nullable values handled with optional chaining or null checks?
- **Async correctness**: Are Promises properly awaited? No floating promises.
- **React patterns** (if applicable): Are hooks rules followed? Are effects properly cleaned up?
```

Rust の場合:

```markdown
- **Ownership & lifetimes**: Are there unnecessary clones, lifetime issues, or potential use-after-move bugs?
- **Error handling**: Are errors propagated properly using `?` with context?
- **Async correctness**: Are `Send`/`Sync` bounds satisfied? No blocking on async runtime?
```

### `{{LANGUAGE_SPECIFIC_REVIEW_RULES}}`

Rust の場合:

```markdown
- Pay special attention to `unsafe` blocks — they must have a `// SAFETY:` comment.
- Flag any use of `#[allow(...)]` — `#[expect(lint, reason = "...")]` is preferred.
- Flag `.unwrap()` / `.expect()` outside of tests — suggest `?` with context instead.
```

TypeScript の場合:

```markdown
- Flag `any` types — suggest proper typing or `unknown`.
- Flag `console.log` in production code — use the project's logger.
- Verify `async` functions are properly `await`ed at call sites.
```

### `{{LANGUAGE_SPECIFIC_IMPLEMENTATION_GUIDELINES}}`

```markdown
- **Error handling**: Use Result types or proper try-catch with typed errors
- **Testing**: Write unit tests for business logic, integration tests for API endpoints
- **Code style**: Follow the project's ESLint/Prettier/formatter configuration
```

### `{{QA_COMMANDS}}`

TypeScript (Next.js) の場合:

```markdown
- **Type check**: `npx tsc --noEmit` — ensure no type errors
- **Lint**: `pnpm lint` — no lint warnings
- **Format**: `pnpm format:check` — verify formatting
- **Test**: `pnpm test` — run the full test suite
- **Build**: `pnpm build` — ensure the project builds successfully
```

Rust の場合:

```markdown
- **Build**: `cargo build --workspace` — ensure compilation
- **Lint**: `cargo clippy --all-targets --all-features -- -D warnings` — no clippy warnings
- **Format**: `cargo fmt --all -- --check` — verify formatting
- **Test**: `cargo test --workspace` — run the full test suite
```

## ワークフロー概要

`/kickstart` を実行すると、以下を順に呼び出す（各ステップで確認を挟む）。

```
/kickstart 実行
  ↓
/spec-builder 実行
  ↓
要求をヒアリング → spec.md 作成
  ↓
/template-setup → spec.md を活用してプレースホルダを一括置換
  ↓
/supply-chain-guard → 依存固定・Claude Code 権限・CI を堅牢化し、残存リスクを Issue 起票
  ↓
spec.md を元にプロジェクトの README.md を更新
  ↓
差分をコミット & push（spec.md・README.md・.claude/・堅牢化設定をリモートへ）
  ↓
/spec-to-issues 実行
  ↓
spec.md をタスク分解 → ユーザー承認 → Issue 起票
  ↓
/auto-issue-worker 実行
  ↓
メインエージェント（PM）が Issue の依存グラフから並列実行可能なバッチを編成
  ↓
issue-implementer（Tech Specialist）が Issue ごとに worktree で並列実装 → PR 作成
  ↓
code-reviewer が観点別（正確性 / セキュリティ / テスト / 設計・性能）に並列レビュー
  ↓
PM が指摘を統合 → issue-implementer が修正 → 再レビュー（最大3回）
  ↓
LGTM → squash merge（マージのみ直列・競合時は rebase）→ 次のバッチへ
  ↓
（ある程度実装後・任意）/pentest 実行
  ↓
静的解析 + 動的テストで脆弱性を検出 → ユーザー承認 → 脆弱性 Issue 起票
  ↓
起票した脆弱性 Issue を /auto-issue-worker で修正
```

## スキル一覧

Claude Code・Codex とも `/skill-name`（Codex は `$skill-name`）で起動する。

| スキル | 起動 | 役割 |
|---|---|---|
| `kickstart` | `/kickstart` | 各スキルを順に呼び出しワークフローを一括実行する |
| `template-setup` | `/template-setup` | ヒアリング結果に応じて全ファイルのプレースホルダを置換する |
| `template-update` | `/template-update` | テンプレートの更新を作業リポジトリへ 3-way マージで反映する |
| `spec-builder` | `/spec-builder` | 要求をヒアリングし `spec.md` にまとめる |
| `spec-to-issues` | `/spec-to-issues` | `spec.md` をタスク分解し GitHub Issue を起票する |
| `task-to-issue` | `/task-to-issue` | 単発タスクをヒアリングし1件の Issue として起票する |
| `supply-chain-guard` | `/supply-chain-guard` | 依存・エージェント環境・CI を堅牢化し、残存リスクを Issue 起票する |
| `auto-issue-worker` | `/auto-issue-worker` | 起票済み Issue を PM として並列で実装・レビュー・マージする |
| `issue-worker` | `/issue-worker` | 単一 Issue を実装し観点別パネルで1回レビューして PR を残す（マージしない） |
| `pentest` | `/pentest` | 静的解析と動的テストで脆弱性を検出し Issue を起票する |

| サブエージェント | 役割 |
|---|---|
| `code-reviewer` | PR を観点別の専門レビュアーとして（または単体でフル）レビューする |
| `issue-implementer` | Tech Specialist として GitHub Issue を worktree で実装し PR を作成・修正する |

## ライセンス

MIT
