# Scutum WAF — Microsoft Sentinel Codeless Connector (CCF)

[Scutum](https://www.scutum.jp/) WAF の防御ログを Microsoft Sentinel に取り込むための Codeless Connector Framework (CCF) コネクターです。

## 概要

このコネクターは Scutum WAF の REST API を使用して防御ログ（WAF アラート）を定期的にポーリングし、Microsoft Sentinel の Log Analytics ワークスペースに取り込みます。

### 主な機能

- Scutum WAF の防御ログをリアルタイムに近い形で Sentinel に取り込み
- ログの詳細情報（HTTP リクエスト/レスポンスの Base64 データ）も自動取得
- 複数ホスト（FQDN）の同時接続に対応
- 接続一覧グリッドで各ホストの状態を確認可能

## アーキテクチャ

```
Scutum API                    Microsoft Sentinel
┌──────────────────┐          ┌──────────────────────────┐
│ /api/v1/alert    │──GET───▶│ CCF RestApiPoller        │
│ (ログ一覧)        │          │   ├─ 認証: API Key       │
├──────────────────┤          │   ├─ ページング: marker    │
│ /api/v1/         │◀─GET───│   └─ ネスト取得           │
│ alert_detail     │          │                          │
│ (ログ詳細)        │──JSON──▶│ DCR (変換 KQL)           │
└──────────────────┘          │   └─▶ ScutumAlert_CL     │
                              └──────────────────────────┘
```

## 前提条件

- Microsoft Sentinel が有効化された Log Analytics ワークスペース
- ワークスペースへの読み取り/書き込み権限
- Scutum API キー（[APIキーの管理](https://support.scutum.jp/manual/waf-setting/api-key.html) から取得）
- Scutum 管理画面の FQDN と管理コンソール ID

## ファイル構成

```
sentinel-connectors/Scutum_CCF/
├── Scutum_ConnectorDefinition.json   # Sentinel UI コネクター定義
├── Scutum_PollingConfig.json         # API ポーリング設定
├── Scutum_Table.json                 # カスタムテーブル スキーマ
└── Scutum_DCR.json                   # データ収集ルール
```

## カスタムテーブル: `ScutumAlert_CL`

| カラム名 | 型 | 説明 |
|---|---|---|
| `TimeGenerated` | datetime | Scutum が通信を検知・ブロックした日時 |
| `LogId` | string | 防御ログ ID |
| `IpAddress` | string | リクエスト送信元 IP アドレス |
| `Block` | boolean | 通信がブロックされたか（WAF OFF や偽陽性の場合は `false`） |
| `Category` | dynamic | 検知した攻撃分類の配列（SQLインジェクション 等） |
| `Uri` | string | リクエストのアクセス先 URI |
| `Request` | string | Base64 エンコードされた HTTP リクエストデータ |
| `Response` | string | Base64 エンコードされた HTTP レスポンスデータ |

## デプロイ方法

### 方法 1: VS Code + Sentinel Connector Builder 拡張機能（推奨）

1. VS Code に [Microsoft Sentinel Connector Builder](https://marketplace.visualstudio.com/items?itemName=ms-sentinel.connector-builder) 拡張機能をインストール
2. このリポジトリをクローン
3. コマンドパレットから `deploy_connector` を実行し、ワークスペース名を指定

### 方法 2: Azure CLI

```bash
az deployment group create \
  --resource-group <リソースグループ名> \
  --template-file <ARM テンプレートパス> \
  --parameters workspace=<ワークスペース名>
```

### 方法 3: Azure Portal

1. Azure Portal → 「カスタム テンプレートのデプロイ」
2. CCF ファイルを統合した ARM テンプレートを貼り付け
3. パラメーターを入力して「作成」

## 接続手順

デプロイ後、Azure Portal で以下の手順を実行します。

1. **Microsoft Sentinel** → 対象ワークスペース → **Data connectors** を開く
2. **「Scutum」** を検索してコネクターページを開く
3. 以下の情報を入力:
   - **Scutum API Key**: Scutum 管理画面から取得した API キー
   - **Host (FQDN)**: 対象サイトの FQDN（例: `www.example.com`）
   - **Scutum Management Console ID**: 管理コンソール ID（例: `ABC1234`）
4. **Connect** をクリック

### 複数ホストの追加

同じコネクターで複数のホストを接続できます。異なる FQDN / ID / API キーの組み合わせで **Connect** を繰り返すだけです。「Active Connections」セクションに接続一覧が表示されます。

## KQL クエリ例

### 最新のアラートを表示

```kql
ScutumAlert_CL
| sort by TimeGenerated desc
| take 100
```

### ブロックされた攻撃を種別ごとに集計

```kql
ScutumAlert_CL
| where Block == true
| mv-expand Category
| summarize Count = count() by tostring(Category)
| sort by Count desc
```

### 送信元 IP アドレス Top 10

```kql
ScutumAlert_CL
| summarize Count = count() by IpAddress
| top 10 by Count
```

### 特定 URI への攻撃を確認

```kql
ScutumAlert_CL
| where Uri contains "/admin"
| project TimeGenerated, IpAddress, Category, Uri, Block
| sort by TimeGenerated desc
```

## API 仕様

| 項目 | 内容 |
|---|---|
| エンドポイント | `https://api.scutum.jp/api/v1/alert` |
| 認証 | HTTPヘッダー `X-Scutum-API-Key` |
| メソッド | GET |
| レート制限 | 25回 / 5分 / APIキー |
| ページネーション | `marker` パラメーター（最大 1000 件/回） |
| 時刻フォーマット | ISO 8601（`yyyy-MM-ddTHH:mm:ss+09:00`） |
| データ保持期間 | 過去1年間 |

詳細は [Scutum API ドキュメント](https://support.scutum.jp/manual/api/api-overview.html) を参照してください。

## ライセンス

MIT License

## 参考リンク

- [Scutum 公式サイト](https://www.scutum.jp/)
- [Scutum API マニュアル](https://support.scutum.jp/manual/api/api-overview.html)
- [Microsoft Sentinel CCF ドキュメント](https://learn.microsoft.com/azure/sentinel/create-codeless-connector)
- [CCF データコネクタ定義リファレンス](https://learn.microsoft.com/azure/sentinel/data-connector-ui-definitions-reference)
