---
lab:
  title: Foundry Agent Service と PostgreSQL を使用して AI エージェントを構築してテストする
  module: Implement generative AI agents with Azure Database for PostgreSQL
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
   - `rentals` という名前の **Azure Database for PostgreSQL フレキシブル サーバー**
   - Function App 操作用の **Azure Storage アカウント**
   - text-embedding-ada-002 モデルがデプロイされた **Azure OpenAI Service**
   - エージェントを構築するための **Microsoft Foundry** プロジェクト

> PostgreSQL サーバーの **FQDN**、ユーザー名 (`pgAdmin`)、パスワードをメモしておいてください。これらは次の手順で使用します。

## Azure Cloud Shell で psql を使用してデータベースに接続する

[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL サーバーに移動します。

1. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。 **[接続]** を選択しても、実際にはデータベースに接続されません。これは、さまざまな方法を使用してデータベースに接続する手順を示しているだけです。 **ブラウザーからまたはローカルで接続する**手順を確認し、それらの手順で、Azure Cloud Shell を使用して接続します。

   ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](./media/17-postgresql-rentals-database-connect.png)

1. まだ開いていない場合は、Azure portal で **Cloud Shell (Bash)** を開きます。

1. Cloud Shell で、**ブラウザーからまたはローカルに接続する**手順で説明されている `psql` コマンドを実行します。 次のコマンドのようになります (`<your-postgresql-server-name>` を実際のサーバー名に置き換えます)。
   ```bash
   psql -h <your-postgresql-server-name>.postgres.database.azure.com -p 5432 -U pgAdmin rentals
   ```

1. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** サインイン用にランダムに生成されたパスワードを入力します。

   サインインすると、`rentals` データベースの `psql` プロンプトが表示されます。

1. この演習の残りの部分では、Cloud Shell で作業を続けます。そのため、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー ウィンドウ内でペインを拡大すると作業しやすくなります。

## タスク 1 – 拡張機能を有効にし、Azure AI 設定を構成する

```sql
CREATE EXTENSION IF NOT EXISTS vector;
CREATE EXTENSION IF NOT EXISTS azure_ai;

SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<your-openai-account>.openai.azure.com');
SELECT azure_ai.set_setting('azure_openai.subscription_key', '<your-api-key>');
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
  FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' WITH (FORMAT csv, HEADER);

\COPY reviews (id, listing_id, date, comments)
  FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' WITH (FORMAT csv, HEADER);
```

PostgreSQL 内で埋め込みを生成します。

```sql
UPDATE listings
SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
WHERE listing_vector IS NULL;
```

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

1. **[確認と作成] → [作成]** を選択し、デプロイを待ってから、新しい Function App を開きます。

1. **[リソースに移動]** を選択して Function App の概要ページを開きます。

### 3.2 マネージド ID とストレージを構成する (Cloud Shell)

次に、最初のデプロイ時に作成されたストレージ アカウントへの安全なアクセスにマネージド ID を使用するように Function App を構成します。

1. Azure portal で **Cloud Shell (Bash)** に切り替えます。

1. Function App 変数とリソース グループ変数を設定します。
   ```bash
   FUNCAPP_NAME=<your-function-app-name>   # e.g., func-rental-search-abc123
   RG_NAME=<your-resource-group-name>      # e.g., rg-learn-postgresql-ai-westus3
   echo "Function App: $FUNCAPP_NAME"
   echo "Resource Group: $RG_NAME"
   ```

1. Function App でシステム割り当てマネージド ID を有効にします。
   ```bash
   az functionapp identity assign \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME
   
   PRINCIPAL_ID=$(az functionapp identity show \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --query principalId \
     --output tsv)
   
   echo "Managed identity enabled with Principal ID: $PRINCIPAL_ID"
   ```

