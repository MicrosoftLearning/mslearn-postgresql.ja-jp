---
lab:
  title: センチメントを分析する
  module: Perform Sentiment Analysis and Opinion Mining using Azure Database for PostgreSQL
---

# センチメントを分析する

Margie's Travel のために構築している AI 搭載アプリの一部として、特定の賃貸物件に対する、個々のレビューのセンチメントと、すべてのレビューの全体的なセンチメントに関する情報をユーザーに提供したいと考えています。 これを達成するために、Azure Database for PostgreSQL フレキシブル サーバーの `azure_ai` 拡張機能を使用して、感情分析機能をデータベースに統合します。

## 開始する前に

管理者権限のある [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

### Azure サブスクリプションにリソースをデプロイする

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/11-portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用している場合は、*Bash* シェルに切り替えます。

3. Cloud Shell プロンプトで、次のように入力して、演習リソースを含んだ GitHub リポジトリをクローンします。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 次に、3 つのコマンドを実行して、Azure CLI コマンドを使用して Azure リソースを作成するときに冗長な入力を減らすための変数を定義します。 これらの変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースがデプロイされる Azure リージョン (`REGION`)、PostgreSQL 管理者ログイン用にランダムに生成されたパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドでは、対応する変数に割り当てられるリージョンは `eastus` ですが、任意の場所に置き換えることもできます。 ただし、既定値を置き換える場合は、このラーニング パスのモジュール内のすべてのタスクを完了できるように、[抽象要約をサポートしている別の Azure リージョン](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)を選択する必要があります。

    ```bash
    REGION=eastus
    ```

    次のコマンドでは、この演習で使用されるすべてのリソースを格納するリソース グループに使用する名前が割り当てられます。 対応する変数に割り当てられたリソース グループ名は `rg-learn-postgresql-ai-$REGION` で、その中の `$REGION` は上で指定した場所です。 ただし、この部分は好みに合わせて他のリソース グループ名に変更できます。

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    最後のコマンドでは、PostgreSQL 管理者ログイン用のパスワードがランダムに生成されます。 後で PostgreSQL フレキシブル サーバーに接続する際に使用できるように、**必ず安全な場所にコピーしてください**。

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

5. 複数の Azure サブスクリプションにアクセスでき、この演習のリソース グループやその他のリソースを作成するサブスクリプションが既定のサブスクリプションでない場合は、次のコマンドを実行して適切なサブスクリプションを設定し、`<subscriptionName|subscriptionId>` トークンを、使用するサブスクリプションの名前または ID に置き換えます。

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. 次の Azure CLI コマンドを実行して、リソース グループを作成します。

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. 最後に、Azure CLI を使用して Bicep デプロイ スクリプトを実行し、リソース グループに Azure リソースをプロビジョニングします。

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL フレキシブル サーバー、Azure OpenAI、Azure AI Language サービスなどです。 また、Bicep スクリプトでは、(azure.extensions サーバー パラメーターを使用して) PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する、サーバーに `rentals` という名前のデータベースを作成する、`text-embedding-ada-002` モデルを使用して `embedding` という名前のデプロイを Azure OpenAI サービスに追加するといった、いくつかの構成手順も実行されます。 なお、Bicep ファイルは、このラーニング パス内のすべてのモジュールで共有されるため、一部の演習ではデプロイされたリソースの一部のみを使用できます。

    デプロイが完了するまでに通常は数分かかります。 Cloud Shell から監視するか、上記で作成したリソース グループの **[デプロイ]** ページに移動し、そこでデプロイの進行状況を確認することができます。

8. リソースのデプロイが完了したら、Cloud Shell 画面を閉じます。

### デプロイ エラーのトラブルシューティング

Bicep デプロイ スクリプトを実行すると、いくつかエラーが発生する場合があります。 最も一般的なメッセージとその解決手順は次のとおりです。

- 以前にこのラーニング パスで Bicep デプロイ スクリプトを実行し、その後でリソースを削除した場合、リソースを削除してから 48 時間以内にスクリプトをまた実行しようとすると、次のようなエラー メッセージが表示される場合があります。

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

このタスクでは、[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して Azure Database for PostgreSQL サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL フレキシブル サーバー インスタンスに移動します。

2. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。 **[接続]** を選択しても、実際にデータベースには接続されないことに注意してください。さまざまな方法を使用してデータベースに接続する手順を示すだけです。 **ブラウザーまたはローカルから接続**する手順を確認し、それらの手順で、Azure Cloud Shell を使用して接続します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** のログイン用にランダムに生成されたパスワードを入力します。

    ログインすると、`rentals` データベースの `psql` プロンプトが表示されます。

## データベースにサンプル データを入力する

`azure_ai` 拡張機能を使用して賃貸物件レビューのセンチメントを分析するには、データベースにサンプル データを追加する必要があります。 `rentals` データベースにテーブルを追加し、顧客レビューを設定して、感情分析を実行するデータを用意します。

1. 次のコマンドを実行して、顧客から届いた物件レビューを格納するための `reviews` という名前のテーブルを作成します。

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int, 
        date date,
        comments text
    );
    ```

2. 次に、`COPY` コマンドを使用して、CSV ファイルのデータをテーブルに入力します。 次のコマンドを実行して、顧客レビューを `reviews` テーブルに読み込みます。

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

    コマンドの出力は `COPY 354` となるはずです。これは、CSV ファイルから 354 行がテーブルに書き込まれたことを示しています。

## `azure_ai` 拡張機能をインストールし構成する

`azure_ai` 拡張機能を使用する前に、それをデータベースにインストールし、Azure AI サービス リソースに接続するように構成する必要があります。 `azure_ai` 拡張機能を使用すると、Azure OpenAI サービスと Azure AI Language サービスをデータベースに統合できます。 データベースで拡張機能を有効にするには、次の手順に従います。

1. `psql` プロンプトで次のコマンドを実行して、環境のセットアップ時に実行した Bicep デプロイ スクリプトによって `azure_ai` および `vector` 拡張機能がサーバーの_許可リスト_に正常に追加されたことを確認します。

    ```sql
    SHOW azure.extensions;
    ```

    このコマンドは、サーバーの_許可リスト_に含まれている拡張機能の一覧を表示します。 すべてが正しくインストールされている場合、出力には次のように `azure_ai` と `vector` が含まれているはずです。

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

## Azure AI サービス アカウントを接続する

`azure_ai` 拡張機能の `azure_cognitive` スキーマに含まれる Azure AI サービス統合は、データベースから直接アクセスできる豊富な AI 言語機能のセットを提供します。 感情分析機能は、[Azure AI Language サービス](https://learn.microsoft.com/azure/ai-services/language-service/overview)を通じて利用できます。

1. `azure_ai` 拡張機能を使用して Azure AI Language サービスに対する呼び出しを正常に行うには、拡張機能のエンドポイントとキーを指定する必要があります。 Cloud Shell が開いているのと同じブラウザー タブを使用して、[Azure portal](https://portal.azure.com/) で言語サービス リソースに移動し、左側のナビゲーション メニューから **[リソース管理]** の下にある **[キーとエンドポイント]** 項目を選択します。

    ![KEY 1 とエンドポイントのコピー ボタンが赤い四角で強調表示された、Azure Language サービスの [キーとエンドポイント] ページのスクリーンショットが表示されます。](media/16-azure-language-service-keys-endpoints.png)

> [!Note]
>
> 上記の `azure_ai` 拡張機能のインストール時にメッセージ `NOTICE: extension "azure_ai" already exists, skipping CREATE EXTENSION` を受信し、以前に言語サービス エンドポイントとキーを使用して拡張機能を構成している場合は、`azure_ai.get_setting()` 関数を使用してこれらの設定が正しいことを確認し、正しい場合は手順 2 をスキップできます。

2. エンドポイントとアクセス キーの値をコピーし、以下のコマンドで、`{endpoint}` トークンと `{api-key}` トークンを、Azure portal からコピーした値に置換します。 Cloud Shell の `psql` コマンド プロンプトからコマンドを実行して、`azure_ai.settings` テーブルに値を追加します。

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

## 拡張機能の感情分析機能を確認する

このタスクでは、`azure_cognitive.analyze_sentiment()` 関数を使用して、賃貸物件のレビューを評価します。

1. この演習の残りの部分では、Cloud Shell でのみ作業するため、Cloud Shell 画面の右上にある **最大化** ボタンを選択して、ブラウザー画面内でウィンドウを拡大すると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/16-azure-cloud-shell-pane-maximize.png)

2. Cloud Shell で `psql` を使用する場合、クエリ結果の拡張表示を有効にすると、後続のコマンドの出力が読みやすくなるので、役に立つ場合があります。 次のコマンドを実行して、拡張表示を自動的に適用できるようにします。

    ```sql
    \x auto
    ```

3. `azure_ai` 拡張機能の感情分析機能は、`azure_cognitive` スキーマ内にあります。 `analyze_sentiment()` 関数を使用します。 [`\df` メタコマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)を使用して、以下を実行して関数を調べます。

    ```sql
    \df azure_cognitive.analyze_sentiment
    ```

    このメタコマンドの出力には、関数のスキーマ、名前、結果のデータ型、引数が表示されます。 この情報は、クエリから関数を操作する方法を理解するのに役立ちます。

    出力には、`analyze_sentiment()` 関数の 3 つのオーバーロードが示されており、それらの違いを確認できます。 出力の `Argument data types` プロパティにより、関数の 3 つのオーバーロードが想定する引数の一覧が表示されます。

    | 引数 | 型 | Default | 説明 |
    | -------- | ---- | ------- | ----------- |
    | text | `text` または `text[]` || センチメントを分析するテキスト。 |
    | language_text | `text` または `text[]` || センチメントを分析するテキストの言語を表す言語コード (または言語コードの配列)。 [サポートされている言語の一覧](https://learn.microsoft.com/azure/ai-services/language-service/sentiment-opinion-mining/language-support)を確認して、必要な言語コードを取得します。 |
    | batch_size | `integer` | 10 | `text[]` の入力を期待する 2 つのオーバーロード専用。 一度に処理するレコードの数を指定します。 |
    | disable_service_logs | `boolean` | false | サービス ログをオフにするかどうかを示すフラグ。 |
    | timeout_ms | `integer` | NULL | 操作が停止するまでのタイムアウト時間 (ミリ秒単位)。 |
    | throw_on_error | `boolean` | true | 関数がエラー時に例外をスローしてトランザクションをラップするロールバックを行うかどうかを示すフラグ。 |
    | max_attempts | `integer` | 1 | エラー発生時に Azure AI サービスへの呼び出しを再試行する回数。 |
    | retry_delay_ms | `integer` | 1000 | Azure AI サービス エンドポイントの呼び出しを再試行する前の待機時間 (ミリ秒単位)。 |

4. また、クエリの出力を正しく処理できるように、関数が返すデータ型の構造を理解しておくことも不可欠です。 次のコマンドを実行して、`sentiment_analysis_result` 型を調べます。

    ```sql
    \dT+ azure_cognitive.sentiment_analysis_result
    ```

5. 上記のコマンドの出力により、`sentiment_analysis_result` 型が `tuple` であることがわかります。 次のコマンドを実行して `sentiment_analysis_result` 型に含まれる列を確認することで、その `tuple` の構造をさらに詳しく調べることができます。

    ```sql
    \d+ azure_cognitive.sentiment_analysis_result
    ```

    そのコマンドの出力は次のようになります。

    ```sql
                     Composite type "azure_cognitive.sentiment_analysis_result"
         Column     |     Type         | Collation | Nullable | Default | Storage  | Description 
    ----------------+------------------+-----------+----------+---------+----------+-------------
     sentiment      | text             |           |          |         | extended | 
     positive_score | double precision |           |          |         | plain    | 
     neutral_score  | double precision |           |          |         | plain    | 
     negative_score | double precision |           |          |         | plain    |
    ```

    `azure_cognitive.sentiment_analysis_result` は、入力テキストのセンチメント予測を含んだ複合型です。 これには、肯定的、否定的、中立的、または混在のセンチメントと、テキストに含まれる肯定的、中立的、否定的な側面のスコアが含まれます。 スコアは、0 から 1 の間の実数として表されます。 たとえば、(neutral, 0.26, 0.64, 0.09) では、センチメントは中立的で、肯定的のスコアは 0.26、中立的は 0.64、否定的は 0.09 です。

## レビューのセンチメントを分析する

1. `analyze_sentiment()` 関数とその戻り値である `sentiment_analysis_result` を確認したので、次は、この関数を配置して使用しましょう。 次の簡単なクエリを実行します。このクエリでは、`reviews` テーブル内の少数のコメントに対して感情分析を実行します。

    ```sql
    SELECT
        id,
        azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment
    FROM reviews
    WHERE id <= 10
    ORDER BY id;
    ```

    分析された 2 つのレコードから、出力、`(mixed,0.71,0.09,0.2)`、および `(positive,0.99,0.01,0)` の `sentiment` 値をメモします。 これらは、上記のクエリで `analyze_sentiment()` 関数から返される `sentiment_analysis_result` を表します。 `reviews` テーブルの `comments` フィールドに対して分析が実行されました。

    > [!Note]
    >
    > `analyze_sentiment()` 関数をインラインで使用すると、クエリ内のテキストのセンチメントをすばやく分析できます。 これは少数のレコードに適していますが、多数のレコードのセンチメントを分析したり、数万件以上のレビューを含むテーブル内のすべてのレコードを更新したりするのには適していない場合があります。

2. レビューが長い場合に役立つもう 1 つのアプローチは、その中の各文のセンチメントを分析することです。 それには、テキストの配列を受け取る、`analyze_sentiment()` 関数のオーバーロードを使用します。

    ```sql
    SELECT
        azure_cognitive.analyze_sentiment(ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en') AS sentence_sentiments
    FROM reviews
    WHERE id = 1;
    ```

    上記のクエリでは、PostgreSQL の `STRING_TO_ARRAY` 関数を使用しました。 さらに、`ARRAY_REMOVE` 関数を使用して、空の文字列である配列要素を削除しました。これらが `analyze_sentiment()` 関数でエラーの原因となるからです。

    クエリからの出力により、レビュー全体に割り当てられた `mixed` センチメントをより深く理解できます。 その文には、肯定的、中立的、否定的なセンチメントが混在しています。

3. 前の 2 つのクエリは、クエリから直接 `sentiment_analysis_result` を返しました。 ただし、`sentiment_analysis_result``tuple` 内の基になる値の取得が必要になる可能性があります。 圧倒的に肯定的なレビューを検索し、センチメント コンポーネントを個々のフィールドに抽出する、次のクエリを実行します。

    ```sql
    WITH cte AS (
        SELECT id, comments, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    SELECT
        id,
        (sentiment).sentiment,
        (sentiment).positive_score,
        (sentiment).neutral_score,
        (sentiment).negative_score,
        comments
    FROM cte
    WHERE (sentiment).positive_score > 0.98
    LIMIT 5;
    ```

    上記のクエリでは、共通のテーブル式または CTE を使用して、`reviews` テーブル内のすべてのレコードのセンチメント スコアを取得します。 次に、CTE によって返される `sentiment_analysis_result` から `sentiment` 複合型の列を選択して、`tuple.` から個々の値を抽出します。

## レビュー テーブルにセンチメントを格納する

Margie's Travel のために構築している賃貸物件のレコメンデーション システムでは、センチメント評価をデータベースに格納して、センチメント評価が要求されるたびに呼び出しを行う必要がなく、そのためのコストが発生しないようにしようと考えています。 感情分析のオンザフライでの実行は、少数のレコードや、ほぼリアルタイムのデータ分析に適しています。 それでも、アプリケーションで使用するためにセンチメント データをデータベースに追加することは、格納されたレビューにとって意味があります。 これを行うには、`reviews` テーブルを変更して、センチメント評価と肯定的、中立的、否定的なスコアを格納するための列を追加します。

1. 次のクエリを実行して、センチメントの詳細を格納できるように `reviews` テーブルを更新します。

    ```sql
    ALTER TABLE reviews
    ADD COLUMN sentiment varchar(10),
    ADD COLUMN positive_score numeric,
    ADD COLUMN neutral_score numeric,
    ADD COLUMN negative_score numeric;
    ```

2. 次に、`reviews` テーブル内の既存のレコードに、センチメント値と関連スコアを反映させます。

    ```sql
    WITH cte AS (
        SELECT id, azure_cognitive.analyze_sentiment(comments, 'en') AS sentiment FROM reviews
    )
    UPDATE reviews AS r
    SET
        sentiment = (cte.sentiment).sentiment,
        positive_score = (cte.sentiment).positive_score,
        neutral_score = (cte.sentiment).neutral_score,
        negative_score = (cte.sentiment).negative_score
    FROM cte
    WHERE r.id = cte.id;
    ```

    テーブル内のすべてのレビューのコメントが分析のために言語サービスのエンドポイントに個別に送信されるので、このクエリの実行には時間がかかります。 多数のレコードを処理する場合は、レコードを一括送信する方が効率的です。

3. 次のクエリを実行して同じ更新アクションを実行しますが、今回は `reviews` テーブルからコメントを 10 個 (可能な最大のバッチ サイズ) ずつまとめて送信し、パフォーマンスの違いを評価します。

    ```sql
    WITH cte AS (
        SELECT azure_cognitive.analyze_sentiment(ARRAY(SELECT comments FROM reviews ORDER BY id), 'en', batch_size => 10) as sentiments
    ),
    sentiment_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            sentiments AS sentiment
        FROM cte
    )
    UPDATE reviews AS r
    SET
        sentiment = (sentiment_cte.sentiment).sentiment,
        positive_score = (sentiment_cte.sentiment).positive_score,
        neutral_score = (sentiment_cte.sentiment).neutral_score,
        negative_score = (sentiment_cte.sentiment).negative_score
    FROM sentiment_cte
    WHERE r.id = sentiment_cte.id;
    ```

    このクエリはもう少し複雑ですが、2 つの CTE を使用するとパフォーマンスが大幅に向上します。 このクエリでは、最初の CTE はレビュー コメントのバッチのセンチメントを分析し、2 つ目の CTE は結果の `sentiment_analysis_results` テーブルを、序数の位置と各行の "sentiment_analysis_result" に基づいて、`id` を含む新しいテーブルに抽出します。 そのあと、2 つ目の CTE を更新ステートメントで使用して、値をデータベースに書き込むことができます。

4. 次に、更新結果を確認するクエリを実行し、センチメントが**否定的な**レビューを、最も否定的なものから順に検索します。

    ```sql
    SELECT
        id,
        negative_score,
        comments
    FROM reviews
    WHERE sentiment = 'negative'
    ORDER BY negative_score DESC;
    ```

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/16-azure-portal-home-azure-services-resource-groups.png)

2. フィールド検索ボックスのフィルターに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/16-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。
