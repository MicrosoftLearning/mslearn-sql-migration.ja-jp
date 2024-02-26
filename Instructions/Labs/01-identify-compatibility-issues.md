---
lab:
  title: SQL 移行の互換性の問題を特定する
---

# SQL 移行の互換性の問題を特定する

このシナリオでは、Azure SQL Database に移行するためのレガシ SQL Server データベースの準備状況を評価するように求められました。 あなたの仕事は、レガシ データベースの評価を実行し、移行前に潜在的な互換性の問題または行う必要がある変更を特定することです。 また、データベースのスキーマを確認し、Azure SQL Database でサポートされていない機能または構成を特定する必要があります。

この演習には約 **15** 分かかります。

> **注**: この演習を完了するには、Azure サブスクリプションにアクセスして、Azure リソースを作成する必要があります。 Azure サブスクリプションをお持ちでない場合は、始める前に[無料アカウントを作成](https://azure.microsoft.com/free/?azure-portal=true)してください。

## 開始する前に

この演習を実行するには、続行する前に以下が満たされていることを確かめます。

- SQL Server 2019 以降のバージョンと、特定の SQL Server インスタンスと互換性のある [**AdventureWorksLT**](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) 軽量データベースが必要です。
- [Azure Data Studio](https://learn.microsoft.com/sql/azure-data-studio/download-azure-data-studio) をダウンロードしてインストールします。 既にインストールされている場合は、最新バージョンを使用していることを確認するように更新します。
- ソース データベースに対して読み取りアクセス権を持つ SQL ユーザー。

## SQL Server データベースを復元してコマンドを実行する

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

1. SQL Server インスタンスの **AdventureWorksLT** データベースで次のコマンドを実行します。

```sql
ALTER TABLE [SalesLT].[Customer] ADD [Next] VARCHAR(5);
```

## Azure Data Studio 用の Azure 移行拡張機能をインストールして起動する

移行拡張機能をインストールするには、次の手順に従います。 Azure 移行拡張機能が既にインストールされている場合は、これらの手順をスキップできます。

1. Azure Data Studio で拡張機能マネージャーを開きます。 s

1. 「***Azure SQL 移行***」と検索して、この拡張機能をインストールします。 インストールすると、インストール済みの拡張機能の一覧に Azure SQL 移行拡張機能が表示されます。

1. **[接続]** アイコンを選択して、**[新しい接続]** を選択します。 

1. 新しい **[接続**] タブで、サーバー名を入力します。 **[暗号化]** オプションに **[オプション] (False)** を選択します。

1. **[接続]** を選択します。 

1. Azure 移行拡張機能を起動するには、ソース インスタンスの名前を右クリックし、**[管理]** を選択します。 

1. サーバー メニューの **[全般]** で、**[Azure SQL 移行]** を選択します。 これにより、Azure SQL 移行拡張機能のメイン ページに移動します。

    > **注**: サーバー メニューに **Azure SQL移行**オプションが表示されない場合、または Azure SQL 移行ページが読み込まれない場合は、Azure Data Studio を再度開きます。

## 互換性評価を実行する

互換性評価は、潜在的な移行の問題を特定するのに役立ち、移行プロセスが開始される前にそれらを解決する方法に関する詳細なガイダンスを提供します。 これにより、時間とリソースを大幅に節約できます。 

Azure Data Studio 用の Azure 移行拡張機能を実行し、互換性評価を実行してから、Azure SQL Database のターゲットの結果を表示します。

1. [Azure SQL 移行] ダッシュボードで、**[Azure SQL への移行]** を選択して移行ウィザードを起動します。

1. **[手順 1: 評価用データベース]** で、*AdventureWorks* データベースを選んでから、**[次へ]** を選択します。

1. **[手順 2: 評価の結果と SKU 推奨事項]** で、評価が完了するまで待ってから、**[次へ]** を選択します。

## 評価結果を確認する

これで、移行拡張機能によって生成された推奨事項を確認できるようになりました。

1. **[手順 3: ターゲット プラットフォームと評価の結果]** で、ターゲット プラットフォームとして **[Azure SQL Database]** 選択します。

1. *AdventureWorks* データベースを選択します。 少し時間を取って、右側の評価結果を確認してください。
    
    > **注:** 以前に追加された `Next` 列にフラグが設定されていることがわかります。これは、Azure SQL Database でエラーが発生する可能性があるためです。

1. **Azure SQL Database** ターゲット プラットフォームとして代わりに **[Azure SQL Managed Instance]** を選択します。
    
    > **注:** `Next` 列には Azure SQL Managed Instance のフラグが設定されなくなりました。これはなぜですか? 
    >
    >これは、`Next` 列を Azure SQL Managed Instance で安全に使用できることを意味します。

1. **[評価レポートの保存]** を選択して、レポートを JSON 形式で保存します。

1. 少し時間をとって、JSON ファイルとそのプロパティを確認します。

## 問題を修正する

1. *AdventureWorks* データベースで次の T-SQL コマンドを実行します。

    ```sql
    ALTER TABLE [SalesLT].[Customer] DROP COLUMN [Next];
    ```

1. ウィザードの **[手順 2: 評価の結果と SKU 推奨事項]** ページで、**[評価の更新]** を選択します。

1. ターゲット プラットフォームとして **[Azure SQL Database]** を選択します。

1. *AdventureWorks* データベースを選択します。

    > **注:**  データベースの移行の準備が整いました。

Azure SQL Database に移行するための SQL Server データベースの準備状況を評価する方法について学習しました。 互換性の問題に対処し、重要なスキーマの変更を行ったり、それらを報告したりすることで、Azure SQL Database で将来発生する可能性のある潜在的な技術的な問題を軽減するための重要な手順を行いました。
