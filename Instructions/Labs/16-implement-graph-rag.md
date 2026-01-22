---
lab:
  title: Azure Database for PostgreSQL を使用して GraphRAG を実装する
  module: Build RAG applications with Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL を使用して GraphRAG を実装する

このハンズオン演習では、これまでのユニットで使用したのと同じ Azure Database for PostgreSQL の中に軽量のナレッジ グラフ レイヤーを追加します。 2 つの小さなテーブルを**ノード**と**エッジ**用に作成し、既存の `company_policies` の行にリンクしてから、**グラフで絞り込まれるベクトル検索**を実行します。この検索では最初にリレーションシップ (トピック/部署) のフィルターで絞り込み、次に pgvector でランク付けします。 目標は、多概念の質問に対して、プロンプトを複雑にしたり別のストアにデータを移動したりすることなく "精度を高める" ことです。**

次のシナリオを検討してみましょう。ある従業員が、エンジニアリング部署のリモート ワークに関する会社のポリシーについて知りたいと考えています。 単純なベクトル検索でも一般的なリモート ワーク ポリシーを返すことができますが、ナレッジ グラフを使用して結果をフィルター処理し、エンジニアリング部署に関連するものに絞り込むという方法で、より正確で関連性の高い情報を提供できます。

この演習を終了すると、次のことができるようになります。

- PostgreSQL でグラフ構造を作成して管理する方法を理解します。
- グラフで絞り込まれるベクトル検索を実行できます。
- RAG アプリケーションへのナレッジ グラフ統合の経験を積みます。

## 開始する前に

管理者権限を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要であり、そのサブスクリプションで Azure OpenAI アクセスが承認されている必要があります。 Azure OpenAI アクセスが必要な場合、[Azure OpenAI の制限付きアクセス](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access)に関するページで申請してください。

### Azure サブスクリプションにリソースをデプロイする

既に非運用の Azure Database for PostgreSQL サーバーと非運用の Azure OpenAI リソースがセットアップされている場合は、このセクションをスキップしてもかまいません。**

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

1. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](./media/16-portal-toolbar-cloud-shell.png)

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

    この Bicep デプロイ スクリプトを実行すると、この演習の実施に必要な Azure サービスがご自身のリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL サーバー、Azure OpenAI、Azure AI Language サービスなどがあります。 また、Bicep スクリプトでは、(`azure.extensions` サーバー パラメーターを使用して) PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する、サーバーに `ContosoHelpDesk` という名前のデータベースを作成する、`text-embedding-ada-002` モデルを使用して `embedding` という名前のデプロイを Azure OpenAI Service に追加するといった、いくつかの構成手順も実行されます。 最後に、`gpt-4o-mini` モデルを使用する `chat` という名前のデプロイがご自身の Azure OpenAI サービスに追加されます。 この Bicep ファイルは、このラーニング パス内のすべてのモジュールを共有するものであるため、演習によってはデプロイされたリソースの一部しか使用しないこともあります。

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

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と ContosoHelpDesk データベースの [接続] が赤いボックスで強調表示されています。](./media/16-postgresql-database-connect.png)

1. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** サインイン用にランダムに生成されたパスワードを入力します。

    サインインすると、`ContosoHelpDesk` データベースの `psql` プロンプトが表示されます。

1. この演習の残りの部分では、Cloud Shell で作業を続けます。そのため、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー ウィンドウ内でペインを拡大すると作業しやすくなります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](./media/16-azure-cloud-shell-pane-maximize.png)

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

`azure_ai` 拡張機能を使用する前に、`ContosoHelpDesk` データベースにテーブルを追加してサンプル データで事前設定します。これでアプリケーションの作成時に使用される情報が用意されます。

1. **ContosoHelpDesk** プロンプトで、次のコマンドを実行して会社のポリシーのデータを格納するための `company_policies` テーブルを作成します。

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

