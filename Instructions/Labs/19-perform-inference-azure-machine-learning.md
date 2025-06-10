---
lab:
  title: Azure Machine Learning を使用して推論を実行する
  module: Use Azure Machine Learning Inferencing with Azure Database for PostgreSQL
---

# Azure Machine Learning を使用して推論を実行する

Margie's Travel (MT) のリード開発者として、短期賃貸の夜間賃貸料金を見積もる機能の開発を支援するよう依頼されました。 いくつかの履歴データをテキスト ファイルとして収集しており、これを使用して Azure Machine Learning でシンプルな回帰モデルをトレーニングしたいと考えています。 そして、Azure Database for PostgreSQL フレキシブル サーバー データベースでホストしているデータに対して、そのモデルを使用したいと考えています。

この演習では、Azure Machine Learning の自動機械学習機能を使用して作成されたモデルをデプロイします。 そして、そのデプロイしたモデルを使用して、短期賃貸物件の夜間販売価格を見積もります。

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
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-azure-machine-learning.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL フレキシブル サーバーや Azure Machine Learning ワークスペースなどです。 デプロイ スクリプトでは、Azure Blob Storage アカウント、Azure Key Vault、Azure コンテナー リポジトリ、Azure Log Analytics ワークスペース、Azure Application Insights のインスタンスなど、Azure Machine Learning ワークスペースをインスタンス化するための前提条件となるサービスもすべて作成されます。 また、Bicep スクリプトでは、PostgreSQL サーバーの_許可リスト_に `azure_ai` 拡張機能や `vector` 拡張機能を追加する (azure.extensions サーバー パラメーターを使用)、サーバーに `rentals` という名前のデータベースを作成するなど、いくつかの構成手順も実行されます。 **なお、Bicep ファイルはこのラーニング パス内の他のモジュールとは異なります。**

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

## Azure Machine Learning モデルをデプロイする

最初のステップは、Azure Machine Learning へのモデルのデプロイです。 リポジトリには、PostgreSQL 統合で使用する、リストアップされたデータのセットでトレーニングされたモデルの例が含まれています。

1. [mslearn-postgresql リポジトリ](../../Allfiles/Labs/Shared/mlflow-model.zip)から `mlflow-model.zip` ファイルをダウンロードします。 ここからファイルを **mlflow-model** というフォルダーに抽出します。

2. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Machine Learning ワークスペースに移動します。

3. **[スタジオの起動]** ボタンを選択して、Azure Machine Learning スタジオを開きます。

    ![[スタジオの起動] ボタンが赤い四角で強調表示された Azure Machine Learning のスクリーンショット。](media/19-aml-launch-studio.png)

4. **[アセット]** メニューから **[モデル]** メニュー オプションを選択します。 そして、**[+ 登録]** メニュー オプションを選択し、**[ローカル ファイルから]** を選択します。

    ![[モデル一覧] ページのスクリーンショット。 [モデル] メニュー オプション、[登録] ドロップダウン ボタン、および [ローカル ファイルから] オプションが赤い四角で囲まれています。](media/19-aml-register-from-local-files.png)

5. **[モデルのアップロード]** メニューで、モデルの種類を **[MLflow]** に設定します。 次に、**[参照]** を選択して **mlflow-model** フォルダーに移動し、アセットをアップロードします。 その後、**[次へ]** ボタンを選択して次に進みます。

    ![[モデルのアップロード] メニュー ページのスクリーンショット。 MLflow というモデルの種類、[参照] ボタン、および [次へ] ボタンが赤い四角で囲まれています。](media/19-aml-register-upload-model.png)

6. モデルの名前を「**RentalListings**と設定し、**[次へ]** ボタンを選択します。

    !["名前" フィールドに RentalListings という値が入力されている [モデル設定] 画面のスクリーンショット。 [名前] テキスト ボックスと [次へ] ボタンが強調表示の赤い四角で囲まれています。](media/19-aml-register-model-settings.png)

7. **[登録]** ボタンを選択して、モデル登録を完了します。 このアクションにより、**[モデル]** ページに戻ります。 新しく作成したモデルを選択します。

> [!Note]
>
> モデルが表示されない場合は、**[最新の情報に更新]** メニュー オプション ボタンを選択してページを再読み込みします。 すると、**RentalListings** モデルが表示されます。

8. **[デプロイ]** ボタン オプションを選択して、新しい**リアルタイム エンドポイント**を作成します。

    ![[リアルタイム エンドポイント] メニュー オプションが赤い四角で強調表示されているスクリーンショット。](media/19-aml-automl-deploy-rte.png)

