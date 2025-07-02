# Azure Functions를 사용하여 Azure Cosmos DB for NoSQL 데이터 처리하기

Azure Functions용 Azure Cosmos DB 트리거(trigger)는 변경 피드 프로세서(change feed processor)를 사용하여 구현됩니다. 이 지식을 바탕으로 Azure Cosmos DB for NoSQL 컨테이너의 생성(create) 및 업데이트(update) 작업에 응답하는 함수를 만들 수 있습니다. 변경 피드 프로세서를 수동으로 구현해 본 경험이 있다면, Azure Functions의 설정이 유사하다는 것을 알 수 있습니다.

이 랩에서는 데이터베이스를 모니터링하고 그 안에서 감지된 각 작업에 대한 로그 정보를 출력하는 함수 앱(function app)과 필요한 모든 리소스를 생성할 것입니다.

> **[배경지식: Azure Functions와 Cosmos DB 트리거]**
> Azure Functions는 특정 이벤트에 응답하여 코드를 실행하는 서버리스(serverless) 컴퓨팅 서비스입니다. 개발자는 서버 인프라를 관리할 필요 없이 비즈니스 로직에만 집중할 수 있습니다.
>
> **Azure Cosmos DB 트리거**는 Azure Functions의 핵심 기능 중 하나로, Cosmos DB 컨테이너의 변경 피드를 자동으로 수신 대기합니다. 컨테이너에 새 항목이 생성되거나 기존 항목이 업데이트되면, 트리거가 활성화되어 연결된 함수 코드를 실행합니다. 이는 내부적으로 이전 랩에서 다룬 변경 피드 프로세서 라이브러리를 기반으로 하므로, 상태 관리(checkpointing)와 확장성(scaling)을 위한 임대 컨테이너(lease container)가 동일하게 필요합니다. Azure Functions를 사용하면 변경 피드 프로세서를 직접 호스팅하고 관리하는 복잡한 작업을 Azure 플랫폼에 위임할 수 있어 매우 효율적입니다.

## Azure Cosmos DB for NoSQL 계정 생성하기

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다. Azure Cosmos DB for NoSQL 계정 프로비저닝이 완료되면, 엔드포인트(endpoint)와 키(key)를 검색하여 Azure SDK for .NET이나 원하는 다른 SDK를 사용해 계정에 연결할 수 있습니다.

1.  새 웹 브라우저 창이나 탭에서 Azure portal (``portal.azure.com``)로 이동합니다.

2.  구독과 연결된 Microsoft 자격 증명을 사용하여 포털에 로그인합니다.

3.  **+ 리소스 만들기**를 선택하고, *Cosmos DB*를 검색한 후, 다음 설정을 사용하여 새로운 **Azure Cosmos DB for NoSQL** 계정 리소스를 생성합니다. 나머지 모든 설정은 기본값으로 둡니다.

| **설정** | **값** |
| ---: | :--- |
| **Workload Type** | **Learning** |
| **Subscription** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Account Name** | *전역적으로 고유한 이름 입력* |
| **Location** | *사용 가능한 지역 선택* |
| **Capacity mode** | *Provisioned throughput* |
| **Apply Free Tier Discount** | *Do Not Apply* |

> &#128221; 랩 환경에서는 새 리소스 그룹 생성이 제한될 수 있습니다. 이 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

4.  이 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

5.  새로 생성된 **Azure Cosmos DB** 계정 리소스로 이동하여 **Keys** 창으로 이동합니다.

6.  이 창에는 SDK에서 계정에 연결하는 데 필요한 연결 정보와 자격 증명이 포함되어 있습니다. 특히 다음을 확인하세요:
    1.  **PRIMARY CONNECTION STRING** 필드를 확인합니다. 이 **connection string** 값은 이 연습의 뒷부분에서 사용됩니다.

7.  리소스 메뉴에서 **Data Explorer**를 선택합니다.

8.  **Data Explorer** 창에서 **New Container**를 확장한 다음 **New Database**를 선택합니다.

9.  **New Database** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *``cosmicworks``* |
| **Provision throughput** | enabled |
| **Database throughput** | **Manual** |
| **Database Required RU/s** | ``1000`` |

