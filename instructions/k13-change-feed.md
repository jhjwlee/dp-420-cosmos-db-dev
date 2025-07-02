# Azure Cosmos DB for NoSQL SDK를 사용하여 변경 피드(change feed) 이벤트 처리하기

Azure Cosmos DB for NoSQL의 변경 피드(change feed)는 플랫폼에서 발생하는 이벤트를 기반으로 보조 애플리케이션을 만드는 핵심 기능입니다. Azure Cosmos DB for NoSQL용 .NET SDK는 변경 피드와 통합하고 컨테이너 내의 작업에 대한 알림을 수신하는 애플리케이션을 구축하기 위한 클래스 모음을 제공합니다.

이 랩에서는 .NET SDK의 변경 피드 프로세서(change feed processor) 기능을 사용하여 지정된 컨테이너의 항목에 생성 또는 업데이트 작업이 수행될 때 알림을 받는 애플리케이션을 만들어 봅니다.

> **[배경지식: 변경 피드(Change Feed)]**
> 변경 피드는 Azure Cosmos DB 컨테이너에서 발생하는 모든 변경 사항(항목 생성 및 업데이트)에 대한 영구적인 로그입니다. 삭제 작업은 기본적으로 기록되지 않습니다. 이 로그는 시간 순서대로 정렬되며 파티션 키(partition key) 범위별로 읽을 수 있습니다.
>
> 변경 피드는 다음과 같은 다양한 시나리오에서 강력한 기능을 제공합니다.
> - **이벤트 기반 아키텍처**: 컨테이너의 데이터 변경을 트리거로 사용하여 다른 작업을 수행합니다(예: 알림 보내기, 데이터 집계, 후처리 작업). Azure Functions와 함께 가장 많이 사용되는 패턴입니다.
> - **실시간 데이터 이동**: 변경된 데이터를 다른 데이터 저장소(예: Azure Blob Storage, Azure Synapse Analytics)로 실시간으로 복제하거나 보관합니다.
> - **데이터 동기화**: 여러 데이터 저장소 간의 데이터를 동기화 상태로 유지합니다.

## 개발 환경 준비

이 랩을 진행하는 환경에 **DP-420**에 대한 랩 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 엽니다.

1.  **Visual Studio Code**를 시작합니다.

    > &#128221; Visual Studio Code 인터페이스에 익숙하지 않다면, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

2.  명령 팔레트(command palette)를 열고 **Git: Clone**을 실행하여 `https://github.com/microsoftlearning/dp-420-cosmos-db-dev` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

    > &#128161; **CTRL+SHIFT+P** 키보드 단축키를 사용하여 명령 팔레트를 열 수 있습니다.

3.  리포지토리가 복제되면, **Visual Studio Code**에서 선택한 로컬 폴더를 엽니다.

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
    1.  **URI** 필드를 확인합니다. 이 **endpoint** 값은 이 연습의 뒷부분에서 사용됩니다.
    2.  **PRIMARY KEY** 필드를 확인합니다. 이 **key** 값은 이 연습의 뒷부분에서 사용됩니다.
    3.  **PRIMARY CONNECTION STRING** 필드를 확인합니다. 이 **connection string** 값은 이 연습의 뒷부분에서 사용됩니다.

7.  리소스 메뉴에서 **Data Explorer**를 선택합니다.

8.  **Data Explorer** 창에서 **New Container**를 확장한 다음 **New Database**를 선택합니다.

9.  **New Database** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *`cosmicworks`* |
| **Provision throughput** | enabled |
| **Database throughput** | **Manual** |
| **Database Required RU/s** | `1000` |

10. **Data Explorer** 창으로 돌아와서 계층 구조 내의 **cosmicworks** 데이터베이스 노드를 확인합니다.

11. **Data Explorer** 창에서 **New Container**를 선택합니다.

12. **New Container** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *Use existing* &vert; *cosmicworks* |
| **Container id** | *`products`* |
| **Partition key** | *`/category/name`* |

13. **Data Explorer** 창으로 돌아와서 **cosmicworks** 데이터베이스 노드를 확장한 다음 계층 구조 내의 **products** 컨테이너 노드를 확인합니다.

14. **Data Explorer** 창에서 다시 **New Container**를 선택합니다.

15. **New Container** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *Use existing* &vert; *cosmicworks* |
| **Container id** | *`productslease`* |
| **Partition key** | *`/partitionKey`* |