1. Azure Cloud Shell で、`COPY` コマンドを使用して CSV ファイルからのデータを先ほど作成した各テーブルに読み込みます。 次のコマンドを実行して `company_policies` テーブルを事前設定します。

    ```sql
    \COPY company_policies (title, department, policy_text, category) FROM 'mslearn-postgresql/Allfiles/Labs/Shared/company-policies.csv' WITH (FORMAT csv, HEADER)
    ```

    コマンドの出力は `COPY 108` となるはずです。これは、CSV ファイルから 108 行がテーブルに書き込まれたことを示しています。

1. 既存の行に対する埋め込みをバックフィルします。

    **psql** セッション (Cloud Shell) で次のコマンドを実行して、まだ埋め込みがない行について埋め込みを計算します。 `<EMBEDDING_DEPLOYMENT_NAME>` を埋め込みデプロイの名前に置き換えてください。

    ```sql
    -- Create embeddings for existing rows that currently have no embeddings
    UPDATE company_policies
    SET embedding = azure_openai.create_embeddings('<EMBEDDING_DEPLOYMENT_NAME>', policy_text)::vector
    WHERE embedding IS NULL;
    ```

    これを実行すると、Azure OpenAI 埋め込みデプロイが SQL から呼び出されて (`azure_ai` 経由で)、結果が列 `embedding` に格納されます。

108 行の埋め込みのバックフィルが正常に完了した場合は、「`\q`」と入力して *psql* を終了し、次のトラブルシューティングのセクションをスキップしてください。 それ以外の場合は、次のトラブルシューティングのステップに進みます。

### 429 エラーが発生した場合のトラブルシューティング

UPDATE ステートメントでの 108 行の埋め込みのバックフィルが正常に完了した場合は、このセクションをスキップしてください。** 

1. ユーザーの Azure OpenAI レート制限によっては、許容される要求の数を超えた場合に **429 Too Many Requests** のエラーが発生することがあります。 前の UPDATE ステートメントでこれが発生した場合は、次のコマンドを実行すると要求をバッチに分けて再試行できます (必要に応じて *batch_size* も手動で縮小してください)。

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

1. 108 行の埋め込みのバックフィルが正常に完了した場合は、「`\q`」と入力して *psql* を終了します。それ以外の場合は、*batch_size* を 10 減らして、前のスクリプトをもう一度実行してみてください。

### ベクトル インデックスを追加する

類似検索のパフォーマンスを向上させるために、`company_policies` テーブルの `embedding` 列にベクトル インデックスを追加します。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. 次のとおりに IVFFlat インデックスを作成します。

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

### 類似性クエリを使用してベクトル テーブルをテストする

すべてが機能していることを確認するために、類似性検索と単純なフィルター処理を SQL から直接実行してみましょう。

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

1. 次の SQL ステートメントを実行してフィルターとベクトル検索を追加します。

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

返される回答は、手始めとしては悪くありませんが、クエリがさらに複雑になると網羅性が不足するおそれがあります。 この問題に対処するには、関連する文をデータベースから取り出して回答生成のためのコンテキストとして使用するように Python RAG (Retrieve-Augmented Generation) アプリケーションを作成します。

## Azure portal で Apache AGE 拡張機能を有効にする

Apache AGE 拡張機能を使用するには、事前に Azure portal で有効にする必要があります。

1. Azure portal にアクセスして Azure Database for PostgreSQL インスタンスに移動します。
1. 左側のメニューの *[設定]* で、*[サーバー パラメーター]* を選びます。
1. **azure.extensions** を見つけます。
1. **[Value]** 列で、有効な拡張機能のリストに `AGE` を追加します。
1. 新規検索で **shared_preload_libraries** を検索します。
1. **[Value]** 列で、有効な拡張機能のリストに `AGE` を追加します。
1. **保存**を選択して、変更を適用します。

これで、Apache AGE 拡張機能を PostgreSQL データベースで使用できるようになります。

## ナレッジ グラフを構築する

ナレッジ グラフとは情報を構造化して表現するものであり、エンティティとそのリレーションシップを表します。 この演習では、会社のポリシーのデータからナレッジ グラフを作成します。 始めに、ノードとエッジのテーブルを作成します。

