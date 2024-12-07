---
lab:
  title: Azure AI 翻訳を使用してテキストを翻訳する
  module: Translate Text using the Azure AI Translator and Azure Database for PostgreSQL
---

# Azure AI 翻訳を使用してテキストを翻訳する

Margie's Travel のリード開発者として、国際化の取り組みを支援するように依頼されました。 現在、同社の短期レンタルサービスのすべてのレンタルリストは英語で提供されています。 これらの一覧を、大がかりな開発作業を行わずにさまざまな言語に翻訳したいと考えています。 すべてのデータは Azure Database for PostgreSQL フレキシブル サーバーでホストされており、Azure AI サービスを利用して翻訳を実行したいと考えています。

この演習では、Azure Database for PostgreSQL フレキシブル サーバー データベースを介して Azure AI 翻訳サービスを使用して、英語のテキストをさまざまな言語に翻訳します。

## 開始する前に

管理者権限のある [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

### Azure サブスクリプションにリソースをデプロイする

この手順では、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/11-portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用している場合は、*Bash* シェルに切り替えます。

3. Cloud Shell プロンプトで、次のように入力して、演習リソースを含む GitHub リポジトリを複製します。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

    前のモジュールでこの GitHub リポジトリを既に複製している場合は、引き続き使用できますが、次のエラー メッセージが表示される場合があります。

    ```bash
    fatal: destination path 'mslearn-postgresql' already exists and is not an empty directory.
    ```

    このメッセージが表示された場合は、問題なく次の手順に進むことができます。

4. 次に、Azure CLI コマンドを使用して Azure リソースを作成するときに、冗長な型指定を減らすための変数を定義する 3 つのコマンドを実行します。 これらの変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースがデプロイされる Azure リージョン (`REGION`)、PostgreSQL 管理者ログイン用にランダムに生成されたパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドでは、対応する変数に割り当てられるリージョンは `eastus` ですが、任意の場所に置き換えることもできます。 ただし、既定値を置き換える場合は、このラーニング パスのモジュール内のすべてのタスクを完了できるように、[抽象要約をサポートしている別の Azure リージョン](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)を選択する必要があります。

    ```bash
    REGION=eastus
    ```

    次のコマンドでは、この演習で使用されるすべてのリソースを格納するリソース グループに使用する名前が割り当てられます。 対応する変数に割り当てられたリソース グループ名は `rg-learn-postgresql-ai-$REGION` で、その中の `$REGION` は上で指定した場所です。 ただし、この部分は好みに合わせて他のリソース グループ名に変更できます。

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

    最後のコマンドでは、PostgreSQL 管理者ログイン用のパスワードがランダムに生成されます。 後で PostgreSQL フレキシブル サーバーに接続する際に使用できるように、必ず安全な場所にコピーしてください。

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-translate.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL フレキシブル サーバーと Azure AI 翻訳サービスなどです。 また、Bicep スクリプトでは、PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する (azure.extensions サーバー パラメーターを使用)、サーバーに `rentals` という名前のデータベースを作成するなど、いくつかの構成手順も実行されます。 **なお、Bicep ファイルはこのラーニング パス内の他のモジュールとは異なります。**

    デプロイが完了するまでに通常数分かかります。 Cloud Shell から監視するか、上記で作成したリソース グループの **[デプロイ]** ページに移動し、そこでデプロイの進行状況を確認することができます。

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

このタスクでは、[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して Azure Database for PostgreSQL サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL フレキシブル サーバーに移動します。

2. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** のログイン用にランダムに生成されたパスワードを入力します。

    ログインすると、`rentals` データベースの `psql` プロンプトが表示されます。

4. この演習の残りの部分は Cloud Shell で作業を続けるので、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー画面内にペインを広げると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/17-azure-cloud-shell-pane-maximize.png)

## データベースに一覧データを入力する

英語の一覧データを翻訳するには、それらを入手する必要があります。 前のモジュールで `rentals` データベースに `listings` テーブルを作成していない場合は、次の手順に従って作成します。

1. 次のコマンドを実行して、賃貸物件一覧データを格納するための `listings` テーブルを作成します。

    ```sql
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

2. 次に、`COPY` コマンドを使用して、上記で作成した各テーブルに CSV ファイルからデータを読み込みます。 まず、次のコマンドを実行して、`listings` テーブルにデータを入力します。

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

    コマンドの出力は `COPY 50` となるはずです。これは、CSV ファイルから 50 行がテーブルに書き込まれたことを示しています。

## 翻訳用に追加のテーブルを作成する

`listings` データは用意できましたが、翻訳を実行するには 2 つの追加テーブルが必要です。

1. 次のコマンドを実行して、`languages` テーブルと `listing_translations` テーブルを作成します。

    ```sql
    CREATE TABLE languages (
        code VARCHAR(7) NOT NULL PRIMARY KEY
    );
    ```

    ```sql
    CREATE TABLE listing_translations(
        id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
        listing_id INT,
        language_code VARCHAR(7),
        description TEXT
    );
    ```

2. 次に、翻訳対象の言語ごとに 1 行を挿入します。 この例では、ドイツ語、簡体字中国語、ヒンディー語、ハンガリー語、スワヒリ語の 5 つの言語の行を作成します。

    ```sql
    INSERT INTO languages(code)
    VALUES
        ('de'),
        ('zh-Hans'),
        ('hi'),
        ('hu'),
        ('sw');
    ```

    コマンドの出力は `INSERT 0 5` となるはずです。これは、新しく 5 行がテーブルに挿入されたことを示しています。

## `azure_ai` 拡張機能のインストールと構成

`azure_ai` 拡張機能を使用する前に、それをデータベースにインストールし、Azure AI サービス リソースに接続するように構成する必要があります。 `azure_ai` 拡張機能を使用すると、Azure OpenAI サービスと Azure AI Language サービスをデータベースに統合できます。 データベースで拡張機能を有効にするには、次の手順に従います。

1. `psql` プロンプトで次のコマンドを実行して、環境のセットアップ時に実行した Bicep デプロイ スクリプトによって `azure_ai` および `vector` 拡張機能がサーバーの_許可リスト_に正常に追加されたことを確認します。

    ```sql
    SHOW azure.extensions;
    ```

    このコマンドは、サーバーの_許可リスト_に含まれている拡張機能の一覧を表示します。 すべてが正しくインストールされている場合、出力には次のように `azure_ai` と `vector` が含まれるはずです。

    ```sql
     azure.extensions 
    ------------------
     azure_ai,vector
    ```

    Azure Database for PostgreSQL フレキシブル サーバーに拡張機能をインストールして使用するには、「[PostgreSQL 拡張機能の使用方法](https://learn.microsoft.com/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)」で説明されているように、サーバーの_許可リスト_に拡張機能を追加する必要があります。

2. これで、[CREATE EXTENSION](https://www.postgresql.org/docs/current/sql-createextension.html) コマンドを使用して `azure_ai` 拡張機能をインストールできるようになりました。

    ```sql
    CREATE EXTENSION IF NOT EXISTS azure_ai;
    ```

    `CREATE EXTENSION` は、スクリプト ファイルを実行して新しい拡張機能をデータベースに読み込みます。 このスクリプトは通常、関数、データ型、スキーマなどの新しい SQL オブジェクトを作成します。 同じ名前の拡張機能が既に存在する場合は、エラーがスローされます。 `IF NOT EXISTS` を追加すると、コマンドが既にインストールされている場合はエラーをスローせずに実行できます。

3. 次に、`azure_ai.set_setting()` 関数を使用して、Azure AI 翻訳サービスへの接続を構成する必要があります。 Cloud Shell が開いているのと同じブラウザー タブを使用して、Cloud Shell 画面を最小化または復元し、[Azure portal](https://portal.azure.com/) で Azure AI 翻訳リソースに移動します。 Azure AI 翻訳リソース ページに移動したら、リソース メニューの **[リソース管理]** セクションで **[キーとエンドポイント]** を選択し、使用可能なキー、リージョン、およびドキュメント翻訳エンドポイントのいずれか 1 つをコピーします。

    ![Azure AI 翻訳サービスの [キーとエンドポイント] ページのスクリーンショットが表示され、キー 1、リージョン、ドキュメント翻訳エンドポイントのコピー ボタンが赤い四角で強調表示されています。](media/18-azure-ai-translator-keys-and-endpoint.png)

    `KEY 1` または `KEY 2` を使用できます。 常に 2 つのキーを用意しておくと、サービスを中断させることなく、キーのローテーションと再生成を安全に行うことができます。

4. AI 翻訳エンドポイント、サブスクリプション キー、リージョンを指すように `azure_cognitive` 設定を構成します。 `azure_cognitive.endpoint` の値は、サービスのドキュメント翻訳 URL になります。 `azure_cognitive.subscription_key` の値は、キー 1 またはキー 2 になります。 `azure_cognitive.region` の値は、Azure AI 翻訳インスタンスのリージョンになります。

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint','https://<YOUR_ENDPOINT>.cognitiveservices.azure.com/');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<YOUR_KEY>');
    SELECT azure_ai.set_setting('azure_cognitive.region', '<YOUR_REGION>');
    ```

## 一覧データを翻訳するストアド プロシージャを作成する

言語翻訳テーブルにデータを事前設定するには、データをバッチで読み込むストアド プロシージャを作成します。

1. `psql` プロンプトで次のコマンドを実行して、`translate_listing_descriptions` という名前の新しいストアド プロシージャを作成します。

    ```sql
    CREATE OR REPLACE PROCEDURE translate_listing_descriptions(max_num_listings INT DEFAULT 10)
    LANGUAGE plpgsql
    AS $$
    BEGIN
        WITH batch_to_load(id, description) AS
        (
            SELECT id, description
            FROM listings l
            WHERE NOT EXISTS (SELECT * FROM listing_translations ll WHERE ll.listing_id = l.id)
            LIMIT max_num_listings
        )
        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT b.id, l.code, (unnest(tr.translations)).TEXT
        FROM batch_to_load b
            CROSS JOIN languages l
            CROSS JOIN LATERAL azure_cognitive.translate(b.description, l.code) tr;
    END;
    $$;
    ```

    このストアド プロシージャは、5 つのレコードを一括で読み込み、選択した各言語で説明を翻訳し、翻訳した説明を `listing_translations` テーブルに挿入します。

2. 次の SQL コマンドを使用して、このストアド プロシージャを実行します。

    ```sql
    CALL translate_listing_descriptions(10);
    ```

    この呼び出しでは、賃貸物件一覧ごとに 5 つの言語に翻訳するのに約 1 秒かかるので、それぞれの実行には約 10 秒かかります。 コマンド出力は `CALL` となり、これはストアド プロシージャの呼び出しが成功したことを示します。

3. このストアド プロシージャを 5 回呼び出した場合は、プロシージャをさらに 4 回呼び出します。 その結果、テーブル内の一覧ごとに翻訳が生成されます。

4. 次のスクリプトを実行すると、一覧の翻訳の数を取得できます。

    ```sql
    SELECT COUNT(*) FROM listing_translations;
    ```

    この呼び出しでは、各リストが 5 つの言語に翻訳されたことを示す値 250 が返されます。 `listing_translations` テーブルに対してクエリを実行して、データをさらに分析できます。

## 翻訳を含む新しい一覧を追加するプロシージャを作成する

既存の一覧を翻訳するためのストアド プロシージャがありますが、国際化計画では、入力時に新しい一覧を翻訳する必要もあります。 これを行うには、別のストアド プロシージャを作成します。

1. `psql` プロンプトで次のコマンドを実行して、`add_listing` という名前の新しいストアド プロシージャを作成します。

    ```sql
    CREATE OR REPLACE PROCEDURE add_listing(id INT, name VARCHAR(255), description TEXT)
    LANGUAGE plpgsql
    AS $$
    DECLARE
    listing_id INT;
    BEGIN
        INSERT INTO listings(id, name, description)
        VALUES(id, name, description);

        INSERT INTO listing_translations(listing_id, language_code, description)
        SELECT id, l.code, (unnest(tr.translations)).TEXT
        FROM languages l
            CROSS JOIN LATERAL azure_cognitive.translate(description, l.code) tr;
    END;
    $$;
    ```

    このストアド プロシージャは、`listings` テーブルに行を挿入します。 次に、`languages` テーブル内の各言語の説明を翻訳し、これらの翻訳を `listing_translations` テーブルに挿入します。

2. 次の SQL コマンドを使用してストアド プロシージャを実行します。

    ```sql
    CALL add_listing(51, 'A Beautiful Home', 'This is a beautiful home in a great location.');
    ```

    コマンド出力は `CALL` となり、これはストアド プロシージャの呼び出しが成功したことを示します。

3. 次のスクリプトを実行して、新しい一覧の翻訳を取得します。

    ```sql
    SELECT l.id, l.name, l.description, lt.language_code, lt.description AS translated_description
    FROM listing_translations lt
        INNER JOIN listings l ON lt.listing_id = l.id
    WHERE l.name = 'A Beautiful Home';
    ```

    この呼び出しでは、次のテーブルに示したような値を持つ 5 つの行が返されます。

    ```sql
     id  | listing_id | language_code |                    description                     
    -----+------------+---------------+------------------------------------------------------
     126 |          2 | de            | Dies ist ein schönes Haus in einer großartigen Lage.
     127 |          2 | zh-Hans       | 这是一个美丽的家，地理位置优越。
     128 |          2 | hi            | यह एक महान स्थान में एक सुंदर घर है।
     129 |          2 | hu            | Ez egy gyönyörű otthon egy nagyszerű helyen.
     130 |          2 | sw            | Hii ni nyumba nzuri katika eneo kubwa.
    ```

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/17-azure-portal-home-azure-services-resource-groups.png)

2. 任意のフィールドのフィルター検索ボックスに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/17-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。