10. **Data Explorer** 창으로 돌아와서 계층 구조 내의 **cosmicworks** 데이터베이스 노드를 확인합니다.

11. **Data Explorer** 창에서 **New Container**를 선택합니다.

12. **New Container** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *Use existing* &vert; *cosmicworks* |
| **Container id** | *``products``* |
| **Partition key** | *``/category/name``* |

13. **Data Explorer** 창으로 돌아와서 **cosmicworks** 데이터베이스 노드를 확장한 다음 계층 구조 내의 **products** 컨테이너 노드를 확인합니다.

14. **Data Explorer** 창에서 다시 **New Container**를 선택합니다.

15. **New Container** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *Use existing* &vert; *cosmicworks* |
| **Container id** | *``productslease``* |
| **Partition key** | *``/id``* |

> **[Note: Lease Container의 역할 (Azure Functions)]**
> Azure Functions의 Cosmos DB 트리거도 내부적으로 변경 피드 프로세서 라이브러리를 사용하므로, 상태 관리를 위한 임대 컨테이너(lease container)가 필수적입니다. 이 컨테이너는 각 파티션의 변경 사항을 어디까지 읽었는지 기록(checkpointing)하고, 여러 함수 인스턴스가 실행될 때 작업을 분배하는 역할을 합니다. 임대 컨테이너의 파티션 키는 일반적으로 `/id`로 설정하는 것이 좋습니다.

16. **Data Explorer** 창으로 돌아와서 **cosmicworks** 데이터베이스 노드를 확장한 다음 계층 구조 내의 **productslease** 컨테이너 노드를 확인합니다.

17. Azure portal의 **홈**으로 돌아갑니다.

## Application Insights 생성하기

*Azure Function Application*을 생성하기 전에, 애플리케이션 함수를 모니터링할 수 있도록 *Azure Application Insights*를 활성화해야 합니다. Application Insights는 먼저 *Log Analytics workspace*가 필요합니다.

> **[배경지식: Application Insights]**
> Application Insights는 Azure Monitor의 기능 중 하나로, 라이브 애플리케이션에 대한 성능 모니터링, 원격 분석 데이터 수집, 문제 진단 등을 제공하는 APM(Application Performance Management) 서비스입니다. 서버리스 환경인 Azure Functions에서는 함수 실행 로그, 성능 메트릭, 예외 등을 확인하고 분석하는 데 필수적인 도구입니다.

1.  검색 상자에서 **Log Analytics workspaces**를 검색합니다.

2.  새 *Log Analytics* 작업 영역을 **+Create**하기 위해 선택합니다.

3.  **Log Analytics workspace** 대화 상자에서 각 설정에 다음 값을 입력한 다음 **Review+Create**를 선택하고 **Create**를 선택합니다:

| **설정** | **값** |
| ---: | :--- |
| **Subscription** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Name** | *``lab14laworkspace``* |
| **Location** | *사용 가능한 지역 선택* |

4.  *Log Analytics workspace*가 생성되면, 검색 상자에서 **Application Insights**를 검색합니다.

5.  새 *Application Insight*를 **+Create**하기 위해 선택합니다.

6.  **Application Insights** 대화 상자에서 각 설정에 다음 값을 입력한 다음 **Review+Create**를 선택하고 **Create**를 선택합니다:

| **설정** | **값** |
| ---: | :--- |
| **Subscription (both entries)** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Name** | *``lab14appinsight``* |
| **Location** | *사용 가능한 지역 선택* |
| **Log Analytics Workspace** | *lab14laworkspace* |

이제 애플리케이션 함수를 모니터링할 수 있습니다.

## Azure 함수 앱 및 Azure Cosmos DB 트리거 함수 생성하기

코드 작성을 시작하기 전에, 생성 마법사를 사용하여 Azure Functions 리소스와 종속 리소스(Application Insights, Storage)를 생성해야 합니다.

