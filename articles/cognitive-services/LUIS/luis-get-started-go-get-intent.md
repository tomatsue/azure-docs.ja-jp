---
title: Language Understanding (LUIS) の自然言語テキスト分析を GO を使って行う - Azure Cognitive Services | Microsoft Docs
description: このクイック スタートでは、提供されているパブリック LUIS アプリを使って、会話形式のテキストからユーザーの意図を判断します。 GO を使用して、パブリック アプリの HTTP 予測エンドポイントにユーザーの意図をテキストとして送信します。 エンドポイントでは、LUIS によってパブリック アプリのモデルが適用されます。これにより自然言語テキストの意味が分析され、全体的な意図が特定されて、アプリのサブジェクト ドメインに関連したデータが抽出されます。
services: cognitive-services
author: diberry
manager: cjgronlund
ms.service: cognitive-services
ms.component: language-understanding
ms.topic: quickstart
ms.date: 08/23/2018
ms.author: diberry
ms.openlocfilehash: b00d815b712d98136b474d1e73afe7e35d1c7ef4
ms.sourcegitcommit: 2d961702f23e63ee63eddf52086e0c8573aec8dd
ms.translationtype: HT
ms.contentlocale: ja-JP
ms.lasthandoff: 09/07/2018
ms.locfileid: "44160168"
---
# <a name="quickstart-call-a-luis-endpoint-using-go"></a>クイック スタート: Go を使って LUIS エンドポイントを呼び出す

[!INCLUDE [Quickstart introduction for endpoint](../../../includes/cognitive-services-luis-qs-endpoint-intro-para.md)]

## <a name="prerequisites"></a>前提条件

* [Go](https://golang.org/) プログラミング言語  
* [Visual Studio Code](https://code.visualstudio.com/)
* パブリック アプリ ID: df67dcdb-c37d-46af-88e1-8b97951ca1c2

[!INCLUDE [Use authoring key for endpoint](../../../includes/cognitive-services-luis-qs-endpoint-luis-repo-note.md)]

## <a name="get-luis-key"></a>LUIS キーを取得する

[!INCLUDE [Use authoring key for endpoint](../../../includes/cognitive-services-luis-qs-endpoint-get-key-para.md)]

## <a name="analyze-text-with-browser"></a>ブラウザーでテキストを分析する

[!INCLUDE [Use authoring key for endpoint](../../../includes/cognitive-services-luis-qs-endpoint-browser-para.md)]

## <a name="analyze-text-with-go"></a>Go でテキストを分析する

Go を使用して、前の手順でブラウザー ウィンドウに表示されたものと同じ結果にアクセスできます。 

1. `endpoint.go` という名前で新しいファイルを作成します。 次のコードを追加します。
    
   [!code-go[Go code that calls a LUIS endpoint](~/samples-luis/documentation-samples/quickstarts/analyze-text/go/endpoint.go?range=36-98)]

2. このファイルを作成したディレクトリからコマンド プロンプトで「`go build endpoint.go`」と入力し、Go ファイルをコンパイルします。 このコマンド プロンプトから、ビルドの成功に関する情報は一切返されません。

3. コマンド プロンプトに次のテキストを入力して、コマンド ラインから Go アプリケーションを実行します。 

    ```CMD
    go run endpoint.go -appID df67dcdb-c37d-46af-88e1-8b97951ca1c2 -endpointKey <add-your-key> -region westus
    ```
    
    `<add-your-key>` は、実際のキーの値に置き換えてください。  
    
    コマンド プロンプトから次のような応答が返されます。 
    
    ```CMD
    appID has value df67dcdb-c37d-46af-88e1-8b97951ca1c2
    endpointKey has value xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
    region has value westus
    utterance has value turn on the bedroom light
    response
    {
        "query": "turn on the bedroom light",
        "topScoringIntent": {
            "intent": "HomeAutomation.TurnOn",
            "score": 0.809439957
        },
        "entities": [
            {
            "entity": "bedroom",
            "type": "HomeAutomation.Room",
            "startIndex": 12,
            "endIndex": 18,
            "score": 0.8065475
            }
        ]
    }
    ```
    
## <a name="luis-keys"></a>LUIS キー

[!INCLUDE [Use authoring key for endpoint](../../../includes/cognitive-services-luis-qs-endpoint-key-usage-para.md)]

## <a name="clean-up-resources"></a>リソースのクリーンアップ
Go ファイルを閉じてファイル システムから削除します。 

## <a name="next-steps"></a>次の手順
> [!div class="nextstepaction"]
> [発話の追加](luis-get-started-go-add-utterance.md)