### AGE を有効にしてグラフを作成する (同じセッション)

最初に Apache AGE 拡張機能を有効にして新しいグラフを作成しましょう。 

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. 次の SQL ステートメントを実行します。

    ```sql
    -- Enable the AGE extension
    CREATE EXTENSION IF NOT EXISTS age CASCADE;
    
    -- Put ag_catalog in the session path
    SET search_path = public, ag_catalog;

    -- Create a fresh graph namespace
    SELECT ag_catalog.create_graph('company_policies_graph');
    ```

これで、ナレッジ グラフのノードとエッジを作成できるようになりました。

### グラフのノードを作成する

会社のポリシーのデータからノードを作成しましょう。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. 最初に、ポリシーのノードをアップサートするための関数を定義しましょう。 次の SQL ステートメントを実行します。

    ```sql
    DROP FUNCTION IF EXISTS public.policy_graph_upsert(BIGINT, TEXT, TEXT, TEXT, TEXT);

    -- Create a policy node
    CREATE OR REPLACE FUNCTION public.policy_graph_upsert(
      _id BIGINT, _title TEXT, _dept TEXT, _cat TEXT, _text TEXT
    ) RETURNS void
    LANGUAGE plpgsql
    VOLATILE
    AS $BODY$
    BEGIN
      -- Use ag_catalog just for this statement (doesn't leak outside the function)
      SET LOCAL search_path TO ag_catalog, public;

      EXECUTE format(
        'SELECT * FROM cypher(''company_policies_graph'', $$ 
          MERGE (p:Policy {policy_id: %s})
          SET  p.title       = %L,
                p.department  = %L,
                p.category    = %L,
                p.policy_text = %L
        $$) AS (n agtype);',
        _id, _title, _dept, _cat, _text
      );
    END
    $BODY$;
    ```

1. 次に、部署、カテゴリ、トピックのノードをアップサートするための関数を追加します。

    ```sql
    DROP FUNCTION IF EXISTS public.create_entity_in_policies_graph(TEXT, TEXT);

    -- Create a new department, category, or topic entity node in the graph
    CREATE OR REPLACE FUNCTION public.create_entity_in_policies_graph(
      _type TEXT, _name TEXT
    ) RETURNS void
    LANGUAGE plpgsql
    VOLATILE
    AS $BODY$
    BEGIN
      SET LOCAL search_path TO ag_catalog, public;

      EXECUTE format(
        'SELECT * FROM cypher(''company_policies_graph'', $$ 
          MERGE (e:Entity {type: %L, name: %L})
        $$) AS (n agtype);',
        _type, _name
      );
    END
    $BODY$;
    ```

1. 次の SQL ステートメントを実行してノードを追加します。

    ```sql
    -- Disable pagination for better output readability
    \pset pager off

    -- Upsert the policy nodes
    SELECT public.policy_graph_upsert(policy_id, title, department, category, policy_text)
    FROM public.company_policies
    ORDER BY policy_id;

    -- Departments
    SELECT public.create_entity_in_policies_graph('Department', d.department)
    FROM (SELECT DISTINCT department FROM public.company_policies) AS d;

    -- Categories
    SELECT public.create_entity_in_policies_graph('Category', c.category)
    FROM (SELECT DISTINCT category FROM public.company_policies) AS c;

    -- Topics - Group policies by list of common terms that might be mentioned in a policy
    WITH topics(name) AS (
      VALUES
        ('Employees'),
        ('Approval'),
        ('Customer'),
        ('Meetings'),
        ('Exit/Termination'),
        ('Legal'),
        ('Devices'),
        ('Events'),
        ('Expense'),
        ('New Hires'),
        ('Reconciled Monthly'),
        ('Remote'),
        ('Vendors/Suppliers'),
        ('Internet/Social Media'),
        ('Onboarding'),
        ('Prior Approval'),
        ('Products'),
        ('Reviewed Quarterly'),
        ('Tickets'),
        ('Training')
    )
    SELECT public.create_entity_in_policies_graph('Topic', name)
    FROM topics;
    ```