1.  **+ 리소스 만들기**를 선택하고, *Functions*를 검색한 후, 새로운 **Function App** 계정 리소스를 생성합니다. 호스팅 옵션으로 **Consumption**을 선택하고 다음 설정으로 앱을 구성합니다. 나머지 모든 설정은 기본값으로 둡니다:

| **설정** | **값** |
| ---: | :--- |
| **Subscription** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Name** | *전역적으로 고유한 이름 입력* |
| **Publish** | *Code* |
| **Runtime stack** | *.NET* |
| **Version** | *8 (LTS) In-process* |
| **Region** | *사용 가능한 지역 선택* |
| **Storage account** | *Create a new storage account* |

> **[Note: Consumption Plan]**
> 소비 계획(Consumption Plan)은 Azure Functions의 서버리스 호스팅 모델입니다. 이벤트가 발생할 때만 함수가 실행되고, 실행된 시간과 리소스 사용량에 대해서만 비용을 지불합니다. 트래픽이 없을 때는 비용이 발생하지 않으며(scale to zero), 트래픽이 증가하면 자동으로 확장됩니다.

> &#128221; 랩 환경에서는 새 리소스 그룹 생성이 제한될 수 있습니다. 이 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

2.  이 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

3.  새로 생성된 **Azure Functions** 계정 리소스로 이동하여 개요 페이지에서 **Create function**을 선택합니다.

4.  **Create function** 팝업에서 다음 설정으로 새 함수를 생성합니다. 나머지 모든 설정은 기본값으로 둡니다:

| **설정** | **값** |
| ---: | :--- |
| **Select a template** | *Azure Cosmos DB trigger* |
| **Function name** | *``ItemsListener``* |
| **Cosmos DB account connection** | *Select New* &vert; *Select Azure Cosmos DB Account* &vert; *이전에 생성한 Azure Cosmos DB 계정 선택* |
| **Database name** | *``cosmicworks``* |
| **Collection name** | *``products``* |
| **Collection name for leases** | *``productslease``* |
| **Create lease collection if it does not exist** | *No* |

> **[Note: 함수 바인딩 구성]**
> 이 단계는 Azure Functions의 강력한 기능인 **바인딩(binding)**을 구성하는 과정입니다. UI에서 Cosmos DB 계정, 데이터베이스, 컨테이너를 선택하면, Azure Functions는 백그라운드에서 필요한 연결 정보(connection string)를 함수 앱의 설정에 안전하게 저장하고, 이를 트리거에 연결합니다. 이로 인해 개발자는 코드에서 직접 연결 문자열을 관리할 필요가 없어집니다.

## .NET에서 함수 코드 구현하기

앞서 생성한 함수는 포털 내에서 편집되는 C# 스크립트입니다. 이제 포털을 사용하여 컨테이너에 삽입되거나 업데이트된 모든 항목의 고유 식별자를 출력하는 짧은 함수를 작성할 것입니다.

1.  **ItemsListener** &vert; **Code + Test** 창에서 **run.csx** 스크립트의 편집기로 이동하여 내용을 모두 삭제합니다.

2.  편집기 영역에서 **Microsoft.Azure.DocumentDB.Core** 라이브러리를 참조합니다:

    ```csharp
    #r "Microsoft.Azure.DocumentDB.Core"
    ```
    > **[Note: `#r` 지시문과 `.csx` 파일]**
    > - **`.csx`**: C# 스크립트 파일 확장자입니다. Visual Studio에서 프로젝트를 빌드하고 컴파일하는 대신, 포털에서 직접 코드를 작성하고 동적으로 실행할 수 있게 해줍니다.
    > - **`#r`**: C# 스크립트에서 외부 어셈블리(라이브러리)를 참조하기 위한 지시문입니다. 여기서는 이전 세대의 Cosmos DB .NET SDK 라이브러리를 참조하고 있습니다.

3.  **System**, **System.Collections.Generic**, 그리고 [Microsoft.Azure.Documents][docs.microsoft.com/dotnet/api/microsoft.azure.documents] 네임스페이스에 대한 using 블록을 추가합니다:

    ```csharp
    using System;
    using System.Collections.Generic;
    using Microsoft.Azure.Documents;
    ```

