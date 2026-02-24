---
lab:
  title: SQL 移行の互換性の問題を特定する
---

# SQL 移行の互換性の問題を特定する

このシナリオでは、Azure SQL Database に移行するためのレガシ SQL Server データベースの準備状況を評価するように求められました。 あなたの仕事は、レガシ データベースの評価を実行し、移行前に潜在的な互換性の問題または行う必要がある変更を特定することです。 また、データベースのスキーマを確認し、Azure SQL Database でサポートされていない機能または構成を特定する必要があります。

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

1. SQL Server インスタンスの **AdventureWorksLT** データベースで次のコマンドを実行します。

```sql
ALTER TABLE [SalesLT].[Customer] ADD [Next] VARCHAR(5);
```

## Azure Database Migration Service を設定する

Azure Database Migration Service (DMS) を使用すると、データベースをシームレスに Azure に移行できます。 このセクションでは、DMS インスタンスを作成し、オンプレミスの SQL Server に接続するためのセルフホステッド統合ランタイムを設定します。

1. ブラウザーを開き、[[Azure portal]](https://portal.azure.com) に移動します。

1. Azure portal の検索バーに「**Azure Database Migration Service**」と入力し検索結果からこれを選択します。

1. **[+ 作成]** を選択して、新しい Database Migration Service を作成します。

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

## 互換性評価を実行する

互換性評価は、潜在的な移行の問題を特定するのに役立ち、移行プロセスが開始される前にそれらを解決する方法に関する詳細なガイダンスを提供します。 これにより、時間とリソースを大幅に節約できます。 

Azure portal の Azure Database Migration Service を使用して互換性評価を実行し、Azure SQL Database ターゲットの結果を表示します。

1. Azure Database Migration Service リソースで、概要ページから **[+ 新しい移行]** を選択します。

1. **[移行シナリオの選択]** ページで、次を選択します。
    - **ソース サーバーの種類**:SQL Server
    - **ターゲット サーバーの種類**:Azure SQL Database
    - **移行モード**:オフライン移行
    - **[選択]** を選択します。

1. **[ソースの詳細]** タブで、ソース SQL Server 接続の詳細を入力します。
    - **ソース SQL Server のインスタンス名**:SQL Server インスタンスの名前 (`localhost` やマシン名など) を入力します。
    - **[認証の種類]**:適切な認証を選択します。
    - **[ユーザー名]** と **[パスワード]**:データベースの読み取りアクセス権を持つ資格情報を入力します。

1. **[次へ: ソース SQL Server に接続]** を選択し、接続が成功したことを確認します。

1. **[データベースの選択]** タブで、*[AdventureWorksLT]* データベースを選択し **[次へ]** を選択します。

1. **[ターゲットの選択]** タブで、Azure サブスクリプションを選択し、ターゲットとして Azure SQL Database を選択します。 お持ちでない場合は、次の手順に従って作成します。

    1. 新しいブラウザー タブで、[[Azure portal]](https://portal.azure.com) に移動し、**[SQL データベース]** を検索します。 **[+ 作成]** を選択します。
    1. DMS に使用したのと同じ **[リソース グループ]** を選択し、**[データベース名]** として `AdventureWorksTarget` を入力し、**[サーバー]** で **[新規作成]** を選択します。 一意のサーバー名を入力し、**[場所]** を DMS と同じリージョンに設定し、**[SQL 認証の使用]** を選択して、管理者のログインとパスワードを指定します。 **[OK]** を選択します。
    1. **[コンピューティングとストレージ]** 設定で **[構成]** を選択し、コスト削減のために **[Basic]** または **[Free]** レベルを選択します。 **[確認および作成]**、**[作成]** の順に選択します。 デプロイが完了するまで待ちます。
    1. [DMS 移行ウィザード] タブに戻り、新しく作成した Azure SQL Database を選択します。

    [**次へ**] を選択します。

1. **[サマリー]** タブで、移行を続行する前に互換性評価の結果を確認します。

## 評価結果を確認する

これで、Azure Database Migration Service で生成された互換性の結果を確認できるようになりました。

1. **[サマリー]** タブで、**[評価の結果]** セクションを確認します。 *[AdventureWorksLT]* データベースを選択して、詳細な結果を表示します。
    
    > **注:** 以前に追加された `Next` 列にフラグが設定されていることがわかります。これは、Azure SQL Database でエラーが発生する可能性があるためです。 `NEXT` という名前の列は予約キーワードです。

1. 互換性の問題の一覧を確認します。 各問題には次のものが含まれます。
    - **問題の説明**:問題の内容。
    - **推奨事項**:移行前に解決する方法。
    - **影響を受けるオブジェクト**:影響を受ける具体的なデータベース オブジェクト。

1. 移行ウィザードをキャンセルし、DMS の概要に戻ります。 新しい移行を開始しますが、今度はターゲットの種類として **[Azure SQL Managed Instance]** を選択します。
    
    > **注:** `Next` 列には Azure SQL Managed Instance のフラグが設定されなくなりました。これはなぜですか? 
    >
    >これは、`Next` 列が、SQL Server 機能との幅広い互換性を有しているため、Azure SQL Managed Instance で安全に使用できることを意味しています。

## 問題を修正する

1. *[AdventureWorksLT]* データベースで次の T-SQL コマンドを実行します。

    ```sql
    ALTER TABLE [SalesLT].[Customer] DROP COLUMN [Next];
    ```

1. Azure portal で Azure Database Migration Service に戻ります。

1. ターゲットの種類として **[Azure SQL Database]** を使用して、*[AdventureWorksLT]* データベースの新しい移行を開始します。

1. **[サマリー]** タブで評価結果を確認します。

    > **注:**  これで、データベースを移行する準備ができました。 Azure SQL Database への移行を妨げる互換性の問題はありません。

Azure SQL Database に移行するための SQL Server データベースの準備状況を評価する方法について学習しました。 互換性の問題に対処し、重要なスキーマの変更を行ったり、それらを報告したりすることで、Azure SQL Database で将来発生する可能性のある潜在的な技術的な問題を軽減するための重要な手順を行いました。
