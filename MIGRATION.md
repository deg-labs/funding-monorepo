# 旧構成からの切り替え

`funding-cmma`（Python / FastAPI）から `funding-cmma-go`（Go）へ切り替える手順です。既存の `cmma.db` と HTTP API の互換性を維持することを前提とします。

## 方針

- データベースのスキーマと履歴を引き継ぎます。
- 公開 API のパス、パラメータ、JSON の形を維持します。
- 通知クライアント `funding-ncmma` の設定を変更しません。
- 旧環境は切り戻せる状態で一定期間残します。
- 切り替え前に新旧のレスポンスを同じデータで比較します。

## 1. 事前確認

### データベースのバックアップ

```bash
cd /opt/apps/funding-cmma
sudo docker compose stop fetcher
sudo sqlite3 data/cmma.db ".backup '/tmp/cmma-$(date +%Y%m%d).db'"
sudo docker compose start fetcher
```

WAL モードで運用している場合は、`fetcher` を停止してからバックアップしてください。

### スキーマの確認

```bash
sqlite3 data/cmma.db ".schema funding_rates"
```

`funding_rates` の列と主キーが次のとおりであることを確認します。

```text
PRIMARY KEY (symbol, funding_rate_timestamp)
```

異なる場合は Go 版を起動する前に原因を調査してください。

### ポートの確認

旧構成と新構成はどちらも 8002 番ポートを使います。同時に起動できないため、切り替え時は旧 `nginx` を停止します。

```bash
sudo ss -ltnp | grep ':8002'
```

## 2. 並行検証

旧構成を止めずに検証する場合は、データベースのコピーを使います。

```bash
mkdir -p /tmp/funding-verify/data
sqlite3 /opt/apps/funding-cmma/data/cmma.db ".backup '/tmp/funding-verify/data/cmma.db'"
```

Go 版の API だけを別ポートで起動します。

```bash
cd /opt/apps/funding-monorepo
DB_PATH=/tmp/funding-verify/data/cmma.db \
HTTP_ADDR=127.0.0.1:18002 \
./server/api  # バイナリを配置している場合
```

Docker を使う場合は次のとおりです。

```bash
docker run --rm -d --name funding-cmma-api-verify \
  -p 18002:8000 \
  -v /tmp/funding-verify/data:/app/data \
  funding-cmma-api:verify
```

### 比較する項目

| 項目 | 確認内容 |
| --- | --- |
| `threshold` | 同じ銘柄が抽出される |
| `direction` | `positive`、`negative`、`both` が一致する |
| `sort` | 並び順が一致する |
| `limit` | 件数が一致する |
| `utilization_gte` | 抽出結果が一致する |
| `funding.utilization` | 小数 7 桁まで一致する |
| `null` | 欠損値が `null` になる |
| 0 件 | `data` が `[]` になる |
| 継続性分析 | 3 つの統計値が一致する |
| エラー | 422 と 404 のステータスと形式が一致する |

```bash
for query in \
  "threshold=0.001" \
  "threshold=0.001&direction=positive" \
  "threshold=0.001&direction=negative" \
  "threshold=0.001&sort=utilization_desc" \
  "threshold=0.001&limit=5" \
  "threshold=0.001&utilization_gte=0.5"
do
  echo "== ${query}"
  curl -s "http://127.0.0.1:8002/funding-rates?${query}" > "/tmp/old-$(echo "$query" | tr '&=' '__').json"
  curl -s "http://127.0.0.1:18002/funding-rates?${query}" > "/tmp/new-$(echo "$query" | tr '&=' '__').json"
done
```

JSON のキー順は比較対象外です。値、配列の順序、型、`null` を比較してください。

## 3. 切り替え

1. 旧構成を停止します。

   ```bash
   cd /opt/apps/funding-cmma
   sudo docker compose down
   ```

2. データベースを新しい配置先へ移します。既存の履歴を保持するため、コピーではなく移動します。

   ```bash
   sudo mkdir -p /opt/apps/funding-monorepo/server/data
   sudo mv /opt/apps/funding-cmma/data/cmma.db /opt/apps/funding-monorepo/server/data/cmma.db
   sudo chown -R "$(id -u):$(id -g)" /opt/apps/funding-monorepo/server/data
   ```

   `cmma.db-wal` と `cmma.db-shm` が残っている場合は、`fetcher` 停止後に合わせて移動してください。停止済みであれば削除しても問題ありません。

3. `.env` を配置します。`funding-cmma` の `.env` を基に、`funding-monorepo` の `.env.example` へ不足項目を追加します。

   ```bash
   cd /opt/apps/funding-monorepo
   cp .env.example .env
   ```

4. 起動します。

   ```bash
   sudo docker compose up -d --build
   ```

5. 状態を確認します。

   ```bash
   sudo docker compose ps
   curl -s http://127.0.0.1:8002/readyz
   curl -s "http://127.0.0.1:8002/funding-rates?threshold=0.001&limit=1"
   ```

6. `fetcher` のログで、取得件数と失敗件数を確認します。

   ```bash
   sudo docker compose logs --tail=100 funding-cmma-go-fetcher
   ```

7. 通知クライアントのログを確認します。

   ```bash
   sudo docker compose logs --tail=100 funding-ncmma-notifier
   ```

## 4. 切り戻し

問題が発生した場合は旧構成へ戻します。

1. 新構成を停止します。

   ```bash
   cd /opt/apps/funding-monorepo
   sudo docker compose down
   ```

2. データベースを戻します。

   ```bash
   sudo mv /opt/apps/funding-monorepo/server/data/cmma.db /opt/apps/funding-cmma/data/cmma.db
   sudo chown -R "$(id -u):$(id -g)" /opt/apps/funding-cmma/data
   ```

3. 旧構成を起動します。

   ```bash
   cd /opt/apps/funding-cmma
   sudo docker compose up -d
   ```

Go 版の `fetcher` が書き込んだ履歴は旧版でもそのまま読めます。スキーマを変更していないためです。

## 5. チェックリスト

- [ ] `cmma.db` のバックアップを取得した
- [ ] `funding_rates` のスキーマが既存と一致する
- [ ] 新旧のレスポンスを同じデータで比較した
- [ ] `funding.utilization` の負値が旧版と一致する
- [ ] 0 件時に `data` が `[]` になる
- [ ] 422 と 404 の形式が一致する
- [ ] 継続性分析の 3 指標が一致する
- [ ] `funding-ncmma` の通知が届く
- [ ] 8002 番ポートの競合がない
- [ ] 切り戻し手順を確認した

## 6. 既知の差分

| 項目 | 旧実装 | Go 実装 | 影響 |
| --- | --- | --- | --- |
| OpenAPI のバージョン | OpenAPI 3 | Swagger 2.0 | ドキュメントの形式のみ。API の応答は変わりません |
| ソートの同値順 | 不定 | `symbol` 昇順で固定 | 同値時の並びが安定します |
| 履歴の既定取得件数 | 10 件 | 200 件（初回） | 継続性分析が起動直後から利用できます |
| ログ出力 | ファイルと標準出力 | 標準出力と標準エラー | コンテナ運用向けです |
| レート制御 | 同時実行数のみ | 同時実行数と秒間リクエスト数 | 取引所の制限に当たりにくくなります |

負の Funding Rate の利用率は旧実装と同じく負の値になります。API 仕様の「0.0〜1.0」とは異なりますが、移行時の挙動変更を避けるため維持しています。修正する場合は `utilization_gte` の検索結果も変わるため、別の変更として扱ってください。