> **[Note: Lease Container의 역할]**
> 변경 피드 프로세서를 사용하려면 두 개의 컨테이너가 필요합니다.
> 1.  **Source Container** (`products`): 변경 사항을 모니터링할 원본 컨테이너입니다.
> 2.  **Lease Container** (`productslease`): 상태 관리 컨테이너입니다. 변경 피드 프로세서는 이 컨테이너를 사용하여 여러 작업자(worker) 인스턴스 간에 작업을 분산하고, 각 파티션에 대해 어디까지 읽었는지를 추적(checkpointing)합니다. 이 "lease" 메커니즘 덕분에, 애플리케이션이 중단되었다가 다시 시작되어도 중단된 지점부터 변경 사항 처리를 재개할 수 있으며, 여러 인스턴스로 확장하여 병렬로 처리할 수 있습니다.

16. **Data Explorer** 창으로 돌아와서 **cosmicworks** 데이터베이스 노드를 확장한 다음 계층 구조 내의 **productslease** 컨테이너 노드를 확인합니다.

17. **Visual Studio Code**로 돌아갑니다.

## .NET SDK에서 변경 피드 프로세서 구현하기

**Microsoft.Azure.Cosmos.Container** 클래스는 변경 피드 프로세서를 플루언트(fluent) 방식으로 구성하기 위한 일련의 메서드를 제공합니다. 시작하려면 모니터링할 컨테이너, 임대 컨테이너, 그리고 각 변경 배치(batch)를 처리할 C# 델리게이트(delegate)에 대한 참조가 필요합니다.

1.  **탐색기(Explorer)** 창에서 **13-change-feed** 폴더로 이동합니다.

2.  **product.cs** 코드 파일을 엽니다.

3.  **Product** 클래스와 그에 상응하는 속성을 확인합니다. 특히 이 랩에서는 **id**와 **name** 속성을 사용합니다.

4.  **Visual Studio Code**의 **탐색기** 창으로 돌아와 **script.cs** 코드 파일을 엽니다.

