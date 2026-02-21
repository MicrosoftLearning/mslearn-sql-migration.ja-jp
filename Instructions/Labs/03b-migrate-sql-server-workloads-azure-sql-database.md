
# SQL Server データベースを Azure SQL Database に移行する

この演習では、Azure CLI と Azure Database Migration Service (DMS) を使用して、特定のテーブルを SQL Server データベースから Azure SQL Database に移行する方法を学びます。 

> **重要**: 移行の所要時間に影響を与えないよう (ネットワークと接続の制約の影響を受ける可能性があります)、例として少数のテーブル (*Customer*、*ProductCategory*、*Product*、*Address*) のみを移行します。 必要に応じて、同じプロセスを使ってスキーマ全体を移行できます。

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
| **Azure CLI** | [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) (バージョン 2.51.0 以降) をインストールします。 インストール後、``az extension add --name datamigration`` を実行して **datamigration** 拡張機能をインストールします。 |
| **SSMS** | [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) をインストールして、ソースおよびターゲット データベースに対して T-SQL スクリプトを実行します。 |
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

1. **[バックアップ ストレージの冗長性]** オプションについては、既定値の **[geo 冗長バックアップ ストレージ]** のままにします。 **[確認および作成]** を選択します。

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

## ターゲット スキーマを作成する

データの移行を始める前に、ターゲット テーブルのスキーマを手動で作成してみましょう。

1. SSMS を開き、Azure SQL Database に接続します。 **[サーバーに接続]** ダイアログで、サーバー名として `<server>.database.windows.net` を入力し、**[SQL Server 認証]** を選択して、前に設定した管理者資格情報を入力します。