1. 最初のデプロイ時に作成された既存のストレージ アカウントを見つけます。

   Azure Functions では、メタデータ、ログ、調整などの内部操作を行うためにストレージ アカウントが必要になります。 Bicep テンプレートは既に作成されています。 次に、接続文字列なしの安全なアクセスのためにマネージド ID を使用して Function App にアクセスするように Function App を構成します。

   ```bash
   # Find the storage account in your resource group
   STORAGE_NAME=$(az storage account list \
     --resource-group $RG_NAME \
     --query "[0].name" \
     --output tsv)
   
   echo "Found storage account: $STORAGE_NAME"
   
   # Get the storage account resource ID
   STORAGE_ID=$(az storage account show \
     --name $STORAGE_NAME \
     --resource-group $RG_NAME \
     --query id \
     --output tsv)
   
   # Grant Storage Blob Data Contributor role to the Function App identity
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Blob Data Contributor" \
     --scope $STORAGE_ID
   
   # Grant Storage Queue Data Contributor role to the Function App identity
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Queue Data Contributor" \
     --scope $STORAGE_ID
   
   # Grant Storage Table Data Contributor role to the Function App identity
   az role assignment create \
     --assignee $PRINCIPAL_ID \
     --role "Storage Table Data Contributor" \
     --scope $STORAGE_ID
   
   echo "Managed identity granted Contributor roles for Blob, Queue, and Table storage."
   ```
   
1. 既存の AzureWebJobsStorage 接続文字列設定を削除します。

   Function App は通常、共有キーを使用してストレージ アカウントを指す既定の接続文字列で作成されます。 マネージド ID を使用する場合は、最初にこの古い設定を削除する必要があります。

   ```bash
   # Remove old connection string setting if it exists
   az functionapp config appsettings delete \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --setting-names "AzureWebJobsStorage"
   
   echo "Removed old AzureWebJobsStorage connection string (if it existed)."
   ```

1. ストレージにマネージド ID を使用するように Function App を更新します。

   ```bash
   # Get the blob endpoint
   BLOB_ENDPOINT=$(az storage account show \
     --name $STORAGE_NAME \
     --resource-group $RG_NAME \
     --query primaryEndpoints.blob \
     --output tsv)
   
   # Configure Function App to use managed identity for storage
   az functionapp config appsettings set \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --settings \
       "AzureWebJobsStorage__accountName=$STORAGE_NAME" \
       "AzureWebJobsStorage__blobServiceUri=${BLOB_ENDPOINT}" \
       "AzureWebJobsStorage__queueServiceUri=$(echo $BLOB_ENDPOINT | sed 's/blob/queue/')" \
       "AzureWebJobsStorage__tableServiceUri=$(echo $BLOB_ENDPOINT | sed 's/blob/table/')" \
       "AzureWebJobsStorage__credential=managedidentity"
   
   echo "Storage configured to use managed identity."
   ```

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

1. Function App を再起動して設定を適用します。
   ```bash
   az functionapp restart \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME
   
   echo "Function App restarted."
   ```

これらのエントリは、Function のランタイム環境変数になります。 Azure Functions は、Python コード内の `os.getenv("<NAME>")` に自動的にマップされるため、`function_app.py` は実行時に PostgreSQL に安全に接続できます。

### 3.4 Python コードを作成する (Cloud Shell)

次に、Function の Python コード ファイルを作成します。

Cloud Shell で、`code` を使用するときに**クラシック モードに切り替える**ように要求された場合は、それを受け入れます。 シェルが再読み込みされた場合は、手順 3.2 の変数コマンドを再実行し、ここからもう一度開始します。

1. 作業フォルダーを設定します。
   ```bash
   mkdir -p $HOME/rental-search-func
   cd $HOME/rental-search-func
   ```

1. `requirements.txt` を作成します。
   ```bash
   code requirements.txt
   ```
   貼り付けて保存します。
   ```text
   azure-functions>=1.20.0,<2.0.0
   psycopg[binary]>=3.2.1,<4.0.0
   ```