これで、すべてのノードが作成されました。次に、これらを接続するエッジを作成します。

### グラフのエッジを作成する

これまでに、ポリシー、部署、カテゴリ、トピックのノードを追加しました。 次に、これらを接続するエッジを作成します。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. ポリシーのノードとそれぞれの部署、カテゴリ、トピックのノードとの間のエッジを確立する関数を作成しましょう。

   ```sql
    -- Policy -> Entity edge upsert
    --   rel ∈ ('BELONGS_TO','IN_CATEGORY','MENTIONS')
    DROP FUNCTION IF EXISTS public.create_policy_link_in_policies_graph(BIGINT, TEXT, TEXT, TEXT);

    CREATE OR REPLACE FUNCTION public.create_policy_link_in_policies_graph(
      _policy_id BIGINT, _etype TEXT, _ename TEXT, _rel TEXT
    ) RETURNS void
    LANGUAGE plpgsql
    VOLATILE
    AS $BODY$
    BEGIN
      SET LOCAL search_path TO ag_catalog, public;

      EXECUTE format(
        'SELECT * FROM cypher(''company_policies_graph'', $$ 
          MATCH (p:Policy {policy_id: %s})
          MATCH (e:Entity {type: %L, name: %L})
          MERGE (p)-[:%s]->(e)
          RETURN 1
        $$) AS (ok agtype);',
        _policy_id, _etype, _ename, _rel
      );
    END
    $BODY$;
    ```

1. 次の SQL コマンドを実行してポリシーのノードとそれぞれの部署、カテゴリ、トピックのノードの間にエッジを作成します。

    ```sql
    -- Disable pagination for better output readability
    \pset pager off

    -- BELONGS_TO
    SELECT public.create_policy_link_in_policies_graph(policy_id, 'Department', department, 'BELONGS_TO')
    FROM public.company_policies;

    -- IN_CATEGORY
    SELECT public.create_policy_link_in_policies_graph(policy_id, 'Category', category, 'IN_CATEGORY')
    FROM public.company_policies;

    -- MENTIONS - Note that you use some regex patterns to match similar terms
    WITH topics(name, pattern) AS (
      VALUES
        ('Employees',              $$\memployee(s)?\M$$),
        ('Approval',               $$\mapprov(e|al|ed|als|ing)?\M$$),
        ('Customer',               $$\mcustomer(s)?\M$$),
        ('Meetings',               $$\mmeeting(s)?\M$$),
        ('Exit/Termination',       $$\m(exit|termination)\M$$),
        ('Legal',                  $$\mlegal\M$$),
        ('Devices',                $$\m(device(s)?|laptop(s)?)\M$$),
        ('Events',                 $$\mevent(s)?\M$$),
        ('Expense',                $$\mexpense(s)?\M$$),
        ('New Hires',              $$\mnew\M\s+\mhires\M$$),
        ('Reconciled Monthly',     $$\mreconciled\M\s+\mmonthly\M$$),
        ('Remote',                 $$\mremote\M(\s+\mwork\M)?$$),
        ('Vendors/Suppliers',      $$\m(vendor(s)?|supplier(s)?)\M$$),
        ('Internet/Social Media',  $$\minternet\M|\msocial\M\s+\mmedia\M$$),
        ('Onboarding',             $$\monboard(ed|ing)?\M|\monboarding\M$$),
        ('Prior Approval',         $$\mprior\M\s+\mapproval\M$$),
        ('Products',               $$\mproduct(s)?\M$$),
        ('Reviewed Quarterly',     $$\mreviewed\M\s+\mquarterly\M$$),
        ('Tickets',                $$\mticket(s)?\M|\mhelp\M\s*\mdesk\M$$),
        ('Training',               $$\mtrain(ing|ed|s)?\M$$)
    )
    SELECT public.create_policy_link_in_policies_graph(p.policy_id, 'Topic', t.name, 'MENTIONS')
    FROM public.company_policies p
    JOIN topics t
      ON p.policy_text ~* t.pattern
    GROUP BY t.name, p.policy_id
    ORDER BY t.name, p.policy_id;
    ```

