---
lab:
  title: EXPLAIN ステートメントを実行する
  module: Understand PostgreSQL query processing
---

# EXPLAIN ステートメントを実行する

この演習では、EXPLAIN 関数と、これを使用して指定されたステートメントに対して PostgreSQL プランナーが生成する実行プランを表示する方法を確認します。

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

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/03-portal-toolbar-cloud-shell.png)

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
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- 責任ある AI 契約に同意する必要があるためにスクリプトで AI リソースを作成できない場合は、次のエラーが発生する場合があります。その場合は、Azure portal ユーザー インターフェイスを使用して Azure AI サービス リソースを作成し、デプロイ スクリプトを再実行します。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## 続行する前に、次の点に注意してください。

次のことを確認してください。

1. Azure Database for PostgreSQL フレキシブル サーバーをインストールして起動しました。 これは、前の Bicep スクリプトによってインストールされているはずです。
1. [PostgreSQL ラボ](https://github.com/MicrosoftLearning/mslearn-postgresql.git) からラボ スクリプトを既に複製してあります。 まだ行っていない場合は、ローカルにリポジトリを複製してください。
    1. コマンド ライン/ターミナルを開きます。
    1. 次のコマンドを実行します。
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > 注
       > 
       > **Git** がインストールされていない場合は、[***Git*** アプリをダウンロードしてインストールし](https://git-scm.com/download)、前のコマンドをもう一度実行してみてください。
1. Azure Data Studio をインストールしました。 インストールしていない場合は、[***Azure Data Studio*** をダウンロードしてインストールしてください](https://go.microsoft.com/fwlink/?linkid=2282284)。
1. Azure Data Studio に **PostgreSQL** 拡張機能をインストールします。
1. Azure Data Studio を開き、Bicep スクリプトで作成した Azure Database for PostgreSQL フレキシブル サーバーに接続します。 ユーザー名 **pgAdmin** と、前に作成した**ランダム管理者パスワード**を入力します。
1. zoodb データベースをまだ作成していない場合は、**[ファイル]**、**[ファイルを開く]** の順に選択し、スクリプトを保存したフォルダーに移動します。 **../Allfiles/Labs/02/Lab2_ZooDb.sql** を選択して、**[開く]** を選択します。
   1. **DROP** および **CREATE** ステートメントを強調表示し、それらを実行します。
   1. 画面上部のドロップダウン矢印を使って、zoodb やシステム データベースなど、サーバー上のデータベースを表示します。 **zoodb** データベースを選択します。
   1. **[テーブルの作成]**、**[外部キーの作成]**、および **[テーブルの事前設定]** セクションを強調表示し、それらを実行します。
   1. スクリプトの最後にある **SELECT** ステートメント 3 つを強調表示してそれらを実行し、テーブルが作成および事前設定されたことを確認します。

## EXPLAIN ANALYZE を実践する

1. [Azure portal](https://portal.azure.com) で、Azure Database for PostgreSQL フレキシブル サーバーに移動します。 サーバーが起動されていることを確認するか、必要に応じて再起動します。
1. Azure Data Studio を開き、Azure Database for PostgreSQL フレキシブル サーバーに接続します。
1. **[ファイル]**、**[ファイルを開く]** の順に選択し、スクリプトを保存したフォルダーに移動します。 **../Allfiles/Labs/03/Lab3_RepopulateZoo.sql**を開きます。 必要に応じてサーバーに再接続します。
1. **[実行]** を選択して、クエリを実行します。 これにより、zoodb データベースが再作成されます。
1. [ファイル]、**[ファイルを開く]**、**../Allfiles/Labs/03/Lab3_explain.sql** の順に選択します。
1. ラボ ファイルのセクション「**1. EXPLAIN ANALYZE を調べる**」で、ステートメント A とステートメント B を個別に強調表示して実行します。
    1. データベースを更新したステートメントとその理由は何ですか?
    1. ステートメント A の計画には何ミリ秒かかりましたか?
    1. ステートメント B の実行時間はどれくらいでしたか?

## EXPLAIN を実践する

1. ラボ ファイルのセクション「**2. EXPLAIN を調べる**」で、そのステートメントを強調表示して実行します。
    1. 使用された並べ替えキーとその理由は何ですか?
1. ラボ ファイルのセクション「**3. EXPLAIN オプションを調べる**」で、各ステートメントを個別に強調表示して実行します。 各オプションのクエリ プラン統計を比較します。

## クリーンアップ

1. 不要な Azure コストが発生しないように、この演習で作成したリソース グループを削除します。
1. 必要に応じて、**.\DP3021Lab** フォルダーを削除します。