1. `function_app.py` を作成します。

   この Python ファイルは、v2 プログラミング モデルを使用して Azure Function を実装します。 "Microsoft Foundry エージェントは、ユーザーの問い合わせに対応するときに、この API を呼び出して賃貸物件データを取得します。"** コード:
   
      - 安全な認証のために環境変数を使用して **PostgreSQL に接続する**
      - クエリ テキストと結果件数を含む POST 要求を受け入れる**検索エンドポイント (`/search`) を定義する**
      - **入力を検証**してクエリの安全性を確保する
      - PostgreSQL の `azure_openai.create_embeddings()` 関数を呼び出してオンデマンドで埋め込みを生成し、pgvector の `<->` 演算子を使用して最も類似性の高い一覧を見つけて、**ベクトル検索を実行する**
      - AI エージェントが使用できるように書式設定された **JSON 結果を返す**
      - データを保護するために**関数レベルの認証を要求する**

   ```bash
   code function_app.py
   ```

   貼り付けて保存します。
   
   ```python
   import os
   import json
   import logging
   import psycopg
   import azure.functions as func
   from datetime import datetime

   # Configure logging
   logging.basicConfig(level=logging.INFO)
   logger = logging.getLogger(__name__)

   app = func.FunctionApp(http_auth_level=func.AuthLevel.FUNCTION)

   # Database connection parameters from environment variables
   PGHOST = os.getenv("PGHOST")
   PGDB = os.getenv("PGDB", "rentals")
   PGUSER = os.getenv("PGUSER")
   PGPASSWORD = os.getenv("PGPASSWORD")
   PGSSLMODE = os.getenv("PGSSLMODE", "require")

   def log_env_vars():
      """Log environment variables for debugging (masked)"""
      logger.info("=== Environment Variables ===")
      logger.info(f"PGHOST: {PGHOST}")
      logger.info(f"PGDB: {PGDB}")
      logger.info(f"PGUSER: {PGUSER}")
      logger.info(f"PGPASSWORD: {'***' if PGPASSWORD else 'NOT SET'}")
      logger.info(f"PGSSLMODE: {PGSSLMODE}")
      logger.info("============================")

   def get_db_conn():
      logger.info("Creating database connection...")
      try:
         conn = psycopg.connect(
               host=PGHOST,
               dbname=PGDB,
               user=PGUSER,
               password=PGPASSWORD,
               sslmode=PGSSLMODE,
               autocommit=True,
               connect_timeout=10
         )
         logger.info("Database connection successful!")
         return conn
      except Exception as e:
         logger.error(f"Database connection failed: {str(e)}")
         raise

   @app.route(route="search", methods=["POST"])
   def search(req: func.HttpRequest) -> func.HttpResponse:
      timestamp = datetime.utcnow().isoformat()
      logger.info(f"========== NEW REQUEST {timestamp} ==========")
      
      try:
         # Log environment on first request
         log_env_vars()
         
         # Parse request body
         logger.info("Parsing request body...")
         try:
               req_body = req.get_json()
               logger.info(f"Request body parsed: {req_body}")
         except Exception as e:
               logger.error(f"Failed to parse JSON: {str(e)}")
               return func.HttpResponse(
                  json.dumps({"error": "Invalid JSON in request body", "details": str(e)}),
                  mimetype="application/json",
                  status_code=400
               )
         
         query = req_body.get('query')
         k = req_body.get('k', 3)
         
         logger.info(f"Query: '{query}', k: {k}")
         
         if not query:
               logger.warning("Query parameter missing")
               return func.HttpResponse(
                  json.dumps({"error": "Missing 'query' parameter"}),
                  mimetype="application/json",
                  status_code=400
               )
         
         # Validate k
         if not isinstance(k, int) or k < 1 or k > 10:
               logger.warning(f"Invalid k value: {k}, using default 3")
               k = 3
         
         # Connect to database
         logger.info("Connecting to PostgreSQL...")
         with get_db_conn() as conn:
               with conn.cursor() as cur:
                  # Generate embedding and perform vector search in one query
                  logger.info("Performing semantic search...")
                  search_query = """
                     WITH query_embedding AS (
                        SELECT azure_openai.create_embeddings('embedding', %s, max_attempts => 5, retry_delay_ms => 500) AS emb
                     )
                     SELECT l.id, l.name, l.description, l.property_type, l.room_type, l.price, l.weekly_price
                     FROM listings l, query_embedding qe
                     WHERE l.listing_vector IS NOT NULL
                     ORDER BY l.listing_vector <-> qe.emb
                     LIMIT %s;
                  """
                  
                  try:
                     cur.execute(search_query, (query, k))
                     rows = cur.fetchall()
                     logger.info(f"Vector search returned {len(rows)} results")
                  except Exception as e:
                     logger.error(f"Vector search failed: {str(e)}")
                     raise
                  
                  # Format results
                  logger.info("Formatting results...")
                  results = []
                  for idx, row in enumerate(rows):
                     logger.info(f"Processing row {idx + 1}: id={row[0]}, name={row[1][:30]}...")
                     results.append({
                           "id": row[0],
                           "name": row[1],
                           "description": row[2],
                           "property_type": row[3],
                           "room_type": row[4],
                           "price": float(row[5]) if row[5] is not None else None,
                           "weekly_price": float(row[6]) if row[6] is not None else None
                     })
                  
                  logger.info(f"Successfully formatted {len(results)} results")
                  logger.info("========== REQUEST COMPLETED SUCCESSFULLY ==========")
                  
                  return func.HttpResponse(
                     json.dumps({"results": results}),
                     mimetype="application/json",
                     status_code=200
                  )
                  
      except ValueError as e:
         logger.error(f"ValueError: {str(e)}", exc_info=True)
         return func.HttpResponse(
               json.dumps({"error": "Invalid JSON in request body", "details": str(e)}),
               mimetype="application/json",
               status_code=400
         )
      except Exception as e:
         logger.error(f"FATAL ERROR: {str(e)}", exc_info=True)
         return func.HttpResponse(
               json.dumps({"error": str(e), "type": type(e).__name__}),
               mimetype="application/json",
               status_code=500
         )
   ```

