# セルフペースドハンズオン

このドキュメントでは、本アーキテクチャを実際に Azure 上にデプロイする手順を紹介します。

## デプロイ用環境について

### 必要なアカウント

- Azureアカウント

クライアントアプリをビルドする場合は、下記のアカウントも必要になります。

- GitHubアカウント
- Googleアカウント (Androidアプリへのプッシュ通知設定のため)

### 必要なソフトウェア

本手順では下記のソフトウェアがインストールされている前提で進めます。

- Visual Studio
- Visual Studio Code
- PowerShell
- Azure CLI
  - IoT 拡張機能
- `sqlcmd` ユーティリティ
- Data migration tool ( `dt` コマンド)
- git

### 各ソフトウェアについて

<details>

<summary>各ソフトウェアについて、詳細を開く</summary>

#### Visual Studio について

本アーキテクチャでは、 Azure Functions やクライアントアプリのビルドに使用します。

インストールする際は、下記をご参考ください。

- [Downloads | IDE, Code, & Team Foundation Server | Visual Studio](https://visualstudio.microsoft.com/downloads/)

また、Azure functions を扱うには、以下のワークロードをインストールしておく必要があります。

- ASP.NET and web development
- Azure development

さらに、モバイルアプリのビルドを行う場合は、下記のワークロードもインストールしてください。

- Mobile development with .NET

#### Visual Studio Code について

この手順では、ドキュメントやソースコードの閲覧、編集に使用します。

インストールする際は、下記をご参考ください。

- [Visual Studio Code - Code Editing. Redefined](https://code.visualstudio.com/#alt-downloads)

#### PowerShell について

本手順ではターミナルとして PowerShell を使用します。bash などの別のターミナルを用いる場合は、読み替えてご参照下さい。

#### Azure CLI について

クロスプラットフォームで利用できる Azure CLI です。デプロイやプロビジョニングで使用します。

インストールする際は、下記をご参考ください。

- [Azure CLI のインストール | Microsoft Doc](https://docs.microsoft.com/ja-jp/cli/azure/install-azure-cli)

さらに、作業で利用するサブスクリプションにログインしておいてください。

```ps1
# Azure にログインする
az login

# ログインしたサブスクリプションを確認する
az account show

# ※ もし上記で表示したサブスクリプションが異なる場合は、下記のようにリスト表示するなどして該当のサブスクリプションIDを取得し、セットしてください
az account list
az account set -s xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

また、 IoT 拡張機能が必要になるので、下記を参考にインストールしてください。

```ps1
# IoT エクステンションをインストールする
az extension add --name azure-iot
```

#### `sqlcmd` ユーティリティ について

`sqlcmd` ユーティリティは、プロビジョニング用スクリプト ( _provision.ps1_ ) の中で使用します。

インストールする際は、下記をご参考ください。

- Windows: [sqlcmd ユーティリティ](https://docs.microsoft.com/ja-jp/sql/tools/sqlcmd-utility?view=sql-server-2017)
- Linux: [sqlcmd および bcp、SQL Server コマンド ライン ツールを Linux にインストールする](https://docs.microsoft.com/ja-jp/sql/linux/sql-server-linux-setup-tools?view=sql-server-2017)

#### Data migration tool ( `dt` コマンド) について

Data migration tool ( `dt` コマンド) は、Cosmos DB に対してデータをアップロードする際に使用します。プロビジョニング用スクリプト ( _provision.ps1_ ) の中で使用しています。

インストールする際は、下記を参考に実行ファイルを展開し、`dt` に対してパスが通るようにしておきましょう。

- [データ移行ツール ( `dt` コマンド) を使用して Azure Cosmos DB にデータを移行する](https://docs.microsoft.com/ja-jp/azure/cosmos-db/import-data)

なお、 この Data migration tool は Windows でしか動作しないので、Linux で作業する場合は別の方法で Cosmos DB にデータをアップロードしてください。（後述）下記は参考です。

- [(ポータルを用いた) サンプル データの追加](https://docs.microsoft.com/ja-jp/azure/cosmos-db/create-sql-api-dotnet#add-sample-data)
- [Azure Cosmos DB Bulk Executor ライブラリの概要](https://docs.microsoft.com/ja-jp/azure/cosmos-db/bulk-executor-overview)

#### git について

ソースコードを作業マシンに用意するために git を利用します。

インストールする際は、下記をご参考下さい。

- [Git](https://git-scm.com/)

</details>

### チェック項目

下記の項目を参考に、デプロイ用環境が準備できたか確認してみましょう。

- Azure ポータル
  - [ ] [Azure ポータル](https://portal.azure.com/)で、作業で使用するAzureアカウントでログインしていること
- 作業に使用するマシン
  - [ ] Visual Studio がインストールされていること
  - [ ] Visual Studio Code がインストールされていること
  - [ ] PowerShell で `az account show` を実行し、作業で使用するAzureアカウントでログインしていること
  - [ ] PowerShell で下記のコマンドが利用できること
    - [ ] `az extension show --name azure-cli-iot-ext` を実行し、Azure CLIにIoT拡張機能がインストールされていること
    - [ ] `sqlcmd -?` を実行し、sqlcmdユーティリティがインストールされていること
    - [ ] `dt` を実行し、dtコマンドがインストールされていること
    - [ ] `git` を実行し、gitが利用できること

----

## デプロイ手順

デプロイ作業の流れは下記のとおりです。

- ソースコードを準備する
- ARMテンプレートでデプロイする
- スクリプトを用いてプロビジョニングする
- プッシュ通知の環境を準備する
- 各 Functions に API key を設定する
- Azure Functions の Application Settings に設定を追加する
  - 各 API key
  - App Center の URL とキー
- API の動作確認
- クライアントアプリのビルド

### ソースコードを準備する

本手順の中でクライアントアプリのビルドまで実施する場合は、本リポジトリを fork した上で作業を進めてください。

下記を参考に、リポジトリのソースコードを作業マシンに準備します。

```ps1
# クライアントアプリのビルドを実施する場合
$REPOSITORY_URL="https://github.com/<Your GitHub account>/smart-store.git"

# クライアントアプリのビルドはしない場合
$REPOSITORY_URL="https://github.com/intelligent-retail/smart-store.git"

# リポジトリをクローンする
git clone ${REPOSITORY_URL}

# リポジトリのディレクトリに移動する
cd smart-store

# 必要に応じて、pull しておく
git checkout master
git pull
```

### ARMテンプレートでデプロイする

それではまず、Azure CLI を使って、 Azure へ各種リソースをデプロイしましょう。

まず、PowerShell で下記の作業を行い、必要となる変数を設定します。

`<...>` と書かれている部分は任意の文字列を設定してください。 `$PREFIX` は、 **小文字** の **2文字** を指定します。 `$STOCK_SERVICE_SQL_SERVER_ADMIN_PASSWORD` は、 大文字小文字、数字、`!$#%` などの記号を含む **8文字以上** を指定します。詳しくは、 [パスワード ポリシー - SQL Server](https://docs.microsoft.com/ja-jp/sql/relational-databases/security/password-policy) に従ってください。

```ps1
$RESOURCE_GROUP="<resource group name>"
$LOCATION="japaneast"

$PREFIX="<prefix string within 2 characters (lower letters)>"
$STOCK_SERVICE_SQL_SERVER_ADMIN_PASSWORD="<sql server admin password>"

$TEMPLATE_URL="https://raw.githubusercontent.com/intelligent-retail/smart-store/master/src/arm-template"
```

なお、各リソースは VNET 統合により外部からのアクセスを遮断しています。本来は、踏み台サーバーなどを用いて内部のリソースにアクセスしますが、ハンズオン等で簡略化する場合は下記のように IP アドレスを指定してアクセス許可を行います。

```ps1
$CURRENT_CLIENT_IP_ADDRESS=$(Invoke-RestMethod http://ifconfig.me/ip)
```

変数が設定できたら、リソースのデプロイを行います。引き続き PowerShell で下記を実行して下さい。

```ps1
# リソースグループを作成する
az group create `
  --name ${RESOURCE_GROUP} `
  --location ${LOCATION}

# 作成したリソースグループの中に、リソースをデプロイする
az deployment group create `
  --resource-group ${RESOURCE_GROUP} `
  --template-uri ${TEMPLATE_URL}/template.json `
  --parameters ${TEMPLATE_URL}/parameters.json `
  --parameters `
    prefix=${PREFIX} `
    stockServiceSqlServerAdminPassword=${STOCK_SERVICE_SQL_SERVER_ADMIN_PASSWORD} `
    allowedWorkspaceIpAddress=$CURRENT_CLIENT_IP_ADDRESS
```

### スクリプトを用いてプロビジョニングする

※ 変数は前項から引き継いでるものとします。

次に、スクリプトを用いてプロビジョニングを行います。

スクリプトでは下記の処理を行っています。

- SQLデータベースのテーブル作成
- IoT Hub の IoT デバイスの登録
- IoT Hub とBOX管理サービスの紐づけ
- 各 Cosmos DB へのデータ投入

それでは、引き続き PowerShell で下記を実行して下さい。

```ps1
# リポジトリのディレクトリに移動する
cd smart-store

# プログラムの実行権限を確認する
Get-ExecutionPolicy -List

# 上記で CurrentUser に RemoteSigned が当たってない場合は、下記を実行する
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# プロビジョニングを実行する
.\src\arm-template\provision.ps1
```

### プッシュ通知の環境を準備する

プッシュ通知の環境を準備します。下記をご参照ください。

- [Azure Notification Hub でのプッシュ通知の環境構築](/docs/azure-notification-hubs.md)

### 各 Functions に API key を設定する

Azure Functions に API key を設定します。

Azure Functions の API key は、関数全体、または関数個別に設定することができます。ここでは、作業簡略化のため、同じ値のキーを関数全体に設定します。

1. Azureポータルで、デプロイした Azure Functions のひとつを開き、メニューの「関数（または、Functions）」>「アプリ キー（または、App keys）」を開きます。
2. 「ホスト キー (すべての関数)（または、Host keys (all functions)）」のフィールドで、「新しいホスト キー（または、New host key）」ボタンをクリックします。
3. 「名前（または、Name）」の欄に `app` と入力し、「OK」ボタンをクリックして保存します。（値は空欄のままとし、自動生成させる）
4. 保存されたら、「値を表示する（または、Show values）」を選択し、「app」の欄から生成された値をコピーします。

次に、コピーしたキーをほかの Azure Functions に設定します。

1. 他の Azure Functions を開き、「Function App の設定」画面に移動します。
2. 「ホスト キー (すべての関数)」のフィールドで、「新しいホスト キー」ボタンをクリックします。
3. 「名前」に `app` 、「値」にコピーしたキーをはりつけて、「OK」ボタンをクリックし保存します。
4. その他の Azure Functions も同様に設定します。

### Azure Functions の Application Settings に設定を追加する

※ 変数は前項から引き継いでるものとします。

前項で設定した API key の情報を Azure Functions の Application Settings に追加します。

下記の変数には、Azure Functions で設定した API key を指定してください。

- `ITEM_MASTER_API_KEY`
- `STOCK_COMMAND_API_KEY`
- `POS_API_KEY`

```ps1
# item-service と stock-service の api key を pos-api に設定する
$ITEM_MASTER_API_KEY="<item service api key>"
$STOCK_COMMAND_API_KEY="<stock service command api key>"
az functionapp config appsettings set `
  --resource-group ${RESOURCE_GROUP} `
  --name ${PREFIX}-pos-api `
  --settings `
    ItemMasterApiKey=${ITEM_MASTER_API_KEY} `
    StockApiKey=${STOCK_COMMAND_API_KEY}

# pos-service の api key と通知の設定を box-api に設定する
$POS_API_KEY="<pos api key>"
az functionapp config appsettings set `
  --resource-group ${RESOURCE_GROUP} `
  --name ${PREFIX}-box-api `
  --settings `
    PosApiKey=${POS_API_KEY}
```

### API の動作確認

API の動作確認については、下記ドキュメントをご参照ください。

- [API の動作確認](/docs/operation-check.md)

### クライアントアプリのビルド

クライアントアプリのビルドについては、下記ドキュメントをご参照ください。

- [クライアントアプリのビルド](/src/client-app/README.md)

----

## 備考

<details>

<summary>備考を開く</summary>

### スクリプトを使わない場合の各種マスタの準備

ここでは、手動でマスタを準備する方法をご紹介します。前項の [スクリプトを用いてプロビジョニングする](#スクリプトを用いてプロビジョニングする) でスクリプトで実施できる方は読み飛ばしてください。

#### 統合商品マスタの準備

Cosmos DB の操作は様々な方法が提供されていますので、適宜ご利用ください。

- Azure Cosmos DB へのインポート
  - [(ポータルを用いた) サンプル データの追加](https://docs.microsoft.com/ja-jp/azure/cosmos-db/create-sql-api-dotnet#add-sample-data)
  - [データ移行ツール ( `dt` コマンド) を使用して Azure Cosmos DB にデータを移行する](https://docs.microsoft.com/ja-jp/azure/cosmos-db/import-data)
  - [Azure Cosmos DB Bulk Executor ライブラリの概要](https://docs.microsoft.com/ja-jp/azure/cosmos-db/bulk-executor-overview)

ここでは、以下の作業を Azure CLI およびデータ移行ツール ( `dt` コマンド) を用いて、コマンドラインで実施する方法をご紹介します。

```bash
# Insert documents to item-service Cosmos DB
ITEM_SERVICE_COSMOSDB_DATABASE="00100"
ITEM_SERVICE_COSMOSDB_DATABASE_THROUGHPUT="400"
ITEM_SERVICE_COSMOSDB_COLLECTION="Items"
ITEM_SERVICE_COSMOSDB_COLLECTION_PARTITIONKEY="/storeCode"

ITEM_SERVICE_COSMOSDB=az cosmosdb list \
    --resource-group ${RESOURCE_GROUP} \
    --query "[?contains(@.name, 'item')==``true``].name" \
    --output tsv
ITEM_SERVICE_COSMOSDB_CONNSTR=$(az cosmosdb list-connection-strings \
    --resource-group ${RESOURCE_GROUP} \
    --name ${ITEM_SERVICE_COSMOSDB} \
    --query "connectionStrings[0].connectionString" \
    --output tsv)

<your-dt-command-path>/dt.exe \
    /s:JsonFile \
    /s.Files:.\\src\\arm-template\\sample-data\\public\\item-service\\itemMasterSampleData.json \
    /t:DocumentDB \
    /t.ConnectionString:"${ITEM_SERVICE_COSMOSDB_CONNSTR};Database=${ITEM_SERVICE_COSMOSDB_DATABASE};" \
    /t.Collection:${ITEM_SERVICE_COSMOSDB_COLLECTION} \
    /t.PartitionKey:${ITEM_SERVICE_COSMOSDB_COLLECTION_PARTITIONKEY} \
    /t.CollectionThroughput:${ITEM_SERVICE_COSMOSDB_DATABASE_THROUGHPUT}
```

##### 商品データに画像を含める場合の事前準備

必要に応じて、画像のアップロードを行います。サンプルの画像とインポートファイルを対象に説明しますが、適宜読み替えてご参考ください。

1. `sample-data/public/item-service/images` ディレクトリ配下に格納されている png 画像を Azure Blog Storage にアップロードする
1. アップロードした URL をインポート用の JSON データに反映する

画像のアップロード操作は様々な方法が提供されていますので、適宜ご利用ください。

- Azure Blob Storage へのアップロード
  - [Azure portal を使用して BLOB をアップロード、ダウンロード、および一覧表示する](https://docs.microsoft.com/ja-jp/azure/storage/blobs/storage-quickstart-blobs-portal)
  - [Azure Storage Explorer を使用してオブジェクト ストレージ内に BLOB を作成する](https://docs.microsoft.com/ja-jp/azure/storage/blobs/storage-quickstart-blobs-storage-explorer)
  - [Azure CLI を使用して BLOB をアップロード、ダウンロード、および一覧表示する](https://docs.microsoft.com/ja-jp/azure/storage/blobs/storage-quickstart-blobs-cli)

ここでは Azure CLI を用いた方法を紹介します。

```bash
# Set variables following above, if you did not set them

# Upload assets images
ASSETS_BLOB_STORAGE_NAME=$(az storage account list \
    --resource-group ${RESOURCE_GROUP} \
    --query "[?contains(@.name, 'assets')==\`true\`].name" \
    --output tsv)
ASSETS_BLOB_STORAGE_CONTAINER=$(az storage container list \
    --account-name ${ASSETS_BLOB_STORAGE_NAME} \
    --query "[0].name" \
    --output tsv)
ASSETS_BLOB_STORAGE_CONNSTR=$(az storage account show-connection-string \
    --name ${ASSETS_BLOB_STORAGE_NAME} \
    --query "connectionString" \
    --output tsv)

az storage blob upload-batch \
    --connection-string ${ASSETS_BLOB_STORAGE_CONNSTR} \
    --destination ${ASSETS_BLOB_STORAGE_CONTAINER} \
    --source src/arm-template/sample-data/public/item-service/images \
    --pattern "*.png"

# Get endpoint of assets storage
ASSETS_BLOB_STORAGE_URL=$(az storage account show \
    --name ${ASSETS_BLOB_STORAGE_NAME} \
    --query "primaryEndpoints.blob" \
    --output tsv)

# Set image paths into the source data
sed -i -e "s|https://sample.blob.core.windows.net/|${ASSETS_BLOB_STORAGE_URL}|g" src/arm-template/sample-data/public/item-service/itemMasterSampleData.json
```

これで商品マスタのインポート用データに画像データを反映できたのので、 [統合商品マスタの準備](#統合商品マスタの準備) に戻り手順を実施してください。

#### Box管理サービス・POSサービスのマスタの準備

Box管理サービス・POSサービスのマスタを Azure Cosmos DB に準備し、必要に応じてデータのを登録します。

つぎに、データの登録を行います。  
データ移行は様々な方法が提供されています。ここでは、以下の作業を Azure CLI およびデータ移行ツール ( `dt` コマンド) を用いて、コマンドラインで実施する方法をご紹介します。インポートファイルを用意しておりますが、適宜読み替えてご参考ください。

- [データ移行ツール ( `dt` コマンド) を使用して Azure Cosmos DB にデータを移行する](https://docs.microsoft.com/ja-jp/azure/cosmos-db/import-data)

```bash
# Set variables following above, if you did not set them
POS_DB_ACCOUNT_NAME=${PREFIX}-pos-service
BOX_DB_ACCOUNT_NAME=${PREFIX}-box-service
POS_DB_NAME='smartretailpos'
BOX_DB_NAME='smartretailboxmanagement'

# Create connection string
POS_SERVICE_COSMOSDB_CONNSTR=$(az cosmosdb list-connection-strings \
    --resource-group ${RESOURCE_GROUP} \
    --name ${POS_DB_ACCOUNT_NAME} \
    --query "connectionStrings[0].connectionString" \
    --output tsv)
BOX_SERVICE_COSMOSDB_CONNSTR=$(az cosmosdb list-connection-strings \
    --resource-group ${RESOURCE_GROUP} \
    --name ${BOX_DB_ACCOUNT_NAME} \
    --query "connectionStrings[0].connectionString" \
    --output tsv)

# Insert documents to Cosmos DB
<your-dt-command-path>/dt.exe \
    /s:JsonFile \
    /s.Files:.\\src\\arm-template\\sample-data\\public\\pos-service\\PosMasters.json \
    /t:DocumentDB \
    /t.ConnectionString:"${POS_SERVICE_COSMOSDB_CONNSTR};Database=${POS_DB_NAME};" \
    /t.Collection:PosMasters
<your-dt-command-path>/dt.exe \
    /s:JsonFile \
    /s.Files:.\\src\\arm-template\\sample-data\\public\\box-service\\Skus.json \
    /t:DocumentDB \
    /t.ConnectionString:"${BOX_SERVICE_COSMOSDB_CONNSTR};Database=${BOX_DB_NAME};" \
    /t.Collection:Skus
<your-dt-command-path>/dt.exe \
    /s:JsonFile \
    /s.Files:.\\src\\arm-template\\sample-data\\public\\box-service\\Terminals.json \
    /t:DocumentDB \
    /t.ConnectionString:"${BOX_SERVICE_COSMOSDB_CONNSTR};Database=${BOX_DB_NAME};" \
    /t.Collection:Terminals
```

</details>
