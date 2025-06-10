---
lab:
  title: システム カタログとビューに基づいてシステム パラメーターを構成し、メタデータを確認する
  module: Configure and manage Azure Database for PostgreSQL
---

# システム カタログとビューに基づいてシステム パラメーターを構成し、メタデータを確認する

この演習では、PostgreSQL のシステム パラメーターとメタデータを確認します。

## 開始する前に

この演習を完了するには、自分の Azure サブスクリプションが必要です。 Azure サブスクリプションをお持ちでない場合は、[Azure 無料試用版](https://azure.microsoft.com/free)を作成できます。

コンピューターに以下がインストールされている必要があります。

- Visual Studio Code。
- Postgres Visual Studio Code Extension by Microsoft。
- Azure CLI。
- Git.

## 演習環境を作成する

この演習以降の演習では、Bicep スクリプトを使用して、Azure Database for PostgreSQL - フレキシブル サーバーとその他のリソースを Azure サブスクリプションにデプロイします。 Bicep スクリプトは、前にクローンした GitHub リポジトリの`/Allfiles/Labs/Shared`フォルダーにあります。

### Visual Studio Code と PostgreSQL 拡張機能をダウンロードしてインストールします。

Visual Studio Code がインストールされていない場合:

1. ブラウザーで、[Visual Studio Code のダウンロード](https://code.visualstudio.com/download)に移動し、オペレーティング システムに適したバージョンを選択します。

1. 使用しているオペレーティング システムのインストール手順に従ってください。

1. Visual Studio Code を開きます。

1. 左側のメニューで **[拡張機能]** を選んで、[拡張機能] パネルを表示します。

1. 検索バーに「**PostgreSQL**」と入力します。 Visual Studio Code 用 PostgreSQL 拡張機能のアイコンが表示されます。 Microsoft による製品が選択されていることを確認します。

1. **[インストール]** を選択します。 拡張機能がインストールされます。

### Azure CLI と Git のダウンロードとインストール

Azure CLI または Git がインストールされていない場合:

1. ブラウザーで、[Azure CLI のインストール](https://learn.microsoft.com/cli/azure/install-azure-cli)の、オペレーティング システムのインストール手順に従ってください。

1. ブラウザーで、 [Git のダウンロードとインストール](https://git-scm.com/downloads)に移動し、オペレーティング システムの指示に従います。

### 演習用ファイルをダウンロードする

演習ファイルを含む GitHub リポジトリを既にクローンしている場合は、*演習ファイルのダウンロードをスキップします*。

演習ファイルをダウンロードするには、演習ファイルを含む GitHub リポジトリをローカル コンピューターにクローンします。 リポジトリには、この演習を完了するために必要なすべてのスクリプトとリソースが含まれています。

1. まだ開いていない場合は、Visual Studio Code を開きます。

1. **[すべてのコマンドを表示]** (Ctrl + Shift + P) を選択してコマンド パレットを開きます。

1. コマンド パレットで、**Git: Clone** を検索して選択します。

1. コマンド パレットで、次のように入力して、演習リソースを含む GitHub リポジトリをクローンし、 **Enter** キーを押します。

    ```bash
    https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

1. 画面の指示に従って、プロジェクトをクローンするリポジトリの場所を選択します。 リポジトリは、選択した場所の `mslearn-postgresql` という名前のフォルダーにクローンされます。

1. 複製したリポジトリを開くかどうかを尋ねられたら **[開く]** を選択します。 Visual Studio Code でリポジトリを開きます。

### Azure サブスクリプションにリソースをデプロイする

Azure リソースが既にインストールされている場合は、*リソースのデプロイをスキップします*。

この手順では、Visual Studio Code の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

> &#128221; このラーニング パスで複数のモジュールを行っている場合は、それらの間で Azure 環境を共有できます。 その場合は、このリソース デプロイ手順を 1 回完了するだけで済みます。

1. まだ開いていない場合は、Visual Studio Code を開き、GitHub リポジトリをクローンしたフォルダーを開きます。

1. エクスプローラー ウィンドウで **mslearn-postgresql** フォルダーを展開します。

1. **Allfiles/Labs/Shared** フォルダーを展開します。

1. **Allfiles/Labs/Shared** フォルダーを右クリックし、**[統合ターミナルで開く]** を選択します。 この選択により、Visual Studio Code ウィンドウでターミナル ウィンドウが開きます。

1. ターミナルでは、既定で **powershell** ウィンドウが開く場合があります。 ラボのこのセクションでは、 **Bash シェル**を使用します。 **+** アイコンの他に、ドロップダウン矢印があります。 それを選択し、使用可能なプロファイルの一覧から **Git Bash** または **Bash** を選択します。 この選択により、新しいターミナル ウィンドウが開き、 **Bash シェル**が表示されます。

    > &#128221; 必要に応じて、 **powershell** ターミナル ウィンドウを閉じることができますが、必要はありません。 複数のターミナル ウィンドウを同時に開くことができます。

1. ターミナル ウィンドウで次のコマンドを実行して、Azure アカウントにサインインします。

    ```bash
    az login
    ```

    このコマンドによりブラウザー ウィンドウが開き、Azure アカウントへのサインインを求められます。 ログイン後、ターミナル ウィンドウに戻ります。

1. 次に、3 つのコマンドを実行して、Azure CLI コマンドを使用して Azure リソースを作成するときに冗長な入力を減らすための変数を定義します。 これらの変数は、リソース グループに割り当てる名前 (`RG_NAME`)、リソースがデプロイされる Azure リージョン (`REGION`)、PostgreSQL 管理者サインイン用にランダムに生成されたパスワード (`ADMIN_PASSWORD`) を表します。

    最初のコマンドでは、対応する変数に割り当てられるリージョンは `eastus` ですが、任意の場所に置き換えることもできます。

    ```bash
    REGION=eastus
    ```

    次のコマンドでは、この演習で使用されるすべてのリソースを格納するリソース グループに使用する名前が割り当てられます。 対応する変数に割り当てられたリソース グループ名は `rg-learn-work-with-postgresql-$REGION` で、その中の `$REGION` は以前に指定した場所です。 *ただし、この部分は好みに合わせたものや既にお持ちのものに応じて、他のリソース グループ名に変更できます*。

    ```bash
    RG_NAME=rg-learn-work-with-postgresql-$REGION
    ```

    最後のコマンドでは、PostgreSQL 管理者サインイン用のパスワードがランダムに生成されます。 後で PostgreSQL フレキシブル サーバーに接続するのに使用できるように、このパスワードを必ず安全な場所にコピーしてください。

    ```bash
    #!/bin/bash
    
    # Define array of allowed characters explicitly
    chars=( {a..z} {A..Z} {0..9} '!' '@' '#' '$' '%' '^' '&' '*' '(' ')' '_' '+' )
    
    a=()
    for ((i = 0; i < 100; i++)); do
        rand_char=${chars[$RANDOM % ${#chars[@]}]}
        a+=("$rand_char")
    done
    
    # Join first 18 characters without delimiter
    ADMIN_PASSWORD=$(IFS=; echo "${a[*]:0:18}")
    
    echo "Your randomly generated PostgreSQL admin user's password is:"
    echo "$ADMIN_PASSWORD"
    echo "Please copy it to a safe place, as you will need it later to connect to your PostgreSQL flexible server."
    ```

1. (既定のサブスクリプションを使用している場合は、この手順をスキップします。) 複数の Azure サブスクリプションにアクセスでき、この演習のリソース グループやその他のリソースを作成するサブスクリプションが既定のサブスクリプション*でない*場合は、次のコマンドを実行して適切なサブスクリプションを設定し、`<subscriptionName|subscriptionId>` トークンを、使用するサブスクリプションの名前または ID に置き換えます。

    ```azurecli
    az account set --subscription 16b3c013-d300-468d-ac64-7eda0820b6d3
    ```

1. (既存のリソース グループを使用している場合は、この手順をスキップします) 次の Azure CLI コマンドを使用してリソース グループを作成します。

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

1. 最後に、Azure CLI を使用して Bicep デプロイ スクリプトを実行し、リソース グループに Azure リソースをプロビジョニングします。

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL - フレキシブル サーバーです。 bicep スクリプトではデータベースも作成されます。このデータベースは、コマンド ラインでパラメーターとして構成できます。

    デプロイが完了するまでに通常数分かかります。 Bash ターミナルから監視するか、以前に作成したリソース グループの **[デプロイ]** ページに移動し、そこでデプロイの進行状況を確認することができます。

1. スクリプトは PostgreSQL サーバーのランダムな名前を作成するため、次のコマンドを実行してサーバーの名前を見つけることができます。

    ```azurecli
    az postgres flexible-server list --query "[].{Name:name, ResourceGroup:resourceGroup, Location:location}" --output table
    ```

    この演習の後半でサーバーに接続する必要があるため、サーバー名を書き留めておいてください。

    > &#128221; Azure ポータルでもサーバー名を見つけることができます。 Azure portal で **[Resource グループ]** に移動し、前に作成したリソース グループを選択します。 PostgreSQL サーバーがリソース グループに一覧表示されます。

### デプロイ エラーのトラブルシューティング

Bicep デプロイ スクリプトを実行すると、いくつかエラーが発生する場合があります。 最も一般的なメッセージとその解決手順は次のとおりです。

- 以前にこのラーニング パスで Bicep デプロイ スクリプトを実行し、その後でリソースを削除した場合、リソースを削除してから 48 時間以内にスクリプトをまた実行しようとすると、次のようなエラー メッセージが表示される場合があります。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is '4e87a33d-a0ac-4aec-88d8-177b04c1d752'. See inner errors for details."}
    
    Inner Errors:
    {"code": "FlagMustBeSetForRestore", "message": "An existing resource with ID '/subscriptions/{subscriptionId}/resourceGroups/rg-learn-postgresql-ai-eastus/providers/Microsoft.CognitiveServices/accounts/{accountName}' has been soft-deleted. To restore the resource, you must specify 'restore' to be 'true' in the property. If you don't want to restore existing resource, please purge it first."}
    ```

    このメッセージが表示された場合は、`restore` パラメーターが `true` に設定されるように前の `azure deployment group create` コマンドを変更し、再実行します。

- 選択したリージョンで特定のリソースのプロビジョニングが制限されている場合は、`REGION` 変数を別の場所に設定してコマンドを再実行し、リソース グループを作成して、Bicep デプロイ スクリプトを実行する必要があります。

    ```bash
    {"status":"Failed","error":{"code":"DeploymentFailed","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGrouName}/providers/Microsoft.Resources/deployments/{deploymentName}","message":"At least one resource deployment operation failed. Please list deployment operations for details. Please see https://aka.ms/arm-deployment-operations for usage details.","details":[{"code":"ResourceDeploymentFailure","target":"/subscriptions/{subscriptionId}/resourceGroups/{resourceGroupName}/providers/Microsoft.DBforPostgreSQL/flexibleServers/{serverName}","message":"The resource write operation failed to complete successfully, because it reached terminal provisioning state 'Failed'.","details":[{"code":"RegionIsOfferRestricted","message":"Subscriptions are restricted from provisioning in this region. Please choose a different region. For exceptions to this rule please open a support request with Issue type of 'Service and subscription limits'. See https://review.learn.microsoft.com/en-us/azure/postgresql/flexible-server/how-to-request-quota-increase for more details."}]}]}}
    ```

- ラボで AI リソースが必要な場合、次のエラーが発生する可能性があります。 このエラーは、責任ある AI 契約に同意する必要があるため、スクリプトで AI リソースを作成できない場合に発生します。 その場合は、Azure portal ユーザー インターフェイスを使用して Azure AI Services リソースを作成し、デプロイ スクリプトを再実行します。

    ```bash
    {"code": "InvalidTemplateDeployment", "message": "The template deployment 'deploy' is not valid according to the validation procedure. The tracking id is 'f8412edb-6386-4192-a22f-43557a51ea5f'. See inner errors for details."}
     
    Inner Errors:
    {"code": "ResourceKindRequireAcceptTerms", "message": "This subscription cannot create TextAnalytics until you agree to Responsible AI terms for this resource. You can agree to Responsible AI terms by creating a resource through the Azure Portal then trying again. For more detail go to https://go.microsoft.com/fwlink/?linkid=2164190"}
    ```

## Visual Studio Code の PostgreSQL 拡張機能に接続する

このセクションでは、Visual Studio Code の PostgreSQL 拡張機能を使用して PostgreSQL サーバーに接続します。 PostgreSQL 拡張機能を使用して、PostgreSQL サーバーに対して SQL スクリプトを実行します。

1. まだ開いていない場合は、Visual Studio Code を開き、GitHub リポジトリをクローンしたフォルダーを開きます。

1. 左側のメニューで **PostgreSQL** アイコンを選択します。

    > &#128221; PostgreSQL アイコンが表示されない場合は、**Extensions** アイコンを選択し、**PostgreSQL** を検索します。 Microsoft の **PostgreSQL** 拡張機能を選択し、**[インストール]** を選択します。

1. PostgreSQL サーバーへの接続を既に作成している場合は、次の手順に進みます。 新しい接続の作成:

    1. **PostgreSQL** 拡張機能で、**[+ 接続の追加]** を選択して、新しい接続を追加します。

    1. **[新しい接続]** ダイアログ ボックスで、次の情報を入力します。

        - **サーバー名**: `<your-server-name>`.postgres.database.azure.com
        - **[認証の種類]**: パスワード
        - **ユーザー名**: pgAdmin
        - **パスワード**: 以前に生成したランダムなパスワード。
        - **[パスワードの保存]** チェック ボックスをオンにします。
        - **接続名**: `<your-server-name>`

    1. 接続をテストするには、**[テスト接続]** を選択します。 接続が成功した場合は、**[保存して接続]** を選択して接続を保存し、それ以外の場合は接続情報を確認して、もう一度やり直します。

1. まだ接続されていない場合は、PostgreSQL サーバーの **[接続]** を選択します。 Azure Database for PostgreSQL サーバーに接続されます。

1. サーバー ノードとそのデータベースを展開します。 既存のデータベースが一覧表示されます。

1. zoodb データベースをまだ作成していない場合は、**[ファイル]**、**[ファイルを開く]** の順に選択し、スクリプトを保存したフォルダーに移動します。 **../Allfiles/Labs/02/Lab2_ZooDb.sql** を選択して、**[開く]** を選択します。

1. Visual Studio Code の右下で、接続が緑色になっていることを確認します。 そうでない場合は、**PGSQL Disconnected** と表示されます。 **[PGSQL Disconnected]** テキストを選択し、コマンド パレットの一覧から [PostgreSQL サーバー接続] を選択します。 パスワードを要求された場合は、前に生成したパスワードを入力します。

1. データベースを作成する時間。

    1. **DROP** および **CREATE** ステートメントを強調表示し、それらを実行します。

    1. **SELECT current_database()** ステートメントだけを強調表示して実行すると、データベースが現在 `postgres` に設定されていることがわかります。 それを `zoodb` に変更する必要があります。

    1. メニューバーの *実行* アイコンのある省略記号を選択し、**[PostgreSQL データベースの変更]** を選択します。 データベースの一覧から `zoodb` を選択します。

        > &#128221; クエリ ペインでデータベースを変更することもできます。 サーバー名とデータベース名は、クエリ タブ自体の下に書き留めることができます。 データベース名を選択すると、データベースの一覧が表示されます。 一覧から `zoodb` データベースを選択します。

    1. **SELECT current_database()** ステートメントをもう一度実行して、データベースが現在 `zoodb` に設定されていることを確認します。

    1. **[テーブルの作成]**、**[外部キーの作成]**、および **[テーブルの事前設定]** セクションを強調表示し、それらを実行します。

    1. スクリプトの最後にある **SELECT** ステートメント 3 つを強調表示してそれらを実行し、テーブルが作成および事前設定されたことを確認します。

## タスク 1: PostgreSQL のバキューム プロセスを探索する

このセクションでは、PostgreSQL のバキューム プロセスについて学習します。 バキューム プロセスは、ストレージ領域を再利用し、データベースのパフォーマンスを最適化するために使用されます。 バキューム プロセスは、手動でも実行できますが、自動実行されるように設定しておくことも可能です。

1. Visual Studio Code ウィンドウで、**[ファイル]**、**[ファイルを開く]** の順に選び、ラボ スクリプトに移動します。 **../Allfiles/Labs/07/Lab7_vacuum.sql** を選択し、**[開く]** を選択します。 必要に応じて、 **[PGSQL Disconnected]** テキストを選択し、コマンド パレットの一覧から [PostgreSQL サーバー接続] を選択して、サーバーに再接続します。 パスワードを要求された場合は、前に生成したパスワードを入力します。

1. Visual Studio Code の右下で、接続が緑色になっていることを確認します。 そうでない場合は、**PGSQL Disconnected** と表示されます。 **[PGSQL Disconnected]** テキストを選択し、コマンド パレットの一覧から [PostgreSQL サーバー接続] を選択します。 パスワードを要求された場合は、前に生成したパスワードを入力します。

1. **SELECT current_database()** ステートメントを実行して、現在のデータベースを確認します。 接続が現在 **zoodb** データベースに設定されているかどうかを確認します。 そうでない場合は、データベースを **zoodb** に変更できます。 データベースを変更するには、メニュー バーの省略記号を選択して、*実行* アイコンを選択し、 **[PostgreSQL データベースの変更]** を選択します。 データベースの一覧から `zoodb` を選択します。 **SELECT current_database();** ステートメントを実行して、データベースが現在 `zoodb` に設定されていることを確認します。

1. **[Display dead tuples](無効タプルを表示する)** セクションを強調表示にして実行します。 このクエリを実行すると、データベース内の無効タプルと有効タプルの数が表示されます。 無効タプルの数を記録します。

1. 「**重みを変更する**」セクションを強調表示にして連続で 10 回実行します。 このクエリは、すべての動物の重み列を更新します。

1. **[Display dead tuples](無効タプルを表示する)** セクションをもう一度実行します。 更新が完了したら、無効タプルの数を記録しておきます。

1. **[Manually run VACUUM](VACUUM を手動で実行する)** セクションを実行して、バキューム プロセスを実行します。

1. **[Display dead tuples](無効タプルを表示する)** セクションをもう一度実行します。 バキューム プロセスが完了したら、無効タプルの数を記録しておきます。

## タスク 2: 自動バキューム サーバー パラメーターを構成する

このセクションでは、自動バキューム サーバー パラメーターを構成します。 自動バキューム プロセスは、ストレージ スペースを自動的に回収し、データベースのパフォーマンスを最適化するために使用されます。 特定のパラメーターに基づいて自動的に動作するように自動バキューム プロセスを構成できます。

1. まだであれば、[Azure portal](https://portal.azure.com) に移動してサインインします。

1. Azure portal で、自分の Azure Database for PostgreSQL フレキシブル サーバーに移動します。

1. **[設定]** で、**[サーバー パラメーター]** を選択します。

1. 検索バーに「**`vacuum`**」と入力します。 次のパラメーターを見つけて、次のように値を変更します。

    - autovacuum = ON (既定で ON になっているはずです)
    - autovacuum_vacuum_scale_factor = 0.1
    - autovacuum_vacuum_threshold = 50

    このように変更すると、削除対象としてマークされた行がテーブルの 10% にある場合や、いずれか 1 つのテーブルで 50 行が更新または削除された場合に、自動バキューム プロセスが実行されます。

1. **[保存]** を選択します。 サーバーが再起動されます。

## タスク 3: Azure portal で PostgreSQL メタデータを表示する

このセクションでは、Azure portal で PostgreSQL メタデータを表示します。 Azure portal には、PostgreSQL サーバーを管理および監視するためのグラフィカル インターフェイスが用意されています。

1. まだであれば、[Azure portal](https://portal.azure.com) に移動してサインインします。

1. 「**Azure Database for PostgreSQL**」を検索して選択します。

1. この演習用に作成した Azure Database for PostgreSQL フレキシブル サーバーを選択します。

1. **[監視]** で **[メトリック]** を選択します。

1. **[メトリック]** を選択し、**[CPU percent] (CPU 使用率)** を選択します。

1. データベースに関するさまざまなメトリックを表示できることがわかります。

## タスク 4: システム カタログ テーブルのデータを表示する

このセクションでは、システム カタログ テーブルのデータを表示します。 システム カタログ テーブルは、PostgreSQL のデータベース オブジェクトに関するメタデータを格納するために使用されます。 これらのテーブルに対してクエリを実行して、データベース オブジェクトに関する情報を取得できます。

1. まだ開いていない場合は、Visual Studio Code を開きます。

1. コマンド パレット (Ctrl + Shift + P) を表示し、**[PGSQL: 新しいクエリ]** を選択します。 コマンド パレットの一覧から、作成した新しい接続を選択します。 パスワードを要求された場合は、新しいロール用に作成したパスワードを入力します。

1. **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

    ```sql
    SELECT datname, xact_commit, xact_rollback FROM pg_stat_database;
    ```

1. 各データベースのコミットとロールバックを表示できることがわかります。

## システム ビューを使用して複雑なメタデータ クエリを表示する

このセクションでは、システム ビューを使用して複雑なメタデータ クエリを表示します。 システム ビューは、PostgreSQL のデータベース オブジェクトに関するメタデータをクエリするためのシンプルなインターフェイスを提供するために使用されます。 

1. まだ開いていない場合は、Visual Studio Code を開きます。

1. コマンド パレット (Ctrl + Shift + P) を表示し、**[PGSQL: 新しいクエリ]** を選択します。 コマンド パレットの一覧から、作成した新しい接続を選択します。 パスワードを要求された場合は、新しいロール用に作成したパスワードを入力します。

1. **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

    ```sql
    SELECT *
    FROM pg_catalog.pg_stats;
    ```

1. 大量の統計情報を表示できることがわかります。

1. システム ビューを使うと、記述する必要がある SQL の複雑さを軽減できます。 **pg_stats** ビューを使わない場合、前述のクエリには次のコードが必要になります。ここで実際にコードを実行してその機能する仕組みを見てみましょう。 **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

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

## クリーンアップ

1. 他の演習でこの PostgreSQL サーバーが不要な場合は、不要な Azure コストが発生しないように、この演習で作成したリソース グループを削除します。

1. PostgreSQL サーバーを稼働させ続けたい場合は、そのまま起動した状態にしておくことができます。 そのまま実行状態にしておきたくない場合は、bash ターミナルで不要なコストが発生しないようにサーバーを停止できます。 サーバーを停止するには、次のコマンドを実行します。

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    `<your-server-name>` をPostgreSQL サーバーの名前に置き換えます。

    > &#128221; Azure portal からサーバーを停止することもできます。 Azure portal で **[Resource グループ]** に移動し、前に作成したリソース グループを選択します。 PostgreSQL サーバーを選択し、メニューから **[停止]** を選択します。

1. 必要に応じて、先ほどクローンした Git リポジトリを削除します。

この演習では、PostgreSQL のシステム パラメーターとメタデータの確認方法について学習しました。 また、Azure portal で PostgreSQL メタデータを表示し、システム カタログ テーブルでデータを表示する方法についても学習しました。 さらに、システム ビューを使用して複雑なメタデータ クエリを表示する方法も学習しました。
