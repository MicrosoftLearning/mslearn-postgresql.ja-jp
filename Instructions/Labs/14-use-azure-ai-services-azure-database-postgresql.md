---
lab:
  title: Azure Database for PostgreSQL で Azure AI サービスを使用する
  module: Integrate AI Services to enrich your applications with intelligent features in Azure Database for PostgreSQL
---

# Azure Database for PostgreSQL で Azure AI サービスを使用する

この演習では、貸し別荘会社である **Margie's Travel** が、物件の登録情報とゲストのレビューをアプリケーションに保存して使用する方法を改善するのを支援します。  

この会社は、次のことを目指しています。  
- 物件に関する長い説明を短いハイライトにまとめる。  
- ゲストのレビューを分析して満足度を把握し、問題を明らかにする。  
- キー フレーズ、エンティティを抽出し、機密データを保護する。  
- 登録情報とレビューを翻訳して、国外からのゲストとホストがお互いを理解できるようにする。  

これらの機能をデータベースに直接実装するには、**Azure Database for PostgreSQL** の `azure_ai` 拡張機能を使用します。  

## 開始する前に

管理者権限を持つ [Azure サブスクリプション](https://azure.microsoft.com/free)が必要であり、そのサブスクリプションで **Azure Language** のアクセスが承認されている必要があります。 Azure Language のアクセスが必要な場合は、[Azure Language の制限付きアクセス](https://learn.microsoft.com/legal/cognitive-services/language-service/limited-access) ページで申請します。

### Azure サブスクリプションにリソースをデプロイする

"非運用環境の Azure Database for PostgreSQL サーバーと非運用環境の Azure Language リソースを既にセットアップしている場合は、このセクションをスキップできます。"**

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

1. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/14-portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用していた場合は、*Bash* シェルに切り替えます。

1. Cloud Shell プロンプトで、次のように入力して、演習リソースを含んだ GitHub リポジトリをクローンします。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. 次に、3 つのコマンドを実行して、Azure CLI コマンドを使用して Azure リソースを作成するときに冗長な入力を減らすための変数を定義します。 これらの変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースのデプロイ先となる Azure リージョン (`REGION`)、PostgreSQL 管理者のサインイン用にランダムに生成されるパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドで、対応する変数に割り当てられるリージョンは `westus3` ですが、任意の場所に置き換えることもできます。 ただし、既定値を置き換える場合は、この演習のすべてのタスクを完了できるように、[概要作成機能と翻訳機能をサポートする別の Azure リージョン](https://learn.microsoft.com/azure/ai-services/language-service/summarization/region-support)を選択する必要があります。

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
    # Core infra: PostgreSQL + DB + firewall + server param, Azure Language account
    az deployment group create \
      --resource-group "$RG_NAME" \
      --template-file "~/mslearn-postgresql/Allfiles/Labs/Shared/deploy-all.bicep" \
      --parameters adminLogin=pgAdmin adminLoginPassword="$ADMIN_PASSWORD" databaseName=rentals
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL サーバーと Azure Language サービスです。 また、Bicep スクリプトでは、PostgreSQL サーバーの "許可リスト" への `azure_ai` 拡張機能の追加 (サーバー パラメーター `azure.extensions` を使用)、サーバー上での `rentals` という名前のデータベースの作成など、いくつかの構成ステップも実行されます。**

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

[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL サーバーに移動します。

1. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。 **[接続]** を選択しても、実際にはデータベースに接続されません。これは、さまざまな方法を使用してデータベースに接続する手順を示しているだけです。 **ブラウザーからまたはローカルで接続する**手順を確認し、それらの手順で、Azure Cloud Shell を使用して接続します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/14-postgresql-database-connect.png)

1. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** サインイン用にランダムに生成されたパスワードを入力します。

    サインインすると、`rentals` データベースの `psql` プロンプトが表示されます。

1. この演習の残りの部分では、Cloud Shell で作業を続けます。そのため、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー ウィンドウ内でペインを拡大すると作業しやすくなります。

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
    -- Configure Azure Language
    SELECT azure_ai.set_setting('azure_cognitive.endpoint', 'https://<your-language-resource>.cognitiveservices.azure.com');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<your-language-key>');
    SELECT azure_ai.set_setting('azure_cognitive.region','<your-region>');  -- e.g., 'westus3'
    ```

## データベースにサンプル データを入力する

`azure_ai` 拡張機能を使用する前に、`rentals` データベースに 2 つのテーブルを追加し、サンプル データを事前設定して、アプリケーションの作成時に使用する情報を用意しておきます。

1. **rentals** プロンプトで、次のコマンドを実行して、賃貸物件情報とゲストのフィードバックを格納するための `listings` および `reviews` テーブルを作成します。

    ```sql
    -- Drop existing tables if they exist
    DROP TABLE IF EXISTS reviews CASCADE;
    DROP TABLE IF EXISTS listings CASCADE;

    -- Create table for property listings
    -- Matches listings.csv (id,name,description,property_type,room_type,price,weekly_price)
    CREATE TABLE listings (
    id             BIGINT PRIMARY KEY,
    name           TEXT NOT NULL,
    description    TEXT NOT NULL,
    property_type  TEXT NOT NULL,
    room_type      TEXT NOT NULL,
    price          NUMERIC(10,2),
    weekly_price   NUMERIC(10,2)
    );

    -- Create table for guest reviews
    -- Matches reviews.csv (id,listing_id,date,comments)
    CREATE TABLE reviews (
    id           BIGINT PRIMARY KEY,
    listing_id   BIGINT NOT NULL REFERENCES listings(id),
    date         DATE,
    comments     TEXT NOT NULL
    );
    ```

1. Azure Cloud Shell で、`COPY` コマンドを使用して、先ほど作成したテーブルに CSV ファイルからデータを読み込みます。

    - `listings.csv` ファイルから `listings` テーブルにデータを読み込みます
    ```sql
    \COPY listings (id, name, description, property_type, room_type, price, weekly_price) FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' WITH (FORMAT csv, HEADER);
    ```

    - `reviews.csv` ファイルから `reviews` テーブルにデータを読み込みます
    ```sql
    \COPY reviews (id, listing_id, date, comments) FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' WITH (FORMAT csv, HEADER);
    ```

    コマンド出力で、CSV ファイルから両方のテーブルに行が書き込まれたことが示されます。

1. 埋め込みベクター列を追加します。

    `text-embedding-ada-002` モデルは 1,536 次元を返すように構成されているので、ベクター列のサイズに使用します。

    ```sql
    ALTER TABLE listings ADD COLUMN listing_vector vector(1536);
    ```

1. `create_embeddings` ユーザー定義関数を使用して Azure OpenAI を呼び出して、各登録情報の説明の埋め込みベクトルを生成します。

    ```sql
    UPDATE listings
    SET listing_vector = azure_openai.create_embeddings('embedding', description, max_attempts => 5, retry_delay_ms => 500)
    WHERE listing_vector IS NULL;
    ```

> [!NOTE]    
> これらの埋め込みを追加するには、使用可能なクォータに応じて数分かかる場合があります。

## Azure AI サービスを使用してアプリケーションを強化する

Margie's Travel が、登録情報とレビューを保存するだけでなく、それ以上のことを行う必要がある段階に達したとしましょう。 ゲストはより短い説明を必要とし、ホストはフィードバックを大規模に理解したいと考え、会社は機密データを保護する必要があり、サービス全体が複数の言語で機能する必要があります。

いくつかの点について検討した後、チームは `azure_ai` 拡張機能を使用して、Azure Database for PostgreSQL から直接次の機能を実装することにしました。

- 物件に関する長い説明を短いハイライトにまとめる。
- ゲストのレビューを分析して満足度を把握し、問題を明らかにする。
- キー フレーズを抽出し、エンティティを認識し、機密データを保護する。
- 登録情報とレビューを翻訳して、国外からのゲストとホストがお互いを理解できるようにする。

SQL コマンドを使用してこれらの各機能を実装する方法を見てみましょう。

## タスク 1:物件の登録情報を要約する

物件の説明が長いと、ゲストは圧倒され、比較が難しくなる可能性があります。 Margie's Travel では、短いバージョンを生成し、各登録情報について、意味を失うことなく重要なポイントを強調することにしました。

1. まだ接続していない場合は、Azure Cloud Shell で `psql` を使用して、`rentals` データベースに接続します。

1. 次の SQL コマンドを実行して、物件の登録情報を要約するための SQL コマンドを生成します。 まず、抽出要約を使用して、元のテキストから主要な文を抜粋します。

  ```sql
  SELECT
    id,
    name,
    azure_cognitive.summarize_extractive(description, 'en', 2) AS extractive_summary
  FROM listings
  ORDER BY id
  LIMIT 3;
  ```

1. 一部の登録情報については、書き換えられた要約の方が、抜粋された文よりも明確になります。 チームは抽象的な要約を実行して、要点を捉えた簡潔なバージョンを作成します。

  ```sql
  SELECT
    id,
    name,
    azure_cognitive.summarize_abstractive(description, 'en', 2) AS abstractive_summary
  FROM listings
  ORDER BY id
  LIMIT 3;
  ```

> [!NOTE] 抽象的な要約は、すべての Azure リージョンでサポートされているわけではありません。 エラーが発生した場合は、この機能をサポートする別のリージョンを試してください。 詳細については、[Azure Language 機能のリージョン サポート](https://learn.microsoft.com/azure/ai-services/language-service/concepts/regional-support)に関する記事を参照してください。

## タスク 2:ゲストのセンチメントを分析する

毎月何千ものレビューが届きますが、会社はレビューを 1 つずつ読まなくても人々の感想を理解する必要があります。 各レビューには、全体的なセンチメント ラベル (肯定的、中立、または否定的) と、信頼度を示すスコアが与えられます。

1. まだ接続していない場合は、Azure Cloud Shell で `psql` を使用して、`rentals` データベースに接続します。

1. ゲストのセンチメントを分析するには、次の SQL コマンドを実行します。

  ```sql
  WITH s AS (
    SELECT id, comments,
          azure_cognitive.analyze_sentiment(comments, 'en') AS res
    FROM reviews
  )
  SELECT
    id,
    (res).sentiment            AS overall_sentiment,
    (res).positive_score,
    (res).neutral_score,
    (res).negative_score
  FROM s
  ORDER BY id
  LIMIT 10;
  ```

1. 問題に対処する必要がある場合、スタッフは、まず最も否定的なレビューに集中し、否定の度合順に対処することができます。 最も否定的なレビューを特定するには、次の SQL コマンドを実行します。

  ```sql
  WITH s AS (
    SELECT id, comments,
          azure_cognitive.analyze_sentiment(comments, 'en') AS res
    FROM reviews
  )
  SELECT
    id,
    (res).negative_score AS negativity,
    comments
  FROM s
  WHERE (res).sentiment = 'negative'
  ORDER BY (res).negative_score DESC
  LIMIT 10;
  ```

1. 次の SQL コマンドを実行して、文レベルでセンチメントを分析します。 これは、否定的な反応を引き起こす可能性のある特定のフレーズを識別するのに役立ちます。

  ```sql
  SELECT
    azure_cognitive.analyze_sentiment(
      ARRAY_REMOVE(STRING_TO_ARRAY(comments, '.'), ''), 'en'
    ) AS sentence_sentiments
  FROM reviews
  ORDER BY id
  LIMIT 1;
  ```

## タスク 3:分析情報を抽出し、機密情報を保護する

運用チームは、個人の詳細情報を一般的に使用されないようにしながら、データをより簡単に検索できるようにしたいと考えています。 説明から、タグやフィルターと同様に機能する短いフレーズを生成します。

1. まだ接続していない場合は、Azure Cloud Shell で `psql` を使用して、`rentals` データベースに接続します。

1. 次の SQL コマンドを実行して、物件の説明からキー フレーズを抽出します。

  ```sql
  SELECT
    id,
    name,
    unnest(azure_cognitive.extract_key_phrases(description)) AS key_phrase
  FROM listings
  ORDER BY id
  LIMIT 50;
  ```

1. 多くの場合、レビューには場所、日付、組織が記載されています。 これらの詳細を特定すると、物件や場所全体のパターンを分析しやすくなります。

  ```sql
  SELECT
    r.id AS review_id,
    (e).text       AS entity_text,
    (e).category   AS entity_category,
    (e).subcategory
  FROM (
    SELECT id, unnest(azure_cognitive.recognize_entities(comments, 'en-us')) AS e
    FROM reviews
  ) r
  ORDER BY review_id
  LIMIT 50;
  ```

1. 個人を特定できる情報 (PII) を保護することは重要です。 電話番号やメールなどの個人の詳細情報は、共有テキストに表示しないでください。 リダクションにより、これらの値が削除され、より安全なバージョンが残されます。

  ```sql
  SELECT
    id,
    (pii).redacted_text AS redacted_description
  FROM (
    SELECT id, azure_cognitive.recognize_pii_entities(description, 'en-us') AS pii
    FROM listings
  ) x
  ORDER BY id
  LIMIT 10;
  ```

1. 監査の場合、スタッフは削除された内容とその分類方法を確認できます。

  ```sql
  SELECT
    l.id,
    (ent).text      AS detected_value,
    (ent).category  AS pii_type
  FROM (
    SELECT id, unnest((azure_cognitive.recognize_pii_entities(description, 'en-us')).entities) AS ent
    FROM listings
  ) l
  ORDER BY l.id
  LIMIT 50;
  ```

## タスク 4:登録情報とレビューを翻訳する

ゲストとホストの出身国はさまざまであるため、言語の壁がないようにする必要があります。 登録情報とレビューを複数の言語で利用できるようにすると、プラットフォームを誰でも快適に使用することができます。


1. まだ接続していない場合は、Azure Cloud Shell で `psql` を使用して、`rentals` データベースに接続します。

1. 現在 Language リソースをポイントしているため、Translator リソースをポイントするようにエンドポイントとリージョンを更新する必要があります。 次のコマンドを実行して設定を更新します。

   ```sql
   SELECT azure_ai.set_setting('azure_cognitive.endpoint', 'https://<your-translator-resource>.cognitiveservices.azure.com');
    SELECT azure_ai.set_setting('azure_cognitive.subscription_key', '<your-translator-key>');
   SELECT azure_ai.set_setting('azure_cognitive.region','<your-region>');  -- e.g., 'westus3'
   ```

1. 物件の説明をスペイン語とフランス語に翻訳すると、旅行者は自分が選択した言語で説明を読むことができます。 次の SQL コマンドを実行して、物件の説明を複数の言語に翻訳します。


  ```sql
  SELECT
    l.id,
    l.name,
    t.target_language AS target_lang,
    t.text            AS translated_text
  FROM listings l
  CROSS JOIN LATERAL azure_cognitive.translate(
    l.description,
    ARRAY['es','fr'],
    'en'
  ) a
  CROSS JOIN LATERAL unnest(a.translations) AS t
  ORDER BY l.id, t.target_language
  LIMIT 6;
  ```

1. レビューも翻訳されます。 元の言語が自動的に検出され、一貫したレビューのためにテキストがフランス語で使用できるようになります。

  ```sql
  SELECT
    r.id,
    t.target_language AS target_lang,
    t.text            AS translated_text
  FROM reviews r
  CROSS JOIN LATERAL azure_cognitive.translate(
    r.comments,
    ARRAY['fr'],
    NULL
  ) a
  CROSS JOIN LATERAL unnest(a.translations) AS t
  ORDER BY r.id
  LIMIT 5;
  ```

Margie's Travel では、ゲストとホストの両方に対して、より堅牢でユーザー フレンドリなエクスペリエンスを提供できるようになりました。 Azure Database for PostgreSQL 内で Azure AI Services を直接使用することによって、同社のプラットフォームは、実際の課題に取り組むインテリジェントな機能で強化されました。

## 要点

この演習では、Azure Database for PostgreSQL で `azure_ai` 拡張機能を使用して、アプリケーションを強化するインテリジェントな機能を実装する方法について学習しました。 テキストの要約、センチメントの分析、キー フレーズとエンティティの抽出、機密情報の保護、複数の言語へのコンテンツの翻訳を行う方法を確認しました。 これらの機能により、アプリケーションはより優れたユーザー エクスペリエンスを提供し、アクセシビリティを向上し、世界中の対象ユーザーをサポートできます。