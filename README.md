# funding-monorepo

`funding-cmma-go`（サーバー）と `funding-ncmma`（通知クライアント）をまとめてデプロイするリポジトリです。両リポジトリは Git サブモジュールとして参照します。

## リポジトリ構成

| パス | サブモジュール | 役割 |
| --- | --- | --- |
| `server` | [deg-labs/funding-cmma-go](https://github.com/deg-labs/funding-cmma-go) | Funding Rate の収集、SQLite、HTTP API |
| `client` | [deg-labs/funding-ncmma](https://github.com/deg-labs/funding-ncmma) | 異常金利の Discord 通知 |

```text
funding-monorepo/
├── .github/workflows/   ビルドとデプロイ
├── client/              funding-ncmma（サブモジュール）
├── server/              funding-cmma-go（サブモジュール）
├── docker-compose.yml
└── .env.example
```

## 必要要件

- Docker
- Docker Compose
- Git（サブモジュールを取得できること）

## ローカル実行

1. リポジトリをサブモジュール込みで取得します。

   ```bash
   git clone --recurse-submodules git@github.com:deg-labs/funding-monorepo.git
   ```

   取得済みの場合は次を実行します。

   ```bash
   git submodule update --init --recursive
   ```

2. 環境変数ファイルを作成します。

   ```bash
   cp .env.example .env
   ```

3. 通知を使う場合は `.env` の `DISCORD_WEBHOOK_URL` を設定します。

4. コンテナを起動します。

   ```bash
   docker compose up --build -d
   ```

5. 動作を確認します。

   ```bash
   curl -s "http://localhost:8002/funding-rates?threshold=0.001&limit=1"
   ```

6. 停止します。

   ```bash
   docker compose down
   ```

## サービス構成

| サービス | コンテナ名 | 説明 |
| --- | --- | --- |
| `fetcher` | `funding-cmma-go-fetcher` | 取引所から取得して SQLite へ保存 |
| `api` | `funding-cmma-go-api` | SQLite を読み取り API を提供 |
| `nginx` | `funding-cmma-go-nginx` | 8002 番ポートで公開 |
| `funding-ncmma-notifier` | `funding-ncmma-notifier` | 異常金利を Discord へ通知 |

`api` は `server/data` を書き込み可能でマウントします。SQLite の WAL は共有メモリファイルを作成するため、ディレクトリ全体を読み取り専用にはできません。読み取り専用は接続 URI と `query_only` で制御します。

## 環境変数

サーバー側の設定は [funding-cmma-go の README](https://github.com/deg-labs/funding-cmma-go#環境変数) を参照してください。このリポジトリで追加する設定は次のとおりです。

| 変数 | 既定値 | 説明 |
| --- | --- | --- |
| `DISCORD_WEBHOOK_URL` | — | 通知先の Webhook URL。必須 |
| `CMMA_API_BASE_URL` | `http://nginx` | 通知クライアントが参照する API |
| `FUNDING_RATE_DIRECTION` | `both` | 通知対象の方向 |
| `FUNDING_RATE_SORT` | `funding_abs_desc` | 通知候補のソート順 |
| `FUNDING_RATE_LIMIT` | `100` | 1 回に取得する候補数 |
| `FUNDING_RATE_LOOKBACK` | `24` | 継続性分析の履歴件数 |
| `MAX_NOTIFICATIONS` | `20` | 1 回の通知の最大件数 |
| `RENOTIFY_BUFFER_MINUTES` | `240` | 同一銘柄の再通知までの待機時間 |
| `CHECK_INTERVAL_SECONDS` | `300` | 通知クライアントの確認間隔 |
| `LOG_MAX_SIZE_MB` | `10` | 通知クライアントのログ上限 |

## デプロイ

GitHub ActionsでGCEへデプロイするための**初回設定、環境別のSecrets（`DEV_*` / `STG_*` / `PRD_*`、`*_APP_ENV`、GitHub App）、WIF、Environment保護、実行・障害対応**は [CI/CD デプロイ設定ガイド](docs/DEPLOYMENT.md) を参照してください。

`main` への push、または GitHub Actions の手動実行で GCE インスタンスへデプロイします。

- `dev` と `stg` はプルリクエストへのラベル付与でデプロイできます。
- 本番は `main` への push、または `workflow_dispatch` でデプロイします。
- デプロイ先は選択した environment で保護されます。
- 認証には Workload Identity Federation を使います。

デプロイ先は `/opt/apps/funding-monorepo` です。

## 旧構成からの切り替え

`funding-cmma` から切り替える手順は [MIGRATION.md](MIGRATION.md) を参照してください。

## ライセンス

各サブモジュールのライセンスに従います。