9. デプロイのポップアップ メニューで、**[仮想マシン]** を **[Standard_DS2_v2]** などに、**[インスタンス数]** を 1 に設定します。 **[デプロイ]** ボタンを選択します。 デプロイ プロセスには仮想マシンのプロビジョニングと Docker コンテナーとしてのモデルのデプロイが含まれるので、デプロイが完了するまでに数分かかる場合があります。

    ![デプロイのポップアップ メニューのスクリーンショット。 [仮想マシン] が [Standard_DS2_v2]、[インスタンス数] が 1 です。 [仮想マシン] ドロップダウン、[インスタンス数] テキストボックス、および [デプロイ] ボタンが赤い四角で強調表示されています。](media/19-aml-automl-deploy-endpoint.png)

10. エンドポイントがデプロイされたら、**[使用]** タブに移動し、次のセクションで使用できるように、REST エンドポイントと主キーをコピーします。

    ![エンドポイントの [使用] タブのスクリーンショット。REST エンドポイントと主認証キーのコピー ボタンが赤い四角で強調表示されています。](media/19-aml-automl-endpoint-consume.png)

11. エンドポイントが正常に実行されていることをテストするには、エンドポイントの **[テスト]** タブを使用できます。 そして、現在存在するすべての入力を置き換て、次のブロックに貼り付けます。 **[テスト]** ボタンを選択すると、この特定の物件で 1 泊分の賃貸として稼げると見込まれる米ドルの数値を示す単一の 10 進値の配列が入った JSON 出力が表示されるはずです。

    ```json
    {
        "input_data": {
            "columns": [
                "host_is_superhost",
                "host_has_profile_pic",
                "host_identity_verified",
                "neighbourhood_group_cleansed",
                "zipcode",
                "property_type",
                "room_type",
                "accommodates",
                "bathrooms",
                "bedrooms",
                "beds"
            ],
            "index": [0],
            "data": [["0", "0", "0", "Central Area", "98122", "House", "Entire home/apt", 4, 1.5, 3, 3]]
        }
    }
    ```

    ![エンドポイントの [テスト] タブのスクリーンショット。[入力] ボックスにはサンプル呼び出しが入っており、[jsonOutput] ボックスには推定値が入っています。 [テスト] ボタンが赤い四角で強調表示されています。](media/19-aml-automl-endpoint-test.png)

## Azure Cloud Shell で psql を使用してデータベースに接続する

このタスクでは、[Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) の [psql コマンドライン ユーティリティ](https://www.postgresql.org/docs/current/app-psql.html)を使用して、Azure Database for PostgreSQL フレキシブル サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、新しく作成した Azure Database for PostgreSQL フレキシブル サーバーに移動します。

2. リソース メニューの **[設定]** で、**[データベース]** を選択し、`rentals` データベースの **[接続]** を選択します。 **[接続]** を選択しても、実際にデータベースには接続されないことに注意してください。さまざまな方法を使用してデータベースに接続する手順を示すだけです。 **ブラウザーまたはローカルから接続**する手順を確認し、それらの手順で、Azure Cloud Shell を使用して接続します。

    ![Azure Database for PostgreSQL の [データベース] ページのスクリーンショット。 [データベース] と rentals データベースの [接続] が赤い四角で強調表示されています。](media/17-postgresql-rentals-database-connect.png)

3. Cloud Shell の "ユーザー pgAdmin のパスワード" というプロンプトで、**pgAdmin** のログイン用にランダムに生成されたパスワードを入力します。

    ログインすると、`rentals` データベースの `psql` プロンプトが表示されます。

4. この演習の残りの部分は Cloud Shell で作業を続けるので、ペインの右上にある **[最大化]** ボタンを選択して、ブラウザー画面内にペインを広げると便利な場合があります。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](media/17-azure-cloud-shell-pane-maximize.png)

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

3. 次に、`azure_ai.set_setting()` 関数を使用して、Azure Machine Learning でデプロイされたエンドポイントへの接続を構成する必要があります。 デプロイされたエンドポイントとそのキーを指すように `azure_ml` 設定を構成します。 `azure_ml.scoring_endpoint` の値はエンドポイントの REST URL になります。 `azure_ml.endpoint_key` の値は、キー 1 またはキー 2 の値になります。

    ```sql
    SELECT azure_ai.set_setting('azure_ml.scoring_endpoint','https://<YOUR_ENDPOINT>.<YOUR_REGION>.inference.ml.azure.com/score');
    ```

    ```sql
    SELECT azure_ai.set_setting('azure_ml.endpoint_key', '<YOUR_KEY>');
    ```

## 価格設定する一覧を含むテーブルを作成する

価格設定したい短期賃貸の一覧を格納するには、1 つのテーブルが必要です。

