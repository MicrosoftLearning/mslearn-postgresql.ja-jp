---
lab:
  title: Azure Database for PostgreSQL と Python を使用して RAG アプリケーションを構築する
  module: Build RAG applications with Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL と Python を使用して RAG アプリケーションを構築する

このシナリオでは、Contoso の会社のポリシーの質問に対する小さな内部アシスタントを構築します。 Azure Database for PostgreSQL でテーブルを設定し、ポリシーの CSV を読み込み、各ポリシーの埋め込みを格納して、データベースがキーワードだけでなく、意味で質問と一致するようにします。 検索を高速に保つために、ベクトル インデックスを追加します。 次に、質問を行い、特に関連性の高いポリシーをフェッチし、それらのポリシーのみに基づいた回答をポリシー タイトルを含めて出力する短い Python スクリプトを作成します。

この演習を終了すると、次のことができるようになります。

- 埋め込みとベクトル検索を強化するデータベース拡張機能を有効にする。
- データのデータベース内埋め込みを生成する。
- ベクトル インデックスを追加して、検索を高速に保つ。
- 上位のチャンクを取得し、根拠のある回答を生成する小さな RAG Python プログラムを作成する。

## 開始する前に

管理者権限を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要であり、そのサブスクリプションで Azure OpenAI アクセスが承認されている必要があります。 Azure OpenAI アクセスが必要な場合、[Azure OpenAI の制限付きアクセス](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access)に関するページで申請してください。

### Azure サブスクリプションにリソースをデプロイする

"非運用環境の Azure Database for PostgreSQL サーバーと非運用環境の Azure OpenAI リソースを既にセットアップしている場合は、このセクションをスキップできます。"**

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

1. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](./media/15-portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用していた場合は、*Bash* シェルに切り替えます。

1. Cloud Shell プロンプトで、次のように入力して、演習リソースを含んだ GitHub リポジトリをクローンします。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. 次に、3 つのコマンドを実行して、Azure CLI コマンドを使用して Azure リソースを作成するときに冗長な入力を減らすための変数を定義します。 これらの変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースのデプロイ先となる Azure リージョン (`REGION`)、PostgreSQL 管理者のサインイン用にランダムに生成されるパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドでは、対応する変数に割り当てられるリージョンは `westus3` ですが、任意の場所に置き換えることもできます。 ただし、既定値を置き換える場合は、このラーニング パスのモジュール内のすべてのタスクを完了できるように、[抽象要約をサポートしている別の Azure リージョン](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)を選択する必要があります。

    ```bash
    REGION=westus3
    ```

    次のコマンドでは、この演習で使用されるすべてのリソースを格納するリソース グループに使用する名前が割り当てられます。 対応する変数に割り当てられたリソース グループ名は `rg-learn-postgresql-ai-$REGION` で、その中の `$REGION` は以前に指定した場所です。 ただし、この部分は好みに合わせて他のリソース グループ名に変更できます。

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    最後のコマンドでは、PostgreSQL 管理者のサインイン用のパスワードがランダムに生成されます。 後で PostgreSQL フレキシブル サーバーに接続する際に使用できるように、これを安全な場所に**必ずコピーしておいてください**。

    ```bash
    a=()
    for i in {a..z} {A..Z} {0..9}; 
     do
     a[$RANDOM]=$i
    done
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]::18}")
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo $ADMIN_PASSWORD
    ```

1. "現在のサブスクリプションを変更する場合にのみ、次のコマンドを実行します。"** 複数の Azure サブスクリプションにアクセスでき、この演習のリソース グループやその他のリソースを作成するサブスクリプションが既定のサブスクリプションではない場合、`<subscriptionName|subscriptionId>` トークンを、使用するサブスクリプションの名前または ID に置き換えて次のコマンドを実行し、適切なサブスクリプションを設定します。

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

1. 次の Azure CLI コマンドを実行して、リソース グループを作成します。

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. 最後に、Azure CLI を使用して Bicep デプロイ スクリプトを実行し、リソース グループ内の Azure リソースをプロビジョニングします。

    ```azurecli
    #1 Core infra: PostgreSQL + DB + firewall + server param, AOAI account, Language account
    az deployment group create \
      --resource-group "$RG_NAME" \
      --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.core.bicep" \
      --parameters restore=false adminLogin=pgAdmin adminLoginPassword="$ADMIN_PASSWORD" databaseName=ContosoHelpDesk
    
    AOAI=$(az cognitiveservices account list -g "$RG_NAME" --query "[?kind=='OpenAI'].name | [0]" -o tsv)
    
    #2 Wait for the parent AOAI account to finish provisioning
    echo "Waiting for AOAI account to be ready..."
    while true; do
      STATE=$(az cognitiveservices account show -g "$RG_NAME" -n "$AOAI" --query "properties.provisioningState" -o tsv)
      echo "provisioningState=$STATE"
      [ "$STATE" = "Succeeded" ] && break
      sleep 10
    done

    #3 OpenAI deployments: embedding + chat
    az deployment group create \
      --resource-group "$RG_NAME" \
      --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.aoai-deployments.bicep" \
      --parameters azureOpenAIServiceName="$AOAI"
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースには、Azure Database for PostgreSQL サーバー、Azure OpenAI、Azure AI Language サービスが含まれます。 また、Bicep スクリプトでは、(`azure.extensions` サーバー パラメーターを使用して) PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する、サーバーに `ContosoHelpDesk` という名前のデータベースを作成する、`text-embedding-ada-002` モデルを使用して `embedding` という名前のデプロイを Azure OpenAI Service に追加するといった、いくつかの構成手順も実行されます。 最終的に、`gpt-4o-mini` モデルを使用して `chat` という名前のデプロイが Azure OpenAI サービスに追加されます。 Bicep ファイルは、このラーニング パス内のすべてのモジュールで共有されるため、一部の演習では、デプロイされたリソースの一部のみを使用できます。

    デプロイが完了するまでに通常は数分かかります。 デプロイの進行状況は、Cloud Shell から監視するか、または先ほど作成したリソース グループの **[デプロイ]** ページに移動し、そこで確認することができます。

1. 後で必要になるため、リソース名とそれに対応する ID、および PostgreSQL サーバーの完全修飾ドメイン名 (FQDN)、ユーザー名、パスワードをメモしておきます。

### デプロイ エラーのトラブルシューティング

Bicep デプロイ スクリプトを実行すると、いくつかエラーが発生する場合があります。 "エラーが発生しない場合は、このセクションをスキップします。"**

- 以前にこのラーニング パスで Bicep デプロイ スクリプトを実行し、その後でリソースを削除した場合、リソースを削除してから 48 時間以内にスクリプトを再実行しようとすると、次のようなエラー メッセージが表示される場合があります。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    このメッセージが表示された場合は、`restore` パラメーターが `true` に設定されるように先ほどの `azure deployment group create` コマンドを変更し、再実行します。

- 選択したリージョンで特定のリソースのプロビジョニングが制限されている場合は、`REGION` 変数を別の場所に設定してコマンドを再実行し、リソース グループを作成して、Bicep デプロイ スクリプトを実行する必要があります。

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 責任ある AI 契約に同意する必要があるためにスクリプトで AI リソースを作成できない場合は、次のエラーが発生します。 このエラーが発生した場合は、Azure portal ユーザー インターフェイスを使用して Azure AI Services リソースを作成し、デプロイ スクリプトを再実行します。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Azure Cloud Shell で psql を使用してデータベースに接続する

[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL サーバー上の `ContosoHelpDesk` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL サーバーに移動します。

1. リソース メニューの **[設定]** で、**[データベース]** を選択し、`ContosoHelpDesk` データベースの **[接続]** を選択します。 **[接続]** を選択しても、実際にはデータベースに接続されません。これは、さまざまな方法を使用してデータベースに接続する手順を示しているだけです。 **ブラウザーからまたはローカルで接続する**手順を確認し、それらの手順で、Azure Cloud Shell を使用して接続します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 ContosoHelpDesk データベースの [データベース] と [接続] は赤いボックスで強調表示されています。](./media/15-postgresql-database-connect.png)

1. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** サインイン用にランダムに生成されたパスワードを入力します。

    サインインすると、`ContosoHelpDesk` データベースの `psql` プロンプトが表示されます。

1. この演習の残りの部分では、Cloud Shell で作業を続けます。そのため、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー ウィンドウ内でペインを拡大すると作業しやすくなります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](./media/15-azure-cloud-shell-db-pane-maximize.png)

## セットアップ: 拡張機能を構成する

ベクトルを格納してそのクエリを実行し、埋め込みを生成するには、Azure Database for PostgreSQL フレキシブル サーバーの 2 つの拡張機能 (`vector` と `azure_ai`) を許可リストに追加して有効にする必要があります。

1. 両方の拡張機能を許可リストに追加するには、[PostgreSQL 拡張機能の使用方法](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)に関する記事で説明されている手順に従って、サーバー パラメーター `azure.extensions` に `vector` と `azure_ai` を追加します。

1. 次の SQL コマンドを実行して、`vector` および `azure_ai` 拡張機能を有効にします。 詳細な手順については、[Azure Database for PostgreSQL で `pgvector` を有効にして使用する方法](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension)に関する記事を参照してください。

    *ContosoHelpDesk* プロンプトで、次の SQL コマンドを実行します。

    ```sql
    -- Enable required extensions
    CREATE EXTENSION vector;
    CREATE EXTENSION azure_ai;
    ```

1. `azure_ai` 拡張機能を有効にするには、次の SQL コマンドを実行します。 Azure OpenAI リソースのエンドポイントと API キーが必要です。 詳細な手順については、「[`azure_ai` 拡張機能を有効にする](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension)」をご覧ください。

    *ContosoHelpDesk* プロンプトで、次のコマンドを実行します。

    ```sql
    -- Configure Azure OpenAI (requires azure_ai_settings_manager role)
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');      -- e.g., https://YOUR-RESOURCE.openai.azure.com
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

## データベースにサンプル データを入力する

`azure_ai` 拡張機能を使用する前に、`ContosoHelpDesk` データベースにテーブルを追加し、サンプル データを事前設定して、アプリケーションの作成時に使用する情報を用意しておきます。

1. **ContosoHelpDesk** プロンプトで、次のコマンドを実行して、会社のポリシー データを格納するための `company_policies` テーブルを作成します。

    ```sql
    -- Create table for policies and embeddings (matches CSV columns)
    DROP TABLE IF EXISTS company_policies CASCADE;

    CREATE TABLE company_policies (
      policy_id          BIGSERIAL PRIMARY KEY,
      title       TEXT NOT NULL,
      department  TEXT NOT NULL,
      policy_text TEXT NOT NULL,
      category    TEXT NOT NULL,
      embedding   vector(1536)  -- The `text-embedding-ada-002` model is configured to return 1,536 dimensions, so use that number for the vector column size.
    );
    ```

1. Azure Cloud Shell で、`COPY` コマンドを使用して、先ほど作成したテーブルに CSV ファイルからデータを読み込みます。 次のコマンドを実行して、`company_policies` テーブルを事前設定します。

    ```sql
    \COPY company_policies (title, department, policy_text, category) FROM 'mslearn-postgresql/Allfiles/Labs/Shared/company-policies.csv' WITH (FORMAT csv, HEADER)
    ```

    コマンドの出力は `COPY 108` となるはずです。これは、CSV ファイルから 108 行がテーブルに書き込まれたことを示しています。

1. 既存の行の埋め込みをバックフィルします。

    **psql** セッション (Cloud Shell) で次のコマンドを実行して、まだ含まれていない行の埋め込みを計算します。 `<EMBEDDING_DEPLOYMENT_NAME>` を埋め込みデプロイの名前に置き換えます。

    ```sql
    -- Create embeddings for existing rows that currently have no embeddings
    UPDATE company_policies
    SET embedding = azure_openai.create_embeddings('<EMBEDDING_DEPLOYMENT_NAME>', policy_text)::vector
    WHERE embedding IS NULL;
    ```

    これにより、(`azure_ai` 経由で) SQL から Azure OpenAI の埋め込みデプロイが呼び出され、その結果が列 `embedding` に格納されます。

埋め込みで 108 行が正常にバックフィルされた場合は、「`\q`」と入力して *psql* を終了し、次のトラブルシューティング セクションをスキップします。 それ以外の場合は、次のトラブルシューティングのステップに進みます。

### 429 エラーが発生した場合のトラブルシューティング

"UPDATE ステートメントで 108 の埋め込みが正常にバックフィルされた場合は、このセクションをスキップしてください。"** 

1. Azure OpenAI のレート制限によっては、許容される要求の数を超えると、**429 Too Many Requests** のエラーが発生することがあります。 前の UPDATE ステートメントの場合は、次のコマンドを実行して要求をバッチ処理し、再試行できます (必要に応じて *batch_size* も手動で減らします)。

    ```sql
    DO $$
    DECLARE
      batch_size       int := 50;   -- rows per batch
      optimistic_pause int := 10;   -- seconds to wait after a successful batch
      pause_secs       int := 10;   -- current wait (resets to optimistic on success)
      max_pause        int := 60;   -- cap the backoff
      updated          int;
    BEGIN
      LOOP
        BEGIN
          WITH todo AS (
            SELECT policy_id, policy_text
            FROM company_policies
            WHERE embedding IS NULL
            ORDER BY policy_id
            LIMIT batch_size
          )
          UPDATE company_policies p
          SET embedding = azure_openai.create_embeddings('embedding', t.policy_text)::vector
          FROM todo t
          WHERE p.policy_id = t.policy_id;

          GET DIAGNOSTICS updated = ROW_COUNT;
    
          IF updated = 0 THEN
            RAISE NOTICE 'No rows left to embed.';
            EXIT;
          END IF;
    
          -- Success: reset to optimistic pause and sleep briefly
          pause_secs := optimistic_pause;
          RAISE NOTICE 'Updated % rows; sleeping % seconds before next batch.', updated, pause_secs;
          PERFORM pg_sleep(pause_secs);
    
        EXCEPTION WHEN OTHERS THEN
          -- Likely throttled (429) or transient error: back off and retry
          RAISE NOTICE 'Throttled/transient error; backing off % seconds.', pause_secs;
          PERFORM pg_sleep(pause_secs);
          pause_secs := LEAST(pause_secs * 2, max_pause);
        END;
      END LOOP;
    END $$;
    
    ```

1. 埋め込みで 108 行が正常にバックフィルされた場合は、「`\q`」と入力して *psql* を終了します。それ以外の場合は、*batch_size* を 10 減らしてみて、前のスクリプトをもう一度実行してください。

### 類似性クエリを使用してベクトル テーブルをテストする

類似性検索と SQL からの単純な直接フィルター処理を使用して検証することで、すべてが機能していることを確認しましょう。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. 次の SQL ステートメントを実行します。

    ```sql
    -- Best match for a question (cosine)
    SELECT policy_id, title, department, policy_text
    FROM company_policies
    ORDER BY embedding <=> azure_openai.create_embeddings('embedding',
             'How many vacation days do employees get?')::vector
    LIMIT 1;
    ```

1. 次の SQL ステートメントを実行して、フィルターとベクトル検索を追加します。

    ```sql
    -- Filter + vector (hybrid)
    SELECT policy_id, title, department, policy_text
    FROM company_policies
    WHERE department = 'HR'
    ORDER BY embedding <=> azure_openai.create_embeddings('embedding',
             'Does the company help me with college expenses')::vector
    LIMIT 3;
    ```

1. 「*\q*」と入力し、Enter キーを押して *psql* を終了します。

これらの回答は出発点として適していますが、より複雑なクエリでは十分に包括的ではない可能性があります。 この問題に対処するには、データベースから関連するパッセージを取得し、それらを回答を生成するためのコンテキストとして使用する Python RAG (検索拡張生成) アプリケーションを作成します。

## 自然言語の回答を取得する Python RAG アプリケーションを作成する

埋め込みができたので、質問をしたり、PostgreSQL から特に関連性の高いポリシーをフェッチしたり、それらのパッセージのみに基づいて回答を出力したりする短い Python スクリプトを作成できます。

### 環境変数を更新する

Python アプリケーションを見る前に、PostgreSQL と Azure OpenAI に適切な環境変数を設定する必要があります。

1. `.env` ファイルを開きます。

    ```bash
    code "mslearn-postgresql/Allfiles/Labs/14/.env"
    ```

1. PostgreSQL と Azure OpenAI の資格情報を使用して、`.env` ファイルを更新します。

    ```text
    # PostgreSQL connection
    PGHOST=<server FQDN from output serverFqdn>
    PGUSER=pgAdmin
    PGPASSWORD=<your admin password>
    PGDATABASE=ContosoHelpDesk
    
    # Azure OpenAI
    AZURE_OPENAI_API_KEY=<your Azure OpenAI key>
    AZURE_OPENAI_ENDPOINT=<value from output azureOpenAIEndpoint>  # e.g., https://oai-learn-<region>-<id>.openai.azure.com
    OPENAI_API_VERSION=2024-02-15-preview

    # Deployment names (match the Bicep resources)
    OPENAI_EMBED_DEPLOYMENT=embedding
    OPENAI_CHAT_DEPLOYMENT=chat
    ```

1. ファイルを保存して、"コード" エディターを閉じます。**

> [!NOTE]
> 保存/終了オプションが見つからない場合は、"コード" エディター ウィンドウで、エディターの右上にマウスを移動します。** アイコンが変わり、マウス ボタンを押すと、保存して閉じるオプションが表示されるはずです。

### Python RAG アプリケーションを更新する

クローンした GitHub リポジトリで、RAG アプリケーションのシェルを含む `app.py` ファイルを見つけることができます。 PostgreSQL からコンテキストに基づいて質問を取得して回答するロジックを実装する時間。

1. `CompanyPolicies.py` を開き、RAG ロジックを追加します。

    ```bash
    code "mslearn-postgresql/Allfiles/Labs/14/CompanyPolicies.py"
    ```

1. アプリケーションが依存するライブラリをレビューします。 Azure OpenAI との対話に使用するメイン ライブラリは `langchain_openai` です。

1. 最初の関数 `get_conn` は、PostgreSQL データベースへの接続を作成するだけです。 これはあらかじめ定義されています。 次の 3 つの関数では、指定した実際のコードにコメントを置き換えます。

1. コメント **# コサイン類似度で上位 k 行を取得する (埋め込みが存在する必要があります)** を次のスクリプトに置き換えます。

    ```python
    # Retrieve top-k rows by cosine similarity (embedding must be present)
    def retrieve_chunks(question, top_k=5):
        sql = """
        WITH q AS (
          SELECT azure_openai.create_embeddings(%s, %s)::vector AS qvec
        )
        SELECT policy_id, title, policy_text
        FROM company_policies, q
        WHERE embedding IS NOT NULL
        ORDER BY embedding <=> q.qvec
        LIMIT %s;
        """
        params = (os.getenv("OPENAI_EMBED_DEPLOYMENT"), question, top_k)
        with get_conn() as conn, conn.cursor() as cur:
            cur.execute(sql, params)
            rows = cur.fetchall()
        return [{"policy_id": r[0], "title": r[1], "text": r[2]} for r in rows]
    ```

    この関数は、ユーザーの質問に基づいて PostgreSQL データベースから上位 k 個の関連チャンクを取得します。

1. コメント **# モデル プロンプトで取得されたチャンクを書式設定する**を次のスクリプトに置き換えます。

    ```python
    # Format retrieved chunks for the model prompt
    def format_context(chunks):
        return "\n\n".join([f"[{c['title']}] {c['text']}" for c in chunks])
    ```

    この関数は、取得したチャンクをモデル プロンプトに適したコンテキスト文字列に書式設定します。

1. コメント **# Azure OpenAI を呼び出して、指定されたコンテキストを使用して回答する**を次のスクリプトに置き換えます。

    ```python
    # Call Azure OpenAI to answer using the provided context
    def generate_answer(question, chunks):
        llm = AzureChatOpenAI(
            azure_deployment=os.getenv("OPENAI_CHAT_DEPLOYMENT"),
            api_key=os.getenv("AZURE_OPENAI_API_KEY"),
            azure_endpoint=os.getenv("AZURE_OPENAI_ENDPOINT"),
            api_version=os.getenv("OPENAI_API_VERSION"),
            temperature=0,
        )
        messages = [
            {"role": "system", "content": "Answer ONLY from the provided context. If it isn't in the context, say you don’t have enough information. Cite policy titles in square brackets, e.g., [Vacation policy]."},
            {"role": "user", "content": f"Question: {question}\nContext:\n{format_context(chunks)}"},
        ]
        return llm.invoke(messages).content
    ```

    この関数は、指定されたコンテキスト チャンクを使用して、ユーザーの質問に対する回答を生成します。

1. アプリケーションの最後のセクションが、主要アプリケーション ロジックです。 コードのこの部分は、ユーザーに質問を求め、関連するチャンクを取得し、回答を生成し、ユーザーが終了を決定するまでループします。

1. ファイルを保存して、"コード" エディターを閉じます。**

## アプリケーションの実行

アプリケーションを実行する前に最後に行う必要があるのは、Python 環境を設定し、必要なパッケージをインストールすることです。 最後に、アプリケーションを実行します。

```bash
# Navigate to the exercise folder
cd ~/mslearn-postgresql/Allfiles/Labs/14

# Set up the Python environment
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# When prompted, enter a question (for example, How many vacation days do employees get?)
python CompanyPolicies.py
```

さまざまな質問を試して、モデルがどのように応答するかを確認してください。

- リモート ワークに関する会社のポリシーは何ですか?
- 一部の地域のお客様を訪問する必要がある場合に、その訪問に会社の車を使用できますか?
- 出産が予定されています。休暇を取ることができますか? 有給休暇になりますか?
- 従業員の行動に関するガイドラインは何ですか?

または、データベースに追加した既存のポリシーによって含まれている場合は、独自の質問を考えてください。 この小さな Python スクリプトが、RAG アプリケーションの基礎です。 データベース内で関連するドキュメントを検索し、それらを使用してユーザーの質問に回答します。 ただし、有効な RAG アプリケーションであるには、ドキュメントの取得が高速でスケーラブルである必要があります。 この高速での取得を実現するため、ベクトル インデックスを実装します。

## ベクトル インデックスを追加する (大規模でも高速に)

company_policies テーブルは小さかったため、クエリは比較的高速に実行された可能性があります。 しかし、テーブルが大きくなると、パフォーマンスを最適化する必要があります。 クエリのパフォーマンスを向上させるための最初のステップは、ベクトル インデックスを追加することです。

しかし、このような小さなテーブルにインデックスを追加しても、大幅な改善が見えない場合があります。 テーブルに 50,000 行を追加して、より大きなテーブルをエミュレートしてみましょう。 このラボでは、テーブルのサイズを増やすため、既存の行を複数回コピーします。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. 次の SQL ステートメントを実行して、company_policies テーブルにさらに行を挿入します。

    ```sql
    -- Inflate to ~50k rows (keeps embeddings the same; OK for a demo)
    INSERT INTO company_policies (title, department, policy_text, category, embedding)
    SELECT title || ' (copy ' || gs || ')', department, policy_text, category, embedding
    FROM company_policies
    CROSS JOIN generate_series(1, 500) AS gs;
    ```

インデックスがある場合とない場合とで、クエリの実行プランをレビューしましょう。

### ベクトル インデックスを使用せずにクエリを実行する

まずは、インデックスなしでクエリを実行してみましょう。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. 次の SQL ステートメントを実行して、インデックスなしでクエリの実行プランを評価します。
  
    ```sql
    -- Disable pagination for better output readability
    \pset pager off
    ```

    ```sql
    EXPLAIN (ANALYZE, BUFFERS)
    SELECT policy_id, title, department, policy_text
    FROM company_policies
    ORDER BY embedding <=> azure_openai.create_embeddings('embedding',
             'How many vacation days do employees get?')::vector
    LIMIT 1;
    ```

このクエリは、詳細な実行プランを返すはずです。 インデックスを使用しないため、[実行時間] と [バッファー] のメトリックは高いリソース使用率を示す可能性があります。**** クエリを 2 回目に実行する場合、クエリ プランナーはキャッシュされた結果を使用するため、パフォーマンスが向上する可能性があります。 後で比較できるように、これらのメトリックをメモしておきます。

### ベクトル インデックスを使用してクエリを実行する

IVFFlat インデックスを作成すると、テーブルが拡大しても上位 k の類似性の速度が維持されます。 最初は簡単に始め、後で調整します。

それでは、インデックスを作成しましょう。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. IVFFlat インデックスを作成します。

    ```sql
    -- Drop the IVFFlat index
    DROP INDEX IF EXISTS company_policies_embedding_ivfflat_idx;
    
    -- Use cosine distance (vector_cosine_ops) for text embeddings
    CREATE INDEX company_policies_embedding_ivfflat_idx
      ON company_policies
      USING ivfflat (embedding vector_cosine_ops)
      WITH (lists = 100);
    
    ANALYZE company_policies;
    ```

1. 次の SQL ステートメントを実行して、インデックスを使用してクエリの実行プランを評価します。

    ```sql
    -- Disable pagination for better output readability
    \pset pager off
    ```

    ```sql
    EXPLAIN (ANALYZE, BUFFERS)
    SELECT policy_id, title, department, policy_text
    FROM company_policies
    ORDER BY embedding <=> azure_openai.create_embeddings('embedding',
             'How many vacation days do employees get?')::vector
    LIMIT 1;
    ```

[実行時間] や [バッファー] のメトリックの削減など、実行プランでいくつかが改善していることに気付きます。**** また、クエリで IVFFlat インデックスが使用されていることがわかります。 クエリを複数回実行しても、パフォーマンスは一貫しているはずです。

### 要点

この演習を完了することで、Python で検索拡張アプリケーションを構築する方法を理解できるようになりました。 アプリケーションは、関連するドキュメントを取得し、ユーザー クエリにインテリジェントに応答しました。 RAG アプリケーションで何ができるかを簡単に確認しました。 さらに機能強化を行うと、ドキュメント取得と応答生成の正確性と効率を向上させることができます。

さらに、ベクトル インデックスを使用してクエリのパフォーマンスを最適化する方法を確認しました。これは、データセットの拡大に合わせてアプリケーションをスケーリングするために重要です。 これらのインデックスを使用すると、データ量が増えても、RAG アプリケーションの応答性と効率性を維持できます。

最後に、時間の経過に伴うアプリケーションの監視と微調整の重要性について学習しました。 ユーザー クエリが進化し、データセットが拡大するにつれて、最適なパフォーマンスと正確性を維持するために、インデックス作成戦略、プロンプト設計、および全体的なアーキテクチャを見直す必要があります。
