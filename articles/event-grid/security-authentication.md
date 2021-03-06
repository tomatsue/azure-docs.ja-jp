---
title: Azure Event Grid のセキュリティと認証
description: Azure Event Grid とその概念について説明します。
services: event-grid
author: banisadr
manager: timlt
ms.service: event-grid
ms.topic: conceptual
ms.date: 08/13/2018
ms.author: babanisa
ms.openlocfilehash: ce0e766a07fd19f523f1f35b9a3cbc865cfb8c71
ms.sourcegitcommit: 0fcd6e1d03e1df505cf6cb9e6069dc674e1de0be
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 08/14/2018
ms.locfileid: "42145185"
---
# <a name="event-grid-security-and-authentication"></a>Event Grid のセキュリティと認証 

Azure Event Grid には、3 種類の認証があります。

* イベントのサブスクリプション
* イベントの発行
* webhook のイベント配信

## <a name="webhook-event-delivery"></a>webHook のイベント配信

Webhook は、Azure Event Grid からイベントを受信する多数ある方法の 1 つです。 新しいイベントの準備ができるたびに、EventGrid サービスは、本文にイベントが含まれる HTTP 要求を構成済み HTTP エンドポイントに POST します。

Webhook をサポートする他の多くのサービスと同様に、EventGrid を使用するには、Webhook エンドポイントへのイベントの配信を開始する前に、そのエンドポイントの "所有権" を証明する必要があります。 この要件は、無防備なエンドポイントが EventGrid からのイベント配信のターゲット エンドポイントになることを防ぐためのものです。 ただし、以下に示す 3 つの Azure サービスのいずれかを使用すると、Azure インフラストラクチャはこの検証を自動的に処理します。

* Azure Logic Apps
* Azure Automation
* EventGrid Trigger 用の Azure Functions

他の種類のエンドポイント (HTTP トリガー ベースの Azure 関数など) を使用する場合は、エンドポイントのコードが EventGrid を使用した検証ハンドシェイクに参加する必要があります。 EventGrid は 2 つの異なる検証ハンドシェイク モデルをサポートします。

1. **ValidationCode ハンドシェイク**: イベント サブスクリプションの作成時に、EventGrid は "サブスクリプション検証イベント" をエンドポイントに POST します。 このイベントのスキーマはその他の EventGridEvent に似ており、このイベントのデータ部分には `validationCode` プロパティが含まれています。 検証の要求が想定されるイベント サブスクリプション用であることをアプリケーションが確認したら、検証コードを EventGrid にエコーで返すことによってアプリケーション コードが応答する必要があります。 このハンドシェイク メカニズムは、すべての EventGrid バージョンでサポートされます。

