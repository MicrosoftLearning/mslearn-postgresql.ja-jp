---
lab:
  title: Foundry Agent Service と PostgreSQL を使用して AI エージェントを構築してテストする
  module: Implement generative AI agents with Azure Database for PostgreSQL
  description: この演習では、Margie's Travel 社がセマンティック検索を使用して Azure Database for PostgreSQL から物件一覧を取得するインテリジェント エージェントを作成するのを支援します。 このエージェントは Foundry Agent Service で実行され、軽量の Python Azure 関数を使って PostgreSQL を呼び出します。PostgreSQL では、azureai および pgvector 拡張機能により、独自の AI 作業が実行されます。
  duration: 70 minutes
  level: 400
  islab: true
  primarytopics:
    - Azure
    - Azure Database for PostgreSQL
---

# Foundry Agent Service と PostgreSQL を使用して AI エージェントを構築してテストする

この演習では、**Margie's Travel** 社がセマンティック検索を使用して **Azure Database for PostgreSQL** から物件一覧を取得するインテリジェント エージェントを作成するのを支援します。  
このエージェントは **Foundry Agent Service** で実行され、軽量の **Python Azure Function** を使用して PostgreSQL を呼び出します。PostgreSQL では、`azure_ai` および `pgvector` 拡張機能により、独自の AI 作業が実行されます。

この演習を終了すると、次のことを行う AI エージェントが作成されます。
- 意味的類似性を使用して貸別荘の一覧を取得し、ランク付けする。
- **PostgreSQL の組み込み AI 機能**を埋め込みとベクトル検索に適用する。
- **Microsoft Foundry** を介して自然に応答する。

## 開始する前に

管理者権限と **Foundry Agent Service (プレビュー)** へのアクセス権を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

### Azure サブスクリプションにリソースをデプロイする

> 非運用環境の **Azure Database for PostgreSQL フレキシブル サーバー** と **Microsoft Foundry** プロジェクトを既に設定している場合は、このセクションをスキップできます。

**Bash** 環境で **Azure Cloud Shell** を使用して、この演習のリソースをデプロイして構成します。

