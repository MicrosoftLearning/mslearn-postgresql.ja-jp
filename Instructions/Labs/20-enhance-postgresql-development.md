---
lab:
  title: Visual Studio Code PostgreSQL 拡張機能と GitHub Copilot を調べる
  module: Develop PostgreSQL solutions in Visual Studio Code with the PostgreSQL extension and GitHub Copilot
  description: この演習の前半では、Azure Database for PostgreSQL フレキシブル サーバーをデプロイし、サンプル データを含む賃貸物件データベースを作成しました。 ここでは、Visual Studio Code の PostgreSQL 拡張機能を使用してそのデータベースに接続します。 サーバー名、管理者ユーザー名、デプロイ時に生成されたパスワードが必要です。
  duration: 45 minutes
  level: 400
  islab: true
  primarytopics:
    - Azure
    - Azure Database for PostgreSQL
    - GitHub
    - Visual Studio
    - Visual Studio Code
---

#  Visual Studio Code PostgreSQL 拡張機能と GitHub Copilot を調べる

あなたは Margie's Travel の開発リーダーとして、Azure Database for PostgreSQL を使用する際のチームの生産性を向上させる任務があります。 Visual Studio Code の PostgreSQL 拡張機能と GitHub Copilot の両方を使用して、SQL をより効率的に記述し改良する方法について学びたいと思っています。 この演習では、必要な Azure リソースをデプロイし、データベースに接続し、後の手順で拡張機能と Copilot を使用できるようにサンプル データを準備します。

## 開始する前に

管理者権限のある [Azure サブスクリプション](https://azure.microsoft.com/free)が必要です。

## Azure サブスクリプションにリソースをデプロイする

このステップ ガイドでは、Azure Cloud Shell の Azure CLI コマンドにより、リソース グループを作成し、Bicep スクリプトを実行して、この演習を完了するのに必要な Azure サービスを Azure サブスクリプションにデプロイします。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) に移動します。

2. Azure portal ツール バーの **[Cloud Shell]** アイコンを選択して、ブラウザー画面の下部に新しい [Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) 画面を開きます。

    ![[Cloud Shell] アイコンが赤の四角で強調表示された Azure portal のスクリーンショット。](./media/portal-toolbar-cloud-shell.png)

    メッセージが表示されたら、必要なオプションを選択して *Bash* シェルを開きます。 以前に *PowerShell* コンソールを使用していた場合は、*Bash* シェルに切り替えます。

3. 演習リソースを含む GitHub リポジトリをクローンします。

    ```bash
    git clone https://github.com/MicrosoftLearning/mslearn-postgresql.git
    ```

4. Azure リージョン、リソース グループ名、PostgreSQL 管理者パスワードの変数を定義します。

    ```bash
    REGION=eastus
    ```

    ```bash
    RG_NAME=rg-learn-postgresql-ai-$REGION
    ```

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

5. 必要に応じて、正しい Azure サブスクリプションに切り替えます。

    ```azurecli
    az account set --subscription <subscriptionName|subscriptionId>
    ```

6. リソース グループを作成します。

    ```azurecli
    az group create --name $RG_NAME --location $REGION
    ```

7. Bicep のデプロイを実行します。

    ```azurecli
    az deployment group create --resource-group $RG_NAME --template-file "mslearn-postgresql/Allfiles/Labs/Shared/deploy.bicep" --parameters restore=false adminLogin=pgAdmin adminLoginPassword=$ADMIN_PASSWORD

    ```