1. 新しい **[新しいクエリ]** ウィンドウに次の T-SQL スクリプトをコピーして貼り付け、移行するテーブルのスキーマを作成します:

    ```sql
        USE [AdventureWorksLT]
        GO
        
        CREATE SCHEMA [SalesLT]
        GO
        
        CREATE TYPE [dbo].[Name] FROM [nvarchar](50) NULL
        GO
        
        CREATE TYPE [dbo].[NameStyle] FROM [bit] NOT NULL
        GO
        
        CREATE TYPE [dbo].[Phone] FROM [nvarchar](25) NULL
        GO
        
        CREATE TABLE [SalesLT].[Address](
            [AddressID] [int] IDENTITY(1,1) NOT FOR REPLICATION NOT NULL,
            [AddressLine1] [nvarchar](60) NOT NULL,
            [AddressLine2] [nvarchar](60) NULL,
            [City] [nvarchar](30) NOT NULL,
            [StateProvince] [dbo].[Name] NOT NULL,
            [CountryRegion] [dbo].[Name] NOT NULL,
            [PostalCode] [nvarchar](15) NOT NULL,
            [rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
            [ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_Address_AddressID] PRIMARY KEY CLUSTERED 
        (
            [AddressID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Address_rowguid] UNIQUE NONCLUSTERED 
        (
            [rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY]
        GO
        
        CREATE TABLE [SalesLT].[Customer](
            [CustomerID] [int] IDENTITY(1,1) NOT FOR REPLICATION NOT NULL,
            [NameStyle] [dbo].[NameStyle] NOT NULL,
            [Title] [nvarchar](8) NULL,
            [FirstName] [dbo].[Name] NOT NULL,
            [MiddleName] [dbo].[Name] NULL,
            [LastName] [dbo].[Name] NOT NULL,
            [Suffix] [nvarchar](10) NULL,
            [CompanyName] [nvarchar](128) NULL,
            [SalesPerson] [nvarchar](256) NULL,
            [EmailAddress] [nvarchar](50) NULL,
            [Phone] [dbo].[Phone] NULL,
            [PasswordHash] [varchar](128) NOT NULL,
            [PasswordSalt] [varchar](10) NOT NULL,
            [rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
            [ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_Customer_CustomerID] PRIMARY KEY CLUSTERED 
        (
            [CustomerID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Customer_rowguid] UNIQUE NONCLUSTERED 
        (
            [rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY]
        GO
        
        CREATE TABLE [SalesLT].[Product](
            [ProductID] [int] IDENTITY(1,1) NOT NULL,
            [Name] [dbo].[Name] NOT NULL,
            [ProductNumber] [nvarchar](25) NOT NULL,
            [Color] [nvarchar](15) NULL,
            [StandardCost] [money] NOT NULL,
            [ListPrice] [money] NOT NULL,
            [Size] [nvarchar](5) NULL,
            [Weight] [decimal](8, 2) NULL,
            [ProductCategoryID] [int] NULL,
            [ProductModelID] [int] NULL,
            [SellStartDate] [datetime] NOT NULL,
            [SellEndDate] [datetime] NULL,
            [DiscontinuedDate] [datetime] NULL,
            [ThumbNailPhoto] [varbinary](max) NULL,
            [ThumbnailPhotoFileName] [nvarchar](50) NULL,
            [rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
            [ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_Product_ProductID] PRIMARY KEY CLUSTERED 
        (
            [ProductID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Product_Name] UNIQUE NONCLUSTERED 
        (
            [Name] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Product_ProductNumber] UNIQUE NONCLUSTERED 
        (
            [ProductNumber] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_Product_rowguid] UNIQUE NONCLUSTERED 
        (
            [rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
        GO
        
        CREATE TABLE [SalesLT].[ProductCategory](
            [ProductCategoryID] [int] IDENTITY(1,1) NOT NULL,
            [ParentProductCategoryID] [int] NULL,
            [Name] [dbo].[Name] NOT NULL,
            [rowguid] [uniqueidentifier] ROWGUIDCOL  NOT NULL,
            [ModifiedDate] [datetime] NOT NULL,
         CONSTRAINT [PK_ProductCategory_ProductCategoryID] PRIMARY KEY CLUSTERED 
        (
            [ProductCategoryID] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_ProductCategory_Name] UNIQUE NONCLUSTERED 
        (
            [Name] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY],
         CONSTRAINT [AK_ProductCategory_rowguid] UNIQUE NONCLUSTERED 
        (
            [rowguid] ASC
        )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON, OPTIMIZE_FOR_SEQUENTIAL_KEY = OFF) ON [PRIMARY]
        ) ON [PRIMARY]
        GO
        
        ALTER TABLE [SalesLT].[Address] ADD  CONSTRAINT [DF_Address_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[Address] ADD  CONSTRAINT [DF_Address_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[Customer] ADD  CONSTRAINT [DF_Customer_NameStyle]  DEFAULT ((0)) FOR [NameStyle]
        GO
        
        ALTER TABLE [SalesLT].[Customer] ADD  CONSTRAINT [DF_Customer_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[Customer] ADD  CONSTRAINT [DF_Customer_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[Product] ADD  CONSTRAINT [DF_Product_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[Product] ADD  CONSTRAINT [DF_Product_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory] ADD  CONSTRAINT [DF_ProductCategory_rowguid]  DEFAULT (newid()) FOR [rowguid]
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory] ADD  CONSTRAINT [DF_ProductCategory_ModifiedDate]  DEFAULT (getdate()) FOR [ModifiedDate]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH CHECK ADD  CONSTRAINT [FK_Product_ProductCategory_ProductCategoryID] FOREIGN KEY([ProductCategoryID])
        REFERENCES [SalesLT].[ProductCategory] ([ProductCategoryID])
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [FK_Product_ProductCategory_ProductCategoryID]
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory]  WITH CHECK ADD  CONSTRAINT [FK_ProductCategory_ProductCategory_ParentProductCategoryID_ProductCategoryID] FOREIGN KEY([ParentProductCategoryID])
        REFERENCES [SalesLT].[ProductCategory] ([ProductCategoryID])
        GO
        
        ALTER TABLE [SalesLT].[ProductCategory] CHECK CONSTRAINT [FK_ProductCategory_ProductCategory_ParentProductCategoryID_ProductCategoryID]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_ListPrice] CHECK  (([ListPrice]>=(0.00)))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_ListPrice]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_SellEndDate] CHECK  (([SellEndDate]>=[SellStartDate] OR [SellEndDate] IS NULL))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_SellEndDate]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_StandardCost] CHECK  (([StandardCost]>=(0.00)))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_StandardCost]
        GO
        
        ALTER TABLE [SalesLT].[Product]  WITH NOCHECK ADD  CONSTRAINT [CK_Product_Weight] CHECK  (([Weight]>(0.00)))
        GO
        
        ALTER TABLE [SalesLT].[Product] CHECK CONSTRAINT [CK_Product_Weight]
        GO
        
    ```

1. **F5** キーを押すか **[実行]** を選択して、スクリプトを実行します。

1. 次を実行して、テーブルが正常に作成されたことを確認します:

    ```sql
    SELECT TABLE_SCHEMA, TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_SCHEMA = 'SalesLT';
    ```

## Azure CLI (DMS) を使用してオフライン移行を実行する

これで、データを移行する準備ができました。 Azure CLI と Azure Database Migration Service を使用してオフライン移行を実行するには、次の手順に従います。

