---
lab:
  title: Azure AI 拡張機能を探索する
  module: Explore Generative AI with Azure Database for PostgreSQL
---

# Azure AI 拡張機能を探索する

Margie's Travel のリード開発者として、顧客に賃貸物件に関するインテリジェントな推奨事項を提供する AI 搭載アプリケーションをビルドする作業を行っています。 Azure Database for PostgreSQL の `azure_ai` 拡張機能の詳細と、それが生成 AI (GenAI) の機能をアプリに統合するのにどのように役立つかについて知りたいと考えています。 この演習では、`azure_ai` 拡張機能を Azure Database for PostgreSQL フレキシブル サーバー データベースにインストールし、Azure AI と ML サービスを統合する機能を調べることで、その拡張機能を確認します。

## 開始する前に

管理者権限を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要であり、そのサブスクリプションで Azure OpenAI アクセスが承認されている必要があります。 Azure OpenAI アクセスが必要な場合、[Azure OpenAI の制限付きアクセス](https://learn.microsoft.com/legal/cognitive-services/openai/limited-access)に関するページで申請してください。

### Azure サブスクリプションにリソースをデプロイする

この手順では、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/12-portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用している場合は、*Bash* シェルに切り替えます。

3. Cloud Shell プロンプトで、次のように入力して、演習リソースを含む GitHub リポジトリを複製します。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 次に、Azure CLI コマンドを使用して Azure リソースを作成するときに、冗長な型指定を減らすための変数を定義する 3 つのコマンドを実行します。 この変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースがデプロイされる Azure リージョン (`REGION`)、PostgreSQL 管理者ログイン用にランダムに生成されたパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドで、対応する変数に割り当てられるリージョンは `eastus` ですが、任意の場所に置き換えることもできます。 ただし、既定値を置き換える場合は、このラーニング パスのモジュール内のすべてのタスクを完了できるように、[抽象的な要約をサポートしている別の Azure リージョン](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)を選択する必要があります。

    ```bash
    REGION=eastus
    ```

    次のコマンドで、この演習で使用されるすべてのリソースを格納するリソース グループに使用する名前が割り当てられます。 対応する変数に割り当てられたリソース グループ名は `rg-learn-postgresql-ai-$REGION` で、その中の `$REGION` は上で指定した場所です。 ただし、この部分は好みに合わせて他のリソース グループ名に変更できます。

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    最後のコマンドでは、PostgreSQL 管理者ログイン用のパスワードがランダムに生成されます。 後で PostgreSQL フレキシブル サーバーに接続するときに使用するために、このパスワードを安全な場所にコピーしてください。

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

5. 複数の Azure サブスクリプションにアクセスでき、既定のサブスクリプションがこの演習でリソース グループとその他のリソースを作成するものでない場合は、次のコマンドを実行して適切なサブスクリプションを設定し、`<subscriptionName|subscriptionId>` トークンを使用するサブスクリプションの名前または ID に置き換えます。

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. 次の Azure CLI コマンドを実行して、リソース グループを作成します。

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. 最後に、Azure CLI を使用して Bicep デプロイ スクリプトを実行し、リソース グループ内の Azure リソースをプロビジョニングします。

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL フレキシブル サーバー、Azure OpenAI、Azure AI 言語サービスなどです。 また、Bicep スクリプトでは、PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する (azure.extensions サーバー パラメーターを使用)、サーバーに `rentals` という名前のデータベースを作成する、`text-embedding-ada-002` モデルを使用して `embedding` という名前のデプロイを Azure OpenAI サービスに追加するなど、いくつかの構成手順も実行されます。 なお、Bicep ファイルは、このラーニング パス内のすべてのモジュールで共有されるため、一部の演習ではデプロイされたリソースの一部のみを使用できます。

    デプロイが完了するまでに通常数分かかります。 Cloud Shell から監視するか、上で作成したリソース グループの **[デプロイ]** ページに移動し、そこでデプロイの進行状況を確認することができます。

 8. リソースのデプロイが完了したら、Cloud Shell 画面を閉じます。
 
### デプロイ エラーのトラブルシューティング

Bicep デプロイ スクリプトを実行すると、いくつかエラーが発生する場合があります。

- 以前にこのラーニング パスで Bicep デプロイ スクリプトを実行してその後リソースを削除した場合、リソースを削除してから 48 時間以内にスクリプトをまた実行しようとすると、次のようなエラー メッセージが表示される場合があります。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    このメッセージが表示された場合は、`restore` パラメーターが `true` に設定されるように上記の `azure deployment group create` コマンドを変更し、再実行します。

- 選択したリージョンで特定のリソースのプロビジョニングが制限されている場合は、`REGION` 変数を別の場所に設定してコマンドを再実行し、リソース グループを作成して、Bicep デプロイ スクリプトを実行する必要があります。

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 責任ある AI 契約に同意する必要があるためにスクリプトで AI リソースを作成できない場合は、次のエラーが発生する場合があります。その場合は、Azure portal ユーザー インターフェイスを使用して Azure AI サービス リソースを作成し、デプロイ スクリプトを再実行します。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Azure Cloud Shell で psql を使用してデータベースに接続する

このタスクでは、[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL フレキシブル サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL フレキシブル サーバーに移動します。

2. リソース メニューの **[設定]** で **[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/12-postgresql-rentals-database-connect.png)

3. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** のログイン用にランダムに生成されたパスワードを入力します。

    ログインすると、`rentals` データベースの `psql` プロンプトが表示されます。

4. この演習の残りの部分は Cloud Shell で作業を続けるため、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー画面内にペインを展開すると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/12-azure-cloud-shell-pane-maximize.png)

## データベースにサンプル データを設定する

`azure_ai` 拡張機能を調べる前に、機能を確認するときに扱う情報が得られるように、`rentals` データベースにテーブルをいくつか追加し、それらにサンプル データを入力します。

1. 次のコマンドを実行して、賃貸物件の一覧と顧客レビュー データを格納するための `listings` テーブルと `reviews` テーブルを作成します。

    ```sql
    DROP TABLE IF EXISTS listings;
    
    CREATE TABLE listings (
      id int,
      name varchar(100),
      description text,
      property_type varchar(25),
      room_type varchar(30),
      price numeric,
      weekly_price numeric
    );
    ```

    ```sql
    DROP TABLE IF EXISTS reviews;
    
    CREATE TABLE reviews (
      id int,
      listing_id int, 
      date date,
      comments text
    );
    ```

2. 次に、`COPY` コマンドを使用して、上記で作成した各テーブルに CSV ファイルからデータを読み込みます。 まず、次のコマンドを実行して、`listings` テーブルにデータを入力します。

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    コマンドの出力は `COPY 50` となるはずです。これは、CSV ファイルから 50 行がテーブルに書き込まれたことを示しています。

3. 最後に、次のコマンドを実行して、顧客レビューを `reviews` テーブルに読み込みます。

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    コマンドの出力は `COPY 354` となるはずです。これは、CSV ファイルから 354 行がテーブルに書き込まれたことを示しています。

## `azure_ai` 拡張機能のインストールと構成

`azure_ai` 拡張機能を使用する前に、それをデータベースにインストールし、Azure AI サービス リソースに接続するように構成する必要があります。 `azure_ai` 拡張機能を使用すると、Azure OpenAI および Azure AI Language サービスをデータベースに統合できます。 データベースで拡張機能を有効にするには、次の手順に従います。

1. `psql` プロンプトで次のコマンドを実行して、環境のセットアップ時に実行した Bicep 配置スクリプトによって `azure_ai` および `vector` 拡張機能がサーバーの_許可リスト_に正常に追加されたことを確認します。

    ```sql
    SHOW azure.extensions;
    ```

    このコマンドは、サーバーの_許可リスト_上の拡張機能の一覧を表示します。 すべてが正しくインストールされている場合、出力には次のように `azure_ai` と `vector` が含まれるはずです。

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Azure Database for PostgreSQL フレキシブル サーバー データベースに拡張機能をインストールして使用するには、「[PostgreSQL 拡張機能の使用方法](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)」の説明に従って、サーバーの_許可リスト_に拡張機能を追加する必要があります。

2. これで、[CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) コマンドを使用して `azure_ai` 拡張機能をインストールできるようになりました。

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` は、スクリプト ファイルを実行して新しい拡張機能をデータベースに読み込みます。 このスクリプトは通常、関数、データ型、スキーマなどの新しい SQL オブジェクトを作成します。 同じ名前の拡張機能が既に存在する場合は、エラーがスローされます。 `IF NOT EXISTS` を追加すると、コマンドが既にインストールされている場合はエラーをスローせずに実行できます。

## `azure_ai` 拡張機能に含まれているオブジェクトを確認する

`azure_ai` 拡張機能に含まれているオブジェクトを確認すると、その機能について理解を深めることができます。 このタスクでは、拡張機能によってデータベースに追加されたさまざまなスキーマ、ユーザー定義関数 (UDF)、複合型を調べます。

1. Cloud Shell で `psql` を使用する場合、クエリ結果の拡張表示を有効にすると、後続のコマンドの出力が読みやすくなるので、役に立つ場合があります。 次のコマンドを実行して、拡張表示を自動的に適用できるようにします。

    ```sql
    \x auto
    ```

2. [`\dx` メタ コマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DX-LC)は、拡張機能に含まれているオブジェクトを一覧表示するために使用されます。 `psql` コマンド プロンプトから以下を実行して、`azure_ai` 拡張機能内のオブジェクトを表示します。 すべてのオブジェクトを一覧表示するには、場合によって、スペース バーを押す必要があります。

    ```psql
    \dx+ azure_ai
    ```

    このメタ コマンド出力は、`azure_ai` 拡張機能によって、4 つのスキーマ、複数のユーザー定義関数 (UDF)、いくつかの複合型、および `azure_ai.settings` テーブルがデータベースに作成されることを示しています。 スキーマ以外は、すべてのオブジェクト名の前に、それが属するスキーマが付きます。 スキーマは、拡張機能がバケットに追加する関連関数および型をグループ化するために使用されます。 拡張機能によって追加されたスキーマの一覧とそれぞれの簡単な説明を次の表に示します。

    | [スキーマ]      | 説明                                              |
    | ----------------- | ------------------------------------------------------------------------------------------------------ |
    | `azure_ai`    | 構成テーブルと、拡張機能を操作するための UDF が存在するプリンシパル スキーマ。 |
    | `azure_openai`  | Azure OpenAI エンドポイントの呼び出しを有効にする UDF が含まれています。                    |
    | `azure_cognitive` | データベースと Azure AI サービスの統合に関連する UDF と複合型を提供します。     |
    | `azure_ml`    | Azure Machine Learning (ML) サービスを統合するための UDF が含まれています。                |

### Azure AI スキーマについて調べる

`azure_ai` スキーマは、データベースから Azure AI サービスおよび ML サービスを直接操作するためのフレームワークを提供します。 これには、これらのサービスへの接続を設定し、同じスキーマでホストされている `settings` テーブルからそれらを取得するための関数が含まれています。 `settings` テーブルは、Azure AI サービスおよび ML サービスに関連付けられているエンドポイントとキー用の、データベース内の安全なストレージとなります。

1. スキーマで定義されている関数を確認するには、[`\df` メタ コマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)を使用して、関数を表示するスキーマを指定できます。 以下を実行すると、`azure_ai` スキーマ内の関数が表示されます。

    ```sql
    \df azure_ai.*
    ```

    コマンドの出力は、次のようなテーブルになります。

    ```sql
                  List of functions
     Schema |  Name  | Result data type | Argument data types | Type 
    ----------+-------------+------------------+----------------------+------
     azure_ai | get_setting | text      | key text      | func
     azure_ai | set_setting | void      | key text, value text | func
     azure_ai | version  | text      |           | func
    ```

    `set_setting()` 関数を使用すると、拡張機能から Azure AI サービスおよび ML サービスに接続できるように、それらのサービスのエンドポイントとキーを設定できます。 この関数は、**キー**と、キーに割り当てる**値**を受け取ります。 `azure_ai.get_setting()` 関数を使用すると、`set_setting()` 関数で設定した値を取得できます。 この関数は、表示する設定の**キー**を受け取り、割り当てられた値を返します。 どちらの方法でも、キーは次のいずれかである必要があります。

    | Key | 説明 |
    | --- | ----------- |
    | `azure_openai.endpoint` | サポートされている OpenAI エンドポイント (例: <https://example.openai.azure.com>)。 |
    | `azure_openai.subscription_key` | Azure OpenAI リソースのサブスクリプション キー。 |
    | `azure_cognitive.endpoint` | サポートされている Azure AI サービス エンドポイント (例: <https://example.cognitiveservices.azure.com>)。 |
    | `azure_cognitive.subscription_key` | Azure AI サービス リソースのサブスクリプション キー。 |
    | `azure_ml.scoring_endpoint` | サポートされている Azure ML スコアリング エンドポイント (例: <https://example.eastus2.inference.ml.azure.com/score>) |
    | `azure_ml.endpoint_key` | Azure ML デプロイのエンドポイント キー。 |

    > 重要
    >
    > API キーを含む Azure AI サービスの接続情報はデータベースの構成テーブルに格納されるため、`azure_ai` 拡張機能で `azure_ai_settings_manager` というロールが定義されることで、この情報が保護され、そのロールが割り当てられたユーザーのみがアクセスできるようになります。 このロールを使用すると、拡張機能に関連する設定の読み取りと書き込みを行えるようになります。 `azure_ai.get_setting()` および `azure_ai.set_setting()` 関数を呼び出すことができるのは、`azure_ai_settings_manager` ロールのメンバーだけです。 Azure Database for PostgreSQL フレキシブル サーバーでは、すべての管理者ユーザー (`azure_pg_admin` ロールが割り当てられたユーザー) に `azure_ai_settings_manager` ロールが割り当てられます。

2. `azure_ai.set_setting()` 関数と `azure_ai.get_setting()` 関数の使用方法を示すには、Azure OpenAI アカウントへの接続を構成します。 Cloud Shell が開いているのと同じブラウザー タブを使用して、Cloud Shell 画面を最小化または復元し、[Azure portal](https://portal.azure.com/) で Azure OpenAI リソースに移動します。 Azure OpenAI リソース ページに移動したら、リソース メニューの **[リソース管理]** セクションで **[キーとエンドポイント]** を選択し、エンドポイントと使用可能なキーの 1 つをコピーします。

    ![KEY 1 とエンドポイントのコピー ボタンが赤い四角で強調表示された、Azure OpenAI サービスの [キーとエンドポイント] ページのスクリーンショットが表示されます。](media/12-azure-openai-keys-and-endpoints.png)

    `KEY 1` または `KEY 2` を使用できます。 常に 2 つのキーを用意しておくと、サービスを中断させることなく、キーのローテーションと再生成を安全に行うことができます。

3. エンドポイントとキーが用意できたら、もう一度 Cloud Shell 画面を最大化し、次のコマンドを使用して値を構成テーブルに追加します。 `{endpoint}` トークンと `{api-key}` トークンは、Azure portal からコピーした値に必ず置き換えてください。

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '{api-key}');
    ```

4. 次のクエリで `azure_ai.get_setting()` 関数を使用して、`azure_ai.settings` テーブルに書き込まれた設定を確認できます。

    ```sql
    SELECT azure_ai.get_setting('azure_openai.endpoint');
    SELECT azure_ai.get_setting('azure_openai.subscription_key');
    ```

    `azure_ai` 拡張機能は現在 Azure OpenAI アカウントに接続されています。

### Azure OpenAI スキーマを確認する

`azure_openai` スキーマは、Azure OpenAI を使用して、テキスト値のベクター埋め込みの作成をデータベースに統合する機能を提供します。 このスキーマを使用すると、データベースから直接 [Azure OpenAI で埋め込みを生成](https://learn.microsoft.com/azure/ai-services/openai/how-to/embeddings)して、入力テキストのベクター表現を作成できます。作成したベクター表現は、ベクター類似性検索に使用でき、機械学習モデルで使用することもできます。 このスキーマには、オーバーロードが 2 つある 1 つの関数 `create_embeddings()` が含まれています。 1 つのオーバーロードは 1 つの入力文字列を受け取り、もう 1 つは入力文字列の配列を要求します。

1. 上で行ったように、[`\df` meta-command](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC) を使用して、`azure_openai` スキーマ内の関数の詳細を表示できます。

    ```sql
    \df azure_openai.*
    ```

    出力には `azure_openai.create_embeddings()` 関数の 2 つのオーバーロードが表示され、関数の 2 つのバージョンと返される型の違いを確認できます。 出力の `Argument data types` プロパティにより、関数の 2 つのオーバーロードが要求する引数の一覧が表示されます。

    | 引数    | 型       | Default | 説明                                                          |
    | --------------- | ------------------ | ------- | ------------------------------------------------------------------------------------------------------------------------------ |
    | deployment_name | `text`      |    | `text-embedding-ada-002` モデルを含む Azure OpenAI Studio でのデプロイの名前。               |
    | input     | `text` または `text[]` |    | 埋め込みを作成する入力テキスト (またはテキストの配列)。                                |
    | batch_size   | `integer`     | 100  | `text[]` の入力を要求するオーバーロード専用。 一度に処理するレコードの数を指定します。          |
    | timeout_ms   | `integer`     | 3600000 | 操作が停止するまでのタイムアウト時間 (ミリ秒単位)。                                 |
    | throw_on_error | `boolean`     | true  | 関数がエラー時に例外をスローしてトランザクションをラップするロールバックを行うかどうかを示すフラグ。 |
    | max_attempts  | `integer`     | 1   | エラー発生時の Azure OpenAI サービスへの呼び出しの再試行回数。                     |
    | retry_delay_ms | `integer`     | 1000  | Azure OpenAI サービス エンドポイントの呼び出しを再試行する前の待機時間 (ミリ秒単位)。        |

2. 関数のシンプルな使用例を提供するには、次のクエリを実行します。このクエリでは、`listings` テーブルの `description` フィールドに対するベクター埋め込みを作成します。 関数の `deployment_name` パラメーターは `embedding` に設定されます。これは、Azure OpenAI サービスでの `text-embedding-ada-002` モデルのデプロイの名前です (Bicep デプロイ スクリプトによってその名前で作成されました)。

    ```sql
    SELECT
      id,
      name,
      azure_openai.create_embeddings('embedding', description) AS vector
    FROM listings
    LIMIT 1;
    ```

    出力の例を次に示します。

    ```sql
     id |      name       |              vector
    ----+-------------------------------+------------------------------------------------------------
      1 | Stylish One-Bedroom Apartment | {0.020068742,0.00022734122,0.0018286322,-0.0064167166,...}
    ```

    簡潔にするために、上の出力ではベクター埋め込みを省略しています。

    [埋め込み](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#embeddings)は、機械学習と自然言語処理 (NLP) の概念で、単語、ドキュメント、エンティティなどのオブジェクトが多次元空間の[ベクトル](https://learn.microsoft.com/azure/postgresql/flexible-server/generative-ai-overview#vectors)として表現されます。 埋め込みにより、機械学習モデルで情報の 2 つの部分の関連性を評価できます。 この手法では、データ間の関連性と類似性を効率的に識別して、アルゴリズムでパターンを識別したり正確に予測したりすることができます。

    `azure_ai` 拡張機能を使用すると、入力テキストの埋め込みを生成できます。 生成されたベクトルをデータベース内の他のデータと共に格納できるようにするには、[データベースでベクトルのサポートを有効にする](https://learn.microsoft.com/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension)方法に関する記事のガイダンスに従って、`vector` 拡張機能をインストールする必要があります。 ただし、これはこの演習の範囲外です。

### azure_cognitive スキーマを探索する

`azure_cognitive` スキーマは、データベースから Azure AI サービスを直接操作するためのフレームワークを提供します。 このスキーマでの Azure AI サービス統合は、データベースから直接アクセスできる豊富な AI 言語機能のセットを提供します。 この機能には、感情分析、言語検出、キー フレーズ抽出、エンティティ認識、テキスト要約、翻訳が含まれます。 これらの機能は、[Azure AI Language サービス](https://learn.microsoft.com/azure/ai-services/language-service/overview)を通じて利用できます。

1. スキーマで定義されているすべての関数を確認するには、前に行った [`\df` メタコマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)を使用できます。 `azure_cognitive` スキーマの関数を表示するには、次を実行します。

    ```sql
    \df azure_cognitive.*
    ```

2. このスキーマには多数の関数が定義されているため、[`\df` メタコマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)からの出力は読み取りが困難な場合があり、小さなチャンクに分割することをお勧めします。 次を実行して、`analyze_sentiment()` 関数のみを確認します。

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    出力で、関数に 3 つのオーバーロードがあり、1 つは 1 つの入力文字列を受け入れ、他の 2 つはテキストの配列を要求することを確認します。 出力には、関数のスキーマ、名前、結果のデータ型、および引数のデータ型が表示されます。 この情報は、関数の使用方法を理解するのに役立ちます。

3. `analyze_sentiment` 関数名を次の各関数名に置き換えて上記のコマンドを繰り返し、スキーマで使用可能なすべての関数を調べます。

   - `detect_language`
   - `extract_key_phrases`
   - `linked_entities`
   - `recognize_entities`
   - `recognize_pii_entities`
   - `summarize_abstractive`
   - `summarize_extractive`
   - `translate`

    各関数について、関数のさまざまな形式と、その想定されている入力と結果のデータ型を調べます。

4. `azure_cognitive` スキーマには、関数以外にも、さまざまな関数からの戻りデータ型として使用される複合型がいくつかあります。 クエリの出力を正しく処理できるように、関数が返すデータ型の構造を理解することが不可欠です。 例として、`sentiment_analysis_result` の型を調べるには次のコマンドを実行します。

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. 上記のコマンドの出力により、`sentiment_analysis_result` の型が `tuple` であることがわかります。 次のコマンドを実行して `sentiment_analysis_result` の型に含まれる列を確認することで、その `tuple` の構造をさらに詳しく調べることができます。

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    そのコマンドの出力は次のようになるはずです。

    ```sql
             Composite type "azure_cognitive.sentiment_analysis_result"
       Column  |   Type   | Collation | Nullable | Default | Storage | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment   | text      |     |     |    | extended | 
     positive_score | double precision |     |     |    | plain  | 
     neutral_score | double precision |     |     |    | plain  | 
     negative_score | double precision |     |     |    | plain  |
    ```

    `azure_cognitive.sentiment_analysis_result` は、入力テキストのセンチメント予測を含んだ複合型です。 これには、肯定的、否定的、中立的、または混在のセンチメントと、テキストに含まれる肯定的、中立的、否定的な側面のスコアが含まれます。 スコアは、0 から 1 の間の実数として表されます。 たとえば、(neutral, 0.26, 0.64, 0.09) では、センチメントは中立的で、肯定的のスコアは 0.26、中立的は 0.64、否定的は 0.09 です。

6. `azure_openai` 関数と同様に、`azure_ai` 拡張機能を使用して Azure AI サービスに対する呼び出しを正常に行うには、Azure AI Language サービスのエンドポイントとキーを指定する必要があります。 Cloud Shell が開いているのと同じブラウザー タブを使用して、Cloud Shell 画面を最小化または復元し、[Azure portal](https://portal.azure.com/) で言語サービス リソースに移動します。 リソース メニューの **[リソース管理]** セクションで、**[キーとエンドポイント]** を選択します。

    ![KEY 1 とエンドポイントのコピー ボタンが赤い四角で強調表示された、Azure Language サービスの [キーとエンドポイント] ページのスクリーンショットが表示されます。](media/12-azure-language-service-keys-and-endpoints.png)

7. エンドポイントとアクセス キーの値をコピーし、`{endpoint}` トークンと `{api-key}` トークンを Azure portal からコピーした値に置き換えます。 Cloud Shell をもう一度最大化し、Cloud Shell の `psql` コマンド プロンプトからコマンドを実行して、構成テーブルに値を追加します。

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

8. 次に、次のクエリを実行して、いくつかのレビューのセンチメントを分析します。

    ```sql
    SELECT
      id,
      comments,
      azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id IN (1, 3);
    ```

    出力、`(mixed,0.71,0.09,0.2)`、および `(positive,0.99,0.01,0)` の `sentiment` 値を確認します。 これらは、上記のクエリで `analyze_sentiment()` 関数によって返される `sentiment_analysis_result` を表します。 分析は、`reviews` テーブルの `comments` フィールドに対して実行されました。

## Azure ML スキーマを調べる

`azure_ml` スキーマを使用すると、関数はデータベースから直接 Azure ML サービスに接続できます。

1. スキーマで定義されている関数を確認するには、[`\df` メタコマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)を使用できます。 `azure_ml` スキーマの関数を表示するには、次を実行します。

    ```sql
    \df azure_ml.*
    ```

    出力で、このスキーマに定義されている 2 つの関数 (`azure_ml.inference()` と `azure_ml.invoke()`) があることを確認します。詳細は次のとおりです。

    ```sql
                  List of functions
    -----------------------------------------------------------------------------------------------------------
    Schema       | azure_ml
    Name        | inference
    Result data type  | jsonb
    Argument data types | input_data jsonb, deployment_name text DEFAULT NULL::text, timeout_ms integer DEFAULT NULL::integer, throw_on_error boolean DEFAULT true, max_attempts integer DEFAULT 1, retry_delay_ms integer DEFAULT 1000
    Type        | func
    ```

    `inference()` 関数は、トレーニング済みの機械学習モデルを使用して、新しい未確認のデータに基づいて予測や出力生成を行うことができます。

    エンドポイントとキーを指定することで、Azure OpenAI エンドポイントや Azure AI サービス エンドポイントに接続した場合と同様に、Azure ML がデプロイされたエンドポイントに接続できます。 Azure ML を操作するにはトレーニング済みでデプロイ済みのモデルが必要であるため、この演習の範囲外であり、その接続を設定してここで試すことはしません。

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/12-azure-portal-home-azure-services-resource-groups.png)

2. フィールド検索ボックスのフィルターに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/12-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。
