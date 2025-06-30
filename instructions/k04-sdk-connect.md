
# SDK를 사용하여 Azure Cosmos DB for NoSQL에 연결

.NET용 Azure SDK는 많은 Azure 서비스와 상호 작용하기 위한 일관된 개발자 인터페이스를 제공하는 라이브러리 모음입니다. .NET용 Azure SDK는 .NET Standard 2.0 사양에 맞게 빌드되어 .NET Framework(4.6.1 이상), .NET Core(2.1 이상) 및 .NET(5 이상) 애플리케이션에서 사용할 수 있도록 보장합니다.

이 실습에서는 .NET용 Azure SDK를 사용하여 Azure Cosmos DB for NoSQL 계정에 연결합니다.

> **배경지식: SDK (Software Development Kit)**
>
> SDK는 개발자가 특정 플랫폼, 시스템 또는 프로그래밍 언어를 위한 애플리케이션을 더 쉽게 만들 수 있도록 돕는 도구, 라이브러리, 설명서의 집합입니다. Azure Cosmos DB SDK를 사용하면 C#, Python, Java, Node.js와 같은 프로그래밍 언어를 사용하여 애플리케이션 코드 내에서 직접 Cosmos DB 리소스를 생성, 읽기, 업데이트, 삭제(CRUD) 및 쿼리할 수 있습니다. 이는 Azure Portal을 통해 수동으로 작업하는 것보다 훨씬 효율적이고 자동화된 방식입니다.

## 개발 환경 준비

이 실습을 진행하는 환경에 **DP-420**에 대한 실습 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 여세요.

1.  **Visual Studio Code**를 시작합니다.

    > 📝 Visual Studio Code 인터페이스에 익숙하지 않은 경우, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

1.  명령 팔레트를 열고 **Git: Clone**을 실행하여 ``https://github.com/microsoftlearning/dp-420-cosmos-db-dev`` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

    > 💡 **CTRL+SHIFT+P** 키보드 단축키를 사용하여 명령 팔레트를 열 수 있습니다.

1.  리포지토리가 복제되면, 선택한 로컬 폴더를 **Visual Studio Code**에서 엽니다.

## Azure Cosmos DB for NoSQL 계정 생성

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다. Azure Cosmos DB for NoSQL 계정 프로비저닝이 완료되면, 엔드포인트와 키를 검색하여 .NET용 Azure SDK나 다른 원하는 SDK를 사용하여 Azure Cosmos DB for NoSQL 계정에 연결할 수 있습니다.

1.  새 웹 브라우저 창 또는 탭에서 Azure portal (``portal.azure.com``)로 이동합니다.

2.  구독과 연결된 Microsoft 자격 증명을 사용하여 포털에 로그인합니다.

3.  **+ Create a resource**를 선택하고, *Cosmos DB*를 검색한 후, 다음 설정으로 새로운 **Azure Cosmos DB for NoSQL** 계정 리소스를 생성합니다. 나머지 설정은 모두 기본값으로 둡니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Workload Type** | **Learning** |
| **Subscription** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Account Name** | *전역적으로 고유한 이름 입력* |
| **Location** | *사용 가능한 지역 선택* |
| **Capacity mode** | *Provisioned throughput* |
| **Apply Free Tier Discount** | *Do Not Apply* |
| **Limit the total amount of throughput that can be provisioned on this account** | *Unchecked* |

    > 📝 실습 환경에서는 새 리소스 그룹 생성을 막는 제약이 있을 수 있습니다. 그럴 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

4.  이 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

5.  새로 생성된 **Azure Cosmos DB** 계정 리소스로 이동하여 **Keys** 창으로 이동합니다.

6.  이 창에는 SDK에서 계정에 연결하는 데 필요한 연결 정보와 자격 증명이 포함되어 있습니다. 특히 다음을 확인합니다.
    1.  **URI** 필드. 이 **endpoint** 값은 나중에 이 연습에서 사용됩니다.
    2.  **PRIMARY KEY** 필드. 이 **key** 값은 나중에 이 연습에서 사용됩니다.

