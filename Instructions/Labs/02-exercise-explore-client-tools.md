---
lab:
  title: クライアント ツールを使用して PostgreSQL を探索する
  module: Understand client-server communication in PostgreSQL
---

# クライアント ツールを使用して PostgreSQL を探索する

この演習では、psql と Azure Data Studio をダウンロードしてインストールします。 お使いのマシンに Azure Data Studio が既にインストールされている場合は、「Azure Database for PostrgreSQL フレキシブル サーバーに接続する」に進むことができます。

## 開始する前に

この演習を完了するには、自分の Azure サブスクリプションが必要です。 Azure サブスクリプションをお持ちでない場合は、[Azure の無料試用版](https://azure.microsoft.com/free)を作成できます。

## 演習環境を作成する

この演習と以降のすべての演習では、Azure Cloud Shell で Bicep を使用して PostgreSQL サーバーをデプロイします。

### Azure サブスクリプションにリソースをデプロイする

この手順では、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

> Note
>
> このラーニング パスで複数のモジュールを行っている場合は、それらの間で Azure 環境を共有できます。 その場合は、このリソース デプロイ手順を 1 回完了するだけで済みます。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/02-portal-toolbar-cloud-shell.png)

3. メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用している場合は、*Bash* シェルに切り替えます。

4. Cloud Shell プロンプトで、次のように入力して、演習リソースを含む GitHub リポジトリを複製します。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

5. 次に、Azure CLI コマンドを使用して Azure リソースを作成するときに、冗長な型指定を減らすための変数を定義する 3 つのコマンドを実行します。 この変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースがデプロイされる Azure リージョン (`REGION`)、PostgreSQL 管理者ログイン用にランダムに生成されたパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドで、対応する変数に割り当てられるリージョンは `eastus` ですが、任意の場所に置き換えることもできます。

    ```bash
    REGION=eastus
    ```

    次のコマンドで、この演習で使用されるすべてのリソースを格納するリソース グループに使用する名前が割り当てられます。 対応する変数に割り当てられたリソース グループ名は `rg-learn-work-with-postgresql-$REGION` で、その中の `$REGION` は上で指定した場所です。 ただし、この部分は好みに合わせて他のリソース グループ名に変更できます。

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    最後のコマンドでは、PostgreSQL 管理者ログイン用のパスワードがランダムに生成されます。 後で PostgreSQL フレキシブル サーバーに接続するのに使用できるように、このパスワードを必ず安全な場所にコピーしてください。

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

6. 複数の Azure サブスクリプションにアクセスでき、既定のサブスクリプションがこの演習でリソース グループとその他のリソースを作成するものでない場合は、次のコマンドを実行して適切なサブスクリプションを設定し、`<subscriptionName|subscriptionId>` トークンを使用するサブスクリプションの名前または ID に置き換えます。

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

7. 次の Azure CLI コマンドを実行して、リソース グループを作成します。

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

8. 最後に、Azure CLI を使用して Bicep デプロイ スクリプトを実行し、リソース グループ内の Azure リソースをプロビジョニングします。

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL - フレキシブル サーバーです。 bicep スクリプトではデータベースも作成されます。このデータベースは、コマンド ラインでパラメーターとして構成できます。

    デプロイが完了するまでに通常数分かかります。 Cloud Shell から監視するか、上で作成したリソース グループの **[デプロイ]** ページに移動し、そこでデプロイの進行状況を確認することができます。

8. リソースのデプロイが完了したら、Cloud Shell 画面を閉じます。

### デプロイ エラーのトラブルシューティング

Bicep デプロイ スクリプトを実行すると、いくつかエラーが発生する場合があります。 最も一般的なメッセージとその解決手順は次のとおりです。

- 以前にこのラーニング パスで Bicep デプロイ スクリプトを実行してその後リソースを削除した場合、リソースを削除してから 48 時間以内にスクリプトをまた実行しようとすると、次のようなエラー メッセージが表示される場合があります。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    このメッセージが表示された場合は、`restore` パラメーターが `true` に設定されるように上記の `azure deployment group create` コマンドを変更し、再実行します。

- 選択したリージョンで特定のリソースのプロビジョニングが制限されている場合は、`REGION` 変数を別の場所に設定してコマンドを再実行し、リソース グループを作成して、Bicep デプロイ スクリプトを実行する必要があります。

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 責任ある AI 契約に同意する必要があるためにスクリプトで AI リソースを作成できない場合は、次のエラーが発生する場合があります。その場合は、Azure portal ユーザー インターフェイスを使用して Azure AI サービス リソースを作成し、デプロイ スクリプトを再実行します。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## PostgreSQL に接続するためのクライアント ツール

### psql を使用して Azure Database for PostgreSQL に接続する

psql をローカルにインストールするか、Azure portal から接続することができ、接続した場合は Cloud Shell が開き、管理者アカウントのパスワードの入力を求められます。

#### ローカル接続

1. [ここ](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893)から psql をインストールします。
    1. セットアップ ウィザードで **[コンポーネントの選択]** ダイアログ ボックスまで進んだら、**[コマンド ライン ツール]** を選択します。
    > Note
    >
    > **psql** が環境に既にインストールされているかどうかを確認するには、コマンド ライン/ターミナルを開き、コマンド ***psql*** を実行します。 "*psql: error: connection to server on socket...*" のようなメッセージが返された場合は、**psql** ツールが環境に既にインストールされており、再インストールする必要はないことを意味します。



1. コマンド ラインを入力します。
1. サーバーに接続するための構文は次のとおりです。

    ```sql
    psql --h <servername> --p <port> -U <username> <dbname>
    ```

