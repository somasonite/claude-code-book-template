# GitHub Actions ワークフロー設定

このドキュメントでは、`.github/workflows` ディレクトリに配置されている GitHub Actions ワークフローの設定内容について説明します。

## 一覧

| ファイル | 名前 | 概要 |
| --- | --- | --- |
| [`claude.yml`](../.github/workflows/claude.yml) | Claude Code | Issue や PR のコメントで `@claude` にメンションすると Claude Code が起動し、対応を行う |
| [`claude-code-review.yml`](../.github/workflows/claude-code-review.yml) | Claude Code Review | Pull Request の作成・更新時に Claude によるコードレビューを自動実行する |

---

## `claude.yml`（Claude Code）

Issue や Pull Request 上で `@claude` にメンションすることで、Claude Code を呼び出して質問への回答や実装作業を行わせるためのワークフローです。

### トリガー（`on`）

以下のイベントで起動します。

- `issue_comment`（`created`）: Issue や PR へのコメント作成時
- `pull_request_review_comment`（`created`）: PR のレビューコメント作成時
- `issues`（`opened`, `assigned`）: Issue の作成・アサイン時
- `pull_request_review`（`submitted`）: PR レビューの送信時

### 実行条件（`if`）

ジョブ全体に条件が設定されており、以下のいずれかを満たす場合のみ実行されます。

- `issue_comment` イベントで、コメント本文に `@claude` が含まれる
- `pull_request_review_comment` イベントで、コメント本文に `@claude` が含まれる
- `pull_request_review` イベントで、レビュー本文に `@claude` が含まれる
- `issues` イベントで、Issue の本文またはタイトルに `@claude` が含まれる

これにより、`@claude` を含むメンションがあった場合にのみ Claude Code が起動します。

### 権限（`permissions`）

| 権限 | 値 | 用途 |
| --- | --- | --- |
| `contents` | `read` | リポジトリ内容の読み取り |
| `pull-requests` | `read` | PR 情報の読み取り |
| `issues` | `read` | Issue 情報の読み取り |
| `id-token` | `write` | OIDC トークン発行（認証用） |
| `actions` | `read` | PR 上の CI 結果を Claude が参照するために必要 |

### ジョブ内容（`jobs.claude`）

1. **Checkout repository**: `actions/checkout@v4` によりリポジトリをチェックアウト（`fetch-depth: 1` で浅いクローン）
2. **Run Claude Code**: `anthropics/claude-code-action@v1` を実行
   - `claude_code_oauth_token`: `secrets.CLAUDE_CODE_OAUTH_TOKEN` を使用して認証
   - `additional_permissions`: `actions: read` を追加指定し、PR 上の CI 結果を Claude が読み取れるようにする
   - `prompt`（コメントアウト）: 固定プロンプトを指定したい場合に使用。未指定時はメンションされたコメントの指示に従って動作する
   - `claude_args`（コメントアウト）: 利用可能なツールの制限など、追加の挙動をカスタマイズする際に使用（詳細は [claude-code-action の usage ドキュメント](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md) や [CLI リファレンス](https://code.claude.com/docs/en/cli-reference) を参照）

---

## `claude-code-review.yml`（Claude Code Review）

Pull Request が作成・更新された際に、Claude によるコードレビューを自動的に実行し、インラインコメントとして指摘を投稿するワークフローです。

### トリガー（`on`）

- `pull_request`（`opened`, `synchronize`, `ready_for_review`, `reopened`）: PR の作成・更新・Draft 解除・再オープン時

コメントアウトされている `paths` 設定を有効化すると、特定のファイル（例: `src/**/*.ts` など）が変更された場合のみレビューを実行するよう制限できます。

### 実行条件（`if`、任意）

デフォルトでは無効化されていますが、コメントアウト部分を有効にすることで PR 作成者に応じたフィルタリングが可能です（例: 外部コントリビューターや初回コントリビューターの PR のみレビュー対象にする）。

### 権限（`permissions`）

| 権限 | 値 | 用途 |
| --- | --- | --- |
| `contents` | `read` | リポジトリ内容の読み取り |
| `pull-requests` | `read` | PR 情報の読み取り |
| `issues` | `read` | Issue 情報の読み取り |
| `id-token` | `write` | OIDC トークン発行（認証用） |

### ジョブ内容（`jobs.claude-review`）

1. **Checkout repository**: `actions/checkout@v4` によりリポジトリをチェックアウト（`fetch-depth: 1`）
2. **Run Claude Code Review**: `anthropics/claude-code-action@v1` を実行
   - `claude_code_oauth_token`: `secrets.CLAUDE_CODE_OAUTH_TOKEN` を使用して認証
   - `plugin_marketplaces`: `https://github.com/anthropics/claude-code.git` をプラグインマーケットプレイスとして指定
   - `plugins`: `code-review@claude-code-plugins` プラグインを使用
   - `prompt`: `/code-review:code-review --comment <repo>/pull/<PR番号>` を実行し、該当 PR に対してレビューを行う
   - `claude_args`: `--allowedTools "mcp__github_inline_comment__create_inline_comment"` を指定し、レビュー結果を PR にインラインコメントとして投稿できるようにする

---

## 共通の前提

両ワークフローとも、以下の Secrets が必要です。

- `CLAUDE_CODE_OAUTH_TOKEN`: Claude Code Action を実行するための OAuth トークン。リポジトリの Settings > Secrets and variables > Actions に登録しておく必要があります。

## 参考リンク

- [claude-code-action リポジトリ](https://github.com/anthropics/claude-code-action)
- [claude-code-action usage ドキュメント](https://github.com/anthropics/claude-code-action/blob/main/docs/usage.md)
- [Claude Code CLI リファレンス](https://code.claude.com/docs/en/cli-reference)