8. この演習では、2 つの方法のうち **1 つ**を使用して Azure OpenAI に対して認証を行います。 ご自分の環境に該当するものを選び、各ステップではそれらの手順にのみ従ってください。

    - **API キー**: Azure portal からコピーしたキーを使います (ほとんどの環境で機能します)。
    - **マネージド ID**: Microsoft Entra ID トークン ベースの認証を使います (組織レベルで API キーが無効になっている場合に必要です)。

    **マネージド ID** をお使いの場合は、ここで次のコマンドを実行してそれを設定します。 それ以外の場合、次のステップに進みます。

    ```bash
    # Re-derive all variables (in case your Cloud Shell session was reset)
    PGSERVER=$(az postgres flexible-server list -g "$RG_NAME" --query "[0].name" -o tsv)
    AOAI=$(az cognitiveservices account list -g "$RG_NAME" --query "[?kind=='OpenAI'].name | [0]" -o tsv)
    AOAI_ID=$(az cognitiveservices account show -g "$RG_NAME" -n "$AOAI" --query "id" -o tsv)
    SUB_ID=$(az account show --query "id" -o tsv)

    # Enable system-assigned managed identity on the PostgreSQL server
    az rest --method patch \
      --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG_NAME/providers/Microsoft.DBforPostgreSQL/flexibleServers/$PGSERVER?api-version=2024-08-01" \
      --body '{"identity":{"type":"SystemAssigned"}}'

    # Wait for the identity to be assigned, then get the principal ID
    echo "Waiting for system-assigned managed identity..."
    SYS_MI=""
    while [ -z "$SYS_MI" ] || [ "$SYS_MI" = "null" ]; do
      sleep 15
      SYS_MI=$(az rest --method get \
        --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG_NAME/providers/Microsoft.DBforPostgreSQL/flexibleServers/$PGSERVER?api-version=2024-08-01" \
        --query "identity.principalId" -o tsv)
      echo "principalId=$SYS_MI"
    done

    # Grant 'Cognitive Services OpenAI User' to the system MI (for in-database embeddings)
    az role assignment create \
      --assignee "$SYS_MI" \
      --role "Cognitive Services OpenAI User" \
      --scope "$AOAI_ID"

    # Restart the server so it picks up the new identity
    az postgres flexible-server restart -g "$RG_NAME" -n "$PGSERVER"
    ```

    > **注:** 再起動後、サーバーが再び稼働状態になり、ロールの割り当てが反映されるまで、2 から 3 分待ちます。 後の手順で認可エラーが発生する場合は、数分待ってからもう一度お試しください。

9. デプロイが完了したら、Cloud Shell を閉じます。

## デプロイ エラーのトラブルシューティング

(トラブルシューティングの完全なテキストは元のソースから変更されません)。

---

## Azure Cloud Shell で psql を使用してデータベースに接続する

このタスクでは、`psql` コマンドライン ユーティリティを使用して、Azure Database for PostgreSQL フレキシブル サーバー上の `rentals` データベースに接続します。

