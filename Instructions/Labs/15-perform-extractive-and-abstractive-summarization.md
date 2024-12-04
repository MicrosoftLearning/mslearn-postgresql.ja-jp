---
lab:
  title: 抽出要約と抽象要約を実行する
  module: Summarize data using Azure AI Services and Azure Database for PostgreSQL
---

# 抽出要約と抽象要約を実行する

Margie's Travel が管理する賃貸物件アプリには、プロパティ マネージャーが賃貸物件の説明を入力する機能があります。 システム内の説明は長くなりがちで、賃貸物件、周辺地域、地元の観光スポット、店舗、その他のアメニティに関する多くの詳細情報を含んでいます。 アプリに AI を搭載した新機能を実装するにあたり、生成 AI を使用してこれらの説明の簡潔な要約を作成し、ユーザーが物件をすばやく簡単に確認できるようにする機能の導入を要求されました。 この演習では、Azure Database For PostgreSQL フレキシブル サーバーの `azure_ai` 拡張機能を使用して、賃貸物件の説明に対して抽象要約および抽出要約を実行し、結果の要約を比較します。

## 開始する前に

管理者権限のある [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

### Azure サブスクリプションにリソースをデプロイする

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/15-portal-toolbar-cloud-shell.png)

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

Bicep デプロイ スクリプトを実行すると、いくつかエラーが発生する場合があります。

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

このタスクでは、[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL フレキシブル サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL フレキシブル サーバー インスタンスに移動します。

2. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/15-postgresql-rentals-database-connect.png)

3. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** のログイン用にランダムに生成されたパスワードを入力します。

    ログインすると、`rentals` データベースの `psql` プロンプトが表示されます。

4. この演習の残りの部分は Cloud Shell で作業を続けるので、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー画面内にペインを広げると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/15-azure-cloud-shell-pane-maximize.png)

## データベースにサンプル データを入力する

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

