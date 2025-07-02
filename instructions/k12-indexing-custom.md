# SDK를 사용하여 Azure Cosmos DB for NoSQL 컨테이너의 인덱스 정책 구성하기

인덱싱 정책(indexing policies)은 모든 Azure Cosmos DB SDK에서 관리할 수 있습니다. 특히 .NET SDK는 Azure Cosmos DB for NoSQL의 컨테이너에 새로운 인덱싱 정책을 설계하고 적용하는 데 사용할 수 있는 클래스 집합을 포함하고 있습니다.

이 랩에서는 .NET SDK를 사용하여 컨테이너를 위한 사용자 지정 인덱싱 정책을 생성해 보겠습니다.

> **[배경지식: SDK를 통한 인덱싱 정책 관리]**
> Azure Portal에서 인덱싱 정책을 수정하는 것은 개발 및 테스트 단계에서 편리하지만, 프로덕션 환경에서는 프로그래밍 방식으로 정책을 관리하는 것이 더 일반적입니다. 애플리케이션 배포 파이프라인의 일부로 인프라를 코드로 관리(Infrastructure as Code, IaC)하거나, 특정 비즈니스 로직에 따라 동적으로 컨테이너 설정을 변경해야 할 때 SDK를 사용하는 것이 필수적입니다. .NET SDK는 JSON 문자열을 직접 다루는 대신, 강력한 형식의(strongly-typed) 클래스를 제공하여 오류 가능성을 줄이고 코드의 가독성을 높여줍니다.

## 개발 환경 준비

이 랩을 진행하는 환경에 **DP-420**에 대한 랩 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 엽니다.

1.  **Visual Studio Code**를 시작합니다.

    > &#128221; Visual Studio Code 인터페이스에 익숙하지 않다면, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

2.  명령 팔레트(command palette)를 열고 **Git: Clone**을 실행하여 ``https://github.com/microsoftlearning/dp-420-cosmos-db-dev`` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

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

7.  **Visual Studio Code**로 돌아갑니다.

## .NET SDK를 사용하여 새로운 인덱싱 정책 생성하기

.NET SDK는 부모 클래스인 [Microsoft.Azure.Cosmos.IndexingPolicy][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy]와 관련된 클래스 모음을 포함하여 코드에서 새로운 인덱싱 정책을 구성할 수 있도록 합니다.

> **[Note: .NET SDK의 인덱싱 정책 클래스]**
> .NET SDK는 인덱싱 정책을 객체 지향적으로 구성할 수 있도록 여러 클래스를 제공합니다.
> - **`IndexingPolicy`**: 인덱싱 정책 전체를 대표하는 메인 클래스입니다. `IndexingMode`, `IncludedPaths`, `ExcludedPaths` 등의 속성을 가집니다.
> - **`IncludedPath`**: 인덱스에 포함할 JSON 경로를 정의하는 클래스입니다.
> - **`ExcludedPath`**: 인덱스에서 제외할 JSON 경로를 정의하는 클래스입니다.
> 이 클래스들을 사용하면 JSON 문자열을 직접 작성하는 것보다 안전하고 명확하게 정책을 정의할 수 있습니다.

1.  **탐색기(Explorer)** 창에서 **12-custom-index-policy** 폴더로 이동합니다.

2.  **script.cs** 코드 파일을 엽니다.