1. [Azure portal](https://portal.azure.com/) で、ツール バーの **[Cloud Shell]** アイコンを選んで Cloud Shell を開きます。

1. 次のコマンドを実行して、`rentals` データベースに接続します。`<server-name>` は、実際の PostgreSQL フレキシブル サーバーの名前に置き換えます (Azure portal の PostgreSQL リソースの **[概要]** ページに記載されています)。

    ```bash
    psql -h <server-name>.postgres.database.azure.com -U pgAdmin -d rentals
    ```

1. ダイアログが表示されたら、`pgAdmin` サインイン用にランダムに生成されたパスワードを入力します。

1. サインインした後、`rentals` データベースに接続されていることを確認します。

1. Cloud Shell で **[最大化]** を選択して、スペースを広げます。

    ![[最大化] ボタンが赤い四角で強調表示されている Azure Cloud Shell 画面のスクリーンショット。](./media/azure-cloud-shell-pane-rentals.png)

## データベースにサンプル データを入力する

PostgreSQL 拡張機能を調べて、Visual Studio Code で GitHub Copilot を使用する前に、サンプル データをデータベースに追加します。

1. `listings` テーブルを作成します。

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

2. `reviews` テーブルを作成します。

    ```sql
    DROP TABLE IF EXISTS reviews;

    CREATE TABLE reviews (
        id int,
        listing_id int,
        date date,
        comments text
    );
    ```

3. リスト データを読み込みます。

    ```sql
    \COPY listings FROM 'mslearn-postgresql/Allfiles/Labs/Shared/listings.csv' CSV HEADER
    ```

4. レビュー データを読み込みます。

    ```sql
    \COPY reviews FROM 'mslearn-postgresql/Allfiles/Labs/Shared/reviews.csv' CSV HEADER
    ```

## Visual Studio Code で PostgreSQL 拡張機能と GitHub Copilot を使用する

Azure Database for PostgreSQL サーバーがデプロイされ、サンプル データが読み込まれたので、PostgreSQL 拡張機能と GitHub Copilot を Visual Studio Code で使用する準備ができました。 以降のユニットの手順に従って、データベースに接続し、データベース オブジェクトを探索し、Copilot を使用して SQL クエリを生成し、クエリ結果を検証します。

### Visual Studio Code PostgreSQL 拡張機能をインストールする

このセクションでは、既に開発用コンピューターに Visual Studio Code がインストールされていることを前提としています。 まだ行っていない場合は、[こちら](https://code.visualstudio.com/)からダウンロードしてインストールします。

Visual Studio Code をインストールしたら、次の手順に従って PostgreSQL 拡張機能をインストールします (まだインストールしていない場合)。

1. Visual Studio Code を開きます。

1. ウィンドウの横にある [アクティビティ バー] の **[拡張機能]** アイコンを選択します。

1. **[拡張機能]** ビューで ***PostgreSQL*** を検索し、Microsoft から発行されている **PostgreSQL** という名前の拡張機能を選択します。

1. **[インストール]** ボタンを選択して、拡張機能を Visual Studio Code に追加します。

PostgreSQL 拡張機能をインストールしたら、Azure Database for PostgreSQL サーバーへの接続に進み、データベース オブジェクトの調査を開始し、SQL の生成と支援に GitHub Copilot を使用できます。

### Azure Database for PostgreSQL サーバーに接続する

この演習の前半では、Azure Database for PostgreSQL フレキシブル サーバーをデプロイし、サンプル データを含む `rentals` データベースを作成しました。 ここでは、Visual Studio Code の PostgreSQL 拡張機能を使用してそのデータベースに接続します。 サーバー名、管理者ユーザー名、デプロイ時に生成されたパスワードが必要です。

1. Visual Studio Code を開きます (まだ開いていない場合)。

1. **[アクティビティ バー]** で **[PostgreSQL]** アイコンを選択して PostgreSQL 拡張機能ビューを開きます。

1. PostgreSQL 拡張機能ビューの、**[新しい接続の追加]** ボタン (プラス記号付きのプラグのアイコン) を選択します。

1. **[新しい接続]** ダイアログで、次の詳細を入力します。

    - **サーバー名**: `<your-server-name>.postgres.database.azure.com` (`<your-server-name>` をお使いのデプロイの実際のサーバー名に置き換えます)
    - **[認証の種類]**: パスワード
    - **ユーザー名**: `pgAdmin`
    - **パスワード**: `<your-generated-password>` (デプロイ時に生成されたパスワードを使用します)
    - **(省略可能) パスワードの保存**:今後の接続のために、パスワードを Visual Studio Code に保存したい場合は、このボックスをオンにします。
    - **データベース名**: `rentals`
    - **接続名**: (任意の名前)

1. **[接続のテスト]** ボタンを選択して、接続の詳細が正しいことを確認します。

1. **[保存して接続]** ボタンを選択して、Azure Database for PostgreSQL サーバーへの接続を保存して確立します。

これらの手順を完了すると、Azure Database for PostgreSQL で `rentals` データベースに接続されます。 これで、データベース オブジェクトの調査、SQL クエリの記述、GitHub Copilot を使用した Visual Studio Code 内での PostgreSQL 開発タスクを支援が行えるようになりました。

### PostgreSQL 拡張機能をテストする

PostgreSQL 拡張機能がインストールされ、Azure Database for PostgreSQL サーバーに接続された状態で、データベース オブジェクトを調べて SQL クエリを実行して、その機能をテストできるようになりました。

1. **PostgreSQL** 拡張機能ビューで、作成した接続を展開して、データベースの一覧を表示します (まだ展開されていない場合)。

1. `rentals` データベースを展開して、スキーマ、テーブル、その他のオブジェクトを表示します。

1. `schemas` ノード、`public` スキーマ、最後に `Tables` ノードを展開して、先ほど作成したテーブル `listings` と `reviews` を表示します。

1. `listings` テーブルを右クリックし **[上位 1000 を選択]** を選ぶと、`listings` テーブルから最初の 1,000 行が返されます。 新しい [SQL クエリ エディター] タブが開き、生成された SQL クエリとその結果が表示されます。

1. クエリ エディターの下にある **[結果]** ペインで結果を確認し、データが正しく読み込まれたことを確認します。

1. `rentals` データベースを右クリックし **[新しいクエリ]** を選択して新しい [SQL クエリ エディター] タブを開きます。

1. 新しいクエリ エディターで、次のような SQL クエリを記述して実行します。

    ```sql
    SELECT COUNT(*) AS total_listings FROM listings;
    ```

    クエリにより、`listings` テーブル内のリストの合計数が返されます。

手順に沿って、データベース オブジェクトを探索し、Azure Database for PostgreSQL サーバーに対して SQL クエリを実行して、PostgreSQL 拡張機能を正常にテストしました。 次は、GitHub Copilot を使用した、Visual Studio Code での SQL クエリの生成と絞り込みの支援に進むことができます。

### Visual Studio Code に GitHub Copilot をインストールする

Visual Studio Code で PostgreSQL 拡張機能と GitHub Copilot を使用するには、まず GitHub Copilot 拡張機能をインストールする必要があります。 GitHub Copilot をインストールしてセットアップするには、次の手順に従います (まだインストールしていない場合)。

1. Visual Studio Code を開きます (まだ開いていない場合)。

1. ウィンドウの横にある [アクティビティ バー] の **[拡張機能]** アイコンを選択します。

1. **[拡張機能]** ビューで、***GitHub Copilot Chat*** を検索し、GitHub から発行されている **GitHub Copilot Chat** という名前の拡張機能を選択します。

1. **[インストール]** ボタンを選択して、GitHub Copilot 拡張機能をインストールします。

GitHub Copilot 拡張機能をインストールした後、ダイアログが表示されたら、GitHub アカウントにサインインし Copilot を有効にする必要がある場合があります。 設定が完了したら、GitHub Copilot の使用を開始して、Visual Studio Code で PostgreSQL データベースを操作しながら SQL クエリの生成と絞り込みの支援を行うことができます。

### GitHub Copilot を PostgreSQL 拡張機能とともに使用する

PostgreSQL 拡張機能と GitHub Copilot の両方が Visual Studio Code にインストールされたので、GitHub Copilot を PostgreSQL 開発ワークフローに挿入できるようになりました。

まず、PostgreSQL 拡張機能から GitHub Copilot を適用します。

1. [PostgreSQL 拡張機能] ビューで、新しいクエリを作成しましょう。 `rentals` データベースを右クリックし、**[新しいクエリ]** を選択します。

1. GitHub Copilot を開始するためのクエリを追加しましょう。 新しいクエリ エディターに次の SQL クエリを入力します。

    ```sql
    SELECT * FROM listings WHERE price < 100;
    ```
1. 入力した SQL クエリを強調表示し、強調表示されたテキストを右クリックして、コンテキスト メニューから **[説明]** を選択します。 [GitHub Copilot Chat] ウィンドウが開き、指定した SQL クエリの説明が表示されることに注意してください。 MCP 拡張機能のアクティブ化を許可するように求められる場合があります。 その場合は、**[許可]** を選び、GitHub Copilot がデータベースに接続して、より多くのコンテキスト応答を提供できるようにします。

もう少し興味深いものを試してみましょう。

1. クエリ エディターで、前のクエリを次の SQL クエリに置き換えますが、まだ実行しないでください。

    ```sql
    SELECT * FROM listings l, reviews r 
    WHERE l.name LIKE '%" + searchInput + "%' 
    AND r.listing_id = l.id 
    AND price < 100
    ORDER BY date;
    ```

1. 新しい SQL クエリを強調表示にし、強調表示されたテキストを右クリックして、コンテキスト メニューから **[レビュー]** を選びます。

1. この手順では、GitHub Copilot から **[コード レビュー]** 応答が返され、SQL クエリの改善方法が提案されます。

1. GitHub Copilot から提供される提案には **[適用]** または **[破棄]** を選択できます。 **[適用]** を選択して、提案された改善点を使って SQL クエリを更新します。

1. GitHub Copilot の提案に基づいて、エディターの SQL クエリがどのように更新されたかに注目してください。    

コンテキスト メニュー オプションは GitHub Copilot を使用する優れた方法ですが、本領が発揮されるのは Copilot Chat を直接使用する場合です。

1. **PostgreSQL** 拡張機能ビューで、`rentals` データベースを右クリックし、**[Copilot の接続]** を選んで Copilot Chat ペインを開きます (ペインは既に開かれているかもしれません)。

1. チャットが `rentals` データベースに接続されていることを示すメッセージが表示されるはずです。これは、GitHub Copilot がこのチャット セッションに PostgreSQL データベースのコンテキストを使っていることを示します。

1. チャット入力ボックスに、次のプロンプトを入力します。

    ```
    Generate a SQL query to find the top 5 most expensive listings.
    ```

1. クエリが生成されるだけでなく、接続されている PostgreSQL データベースに対して実行されたクエリの結果も返されていることに注目します。

1. チャット入力ボックスに、次のプロンプトを入力します。

    ```
    Run the query.
    ```

1. 生成されたクエリが実行され、結果がチャットで返されることに注意してください。

問題が発生するおそれがあるいくつかのクエリのトラブルシューティングを試してみましょう。

1. [PostgreSQL 拡張機能] ビューで、新しいクエリを作成しましょう。 `rentals` データベースを右クリックし、**[新しいクエリ]** を選択します。

1. 新しいクエリ エディターに次の SQL クエリを入力し、クエリを強調表示にしますが、実行しないでください。

    ```sql
    SELECT id, title, price FROM listings WHERE price < 100;
    ```

1. Copilot Chat ウィンドウに戻り、次のプロンプトを入力して実行します。 

    ```
    There is an error in the query in the query editor. Can you help me identify and fix it?
    ```

1. `title` 列が `listings` テーブルに存在しないことが識別され、SQL クエリの修正バージョンが提供されます。 `title` を `name` に変更することが提案されます。

1. Copilot Chat ウィンドウに戻り、次のプロンプトを入力して実行します。 

    ```
    Fix the query in the query editor and execute it.
    ```

1. 変更の `Keep` または `Undo` の確認を求められます。 **[保持]** を選んで修正を適用します。 修正したクエリが実行され、チャットで結果が返されるはずです。

もう 1 つの複雑なクエリを試してみましょう。

1. 新しいクエリ エディターで前のクエリを次の SQL クエリに置き換えて実行します (このクエリは、次の手順でトラブルシューティングを行う問題が意図的に含まれているため、最初は失敗します)。

    ```sql
    WITH listing_stats AS (
        SELECT 
            name,
            price,
            description,
            LENGTH(description) AS desc_length,
            AVG(price) OVER () AS avg_price
        FROM listings
        WHERE price IS NOT NULL
    ),
    expensive_listings AS (
        SELECT 
            name,
            price,
            desc_length,
            avg_price,
            price - avg_price AS price_diff,
            CASE 
                WHEN price > avg_price THEN 'Above Average'
                ELSE 'Below Average'
            END AS price_category
        FROM listing_stats
        WHERE price > avg_price * 1.5
    ),
    categorized AS (
        SELECT 
            price_category,
            COUNT(*) AS listing_count,
            AVG(price) AS category_avg_price,
            SUM(desc_length) AS total_desc_length
        FROM expensive_listings
        GROUP BY price_category
    )
    SELECT 
        price_category,
        listing_count,
        category_avg_price,
        total_desc_length / listing_count AS avg_desc_length,
        ROUND(listing_count / SUM(listing_count) OVER () * 100, 2) AS pct_of_total
    FROM categorized
    WHERE avg_desc_length > 50
    ORDER BY category_avg_price DESC;
    ```

    このクエリでは、リストを分析して、高価なアイテム (平均の 150% を超える価格のアイテム) を識別します。 クエリでは、"平均を超える" または "平均未満" として分類され、各カテゴリの統計が計算されます。 最後に、クエリでフィルター処理が行われ、平均の説明の長さが 50 文字を超えるカテゴリのみが表示されます。

1. 次に、Copilot Chat ウィンドウに戻り、次のプロンプトを入力して実行します。 

    ```
    This query is erroring out. Can you help me fix it and explain what I did wrong?
    ```

1. SQL クエリのエラーが特定され、修正されたバージョンと間違っていた箇所の説明が提供されます。 修正の適用とクエリの再実行を許可します。 クエリの結果は次のようになるはずです。

    | price_category | listing_count | category_avg_price | avg_desc_length | pct_of_total |
    | --------------- | -------------- | ------------------ | ---------------- | ------------- |
    | 平均より上 | 8 | 287.375 | 895 | 100.00 |

ここまでは、データ操作言語 (DML) クエリについて調べてきました。 次に、いくつかのデータ定義言語 (DDL) コマンドを試してみましょう。 新しいテーブルを作成することから始めましょう。

1. [Copilot Chat] ペインで、次のプロンプトを入力して実行します。 

    ```
    Create a new table called customers.
    ```

1. この要求は非常に一般的であるため、Copilot からより詳細なテーブル構造が要求されたり、テーブル構造が提案されたりする可能性があります。 その場合、次の詳細でプロンプトに応答します。

    ```
    The customers table should have the following columns: id (integer, primary key), first_name (varchar(50)), last_name (varchar(50)), email (varchar(100), unique), created_at (timestamp, default current_timestamp).
    ```

1. この要求はデータベース スキーマを変更する DDL コマンドであるため、Copilot はそれをすぐには実行しません。 Copilot は DDL コマンドを実行する確認を求めます。 ただし、[コマンドを許可する] を選ぶときは注意してください。 プルダウンのオプションを確認し、**[一度のみ許可]** のみを選んでこの特定のコマンドを実行します。すべての DDL コマンドにまとめてアクセス許可を与えないでください。

1. Copilot によってデータベースにテーブルが作成されます。

1. テーブルが作成されたら、[PostgreSQL 拡張機能] ビューで `public` スキーマの下にある `Tables` ノードを展開すると、それを確認できます。 新しい `customers` テーブルを表示するには、ビューの更新が必要な場合があります。

次に、`customers` テーブルを変更して新しい列を追加してみましょう。次のように指示します。 

1. [Copilot Chat] ペインで、次のプロンプトを入力して実行します。 

    ```
    Add a new column called phone_number (varchar(15)) to the customers table.
    ```

1. これは別の DDL コマンドであり、前に DDL コマンドを 1 回だけ許可することを選んだので、Copilot は再度確認を求めます。 **[一度のみ許可]** を選んで、この特定のコマンドを実行します。

1. `customers` テーブルに 10 個のサンプル レコードを追加するよう Copilot に指示し、メッセージが表示されたら変更を確認します。

    ```
    Insert 10 sample records into the customers table.
    ```

1. 最後に、新しいクエリ エディターで次の SQL クエリを実行して、レコードが正常に追加されたことを確認します。

    ```sql
    SELECT * FROM customers;
    ```
1. これは単純な DML ですが、ここまで DDL コマンドを 1 回だけ許可することを選んできた場合、Copilot はここで DML コマンドを許可する確認を求めます。 DML コマンドをセッションに対して許可することも、1 回だけ許可することもできます。 セッションに対して許可した場合、Copilot はセッションの残りの部分で確認を求めずに任意の DML コマンドを実行できます。そのため、この選択の影響をよく検討してください。

1. クエリ エディターの下にある **[結果]** ペインで結果を確認し、サンプル データが正しく挿入されたことを確認します。

DDL または DML コマンドをさらにいくつか作成してみましょう。 [Copilot Chat] ペインを使用して、SQL の生成と、接続された PostgreSQL データベースに対する実行を支援できます。

### 拡張機能エージェント モードを調べる

ここまでは、質問と回答の形式で Copilot Chat を使用することに重点を置きました。 GitHub Copilot Chat では、複数ステップのデータベース タスクを自律的に実行できるエージェント モードもサポートされています。

1. [Copilot Chat] ペインで、チャット入力ボックスの下のプルダウン メニューで **[エージェント]** がまだ選択されていない場合は選択します。

1. ***[モデル]*** プルダウン メニューを選択し、**GPT-5.2** または任意のモデルを選択します。

1. チャット入力ボックスに、次のプロンプトを入力します。

    ```
    Add a new table under the rental database for cities, make sure to include the country name, the city name and an id for the city.
    ```

1. エージェントによるワークスペースへのアクセスを許可するアクセス許可を求めるダイアログ ボックスが表示される場合があります。 続行するには **[このセッションでは許可する]** を選択します。

1. エージェントがタスクを完了するために実行する手順に注目します。 タスクを通じて動作するにつれて、アクセス許可がさらに要求される場合があります。 各手順を確認し、必要に応じてアクセス許可を提供します。

1. エージェントがタスクを完了したら、[PostgreSQL 拡張機能] ビューの `public` スキーマの下にある `Tables` ノードを展開して、`cities` テーブルが正常に作成されたことを確認します。 新しい `cities` テーブルを表示するには、ビューの更新が必要な場合があります。

1. チャット入力ボックスに、次のプロンプトを入力します。

    ```
    How could we improve this current database?
    ```

1. エージェントの応答とデータベースの改善に関する推奨事項を確認します。 推奨された改善の一部または全部を実装するようエージェントに指示し、PostgreSQL 拡張機能ビューで変更を確認したり、データベースに対して SQL クエリを実行したりできます。 


この例はシンプルなタスクでしたが、エージェントは、より複雑な複数ステップのタスクを処理できます。 エージェントにより、新しいデータベース スキーマが作成され、サンプル データの入力と、そのデータに基づくレポートを生成できました。

## クリーンアップ

この演習が完了したら、作成した Azure リソースを削除します。 課金は、データベースの使用量ではなく、構成した容量が対象になります。 リソース グループと、このラボ用に作成したすべてのリソースを削除するには、次の手順に従います。

1. Web ブラウザーを開いて [Azure portal](https://portal.azure.com/) にアクセスし、ホーム ページで [Azure サービス] の下にある **[リソース グループ]** を選択します。

    ![Azure portal の [Azure サービス] の下で赤いボックスで強調表示されている [リソース グループ] のスクリーンショット。](media/azure-portal-home-azure-services-resource-groups.png)

2. 任意のフィールドのフィルター検索ボックスに、このラボ用に作成したリソース グループの名前を入力し、一覧からリソース グループを選択します。

3. 対象のリソース グループの **[概要]** ページで、 **[リソース グループの削除]** を選択します。

    ![[リソース グループの削除] ボタンが赤い四角で強調表示されている、リソース グループの [概要] ブレードのスクリーンショット。](media/azure-portal-overview-resource-group-delete.png)

4. 確認ダイアログで、削除するリソース グループの名前を入力して確認し、**[削除]** を選択します。

## 要点

この演習では、Visual Studio Code 用 PostgreSQL 拡張機能と GitHub Copilot を使用して PostgreSQL 開発ワークフローを強化する方法を調べました。 Azure Database for PostgreSQL サーバーに接続し、サンプル データを入力し、GitHub Copilot を使用して SQL クエリの生成、調整、トラブルシューティングを行いました。 Copilot Chat 機能を使用すると、SQL の生成と変更を対話形式で要求できるため、Visual Studio Code 内で直接データベースを操作しやすくなります。 これは、これらの強力なツールで実現できることの始まりにすぎません。 シンプルなクエリから複雑なデータベース操作にいたるまで、データベースとやり取りするアプリケーション全体の作成を Copilot に依頼することができます。
