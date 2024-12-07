---
lab:
  title: Azure AI Language を使用して分析情報を抽出する
  module: Extract insights using the Azure AI Language service with Azure Database for PostgreSQL
---

# Azure AI Language サービスと Azure Database for PostgreSQL を使用して分析情報を抽出する

物件取扱会社は、最も人気のあるフレーズや場所など、市場の傾向を分析しようと考えていることを思い出してください。 また、そのチームは個人を特定できる情報 (PII) の保護を強化する予定です。 現在のデータは、Azure Database for PostgreSQL フレキシブル サーバーに保存されています。 プロジェクトの予算が少ないため、キーワードとタグの維持における先行コストと継続的なコストを最小限に抑えることが重要です。 開発者は、PII が受け取ることができるフォームの数に警戒しており、社内の正規表現マッチャーよりも、コスト効率が高い検証済みのソリューションを望んでいます。

あなたは、`azure_ai` 拡張機能を使用して、データベースを Azure AI Language サービスと統合することになります。 この拡張機能は、次のような複数の Azure Cognitive Service API にユーザー定義 SQL 関数 API を提供します。

- キー フレーズ抽出
- エンティティ認識
- PII 認識

このアプローチにより、データ サイエンス チームは物件の人気データをすばやくチェックして、市場の傾向を判断できます。 また、アプリケーション開発者に、アクセスを必要としない状況で提示する PII に関して安全なテキストも提供されます。 識別されたエンティティを格納すると、問い合わせや誤検知の PII 認識 (そうではないものを PII だと考える) の場合に、人間によるレビューが可能になります。

最終的には、抽出された分析情報を含む 4 つの新しい列が `listings` テーブルにできます。

- `key_phrases`
- `recognized_entities`
- `pii_safe_description`
- `pii_entities`

## 開始する前に

管理者権限のある [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

### Azure サブスクリプションにリソースをデプロイする

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/17-portal-toolbar-cloud-shell.png)

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

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL - フレキシブル サーバー、Azure OpenAI、Azure AI Language サービスなどです。 また、Bicep スクリプトでは、(`azure.extensions` サーバー パラメーターを使用して) PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する、サーバーに `rentals` という名前のデータベースを作成する、`text-embedding-ada-002` モデルを使用して `embedding` という名前のデプロイを Azure OpenAI Service に追加するといった、いくつかの構成手順も実行されます。 なお、Bicep ファイルは、このラーニング パス内のすべてのモジュールで共有されるため、一部の演習ではデプロイされたリソースの一部のみを使用できます。

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

このタスクでは、[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して Azure Database for PostgreSQL サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL - フレキシブル サーバーに移動します。

2. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** のログイン用にランダムに生成されたパスワードを入力します。

    ログインすると、`rentals` データベースの `psql` プロンプトが表示されます。

4. この演習の残りの部分は Cloud Shell で作業を続けるので、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー画面内にペインを広げると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/17-azure-cloud-shell-pane-maximize.png)

## セットアップ: 拡張機能を構成する

ベクトルを格納してそのクエリを実行し、埋め込みを生成するには、Azure Database for PostgreSQL フレキシブル サーバーの 2 つの拡張機能 (`vector` と `azure_ai`) を許可リストに追加して有効にする必要があります。