### Azure にログインしてサブスクリプションを設定する

1. コマンド プロンプトまたは PowerShell ウィンドウを開き、Azure にログインします:

    ```bash
    az login
    ```

1. 複数のサブスクリプションがある場合は、適切なサブスクリプションを選択します:

    ```bash
    az account set --subscription "<Your subscription name or ID>"
    ```

### Azure Database Migration Service インスタンスを作成する

1. Azure Database Migration Service インスタンスがまだない場合は作成します。 プレースホルダーの値を実際の値に置き換えます:

    ```bash
    az datamigration sql-service create \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service" \
        --location "<Your region>"
    ```

1. サービスが作成されるまで待ちます。 状態を確認するには、次を実行します:

    ```bash
    az datamigration sql-service show \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service"
    ```

### セルフホステッド Integration Runtime を登録する

1. DMS サービスの認証キーを取得します:

    ```bash
    az datamigration sql-service list-auth-key \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service"
    ```

1. 返された認証キーのいずれかをコピーします。

1. Integration Runtime をインストールしたコンピューターで **Microsoft Integration Runtime Configuration Manager** を開き、認証キーを貼り付けることでノードを登録します。 **[登録]**、**[完了]** の順に選択します。

1. ノードの状態を確認して、Integration Runtime が DMS サービスに接続されていることを確認します:

    ```bash
    az datamigration sql-service list-integration-runtime-metric \
        --resource-group "<Your resource group>" \
        --sql-migration-service-name "DMS-Migration-Service"
    ```

    > **注:**  続行するには、ノードの状態が**オンライン**である必要があります。

### データベースの移行を開始する

1. 次のコマンドを使用してオフライン移行を開始します。 プレースホルダーの値をご利用の環境の詳細に置き換えます:

    ```bash
    az datamigration sql-db create \
        --resource-group "<Your resource group>" \
        --sqldb-instance-name "<Target Azure SQL Server name>" \
        --target-db-name "AdventureWorksLT" \
        --migration-service "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.DataMigration/sqlMigrationServices/DMS-Migration-Service" \
        --scope "/subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Sql/servers/<Target Azure SQL Server name>" \
        --source-database-name "AdventureWorksLT" \
        --source-sql-connection authentication="SqlAuthentication" data-source="<Source SQL Server IP or hostname>" encrypt-connection=true password="<Source SQL password>" trust-server-certificate=true user-name="<Source SQL username>" \
        --target-sql-connection authentication="SqlAuthentication" data-source="<Target server>.database.windows.net" encrypt-connection=true password="<Target SQL password>" user-name="sqladmin" \
        --table-list "[SalesLT].[Address]" "[SalesLT].[Customer]" "[SalesLT].[Product]" "[SalesLT].[ProductCategory]" \
        --migration-scope "SelectedTables"
    ```

    > **注:**  `--table-list` パラメーターには、移行する 4 つのテーブルを指定します:*Address*、*Customer*、*Product*、*ProductCategory*。 スキーマは既に手動で作成されているため、データのみが移行されます。

### 移行を監視する

1. 次を実行して移行状態を監視します:

    ```bash
    az datamigration sql-db show \
        --resource-group "<Your resource group>" \
        --sqldb-instance-name "<Target Azure SQL Server name>" \
        --target-db-name "AdventureWorksLT"
    ```

1. **migrationStatus** に **Succeeded** が表示されるまで、上記のコマンドを繰り返します。 Azure SQL Database リソースに移動し、**[データ管理]** で **[移行]** を選択して、Azure portal の状態を確認することもできます。

    > **注:**  ネットワーク接続とデータの量によっては、移行が完了するまで数分かかる場合があります。

### データ移行の確認

1. 移行状態が **Succeeded** になったら、SSMS を使用してターゲット Azure SQL Database に接続し、次のクエリを実行してデータ移行が成功したことを確認します:

    ```sql
    -- Check row counts for migrated data
    SELECT 'Customer' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[Customer]
    UNION ALL
    SELECT 'ProductCategory' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[ProductCategory]
    UNION ALL
    SELECT 'Product' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[Product]
    UNION ALL
    SELECT 'Address' AS TableName, COUNT(*) AS [RowCount] FROM [SalesLT].[Address];
    ```

Azure CLI と Azure Database Migration Service (DMS) を使用して、特定のテーブルを SQL Server データベースから Azure SQL Database に選択的に移行できました。 また、コマンド ラインを使用して移行プロセスを監視する方法についても学習しました。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細情報

Azure SQL Database については、「[Azure SQL Database とは](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview)」を参照してください。