1. コマンド プロンプトで、「**`--host=<servername>.postgres.database.azure.com`**」と入力します。ここで、`<servername>` は上で作成した Azure Database for PostgreSQL の名前です。
    1. サーバー名は、Azure portal の **[概要]**、または bicep スクリプトからの出力で確認できます。

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin postgres
    ```

    1. 上でコピーした管理者アカウントのパスワードの入力を求めるメッセージが表示されます。

1. プロンプトで空のデータベースを作成するには、次のように入力します。

    ```sql
    CREATE DATABASE mypgsqldb;
    ```

1. プロンプトで次のコマンドを実行し、新しく作成したデータベース **mypgsqldb** に接続を切り替えます。

    ```sql
    \c mypgsqldb
    ```

1. サーバーに接続し、データベースを作成したので、データベースでのテーブルの作成など、使い慣れた SQL クエリを実行できます。

    ```sql
    CREATE TABLE inventory (
        id serial PRIMARY KEY,
        name VARCHAR(50),
        quantity INTEGER
        );
    ```

1. テーブルにデータを読み込む

    ```sql
    INSERT INTO inventory (id, name, quantity) VALUES (1, 'banana', 150);
    INSERT INTO inventory (id, name, quantity) VALUES (2, 'orange', 154);
    ```

1. クエリを実行し、テーブル内のデータを更新する

    ```sql
    SELECT * FROM inventory;
    ```

1. テーブル内のデータを更新します。

    ```sql
    UPDATE inventory SET quantity = 200 WHERE name = 'banana';
    ```

## Azure Data Studio をインストールする

> Note
>
> Azure Data Studio が既にインストールされている場合は、「* PostgreSQL 拡張機能をインストールする*」の手順に進みます。

Azure Database for PostgreSQL で使うために Azure Data Studio をインストールするには:

1. ブラウザーで「[Azure Data Studio のダウンロードとインストール](https://go.microsoft.com/fwlink/?linkid=2282284)」に移動し、Windows プラットフォームで「**ユーザー インストーラー (推奨)**」を選びます。 実行可能ファイルがダウンロード フォルダーにダウンロードされます。
1. **[ファイルを開く]** を選びます。
1. 使用許諾契約書が表示されます。 **契約書を読んで同意**してから、**[次へ]** を選びます。
1. **[追加タスクの選択]** で、**[PATH に追加]** と、その他の必要な追加を選びます。 [**次へ**] を選択します。
1. **[インストールの準備完了]** ダイアログ ボックスが表示されます。 設定を確認します。 **[戻る]** を選んで変更を行うか、**[インストール]** を選びます。
1. **[Completing the Azure Data Studio Setup Wizard](Azure Data Studio セットアップ ウィザードの完了)** ダイアログ ボックスが表示されます。 **完了** を選択します。 Azure Data Studio が開始します。

## PostgreSQL 拡張機能をインストールする

1. Azure Data Studio がまだ開いていない場合は、開きます。
2. 左側のメニューで **[拡張機能]** を選んで、[拡張機能] パネルを表示します。
3. 検索バーに「**PostgreSQL**」と入力します。 Azure Data Studio 用 PostgreSQL 拡張機能のアイコンが表示されます。
   
![Azure Data Studio 用 PostgreSQL 拡張機能のスクリーンショット。](media/02-postgresql-extension.png)
   
4. **[インストール]** を選択します。 拡張機能がインストールされます。

## Azure Database for PostgreSQL フレキシブル サーバーに接続する

1. Azure Data Studio がまだ開いていない場合は、開きます。
2. 左側のメニューから **[接続]** を選びます。
   
![Azure Data Studio の [接続] を示すスクリーンショット。](media/02-connections.png)

3. **新規接続** を選択します。
   
![Azure Data Studio で新しい接続を作成しているスクリーンショット。](media/02-create-connection.png)

4. **[接続の詳細]** の **[接続の種類]** で、ドロップダウン リストから **[PostgreSQL]** を選びます。
5. **[サーバー名]** に、Azure portal に表示される完全なサーバー名を入力します。
6. **[認証の種類]** は [パスワード] のままにします。
7. [ユーザー名] と [パスワード] に、ユーザー名 **pgAdmin** と上で作成した**ランダム管理者パスワード**を入力します。
8. [パスワードを記憶する] を選びます。
9. 残りのフィールドは省略可能です。
10. **[接続]** を選択します。 Azure Database for PostgreSQL サーバーに接続されます。
11. サーバー データベースの一覧が表示されます。 これには、システム データベースとユーザー データベースが含まれます。

## zoo データベースを作成する

1. 演習スクリプト ファイルがあるフォルダーに移動するか、[MSLearn PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/02) から **Lab2_ZooDb.sql** をダウンロードします。
1. Azure Data Studio がまだ開いていない場合は、開きます。
1. **[ファイル]**、**[ファイルを開く]** の順に選択し、スクリプトを保存したフォルダーに移動します。 **../Allfiles/Labs/02/Lab2_ZooDb.sql** を選択して、**[開く]** を選択します。
   1. **DROP** および **CREATE** ステートメントを強調表示し、それらを実行します。
   1. 画面上部のドロップダウン矢印を使って、zoodb やシステム データベースなど、サーバー上のデータベースを表示します。 **zoodb** データベースを選択します。
   1. **[テーブルの作成]**、**[外部キーの作成]**、および **[テーブルの事前設定]** セクションを強調表示し、それらを実行します。
   1. スクリプトの最後にある **SELECT** ステートメント 3 つを強調表示してそれらを実行し、テーブルが作成および事前設定されたことを確認します。