1. `host.json` を作成します。
   ```bash
   code host.json
   ```
   貼り付けて保存します。
   ```json
   {
      "version": "2.0",
      "logging": {
         "applicationInsights": {
            "samplingSettings": {
            "isEnabled": true,
            "excludedTypes": "Request"
            }
         },
         "logLevel": {
            "default": "Information",
            "Function": "Information"
         }
      },
      "extensionBundle": {
         "id": "Microsoft.Azure.Functions.ExtensionBundle",
         "version": "[4.*, 5.0.0)"
      }
   }
   ```

1. ファイルが作成されたことを確認します。
   ```bash
   ls -la
   echo "---"
   echo "Files created successfully:"
   echo "- requirements.txt"
   echo "- function_app.py"
   echo "- host.json"
   ```

### 3.5 デプロイしてテストする (Cloud Shell)

1. **変数が設定されていることを確認します** (Cloud Shell が再読み込みされた場合は、次のコマンドを再実行します)。

   ```bash
   # If variables are not set, run:
   FUNCAPP_NAME=<your-function-app-name>
   RG_NAME=<your-resource-group-name>
   echo "Function App: $FUNCAPP_NAME"
   echo "Resource Group: $RG_NAME"
   ```

1. **zip を使用して関数をデプロイします** (この手順の実行には数分かかります)。

   ```bash
   cd $HOME/rental-search-func
   
   # Clean up any previous deployment
   rm -f app.zip
   
   # Create zip file (exclude hidden files and git directories)
   zip -r app.zip . -x ".*" "*.git*" "__pycache__/*" "*.pyc"
   
   # Deploy to Azure
   az functionapp deployment source config-zip \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME \
     --src app.zip \
     --build-remote true

   echo "Deployment initiated. Waiting for completion..."
   sleep 30
   ```

