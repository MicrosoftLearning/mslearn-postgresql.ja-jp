---
lab:
  title: クエリ ストアを使用してクエリ パフォーマンスを評価する
  module: Tune queries in Azure Database for PostgreSQL
---

# クエリ ストアを使用してクエリ パフォーマンスを評価する

この演習では、Azure Database for PostgreSQL のクエリ ストアを使用してパフォーマンス メトリックのクエリを実行する方法について学びます。

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
    az deployment group create --resource-group $RG_NAME --template-file "Allfiles/Labs/Shared/deploy-postgresql-server.bicep" --parameters adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD databaseName=adventureworks
    ```

    Bicep デプロイ スクリプトでは、この演習を完了するのに必要な Azure サービスがリソース グループにプロビジョニングされます。 デプロイされるリソースは、Azure Database for PostgreSQL - フレキシブル サーバーです。 bicep スクリプトは、adventureworks データベースも作成します。

    デプロイが完了するまでに通常は数分かかります。 Bash ターミナルから監視するか、以前に作成したリソース グループの **[デプロイ]** ページに移動し、そこでデプロイの進行状況を確認することができます。

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

### psql をインストールする

CSV ファイルから PostgreSQL データベースにデータをコピーするため、ローカル コンピューターに `psql` をインストールしておく必要があります。 *既に `psql` がインストールされている場合は、このセクションをスキップしてください。*

1. **psql** が環境に既にインストールされているかどうかを確認するには、コマンド ライン/ターミナルを開き、コマンド ***psql*** を実行します。 "*psql: error: connection to server on socket...*" のようなメッセージが返された場合は、**psql** ツールが環境に既にインストールされており、再インストールする必要はなく、このセクションをスキップできます。

1. [psql](https://sbp.enterprisedb.com/getfile.jsp?fileid=1258893) をインストールします。

1. セットアップ ウィザードで、**[コンポーネントの選択]** ダイアログ ボックスに到達するまでプロンプトに従い、**[コマンド ライン ツール]** を選択します。 他のコンポーネントを使用する予定がなければ、それらのコンポーネントのチェックマークは外してかまいません。 画面に表示される指示に従って、インストールを完了します。

1. インストールが正常に完了したかどうか確認するには、新しいコマンド プロンプトまたはターミナル ウィンドウを開いて **psql** を実行してみます。 "*psql: error: connection to server on socket...*" のようなメッセージが表示された場合は、**psql** ツールが正常にインストールされていることを意味します。 正常にインストールされていない場合は、PostgreSQL bin ディレクトリをシステム PATH 変数に追加する必要があります。

    1. Windows を使用している場合は、PostgreSQL bin ディレクトリをシステム PATH 変数に追加してください。 bin ディレクトリは通常、`C:\Program Files\PostgreSQL\<version>\bin` にあります。
        1. コマンド プロンプト内でコマンド `echo %PATH%` を実行し、PostgreSQL bin ディレクトリが一覧表示されるかどうかを確認することで、bin ディレクトリが PATH 変数に含まれているかどうかを確認できます。 もし表示されていなければ、bin ディレクトリを手動で追加します。
        1. 手動で追加するには、まず **[スタート]** ボタンを右クリックします。
        1. **[システム]** を選択し、**[システムの詳細設定]** を選択します。
        1. **[環境変数]** ボタンを選択します。
        1. **[システム変数]** セクションで、**PATH** 変数をダブルクリックします。
        1. **[新規]** を選択し、PostgreSQL bin ディレクトリへのパスを追加します。
        1. 追加した後、コマンド プロンプトをいったん閉じて再度開き、変更が反映されるようにします。

    1. macOS または Linux を使用している場合、PostgreSQL `bin` ディレクトリは通常、`/usr/local/pgsql/bin` にあります。  
        1. ターミナルで `echo $PATH` を実行することで、このディレクトリが `PATH` 環境変数に含まれているかを確認できます。  
        1. 含まれていない場合は、使用するシェルに応じてシェル構成ファイル (通常は `.bash_profile`、`.bashrc`、または `.zshrc`) を編集することで追加できます。

### データベースにデータを事前設定する

**psql**がインストールされていることを確認したら、コマンド プロンプトまたはターミナル ウィンドウを開き、コマンド ラインを使用して PostgreSQL サーバーに接続できます。

> &#128221; Windows を使用している場合は、**Windows PowerShell** または**コマンド プロンプト**を使用できます。 macOS または Linux を使用している場合は、**ターミナル** アプリケーションを使用できます。

サーバーに接続するための構文は次のとおりです。

```sql
psql -h <servername> -p <port> -U <username> <dbname>
```

1. コマンド プロンプトまたはターミナルで「**`--host=<servername>.postgres.database.azure.com`**」と入力します。ここで、`<servername>` は、先ほど作成した Azure Database for PostgreSQL の名前です。

    サーバー名は、Azure portal の **[概要]** で、または bicep スクリプトの出力として確認できます。

    ```sql
   psql -h <servername>.postgres.database.azure.com -p 5432 -U pgAdmin adventureworks
    ```

    先ほどコピーした管理者アカウントのパスワードの入力を求めるメッセージが表示されます。

1. この演習でロックを確認するときに使用する情報が得られるように、データベース内にテーブルを作成し、サンプル データを事前設定する必要があります。

1. 次のコマンドを実行して、データを読み込むための `production.workorder` テーブルを作成します。

    ```sql
    DROP SCHEMA IF EXISTS production CASCADE;
    CREATE SCHEMA production;
    
    DROP TABLE IF EXISTS production.workorder;
    CREATE TABLE production.workorder
    (
        workorderid integer NOT NULL,
        productid integer NOT NULL,
        orderqty integer NOT NULL,
        scrappedqty smallint NOT NULL,
        startdate timestamp without time zone NOT NULL,
        enddate timestamp without time zone,
        duedate timestamp without time zone NOT NULL,
        scrapreasonid smallint,
        modifieddate timestamp without time zone NOT NULL DEFAULT now()
    )
    WITH (
        OIDS = FALSE
    )
    TABLESPACE pg_default;
    ```

1. 次に、`COPY` コマンドを使用して、先ほど作成したテーブルに CSV ファイルからデータを読み込みます。 まず、次のコマンドを実行して、`production.workorder` テーブルにデータを入力します。

    ```sql
    \COPY production.workorder FROM 'mslearn-postgresql/Allfiles/Labs/08/Lab8_workorder.csv' CSV HEADER
    ```

    コマンドの出力は `COPY 72591` となるはずです。これは、CSV ファイルから 72,591 行がテーブルに書き込まれたことを示しています。

1. コマンド プロンプトまたはターミナル ウィンドウを閉じます。

### Visual Studio Code でデータベースに接続する

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

1. まだ開いていない場合は、Visual Studio Code を開きます。

1. コマンド パレット (Ctrl + Shift + P) を表示し、**[PGSQL: 新しいクエリ]** を選択します。 コマンド パレットの一覧から、作成した新しい接続を選択します。 パスワードを要求された場合は、新しいロール用に作成したパスワードを入力します。

1. **[新しいクエリ]** ウィンドウの右下で、接続が緑色になっていることを確認します。 そうでない場合は、**PGSQL Disconnected** と表示されます。 **[PGSQL Disconnected]** テキストを選択し、コマンド パレットの一覧から [PostgreSQL サーバー接続] を選択します。 パスワードを要求された場合は、前に生成したパスワードを入力します。

1. **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

    ```sql
    SELECT current_database();
    ```

1. 現在のデータベースが **adventureworks** に設定されていない場合は、データベースを **adventureworks** に変更する必要があります。 データベースを変更するには、メニュー バーの省略記号を選択して、*実行* アイコンを選択し、 **[PostgreSQL データベースの変更]** を選択します。 データベースの一覧から `adventureworks` を選択します。 **SELECT current_database();** ステートメントを実行して、データベースが現在 `adventureworks` に設定されていることを確認します。

1. **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

    ```sql
    SELECT * FROM production.workorder;
    ```

1. **[新しいクエリ]** ウィンドウを最小化します。この演習で後ほど、このウィンドウに戻ります。

## タスク 1: クエリ キャプチャ モードを有効にする

1. Azure portal に移動してサインインします。

1. この演習では、Azure Database for PostgreSQL サーバーを選択します。

1. **[設定]** で、**[サーバー パラメーター]** を選択します。

1. **`pg_qs.query_capture_mode`** 設定に移動します。

1. **[上位]** を選択します。

1. **`pgms_wait_sampling.query_capture_mode`** に移動して、**[すべて]** を選択し、**[保存]** を選択します。

1. サーバー パラメーターが更新されるまで待ちます。

## pg_stat データを表示する

1. Visual Studio Code に戻り、前に開いた **[新しいクエリ]** ウィンドウを選択します。

1. 既存のクエリを次のクエリに置き換えて、**[実行]** を選択します。

    ```sql
    SELECT 
        pid,                    -- Process ID of the server process
        datid,                  -- OID of the database
        datname,                -- Name of the database
        usename,                -- Name of the user
        application_name,       -- Name of the application connected to the database
        client_addr,            -- IP address of the client
        client_hostname,        -- Hostname of the client (if available)
        client_port,            -- TCP port number that the client is using for the connection
        backend_start,          -- Timestamp when the backend process started
        xact_start,             -- Timestamp of the current transaction start, if any
        query_start,            -- Timestamp when the current query started, if any
        state_change,           -- Timestamp when the state was last changed
        wait_event_type,        -- Type of event the backend is waiting for, if any
        wait_event,             -- Event that the backend is waiting for, if any
        state,                  -- Current state of the session (e.g., active, idle, etc.)
        backend_xid,            -- Transaction ID, if active
        backend_xmin,           -- Transaction ID that the process is working with
        query,                  -- Text of the query being executed
        encode(backend_type::bytea, 'escape') AS backend_type,           -- Type of backend (e.g., client backend, autovacuum worker). We use encode(…, 'escape') to safely display raw data with invalid characters by converting it into a readable format, doing this prevents a UTF-8 conversion error in Visual Studio Code.
        leader_pid,             -- PID of the leader process, if this is a parallel worker
        query_id               -- Query ID (added in more recent PostgreSQL versions)
    FROM pg_stat_activity;
    ```

1. 使用可能なメトリックを確認します。

1. 次のタスクのために Visual Studio Code を開いたままにします。

## タスク 2: クエリ統計を調べる

> &#128221; 新しく作成されたデータベースの場合、統計があるとしても、限られたものしかない可能性があります。 30 分待つと、バックグラウンド プロセスからの統計情報が表示されます。

1. **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

    ```sql
    SELECT current_database();
    ```

1. データベースを **azure_sys** に変更する必要があります。 データベースを変更するには、メニュー バーの省略記号を選択して、*実行* アイコンを選択し、 **[PostgreSQL データベースの変更]** を選択します。 データベースの一覧から `azure_sys` を選択します。 **SELECT current_database();** ステートメントを実行して、データベースが現在 `azure_sys` に設定されていることを確認します。

1. **[新しいクエリ]** ウィンドウで、次の SQL ステートメントをコピー、強調表示、および実行します。

    ```sql
    SELECT * FROM query_store.query_texts_view;
    ```

    ```sql
    SELECT * FROM query_store.qs_view;
    ```

    ```sql
    SELECT * FROM query_store.runtime_stats_view;
    ```

    ```sql
    SELECT * FROM query_store.pgms_wait_sampling_view;
    ```

1. 使用可能なメトリックを確認します。

## クリーンアップ

1. 他の演習でこの PostgreSQL サーバーが不要な場合は、不要な Azure コストが発生しないように、この演習で作成したリソース グループを削除します。

1. PostgreSQL サーバーを稼働させ続けたい場合は、そのまま起動した状態にしておくことができます。 起動した状態にしておきたくない場合は、不要なコストが発生しないように、Bash ターミナルでサーバーを停止できます。 サーバーを停止するには、次のコマンドを実行します。

    ```azurecli
    az postgres flexible-server stop --name <your-server-name> --resource-group $RG_NAME
    ```

    `<your-server-name>` をPostgreSQL サーバーの名前に置き換えます。

    > &#128221; Azure portal からサーバーを停止することもできます。 Azure portal で **[Resource グループ]** に移動し、前に作成したリソース グループを選択します。 PostgreSQL サーバーを選択し、メニューから **[停止]** を選択します。

1. 必要に応じて、先ほどクローンした Git リポジトリを削除します。

この演習では、Azure Database for PostgreSQL のクエリ ストアを使用してクエリ パフォーマンスを評価する方法を学びました。 また、pg_stat データを表示してクエリの統計情報を調べる方法についても学習しました。