7.  나중에 다시 사용할 것이므로 브라우저 탭을 열어 둡니다.

## NuGet에서 Microsoft.Azure.Cosmos 라이브러리 보기

NuGet 웹사이트에는 .NET 애플리케이션으로 가져올 수 있는 패키지의 검색 가능한 인덱스가 포함되어 있습니다. **Microsoft.Azure.Cosmos**와 같은 시험판 패키지를 가져오려면 NuGet 웹사이트를 사용하여 적절한 버전과 명령을 얻어 애플리케이션에 패키지를 가져올 수 있습니다.

1.  새 브라우저 탭에서 NuGet 웹사이트(``nuget.org``)로 이동합니다.
2.  .NET용 패키지 관리자인 NuGet의 설명과 기능을 검토합니다.
3.  NuGet.org에서 **Microsoft.Azure.Cosmos** 라이브러리를 검색합니다.
4.  **.NET CLI** 탭을 선택하여 이 라이브러리의 최신 버전을 .NET 프로젝트로 가져오는 데 필요한 명령을 확인합니다.

    > 💡 이 명령을 기록할 필요는 없습니다. 이 연습의 뒷부분에서 특정 버전의 라이브러리를 사용할 것입니다.

5.  웹 브라우저 창 또는 탭을 닫습니다.

## .NET 프로젝트에 Microsoft.Azure.Cosmos 라이브러리 가져오기

.NET CLI에는 미리 구성된 패키지 피드에서 패키지를 가져오는 [add package][docs.microsoft.com/dotnet/core/tools/dotnet-add-package] 명령이 포함되어 있습니다. .NET 설치는 NuGet을 기본 패키지 피드로 사용합니다.

1.  **Visual Studio Code**의 **Explorer** 창에서 **04-sdk-connect** 폴더로 이동합니다.
2.  **04-sdk-connect** 폴더의 컨텍스트 메뉴를 열고 **Open in Integrated Terminal**을 선택하여 새 터미널 인스턴스를 엽니다.

    > 📝 이 명령은 시작 디렉토리가 이미 **04-sdk-connect** 폴더로 설정된 터미널을 엽니다.

3.  다음 명령을 사용하여 NuGet에서 [Microsoft.Azure.Cosmos][nuget.org/packages/microsoft.azure.cosmos] 패키지를 추가합니다.

    ```powershell
    dotnet add package Microsoft.Azure.Cosmos --version 3.49.0
    ```

4.  통합 터미널을 닫습니다.

## Microsoft.Azure.Cosmos 라이브러리 사용하기

.NET용 Azure SDK의 Azure Cosmos DB 라이브러리를 가져온 후에는 즉시 [Microsoft.Azure.Cosmos][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos] 네임스페이스 내의 클래스를 사용하여 Azure Cosmos DB for NoSQL 계정에 연결할 수 있습니다. [CosmosClient][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient] 클래스는 Azure Cosmos DB for NoSQL 계정에 대한 초기 연결을 만드는 데 사용되는 핵심 클래스입니다.

### **C# 코드**

1.  **Visual Studio Code**의 **Explorer** 창에서 **04-sdk-connect** 폴더로 이동합니다.
2.  비어 있는 **script.cs** 코드 파일을 엽니다.
3.  내장 **System** 및 **System.Linq** 네임스페이스에 대한 `using` 블록을 추가합니다.

    ```csharp
    using System;
    using System.Linq;
    ```

4.  [Microsoft.Azure.Cosmos][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos] 네임스페이스에 대한 `using` 블록을 추가합니다.

    ```csharp
    using Microsoft.Azure.Cosmos;
    ```

