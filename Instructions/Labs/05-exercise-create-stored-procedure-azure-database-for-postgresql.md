---
lab:
  title: Azure Database for PostgreSQL でストアド プロシージャを作成する
  module: Procedures and functions in PostgreSQL
---

# Azure Database for PostgreSQL でストアド プロシージャを作成する

この演習では、ストアド プロシージャを作成します。

## 開始する前に

この演習を完了するには、自分の Azure サブスクリプションが必要です。 Azure サブスクリプションをお持ちでない場合は、[Azure の無料試用版](https://azure.microsoft.com/free)を作成できます。

## 演習環境を作成する

この演習と以降のすべての演習では、Azure Cloud Shell で Bicep を使用して PostgreSQL サーバーをデプロイします。
既にインストールしている場合は、リソースのデプロイと Azure Data Studio のインストールをスキップしてください。

### Azure サブスクリプションにリソースをデプロイする

この手順では、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

> Note
>
> このラーニング パスで複数のモジュールを行っている場合は、それらの間で Azure 環境を共有できます。 その場合は、このリソース デプロイ手順を 1 回完了するだけで済みます。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/05-portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用している場合は、*Bash* シェルに切り替えます。

3. Cloud Shell プロンプトで、次のように入力して、演習リソースを含む GitHub リポジトリを複製します。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. 次に、Azure CLI コマンドを使用して Azure リソースを作成するときに、冗長な型指定を減らすための変数を定義する 3 つのコマンドを実行します。 この変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースがデプロイされる Azure リージョン (`REGION`)、PostgreSQL 管理者ログイン用にランダムに生成されたパスワード (`ADMIN_PASSWORD`) を表します。

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

## GitHub リポジトリをローカルにクローンする

[PostgreSQL Labs](https://github.com/MicrosoftLearning/mslearn-postgresql.git) からラボ スクリプトが既にクローンされていることを確認します。 まだ完了していない場合、リポジトリをローカルにクローンするには、次の手順を実行します。

1. コマンド ライン/ターミナルを開きます。
1. 次のコマンドを実行します。
    ```bash
    md .\DP3021Lab
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
    ```
    > 注
    > 
    > **Git** がインストールされていない場合は、[***Git*** アプリをダウンロードしてインストールし](https://git-scm.com/download)、前のコマンドをもう一度実行してみてください。

## Azure Data Studio をインストールする

Azure Data Studio がインストールされていない場合は、次の手順を実行します。

1. ブラウザーで「[Azure Data Studio のダウンロードとインストール](/sql/azure-data-studio/download-azure-data-studio)」に移動し、Windows プラットフォームで「**ユーザー インストーラー (推奨)**」を選びます。 実行可能ファイルがダウンロード フォルダーにダウンロードされます。
1. **[ファイルを開く]** を選びます。
1. 使用許諾契約書が表示されます。 **契約書を読んで同意**してから、**[次へ]** を選びます。
1. **[追加タスクの選択]** で、**[PATH に追加]** と、その他の必要な追加を選びます。 [**次へ**] を選択します。
1. **[インストールの準備完了]** ダイアログ ボックスが表示されます。 設定を確認します。 **[戻る]** を選んで変更を行うか、**[インストール]** を選びます。
1. **[Completing the Azure Data Studio Setup Wizard](Azure Data Studio セットアップ ウィザードの完了)** ダイアログ ボックスが表示されます。 **完了** を選択します。 Azure Data Studio が開始します。

## PostgreSQL 拡張機能をインストールする

Azure Data Studio に PostgreSQL 拡張機能がインストールされていない場合は、次の手順を実行します。

1. Azure Data Studio がまだ開いていない場合は、開きます。
1. 左側のメニューで **[拡張機能]** を選んで、[拡張機能] パネルを表示します。
1. 検索バーに「**PostgreSQL**」と入力します。 Azure Data Studio 用 PostgreSQL 拡張機能のアイコンが表示されます。
1. **[インストール]** を選択します。 拡張機能がインストールされます。

## Azure Database for PostgreSQL フレキシブル サーバーに接続する

1. Azure Data Studio がまだ開いていない場合は、開きます。
1. 左側のメニューから **[接続]** を選びます。
1. **[新しい接続]** を選びます。
1. **[接続の詳細]** の **[接続の種類]** で、ドロップダウン リストから **[PostgreSQL]** を選びます。
1. **[サーバー名]** に、Azure portal に表示される完全なサーバー名を入力します。
1. **[認証の種類]** は [パスワード] のままにします。
1. [ユーザー名] と [パスワード] に、ユーザー名 **pgAdmin** と上で作成した**ランダム管理者パスワード**を入力します。
1. [パスワードを記憶する] を選びます。
1. 残りのフィールドは省略可能です。
1. **[接続]** を選択します。 Azure Database for PostgreSQL サーバーに接続されます。
1. サーバー データベースの一覧が表示されます。 これには、システム データベースとユーザー データベースが含まれます。
1. zoodb データベースをまだ作成していない場合は、**[ファイル]**、**[ファイルを開く]** の順に選択し、スクリプトを保存したフォルダーに移動します。 **../Allfiles/Labs/02/Lab2_ZooDb.sql** を選択して、**[開く]** を選択します。
   1. **DROP** および **CREATE** ステートメントを強調表示し、それらを実行します。
   1. 画面上部のドロップダウン矢印を使って、zoodb やシステム データベースなど、サーバー上のデータベースを表示します。 **zoodb** データベースを選択します。
   1. **[テーブルの作成]**、**[外部キーの作成]**、および **[テーブルの事前設定]** セクションを強調表示し、それらを実行します。
   1. スクリプトの最後にある **SELECT** ステートメント 3 つを強調表示してそれらを実行し、テーブルが作成および事前設定されたことを確認します。

## repopulate_zoo() ストアド プロシージャを作成する

1. 画面の上部にあるドロップダウン矢印を使って、zoodb を現在のデータベースにします。
1. Azure Data Studio で、**[ファイル]**、**[ファイルを開く]** の順に選び、ラボ スクリプトに移動します。 **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql** を選択し、**[開く]** を選択します。 必要な場合は、サーバーに再接続します。
1. **[ストアド プロシージャの作成]** の下の **DROP PROCEDURE** から **END $$** までのセクションを強調表示します。 強調表示されたテキストを実行します。
1. Azure Data Studio でファイルを開いたままにして、次の演習の準備をします。

## new_exhibit() ストアド プロシージャを作成する

1. 画面の上部にあるドロップダウン矢印を使って、zoodb を現在のデータベースにします。
1. Azure Data Studio で、**[ファイル]**、**[ファイルを開く]** の順に選び、ラボ スクリプトに移動します。 **../Allfiles/Labs/05/Lab5_StoredProcedure.sql** を選択し、**[開く]** を選択します。 必要な場合は、サーバーに再接続します。
1. **[ストアド プロシージャの作成]** の下の **DROP PROCEDURE** から **END $$** までのセクションを強調表示します。 強調表示されたテキストを実行します。 手順を読みます。 入力パラメーターがいくつか宣言され、それらを使って enclosure テーブルと animal テーブルに行が挿入されていることがわかります。
1. Azure Data Studio でファイルを開いたままにして、次の演習の準備をします。

## ストアド プロシージャ

1. **Call the stored procedure (ストアド プロシージャを呼び出す)** の下のセクションを強調表示にします。 強調表示されたテキストを実行します。 これにより、入力パラメーターに値を渡すことでストアド プロシージャが呼び出されます。
1. 2 つの **SELECT** ステートメントを強調表示にして実行します。 強調表示されたテキストを実行します。 enclosure に新しい行が挿入され、5 つの新しい行が animal に挿入されたことがわかります。

## テーブル値関数を作成して呼び出す

1. Azure Data Studio で、**[ファイル]**、**[ファイルを開く]** の順に選び、ラボ スクリプトに移動します。 **../Allfiles/Labs/05/Lab5_Table_Function.sql** を選択し、**[開く]** を選択します。
1. 最初の **SELECT** ステートメントを強調表示にして実行し、zoodb データベースが選ばれていることを確認します。
1. **repopulate_zoo()** ストアド プロシージャを強調表示にして実行し、クリーン データから始めます。
1. **Create a table valued function (テーブル値関数を作成する)** の下のセクションを強調表示にして実行します。 この関数は、**enclosure_summary** という名前のテーブルを返します。 関数のコードを読み、テーブルの設定方法を理解します。
1. 2 つの SELECT ステートメントを強調表示にして実行し、毎回異なるエンクロージャ ID を渡します。
1. **How to use a table valued function with a LATERAL join (LATERAL 結合でテーブル値関数を使用する方法)** の下のセクションを強調表示にして実行します。 これは、結合でテーブル名の代わりに使われているテーブル値関数を示します。

## オプションの演習 - 組み込みの関数

1. Azure Data Studio で、**[ファイル]**、**[ファイルを開く]** の順に選び、ラボ スクリプトに移動します。 **../Allfiles/Labs/05/Lab5_SimpleFunctions.sql** を選択し、**[開く]** を選択します。
1. 各関数を強調表示にして実行して、その動作を確認します。 各関数について詳しくは、[オンライン ドキュメント](https://www.postgresql.org/docs/current/functions.html)をご覧ください。
1. スクリプトを保存せずに Azure Data Studio を閉じます。
1. サーバーを使っていないときに課金されないように、Azure Database for PostgreSQL サーバーを停止します。

## クリーンアップ

1. 不要な Azure コストが発生しないように、この演習で作成したリソース グループを削除します。
1. 必要に応じて、.\DP3021Lab フォルダーを削除します。

