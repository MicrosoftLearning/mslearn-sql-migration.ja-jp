---
lab:
  title: Log Replay Service を使用して SQL Server データベースを Azure SQL Managed Instance に移行する
---

# Log Replay Service を使用して SQL Server データベースを Azure SQL Managed Instance に移行する

この演習では、Log Replay Service を使用して SQL Server データベースを Azure SQL Managed Instance に移行する方法を学習します。 

最初に、Azure SQL Managed Instance をデプロイします。 次に、Log Replay Service を使用して、SQL Server データベースの Azure SQL Managed Instance へのオンライン移行を実行します。 また、PowerShell で移行プロセスを監視する方法についても学習します。

この演習は約 **45** 分かかります。

> **注**: この演習を完了するには、Azure サブスクリプションにアクセスして、Azure リソースを作成する必要があります。 Azure サブスクリプションをお持ちでない場合は、始める前に[無料アカウントを作成](https://azure.microsoft.com/free/?azure-portal=true)してください。

## 開始する前に

この演習を実行するには、次のものが必要です。

| 項目 | 説明 |
| --- | --- |
| **ターゲット サーバー** | Azure SQL Managed Instance。 この演習中に作成します。|
| **ソース サーバー** | 好みのサーバーにインストールされている SQL Server 2019 [以降](https://www.microsoft.com/en-us/sql-server/sql-server-downloads)のバージョン のインスタンス。 |
| **ソース データベース** | SQL Server インスタンスで復元される軽量の [AdventureWorks](https://learn.microsoft.com/sql/samples/adventureworks-install-configure) データベース。 |
| **[Microsoft SQL Server Management Studio]** | [Microsoft SQL Server Management Studio](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) を、ソース データベースが配置されているのと同じサーバーにインストールします。 既にインストールされている場合は、最新バージョンを使用していることを確認するように更新します。 |

## SQL Server データベースを復元する

*AdventureWorksLT* データベースを SQL Server インスタンスに復元しましょう。 このデータベースを、このラボ演習のソース データベースとして使います。 データベースが既に復元されている場合は、これらの手順をスキップできます。

1. Windows の [スタート] ボタンを選択し、SSMS と入力します。 一覧から **[Microsoft SQL Server Management Studio]** を選びます。  

1. SSMS が開いたら、**[サーバーへの接続]** ダイアログに既定のインスタンス名が事前に設定されていることに注意してください。 **[接続]** を選択します。

1. **Databases** フォルダーを選択し、**[新しいクエリ]** を選択します。

1. 新しいクエリ ウィンドウで、次の T-SQL をコピーして貼り付けます。 クエリを実行してデータベースを復元します。

    ```sql
    RESTORE DATABASE AdventureWorksLT
    FROM DISK = 'C:\LabFiles\AdventureWorksLT2019.bak'
    WITH RECOVERY,
          MOVE 'AdventureWorksLT2019_Data' 
            TO 'C:\LabFiles\AdventureWorksLT2019.mdf',
          MOVE 'AdventureWorksLT2019_Log'
            TO 'C:\LabFiles\AdventureWorksLT2019.ldf';
    ```

    > **注**: 上記の例のデータベース バックアップ ファイルの名前とパスが実際のバックアップ ファイルと一致していることを確認します。 そうでない場合は、コマンドが失敗する可能性があります。

1. 復元が完了すると、成功メッセージが表示されます。

## Azure SQL Managed Instance をデプロイする

次の手順のようにして、Azure SQL マネージド インスタンスを作成します。

1. [Azure portal](https://portal.azure.com/learn.docs.microsoft.com?azure-portal=true) にサインインし、左上隅にある **[リソースの作成]** を選択します。
1. **[マネージド インスタンス]** を探し、**[Azure SQL Managed Instance]** を選択し、その後 **[作成]** を選択します。
1. 次の表の情報を参考にして、SQL Managed Instance フォームに必要事項を入力します。

    |  | 推奨値 |
    |---|---|
    | **サブスクリプション** | 該当するサブスクリプション。 |
    | **マネージド インスタンスの名前** | 有効な名前。 |
    | **マネージド インスタンスの管理者ログイン** | 任意の有効なユーザー名。 "serveradmin" は予約済みのサーバー レベルのロールであるため、使用しないでください。 |
    | **パスワード** | 16 文字より長く、複雑さの要件を満たす任意のパスワード。 |
    | **タイム ゾーン** | マネージド インスタンスで監視するタイム ゾーン。 |
    | **Collation** | マネージド インスタンスに使用する照合順序。 SQL Server からデータベースを移行する場合は、SELECT SERVERPROPERTY(N'Collation') を使用してソースの照合順序を確認し、その値を使用してください。 |
    | **場所** | マネージド インスタンスを作成する Azure リージョン。 |
    | **仮想ネットワーク** | [新しい仮想ネットワークの作成] または有効な仮想ネットワークとサブネットを選択します。 |
    | **パブリック エンドポイントを有効にする** | このオプションをオンにするとパブリック エンドポイントが有効になり、Azure の外部のクライアントがデータベースにアクセスできます。 |
    | **許可するアクセス元** | Azure サービス、インターネット、またはアクセスなしから選択します。 |
    | **接続の種類** | 接続の種類として、プロキシまたはリダイレクトを選択します。 |
    | **リソース グループ** | 新規または既存のリソース グループ。 |

1. **[価格レベル]** を選択して、コンピューティング リソースとストレージ リソースのサイズを指定し、価格レベルのオプションを確認します。 
1. 操作が完了したら、**[適用]** を選択して選択内容を保存し、**[作成]** を選択してマネージド インスタンスをデプロイします。
1. **[通知]** アイコンを選択してデプロイの状態を表示します。
1. **[デプロイを実行しています]** を選択して [マネージド インスタンス] ウィンドウを開き、デプロイの進行状況を詳しく監視します。

## Azure Blob Storage アカウントとコンテナーを作成する

お使いの Azure SQL マネージド インスタンスと同じリージョンに Azure Blob Storage アカウントを作成します。 ここに、移行のためのデータベース バックアップを格納します。

1. [Azure portal](https://portal.azure.com) に移動し、自分のアカウント資格情報を使ってサインインします。
1. 左側のメニューで、**[すべてのサービス]** を選んで [ストレージ アカウント] を見つけます。** **[ストレージ アカウント]** を選んでストレージ アカウントのページを開きます。
1. [ストレージ アカウント] ページで、**[+ 追加]** を選んで新しいストレージ アカウントを作成します。
1. **[ストレージ アカウントの作成]** ページの **[基本]** タブで、ストレージ アカウントに使うサブスクリプションを選びます。 次に、お使いの Azure SQL マネージド インスタンスを含むリソース グループを選びます。
1. ストレージ アカウントに一意の名前を入力します。 
    
    > **注:**  名前は、3 文字から 24 文字の長さにする必要があり、小文字と数字のみを使用できます。

1. お使いの Azure SQL マネージド インスタンスがある場所 (リージョン) を選びます。
1. ストレージ アカウントのパフォーマンスレベルを選びます。
1. ストレージ アカウントのアカウントの種類として、**[BlobStorage]** を選びます。 
1. ストレージ アカウントのレプリケーション オプションとして、**[ローカル冗長ストレージ (LRS)]** を選びます。
1. 確認し、**[確認と作成]** を選んでストレージ アカウントを作成します。
1. ストレージ アカウントが作成されたら、ストレージ アカウントのページに移動して、左側のメニューで **[コンテナー]** オプションを選びます。 次に、**[+ コンテナー]** を選んで新しいコンテナーを作成します。 コンテナーの名前を入力し、パブリック アクセス レベルを選びます。 
1. **[作成]** ボタンを選んでコンテナーを作成します。

以上の手順を完了すると、お使いの Azure SQL マネージド インスタンスと同じリージョンに、Azure Blob Storage アカウントと、移行用のデータベース バックアップを格納できるコンテナーが作成されます。

## SQL Server データベースをバックアップする

SQL Server インスタンス上の *AdventureWorksLT* データベースの完全バックアップを作成した後、`CHECKSUM` を有効にして差分バックアップとログ バックアップを作成しましょう。 

1. Windows の [スタート] ボタンを選択し、SSMS と入力します。 一覧から **[Microsoft SQL Server Management Studio 18]** を選択します。  
1. SSMS が開いたら、**[サーバーへの接続]** ダイアログに既定のインスタンス名が事前に設定されていることに注意してください。 **[接続]** を選択します。
1. **Databases** フォルダーを選択し、**[新しいクエリ]** を選択します。
1. 次の T-SQL をコピーして、新しいクエリ ウィンドウに貼り付けます。 クエリを実行してデータベースを復元します。

    ```sql
    BACKUP DATABASE AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_full.bak'
    WITH CHECKSUM;

    BACKUP DATABASE AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_diff.dif'
    WITH DIFFERENTIAL, CHECKSUM;

    BACKUP LOG AdventureWorksLT
    TO DISK = 'C:\LabFiles\AdventureWorksLT_log.trn'
    WITH CHECKSUM;
    ```

    > **注**:上の例のファイル パスが実際のファイル パスと一致していることを確認します。 そうでない場合は、コマンドが失敗する可能性があります。

1. 復元が完了すると、成功メッセージが表示されます。
1. SQL Server のバージョン (SQL Server 2012 SP1 CU2 以降および SQL Server 2014) を実行している場合は、ネイティブ SQL Server の `BACKUP TO URL` オプションを使って、SQL Server から Blob Storage アカウントに直接バックアップを作成できます。 

    ```sql
    CREATE CREDENTIAL [https://<mystorageaccountname>.blob.core.windows.net/<containername>] 
    WITH IDENTITY = 'SHARED ACCESS SIGNATURE',  
    SECRET = '<SAS_TOKEN>';  
    GO
    
    -- Take a full database backup to a URL
    BACKUP DATABASE [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_full.bak'
    WITH INIT, COMPRESSION, CHECKSUM
    GO
    
    -- Take a differential database backup to a URL
    BACKUP DATABASE [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_diff.bak'  
    WITH DIFFERENTIAL, COMPRESSION, CHECKSUM
    GO
    
    -- Take a transactional log backup to a URL
    BACKUP LOG [AdventureWorksLT]
    TO URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<databasefolder>/AdventureWorksLT_log.trn'  
    WITH COMPRESSION, CHECKSUM
    ```

    > **注:**  このオプションを使う場合は、次の「**バックアップ ファイルを Azure ストレージ アカウントにコピーする**」セクションをスキップできます。

## バックアップ ファイルを Azure ストレージ アカウントにコピーする

それでは、バックアップ ファイルを先ほど作成した Azure Blob Storage アカウントにコピーしましょう。

1. [Azure portal](https://portal.azure.com) に移動し、自分のアカウント資格情報を使ってサインインします。
1. 左側のメニューで **[ストレージ アカウント]** を選んでから、前に作成したストレージ アカウントを選びます。
1. ストレージ アカウントの概要ページで、**[Blob service]** セクションまで下にスクロールして、**[コンテナー]** を選びます。 前に作成したコンテナーを選びます。
1. コンテナー ページの上部にある **[アップロード]** を選びます。 **[BLOB のアップロード]** ページで、**[フォルダー]** を選んでバックアップ ファイルが含まれるフォルダーを選ぶか、**[ファイル]** を選んでバックアップ ファイルを個別に選びます。 ファイルを選んだら、**[アップロード]** を選んでアップロード プロセスを始めます。

## アクセスを検証する

お使いの SQL Server と SQL マネージド インスタンス が Blob Storage アカウントに正常にアクセスできるかどうかを検証することが重要です。 ここでは、サンプル テスト クエリを実行して、お使いのマネージド インスタンスがコンテナー内のバックアップにアクセスできるかどうかを確認します。

1. SSMS 経由でお使いの SQL マネージド インスタンスに接続します。
1. 新しいクエリ エディターを開き、コマンドを実行します。

```sql
CREATE CREDENTIAL [https://<mystorageaccountname>.blob.core.windows.net/databases] 
WITH IDENTITY = 'SHARED ACCESS SIGNATURE' 
, SECRET = '<sastoken>' 

RESTORE HEADERONLY 
FROM URL = 'https://<mystorageaccountname>.blob.core.windows.net/<containername>/<backup_file_name>.bak'
```
1. お使いの SQL Server インスタンスに接続して、このプロセスを繰り返します。

## Log Replay Service を使用してバックアップ ファイルを復元する

Log Replay Service (LRS) を使って、Azure Blob Storage からお使いの Azure SQL マネージド インスタンスにバックアップ ファイルを復元します。 LRS は、SQL Server のログ配布テクノロジに基づく無料サービスです。

1. ストレージ アカウントの概要ページで、**[Blob service]** セクションまで下にスクロールして、**[コンテナー]** を選びます。 バックアップ ファイルが格納されているコンテナーを選びます。
1. コンテナー ページの上部にある **[SAS の生成]** を選びます。 **[Shared Access Signature の生成]** ページで、付与するアクセス許可を選び、SAS トークンの開始時刻と有効期限を設定して、**[SAS と接続文字列を生成する]** を選びます。 SAS トークンが **[SAS トークン]** フィールドに表示されるので、それをコピーします。
1. PowerShell を使って `Connect-AzAccount` コマンドレットを実行し、ご自分の Azure アカウントに接続します。

    ```powershell
    Login-AzAccount
    Select-AzSubscription -SubscriptionId <subscription ID>
    ```

1. `Start-AzSqlInstanceDatabaseLogReplay` コマンドレットを使い、復元するデータベースに対して Log Replay Service を開始します。 リソース グループ名、インスタンス名、データベース名、ストレージ コンテナー URI、先ほどコピーした SAS トークンを指定する必要があります。

```PowerShell
Import-Module Az.Sql

Start-AzSqlInstanceDatabaseLogReplay -ResourceGroupName "YourResourceGroupName" -InstanceName "YourInstanceName" -Name "YourDatabaseName" -StorageContainerUri "https://yourstorageaccount.blob.core.windows.net/yourcontainer" -StorageContainerSasToken "YourSasToken"
```

## 移行の進行状況を監視する

`Get-AzSqlInstanceDatabaseLogReplay` コマンドレットを使って、Log Replay Service の進行状況を監視できます。 このコマンドレットは、復元された最後のログ バックアップ ファイルなど、サービスの現在の状態に関する情報を返します。

1. 次の PowerShell コードを実行します。

```powershell
# Import the Az.Sql module
Import-Module Az.Sql

# Set the resource group name, instance name, and database name
$resourceGroupName = "YourResourceGroupName"
$instanceName = "YourInstanceName"
$databaseName = "YourDatabaseName"

# Get the log replay status
$logReplayStatus = Get-AzSqlInstanceDatabaseLogReplay -ResourceGroupName $resourceGroupName -InstanceName $instanceName -Name $databaseName

# Display the log replay status
$logReplayStatus | Format-List
```

## 移行カットオーバーの実行

Azure SQL Database マネージド インスタンスのターゲット インスタンスで完全なデータベース バックアップが復元されたら、データベースは移行カットオーバーに使用できます。

1. オンライン データベースの移行を完了する準備が整ったら、 **[カットオーバーの開始]** を選択します。
1. ソース データベースへの着信トラフィックをすべて停止します。
1. ログ末尾のバックアップを採用し、バックアップ ファイルを SMB ネットワーク共有で使用できるようにしたら、この最後のトランザクション ログのバックアップが復元されるまで待機します。
1. その時点で、 **[保留中の変更]** が 0 に設定されたことがわかります。
1. **[確認]** を選択したら、 **[適用]** を選択します。

    ![移行カットオーバー画面](../media/3-migration-cutover-screen.png)

1. データベース移行の状態が **[完了]** と表示されたら、Azure SQL Database Managed Instance の新しいターゲット インスタンスにアプリケーションを接続します。