3.  **endpoint**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **endpoint** 값으로 업데이트합니다.
  
    ```csharp
    string endpoint = "<cosmos-endpoint>";
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이라면, C# 문은 **string endpoint = "https&shy;://dp420.documents.azure.com:443/";** 가 됩니다.

4.  **key**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **key** 값으로 업데이트합니다.

    ```csharp
    string key = "<cosmos-key>";
    ```

    > &#128221; 예를 들어, 키가 **fDR2ci9QgkdkvERTQ==** 라면, C# 문은 **string key = "fDR2ci9QgkdkvERTQ==";** 가 됩니다.

5.  기본 빈 생성자를 사용하여 [IndexingPolicy][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy] 타입의 새 변수 **policy**를 생성합니다:

    ```csharp
    IndexingPolicy policy = new ();
    ```

6.  **policy** 변수의 [IndexingMode][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.indexingmode] 속성을 [IndexingMode.Consistent][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingmode#fields] 값으로 설정합니다:

    ```csharp
    policy.IndexingMode = IndexingMode.Consistent;
    ```

7.  **Path** 속성이 **/*** 값으로 설정된 새로운 [ExcludedPath][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.excludedpath] 타입의 객체를 **policy** 변수의 [ExcludedPaths][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.excludedpaths] 컬렉션 속성에 추가합니다:

    ```csharp
    policy.ExcludedPaths.Add(
        new ExcludedPath{ Path = "/*" }
    );
    ```

8.  **Path** 속성이 **/name/?** 값으로 설정된 새로운 [IncludedPath][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.includedpath] 타입의 객체를 **policy** 변수의 [IncludedPaths][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.includedpaths] 컬렉션 속성에 추가합니다:

    ```csharp
    policy.IncludedPaths.Add(
        new IncludedPath{ Path = "/name/?" }
    );
    ```

9.  생성자 매개변수로 ``products``와 ``/category/name`` 값을 전달하여 [ContainerProperties][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.containerproperties] 타입의 새 변수 **options**를 생성합니다:

    ```csharp
    ContainerProperties options = new ("products", "/category/name");
    ```

10. **policy** 변수를 **options** 변수의 [IndexingPolicy][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.containerproperties.indexingpolicy] 속성에 할당합니다:

    ```csharp
    options.IndexingPolicy = policy;
    ```

11. **database** 변수의 [CreateContainerIfNotExistsAsync][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.database.createcontainerifnotexistsasync] 메서드를 비동기적으로 호출하고, **options** 변수를 생성자 매개변수로 전달합니다. 결과를 [Container][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container] 타입의 변수 **container**에 저장합니다:

    ```csharp
    Container container = await database.CreateContainerIfNotExistsAsync(options);
    ```

12. 내장된 **Console.WriteLine** 정적 메서드를 사용하여 **Container Created**라는 제목과 함께 Container 클래스의 [Id][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.id] 속성을 출력합니다:

    ```csharp
    Console.WriteLine($"Container Created [{container.Id}]");
    ```

13. 완료되면, 코드 파일은 이제 다음을 포함해야 합니다:
  
    ```csharp
    using System;
    using Microsoft.Azure.Cosmos;

    string endpoint = "<cosmos-endpoint>";
    string key = "<cosmos-key>";

    CosmosClient client = new CosmosClient(endpoint, key);
    Database database = await client.CreateDatabaseIfNotExistsAsync("cosmicworks");
    
    // 1. 새로운 IndexingPolicy 객체 생성
    IndexingPolicy policy = new ();
    policy.IndexingMode = IndexingMode.Consistent;

    // 2. 최적화 전략: 먼저 모든 경로를 인덱스에서 제외
    policy.ExcludedPaths.Add(
        new ExcludedPath{ Path = "/*" }
    );

    // 3. 필요한 경로만 명시적으로 인덱스에 포함
    policy.IncludedPaths.Add(
        new IncludedPath{ Path = "/name/?" }
    );

    // 4. 컨테이너 생성 옵션 설정 (이름, 파티션 키)
    ContainerProperties options = new ("products", "/category/name");

    // 5. 생성된 인덱싱 정책을 컨테이너 옵션에 할당
    options.IndexingPolicy = policy;

    // 6. 설정된 옵션으로 컨테이너 생성 요청
    Container container = await database.CreateContainerIfNotExistsAsync(options);
    Console.WriteLine($"Container Created [{container.Id}]");
    ```
    > **[코드 설명]**
    > 이 코드는 사용자 지정 인덱싱 정책을 사용하여 `products` 컨테이너를 생성하는 전체 과정을 보여줍니다.
    > - 먼저, 비어있는 `IndexingPolicy` 객체를 생성합니다.
    > - **"Exclude All, Include Specific"** 전략을 사용합니다. `ExcludedPaths`에 와일드카드(`/*`)를 추가하여 기본적으로 모든 속성의 인덱싱을 비활성화합니다. 이는 쓰기 성능을 향상시키고 RU 비용을 절감하는 모범 사례입니다.
    > - 그 다음, `IncludedPaths`에 쿼리에 필요한 `/name/?` 경로만 명시적으로 추가합니다. `?` 와일드카드는 해당 경로의 스칼라 값(문자열, 숫자 등)을 인덱싱하라는 의미입니다.
    > - `ContainerProperties` 객체에 컨테이너 이름, 파티션 키 경로(`"/category/name"`), 그리고 방금 만든 `IndexingPolicy` 객체를 담습니다.
    > - 마지막으로 `CreateContainerIfNotExistsAsync` 함수가 이 `ContainerProperties`를 Azure Cosmos DB 서비스로 보내 컨테이너를 생성합니다.

14. **script.cs** 파일을 **저장**합니다.

15. **Visual Studio Code**에서 **12-custom-index-policy** 폴더의 컨텍스트 메뉴를 열고 **통합 터미널에서 열기(Open in Integrated Terminal)**를 선택하여 새 터미널 인스턴스를 엽니다.

16. [dotnet run][docs.microsoft.com/dotnet/core/tools/dotnet-run] 명령을 사용하여 프로젝트를 빌드하고 실행합니다:

    ```
    dotnet run
    ```

17. 이제 스크립트는 새로 생성된 컨테이너의 이름을 출력합니다:

    ```
    Container Created [products]
    ```

18. 통합 터미널을 닫습니다.

19. 웹 브라우저로 돌아갑니다.

## Data Explorer를 사용하여 .NET SDK로 생성된 인덱싱 정책 관찰하기

다른 인덱싱 정책과 마찬가지로, .NET SDK를 사용하여 적용한 정책도 Data Explorer를 통해 볼 수 있습니다. 이제 포털을 사용하여 이 랩에서 코드로 생성한 정책을 검토할 것입니다.

1.  **Azure Cosmos DB** 계정 리소스 내에서 **Data Explorer** 창으로 이동합니다.

2.  **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장한 다음, **NOSQL API** 탐색 트리 내의 새로운 **products** 컨테이너 노드를 확인합니다.

3.  **NOSQL API** 탐색 트리의 **products** 컨테이너 노드 내에서 **Settings**를 선택합니다.

4.  **Indexing Policy** 섹션에서 인덱싱 정책을 관찰합니다:

    ```json
    {
      "indexingMode": "consistent",
      "automatic": true,
      "includedPaths": [
        {
          "path": "/name/?"
        }
      ],
      "excludedPaths": [
        {
          "path": "/*"
        },
        {
          "path": "/\"_etag\"/?"
        }
      ]
    }
    ```

    > &#128221; 이것이 이 랩에서 .NET SDK를 사용하여 생성한 인덱싱 정책의 JSON 표현입니다.
    > **[Note]** 코드에서는 `automatic` 속성을 명시적으로 설정하지 않았지만, 결과 JSON에는 `"automatic": true`가 포함되어 있습니다. 이는 `IndexingPolicy` 클래스의 기본값이기 때문입니다. 마찬가지로, 시스템 속성인 `_etag`도 기본적으로 제외 경로에 추가됩니다. 이는 SDK가 개발자의 편의를 위해 일반적인 기본값들을 자동으로 처리해준다는 것을 보여줍니다.

5.  웹 브라우저 창이나 탭을 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.id]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.id
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.containerproperties]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.containerproperties
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.containerproperties.indexingpolicy]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.containerproperties.indexingpolicy
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.database.createcontainerifnotexistsasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.database.createcontainerifnotexistsasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.excludedpath]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.excludedpath
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.excludedpath.path]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.excludedpath.path
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.includedpath]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.includedpath
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.includedpath.path]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.includedpath.path
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingmode#fields]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingmode#fields
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.excludedpaths]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.excludedpaths
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.includedpaths]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.includedpaths
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.indexingmode]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.indexingpolicy.indexingmode
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/cosmicworks]: https://www.nuget.org/packages/cosmicworks/