5.  **endpoint**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **endpoint** 값으로 업데이트합니다.
  
    ```csharp
    string endpoint = "<cosmos-endpoint>";
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이라면, C# 문은 **string endpoint = "https&shy;://dp420.documents.azure.com:443/";** 가 됩니다.

6.  **key**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **key** 값으로 업데이트합니다.

    ```csharp
    string key = "<cosmos-key>";
    ```

    > &#128221; 예를 들어, 키가 **fDR2ci9QgkdkvERTQ==** 라면, C# 문은 **string key = "fDR2ci9QgkdkvERTQ==";** 가 됩니다.

7.  **client** 변수의 **GetContainer** 메서드를 사용하여 데이터베이스 이름(*cosmicworks*)과 컨테이너 이름(*products*)으로 기존 컨테이너를 검색하고, 결과를 **Container** 타입의 **sourceContainer** 변수에 저장합니다:

    ```csharp
    Container sourceContainer = client.GetContainer("cosmicworks", "products");
    ```

8.  **client** 변수의 **GetContainer** 메서드를 사용하여 데이터베이스 이름(*cosmicworks*)과 컨테이너 이름(*productslease*)으로 기존 컨테이너를 검색하고, 결과를 **Container** 타입의 **leaseContainer** 변수에 저장합니다:

    ```csharp
    Container leaseContainer = client.GetContainer("cosmicworks", "productslease");
    ```

9.  두 개의 입력 매개변수를 가진 빈 비동기 익명 함수를 사용하여 [ChangesHandler<>][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.changefeedhandler-1] 타입의 새 델리게이트 변수 **handleChanges**를 생성합니다:
    1.  **IReadOnlyCollection\<Product\>** 타입의 **changes** 매개변수.
    2.  **CancellationToken** 타입의 **cancellationToken** 매개변수.

    ```csharp
    ChangesHandler<Product> handleChanges = async (
        IReadOnlyCollection<Product> changes,
        CancellationToken cancellationToken
    ) => {
    };
    ```
    > **[Note: ChangesHandler Delegate]**
    > `ChangesHandler<T>`는 변경 피드에서 새로운 변경 사항이 감지되었을 때 SDK가 호출할 메서드를 정의하는 델리게이트(delegate)입니다.
    > - `IReadOnlyCollection<Product> changes`: 감지된 변경 사항들의 배치(batch)입니다. SDK는 항목을 지정된 `Product` 타입으로 자동 역직렬화(deserialize)합니다.
    > - `CancellationToken cancellationToken`: 프로세서가 정상적으로 종료될 때 취소 신호를 보내는 토큰입니다. 장기 실행 작업의 안전한 중단을 위해 사용됩니다.

10. 익명 함수 내에서 내장된 **Console.WriteLine** 정적 메서드를 사용하여 **START\tHandling batch of changes...**라는 원시 문자열을 출력합니다:

    ```csharp
    Console.WriteLine($"START\tHandling batch of changes...");
    ```

11. 여전히 익명 함수 내에서, **changes** 변수를 반복하는 foreach 루프를 만들고, **Product** 타입의 인스턴스를 나타내는 변수로 **product**를 사용합니다:

    ```csharp
    foreach(Product product in changes)
    {
    }
    ```

12. 익명 함수의 foreach 루프 내에서, 내장된 비동기 **Console.WriteLineAsync** 정적 메서드를 사용하여 **product** 변수의 **id**와 **name** 속성을 출력합니다:

    ```csharp
    await Console.Out.WriteLineAsync($"Detected Operation:\t[{product.id}]\t{product.name}");
    ```

13. foreach 루프와 익명 함수 밖에서, **sourceContainer** 변수에 대해 [GetChangeFeedProcessorBuilder<>][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.getchangefeedprocessorbuilder]를 호출한 결과를 저장하는 **builder**라는 새 변수를 만듭니다. 다음 매개변수를 사용합니다:

| **매개변수** | **값** |
| ---: | :--- |
| **processorName** | *productsProcessor* |
| **onChangesDelegate** | *handleChanges* |

    ```csharp
    var builder = sourceContainer.GetChangeFeedProcessorBuilder<Product>(
        processorName: "productsProcessor",
        onChangesDelegate: handleChanges
    );
    ```

14. **builder** 변수에 대해 [WithInstanceName][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.withinstancename] 메서드(매개변수 **consoleApp**), [WithLeaseContainer][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.withleasecontainer] 메서드(매개변수 **leaseContainer**), 그리고 [Build][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.build] 메서드를 플루언트(fluent) 방식으로 호출하고, 결과를 [ChangeFeedProcessor][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor] 타입의 **processor** 변수에 저장합니다:

    ```csharp
    ChangeFeedProcessor processor = builder
        .WithInstanceName("consoleApp")
        .WithLeaseContainer(leaseContainer)
        .Build();
    ```
    > **[Note: Change Feed Processor Builder]**
    > 빌더 패턴(builder pattern)은 객체 생성을 단계별로 명확하게 구성하는 데 사용됩니다.
    > - **`GetChangeFeedProcessorBuilder<Product>`**: 프로세서 빌더를 초기화합니다. `processorName`은 이 프로세서 그룹의 고유 이름이며, `onChangesDelegate`는 변경 사항을 처리할 함수입니다.
    > - **`.WithInstanceName("consoleApp")`**: 이 특정 애플리케이션 인스턴스의 고유 이름입니다. 여러 인스턴스를 실행하여 처리를 확장할 때, 각 인스턴스는 다른 이름을 가져야 합니다.
    > - **`.WithLeaseContainer(leaseContainer)`**: 상태 관리에 사용할 임대 컨테이너를 지정합니다.
    > - **`.Build()`**: 구성된 설정으로 `ChangeFeedProcessor` 객체를 생성합니다.

15. **processor** 변수의 [StartAsync][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor.startasync]를 비동기적으로 호출합니다:

    ```csharp
    await processor.StartAsync();
    ```

16. 내장된 **Console.WriteLine**과 **Console.ReadKey** 정적 메서드를 사용하여 콘솔에 출력을 인쇄하고 애플리케이션이 키 입력을 기다리도록 합니다:

    ```csharp
    Console.WriteLine($"RUN\tListening for changes...");
    Console.WriteLine("Press any key to stop");
    Console.ReadKey();  
    ```

17. **processor** 변수의 [StopAsync][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor.stopasync]를 비동기적으로 호출합니다:

    ```csharp
    await processor.StopAsync();
    ```

18. 완료되면, 코드 파일은 이제 다음을 포함해야 합니다:
  
    ```csharp
    using Microsoft.Azure.Cosmos;
    using static Microsoft.Azure.Cosmos.Container;

    string endpoint = "<cosmos-endpoint>";
    string key = "<cosmos-key>";

    CosmosClient client = new CosmosClient(endpoint, key);
    
    // 1. 소스 컨테이너와 임대 컨테이너에 대한 참조 가져오기
    Container sourceContainer = client.GetContainer("cosmicworks", "products");
    Container leaseContainer = client.GetContainer("cosmicworks", "productslease");
    
    // 2. 변경 사항을 처리할 델리게이트(핸들러) 정의
    ChangesHandler<Product> handleChanges = async (
        IReadOnlyCollection<Product> changes,
        CancellationToken cancellationToken
    ) => {
        Console.WriteLine($"START\tHandling batch of changes...");
        // 3. 배치로 수신된 각 변경 사항을 반복 처리
        foreach(Product product in changes)
        {
            await Console.Out.WriteLineAsync($"Detected Operation:\t[{product.id}]\t{product.name}");
        }
    };
    
    // 4. 변경 피드 프로세서 빌더 초기화
    var builder = sourceContainer.GetChangeFeedProcessorBuilder<Product>(
            processorName: "productsProcessor",
            onChangesDelegate: handleChanges
        );
    
    // 5. 빌더를 사용하여 프로세서 인스턴스 구성 및 빌드
    ChangeFeedProcessor processor = builder
        .WithInstanceName("consoleApp")
        .WithLeaseContainer(leaseContainer)
        .Build();
    
    // 6. 변경 피드 리스닝 시작
    await processor.StartAsync();
    
    // 7. 애플리케이션이 종료되지 않고 계속 실행되도록 대기
    Console.WriteLine($"RUN\tListening for changes...");
    Console.WriteLine("Press any key to stop");
    Console.ReadKey();
    
    // 8. 사용자가 키를 누르면 프로세서 정상 종료
    await processor.StopAsync();
    ```

19. **script.cs** 파일을 **저장**합니다.

20. **Visual Studio Code**에서 **13-change-feed** 폴더의 컨텍스트 메뉴를 열고 **통합 터미널에서 열기(Open in Integrated Terminal)**를 선택하여 새 터미널 인스턴스를 엽니다.

21. [dotnet run][docs.microsoft.com/dotnet/core/tools/dotnet-run] 명령을 사용하여 프로젝트를 빌드하고 실행합니다:

    ```
    dotnet run
    ```

22. **Visual Studio Code**와 터미널을 모두 열어 둡니다.

> &#128221; 다른 도구를 사용하여 Azure Cosmos DB for NoSQL 컨테이너에 항목을 생성할 것입니다. 항목을 생성한 후, 이 터미널로 돌아와 출력을 관찰할 것입니다. 터미널을 미리 닫지 마세요.

## Azure Cosmos DB for NoSQL 계정에 샘플 데이터 시딩하기

명령줄 유틸리티를 사용하여 **cosmicworks** 데이터베이스와 **products** 컨테이너를 생성합니다. 그런 다음 이 도구는 터미널 창에서 실행 중인 변경 피드 프로세서를 통해 관찰할 항목 세트를 생성합니다.

1.  **Visual Studio Code**에서 **터미널(Terminal)** 메뉴를 열고 **터미널 분할(Split Terminal)**을 선택하여 기존 인스턴스 옆에 새 터미널을 엽니다.

2.  머신에서 전역적으로 사용할 수 있도록 [cosmicworks][nuget.org/packages/cosmicworks] 명령줄 도구를 설치합니다.

    ```
    dotnet tool install --global CosmicWorks --version 2.3.1
    ```

    > &#128161; 이 명령은 완료하는 데 몇 분이 걸릴 수 있습니다. 이전에 이 도구의 최신 버전을 이미 설치한 경우, `Tool 'cosmicworks' is already installed`라는 경고 메시지가 출력됩니다.

3.  다음 명령줄 옵션을 사용하여 cosmicworks를 실행하여 Azure Cosmos DB 계정을 시딩합니다:

| **옵션** | **값** |
| ---: | :--- |
| **-c** | *이 랩의 앞부분에서 확인한 connection string 값* |
| **--number-of-employees** | *달리 지정하지 않으면 cosmicworks 명령은 데이터베이스에 각각 1000개와 200개의 항목을 가진 employees와 products 컨테이너를 채웁니다.* |

    ```powershell
    cosmicworks -c "connection-string" --number-of-employees 0 --disable-hierarchical-partition-keys
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이고 키가 **fDR2ci9QgkdkvERTQ==** 라면, 명령어는 다음과 같습니다:
    > ``cosmicworks -c "AccountEndpoint=https://dp420.documents.azure.com:443/;AccountKey=fDR2ci9QgkdkvERTQ==" --number-of-employees 0 --disable-hierarchical-partition-keys``

4.  **cosmicworks** 명령이 계정에 데이터베이스, 컨테이너, 그리고 항목들로 채우는 것을 완료할 때까지 기다립니다.

5.  .NET 애플리케이션의 터미널 출력을 관찰합니다. 터미널은 변경 피드를 통해 전송된 각 변경 사항에 대해 **Detected Operation** 메시지를 출력합니다.

6.  두 통합 터미널을 모두 닫습니다.

7.  **Visual Studio Code**를 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.changefeedhandler-1]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.changefeedhandler-1
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor.startasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor.startasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor.stopasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessor.stopasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.build]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.build
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.withinstancename]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.withinstancename
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.withleasecontainer]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.changefeedprocessorbuilder.withleasecontainer
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.getchangefeedprocessorbuilder]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.getchangefeedprocessorbuilder
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/cosmicworks]: https://www.nuget.org/packages/cosmicworks
