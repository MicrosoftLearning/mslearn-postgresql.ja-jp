---
lab:
  title: オンライン PostgreSQL データベース移行
  module: Migrate to Azure Database for PostgreSQL Flexible Server
---

## オンライン PostgreSQL データベース移行

この演習では、オンライン移行アクティビティを実行できるように、ソース PostgreSQL サーバーと Azure Database for PostgreSQL フレキシブル サーバーの間で論理レプリケーションを構成します。

## 開始する前に

この演習を完了するには、自分の Azure サブスクリプションが必要です。 Azure サブスクリプションをお持ちでない場合は、[Azure 無料試用版](https://azure.microsoft.com/free)を作成できます。

> **注:**: この演習では、移行のソースとして使用するサーバーがデータベースに接続して移行できるように、そのサーバーが Azure Database for PostgreSQL フレキシブル サーバーにアクセスできる必要があります。 このためには、ソース サーバーがパブリック IP アドレスおよびポートを介してアクセスできる必要があります。 > Azure リージョンの IP アドレスの一覧を「[Azure IP Ranges and Service Tags – Public Cloud](https://www.microsoft.com/en-gb/download/details.aspx?id=56519)」からダウンロードすることで、使用される Azure リージョンに基づいてファイアウォール規則で許可される IP アドレスの範囲を最小限に抑えることができます。

サーバーのファイアウォールを開いて、Azure Database for PostgreSQL フレキシブル サーバー内の移行機能によるソース PostgreSQL サーバー (既定で TCP ポート 5432) へのアクセスを許可してください。
>
ソース データベースの前でファイアウォール アプライアンスを使用している場合は、Azure Database for PostgreSQL フレキシブル サーバー内の機能が移行のためにソース データベースにアクセスすることを許可するファイアウォール規則を追加することが必要な場合があります。
>
> 移行がサポートされている PostgreSQL の最大バージョンはバージョン 16 です。

### 前提条件

> **注**: この演習を開始する前に、前の演習を完了して、ソース データベースとターゲット データベースを配置して論理レプリケーションを構成する必要があります。この演習はそのアクティビティに基づいています。

## パブリケーションの作成 - ソース サーバー

1. PGAdmin を開き、Azure Database for PostgreSQL フレキシブル サーバーへのデータ同期のソースとして機能するデータベースを含むソース サーバーに接続します。
1. 同期するデータを使用して、ソース データベースに接続された新しいクエリ ウィンドウを開きます。
1. ソース サーバーの wal_level を **logical** に構成して、データのパブリケーションを許可します。
    1. PostgreSQL インストール ディレクトリ内の bin ディレクトリで、**postgresql.conf** ファイルを見つけて開きます。
    1. 構成設定 **wal_level** を含む行を見つけます。
    1. その行にコメントが付いていないことを確認し、値を **logical** に設定します。
    1. ファイルを保存して閉じます。
    1. PostgreSQL サービスを再起動します。
1. 次に、データベース内のすべてのテーブルを含むパブリケーションを構成します。

    ```SQL
    CREATE PUBLICATION migration1 FOR ALL TABLES;
    ```

## サブスクリプションの作成 - ターゲット サーバー

1. PGAdmin を開き、ソース サーバーからのデータ同期のターゲットとして機能するデータベースを含む Azure Database for PostgreSQL フレキシブル サーバーに接続します。
1. 同期するデータを使用して、ソース データベースに接続された新しいクエリ ウィンドウを開きます。
1. ソース サーバーへのサブスクリプションを作成します。

    ```sql
    CREATE SUBSCRIPTION migration1
    CONNECTION 'host=<source server name> port=<server port> dbname=adventureworks application_name=migration1 user=<username> password=<password>'
    PUBLICATION migration1
    WITH(copy_data = false)
    ;    
    ```

1. テーブル レプリケーションの状態を確認します。

    ```SQL
    SELECT PT.schemaname, PT.tablename,
        CASE PS.srsubstate
            WHEN 'i' THEN 'initialize'
            WHEN 'd' THEN 'data is being copied'
            WHEN 'f' THEN 'finished table copy'
            WHEN 's' THEN 'synchronized'
            WHEN 'r' THEN ' ready (normal replication)'
            ELSE 'unknown'
        END AS replicationState
    FROM pg_publication_tables PT,
            pg_subscription_rel PS
            JOIN pg_class C ON (C.oid = PS.srrelid)
            JOIN pg_namespace N ON (N.oid = C.relnamespace)
    ;
    ```

## データ レプリケーションのテスト

1. ソース サーバーで、ワークオーダー テーブルの行数を確認します。

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. ターゲット サーバーで、ワークオーダー テーブルの行数を確認します。

    ```SQL
    SELECT COUNT(*) FROM production.workorder;
    ```

1. 行数の値が一致することを確認します。
1. 次に、[ここ](https://github.com/MicrosoftLearning/mslearn-postgresql/tree/main/Allfiles/Labs/11)にあるリポジトリから Lab11_workorder.csv ファイルを C:\ にダウンロードします。
1. 次のコマンドを使用して、CSV からソース サーバーのワークオーダー テーブルに新しいデータを読み込みます。

    ```Bash
    psql --host=localhost --port=5432 --username=postgres --dbname=adventureworks --command="\COPY production.workorder FROM 'C:\Lab11_workorder.csv' CSV HEADER"
    ```

コマンド出力は、CSV ファイルから 490 行がテーブルに追加で書き込まれたことを示す `COPY 490` となるはずです。

1. ソース (72591 行) とターゲットのワークオーダー テーブルの行数が一致するかを確認して、データ レプリケーションが機能していることを確認します。

## 演習のクリーンアップ

この演習でデプロイした Azure Database for PostgreSQL には料金が発生しますが、この演習の後にサーバーを削除できます。 または、**rg-learn-work-with-postgresql-eastus** リソース グループを削除して、この演習の一環としてデプロイしたすべてのリソースを削除することもできます。
