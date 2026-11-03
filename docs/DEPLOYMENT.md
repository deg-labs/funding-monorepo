# GCE CI/CD デプロイ設定

このリポジトリの [deploy-to-gce](../.github/workflows/deploy.yml) は、[共通ワークフロー `deg-labs/github-workflows/.github/workflows/gce-deploy.yml`](https://github.com/deg-labs/github-workflows/blob/8b7ede731f512eabd5092c6270a5bf95ffe68216/.github/workflows/gce-deploy.yml) を固定コミットで呼び出し、Google Compute Engine (GCE) 上の Docker Compose アプリケーションを更新します。**Secret名はその共通ワークフローの現行実装に基づきます。** JSON鍵は使用せず、GitHub Actions OIDC と Google Cloud Workload Identity Federation (WIF) で認証します。

## 1. 初回セットアップ

1. GCPに `dev` / `stg` / `prd` 用のGCEインスタンスを用意します。Ubuntu/Debian系のSSH可能なVMにDocker Engine、Docker Compose v2プラグインを導入し、ワークフローのSSHユーザーが `sudo docker compose` を無人実行できるようにします。SSH経路（パブリックIP等）とファイアウォール、必要に応じOS Loginを設定します。ワークフローはIAPトンネルを明示していません。VMへの接続ができない構成は、先に共通ワークフロー側の対応が必要です。
2. GCPでデプロイ用サービスアカウントとWorkload Identity Pool/Providerを作成します。ProviderのGitHub発行元は `https://token.actions.githubusercontent.com` で、Organization、リポジトリ、ref、利用するenvironmentに応じて対象を絞り込みます。サービスアカウントには該当Providerからの `roles/iam.workloadIdentityUser`（通常は `principalSet://...` 条件付きバインド）を付与します。GCEインスタンスの参照、`dev`/`stg` の起動、SSH/SCPに必要な最小権限（Compute管理、必要に応じOS Login/サービスアカウント使用権限）を個別に確認します。管理者権限を一律に与えないでください。
3. GitHubリポジトリの **Settings → Environments** で **`dev`、`stg`、`prd`** を正確な名前で作成します。少なくとも `prd` にはRequired reviewersや許可ブランチ等の保護を設定してください。**このYAMLはPRに `prd` ラベルを付けた場合も本番デプロイを試みるため、環境保護なしで運用しないでください。** 利用可能な保護ルールはGitHubプランとリポジトリ可視性によって異なります。
4. GitHub Organizationのチーム `deg-develop` を確認し、起動するユーザーと（Dependabot PRを除く）PR作成者が所属するようにします。下記GitHub Appの認証も登録します。
5. 下表のSecretsを **Settings → Secrets and variables → Actions** から登録します。共通ワークフローは `secrets: inherit` で値を受け取ります。**Environment SecretかRepository/Organization Secretかはこのワークフローの認証境界を考慮して選択**し、環境別に分離します。Organization Secretを使う場合、このリポジトリへのアクセス許可が必要です。

## 2. GitHub Actions Secrets

| Secret名 | 使う環境 | 値と用途 |
| --- | --- | --- |
| `DEV_GCE_INSTANCE_NAME`, `STG_GCE_INSTANCE_NAME`, `PRD_GCE_INSTANCE_NAME` | 各環境 | GCEのVMインスタンス名 |
| `DEV_GCE_ZONE`, `STG_GCE_ZONE`, `PRD_GCE_ZONE` | 各環境 | VMのゾーン（例: `asia-northeast1-b`）。共通 `GCE_ZONE` もフォールバックで使用可能 |
| `DEV_GCP_SA_EMAIL`, `STG_GCP_SA_EMAIL`, `PRD_GCP_SA_EMAIL` | 各環境 | デプロイ用GCPサービスアカウントのメールアドレス |
| `DEV_GCP_WIF_PROVIDER`, `STG_GCP_WIF_PROVIDER`, `PRD_GCP_WIF_PROVIDER` | 各環境 | WIF Providerのフルリソース名（`projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL/providers/PROVIDER`）。プロジェクトIDではなく番号を使用 |
| `DEV_APP_ENV`, `STG_APP_ENV`, `PRD_APP_ENV` | 各環境 | 実行時の **.envファイル全文**。単一のWebhook URLだけではなく、Composeが参照する設定を複数行で保存 |
| `ORG_GH_APP_ID` | チーム制限がある全環境 | チームメンバーの所属確認に使うGitHub App ID |
| `ORG_GH_APP_PRIVATE_KEY` | チーム制限がある全環境 | 同GitHub AppのPEM秘密鍵全文 |

共通ワークフローには本番向けの旧互換名 `PROD_GCE_INSTANCE_NAME` / `PROD_GCE_ZONE` / `PROD_GCP_SA_EMAIL` / `PROD_GCP_WIF_PROVIDER` へのフォールバックがあります。ただし新規設定では `PRD_*` に統一してください。`DEV_APP_ENV` などは空にできません。GCPアカウントID、プロバイダーやSecret**の実値**をリポジトリにコミットしないでください。

GitHub Appは`deg-labs`にインストールし、チームメンバー一覧の読み取りに必要なOrganization **Members: read** 権限を付与します。アプリの認証なしに所属確認を成功させる構成ではありません。ID・秘密鍵を登録するだけでなく、Appのインストールと対象Organizationへのアクセスも確認してください。

### アプリの.envの例

下記は形式の例です。**Webhookの実URLを値に入れ、実行環境に合わせて不足する設定を追加**します。各環境の完全なテンプレートは [`.env.example`](../.env.example) を参照してください。

```dotenv
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/REPLACE_WITH_REAL_URL
```

Actions Secretへは `cp .env.example` だけでなく、必要な変数を満たしたdotenvの**内容全体**を登録します。配置時に共有ワークフローが `DEPLOY_ENV=dev|stg|prd` を追加してVMの `/opt/apps/funding-monorepo/.env` に保存します。Secretを設定しただけではデプロイは起動しません。

## 3. 実行条件

| トリガー | デプロイ先 | 注意点 |
| --- | --- | --- |
| `main` へのpush | `prd` | PRマージ後の本番自動デプロイ |
| Actions → `deploy-to-gce` → Run workflow | 入力 `dev` / `stg` / `prd` | **`main` refから実行**。共通側が他refでの手動デプロイを拒否 |
| PRに `dev` / `stg` / `prd` ラベルを新たに追加 | ラベルと同じ環境 | `pull_request: labeled` のイベントがトリガー。既にラベルが付いているPRへのpushでは再実行しない。再実行はラベルを一度外して付け直すか、別のトリガーを使う |

通常のPRやブランチpushは `Docker Compose Build` がビルド検証を行いますが、ラベルなしPRのdeployは実行されません。デプロイ側の `prepare` ジョブはサブモジュールを再帰的にチェックアウトして、コミット識別子付きのtar.gzを作成します。`.env`・`.git`・`.github` および [`.gce-deployignore`](../.gce-deployignore) で指定した永続化ディレクトリは成果物から除外します。

共通側でアーカイブ・Compose構成を検証後、GCEに転送し、`/opt/apps/funding-monorepo` に展開して `sudo docker compose -f docker-compose.yml up -d --build --remove-orphans` を実行します。`dev`/`stg` では停止VMの起動も試みます。本番ではインスタンス起動処理はありません。

> **セキュリティ:** PRラベルを使うdeployはPRのコードとComposeをVM上で実行します。チーム所属確認と`environment`の承認を利用しても、未レビューのPRへ`prd`ラベルを付けるべきではありません。共有ワークフローにある`enforce_deploy_team: true` は明示的に維持してください。

## 4. サブモジュール・永続化と初回動作確認

- `server/` と `client/` はGitサブモジュールです。**子リポジトリのmainを更新しただけではデプロイに反映されません。** このリポジトリのsubmoduleポインター更新PRをマージしてください。
- publicリポジトリのサブモジュールは `actions/checkout` がHTTPSとして取得できる構成です。非公開へ変更した場合は追加のリポジトリアクセス設定が必要です。
- `server/data` と `client/ncmma/data` などの永続データはデプロイ成果物から除外され、既存ディレクトリは削除されません。**除外はバックアップではありません**。バックアップと復元テストを別途実施してください。
- GitHub Actionsのrunで `prepare → verify → deploy` の各ジョブを確認します。デプロイ先では `cd /opt/apps/funding-monorepo && sudo docker compose ps`、`sudo docker compose logs --tail=100` で状態を確認してください。
- エラーの切り分け: `prepare`失敗ならサブモジュール取得とtar除外設定、`verify`失敗ならComposeの構成・禁止バインドマウント、`deploy`の認証失敗ならWIFの属性条件とGitHub App権限、SSH失敗ならVMの状態とネットワーク、`docker compose`失敗なら.envの必須変数とコンテナログを確認します。
- 秘密値はActionsログやIssue/PR、スクリーンショットに貼り付けないでください。ロールバックは前の安全なコミットへsubmoduleポインターとCompose定義を戻すPR/コミットを作り再deployします。DBのロールバックが必要なら別に整備した復元手順を利用してください。

## 5. Secrets登録後の実施チェック

- [ ] `dev` / `stg` / `prd` Environmentsと本番承認を設定
- [ ] GitHub AppのOrganizationチーム参照権限と2つのApp Secretsを登録
- [ ] 各環境のGCE接続/WIF情報と `*_APP_ENV` を登録
- [ ] GCEへのSSH・sudo・Docker Compose実行経路を検証
- [ ] Actionsから`main`の`dev`を手動実行し、VM内のコンテナ状態を検証
- [ ] 同様に`stg`を確認した後、`prd`の承認設定を確認

## 6. funding-monorepo 固有の確認

- デプロイ先: `/opt/apps/funding-monorepo`。外部に公開するNginxのポートは `8002`（現行 [Compose](../docker-compose.yml)）。アクセス制限とファイアウォールを環境に合わせて設定します。
- `*_APP_ENV` には `DISCORD_WEBHOOK_URL` を必ず入力します。通知クライアントが使用する `CMMA_API_BASE_URL=http://nginx` などの設定も [`.env.example`](../.env.example) に従って含めます。
- デプロイ後のAPI確認例: `curl -fsS 'http://127.0.0.1:8002/funding-rates?threshold=0.001&limit=1'`（VM上）。ポートの到達可能性とパスは現在の構成で確認してください。
- Python旧版からGo版への切替時は [MIGRATION.md](../MIGRATION.md) のDBバックアップ・比較・切り戻し手順を先に実施します。