4.  두 개의 매개변수를 갖는 **Run**이라는 새 정적 메서드를 생성합니다:
    1.  제네릭 타입이 [Document][docs.microsoft.com/dotnet/api/microsoft.azure.documents.document]인 **IReadOnlyList\<\>** 타입의 **input** 매개변수.
    2.  **ILogger** 타입의 **log** 매개변수.

    ```csharp
    public static void Run(IReadOnlyList<Document> input, ILogger log)
    {
    }
    ```
    > **[Note: Azure Function의 시그니처]**
    > - `public static void Run(...)`: Azure Functions에서 실행될 기본 진입점 메서드입니다.
    > - `IReadOnlyList<Document> input`: **입력 바인딩(input binding)**입니다. Cosmos DB 트리거가 감지한 변경된 항목들의 배치(batch)가 이 매개변수로 전달됩니다. `Document` 타입은 특정 클래스에 매핑되지 않은 일반적인 JSON 문서를 나타내는 유연한 타입입니다.
    > - `ILogger log`: 함수 실행 중 로그를 기록하기 위한 로깅 인터페이스입니다. 이 로그는 Application Insights로 전송됩니다.

5.  **Run** 메서드 내에서 **log** 변수의 **LogInformation** 메서드를 호출하여 현재 배치의 항목 수를 계산하는 문자열을 전달합니다:

    ```csharp
    log.LogInformation($"# Modified Items:\t{input?.Count ?? 0}"); 
    ```

6.  여전히 **Run** 메서드 내에서, **input** 변수를 반복하는 foreach 루프를 만들고, **Document** 타입의 인스턴스를 나타내는 변수로 **item**을 사용합니다:

    ```csharp
    foreach(Document item in input)
    {
    }
    ```

7.  **Run** 메서드의 foreach 루프 내에서, **log** 변수의 **LogInformation** 메서드를 호출하여 **item** 변수의 [Id][docs.microsoft.com/dotnet/api/microsoft.azure.documents.resource.id] 속성을 출력하는 문자열을 전달합니다:

    ```csharp
    log.LogInformation($"Detected Operation:\t{item.Id}");
    ```

8.  완료되면, 코드 파일은 이제 다음을 포함해야 합니다:
  
    ```csharp
    #r "Microsoft.Azure.DocumentDB.Core"
    
    using System;
    using System.Collections.Generic;
    using Microsoft.Azure.Documents;
    
    public static void Run(IReadOnlyList<Document> input, ILogger log)
    {
        // 배치가 비어있지 않은지 확인하고, 수신된 항목의 수를 로그로 기록합니다.
        log.LogInformation($"# Modified Items:\t{input?.Count ?? 0}");
    
        // 배치 내의 각 문서를 순회합니다.
        foreach(Document item in input)
        {
            // 각 변경된 문서의 ID를 로그로 기록합니다.
            log.LogInformation($"Detected Operation:\t{item.Id}");
        }
    }
    ```
    > **[코드 설명]**
    > 이 함수는 매우 간단합니다.
    > 1. Cosmos DB 트리거가 `products` 컨테이너에서 변경 사항을 감지하면 이 `Run` 메서드를 호출합니다.
    > 2. 변경된 항목들은 `IReadOnlyList<Document>` 타입의 `input` 매개변수로 전달됩니다.
    > 3. 함수는 먼저 수신된 항목의 총 개수를 로그로 남깁니다.
    > 4. 그런 다음, 루프를 돌며 각 변경된 항목의 `Id`를 개별적으로 로그에 출력합니다.
    > 5. 이 모든 로그는 Application Insights로 스트리밍되어 실시간으로 확인할 수 있습니다.

9.  페이지 하단의 **Logs** 섹션을 확장하고, **App Insights Logs**를 확장한 다음, **Filesystem Logs**를 선택하여 현재 함수의 스트리밍 로그에 연결합니다.

    > &#128161; 스트리밍 로그 서비스에 연결하는 데 몇 초가 걸릴 수 있습니다. 연결되면 로그 출력에 메시지가 표시됩니다.

10. 현재 함수 코드를 **저장**합니다.