1. 両方の拡張機能を許可リストに追加するには、「[PostgreSQL 拡張機能の使用方法](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/concepts-extensions#how-to-use-postgresql-extensions)」で説明されている手順に従って、サーバー パラメーター `azure.extensions` に `vector` と `azure_ai` を追加します。

2. 次の SQL コマンドを実行して、`vector` 拡張機能を有効にします。 詳細な手順については、「[Azure Database for PostgreSQL - フレキシブル サーバーで `pgvector` を有効にして使用する方法](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-use-pgvector#enable-extension)」をご覧ください。

    ```sql
    CREATE EXTENSION vector;
    ```

3. `azure_ai` 拡張機能を有効にするには、次の SQL コマンドを実行します。 ***Azure OpenAI*** リソースのエンドポイントと API キーが必要になります。 詳細な手順については、「[`azure_ai` 拡張機能を有効にする](https://learn.microsoft.com/en-us/azure/postgresql/flexible-server/generative-ai-azure-overview#enable-the-azure_ai-extension)」をご覧ください。

    ```sql
    CREATE EXTENSION azure_ai;
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.endpoint', 'https://<endpoint>.openai.azure.com');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_openai.subscription_key', '<API Key>');
    ```

4. `azure_ai` 拡張機能を使用して ***Azure AI Language*** サービスに対する呼び出しを正常に行うには、そのエンドポイントとキーを拡張機能に提供する必要があります。 Cloud Shell が開いているのと同じブラウザー タブを使用して、[Azure portal](https://portal.azure.com/) で言語サービス リソースに移動し、左側のナビゲーション メニューから **[リソース管理]** の下にある **[キーとエンドポイント]** 項目を選択します。

5. エンドポイントとアクセス キーの値をコピーし、以下のコマンドで、`{endpoint}` トークンと `{api-key}` トークンを Azure portal からコピーした値に置換します。 Cloud Shell の `psql` コマンド プロンプトからコマンドを実行して、`azure_ai.settings` テーブルに値を追加します。

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', '{endpoint}');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '{api-key}');
    ```

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

サンプル データを再設定するには、`DROP TABLE listings`を実行し、この手順を繰り返します。

## キー フレーズ抽出

1. キー フレーズは、`pg_typeof` 関数によって判明する `text[]` として抽出されます。

    ```sql
    SELECT pg_typeof(azure_cognitive.extract_key_phrases('The food was delicious and the staff were wonderful.', 'en-us'));
    ```

    キーの結果を格納する列を作成します。

    ```sql
    ALTER TABLE listings ADD COLUMN key_phrases text[];
    ```

1. 列をバッチで事前設定します。 クォータによっては、`LIMIT` 値を調整することもできます。 *このコマンドは何度実行しても構いません*。この演習では、すべての行を事前設定する必要はありません。

    ```sql
    UPDATE listings
    SET key_phrases = azure_cognitive.extract_key_phrases(description)
    FROM (SELECT id FROM listings WHERE key_phrases IS NULL ORDER BY id LIMIT 100) subset
    WHERE listings.id = subset.id;
    ```

1. キー フレーズで物件のクエリを実行します。

    ```sql
    SELECT id, name FROM listings WHERE 'closet' = ANY(key_phrases);
    ```

    キー フレーズが設定されている物件に応じて、次のような結果が得られます。

    ```sql
       id    |                name                
    ---------+-------------------------------------
      931154 | Met Tower in Belltown! MT2
      931758 | Hottest Downtown Address, Pool! MT2
     1084046 | Near Pike Place & Space Needle! MT2
     1084084 | The Best of the Best, Seattle! MT2
    ```

## 名前付きエンティティの認識

1. エンティティは、`pg_typeof` 関数によって判明する `azure_cognitive.entity[]` として抽出されます。

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    キーの結果を格納する列を作成します。

    ```sql
    ALTER TABLE listings ADD COLUMN entities azure_cognitive.entity[];
    ```

2. 列をバッチで事前設定します。 このプロセスには数分かかることがあります。 クォータに応じて `LIMIT` 値を調整するか、部分的な結果でより速く返すことができます。 *このコマンドは何度実行しても構いません*。この演習では、すべての行を事前設定する必要はありません。

    ```sql
    UPDATE listings
    SET entities = azure_cognitive.recognize_entities(description, 'en-us')
    FROM (SELECT id FROM listings WHERE entities IS NULL ORDER BY id LIMIT 500) subset
    WHERE listings.id = subset.id;
    ```

3. すべての物件のエンティティに対してクエリを実行して、地下室がある物件を検索できるようになりました。

    ```sql
    SELECT id, name
    FROM listings, unnest(listings.entities) AS e
    WHERE e.text LIKE '%roof%deck%'
    LIMIT 10;
    ```

    この結果、次のようなものが返されます。

    ```sql
       id    |                name                
    ---------+-------------------------------------
      430610 | 3br/3ba. modern, roof deck, garage
      430610 | 3br/3ba. modern, roof deck, garage
     1214306 | Private Bed/bath in Home: green (A)
       74328 | Spacious Designer Condo
      938785 | Best Ocean Views By Pike Place! PA1
       23430 | 1 Bedroom Modern Water View Condo
      828298 | 2 Bedroom Sparkling City Oasis
      338043 | large modern unit & fab location
      872152 | Luxurious Local Lifestyle 2Bd/2+Bth
      116221 | Modern, Light-Filled Fremont Flat
    ```

## PII 認識

1. エンティティは、`pg_typeof` 関数によって判明する `azure_cognitive.pii_entity_recognition_result` として抽出されます。

    ```sql
    SELECT pg_typeof(azure_cognitive.recognize_pii_entities('For more information, see Cognitive Services Compliance and Privacy notes.', 'en-us'));
    ```

    この値は、次によって検証された、編集されたテキストと PII エンティティの配列を含む複合型です。

    ```sql
    \d azure_cognitive.pii_entity_recognition_result
    ```

    出力は次のとおりです。

    ```sql
         Composite type "azure_cognitive.pii_entity_recognition_result"
         Column    |           Type           | Collation | Nullable | Default 
    ---------------+--------------------------+-----------+----------+---------
     redacted_text | text                     |           |          | 
     entities      | azure_cognitive.entity[] |           |          | 
    ```

    編集されたテキストと、認識されたエンティティに対する別のテキストを格納する列を作成します。

    ```sql
    ALTER TABLE listings ADD COLUMN description_pii_safe text;
    ALTER TABLE listings ADD COLUMN pii_entities azure_cognitive.entity[];
    ```

2. 列をバッチで事前設定します。 このプロセスには数分かかることがあります。 クォータに応じて `LIMIT` 値を調整するか、部分的な結果でより速く返すことができます。 *このコマンドは何度実行しても構いません*。この演習では、すべての行を事前設定する必要はありません。

    ```sql
    UPDATE listings
    SET
        description_pii_safe = pii.redacted_text,
        pii_entities = pii.entities
    FROM (SELECT id, description FROM listings WHERE description_pii_safe IS NULL OR pii_entities IS NULL ORDER BY id LIMIT 100) subset,
    LATERAL azure_cognitive.recognize_pii_entities(subset.description, 'en-us') as pii
    WHERE listings.id = subset.id;
    ```

3. これで、PII が編集された可能性がある物件の説明を表示できるようになりました。

    ```sql
    SELECT description_pii_safe
    FROM listings
    WHERE description_pii_safe IS NOT NULL
    LIMIT 1;
    ```

    この結果、次のように表示されます。

    ```sql
    A lovely stone-tiled room with kitchenette. New full mattress futon bed. Fridge, microwave, kettle for coffee and tea. Separate entrance into book-lined mudroom. Large bathroom with Jacuzzi (shared occasionally with ***** to do laundry). Stone-tiled, radiant heated floor, 300 sq ft room with 3 large windows. The bed is queen-sized futon and has a full-sized mattress with topper. Bedside tables and reading lights on both sides. Also large leather couch with cushions. Kitchenette is off the side wing of the main room and has a microwave, and fridge, and an electric kettle for making coffee or tea. Kitchen table with two chairs to use for meals or as desk. Extra high-speed WiFi is also provided. Access to English Garden. The Ballard Neighborhood is a great place to visit: *10 minute walk to downtown Ballard with fabulous bars and restaurants, great ****** farmers market, nice three-screen cinema, and much more. *5 minute walk to the Ballard Locks, where ships enter and exit Puget Sound
    ```

4. たとえば、上と全く同じ物件を使用するなど、PII で認識されるエンティティを識別することもできます。

    ```sql
    SELECT entities
    FROM listings
    WHERE entities IS NOT NULL
    LIMIT 1;
    ```

    この結果、次のように表示されます。

    ```sql
                            pii_entities                        
    -------------------------------------------------------------
    {"(hosts,PersonType,\"\",0.93)","(Sunday,DateTime,Date,1)"}
    ```

## 作業を確認

抽出されたキー フレーズ、認識されたエンティティ、PII が事前設定されていることを確認しましょう。

1. キー フレーズの確認:

    ```sql
    SELECT COUNT(*) FROM listings WHERE key_phrases IS NOT NULL;
    ```

    実行したバッチの数に応じて、次のような内容が表示されるはずです。

    ```sql
    count 
    -------
     100
    ```

2. 認識されたエンティティの確認:

    ```sql
    SELECT COUNT(*) FROM listings WHERE entities IS NOT NULL;
    ```

    次のような結果が表示されます。

    ```sql
    count 
    -------
     500
    ```

3. 編集された PII の確認:

    ```sql
    SELECT COUNT(*) FROM listings WHERE description_pii_safe IS NOT NULL;
    ```

    100 個のバッチ 1 つを読み込んだ場合は、次のように表示されるはずです。

    ```sql
    count 
    -------
     100
    ```

    PII が検出された物件の数を確認できます。

    ```sql
    SELECT COUNT(*) FROM listings WHERE description != description_pii_safe;
    ```

    次のような結果が表示されます。

    ```sql
    count 
    -------
        87
    ```

4. 検出された PII エンティティを確認します。上記により、空の PII 配列抜きで 13 になるはずです。

    ```sql
    SELECT COUNT(*) FROM listings WHERE pii_entities IS NULL AND description_pii_safe IS NOT NULL;
    ```

    結果:

    ```sql
    count 
    -------
        13
    ```

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/17-azure-portal-home-azure-services-resource-groups.png)

2. フィールド検索ボックスのフィルターに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/17-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。
