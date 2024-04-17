---
lab:
  title: SQL Server データベースを Azure SQL Database に移行する
---

# SQL Server データベースを Azure SQL Database に移行する

この演習では、Azure Data Studio 用の Azure 移行拡張機能を使用して、SQL Server データベースを Azure SQL Database に移行する方法について説明します。 まず、Azure Data Studio 用の Azure 移行拡張機能をインストールして起動します。 次に、SQL Server データベースから Azure SQL Database へのオフライン移行を実行します。 また、Azure portal で移行プロセスを監視する方法についても説明します。

この演習は約 **45** 分かかります。

> **注**: この演習を完了するには、Azure サブスクリプションにアクセスして、Azure リソースを作成する必要があります。 Azure サブスクリプションをお持ちでない場合は、始める前に[無料アカウントを作成](https://azure.microsoft.com/free/?azure-portal=true)してください。

## 開始する前に

この演習を実行するには、次のものが必要です。

| 項目 | 説明 |
| --- | --- |
| **ターゲット サーバー** | Azure SQL Database サーバー。 この演習中に作成します。|
| **ターゲット データベース** | Azure SQL Database サーバーのデータベース。 この演習中に作成します。|
| **ソース サーバー** | 好みのサーバーにインストールされている SQL Server 2019 [以降](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)のバージョン のインスタンス。 |
| **ソース データベース** | SQL Server インスタンスで復元される軽量の [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) データベース。 |
| **Azure Data Studio** | このソース データベースがあるのと同じサーバーに [Azure Data Studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio) をインストールします。 既にインストールされている場合は、最新バージョンを使用していることを確認するように更新します。 |
| **Microsoft Data Migration Assistant** | このソース データベースがあるのと同じサーバーに [Data Migration Assistant](https://www.microsoft.com/en-us/download/details.aspx?id=53595) をインストールします。 |
| **Microsoft.DataMigration** リソース プロバイダー | サブスクリプションが名前空間 **Microsoft.DataMigration** を使用するように登録されていることを確認します。 リソース プロバイダーの登録を実行する方法については、「[リソース プロバイダーの登録](https://learn.microsoft.com/azure/dms/quickstart-create-data-migration-service-portal#register-the-resource-provider)」を参照してください。 |
| **Microsoft Integration Runtime** | [Microsoft Integration Runtime](https://aka.ms/sql-migration-shir-download) をインストール します。 |

## SQL Server データベースを復元する

*AdventureWorksLT* データベースを SQL Server インスタンスに復元しましょう。 このデータベースは、このラボ演習のソース データベースとして機能します。 データベースが既に復元されている場合は、これらの手順をスキップできます。

1. Windows の [スタート] ボタンを選択し、SSMS と入力します。 一覧から **[Microsoft SQL Server Management Studio 18]** を選択します。  

1. SSMS が開くと、 **[サーバーに接続]** ダイアログに既定のインスタンス名が事前に入力されていることがわかります。 **[接続]** を選択します。

1. **Databases** フォルダーを選択し、**[新しいクエリ]** を選択します。

1. 次の T-SQL をコピーして、新しいクエリ ウィンドウに貼り付けます。 データベース バックアップ ファイルの名前とパスが実際のバックアップ ファイルと一致していることを確認します。 していない場合、コマンドは失敗します。 クエリを実行してデータベースを復元します。

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\<FolderName>\AdventureWorksLT2019.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2019_Data' 
            TO 'C:\<FolderName>\AdventureWorksLT2019.mdf',
          MOVE 'AdventureWorksLT2019_Log'
            TO 'C:\<FolderName>\AdventureWorksLT2019.ldf';
    ```

    > **注**:T-SQL コマンドを実行する前に、SQL Server マシンに軽量の [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) バックアップ ファイルがあることを確認してください。

1. 復元が完了すると、成功メッセージが表示されます。

## Microsoft.DataMigration 名前空間を登録する

Microsoft.DataMigration** 名前空間がサブスクリプションに**既に登録されている場合は、これらの手順をスキップします。

1. Azure portal で、上部にある検索ボックスで**サブスクリプション**を検索し、**[サブスクリプション]** を選択します。 **[サブスクリプション]** ブレードで、自分のサブスクリプションを選択します。

1. [サブスクリプション] ページの **[設定]** で、**[リソース プロバイダー]** を選択します。 上部の検索ボックスで **Microsoft.DataMigration** を検索し、**Microsoft.DataMigration** を選択します。 

    > **注**: [リソース プロバイダーの詳細] **** サイド バーが開いた場合は、それを閉じます。

1. **[登録]** を選択します。

## Azure SQL Database をプロビジョニングする

ターゲット環境として機能する Azure SQL Database を設定しましょう。

1. Azure portal で、上部にある検索ボックスで **SQL データベース**を検索し、**[SQL データベース]** を選択します。

1. **[SQL データベース]** ブレードで、**[+ 作成]** を選択します。

1. **[SQL Database の作成]** ページで 、次のオプションを選択します。

    - **サブスクリプション** &lt;お使いのサブスクリプション&gt;
    - **リソース グループ:** &lt;お使いのリソース グループ&gt;
    - **データベース名**: AdventureWorksLT
    - **サーバー:** **[新規作成]** リンクを選択します。 **[SQL Database Server の作成]** ページでサーバーの詳細を指定します。
        - **サーバー名:** &lt;サーバー名を選択します&gt;。 サーバー名はグローバルに一意である必要があります。
        - **場所:** &lt;リソース グループと同じリージョン&gt;
        - **認証方法:** SQL 認証を使用する
        - **サーバー管理者ログイン:** sqladmin
        - **パスワード:** &lt;お使いのパスワード&gt;
        - **パスワードの確認:** &lt;お使いのパスワード&gt;
        
            **注:** このサーバー名と資格情報を書き留めます。 これは後のタスクで使用します。

    - **SQL エラスティック プールを使用しますか?** いいえ
    - **ワークロード環境:** 実稼働

1. **[コンピューティングとストレージ]** で、**[データベースの構成]** を選択します。 **[構成]** ページの **[サービス レベル]** ドロップダウンで、 **[基本]**、**[適用]** の順に選択します。

1. **[バックアップ ストレージの冗長性]** オプションについては、既定値の **[geo 冗長バックアップ ストレージ]** のままにします。 **[次へ: ネットワーク]** を選択します。

1. **[ネットワーク]** タブで、**[次へ: セキュリティ]**、**[次へ: 追加設定]** の順に選択します。

1. **[追加設定]** ページで、**[確認および作成]** を選択します。

1. 設定を確認して、**[作成]** を選択します。

1. デプロイが完了したら、**[リソースに移動]** を選択します。

## Azure SQL Database へのアクセスを有効にする

クライアント ツールを使用して Azure SQL Database に接続できるように、Azure SQL Database へのアクセスを有効にしましょう。

1. **[SQL データベース]** ページで **[概要]** セクションを選択し、上部セクション内のサーバー名のリンクを選択します。

1. **[SQL サーバー]** ブレードで、**[セキュリティ]** セクションの **[ネットワーク]** を選択します。

1. **[パブリック アクセス]** タブで、**[選択されたネットワーク]** を選択します。 

1. **[ファイアウォール ルール]** で、**[+ クライアント IPv4 アドレスの追加]** を選択します。

1. **[例外]** で、**[Azure サービスおよびリソースにこのサーバーへのアクセスを許可する]** プロパティのチェックボックスをオンにします。 

1. **[保存]** を選択します。

## Azure Data Studio で Azure SQL Database に接続する

Azure 移行拡張機能の使用を開始する前に、ターゲット データベースに接続しましょう。

1. Azure Data Studio を起動します。

1. **[接続]**、**[接続の追加]** の順に選択します。

1. **[接続の詳細] ** フィールドに SQL Server 名とその他の情報を入力します。

    > **注**: 前もって作成した SQL Server の名前を入力します。 形式は **<server>.database.windows.net** の形式である必要があります。

1. **[認証の種類]** で、**[SQL ログイン]** を選択し、ユーザー名とパスワードを指定します。

1. **[接続]** を選択します。

## Azure Data Studio 用の Azure 移行拡張機能をインストールして起動する

移行拡張機能をインストールするには、次の手順に従います。 拡張機能が既にインストールされている場合は、これらの手順をスキップできます。

1. Azure Data Studio で拡張機能マネージャーを開きます。

1. 「***Azure SQL 移行***」と検索して、この拡張機能を選択します。

1. 拡張機能をインストールします。 インストールすると、インストール済みの拡張機能の一覧に Azure SQL 移行拡張機能が表示されます。

1. Azure Data Studio で SQL Server インスタンスに接続します。 [新しい接続] タブで、**[暗号化]** オプションに **[オプション]** (False) を選択します。

1. Azure 移行拡張機能を起動するには、SQL Server インスタンス名を右クリックし、**[管理]** を選択して、Azure SQL 移行拡張機能のダッシュボードとランディング ページにアクセスします。

    > **注**: **Azure SQL 移行**がサーバー ダッシュボードのサイド バーに表示されない場合は、Azure Data Studio をもう一度開きます。
 
## SQL Server データベースから Azure SQL Database へのオフライン移行を実行する

これで、データを移行する準備ができました。 Azure Data Studio を使用してオフライン移行を実行するには、次の手順に従います。

1. Azure Data Studio の拡張機能で **Azure SQL への移行**ウィザードを起動し、**[Azure SQL への移行]** を選択します。

1. **[手順 1: 評価用データベース]** で、*AdventureWorks* データベースを選んでから、**[次へ]** を選択します。

1. **[手順 2: 評価の概要と SKU の推奨事項]** で、評価が完了するまで待ち、結果を確認します。 [**次へ**] を選択します。

1. **[手順 3: ターゲット プラットフォームと評価結果]** で、データベースを選択して評価結果を確認します。

    > **注**: 少し時間を取って、右側の評価結果を確認してください。

1. **[手順 3:ターゲット プラットフォームと評価結果]** ページの上部で、**Azure SQL** ターゲットとして **[Azure SQL Database]** を選びます。

1. **[手順 4:Azure SQL ターゲット]** で、アカウントがまだリンクされていない場合は、**[アカウントのリンク]** リンクを選択してアカウントを追加してください。 次に、Azure アカウント、AD テナント、サブスクリプション、場所、リソース グループ、Azure SQL Database サーバー、Azure SQL Database の資格情報を選択します。

1. **[接続]** を選択し、*AdventureWorks* データベースを**ターゲット データベース**として選択します。 [**次へ**] を選択します。

1. **[手順 5:Azure Database Migration Service]** で、**[新規作成]** リンクを選び、ウィザードを使用して新しい Azure Database Migration Service を作成します。 ウィザードで提供される手順に従って、新しいセルフホステッド統合ランタイムを設定します。 以前に作成してある場合は、再利用できます。

1. **[手順 6:データ ソースの構成]** で、セルフホステッド統合ランタイムから SQL Server インスタンスに接続するための資格情報を入力します。 

1. ソースからターゲットに移行するすべてのテーブルを選び、**[不足しているスキーマの移行]** オプションをオンにします。

1. **[検証の実行]** を選択します。

    ![Azure Data Studio 用の Azure 移行拡張機能の検証手順実行のスクリーンショット。](../media/3-run-validation.png) 

1. 検証が完了したら、**[次へ]** を選択します。

1. **[手順 7:概要]** で、**[移行の開始]** を選びます。

1. 移行ダッシュボードで **[データベースの移行が進行中]** を選択して、移行の状態を表示します。 

    ![Azure Data Studio 用の Azure 移行拡張機能の移行ダッシュボードのスクリーンショット。](../media/3-data-migration-dashboard.png)

1. *AdventureWorks* データベース名を選択すると、詳細が表示されます。

    ![Azure Data Studio 用の Azure 移行拡張機能の移行詳細のスクリーンショット。](../media/3-dashboard-sqldb.png)

1. 状態が **[成功]** になったら、ターゲット サーバーに移動し、ターゲット データベースを検証します。 データベース スキーマと移行したデータを確認します。

移行拡張機能をインストールし、Data Migration Assistant を使用してデータベースのスキーマを生成する方法を説明しました。 また、Azure Data Studio 用の Azure 移行拡張機能を使用して、SQL Server データベースを Azure SQL Database に移行する方法についても説明しました。 移行が完了したら、新しい Azure SQL Database リソースの使用を開始できます。 

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細情報

Azure SQL Database については、「[Azure SQL Database とは](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview)」を参照してください。