11. C# 코드 컴파일 결과를 관찰합니다. 로그 출력 끝에 **Compilation succeeded** 메시지가 표시되어야 합니다.

    > &#128221; 로그 출력에 경고 메시지가 표시될 수 있습니다. 이 경고는 이 랩에 영향을 미치지 않습니다.

12. 로그 섹션을 **최대화**하여 출력 창을 사용 가능한 최대 공간으로 확장합니다.

> &#128221; 다른 도구를 사용하여 Azure Cosmos DB for NoSQL 컨테이너에 항목을 생성할 것입니다. 항목을 생성한 후, 이 브라우저 창으로 돌아와 출력을 관찰할 것입니다. 브라우저 창을 미리 닫지 마세요.

## Azure Cosmos DB for NoSQL 계정에 샘플 데이터 시딩하기

명령줄 유틸리티를 사용하여 **cosmicworks** 데이터베이스와 **products** 컨테이너를 생성합니다. 그런 다음 이 도구는 터미널 창에서 실행 중인 변경 피드 프로세서를 통해 관찰할 항목 세트를 생성합니다.

1.  **Visual Studio Code**를 시작합니다.

    > &#128221; Visual Studio Code 인터페이스에 익숙하지 않다면, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

2.  **Visual Studio Code**에서 **터미널(Terminal)** 메뉴를 열고 **새 터미널(New Terminal)**을 선택하여 새 터미널 인스턴스를 엽니다.

3.  머신에서 전역적으로 사용할 수 있도록 [cosmicworks][nuget.org/packages/cosmicworks] 명령줄 도구를 설치합니다.

    ```
    dotnet tool install --global CosmicWorks --version 2.3.1
    ```

    > &#128161; 이 명령은 완료하는 데 몇 분이 걸릴 수 있습니다. 이전에 이 도구의 최신 버전을 이미 설치한 경우, `Tool 'cosmicworks' is already installed`라는 경고 메시지가 출력됩니다.

4.  다음 명령줄 옵션을 사용하여 cosmicworks를 실행하여 Azure Cosmos DB 계정을 시딩합니다:

| **옵션** | **값** |
| ---: | :--- |
| **-c** | *이 랩의 앞부분에서 확인한 connection string 값* |
| **--number-of-employees** | *달리 지정하지 않으면 cosmicworks 명령은 데이터베이스에 각각 1000개와 200개의 항목을 가진 employees와 products 컨테이너를 채웁니다.* |

    ```powershell
    cosmicworks -c "connection-string" --number-of-employees 0 --disable-hierarchical-partition-keys
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이고 키가 **fDR2ci9QgkdkvERTQ==** 라면, 명령어는 다음과 같습니다:
    > ``cosmicworks -c "AccountEndpoint=https://dp420.documents.azure.com:443/;AccountKey=fDR2ci9QgkdkvERTQ==" --number-of-employees 0 --disable-hierarchical-partition-keys``

5.  **cosmicworks** 명령이 계정에 데이터베이스, 컨테이너, 그리고 항목들로 채우는 것을 완료할 때까지 기다립니다.

6.  통합 터미널을 닫습니다.

7.  **Visual Studio Code**를 닫습니다.

8.  현재 열려 있는 브라우저 창이나 탭으로 돌아가 Azure Functions 로그 섹션을 확장한 상태로 둡니다.

9.  함수의 로그 출력을 관찰합니다. 터미널은 변경 피드를 통해 전송된 각 변경 사항에 대해 **Detected Operation** 메시지를 출력합니다. 작업은 약 100개의 작업 그룹으로 배치(batch) 처리됩니다.

10. 웹 브라우저 창이나 탭을 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.documents]: https://docs.microsoft.com/dotnet/api/microsoft.azure.documents
[docs.microsoft.com/dotnet/api/microsoft.azure.documents.document]: https://docs.microsoft.com/dotnet/api/microsoft.azure.documents.document
[docs.microsoft.com/dotnet/api/microsoft.azure.documents.resource.id]: https://docs.microsoft.com/dotnet/api/microsoft.azure.documents.resource.id
