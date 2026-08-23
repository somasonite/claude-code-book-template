# GitHub Actions ワークフロー設定

このドキュメントでは、`.github/workflows` 配下に定義されている GitHub Actions ワークフローの設定内容について説明します。

## 概要

このリポジトリには、[Claude Code Action](https://github.com/anthropics/claude-code-action) を利用した2つのワークフローが定義されています。

| ファイル | 名前 | 役割 |
| --- | --- | --- |
| [`claude.yml`](../.github/workflows/claude.yml) | Claude Code | Issue や PR 上で `@claude` にメンションすると Claude が応答・作業を行う |
| [`claude-code-review.yml`](../.github/workflows/claude-code-review.yml) | Claude Code Review | Pull Request が作成・更新された際に Claude が自動でコードレビューを行う |

いずれのワークフローも実行には Anthropic の OAuth トークンを [`CLAUDE_CODE_OAUTH_TOKEN`](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md) というリポジトリシークレットとして事前に設定しておく必要があります。

## `claude.yml`（Claude Code）

Issue コメントや PR レビューコメントなどで `@claude` にメンションされたときに Claude を起動し、応答・調査・コード変更・PR作成などのタスクを実行させるワークフローです。

### トリガー（`on`）

- `issue_comment`（`created`）: Issue や PR 上のコメントが作成されたとき
- `pull_request_review_comment`（`created`）: PR のレビューコメントが作成されたとき
- `issues`（`opened`, `assigned`）: Issue が作成、またはアサインされたとき
- `pull_request_review`（`submitted`）: PR レビューが送信されたとき

### 実行条件（`if`）

ジョブ全体は、以下のいずれかの条件を満たした場合のみ実行されます。

- Issue コメント本文に `@claude` が含まれる
- PR レビューコメント本文に `@claude` が含まれる
- PR レビュー本文に `@claude` が含まれる
- Issue の本文またはタイトルに `@claude` が含まれる

これにより、`@claude` へのメンションがない限りワークフローは実行されません。

### 権限（`permissions`）

| 権限 | 値 | 用途 |
| --- | --- | --- |
| `contents` | `read` | リポジトリのコード読み取り |
| `pull-requests` | `read` | PR情報の読み取り |
| `issues` | `read` | Issue情報の読み取り |
| `id-token` | `write` | OIDC トークンの発行（認証に使用） |
| `actions` | `read` | PR上のCI結果をClaudeが読み取れるようにする |

### ジョブ内容（`claude` ジョブ）

1. `actions/checkout@v4` でリポジトリをチェックアウト（`fetch-depth: 1` で浅いクローン）
2. `anthropics/claude-code-action@v1` を実行し、以下を設定
   - `claude_code_oauth_token`: シークレット `CLAUDE_CODE_OAUTH_TOKEN` を使用
   - `additional_permissions`: `actions: read` を追加指定し、PR上のCI結果を読めるようにする
   - `prompt`（コメントアウト）: カスタムプロンプトを指定したい場合に使用可能。未指定の場合、コメントで `@claude` に続けて書かれた指示内容がそのまま実行される
   - `claude_args`（コメントアウト）: 利用可能なツールの制限などの追加設定例

## `claude-code-review.yml`（Claude Code Review）

Pull Request が作成・更新された際に、Claude が自動でコードレビューを行い、PRにインラインコメントを投稿するワークフローです。

### トリガー（`on`）

- `pull_request`（`opened`, `synchronize`, `ready_for_review`, `reopened`）

コメントアウトされた `paths` 設定を有効にすることで、特定のファイル（例: `src/**/*.ts` など）が変更された場合のみレビューを実行するよう絞り込むことも可能です。

### 実行条件

デフォルトではジョブレベルの `if` 条件は設定されておらず、対象イベントが発生すれば常に実行されます。コメントアウトされた設定を有効にすることで、以下のような条件で対象PRを絞り込むことができます。

- 特定のPR作成者（例: `external-contributor`, `new-developer`）のみ
- 初回コントリビューター（`FIRST_TIME_CONTRIBUTOR`）のみ

### 権限（`permissions`）

| 権限 | 値 | 用途 |
| --- | --- | --- |
| `contents` | `read` | リポジトリのコード読み取り |
| `pull-requests` | `read` | PR情報の読み取り |
| `issues` | `read` | Issue情報の読み取り |
| `id-token` | `write` | OIDC トークンの発行（認証に使用） |

### ジョブ内容（`claude-review` ジョブ）

1. `actions/checkout@v4` でリポジトリをチェックアウト（`fetch-depth: 1` で浅いクローン）
2. `anthropics/claude-code-action@v1` を実行し、以下を設定
   - `claude_code_oauth_token`: シークレット `CLAUDE_CODE_OAUTH_TOKEN` を使用
   - `plugin_marketplaces`: `anthropics/claude-code` を Claude Code のプラグインマーケットプレイスとして登録
   - `plugins`: `code-review@claude-code-plugins` プラグインを使用
   - `prompt`: `/code-review:code-review --comment` コマンドを対象PRに対して実行し、レビュー結果をインラインコメントとして投稿
   - `claude_args`: `mcp__github_inline_comment__create_inline_comment` ツールのみを許可し、インラインコメント投稿に必要な権限のみを付与

## 参考リンク

- [claude-code-action リポジトリ](https://github.com/anthropics/claude-code-action)
- [claude-code-action 利用方法ドキュメント](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md)
- [Claude Code CLI リファレンス](https://code.claude.com/docs/en/cli-reference)