5.  이전에 생성한 Azure Cosmos DB 계정의 **endpoint**로 값을 설정한 **endpoint**라는 이름의 `string` 변수를 추가합니다.

    ```csharp
    string endpoint = "<cosmos-endpoint>";
    ```

    > 📝 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/**이면 C# 문은 **string endpoint = "https&shy;://dp420.documents.azure.com:443/";**가 됩니다.

6.  이전에 생성한 Azure Cosmos DB 계정의 **key**로 값을 설정한 **key**라는 이름의 `string` 변수를 추가합니다.

    ```csharp
    string key = "<cosmos-key>";
    ```

    > 📝 예를 들어, 키가 **fDR2ci9QgkdkvERTQ==**이면 C# 문은 **string key = "fDR2ci9QgkdkvERTQ==";**가 됩니다.

7.  생성자에 **endpoint**와 **key** 변수를 사용하여 [CosmosClient][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient] 타입의 **client**라는 새 변수를 추가합니다.

    ```csharp
    CosmosClient client = new (endpoint, key);
    ```
    > **Note: `CosmosClient` 클래스**
    >
    > `CosmosClient`는 Azure Cosmos DB 서비스와 상호 작용하기 위한 기본 진입점입니다. 이 클라이언트 인스턴스는 애플리케이션의 수명 주기 동안 하나만 생성하여 재사용하는 것이 좋습니다(싱글톤 패턴). 스레드로부터 안전하며 효율적인 연결 관리를 수행합니다.

8.  **client** 변수의 [ReadAccountAsync][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient.readaccountasync] 메서드를 호출한 비동기 결과 값을 사용하여 [AccountProperties][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties] 타입의 **account**라는 새 변수를 추가합니다.

    ```csharp
    AccountProperties account = await client.ReadAccountAsync();
    ```
    > **Note: 비동기 프로그래밍 (`async/await`)**
    >
    > `ReadAccountAsync`와 같은 네트워크 I/O 작업은 완료되는 데 시간이 걸릴 수 있습니다. `async/await` 패턴을 사용하면 이 작업이 실행되는 동안 애플리케이션의 기본 스레드가 차단되지 않아 응답성을 유지할 수 있습니다. `await` 키워드는 비동기 작업이 완료될 때까지 메서드의 실행을 일시 중단하고, 완료되면 결과를 반환받아 계속 진행합니다.

9.  내장 **Console.WriteLine** 정적 메서드를 사용하여 **Account Name**이라는 제목과 함께 AccountProperties 클래스의 [Id][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties.id] 속성을 출력합니다.

    ```csharp
    Console.WriteLine($"Account Name:\t{account.Id}");
    ```

10. 내장 **Console.WriteLine** 정적 메서드를 사용하여 AccountProperties 클래스의 [WritableRegions][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties.writableregions] 속성을 쿼리한 다음, **Primary Region**이라는 제목과 함께 첫 번째 결과의 [Name][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountregion.name] 속성을 출력합니다.

    ```csharp
    Console.WriteLine($"Primary Region:\t{account.WritableRegions.FirstOrDefault()?.Name}");
    ```

11. 완료되면 코드 파일은 다음과 같아야 합니다.

    ```csharp
    using System;
    using System.Linq;
    
    using Microsoft.Azure.Cosmos;

    string endpoint = "<cosmos-endpoint>";
    string key = "<cosmos-key>";

    CosmosClient client = new (endpoint, key);

    AccountProperties account = await client.ReadAccountAsync();

    Console.WriteLine($"Account Name:\t{account.Id}");
    Console.WriteLine($"Primary Region:\t{account.WritableRegions.FirstOrDefault()?.Name}");
    ```

12. **script.cs** 코드 파일을 **저장**합니다.

---

### **Python 코드 (추가)**

이제 C#과 동일한 작업을 수행하는 Python 코드를 작성해 보겠습니다.

1.  **Visual Studio Code**의 터미널에서 Python용 Azure Cosmos DB 라이브러리를 설치합니다.

    ```bash
    pip install azure-cosmos
    ```

2.  **04-sdk-connect** 폴더에 **script.py**라는 새 파일을 만듭니다.

3.  다음 코드를 **script.py** 파일에 추가합니다.

    ```python
    import os
    from azure.cosmos import CosmosClient

    # 이전에 생성한 Azure Cosmos DB 계정의 엔드포인트와 키로 값을 설정합니다.
    endpoint = "<cosmos-endpoint>"
    key = "<cosmos-key>"

    # CosmosClient 인스턴스를 생성합니다.
    # C#의 'new CosmosClient(endpoint, key)'와 동일한 역할을 합니다.
    client = CosmosClient(url=endpoint, credential=key)

    try:
        # 계정 속성을 가져옵니다.
        # C#의 'client.ReadAccountAsync()'와 유사합니다.
        account_properties = client.get_account_properties()
    
        # 계정 ID(이름)를 출력합니다.
        print(f"Account Name:\t{account_properties['id']}")
    
        # 쓰기 가능한 첫 번째 지역의 이름을 출력합니다.
        # Python SDK에서는 writable_locations가 리스트이며, 각 요소는 지역 정보를 담은 딕셔너리입니다.
        primary_region = account_properties['writable_locations'][0]['name']
        print(f"Primary Region:\t{primary_region}")

    except Exception as e:
        print(f"An error occurred: {e}")
    ```
    > **Note: Python SDK**
    >
    > - **`azure-cosmos`**: Python용 Cosmos DB SDK 패키지 이름입니다.
    > - **`CosmosClient`**: C#과 마찬가지로 서비스에 연결하기 위한 기본 클래스입니다. `url`과 `credential` 매개변수를 사용하여 초기화합니다.
    > - **`get_account_properties()`**: 계정의 속성을 동기적으로 가져옵니다. Python SDK의 많은 메서드는 동기적으로 작동하지만, 비동기 클라이언트(`azure.cosmos.aio.CosmosClient`)를 사용하면 비동기 작업도 가능합니다.

4.  `<cosmos-endpoint>`와 `<cosmos-key>`를 실제 값으로 바꾼 후 **script.py** 파일을 **저장**합니다.

## 스크립트 테스트

이제 Azure Cosmos DB for NoSQL 계정에 연결하는 코드가 완료되었으므로 스크립트를 테스트할 수 있습니다. 이 스크립트는 계정의 이름과 첫 번째 쓰기 가능 지역의 이름을 출력합니다. 계정을 생성할 때 지정한 위치가 이 스크립트의 결과로 출력되어야 합니다.

1.  **Visual Studio Code**에서 **04-sdk-connect** 폴더의 컨텍스트 메뉴를 열고 **Open in Integrated Terminal**을 선택하여 새 터미널 인스턴스를 엽니다.

2.  .NET 프로젝트를 빌드하고 실행하려면 [dotnet run][docs.microsoft.com/dotnet/core/tools/dotnet-run] 명령을 사용합니다.

    ```powershell
    dotnet run
    ```
   
   Python 스크립트를 실행하려면 다음 명령을 사용합니다.
   
    ```bash
    python script.py
    ```

3.  스크립트는 계정 이름과 첫 번째 쓰기 가능 지역을 출력합니다. 예를 들어, 계정 이름을 **dp420**으로, 첫 번째 쓰기 가능 지역을 **West US 2**로 지정했다면 스크립트는 다음과 같이 출력합니다.

    ```
    Account Name:   dp420
    Primary Region: West US 2
    ```

4.  통합 터미널을 닫습니다.
5.  **Visual Studio Code**를 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties.id]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties.id
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties.writableregions]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountproperties.writableregions
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountregion.name]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.accountregion.name
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient.readaccountasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient.readaccountasync
[docs.microsoft.com/dotnet/core/tools/dotnet-add-package]: https://docs.microsoft.com/dotnet/core/tools/dotnet-add-package
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/microsoft.azure.cosmos]: https://www.nuget.org/packages/Microsoft.Azure.Cosmos
