---
lab:
  title: システム カタログとビューを使用して、システム パラメーターを構成し、メタデータを探索する
  module: Configure and manage Azure Database for PostgreSQL
---

# システム カタログとビューを使用して、システム パラメーターを構成し、メタデータを探索する

この演習では、PostgreSQL のシステム パラメーターとメタデータを確認します。

## 開始する前に

> [!IMPORTANT]
> このモジュールの演習を完了するには、自分の Azure サブスクリプションが必要です。 Azure サブスクリプションをお持ちでない場合は、「[Azure の無料アカウントを使ってクラウドで構築](https://azure.microsoft.com/free/)」で無料試用版のアカウントを設定できます。

## 演習環境を作成する

### Azure サブスクリプションにリソースをデプロイする

この手順では、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

> Note
>
> このラーニング パスで複数のモジュールを行っている場合は、それらの間で Azure 環境を共有できます。 その場合は、このリソース デプロイ手順を 1 回完了するだけで済みます。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](media/07-portal-toolbar-cloud-shell.png)

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

### Azure Data Studio でデータベースに接続する

1. まだ行っていない場合は、[PostgreSQL ラボ](https://github.com/MicrosoftLearning/mslearn-postgresql.git) GitHub リポジトリからラボ スクリプトをローカルに複製します。
    1. コマンド ライン/ターミナルを開きます。
    1. 次のコマンドを実行します。
       ```bash
       md .\DP3021Lab
       git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git .\DP3021Lab
       ```
       > 注
       > 
       > **Git** がインストールされていない場合は、[***Git*** アプリをダウンロードしてインストールし](https://git-scm.com/download)、前のコマンドをもう一度実行してみてください。
1. Azure Data Studio をまだインストールしていない場合は、[Azure Data Studio をダウンロードしてインストールします******](https://go.microsoft.com/fwlink/?linkid=2282284)。
1. Azure Data Studio に **PostgreSQL** 拡張機能をインストールしていない場合は、ここでインストールします。
1. Azure Data Studio を開きます。
1. **接続**を選択します。
1. **[サーバー]** を選択し、**[新しい接続]** を選択します。
1. **[接続の種類]** で、**[PostgreSQL]** を選択します。
1. **[サーバー名]** に、サーバーのデプロイ時に指定した値を入力します。
1. **[ユーザー名]** に「**pgAdmin**」と入力します。
1. **[パスワード]** に、生成した **pgAdmin** ログインのランダムに生成されたパスワードを入力します。
1. **[パスワードを記録する]** を選択します。
1. **[接続]**
1. zoodb データベースをまだ作成していない場合は、**[ファイル]**、**[ファイルを開く]** の順に選択し、スクリプトを保存したフォルダーに移動します。 **../Allfiles/Labs/02/Lab2_ZooDb.sql** を選択して、**[開く]** を選択します。
   1. **DROP** および **CREATE** ステートメントを強調表示し、それらを実行します。
   1. 画面上部のドロップダウン矢印を使って、zoodb やシステム データベースなど、サーバー上のデータベースを表示します。 **zoodb** データベースを選択します。
   1. **[テーブルの作成]**、**[外部キーの作成]**、および **[テーブルの事前設定]** セクションを強調表示し、それらを実行します。
   1. スクリプトの最後にある **SELECT** ステートメント 3 つを強調表示してそれらを実行し、テーブルが作成および事前設定されたことを確認します。

## タスク 1: PostgreSQL のバキューム プロセスを探索する

1. 開いていない場合は、Azure Data Studio を開きます。
1. Azure Data Studio で、**[ファイル]**、**[ファイルを開く]** の順に選び、ラボ スクリプトに移動します。 **../Allfiles/Labs/07/Lab7_vacuum.sql** を選択し、**[開く]** を選択します。 必要な場合は、サーバーに再接続します。
1. データベース ドロップダウンから **zoodb** データベースを選択します。
1. **[Check zoodb database is selected](zoodb データベースが選択されていることを確認する)** セクションを強調表示にして実行します。 必要な場合は、ドロップダウン リストを使って zoodb を現在のデータベースにします。
1. **[Display dead tuples](無効タプルを表示する)** セクションを強調表示にして実行します。 このクエリを実行すると、データベース内の無効タプルと有効タプルの数が表示されます。 無効タプルの数を記録します。
1. 「**重みを変更する**」セクションを強調表示にして連続で 10 回実行します。 このクエリは、すべての動物の重み列を更新します。
1. **[Display dead tuples](無効タプルを表示する)** セクションをもう一度実行します。 更新が完了した後で、無効タプルの数を記録しておきます。
1. **[Manually run VACUUM](VACUUM を手動で実行する)** セクションを実行して、バキューム プロセスを実行します。
1. **[Display dead tuples](無効タプルを表示する)** セクションをもう一度実行します。 バキューム プロセスが実行された後で、無効タプルの数を記録します。

## タスク 2: 自動バキューム サーバー パラメーターを構成する

1. Azure portal で、自分の Azure Database for PostgreSQL フレキシブル サーバーに移動します。
1. **[設定]** で、**[サーバー パラメーター]** を選択します。
1. 検索バーに「**`vacuum`**」と入力します。 次のパラメーターを見つけて、次のように値を変更します。
    1. autovacuum = ON (既定で ON になっているはずです)
    1. autovacuum_vacuum_scale_factor = 0.1
    1. autovacuum_vacuum_threshold = 50

    このように設定すると、テーブルの 10% に削除対象としてマークされた行がある場合、またはいずれか 1 つのテーブルで 50 行が更新または削除された場合に、自動バキューム プロセスが実行されます。

1. **[保存]** を選択します。 サーバーが再起動されます。

## タスク 3: Azure portal で PostgreSQL メタデータを表示する

1. [Azure portal](https://portal.azure.com) に移動してサインインします。
1. 「**Azure Database for PostgreSQL**」を検索して選択します。
1. この演習用に作成した Azure Database for PostgreSQL フレキシブル サーバーを選択します。
1. **[監視]** で **[メトリック]** を選択します。
1. **[メトリック]** を選択し、**[CPU percent] (CPU 使用率)** を選択します。
1. データベースに関するさまざまなメトリックを表示できることがわかります。

## タスク 4: システム カタログ テーブルのデータを表示する

1. Azure Data Studio に切り替えます。
1. **[SERVERS]** で PostgreSQL サーバーを選択し、接続が確立されてサーバーの上に緑色の丸が表示されるまで待ちます。
1. サーバーを右クリックし、**[新しいクエリ]** を選択します。
1. 次の SQL を入力し、**[実行]** を選択します。

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. 各データベースのコミットとロールバックを表示できることがわかります。

## システム ビューを使用して複雑なメタデータ クエリを表示する

1. サーバーを右クリックし、**[新しいクエリ]** を選択します。
1. 次の SQL を入力し、**[実行]** を選択します。

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. 大量の統計情報を表示できることがわかります。
1. システム ビューを使うと、記述する必要がある SQL の複雑さを軽減できます。 **pg_stats** ビューを使わなかった場合、前のクエリには次のコードが必要になります。

    ```sql
    SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    a.attname,
    s.stainherit AS inherited,
    s.stanullfrac AS null_frac,
    s.stawidth AS avg_width,
    s.stadistinct AS n_distinct,
        CASE
            WHEN s.stakind1 = 1 THEN s.stavalues1
            WHEN s.stakind2 = 1 THEN s.stavalues2
            WHEN s.stakind3 = 1 THEN s.stavalues3
            WHEN s.stakind4 = 1 THEN s.stavalues4
            WHEN s.stakind5 = 1 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_vals,
        CASE
            WHEN s.stakind1 = 1 THEN s.stanumbers1
            WHEN s.stakind2 = 1 THEN s.stanumbers2
            WHEN s.stakind3 = 1 THEN s.stanumbers3
            WHEN s.stakind4 = 1 THEN s.stanumbers4
            WHEN s.stakind5 = 1 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_freqs,
        CASE
            WHEN s.stakind1 = 2 THEN s.stavalues1
            WHEN s.stakind2 = 2 THEN s.stavalues2
            WHEN s.stakind3 = 2 THEN s.stavalues3
            WHEN s.stakind4 = 2 THEN s.stavalues4
            WHEN s.stakind5 = 2 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS histogram_bounds,
        CASE
            WHEN s.stakind1 = 3 THEN s.stanumbers1[1]
            WHEN s.stakind2 = 3 THEN s.stanumbers2[1]
            WHEN s.stakind3 = 3 THEN s.stanumbers3[1]
            WHEN s.stakind4 = 3 THEN s.stanumbers4[1]
            WHEN s.stakind5 = 3 THEN s.stanumbers5[1]
            ELSE NULL::real
        END AS correlation,
        CASE
            WHEN s.stakind1 = 4 THEN s.stavalues1
            WHEN s.stakind2 = 4 THEN s.stavalues2
            WHEN s.stakind3 = 4 THEN s.stavalues3
            WHEN s.stakind4 = 4 THEN s.stavalues4
            WHEN s.stakind5 = 4 THEN s.stavalues5
            ELSE NULL::anyarray
        END AS most_common_elems,
        CASE
            WHEN s.stakind1 = 4 THEN s.stanumbers1
            WHEN s.stakind2 = 4 THEN s.stanumbers2
            WHEN s.stakind3 = 4 THEN s.stanumbers3
            WHEN s.stakind4 = 4 THEN s.stanumbers4
            WHEN s.stakind5 = 4 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS most_common_elem_freqs,
        CASE
            WHEN s.stakind1 = 5 THEN s.stanumbers1
            WHEN s.stakind2 = 5 THEN s.stanumbers2
            WHEN s.stakind3 = 5 THEN s.stanumbers3
            WHEN s.stakind4 = 5 THEN s.stanumbers4
            WHEN s.stakind5 = 5 THEN s.stanumbers5
            ELSE NULL::real[]
        END AS elem_count_histogram
    FROM pg_statistic s
     JOIN pg_class c ON c.oid = s.starelid
     JOIN pg_attribute a ON c.oid = a.attrelid AND a.attnum = s.staattnum
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE NOT a.attisdropped AND has_column_privilege(c.oid, a.attnum, 'select'::text) AND (c.relrowsecurity = false OR NOT row_security_active(c.oid));
    ```

## 演習のクリーンアップ

1. この演習でデプロイした Azure Database for PostgreSQL には料金が発生しますが、この演習の後にサーバーを削除できます。 または、**rg-learn-work-with-postgresql-eastus** リソース グループを削除して、この演習の一環としてデプロイしたすべてのリソースを削除することもできます。
1. 必要に応じて、.\DP3021Lab フォルダーを削除します。