2. **ValidationURL ハンドシェイク (手動のハンドシェイク)**: 特定のケースでは、ValidationCode ベースのハンドシェイクを実装できるようにするためのエンドポイントのソース コードを制御できない場合があります。 たとえば、サード パーティのサービス ([Zapier](https://zapier.com) や [IFTTT](https://ifttt.com/) など) を使用する場合は、プログラムによって検証コードに応答できない可能性があります。 そのため、2018-05-01-preview バージョン以降では、手動の検証ハンドシェイクが EventGrid によってサポートされるようになりました。 この新しい API バージョン (2018-05-01-preview) を使用する SDK/ツールを使用してイベント サブスクリプションを作成すると、EventGrid はサブスクリプション検証イベントのデータ部分の一部として (`validationCode` プロパティに加えて) `validationUrl` プロパティを送信します。 ハンドシェイクを完了するには、REST クライアントまたは Web ブラウザーを使用して、その URL に対して GET 要求を実行します。 提供された検証 URL は、約 10 分間だけ有効です。 この期間、イベント サブスクリプションのプロビジョニング状態は `AwaitingManualAction` になります。 手動による検証を 10 分以内に完了しなかった場合、プロビジョニング状態には `Failed` が設定されます。 手動による検証をもう一度実行する前に、イベント サブスクリプションの作成を再試行する必要があります。

この手動の検証のメカニズムはプレビュー段階です。 使用するには、[AZ CLI 2.0](/cli/azure/install-azure-cli) 向けの [Event Grid 拡張機能](/cli/azure/azure-cli-extensions-list)をインストールする必要があります。 `az extension add --name eventgrid` でインストールできます。 REST API を使用している場合には、`api-version=2018-05-01-preview` を使用してください。

### <a name="validation-details"></a>検証の詳細

* イベント サブスクリプションの作成時または更新時に、Event Grid はサブスクリプション検証イベントをターゲット エンドポイントに投稿します。 
* このイベントには、ヘッダー値 "aeg-event-type: SubscriptionValidation" が含まれています。
* イベント本文のスキーマは、他の Event Grid イベントと同じです。
* イベントの eventType プロパティは "Microsoft.EventGrid.SubscriptionValidationEvent" です。
* イベントの data プロパティには、ランダムに生成された文字列を持つ "validationCode" プロパティが含まれています。 たとえば、"validationCode: acb13…" のようなプロパティです。
* API バージョン 2018-05-01-preview を使用する場合、イベント データには、サブスクリプションを手動で検証するための URL と `validationUrl` プロパティも含まれます。
* 配列には、検証イベントのみが含まれています。 その他のイベントは、検証コードをエコーで返した後、別の要求で送信されます。
* EventGrid DataPlane SDK には、サブスクリプション検証イベント データとサブスクリプション検証の応答に対応するクラスがあります。

SubscriptionValidationEvent の例を以下に示します。

```json
[{
  "id": "2d1781af-3a4c-4d7c-bd0c-e34b19da4e66",
  "topic": "/subscriptions/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "subject": "",
  "data": {
    "validationCode": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6",
    "validationUrl": "https://rp-eastus2.eventgrid.azure.net:553/eventsubscriptions/estest/validate?id=B2E34264-7D71-453A-B5FB-B62D0FDC85EE&t=2018-04-26T20:30:54.4538837Z&apiVersion=2018-05-01-preview&token=1BNqCxBBSSE9OnNSfZM4%2b5H9zDegKMY6uJ%2fO2DFRkwQ%3d"
  },
  "eventType": "Microsoft.EventGrid.SubscriptionValidationEvent",
  "eventTime": "2018-01-25T22:12:19.4556811Z",
  "metadataVersion": "1",
  "dataVersion": "1"
}]
```

エンドポイントの所有権を証明するには、次の例に示すように、validationResponse プロパティで検証コードをエコーで返します。

```json
{
  "validationResponse": "512d38b6-c7b8-40c8-89fe-f46f9e9622b6"
}
```

または、検証 URL に GET 要求を送信することでサブスクリプションを手動で検証できます。 イベント サブスクリプションは、検証されるまで保留状態にとどまります。

サブスクリプション検証ハンドシェイクを処理する方法を示す C# のサンプルを https://github.com/Azure-Samples/event-grid-dotnet-publish-consume-events/blob/master/EventGridConsumer/EventGridConsumer/Function1.cs で確認できます。

### <a name="checklist"></a>チェック リスト

イベント サブスクリプションの作成時に表示される "指定されたエンドポイント https://your-endpoint-here の検証の試行に失敗しました。 詳細については、https://aka.ms/esvalidation を参照してください" のようなエラー メッセージは、検証ハンドシェイクでエラーが発生したことを示します。 このエラーを解決するには、次の点を確認します。

* ターゲット エンドポイントでアプリケーション コードを制御しているか。 たとえば、HTTP トリガー ベースの Azure 関数を記述する場合は、その関数を変更するためにアプリケーション コードにアクセスできるかどうか。
* アプリケーション コードにアクセスできる場合は、上記のサンプルに示すように ValidationCode ベースのハンドシェイクのメカニズムを実装してください。

* アプリケーション コードにアクセスできない場合 (Webhook をサポートするサード パーティのサービスを使用している場合など) は、手動のハンドシェイクのメカニズムを使用できます。 そのためには、2018-05-01-preview API バージョン (たとえば、前述の EventGrid CLI 拡張機能) を使用して、検証イベントで validationUrl を受け取ってください。 手動の検証ハンドシェイクを完了するには、"validationUrl" プロパティの値を取得し、Web ブラウザーでその URL にアクセスしてください。 検証が成功すると、そのことを示すメッセージが Web ブラウザーに表示され、イベント サブスクリプションの provisioningState が "Succeeded" であることが示されます。 

### <a name="event-delivery-security"></a>イベント配信のセキュリティ

イベント サブスクリプションを作成するときに、webhook URL にクエリ パラメーターを追加することで、webhook エンドポイントをセキュリティで保護できます。 これらのクエリ パラメーターのいずれかを、[アクセス トークン](https://en.wikipedia.org/wiki/Access_token)などのシークレットとして設定します。webhook は、これを使用して、イベントが有効なアクセス許可を持つ Event Grid からのものであることを が認識できます。 Event Grid には、webhook へのすべてのイベント配信にこれらのクエリ パラメーターが含められます。

イベント サブスクリプションを編集すると、Azure [CLI](https://docs.microsoft.com/cli/azure?view=azure-cli-latest) で [--include-full-endpoint-url](https://docs.microsoft.com/cli/azure/eventgrid/event-subscription?view=azure-cli-latest#az-eventgrid-event-subscription-show) パラメーターを使用した場合を除き、クエリ パラメーターが表示されなくなるか、返されなくなります。

最後に、Azure Event Grid がサポートするのは HTTPS webhook エンドポイントのみであることにご注意ください。

## <a name="event-subscription"></a>イベント サブスクリプション

イベントにサブスクライブするには、必要なリソースに **Microsoft.EventGrid/EventSubscriptions/Write** アクセス許可を持っている必要があります。 リソースのスコープで新しいサブスクリプションを作成するため、このアクセス許可が必要です。 必要なリソースは、サブスクライブしているのがシステム トピックかカスタム トピックかによって異なります。 ここでは、この 2 つの種類について説明します。

### <a name="system-topics-azure-service-publishers"></a>システム トピック (Azure サービスの発行元)

システム トピックの場合、イベントを発行するリソースのスコープに新しいイベント サブスクリプションを書き込むアクセス許可が必要です。 リソースの形式は次のとおりです。`/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/{resource-provider}/{resource-type}/{resource-name}`

たとえば、**myacct** というストレージ アカウントでイベントにサブスクライブするには、Microsoft.EventGrid/EventSubscriptions/Write アクセス許可が必要です。`/subscriptions/####/resourceGroups/testrg/providers/Microsoft.Storage/storageAccounts/myacct`

### <a name="custom-topics"></a>カスタム トピック

カスタム トピックの場合、Event Grid トピックのスコープに新しいイベント サブスクリプションを書き込むアクセス許可が必要です。 リソースの形式は次のとおりです。`/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.EventGrid/topics/{topic-name}`

たとえば、**mytopic** というカスタム トピックにサブスクライブするには、Microsoft.EventGrid/EventSubscriptions/Write アクセス許可が必要です。`/subscriptions/####/resourceGroups/testrg/providers/Microsoft.EventGrid/topics/mytopic`

## <a name="topic-publishing"></a>トピックの発行

トピックは、Shared Access Signature (SAS) またはキー認証を使用します。 SAS が推奨されますが、キー認証はプログラミングが簡単で、既存の多くの webhook 発行元と互換性があります。 

認証値は、HTTP ヘッダーに含めます。 SAS の場合は、ヘッダー値に **aeg-sas-token** を使用します。 キー認証の場合は、ヘッダー値に **aeg-sas-key** を使用します。

### <a name="key-authentication"></a>キー認証

キー認証は、最も単純な形式の認証です。 次の形式を使用します。`aeg-sas-key: <your key>`

たとえば、次のようにキーを渡します。

```
aeg-sas-key: VXbGWce53249Mt8wuotr0GPmyJ/nDT4hgdEj9DpBeRr38arnnm5OFg==
```

### <a name="sas-tokens"></a>SAS トークン

Event Grid の SAS トークンには、リソース、有効期限、および署名が含まれます。 SAS トークンの形式は `r={resource}&e={expiration}&s={signature}` です。

リソースは、イベントを送信する Event Grid グリッド トピックのパスです。 たとえば、有効なリソース パスは次のとおりです。`https://<yourtopic>.<region>.eventgrid.azure.net/eventGrid/api/events`

キーから署名を生成します。

たとえば、有効な **aeg-sas-token** 値は次のとおりです。

```http
aeg-sas-token: r=https%3a%2f%2fmytopic.eventgrid.azure.net%2feventGrid%2fapi%2fevent&e=6%2f15%2f2017+6%3a20%3a15+PM&s=a4oNHpRZygINC%2fBPjdDLOrc6THPy3tDcGHw1zP4OajQ%3d
```

次の例では、Event Grid で使用する SAS トークンを作成します。

```cs
static string BuildSharedAccessSignature(string resource, DateTime expirationUtc, string key)
{
    const char Resource = 'r';
    const char Expiration = 'e';
    const char Signature = 's';

    string encodedResource = HttpUtility.UrlEncode(resource);
    var culture = CultureInfo.CreateSpecificCulture("en-US");
    var encodedExpirationUtc = HttpUtility.UrlEncode(expirationUtc.ToString(culture));

    string unsignedSas = $"{Resource}={encodedResource}&{Expiration}={encodedExpirationUtc}";
    using (var hmac = new HMACSHA256(Convert.FromBase64String(key)))
    {
        string signature = Convert.ToBase64String(hmac.ComputeHash(Encoding.UTF8.GetBytes(unsignedSas)));
        string encodedSignature = HttpUtility.UrlEncode(signature);
        string signedSas = $"{unsignedSas}&{Signature}={encodedSignature}";

        return signedSas;
    }
}
```

## <a name="management-access-control"></a>管理アクセス制御

Azure Event Grid を使用すると、イベント サブスクリプションの一覧表示、新しいサブスクリプションの作成、キーの生成など、多様な管理操作を実行する各ユーザーに付与するアクセス レベルを制御できます。 Event Grid は、Azure のロール ベースのアクセス確認 (RBAC) を使います。

### <a name="operation-types"></a>操作の種類

Azure Event Grid では、次の操作がサポートされています。

* Microsoft.EventGrid/*/read
* Microsoft.EventGrid/*/write
* Microsoft.EventGrid/*/delete
* Microsoft.EventGrid/eventSubscriptions/getFullUrl/action
* Microsoft.EventGrid/topics/listKeys/action
* Microsoft.EventGrid/topics/regenerateKey/action

最後の 3 つの操作は秘密情報を返す可能性がありますが、この情報は通常の読み取り操作からはフィルターで除外されます。 このような操作へのアクセスは制限することをお勧めします。 カスタム ロールは、[Azure PowerShell](../role-based-access-control/role-assignments-powershell.md)、[Azure コマンド ライン インターフェイス (CLI)](../role-based-access-control/role-assignments-cli.md)、および [REST API](../role-based-access-control/role-assignments-rest.md) で作成することができます。

### <a name="enforcing-role-based-access-check-rbac"></a>ロール ベースのアクセス確認 (RBAC) の適用

次の手順を使用して、ユーザーごとに RBAC を適用します。

#### <a name="create-a-custom-role-definition-file-json"></a>カスタム ロールの定義ファイル (.json) を作成する

さまざまなアクション セットの実行をユーザーに許可する Event Grid ロール定義の例を以下に示します。

**EventGridReadOnlyRole.json**: 読み取り専用操作のみを許可します。

```json
{
  "Name": "Event grid read only role",
  "Id": "7C0B6B59-A278-4B62-BA19-411B70753856",
  "IsCustom": true,
  "Description": "Event grid read only role",
  "Actions": [
    "Microsoft.EventGrid/*/read"
  ],
  "NotActions": [
  ],
  "AssignableScopes": [
    "/subscriptions/<Subscription Id>"
  ]
}
```

**EventGridNoDeleteListKeysRole.json**: 制限付きの投稿アクションを許可しますが、削除アクションは禁止します。

```json
{
  "Name": "Event grid No Delete Listkeys role",
  "Id": "B9170838-5F9D-4103-A1DE-60496F7C9174",
  "IsCustom": true,
  "Description": "Event grid No Delete Listkeys role",
  "Actions": [
    "Microsoft.EventGrid/*/write",
    "Microsoft.EventGrid/eventSubscriptions/getFullUrl/action"
    "Microsoft.EventGrid/topics/listkeys/action",
    "Microsoft.EventGrid/topics/regenerateKey/action"
  ],
  "NotActions": [
    "Microsoft.EventGrid/*/delete"
  ],
  "AssignableScopes": [
    "/subscriptions/<Subscription id>"
  ]
}
```

**EventGridContributorRole.json**: すべての Event Grid アクションを許可します。

```json
{
  "Name": "Event grid contributor role",
  "Id": "4BA6FB33-2955-491B-A74F-53C9126C9514",
  "IsCustom": true,
  "Description": "Event grid contributor role",
  "Actions": [
    "Microsoft.EventGrid/*/write",
    "Microsoft.EventGrid/*/delete",
    "Microsoft.EventGrid/topics/listkeys/action",
    "Microsoft.EventGrid/topics/regenerateKey/action",
    "Microsoft.EventGrid/eventSubscriptions/getFullUrl/action"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<Subscription id>"
  ]
}
```

#### <a name="create-and-assign-custom-role-with-azure-cli"></a>Azure CLI によるカスタム ロールの作成および割り当て

カスタム ロールを作成するには、次のコマンドを使用します。

```azurecli
az role definition create --role-definition @<file path>
```

ロールをユーザーに割り当てるには、次のコマンドを使用します。

```azurecli
az role assignment create --assignee <user name> --role "<name of role>"
```

## <a name="next-steps"></a>次の手順

* Event Grid の概要については、[Event Grid の紹介](overview.md)に関する記事を参照してください。
