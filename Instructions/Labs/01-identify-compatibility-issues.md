---
lab:
  title: SQL 移行の互換性の問題を特定する
  description: この演習では、レガシ SQL Server データベースを Azure SQL Database に移行し、互換性の問題を特定します。
  duration: 20
  level: 300
  islab: true
  status: released
  targetDate: '2099-01-01'
---

# SQL 移行の互換性の問題を特定する

このシナリオでは、レガシ SQL Server データベースを Azure SQL Database に移行するように求められています。 あなたのタスクは、移行を実行し、デプロイ エラーとして発生する互換性の問題を特定することです。 また、データベースのスキーマを確認し、Azure SQL Database でサポートされていない機能または構成を特定する必要があります。

この演習には約 **20** 分かかります。

> **注**: この演習を完了するには、Azure サブスクリプションにアクセスして、Azure リソースを作成する必要があります。 Azure サブスクリプションをお持ちでない場合は、始める前に[無料アカウントを作成](https://azure.microsoft.com/free/?azure-portal=true)してください。

## 開始する前に

この演習を実行するには、続行する前に以下が満たされていることを確かめます。

- SQL Server 2019 以降のバージョンと、特定の SQL Server インスタンスと互換性のある [**AdventureWorksLT**](https://learn.microsoft.com/sql/samples/adventureworks-install-configure#download-backup-files) 軽量データベースが必要です。
- アクティブなサブスクリプションが含まれる Azure アカウント。 [無料でアカウントを作成できます](https://azure.microsoft.com/free/?azure-portal=true)。
- ソース データベースに対して読み取りアクセス権を持つ SQL ユーザー。

## SQL Server データベースを復元してコマンドを実行する

1. Windows の [スタート] ボタンを選択し、SSMS と入力します。 一覧から **[Microsoft SQL Server Management Studio]** を選びます。  

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

1. SQL Server インスタンスの **AdventureWorksLT** データベースで次のコマンドを実行します。 これにより、Azure SQL Database ではサポートされていない別のデータベース内のテーブルを参照するビューが作成されます。

```sql
CREATE VIEW [SalesLT].[vServerPrincipals]
AS
    SELECT name, create_date FROM master.sys.server_principals;
GO
```

## Azure Database Migration Service を設定する

Azure Database Migration Service (DMS) を使用すると、データベースをシームレスに Azure に移行できます。 このセクションでは、DMS インスタンスを作成し、オンプレミスの SQL Server に接続するためのセルフホステッド統合ランタイムを設定します。

1. ブラウザーを開き、[[Azure portal]](https://portal.azure.com) に移動します。

1. Azure portal の検索バーに「**Azure Database Migration Service**」と入力し検索結果からこれを選択します。

1. **[新しい移行の開始]** を選択して、新しい Database Migration Service を作成します。

1. **[移行シナリオと Database Migration Service]** ページで、次を選択します。
    - **ソース サーバーの種類**:SQL Server
    - **ターゲット サーバーの種類**:Azure SQL Database
    - **[Database Migration Service]**、を選び、**[選択]** を選びます。

1. **[移行サービスの作成]** ページで、次の詳細を入力します。
    - **サブスクリプション**:Azure サブスクリプションを選択します。
    - **リソース グループ**: 新しいリソース グループを作成するか、既存のリソース グループを選択します。
    - **[場所]**: 最も近いリージョンを選択します。
    - **サービス名**:「`AdventureWorksDMS`」と入力します。

1. **[確認と作成]**、**[作成]** の順に選択します。 デプロイが完了するまで待ちます。

1. 作成したら、Database Migration Service リソースに移動します。

1. **[設定]** で、**[統合ランタイム]** を選択します。

1. **[統合ランタイムの構成]** を選択し、表示されている **[認証キー]** のいずれかをコピーします。

1. インストーラーをダウンロードするには、**[統合ランタイムをダウンロードしてインストールする]** リンクを選択します。

1. ローカル コンピューター (SQL Server がインストールされているか、SQL Server へのネットワーク アクセス権を持つマシン) でインストーラーを実行します。 既定の設定のままで、インストール ウィザードに従って進めます。

1. インストール後 Microsoft Integration Runtime 構成マネージャーが開いたら、先ほどコピーした**認証キー**を貼り付けて **[登録]** を選択します。

1. **[完了]** を選択して登録を完了します。 Azure portal で状態が **[接続済み]** と表示されるまでしばらく待ちます。

    > **注**:セルフホステッド統合ランタイムを使用すると、Azure Database Migration Service をオンプレミスの SQL Server インスタンスに安全に接続できます。

## SQL データベースを作成する

SQL データベースがない場合は、次の手順に従って Azure portal に作成します。

1. 新しいブラウザー タブで、[[Azure portal]](https://portal.azure.com) に移動し、**[SQL データベース]** を検索します。 **[+ 作成]** を選択します。

1. DMS に使用したのと同じ **[リソース グループ]** を選択し、**[データベース名]** として `AdventureWorksTarget` を入力し、**[サーバー]** で **[新規作成]** を選択します。 一意のサーバー名を入力し、**[場所]** を DMS と同じリージョンに設定し、**[SQL 認証の使用]** を選択して、管理者のログインとパスワードを指定します。 **[OK]** を選択します。

1. **[コンピューティングとストレージ]** 設定で **[構成]** を選択し、コスト削減のために **[Basic]** または **[Free]** レベルを選択します。 

1. **[確認および作成]**、**[作成]** の順に選択します。 デプロイが完了するまで待ちます。

## データベースを移行する

Azure Database Migration Service を使用してデータベースを Azure SQL Database に移行します。 このプロセス中に、互換性のないオブジェクトがある場合、デプロイ エラーが発生し、互換性の問題が明らかになります。

1. Azure Database Migration Service リソースで、概要ページから **[+ 新しい移行]** を選択します。

1. **[移行シナリオの選択]** ページで、次を選択します。
    - **ソース サーバーの種類**:SQL Server
    - **ターゲット サーバーの種類**:Azure SQL Database
    - **移行モード:** オフライン

1. **[次へ]** を選択します。

1. **[ソースの詳細]** タブで、次の手順を実行します。
   - **[ソース SQL Server インスタンスは Azure で追跡されていますか?]**: **[いいえ]** を選択します (Azure VM または Azure Arc を使用していない場合)。
   - **[ソース インフラストラクチャの種類]**: **[その他]** を選択します。
   - **[ソース SQL Server インスタンス名]**: 正確な SQL Server インスタンスの名前 (`localhost`、`localhost\SQLEXPRESS`、`(localdb)\MSSQLLocalDB` など) を入力します。

1. **[次へ]** を選択します。

1. **[ソース SQL Server への接続]** タブで、次のように入力します。
   - **[ソース サーバー名]**: SQL Server インスタンス名と同じ値を使用します。
   - **[認証の種類]**: お使いの認証の種類を選択します。 Windows 認証を推奨します。
   - **[ユーザー名とパスワード]**: ソース データベースに対して少なくとも `db_owner` のアクセス許可を持つアカウントを使用します。
   - **[接続のプロパティ]**: **[暗号化接続]** と **[サーバー証明書を信頼する]** のチェック ボックスをオフにします。

1. **[次へ]** を選択します。

1. **[移行するデータベースを選択]** タブで、*[AdventureWorksLT]* データベースを選択します。

1. **[次へ]** を選択します。

1. **[ターゲット Azure SQL Database に接続]** タブで、Azure サブスクリプションをまだ選択していない場合は選択し、適切なユーザー名とパスワードで先ほど作成したターゲット Azure SQL Database を選択します。

1. **[次へ]** を選択します。

1. **[ソースおよびターゲット データベースのマップ]** タブで、前に作成した`AdventureWorksTarget` データベースを選択します。

1. **[次へ]** を選択します。

1. **[移行するデータベース テーブルの選択]** タブで、移行するテーブルを選択します。 この演習では、すべてのテーブルを選択し、**[不足しているスキーマの移行]** オプションが選択されていることを確認します。

1. **[次へ]** を選択します。

1. **[データベース移行の概要]** タブで、移行の詳細を確認してから続行します。

1. その後、**[移行の開始]** を選択します。

## 移行結果を確認する

1. [DMS の概要] ページで、移行を選択して進行状況を表示します。

1. 移行の状態が **Succeeded** になるまで待ちます。 ソース データベース名を選択すると、移行の詳細と進行状況を表示できます。

    > **注:**  ネットワーク接続とデータの量によっては、移行が完了するまで数分かかる場合があります。

1. **[デプロイ エラー]** 列を確認します。 `vServerPrincipals` のビューに次のエラーが表示されるはずです。

    > **デプロイ エラー: 'master.sys.server_principals' のデータベース名またはサーバー名への参照は、このバージョンの SQL Server ではサポートされていません。オブジェクト要素: [SalesLT].[vServerPrincipals].**

    このエラーで、Azure SQL Database でデータベース間参照がサポートされていないことが確定します。 このビューは、現在のデータベース境界の外側に存在する `master.sys.server_principals` を参照します。

1. デプロイ エラーを確認します。 各エラーには、互換性のないオブジェクトの詳細と、Azure SQL Database へのデプロイに失敗した理由が含まれています。

    > **注:** 互換性の問題が原因でデプロイ エラーが発生した場合でも、移行は正常に完了し、残りのオブジェクトはすべて移行しました。 互換性のないオブジェクトはスキップされますが、残りのデータベースは想定どおりに移行されます。

## 問題を修正する (省略可能)

1. *[AdventureWorksLT]* データベースで次の T-SQL コマンドを実行します。

    ```sql
    DROP VIEW [SalesLT].[vServerPrincipals];
    ```

1. Azure portal で Azure Database Migration Service に戻ります。

1. ターゲットの種類として **[Azure SQL Database]** を使用して、*[AdventureWorksLT]* データベースの新しい移行を開始します。 新しい移行を開始する前に、新しいターゲット データベースを作成するか、ターゲット データベース内のすべてのオブジェクトを削除してください。

1. これで、デプロイ エラーなしで移行が正常に完了することがわかるはずです。

Azure SQL Database に移行するときに SQL Server データベースの互換性の問題を特定する方法について学習しました。 デプロイ エラーを確認し、互換性のないオブジェクトに対処し、重要なスキーマ変更を行うことで、Azure SQL Database への移行を成功させるための重要な手順を実行しました。