1. `rentals` データベースで次のコマンドを実行して、新しい `listings_to_price` テーブルを作成します。

    ```sql
    CREATE TABLE listings_to_price (
        id INT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
        host_is_superhost INT NOT NULL,
        host_has_profile_pic INT NOT NULL,
        host_identity_verified INT NOT NULL,
        neighbourhood_group_cleansed VARCHAR(75) NOT NULL,
        zipcode VARCHAR(5) NOT NULL,
        property_type VARCHAR(30) NOT NULL,
        room_type VARCHAR(30) NOT NULL,
        accommodates INT NOT NULL,
        bathrooms DECIMAL(3,1) NOT NULL,
        bedrooms INT NOT NULL,
        beds INT NOT NULL
    );
    ```

2. 次に、`rentals` データベースで次のコマンドを実行して、賃貸登録情報データを挿入します。

    ```sql
    INSERT INTO listings_to_price(host_is_superhost, host_has_profile_pic, host_identity_verified,
        neighbourhood_group_cleansed, zipcode, property_type, room_type,
        accommodates, bathrooms, bedrooms, beds)
    VALUES
        (1, 1, 1, 'Queen Anne', '98119', 'House', 'Private room', 2, 1.0, 1, 1),
        (0, 1, 1, 'University District', '98105', 'Apartment', 'Entire home/apt', 4, 1.5, 2, 2),
        (0, 0, 0, 'Central Area', '98122', 'House', 'Entire home/apt', 4, 1.5, 3, 3),
        (0, 0, 0, 'Downtown', '98101', 'House', 'Entire home/apt', 4, 1.5, 3, 3),
        (0, 0, 0, 'Capitol Hill', '98122', 'House', 'Entire home/apt', 4, 1.5, 3, 3);
    ```

    このコマンドは、5 行の新しい一覧データを挿入します。

## 一覧データを翻訳する関数を作成する

言語翻訳テーブルにデータを事前設定するには、データをバッチで読み込むストアド プロシージャを作成します。

1. `psql` プロンプトで次のコマンドを実行して、`price_listing` という名前の新しい関数を作成します。

    ```sql
    CREATE OR REPLACE FUNCTION price_listing (
        IN host_is_superhost INT, IN host_has_profile_pic INT, IN host_identity_verified INT,
        IN neighbourhood_group_cleansed VARCHAR(75), IN zipcode VARCHAR(5), IN property_type VARCHAR(30),
        IN room_type VARCHAR(30), IN accommodates INT, IN bathrooms DECIMAL(3,1), IN bedrooms INT, IN beds INT)
    RETURNS DECIMAL(6,2)
    AS $$
        SELECT CAST(jsonb_array_elements(inference.inference) AS DECIMAL(6,2)) AS expected_price
        FROM azure_ml.inference(('
        {
            "input_data": {
                "columns": [
                    "host_is_superhost",
                    "host_has_profile_pic",
                    "host_identity_verified",
                    "neighbourhood_group_cleansed",
                    "zipcode",
                    "property_type",
                    "room_type",
                    "accommodates",
                    "bathrooms",
                    "bedrooms",
                    "beds"
                ],
                "index": [0],
                "data": [["' || host_is_superhost || '", "' || host_has_profile_pic || '", "' || host_identity_verified || '", "' ||
                neighbourhood_group_cleansed || '", "' || zipcode || '", "' || property_type || '", "' || room_type || '", ' ||
                accommodates || ', ' || bathrooms || ', ' || bedrooms || ', ' || beds || ']]
            }
        }')::jsonb, deployment_name=>'rentallistings-1');
    $$ LANGUAGE sql;
    ```

> [!Note]
>
> デフォルトでは、デプロイ名はモデル名 (**rentallistings**) とバージョン番号 (**1**) の組み合わせです。 新しいバージョンのモデルをデプロイし、デフォルトのデプロイ名を使用する場合、新しいデプロイ名は **rentallistings-2** になります。

2. 次の SQL コマンドを使用して関数を実行します。

    ```sql
    SELECT * FROM price_listing(0, 0, 0, 'Central Area', '98122', 'House', 'Entire home/apt', 4, 1.5, 3, 3);
    ```

    このクエリは、1 泊あたりの賃貸料金の見積もりを 10 進形式で返します。

3. 次の SQL コマンドを使用して、`listings_to_price` テーブルの各行に対して関数を呼び出します。

    ```sql
    SELECT l2p.*, expected_price
    FROM listings_to_price l2p
        CROSS JOIN LATERAL price_listing(l2p.host_is_superhost, l2p.host_has_profile_pic, l2p.host_identity_verified,
            l2p.neighbourhood_group_cleansed, l2p.zipcode, l2p.property_type, l2p.room_type,
            l2p.accommodates, l2p.bathrooms, l2p.bedrooms, l2p.beds) expected_price;
    ```

    このクエリは、`listings_to_price` の行ごとに 1 つずつ、合計 5 行を返します。 これには、`listings_to_price` テーブルのすべての列と、`price_listing()` 関数の結果が `expected_price` として含まれます。

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/17-azure-portal-home-azure-services-resource-groups.png)

2. 任意のフィールドのフィルター検索ボックスに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/17-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。