1. **すべての変更を確実に有効にするには、Function App を再起動します**。
   ```bash
   az functionapp restart \
     --name $FUNCAPP_NAME \
     --resource-group $RG_NAME
   
   echo "Function App restarted. Waiting for startup..."
   sleep 20
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
   echo "Your Function Key (save this for Task 4):"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo "$FUNC_KEY"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo ""
   echo "Search endpoint:"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo "https://$HOST/api/search"
   echo "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
   echo ""
   echo "For Task 4, you'll need:"
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
> ```json
> {
>   "results": [
>     {
>       "id": 41,
>       "name": "Magazine Profiled with Gorgeous View",
>       "description": "...",
>       "property_type": "House",
>       "room_type": "Entire home/apt",
>       "price": 395.0,
>       "weekly_price": null
>     }
>   ]
> }
> ```

**トラブルシューティング**:
> - 404 が表示された場合は、関数が完全に起動するまで 30 秒ほど待ってから、もう一度試します
> - 検索が失敗した場合は、ポータルの [Function App] → [構成] で PostgreSQL 環境変数を確認します
> - Function App のログを確認します (ポータル → [Function App] → [監視] → [ログ ストリーム])

---

## タスク 4 – API を Microsoft Foundry にカスタム ツールとして登録する

次に、エージェントが Function API を呼び出すことができるように、Function API を Microsoft Foundry に登録します。

> **注**:この演習では、OpenAPI 仕様で HTTP によってトリガーされる Azure Function を使用します。これにより、エージェントは、関数をカスタム ツールとして呼び出すことができます。 Microsoft Foundry では、キューによってトリガーされる関数のキュー ベースのネイティブ統合もサポートされていますが、OpenAPI を使用した HTTP アプローチを使用すると、この学習シナリオに適したデプロイと REST API への直接アクセスが簡素化されます。

1. [AI Foundry (プレビュー)](https://ai.azure.com/) に移動します。  

1. プロジェクトに移動し、左側のメニューから **[エージェント]** を選択します。

1. **[+ 新しいエージェント]** を選択し、次の構成を行います。
   - **エージェント名**: `RentalAdvisor`
   - **デプロイ**: 利用可能な最新の GPT-4 デプロイを選択します
   - **[作成]** を選択します

1. エージェントの [セットアップ] ページで、**[アクション (0)]** セクションまで下にスクロールし、**[追加]** を選択します。

1. **カスタム ツールの作成**ウィザードで、次の操作を行います。

   **[手順 1 - ツールの詳細]**:
   - **名前**: `postgresqlRentalSearch`
   - **説明**: `Searches vacation rental properties using semantic search on PostgreSQL. Returns property listings matching natural language queries.`
   - **[次へ]** を選択します

1. **[手順 2 - スキーマの定義]** で、次の操作を行います。

   - **認証方法**: ドロップダウンから **[匿名]** を選択します。
   - **[OpenAPI 仕様]** テキスト領域に次の仕様を貼り付け、`<your-func-host>` をお使いの Function App のホスト名に置き換え、`<your-function-key>` を手順 3.5 の関数キーに置き換えます。
   
   > **注**:この方法により、関数キーがクエリ パラメーターとして OpenAPI 仕様に埋め込まれます。 仕様は Microsoft Foundry によって安全に保存され、キーはエンド ユーザーに公開されません。
   
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
         "url": "https://<your-func-host>/api/search"
       }
     ],
     "paths": {
       "/": {
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

1. **[次へ]** を選択し、**[ツールの作成]** を選択します。

1. エージェントへの指示を更新します。

   ```
   You are an assistant for Margie's Travel helping customers find vacation rental properties.
   
   When users ask for property recommendations, use the postgresqlRentalSearch tool with their 
   natural language query and a reasonable k value (3-5 results). 
   
   Use the JSON results from the tool to craft a friendly, natural-language response that 
   highlights the property names, descriptions, and prices. Be conversational and helpful.
   ```

これで、エージェントは PostgreSQL でサポートされる賃貸検索ツールを使用する準備ができました。

---

## タスク 5 – エージェントをテストする

エージェントの動作を確認してみましょう。

**[プレイグラウンド]** を選択します。

チャット画面で、次のようなクエリを入力します。

```
Find beachside apartments with great reviews.
```

```
Recommend a quiet cabin for families.
```

```
Show modern apartments near downtown.
```

思いつく他のバリエーションも試してみてください。

エージェントは Function を呼び出します。Function により、PostgreSQL が埋め込まれ、クエリが実行され、結果が要約されます。


## タスク 6 – クリーンアップする

```azurecli
az group delete --name $RG_NAME --yes --no-wait
```

---

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