`azure_ai` 拡張機能の `azure_cognitive` スキーマに含まれる Azure AI サービス統合は、データベースから直接アクセスできる豊富な AI 言語機能のセットを提供します。 テキスト要約機能は、[Azure AI Language サービス](https://learn.microsoft.com/azure/ai-services/language-service/overview)を通じて利用できます。

1. `azure_ai` 拡張機能を使用して Azure AI Language サービスに対する呼び出しを正常に行うには、拡張機能のエンドポイントとキーを指定する必要があります。 Cloud Shell が開いているのと同じブラウザー タブを使用して、[Azure portal](https://portal.azure.com/) で言語サービス リソースに移動し、左側のナビゲーション メニューから **[リソース管理]** の下にある **[キーとエンドポイント]** 項目を選択します。

    ![KEY 1 とエンドポイントのコピー ボタンが赤い四角で強調表示された、Azure Language サービスの [キーとエンドポイント] ページのスクリーンショットが表示されます。](media/15-azure-language-service-keys-endpoints.png)

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

## 拡張機能の要約機能を確認する

このタスクでは、`azure_cognitive` スキーマの 2 つの要約関数を確認します。

1. この演習の残りの部分では、Cloud Shell でのみ作業するため、Cloud Shell 画面の右上にある **最大化** ボタンを選択して、ブラウザー画面内でウィンドウを拡大すると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/15-azure-cloud-shell-pane-maximize.png)

2. Cloud Shell で `psql` を使用する場合、クエリ結果の拡張表示を有効にすると、後続のコマンドの出力が読みやすくなるので、役に立つ場合があります。 次のコマンドを実行して、拡張表示を自動的に適用できるようにします。

    ```sql
    \x auto
    ```

3. `azure_ai` 拡張機能のテキスト要約関数は、`azure_cognitive` スキーマ内にあります。 抽出要約には、`summarize_extractive()` 関数を使用します。 [`\df` メタコマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)を使用して、次を実行して関数を調べます。

    ```sql
    \df azure_cognitive.summarize_extractive
    ```

    このメタコマンドの出力には、関数のスキーマ、名前、結果のデータ型、引数が表示されます。 この情報は、クエリから関数を操作する方法を理解するのに役立ちます。

    出力には、`summarize_extractive()` 関数の 3 つのオーバーロードが示されており、それらの違いを確認できます。 出力の `Argument data types` プロパティにより、関数の 3 つのオーバーロードが想定する引数の一覧が表示されます。

    | 引数 | 型 | Default | 説明 |
    | -------- | ---- | ------- | ----------- |
    | text | `text` または `text[]` || 要約を生成するテキスト。 |
    | language_text | `text` または `text[]` || 要約するテキストの言語を表す言語コード (または言語コードの配列)。 [サポートされている言語の一覧](https://learn.microsoft.com/azure/ai-services/language-service/summarization/language-support)を確認して、必要な言語コードを取得します。 |
    | sentence_count | `integer` | 3 | 生成する要約文の数。 |
    | sort_by | `text` | 'offset' | 生成された要約文の並べ替え順序。 指定できる値は "offset" と "rank" で、"offset" は元のコンテンツ内の抽出された各文の開始位置を表し、"rank" は文がコンテンツの主要な概念にどれほど関連しているかを示す AI 生成インジケーターになります。 |
    | batch_size | `integer` | 25 | `text[]` の入力を期待する 2 つのオーバーロード専用。 一度に処理するレコードの数を指定します。 |
    | disable_service_logs | `boolean` | false | サービス ログをオフにするかどうかを示すフラグ。 |
    | timeout_ms | `integer` | NULL | 操作が停止するまでのタイムアウト時間 (ミリ秒単位)。 |
    | throw_on_error | `boolean` | true | 関数がエラー時に例外をスローしてトランザクションをラップするロールバックを行うかどうかを示すフラグ。 |
    | max_attempts | `integer` | 1 | エラー発生時に Azure AI サービスへの呼び出しを再試行する回数。 |
    | retry_delay_ms | `integer` | 1000 | Azure AI サービス エンドポイントの呼び出しを再試行する前の待機時間 (ミリ秒単位)。 |

4. 上記の手順を繰り返しますが、今回は `azure_cognitive.summarize_abstractive()` 関数の [`\df` メタコマンド](https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMAND-DF-LC)を実行し、出力を確認します。

    2 つの関数のシグネチャは似ていますが、`summarize_abstractive()` には `sort_by` パラメーターがなく、`text` の配列が戻り値になります。これに対して、`summarize_extractive()` 関数では、`azure_cognitive.sentence` 複合型の配列が返されます。 この不一致は、要約の 2 つの異なる生成方法に関係があります。 抽出要約では、要約対象のテキスト内で最も重要な文を特定し、それらをランク付けしたあと、要約として返します。 一方、抽象要約では、生成 AI を使用して、テキストの重要なポイントを要約した新しいオリジナルの文を作成します。

5. また、クエリの出力を正しく処理できるように、関数が返すデータ型の構造を理解しておくことも不可欠です。 `summarize_extractive()` 関数で返される `azure_cognitive.sentence` 型を調べるには、次のコマンドを実行します。

    ```sql
    \dT+ azure_cognitive.sentence
    ```

6. 上記のコマンドの出力により、`sentence` の型が `tuple` であることがわかります。 その `tuple` の構造を調べ、`sentence` 複合型に含まれている列を確認するには、以下を実行します。

    ```sql
    \d+ azure_cognitive.sentence
    ```

    そのコマンドの出力は次のようになるはずです。

    ```sql
                            Composite type "azure_cognitive.sentence"
        Column  |     Type         | Collation | Nullable | Default | Storage  | Description 
    ------------+------------------+-----------+----------+---------+----------+-------------
     text       | text             |           |           |        | extended | 
     rank_score | double precision |           |           |        | plain    |
    ```

    `azure_cognitive.sentence` は、抽出文のテキストと各文のランク スコアを含んだ複合型です。ランク スコアは、テキストのメイン トピックに対する文の関連度を示します。 ドキュメント要約では、抽出された文がランク付けされるので、それらの文が出現順に返されているのか、ランクに従って返されているのかを判断できます。

## 物件の説明の要約を作成する

このタスクでは、`summarize_extractive()` 関数と `summarize_abstractive()` 関数を使用して、物件の説明に対する 2 文から成る簡潔な要約を作成します。

1. `summarize_extractive()` 関数とその戻り値である `sentiment_analysis_result` を確認したので、この関数を使用しましょう。 次の簡単なクエリを実行します。このクエリでは、`reviews` テーブル内の少数のコメントに対して感情分析を実行します。

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    出力の `extractive_summary` フィールドに含まれている 2 文を元の `description` と比較します。なお、この文は元の文ではなく、`description` から抽出されたものです。 各文の後に記載されている数値は、言語サービスによって割り当てられたランク スコアです。

2. 次に、同じレコードに対して抽象要約を実行します。

    ```sql
    SELECT
        id,
        name,
        description,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    拡張機能の抽象要約機能を使用すると、元のテキストの全体的な意図をカプセル化した独自の自然言語要約を得られます。

    次のようなエラーが表示された場合は、Azure 環境の作成時に抽象要約をサポートしていないリージョンが選択されています。

    ```bash
    ERROR: azure_cognitive.summarize_abstractive: InvalidRequest: Invalid Request.

    InvalidParameterValue: Job task: 'AbstractiveSummarization-task' failed with validation errors: ['Invalid Request.']

    InvalidRequest: Job task: 'AbstractiveSummarization-task' failed with validation error: Document abstractive summarization is not supported in the region Central US. The supported regions are North Europe, East US, West US, UK South, Southeast Asia.
    ```

    この手順を実行し、抽象要約を使用して残りのタスクを完了するには、エラー メッセージで指定されているサポート対象リージョンのいずれかに新しい Azure AI Language サービスを作成する必要があります。 このサービスは、他のラボ リソースに使用したのと同じリソース グループにプロビジョニングできます。 または、残りのタスクを抽出要約に置き換えることもできますが、2 つの異なる要約手法の出力を比較できるという利点は得られません。

3. 最終的なクエリを実行して、2 つの要約手法を並べて比較します。

    ```sql
    SELECT
        id,
        azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary,
        azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
    FROM listings
    WHERE id IN (1, 2);
    ```

    生成された要約を並べて配置することで、各手法で生成された要約の品質を簡単に比較できます。 Margie's Travel のアプリケーションでは、抽象要約がより適切なオプションであり、高品質な情報を自然かつ読みやすい文章でまとめた簡潔な要約が提供されます。 抽出要約でもある程度の詳細は提供されますが、抽象要約で作成された元のコンテンツよりもまとまりがなく、価値が高いとは言えません。

## データベースに説明の要約を格納する

1. 次のクエリを実行して `listings` テーブルを変更し、新しい `summary` 列を追加します。

    ```sql
    ALTER TABLE listings
    ADD COLUMN summary text;
    ```

2. 生成 AI を使用してデータベース内の既存のすべてのプロパティの概要を作成するには、説明をバッチで送信し、言語サービスで複数のレコードを同時に処理できるようにするのが最も効率的です。

    ```sql
    WITH batch_cte AS (
        SELECT azure_cognitive.summarize_abstractive(ARRAY(SELECT description FROM listings ORDER BY id), 'en', batch_size => 25) AS summary
    ),
    summary_cte AS (
        SELECT
            ROW_NUMBER() OVER () AS id,
            ARRAY_TO_STRING(summary, ',') AS summary
        FROM batch_cte
    )
    UPDATE listings AS l
    SET summary = s.summary
    FROM summary_cte AS s
    WHERE l.id = s.id;
    ```

    update ステートメントでは、`listings` テーブルを要約で更新する前に、2 つの共通テーブル式 (CTE) を使用してデータを処理します。 最初の CTE (`batch_cte`) は、`listings` テーブルのすべての `description` 値を言語サービスに送信して、抽象要約を生成します。 これは、一度に 25 件のレコードのバッチで実行されます。 2 番目の CTE (`summary_cte`) は、`summarize_abstractive()` 関数によって返された要約の序数位置を使用して、`listings` テーブルの `description` の元のレコードに対応する `id` を各要約に割り当てます。 また、`ARRAY_TO_STRING` 関数を使用し、生成された要約をテキスト配列 (`text[]`) の戻り値からプルして、単純な文字列に変換します。 最後に、`UPDATE` ステートメントは、関連付けられている一覧の `listings` テーブルに要約を書き込みます。

3. 最後のステップとして、クエリを実行し、`listings` テーブルに書き込まれた要約を表示します。

    ```sql
    SELECT
        id,
        name,
        description,
        summary
    FROM listings
    LIMIT 5;
    ```

## 一覧のレビューの AI 分析の概要を生成する

Margie's Travel のアプリでは、宿泊施設のすべてのレビューの概要を表示することによって、ユーザーがレビューの全体的な概要をすばやく把握できます。

1. 次のクエリを実行します。このクエリは、一覧のすべてのレビューを 1 つの文字列に結合し、その文字列に対して抽象要約を生成します。

    ```sql
    SELECT unnest(azure_cognitive.summarize_abstractive(reviews_combined, 'en')) AS review_summary
    FROM (
        -- Combine all reviews for a listing
        SELECT string_agg(comments, ' ') AS reviews_combined
        FROM reviews
        WHERE listing_id = 1
    );
    ```

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/15-azure-portal-home-azure-services-resource-groups.png)

2. フィールド検索ボックスのフィルターに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/15-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。