1. ここで、作成したグラフのノードとエッジの数を確認してみましょう。

    ```sql
    -- Total policy nodes
    SELECT * FROM cypher('company_policies_graph',
    $$ MATCH (p:Policy) RETURN count(p) $$) AS (count agtype);

    -- Entities by type (Department/Category/Topic)
    SELECT * FROM cypher('company_policies_graph',
    $$ MATCH (e:Entity) RETURN e.type, count(e) ORDER BY e.type $$) AS (type agtype, count agtype);

    -- Edges by relationship type
    SELECT * FROM cypher('company_policies_graph',
    $$ MATCH ()-[r]->() RETURN type(r), count(r) ORDER BY type(r) $$) AS (rel agtype, count agtype);
    ```

これで会社のポリシーの完全なグラフ表現が作成されました。これには関連するすべてのエンティティとそれぞれのリレーションシップも含まれています。

## グラフで絞り込まれるベクトル検索を実行する

ポリシー、部署、トピックのノードが作成されてこれらがエッジで接続されたので、これでグラフを使用して**候補の絞り込み**をしてから **pgvector** を使用してこれらの候補を質問に対するセマンティック類似性によって**ランク付け**できるようになりました。 このプロセスでは、取得フロー全体が *Azure Database for PostgreSQL* の中にとどまります。

### 部署とトピックを使用して候補を絞り込む

検索するポリシーを、**財務**という部署に属していて特定のトピックに言及しているものに絞り込んでみましょう。

1. Azure Cloud Shell で、前と同様に *psql* を使用して *ContosoHelpDesk* データベースに接続します。

1. *ContosoHelpDesk* プロンプトで、検索したい質問を変数として設定します。

    ```
    \set question 'What expenses require prior approval for remote work travel?'
    ```

1. 次の SQL ステートメントを実行してポリシー文のうち上位 5 件を取得します。 このグラフは、**財務 (Finance)** のポリシーのうち、選択されたトピックのいずれかに**言及 (mention)** しているものを選び出します。 ベクトルのステップで、これらの候補を質問に対するセマンティック類似性でランク付けします。 小さな重複除去のステップで、重複を取り除いてポリシーごとに 1 行だけを残します。

    ```sql
    /* Graph-narrowed vector search: filter by graph, then rank with pgvector */
    WITH
    /* 1) GRAPH FILTER: candidate policy_ids from AGE */
    graph_ids AS (
      SELECT ((pid)::text)::bigint AS policy_id
      FROM ag_catalog.cypher('company_policies_graph'::name, $$
        MATCH (p:Policy)-[:BELONGS_TO]->(:Entity {type:'Department', name:'Finance'})
        MATCH (p)-[:MENTIONS]->(t:Entity {type:'Topic'})
        WHERE t.name IN ['Expense','Approval','Remote']   /* adjust topics as needed */
        RETURN p.policy_id AS pid
      $$::cstring) AS (pid agtype)
    ),

    /* 2) QUESTION EMBEDDING: compute once from the psql 'question' variable */
    q AS (
      SELECT azure_openai.create_embeddings('embedding', :'question')::vector AS qv
    ),

    /* 3) VECTOR RANK: smaller cosine distance is better */
    ranked AS (
      SELECT
        cp.policy_id,
        cp.title,
        cp.department,
        cp.category,
        cp.policy_text,
        (cp.embedding <=> q.qv) AS distance
      FROM public.company_policies cp
      JOIN graph_ids USING (policy_id)
      CROSS JOIN q
      WHERE cp.embedding IS NOT NULL
    ),

    /* 4) DEDUP: keep the best (smallest distance) row per policy */
    dedup AS (
      SELECT *,
             ROW_NUMBER() OVER (PARTITION BY policy_id ORDER BY distance) AS rn
      FROM ranked
    )

    /* 5) RESULT: unique top 5 */
    SELECT policy_id, title, department, category, policy_text
    FROM dedup
    WHERE rn = 1
    ORDER BY distance
    LIMIT 5;
    ```

