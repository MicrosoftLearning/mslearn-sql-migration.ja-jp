
# SQL Server データベースを Azure SQL Database に移行する

この演習では、Azure Database Migration Service (DMS) を使用して、特定のテーブルを SQL Server データベースから Azure SQL Database に移行する方法を学びます。 

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
| **Azure Cloud Shell** | Azure portal から [Azure Cloud Shell](https://learn.microsoft.com/azure/cloud-shell/overview) (PowerShell) を使用して、Azure CLI コマンドを実行します。 ローカルのインストールは必要ありません。 |
| **SSMS** | [SQL Server Management Studio (SSMS)](https://learn.microsoft.com/sql/ssms/download-sql-server-management-studio-ssms) をインストールして、ソースおよびターゲット データベースに対して T-SQL スクリプトを実行します。 |
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

## SQL Server 認証を有効にしてログインを作成する

ソース SQL Server で SQL Server 認証が有効になっていることを確認し、移行サービスで使用されるログインを作成します。

1. SSMS で、ソース SQL Server インスタンスに接続します。 **[オブジェクト エクスプローラー]** でサーバー名を右クリックし、**[プロパティ]** を選択します。

1. **[サーバーのプロパティ]** ページで、**[セキュリティ]** ページを選択します。 **[サーバー認証]** で **[SQL Server と Windows 認証モード]** を選択し、**[OK]** を選択します。

1. 変更を有効にするには、SQL Server サービスを再起動します。 **[SQL Server 構成マネージャー]** または **[サービス]** コンソールから再起動できます。

1. SQL Server サービスを再起動した後、SSMS のソース SQL Server に再接続し、次のスクリプトを実行して **sqladmin** ログインを作成し、必要なアクセス許可を付与します:

    ```sql
    -- Create the sqladmin login
    CREATE LOGIN sqladmin WITH PASSWORD = '<Your password>';

    -- Grant sysadmin role to sqladmin
    ALTER SERVER ROLE sysadmin ADD MEMBER sqladmin;
    ```

    > **注:**  このログインとパスワードを書き留めます。 移行中にソース接続を構成するときに、後で必要になります。

## Azure DMS を使用してオフライン移行を実行する

これで、データを移行する準備ができました。 Azure Database Migration Service を使用してオフライン移行を実行するには、次の手順に従います。

### Azure Database Migration Service インスタンスを作成する

1. Azure portal で、上部のツール バーの **[Cloud Shell]** アイコンを選択します。 シェルの種類として **[PowerShell]** を選択します。 メッセージが表示されたら **[ストレージ アカウントは必要ありません]** を選択し、ご自分のサブスクリプションを選択し、**[適用]** を選択します。

1. Azure Database Migration Service インスタンスを作成する。 プレースホルダーの値を実際の値に置き換えます:

    ```powershell
    az datamigration sql-service create `
        --resource-group "<Your resource group>" `
        --sql-migration-service-name "DMS-Migration-Service" `
        --location "<Your region>"
    ```

    > **注:**  **datamigration** 拡張機能をインストールするように求められたら、**[Y]** を選択して確認します。 このコマンドは、拡張機能のインストール後も続行されます。

1. サービスが作成されるまで待ちます。 状態を確認するには、次を実行します:

    ```powershell
    az datamigration sql-service show `
        --resource-group "<Your resource group>" `
        --sql-migration-service-name "DMS-Migration-Service"
    ```

### セルフホステッド Integration Runtime を登録する

1. まだインストールしていない場合は、ソース SQL Server インスタンスにアクセスできるマシンに [Microsoft Integration Runtime](https://aka.ms/sql-migration-shir-download) をダウンロードしてインストールします。 利用可能な最新バージョンをダウンロードしてください。

1. Cloud Shell で、DMS サービスの認証キーを取得します:

    ```powershell
    az datamigration sql-service list-auth-key `
        --resource-group "<Your resource group>" `
        --sql-migration-service-name "DMS-Migration-Service"
    ```

1. 返された認証キーのいずれかをコピーします。

1. Integration Runtime をインストールしたコンピューターで **Microsoft Integration Runtime Configuration Manager** を開き、認証キーを貼り付けることでノードを登録します。 **[登録]**、**[完了]** の順に選択します。

1. ノードの状態を確認して、Integration Runtime が DMS サービスに接続されていることを確認します:

    ```powershell
    az datamigration sql-service list-integration-runtime-metric `
        --resource-group "<Your resource group>" `
        --sql-migration-service-name "DMS-Migration-Service"
    ```

    > **注:**  続行するには、ノードの状態が**オンライン**である必要があります。

### データベースの移行を開始する

1. Azure portal で、**DMS-Migration-Service** リソースに移動します。 [概要] ページで、**[+ 新しい移行]** を選択します。

1. **[新しい移行シナリオの選択]** で、次の詳細を入力します:
    - **ソース サーバーの種類**:SQL Server
    - **ターゲット サーバーの種類**:Azure SQL Database
    - **移行モード:** オフライン
    - **[選択]** を選択します。

1. **[手順 1: ソースの詳細]** で、次のように構成します:
    - **ソース SQL Server インスタンスは Azure で追跡されていますか?**:**[いいえ]** を選択します。
    - **ソース インフラストラクチャの種類:****仮想マシン**を選択します。
    - **[SQL Server インスタンスの詳細を選択する]** で、次のように入力します:
        - **[サブスクリプション]**:サブスクリプションを選択します。
        - **リソース グループ**: リソース グループを選択します。
        - **場所:** ソース SQL Server の場所を選択します。
        - **SQL Server インスタンス名:** ソース SQL Server インスタンス名を入力します。
    - **[次へ: ソース SQL Server に接続します>>]** を選択します。

1. **[手順 2: ソース SQL Server に接続]** で、次の詳細を入力します:
    - **ソース サーバー名:** ソース SQL Server のホスト名を入力します。
    - **認証の種類:** SQL 認証
    - **ユーザー名:** &lt;ソース SQL ユーザー名&gt;
    - **パスワード:**&lt;ソース SQL パスワード&gt;
    - **[接続のプロパティ]** で、**[接続の暗号化]** と **[サーバー証明書を信頼する]** のチェック ボックスをオフにします。
    - **[次へ: 移行するデータベースを選択]** を選択します。

1. **[手順 3: 移行するデータベースを選択]** で、**AdventureWorksLT** データベースを選択します。 **[次へ: ターゲット Azure SQL Database に接続]** を選択します。

1. **[手順 4:ターゲット Azure SQL Database に接続]** で、ターゲット サーバーの詳細を入力します:
    - **Azure SQL Database サーバー名:**&lt;ターゲット サーバー&gt;.database.windows.net
    - **認証の種類:** SQL 認証
    - **ユーザー名:** sqladmin
    - **パスワード:** &lt;お使いのパスワード&gt;
    - **[接続]** を選択し、**[次へ: ソースおよびターゲット データベースのマップ]** を選択します。

1. **[手順 5:ソースおよびターゲット データベースのマップ]** で、ソース データベース **AdventureWorksLT** がターゲット データベース **AdventureWorksLT** にマップされていることを確認します。 **[次へ: 移行するデータベース テーブルを選択]** を選択します。

1. **[移行するデータベース テーブルを選択]** ページで、**AdventureWorksLT** セクションを展開します。 ターゲットにスキーマが既に作成されているため、**[不足しているスキーマ]** オプションをオフにします。 次の 4 つのテーブルを選択します:
     - **[SalesLT].[Address]**
     - **[SalesLT].[Customer]**
     - **[SalesLT].[Product]**
     - **[SalesLT].[ProductCategory]**

    > **注:**  **[ターゲット テーブルは存在しません]** 状態のテーブルでは、**[不足しているスキーマ]** オプションをオンにするか、スキーマを事前に手動で作成する必要があります。

1. **[次へ: データベース移行の概要 >>]** を選択します。

1. **[データベースの移行の概要]** ページで、移行の詳細を確認し、**[移行の開始]** を選択します。

### 移行を監視する

1. [DMS サービス] ページで、移行を選択して進行状況を表示します。

1. 移行の状態が **Succeeded** になるまで待ちます。 ソース データベース名を選択すると、移行の詳細と進行状況を表示できます。

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

Azure Database Migration Service (DMS) を使用して、SQL Server データベースから Azure SQL Database への特定のテーブルの選択的移行が正常に実行されました。 また、Azure portal を通じて移行プロセスを監視する方法についても学習しました。

## クリーンアップ

独自のサブスクリプションを使用している場合は、プロジェクトの最後に、作成したリソースがまだ必要かどうかを確認してください。 

リソースを不必要に実行したままにしておくと、追加コストが発生する可能性があります。 [Azure portal](https://portal.azure.com?azure-portal=true) でリソースを個別に削除することも、リソースのセット全体を削除することもできます。

## 詳細情報

Azure SQL Database については、「[Azure SQL Database とは](https://learn.microsoft.com/en-us/azure/azure-sql/database/sql-database-paas-overview)」を参照してください。