1. [Azure Portal](https://portal.azure.com/)を開きます。  
1. 上部にあるツール バーで **Cloud Shell** アイコンを選択し、**[Bash]** を選択します。  
1. ラボ リソースを複製します。
   ```bash
   git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
   ```
1. リージョン、リソース グループ、PostgreSQL 管理者パスワードの変数を定義します。
   ```bash
   REGION=westus3
   RG_NAME=rg-learn-postgresql-ai-$REGION

   a=()
   for i in {a..z} {A..Z} {0..9}; do a[$RANDOM]=$i; done
   ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
   echo "Your generated PostgreSQL admin password is:"
   echo $ADMIN_PASSWORD
   ```
1. (省略可能) 複数のサブスクリプションがある場合は、サブスクリプションを設定します。
   ```azurecli
   az account set --subscription <subscriptionName|subscriptionId>
   ```
1. リソース グループを作成します。
   ```azurecli
   az group create --name $RG_NAME --location $REGION
   ```
1. 必要な Azure リソースをデプロイします。
   ```azurecli
   az deployment group create      --resource-group "$RG_NAME"      --template-file "~/mslearn-postgresql/Allfiles/Labs/Shared/deploy-all-plus-foundry.bicep"      --parameters adminLogin=pgAdmin adminLoginPassword="$ADMIN_PASSWORD" databaseName=rentals
   ```
   デプロイでは、次のものがプロビジョニングされます。
   - `rentals` データベースを含む **Azure Database for PostgreSQL フレキシブル サーバー**
   - text-embedding-ada-002 モデルがデプロイされた **Azure OpenAI Service**
   - エージェント用の **gpt-5.1** モデル デプロイを含む **Microsoft Foundry** AI サービス リソース

   > **注:** Bicep テンプレートは Foundry リソースとモデル デプロイを作成しますが、タスク 4 で Foundry **プロジェクト**を手動で作成します。 これにより、エージェント ツールが確実に動作します。

> PostgreSQL サーバーの **FQDN**、ユーザー名 (`pgAdmin`)、パスワードをメモしておいてください。これらは次の手順で使用します。

1. この演習では、2 つの方法のうち **1 つ**を使用して Azure OpenAI に対して認証を行います。 ご自分の環境に該当するものを選び、各ステップではそれらの手順にのみ従ってください。

    - **API キー**: Azure portal からコピーしたキーを使います (ほとんどの環境で機能します)。
    - **マネージド ID**: Microsoft Entra ID トークン ベースの認証を使います (組織レベルで API キーが無効になっている場合に必要です)。

    **マネージド ID** をお使いの場合は、ここで次のコマンドを実行してそれを設定します。 それ以外の場合、次のステップに進みます。

    ```bash
    # Re-derive all variables (in case your Cloud Shell session was reset)
    PGSERVER=$(az postgres flexible-server list -g "$RG_NAME" --query "[0].name" -o tsv)
    AOAI=$(az cognitiveservices account list -g "$RG_NAME" --query "[?kind=='OpenAI'].name | [0]" -o tsv)
    AOAI_ID=$(az cognitiveservices account show -g "$RG_NAME" -n "$AOAI" --query "id" -o tsv)
    SUB_ID=$(az account show --query "id" -o tsv)

    # Enable system-assigned managed identity on the PostgreSQL server
    # (The az CLI has no direct flag for this, so we use az rest per Microsoft docs)
    az rest --method patch \
      --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG_NAME/providers/Microsoft.DBforPostgreSQL/flexibleServers/$PGSERVER?api-version=2024-08-01" \
      --body '{"identity":{"type":"SystemAssigned"}}'

    # Wait for the identity to be assigned, then get the principal ID
    echo "Waiting for system-assigned managed identity..."
    SYS_MI=""
    while [ -z "$SYS_MI" ] || [ "$SYS_MI" = "null" ]; do
      sleep 15
      SYS_MI=$(az rest --method get \
        --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG_NAME/providers/Microsoft.DBforPostgreSQL/flexibleServers/$PGSERVER?api-version=2024-08-01" \
        --query "identity.principalId" -o tsv)
      echo "principalId=$SYS_MI"
    done

    # Grant 'Cognitive Services OpenAI User' to the system MI (for in-database embeddings)
    az role assignment create \
      --assignee "$SYS_MI" \
      --role "Cognitive Services OpenAI User" \
      --scope "$AOAI_ID"

    # Restart the server so it picks up the new identity
    az postgres flexible-server restart -g "$RG_NAME" -n "$PGSERVER"
    ```

    > **注:** 再起動後、サーバーが再び稼働状態になり、ロールの割り当てが反映されるまで、2 から 3 分待ちます。 後の手順で認可エラーが発生する場合は、数分待ってからもう一度お試しください。

## Azure Cloud Shell で psql を使用してデータベースに接続する

[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、ツール バーの **[Cloud Shell]** アイコンを選んで Cloud Shell を開きます。

1. 次のコマンドを実行して、`rentals` データベースに接続します。`<server-name>` は、実際の PostgreSQL フレキシブル サーバーの名前に置き換えます (Azure portal の PostgreSQL リソースの **[概要]** ページに記載されています)。

   ```bash
   psql -h <server-name>.postgres.database.azure.com -p 5432 -U pgAdmin rentals
   ```

1. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** サインイン用にランダムに生成されたパスワードを入力します。

   サインインすると、`rentals` データベースの `psql` プロンプトが表示されます。

1. この演習の残りの部分では、Cloud Shell で作業を続けます。そのため、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー ウィンドウ内でペインを拡大すると作業しやすくなります。

## タスク 1 – 拡張機能を有効にし、Azure AI 設定を構成する

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS azure_ai;
```

次に、Azure OpenAI への `azure_ai` 拡張機能の接続を構成します。 Azure OpenAI リソースのエンドポイントが必要です (Azure portal の **[リソース管理]** の下の **[キーとエンドポイント]** ページにあります)。

選択した認証方法のコマンドを実行します。

> **API キーを使用:** 同じページから使用可能なキーのいずれかをコピーします。 `KEY 1` または `KEY 2` を使用できます。

```sql
SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
SELECT azure_ai.set_setting('azure_openai.subscription_key', '{api-key}');
```

> **マネージド ID を使用:** エンドポイントのみを設定します。 `subscription_key` が構成されていない場合、拡張機能はサーバーのシステム割り当てマネージド ID を自動的に使用します。

```sql
SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
SELECT azure_ai.set_setting('azure_openai.auth_type', 'managed-identity');
```

これらの設定により、PostgreSQL は Azure AI を呼び出して埋め込みを生成することができます。

## タスク 2 – テーブルを作成し、データを読み込み、埋め込みを生成する

```sql
DROP TABLE IF EXISTS reviews CASCADE;
DROP TABLE IF EXISTS listings CASCADE;

CREATE TABLE listings (
  id BIGINT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT NOT NULL,
  property_type TEXT NOT NULL,
  room_type TEXT NOT NULL,
  price NUMERIC(10,2),
  weekly_price NUMERIC(10,2),
  listing_vector vector(1536)
);

CREATE TABLE reviews (
  id BIGINT PRIMARY KEY,
  listing_id BIGINT NOT NULL REFERENCES listings(id),
  date DATE,
  comments TEXT NOT NULL
);
```

CSV データを読み込みます。

```sql
\COPY listings (id, name, description, property_type, room_type, price, weekly_price)
  FROM '~/mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' WITH (FORMAT csv, HEADER);
```
```sql
\COPY reviews (id, listing_id, date, comments)
  FROM '~/mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' WITH (FORMAT csv, HEADER);
```

PostgreSQL 内で埋め込みを生成します。

```sql
UPDATE listings
SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
WHERE listing_vector IS NULL;
```

終わったら、「`\q`」と入力して `psql` を終了します。


## タスク 3 – Foundry エージェントが PostgreSQL に対してクエリを実行できるように API を作成する

### 3.1 Function App を作成する (ポータル)

エージェント サービスを作成する前に、そのエージェントが呼び出す API コードをホストする **Azure Function App** を作成する必要があります。

1. Azure portal を開き、**Function App** を検索して選択します。

1. **［作成］** を選択します

1. **[ホスティング プラン]** ダイアログで、**[App Service]** と **[選択]** ボタンを選択します。

   > 💡 "運用環境では、実行ごとの支払いのスケーラビリティのために **Flex 従量課金**またはその他のプランを検討してください。"**

1. **[基本]** タブを次のように完成します。

   - **サブスクリプション:** アクティブな Azure サブスクリプション
   - **リソース グループ:** 既存のグループ (`$RG_NAME`)
   - **Function App 名:** `func-rental-search-<uniqueID>`
   - **コードまたはコンテナー イメージをデプロイする:** "**コード**"
   - **オペレーティング システム**: **Linux**
   - **ランタイム スタック:** **Python 3.11**
   - **リージョン:** PostgreSQL サーバーと同じ
   - **Linux プラン:** 既定値を受け入れるか、新規に作成する
   - **価格プラン:** 利用可能な最低レベル (Basic B1 や Standard S1 など)
   - **ゾーン冗長:** 無効

1. **[ストレージ]** タブで、ストレージ アカウントの **[新規作成]** を選択し、必要に応じて名前を生成します。

1. **[ネットワーク]**、**[監視]**、**[Durable Functions]**、**[デプロイ]** タブでは既定値をそのまま使用します。

1. **[認証]** タブで、認証の種類 **[ホスト ストレージ (AzureWebJobsStorage)]** を **[マネージド ID]** に変更します。 **[マネージド ID]** セクションが、新しいユーザー割り当て ID (`func-rental-search-<uniqueID>-uami` など) と共に下に表示されます。 既定値をそのまま使用します。必要な**ストレージ BLOB データ所有者**ロールが、自動的に割り当てられます。 Application Insights はそのままにしておきます。

1. **[タグ]** タブでは、既定値をそのまま使用します。

1. **[確認と作成] → [作成]** を選択し、デプロイを待ってから、新しい Function App を開きます。

1. **[リソースに移動]** を選択して Function App の概要ページを開きます。

### 3.2 関数アプリ変数の設定 (Cloud Shell)

1. Azure portal で **Cloud Shell (Bash)** に切り替えます。

1. Function App 変数とリソース グループ変数を設定します。
   ```bash
   FUNCAPP_NAME=<your-function-app-name>   # e.g., func-rental-search-abc123
   RG_NAME=<your-resource-group-name>      # e.g., rg-learn-postgresql-ai-westus3
   echo "Function App: $FUNCAPP_NAME"
   echo "Resource Group: $RG_NAME"
   ```

    > **注:** 関数アプリでは、関数アプリのセットアップ中に Azure で自動的に作成されたストレージ アカウントが使用されます。 追加のストレージの構成は必要ありません。

### 3.3 PostgreSQL 環境変数を追加する (Cloud Shell)

Function App で PostgreSQL 接続の値を構成します。

1. PostgreSQL サーバーの詳細を取得します。
   ```bash
   PGHOST=$(az postgres flexible-server list \
     --resource-group $RG_NAME \
     --query "[0].fullyQualifiedDomainName" \
     --output tsv)
   
   echo "PostgreSQL server: $PGHOST"
   echo "Admin password: $ADMIN_PASSWORD"
   ```

1. すべての環境変数を Function App に追加します。
   ```bash
   az functionapp config appsettings set \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --settings \
       "PGHOST=$PGHOST" \
       "PGDB=rentals" \
       "PGUSER=pgAdmin" \
       "PGPASSWORD=$ADMIN_PASSWORD" \
       "PGSSLMODE=require" 

   echo "Environment variables configured successfully."
   ```

1. **リモート ビルドを有効にして**、Azure が `requirements.txt` から Python の依存関係をインストールできるようにします。

   ```bash
   az functionapp config appsettings set \
     --name $FUNCAPP_NAME --resource-group $RG_NAME \
     --settings "SCM_DO_BUILD_DURING_DEPLOYMENT=true" "ENABLE_ORYX_BUILD=true"
   ```

1. Function App を再起動して設定を適用します。
   ```bash
   az functionapp restart \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME
   
   echo "Function App restarted."
   ```

これらのエントリは、Function のランタイム環境変数になります。 Azure Functions は、Python コード内の `os.getenv("<NAME>")` に自動的にマップされるため、`function_app.py` は実行時に PostgreSQL に安全に接続できます。

### 3.4 関数コードの確認

ラボ リポジトリには、Azure 関数を構成する 3 つの事前構築済みファイルと、デプロイ用の事前構築済みの ZIP ファイルが含まれています。 それらを確認して役割を理解し、必要に応じて変更できるようにする必要があります。

ファイルは `mslearn-postgresql/Allfiles/Labs/18/` にあります。

| ファイル | パーパス |
|------|---------|
| `requirements.txt` | Python の依存関係: Azure Functions SDK と `psycopg` (PostgreSQL ドライバー) |
| `host.json` | Azure Functions Runtime 構成: ログ設定と拡張機能バンドル |
| `function_app.py` | 検索 API の実装 (後述します) |
| `rental-search-func.zip` | 上記の 3 つのファイルの事前構築済み zip (デプロイできるもの) |

#### `function_app.py`: 検索 API

これがメイン ファイルです。 v2 プログラミング モデルを使用して、単一の HTTP によってトリガーされる関数を実装します。 これは次のように動作します。

- 環境変数 (`PGHOST`、`PGDB`、`PGUSER`、`PGPASSWORD`、`PGSSLMODE`) から **PostgreSQL 接続の詳細**を読み取ります。これらは、手順 3.3 で設定した変数です。
- `query` (テキスト) と`k` (結果の数) を含む JSON を受け入れる** `POST /api/search` エンドポイントを公開します**。
- PostgreSQL の `azure_openai.create_embeddings()` 関数を呼び出してクエリ テキストから埋め込みを生成することによって**ベクトル検索を実行**した後、pgvector の `<->` 演算子を使用して最も類似性の高い一覧を見つけます。
- Foundry エージェントで使用できるように書式設定された プロパティ ID、名前、説明、種類、価格を含む **JSON 結果を返します**。
- **関数レベルの認証 (`AuthLevel.FUNCTION`) が必要**なため、有効な関数キーを持つ呼び出し元のみがアクセスできます。

> **ヒント:** 検索の動作をカスタマイズする場合 (価格範囲やプロパティの種類でフィルター処理する場合など)、デプロイする前に `function_app.py` を変更して ZIP ファイルを作成し直します。

### 3.5 デプロイとテスト

1. Cloud Shell から**関数コードをデプロイ**します。

   ```bash
   # Get the correct SCM hostname (new Function Apps use a different URL format)
   HOST=$(az functionapp show --name $FUNCAPP_NAME --resource-group $RG_NAME --query defaultHostName -o tsv)
   SCM_HOST=$(echo $HOST | sed 's/\./.scm./')
   echo "SCM host: $SCM_HOST"

   # Deploy using a bearer token (required when Basic auth is disabled)
   cd ~/mslearn-postgresql/Allfiles/Labs/18
   zip deploy.zip function_app.py host.json requirements.txt
   TOKEN=$(az account get-access-token --query accessToken -o tsv)
   curl -s -X POST --data-binary "@deploy.zip" \
     -H "Authorization: Bearer $TOKEN" \
     "https://$SCM_HOST/api/zipdeploy"
   ```

   空の応答が表示されると、成功したことを意味します。

   > **注:** `401 Unauthorized` エラーが発生した場合、トークンの有効期限が切れている可能性があります。 `TOKEN=...` と `curl` コマンドをもう一度実行します。

   リモート ビルドによって Python の依存関係がインストールされるまで、1、2 分待ちます。

1. **関数アプリを再起動します**。
   ```bash
   az functionapp restart \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME
   
   echo "Function App restarted. Waiting for startup..."
   sleep 30
   ```

1. **Function App の URL と関数キーを取得します**。
   ```bash
   HOST=$(az functionapp show \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --query defaultHostName \
     --output tsv)
   
   # Get the default host key for authentication
   FUNC_KEY=$(az functionapp keys list \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --query "functionKeys.default" \
     --output tsv)
   
   # If default is null, get the master host key
   if [ -z "$FUNC_KEY" ] || [ "$FUNC_KEY" = "null" ]; then
     FUNC_KEY=$(az functionapp keys list \
       --name $FUNCAPP_NAME \
       --resource-group $RG_NAME \
       --query "masterKey" \
       --output tsv)
   fi
   
   echo ""
   echo "Deployment complete!"
   echo ""
   echo "Your Function Key (save this for the next task):"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo "$FUNC_KEY"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo ""
   echo "Search endpoint:"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo "https://$HOST/api/search"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo ""
   echo "For the next task, you'll need:"
   echo "  • Function App Host: $HOST"
   echo "  • Function Key: $FUNC_KEY"
   ```

1. **検索エンドポイントをテストします**。
   ```bash
   echo ""
   echo "Testing search endpoint..."
   curl -s -X POST "https://$HOST/api/search?code=$FUNC_KEY" \
     -H "Content-Type: application/json" \
     -d '{"query": "beachfront property with ocean view", "k": 3}' \
     | python3 -m json.tool
   ```

**予想される結果**:

> Search により、次のような賃貸一覧を含む JSON 応答が返されます。

   ```json
   {
   "results": [
      {
         "id": 41,
         "name": "Magazine Profiled with Gorgeous View",
         "description": "...",
         "property_type": "House",
         "room_type": "Entire home/apt",
         "price": 395.0,
         "weekly_price": null
      }
   ]
   }
   ```

**トラブルシューティング**:
> - 404 が表示された場合は、関数が完全に起動するまで 30 秒ほど待ってから、もう一度試します
> - 検索が失敗した場合は、ポータルの [Function App] → [構成] で PostgreSQL 環境変数を確認します
> - Function App のログを確認します (ポータル → [Function App] → [監視] → [ログ ストリーム])

---

## タスク 4 – Foundry プロジェクトとエージェントを作成し、API を登録する

次に、Microsoft Foundry でプロジェクトを作成し、エージェントを設定し、Function API をツールとして登録して、エージェントから呼び出せるようにします。

1. [Microsoft Foundry](https://ai.azure.com/) に移動してサインインします。

    > **注:** 右上隅に **[新しいFoundry]** トグルが表示される場合は、Foundry ポータルの最新バージョンを使用するために **[オン]** になっていることを確認してください。

1. 新しいプロジェクトを作成します。
   1. 左上隅にプロジェクト名が表示されている場合は、それを選択して **[新しいプロジェクトの作成]** を選択します。 プロジェクトが存在しない場合は、ホーム ページから **[+ プロジェクトの作成]** を選択します。 [プロジェクト] ダイアログボックスが表示されない場合は、**[エージェントの作成]** を選択すると新しいプロジェクトを作成するダイアログが表示されます。
   1. メッセージが表示されたら、**[Microsoft Foundry リソース]** を選択します。
   1. プロジェクトの名前を入力します (`rental-advisor-project` など)。
   1. **[作成]** を選択して完了するまで待ちます。

1. プロジェクトが開いたら、プロジェクトのホーム ページまたは **[ビルドの開始]** プルダウン メニューで **[エージェントの作成]** を選択します。

1. **[+ 新しいエージェント]** を選択し、次の構成を行います。
   - **エージェント名**: `RentalAdvisor`
   - **[作成]** を選択します

1. [モデルの選択] で、Foundry リソースに作成された **gpt-5.1** デプロイを選択します。 表示されない場合は、**[他のモデルの参照]** を選択し、リソース名でフィルター処理して、それを見つけて **[デプロイ]** します。

    > **重要:** すべてのモデル バージョンが OpenAPI ツールをサポートしているわけではありません。また、すべてのモデルがすべてのリージョンで使用できるわけではありません。 この演習では、**gpt-5.1** を使用します。これは、OpenAPI ツールをサポートしおり、米国西部 3 リージョンで使用できるためです。 別のリージョンをお使いの場合は、[ツールのリージョンおよびモデル別のサポート](https://learn.microsoft.com/en-us/azure/foundry/agents/concepts/tool-best-practice#tool-support-by-region-and-model)のドキュメントを参照して、互換性のあるモデルを選択してください。

1. **[手順]** フィールドに、次のコードを貼り付けます。

   ```
   You are an assistant for Margie's Travel helping customers find vacation rental properties.

   When users ask for property recommendations, use the postgresqlRentalSearch tool with their
   natural language query and a reasonable k value (3-5 results).

   Use the JSON results from the tool to craft a friendly, natural-language response that
   highlights the property names, descriptions, and prices. Be conversational and helpful.
   ```

1. **[ツール]** セクションまで下にスクロールします。 **[追加]** を選択し、**[すべてのツールを参照]** を選択します。

1. **[ツールの選択]** ダイアログで、**[カスタム]** タブを選択します。

1. **[OpenAPI ツール]** を選択し、**[作成]** を選択します。

1. ツールを構成します。

   - **名前**: `postgresqlRentalSearch`
   - **説明**: `Searches vacation rental properties using semantic search on PostgreSQL. Returns property listings matching natural language queries.`
   - **認証**: **[匿名]** を選択します
   - **[OpenAPI 仕様]** テキスト領域に、次の JSON を貼り付けます。 `<your-func-host>` を関数アプリのホスト名に置き換え、`<your-function-key>` を前のタスクの関数キーに置き換えます。

   ```json
   {
     "openapi": "3.0.0",
     "info": {
       "title": "PostgreSQL Rental Search API",
       "version": "1.0.0",
       "description": "Semantic search API for vacation rental properties using PostgreSQL vector search"
     },
     "servers": [
       {
         "url": "https://<your-func-host>"
       }
     ],
     "paths": {
       "/api/search": {
         "post": {
           "summary": "Search rental properties",
           "description": "Performs semantic search on rental property listings using natural language queries",
           "operationId": "searchRentals",
           "parameters": [
             {
               "name": "code",
               "in": "query",
               "required": true,
               "schema": {
                 "type": "string",
                 "default": "<your-function-key>"
               }
             }
           ],
           "requestBody": {
             "required": true,
             "content": {
               "application/json": {
                 "schema": {
                   "type": "object",
                   "required": ["query"],
                   "properties": {
                     "query": {
                       "type": "string",
                       "description": "Natural language search query (e.g., 'beachfront property with ocean view')"
                     },
                     "k": {
                       "type": "integer",
                       "description": "Number of results to return (1-10)",
                       "default": 3,
                       "minimum": 1,
                       "maximum": 10
                     }
                   }
                 }
               }
             }
           },
           "responses": {
             "200": {
               "description": "Successful search",
               "content": {
                 "application/json": {
                   "schema": {
                     "type": "object",
                     "properties": {
                       "results": {
                         "type": "array",
                         "items": {
                           "type": "object",
                           "properties": {
                             "id": {"type": "integer"},
                             "name": {"type": "string"},
                             "description": {"type": "string"},
                             "property_type": {"type": "string"},
                             "room_type": {"type": "string"},
                             "price": {"type": "number"},
                             "weekly_price": {"type": "number", "nullable": true}
                           }
                         }
                       }
                     }
                   }
                 }
               }
             }
           }
         }
       }
     }
   }
   ```

1. **[ツールの作成]** ボタンを選択して、エージェントにツールを追加します。

1. **[保存]** を選んで、エージェント構成を保存します。

---
## タスク 5 – エージェントをテストする

エージェントの動作を確認してみましょう。

チャット画面で、次のようなメッセージを入力します。

```
Find beachside apartments with great reviews.
```

```
Recommend a quiet cabin for families.
```

```
Show modern apartments near downtown.
```

   > **注:** "要求が多すぎます" というエラーが発生する場合は、エージェント構成でビジー状態の少ない別のモデルを選択してエージェントを保存するか、数分待ってからやり直してみてください。

思いつく他のバリエーションも試してみてください。

エージェントは Function を呼び出します。Function により、PostgreSQL が埋め込まれ、クエリが実行され、結果が要約されます。



## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/azure-portal-home-azure-services-resource-groups.png)

2. 任意のフィールドのフィルター検索ボックスに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/azure-portal-overview-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。

## 要点

この演習では、PostgreSQL と Microsoft Foundry を使用して AI エージェントを構築するための基本的なパターンを示しています。 作成した **RentalAdvisor** エージェントは 1 つの例にすぎません。 同じアーキテクチャにより、連携する複数の特殊なエージェントがサポートされます。 たとえば、予約、レビュー、価格設定などを行うより多くのエージェントを構築できます。

**このプロジェクトに追加できるエージェント:**

- **BookingAgent** – PostgreSQL トランザクション テーブルを使用して、予約の処理、空き状況の確認、予約確認の管理を行います
- **ReviewAnalyzer** – レビュー テーブルからのセンチメントの分析、ゲストのフィードバックの要約、物件の長所と短所の特定を行います
- **PriceOptimizer** – 季節の傾向、需要パターン、過去の予約データに基づく動的な価格を推奨します
- **MaintenanceScheduler** – 物件の保守依頼の追跡、修理のスケジュール、物件管理人への注意喚起を行います
- **CustomerSupportAgent** – FAQ に対する回答、ゲストからの問い合わせの処理、人間のスタッフへの複雑な問題のエスカレーションを行います

各エージェントは同じパターンを使用します。つまり、Azure Function は PostgreSQL に接続され、Microsoft Foundry にカスタム ツールとして登録されます。 エージェントは個別に作業することも、共同で作業することもできます。たとえば、**RentalAdvisor** で物件を検索し、**BookingAgent** に引き渡して予約を完了します。

**アーキテクチャの強み:**

- **PostgreSQL の AI 機能**により、埋め込みとベクトル検索がネイティブに処理されるため、個別のベクトル データベースが不要になります
- **Microsoft Foundry** は、複数エージェントの会話を調整し、コンテキストを管理し、複雑な推論を処理します
- **Azure Functions は、データを AI エージェントに接続する軽量でスケーラブルな API エンドポイントを提供します
- **設計によるセキュリティ保護** – マネージド ID、関数キー、Azure のセキュリティ機能によってデータが保護されます

このモジュール式のアプローチは、単純な単一エージェント シナリオから、データ全体の複雑なビジネス プロセスを処理する高度なマルチエージェント システムまで拡張できます。