> [!TIP]  
> `psql` で、`\set question` コマンド**だけを入力して実行**し、**Enter** キーを押してから SQL クエリを実行してください。 両方を一度に貼り付けると、共通テーブル式 (CTE) が期待どおりに実行されないことがあります。

このクエリは、指定された質問に関連するポリシー文をランク付けしたリストを生成するものです。 最初にグラフ構造を使用して候補を部署とトピックのフィルターで絞り込み、その結果を質問に対するセマンティック類似性でランク付けします。

別のフィルターを試してみましょう。

- **トピックのみ (部署フィルターなし)** — 前のフィルターでは *Finance* (財務) という部署に絞り込んでいましたが、すべての部署が含まれるようにしましょう。 次のクエリを実行します。

    ```sql
    WITH graph_ids AS (
      SELECT ((pid)::text)::bigint AS policy_id
      FROM ag_catalog.cypher('company_policies_graph'::name, $$
        MATCH (p:Policy)-[:MENTIONS]->(t:Entity {type:'Topic'})
        WHERE t.name IN ['Customer','Meetings','Expense']
        RETURN p.policy_id AS pid
      $$::cstring) AS (pid agtype)
    ),
    q AS (SELECT azure_openai.create_embeddings('embedding', :'question')::vector AS qv),
    ranked AS (
      SELECT cp.policy_id, cp.title, cp.department, cp.category, cp.policy_text,
             (cp.embedding <=> q.qv) AS distance
      FROM public.company_policies cp
      JOIN graph_ids USING (policy_id)
      CROSS JOIN q
      WHERE cp.embedding IS NOT NULL
    ),
    dedup AS (
      SELECT *, ROW_NUMBER() OVER (PARTITION BY policy_id ORDER BY distance) AS rn
      FROM ranked
    )
    SELECT policy_id, title, department, category, policy_text
    FROM dedup
    WHERE rn = 1
    ORDER BY distance
    LIMIT 5;
    ```

  候補の集合がどのように変化したかに注目してください。候補の範囲が広がるようにトピックのフィルターが指定されているためです。

- **トピックのリストを変更する** — スクリプト内の `['Expense','Approval','Remote']` を、前に作成した 20 個のトピックから任意に選んだ集合 (たとえば `['Customer','Meetings','Expense']`) に置き換えます。 その後で、クエリをもう一度実行します。 候補の集合が、選択されたトピックに基づいてどのように再び変化するかに注目してください。

質問を変更するには、`psql` で `\set question` コマンドに変更を加えるという方法もあります。 次の質問を使って、前のクエリを再実行してみてください。

- カスタマー インタラクションに関連するポリシーはどれですか?
- 会議のメモとアクション アイテムをどのように扱うのですか?
- リモート ワークと出張のガイドラインはどのようなものがありますか?
- どのようにしてデータ プライバシー規制を確実に遵守するのですか?

独自の質問を試すこともできるので、トピックのリストを自由に変更してみてください。

グラフとベクトル検索のスキルを組み合わせると、強力な検索アプリケーションを作成できるようになります。 扱うデータセットが大きいほど、検索能力の効果も大きくなります。

## 要点

この演習では、小さなグラフを使用して取得に構造を追加しました。 よく似ているテキストのみに依存するのではなく、最初に部署やトピックなどのつながりによって一連の候補を取り出してから、そのリストを `pgvector` で質問を基準としてランク付けします。 このすべてが *Azure Database for PostgreSQL* の中の 1 つのデータベース内で実行されるため、フローがシンプルなままで運用しやすくなり、フィルターとパスが明示されるため説明も簡単になります。

ここで説明した方法をご自身のデータに適用するには、小さく始めてください。 注目したいエンティティとリレーションシップをいくつか選び出して、ご自身の行にリンクし、短い `openCypher` クエリを使用して候補の `ids` をフェッチしてから、ベクトル ランク付けを適用します。 必要に応じて検索対象を絞り込むか広げるようにフィルターを設定し、他の概念と入れ替えます。ワークフローが SQL の中にとどまるようにすると、保守が単純明快になります。
