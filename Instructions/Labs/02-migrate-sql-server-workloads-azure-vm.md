---
lab:
  title: SQL Server データベースを Azure 仮想マシン上の SQL Server に移行する
---

# SQL Server データベースを Azure 仮想マシン上の SQL Server に移行する

この演習では、Azure 仮想マシンで実行されている SQL Server への SQL Server データベースの移行を、Azure Blob Storage を介したバックアップと復元という方法で行う方法を学習します。 ソース データベースを Azure ストレージ アカウントにバックアップし、これを Azure VM 上のターゲット SQL Server に復元します。 この単純明快なオフライン移行アプローチは、データベースが小さい場合や、ダウンタイムが許容される場合に適しています。

この演習には約 **45** 分かかります。

> **注**: この演習を完了するには、Azure サブスクリプションにアクセスして、Azure リソースを作成する必要があります。 Azure サブスクリプションをお持ちでない場合は、始める前に[無料アカウントを作成](https://azure.microsoft.com/free/?azure-portal=true)してください。

## 開始する前に

この演習を実行するには、次のものが必要です。

| 項目 | 説明 |
| --- | --- |
| **ターゲット サーバー** | Azure 仮想マシン上の SQL Server。 詳細については、[Azure 仮想マシンで SQL Server をプロビジョニングする](https://microsoftlearning.github.io/dp-300-database-administrator/Instructions/Labs/01-provision-sql-vm.html)を参照してください。 **注:** ターゲットとサーバーの間の SQL Server のバージョンは同じである必要があります。 |
| **ソース サーバー** | 選択したサーバーにインストールされている最新の [SQL Server](https://www.microsoft.com/en-us/sql-server/sql-server-downloads) のバージョン。 |
| **ソース データベース** | SQL Server インスタンスで復元される軽量の [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) データベース。 |
| **SSMS** | [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) をソースとターゲット両方のサーバーにインストールします。これをデータベースのバックアップ、復元、確認に使用します。 |

## SQL Server データベースを復元する

*AdventureWorksLT* データベースを SQL Server インスタンスに復元しましょう。 このデータベースは、このラボ演習のソース データベースとして機能します。 データベースが既に復元されている場合は、これらの手順をスキップできます。

1. Windows の [スタート] ボタンを選択し、SSMS と入力します。 一覧から **[Microsoft SQL Server Management Studio 18]** を選択します。  

1. SSMS が開くと、 **[サーバーに接続]** ダイアログに既定のインスタンス名が事前に入力されていることがわかります。 **[接続]** を選択します。

1. **Databases** フォルダーを選択し、**[新しいクエリ]** を選択します。

1. 次の T-SQL をコピーして、新しいクエリ ウィンドウに貼り付けます。 データベース バックアップ ファイルの名前とパスが実際のバックアップ ファイルと一致していることを確認します。 このとおりでない場合は、このコマンドは失敗します。 クエリを実行してデータベースを復元します。

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

## BLOB コンテナーがある Azure ストレージ アカウントをプロビジョニングする

Azure ストレージ アカウントを作成する目的は、移行用の完全バックアップとトランザクション ログ バックアップを格納することです。 このストレージ アカウントは後の手順で使用します。

1. [Azure portal](https://portal.azure.com/) にサインインします。
1. 左側のポータル メニューで **[ストレージ アカウント]** を選択して、ストレージ アカウントの一覧を表示します。 ポータル メニューが表示されない場合は、メニュー ボタンを選択してオンに切り替えます。
1. **[ストレージ アカウント]** ページで、**[作成]** を選択します。
1. **[プロジェクトの詳細]** で、Azure 仮想マシンを作成したのと同じ Azure サブスクリプションを選択します。
1. Azure 仮想マシンを作成したのと同じリソース グループを選択します。 
1. ストレージ アカウントの一意の名前を選択し、Azure 仮想マシンと同じリージョンを選択します。
1. サービス レベルとして **[Standard]** を選択します。
1. 残りのオプションについては既定値のままにします。
1. **[確認および作成]** を選択し、次に **[作成]** を選択します。

ストレージ アカウントが作成されたら、次の手順に従ってコンテナーを作成できます。

1. Azure portal で新しく作成されたストレージ アカウントに移動します。
1. ストレージ アカウントの左側のメニューで、**[BLOB サービス]** セクションまでスクロールし、**[コンテナー]** を選択します。
1. **[+ コンテナー]** を選択して、新しいコンテナーを作成します。
1. [新しいコンテナー側] ページで次の情報を入力します。
    - **名前:** *ユーザー設定の名前*
    - **パブリック アクセス レベル:** 非公開
1. **［作成］** を選択します

## ストレージ アカウントの SQL Server 資格情報を作成する

ソース データベースを Azure Blob Storage にバックアップするには、ストレージ アカウントのアクセス キー情報を格納する資格情報をあらかじめ作成する必要があります。

1. SSMS を開いて**ソース**の SQL Server インスタンスに接続します。

1. **[新しいクエリ]** ウィンドウを開き、次に示す T-SQL ステートメントを実行して資格情報を作成します。 プレースホルダーの値を実際のストレージ アカウント名とアクセス キーで置き換えてください。

    ```sql
    CREATE CREDENTIAL [https://<storage_account_name>.blob.core.windows.net/<container_name>]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<SAS token>';
    ```

    > **注:**  SAS トークンを生成するには、Azure portal で自分のストレージ アカウントに移動し、**[セキュリティとネットワーク]** の下の **[Shared Access Signature]** を選択し、**[Blob]** と **[コンテナー]** にアクセス許可として **[読み取り]**、**[書き込み]**、**[リスト]**、**[作成]** を付与し、適切な有効期限を設定してから **[SAS と接続文字列を生成する]** を選択します。 **[SAS トークン]** の値をコピーします (先頭の `?` を除く)。

## ソース データベースを Azure Blob Storage にバックアップする

次に、SQL Server の URL へのバックアップ機能を使用してソース データベースを Azure Blob Storage コンテナーに直接バックアップします。

1. SSMS で、**ソース**の SQL Server に接続し、**[新しいクエリ]** ウィンドウを開いて次に示す T-SQL ステートメントを実行します。

    ```sql
    BACKUP DATABASE AdventureWorksLT
    TO URL = 'https://<storage_account_name>.blob.core.windows.net/<container_name>/AdventureWorksLT.bak'
    WITH COMPRESSION, STATS = 10;
    ```

    > **注:**  `<storage_account_name>` と `<container_name>` を実際の値で置き換えてください。 `COMPRESSION` オプションを指定するとバックアップのサイズが小さくなり、転送に要する時間が短くなります。

1. バックアップが正常に完了したことを確認します。 データベースが正常にバックアップされたことを示すメッセージが表示されるはずです。

1. バックアップ ファイルが存在するかどうかを確認するには、Azure portal で自分のストレージ アカウント コンテナーに移動します。

## データベースを Azure VM 上のターゲット SQL Server 上に復元する

次に、バックアップ ファイルを Azure Blob Storage から、Azure 仮想マシン上で実行されているターゲット SQL Server に復元します。

1. SSMS を開き、Azure VM 上の**ターゲット** SQL Server に接続します。

1. **[新しいクエリ]** ウィンドウを開き、次に示す T-SQL ステートメントを実行して同じ資格情報をターゲット サーバー上に作成します。

    ```sql
    CREATE CREDENTIAL [https://<storage_account_name>.blob.core.windows.net/<container_name>]
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
    SECRET = '<SAS token>';
    ```

    > **注:**  前に生成した、同じ SAS トークンを使用します。 有効期限が切れていないことを確認してください。

1. 次に示す T-SQL ステートメントを実行してデータベースをバックアップ ファイルから復元します。

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM URL = 'https://<storage_account_name>.blob.core.windows.net/<container_name>/AdventureWorksLT.bak'
    WITH REPLACE, STATS = 10;
    ```

    > **注:**  `REPLACE` オプションを指定すると、同じ名前の既存のデータベースが上書きされます。 データベースがターゲット上に存在しない場合は、このオプションを省略してもかまいません。 データベースのサイズとネットワーク速度によっては、復元に数分かかることがあります。

1. 復元が正常に完了したことを確認します。 データベースが正常に復元されたことを示すメッセージが表示されるはずです。

## 移行を検証する

1. SSMS で、Azure VM 上の**ターゲット** SQL Server に接続し、**[オブジェクト エクスプローラー]** で **[データベース]** を展開して **AdventureWorksLT** データベースが表示されることを確認します。

1. **AdventureWorksLT** データベースを右クリックして **[新しいクエリ]** を選択します。 データが正しく移行されたことを確認するために、次のサンプル クエリを実行します。

    ```sql
    -- Verify table count
    SELECT COUNT(*) AS TableCount
    FROM AdventureWorksLT.INFORMATION_SCHEMA.TABLES
    WHERE TABLE_TYPE = 'BASE TABLE';

    -- Verify row counts for a sample table
    SELECT COUNT(*) AS RowCount
    FROM AdventureWorksLT.SalesLT.Customer;
    ```

1. 同じクエリをソース データベースに対して実行したときの結果と比較してデータ整合性を確認します。

これで、Azure 仮想マシン上で実行されている SQL Server に、Azure Blob Storage を介したバックアップと復元という方法を使用して SQL Server データベースを移行する作業が完了しました。 この方法では、SQL Server のネイティブの機能である URL へのバックアップを使用してデータベースが Azure ストレージ アカウントを通して転送されるため、オフライン移行をシンプルかつ効率的に行うことができます。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細

Azure 仮想マシンの SQL Server の詳細については、「[Azure 仮想マシンにおける SQL Server の概要](https://learn.microsoft.com/en-us/azure/azure-sql/virtual-machines/windows/sql-server-on-azure-vm-iaas-what-is-overview?view=azuresql-vm)」を参照してください。
