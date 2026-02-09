# mcp-on-cloudrun-app

Cloud Run 上で動作する MCP (Model Context Protocol) サーバーのアプリケーションリポジトリ。

## 関連リポジトリ

インフラ（Terraform）は別リポジトリで管理しています：

- [mcp-on-cloudrun-terraform](https://github.com/Kohei-Suzuki22/mcp-on-cloudrun-terraform) — Artifact Registry、Cloud Run、Workload Identity Federation、IAM などのリソースを管理

## 構成

| ファイル | 説明 |
|----------|------|
| `server.py` | FastMCP サーバー本体（`add`, `subtract` ツールを提供） |
| `Dockerfile` | Python 3.13-slim + uv によるコンテナイメージ |
| `pyproject.toml` | Python プロジェクト定義（依存: `fastmcp==2.13.1`） |
| `.github/workflows/deploy.yml` | GitHub Actions CI/CD ワークフロー |

## アーキテクチャ

```
GitHub (push to main)
  │
  ├─ GitHub Actions (Workload Identity Federation で認証)
  │    ├─ Docker build & push → Artifact Registry
  │    └─ Deploy → Cloud Run
  │
  └─ Cloud Run (asia-northeast1)
       └─ MCP Server (streamable-http, port 8080)
            ├─ /mcp エンドポイント
            ├─ add(a, b) → a + b
            └─ subtract(a, b) → a - b
```

## ローカル開発

```bash
uv sync
uv run server.py
```

サーバーが `http://localhost:8080` で起動します。

## Cloud Run への接続

Cloud Run 上のサービスは IAM 認証が必要です（`roles/run.invoker`）。接続方法は2つあります。

### 方法 1: ID トークンを直接指定（proxy 不要）

Cloud Run の URL に直接接続する方法です。MCP 設定の `headers` に ID トークンを指定します。

**1. ID トークンを取得**

```bash
export ID_TOKEN=$(gcloud auth print-identity-token)
```

`gcloud auth login` で認証済みのアカウントに `roles/run.invoker`（または `roles/owner`）が必要です。

**2. claude.json に MCP サーバーを設定**

```json
{
  "mcpServers": {
    "mcp-on-cloudrun": {
      "type": "http",
      "url": "https://mcp-on-cloudrun-app-l5bsvyyweq-an.a.run.app/mcp",
      "headers": {
        "Authorization": "Bearer ${ID_TOKEN}"
      }
    }
  }
}
```

環境変数は `${ID_TOKEN}` の構文で展開されます（`$ID_TOKEN` では展開されません）。

**3. Claude Code を起動して接続確認**

ID トークンの有効期限は **1時間** です。期限切れ後は `export ID_TOKEN=$(gcloud auth print-identity-token)` を再実行して Claude Code を再起動してください。

### 方法 2: Cloud Run Proxy 経由

プロキシを経由して接続する方法です。プロキシが認証トークンを自動付与するため、MCP 設定にトークンを書く必要がありません。

#### Cloud Run Proxy とは

`gcloud run services proxy` は、ローカルマシンに認証付きのプロキシサーバーを立ち上げるコマンドです。ローカルへのリクエストに Google の認証トークンを自動付与し、Cloud Run サービスに転送します。

```
クライアント → localhost:8080 → [proxy が認証トークンを付与] → Cloud Run サービス
```

#### プロキシの認証情報

プロキシは **`gcloud auth login` の認証情報**を使用します（ADC ではありません）。

- `gcloud auth login` — `gcloud` コマンドや `gcloud run services proxy` が使う認証情報
- `gcloud auth application-default login`（ADC） — Terraform やアプリケーションが使う認証情報

これらは独立したもので、ADC を変更してもプロキシの認証には影響しません。

#### --impersonate-service-account の制限

`gcloud run services proxy --impersonate-service-account` フラグは存在しますが、Cloud Run の認証に必要な ID トークンの生成が借用した認証情報からはサポートされておらず、以下のエラーで失敗します：

```
failed to find token source: failed to get idtoken source: idtoken: unsupported credentials type
```

そのため、プロキシは常に `gcloud auth login` のアカウントで認証されます。

#### プロキシのポート

デフォルトは `8080` ですが `--port` で変更可能です。プロキシのポートと Cloud Run コンテナのポートは独立しており、揃える必要はありません。

#### プロキシと Cloud Run サービスの紐づけ

プロキシ起動時のコマンド引数（サービス名、`--region`、`--project`）で Cloud Run サービスが一意に特定されます。設定ファイルではなく、コマンド引数そのものが紐づけです。

#### 接続手順

**1. プロキシを起動**

```bash
gcloud run services proxy mcp-on-cloudrun-app \
  --region=asia-northeast1 \
  --project=szkou-mcp-practice
```

`gcloud auth login` で認証済みのアカウントに `roles/run.invoker`（または `roles/owner`）が必要です。

**2. Claude Code に MCP サーバーを追加（別ターミナル）**

```bash
claude mcp add --transport http mcp-on-cloudrun http://localhost:8080/mcp
```

**3. Claude Code で接続確認**

Claude Code を起動し、`add` や `subtract` ツールが利用できることを確認します。

### 方法の比較

| | 方法 1（ID トークン直接指定） | 方法 2（Cloud Run Proxy） |
|---|---|---|
| proxy の起動 | 不要 | 毎回必要 |
| トークン管理 | 手動取得（1時間で失効） | proxy が自動管理 |
| 設定の簡潔さ | claude.json のみ | proxy + claude.json |
| SA 借用での認証テスト | `--impersonate-service-account` で可能 | `cloud-run-proxy -token` が必要 |

### 認証の検証（SA 借用による方法）

`gcloud auth login` のアカウントが `roles/owner` を持っている場合、通常の方法では常に認証が成功してしまいます。特定の SA の権限で認証の成功/失敗を検証するには、以下の方法を使用します。

#### 前提

- Terraform で `mcp-client` SA を作成済み（`mcp-client@szkou-mcp-practice.iam.gserviceaccount.com`）

#### 方法 A: ID トークン直接指定で検証

```bash
# SA の ID トークンを取得
export ID_TOKEN=$(gcloud auth print-identity-token \
  --impersonate-service-account=mcp-client@szkou-mcp-practice.iam.gserviceaccount.com)
```

claude.json で `${ID_TOKEN}` を使い、Claude Code を起動して接続を確認します。

- SA に `roles/run.invoker` あり → 接続成功
- SA に `roles/run.invoker` なし → 403 Forbidden

#### 方法 B: cloud-run-proxy -token で検証

`cloud-run-proxy` の `-token` フラグに SA の ID トークンを渡してプロキシを起動します。

**認証成功テスト**（`roles/run.invoker` 付与済み）:

```bash
TOKEN=$(gcloud auth print-identity-token \
  --impersonate-service-account=mcp-client@szkou-mcp-practice.iam.gserviceaccount.com \
  --audiences=https://mcp-on-cloudrun-app-l5bsvyyweq-an.a.run.app)

cloud-run-proxy \
  -host https://mcp-on-cloudrun-app-l5bsvyyweq-an.a.run.app \
  -token "$TOKEN"
```

**認証失敗テスト**（`roles/run.invoker` 剥奪済み）:

同じコマンドで 403 Forbidden になります。

#### invoker の付与/剥奪

```bash
# 付与
gcloud run services add-iam-policy-binding mcp-on-cloudrun-app \
  --region=asia-northeast1 \
  --project=szkou-mcp-practice \
  --member="serviceAccount:mcp-client@szkou-mcp-practice.iam.gserviceaccount.com" \
  --role="roles/run.invoker"

# 剥奪
gcloud run services remove-iam-policy-binding mcp-on-cloudrun-app \
  --region=asia-northeast1 \
  --project=szkou-mcp-practice \
  --member="serviceAccount:mcp-client@szkou-mcp-practice.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

#### なぜ cloud-run-proxy -token を使うのか

`gcloud run services proxy` は `gcloud auth login` の認証情報を使用するため、`roles/owner` を持つアカウントでは常に成功してしまいます。`--impersonate-service-account` フラグは ID トークン生成の制限により動作しません。

`cloud-run-proxy -token` を使うと、手動で取得した SA の ID トークンをプロキシに渡せるため、特定の SA の権限で認証テストが可能になります。ただし ID トークンは約1時間で失効します。

#### 認証失敗の確認（curl）

```bash
CLOUD_RUN_URL=$(gcloud run services describe mcp-on-cloudrun-app \
  --region=asia-northeast1 \
  --project=szkou-mcp-practice \
  --format="value(status.url)")

# 認証なしで直接アクセス → 403 Forbidden
curl -X POST ${CLOUD_RUN_URL}/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```
