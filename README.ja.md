# `gh looppilot` — LoopPilot セットアップ CLI

[English](README.md) | **日本語**

LoopPilot の導入（そのままだとコピー＆ペーストした約 461 行の YAML になる作業）を
1 つのコマンドにまとめる
[GitHub CLI 拡張機能](https://docs.github.com/en/github-cli/github-cli/creating-github-cli-extensions)
です。`team-yubune/loop-pilot@v1` の再利用可能ワークフローを参照する薄い caller
ワークフローを生成し、ゲートラベルを作成し、`CHECK_COMMAND` を提案し、手動セット
アップ手順を一覧表示し、認証済みの `gh` セッションに対して読み取り専用の pre-flight
を実行します。

配布方針と Action 対 CLI のリリース境界については、メインリポジトリの
[ADR-0001](https://github.com/team-yubune/loop-pilot/blob/main/docs/architecture/adr-0001-cli-distribution.md)
を参照してください。このリポジトリが拡張機能の正規の置き場所です（gh 拡張機能は
`gh-<name>` リポジトリに置く必要があります）。

## インストール

```bash
gh extension install team-yubune/gh-looppilot
gh looppilot init      # caller のスキャフォールド + ラベル + CHECK_COMMAND + 手動手順 + pre-flight
gh looppilot doctor    # 読み取り専用の pre-flight のみ
```

PATH 上に Node ≥ 20 が必要です（この拡張機能はインタプリタ実行の Node CLI で、
シム `gh-looppilot` が `node cli.cjs` を exec します）。

## コマンド

| コマンド | 内容 |
|---|---|
| `gh looppilot init` | ツールチェーンを検出し、`.github/workflows/looppilot-{init,loop}.yml`（`@v1` に固定された薄い caller）を書き出し、ゲートラベルを作成し、`CHECK_COMMAND` を提案し、手動（自動化できない）手順を表示してから pre-flight を実行します。 |
| `gh looppilot doctor` | 読み取り専用の pre-flight のみを実行します（= `init --preflight-only`）。 |

主なフラグ: `--full-auto`（ゲートラベルなし）、`--same-repo`（`secrets: inherit`）、
`--label <name>`、`--check-command <c>`、`--ref <ref>`、`--repo <owner/repo>`、
`--dry-run`、`--force`、`--no-preflight`、`--json`（doctor）。`gh looppilot --help` を参照してください。

pre-flight の終了コード: `0` = エラーなし（警告 / unknown は許容）、`1` = 初回 PR の
前に修正すべきエラー、`2` = チェック実行自体が進められなかった（認証 / リポジトリ解決）。

## pre-flight チェック（`doctor`）

開発者の認証済み `gh` セッションに対する読み取り専用チェックです。各チェックは、
そうでなければ初回 PR の後にしか現れないサイレント障害の各種類を表面化させます。
権限がなく答えを判定できないチェックは `unknown` を返します（サイレントなパスには
なりません）。403 が返った場合はプローブ単位で縮退（degrade）します。

| チェック id | 表面化させる内容 | ステータス |
|---|---|---|
| `label.gate` | ゲートラベルが欠落 → Actions の実行が生成されない | ok / error / unknown |
| `secret.anthropicAuth` | Anthropic 認証情報の二重設定 / 未設定（repo + org secrets） | ok / error / unknown |
| `codex.connection` | Codex GitHub App の接続（最近の bot アクティビティから推定） | ok / **unknown** |
| `secret.loopPilotPushToken` | required checks / auto-merge が有効だが push token が欠落 | ok / warning / unknown |
| `autoMerge.config` | `LOOPPILOT_AUTO_MERGE=true` だがリポジトリの「Allow auto-merge」がオフ | ok / error / unknown |
| `secret.codexReviewToken` | `CODEX_REVIEW_REQUEST_TOKEN` が欠落（推奨） | ok / warning / unknown |
| `toolchain.checkCommand` | CHECK_COMMAND が安全でない / 検出したツールチェーンと不整合 | ok / warning / error |

### `--json` スキーマ

```json
{
  "ok": false,
  "repository": "owner/repo",
  "checks": [
    {
      "id": "secret.loopPilotPushToken",
      "status": "ok|warning|error|unknown",
      "summary": "short human text",
      "details": "actionable detail or null",
      "nextSteps": ["concrete command or UI step"]
    }
  ]
}
```

`ok` が `false` になるのは、いずれかのチェックが `error` の場合だけです。フィールドの
順序は安定しています。終了コードは `ok`（0/1）に対応し、実行を開始できなかった場合は
`2` になります。

## 開発

```bash
npm install
npm run cli -- doctor --json   # tsx 経由でソースから実行
npm run check                  # tsc --noEmit + vitest run
npm run bundle                 # コミット済みの cli.cjs を再ビルド（esbuild）
gh extension install .         # このローカルチェックアウトを拡張機能としてインストール
```

`cli.cjs` は、インストール済みの拡張機能が実行するコミット済みバンドルです（gh は
インタプリタ実行の拡張機能に対して `npm install` を行いません）。**`src/` を変更する
たびに再ビルドしてコミットしてください**（`npm run bundle`）。

レイアウト: `gh-looppilot`（シム）· `cli.cjs`（コミット済みバンドル）· `src/`（TS）·
`tests/`（vitest）。

## ライセンス

MIT（[team-yubune/loop-pilot](https://github.com/team-yubune/loop-pilot) と同じ）。
