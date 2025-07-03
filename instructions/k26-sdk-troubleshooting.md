# Azure Cosmos DB for NoSQL SDK를 사용하여 애플리케이션 문제 해결하기

Azure Cosmos DB는 다양한 작업 유형에서 발생할 수 있는 문제를 쉽게 해결하는 데 도움이 되는 광범위한 응답 코드(response codes) 세트를 제공합니다. 핵심은 Azure Cosmos DB용 앱을 만들 때 적절한 오류 처리(error handling)를 프로그래밍하는 것입니다.

이 랩에서는 두 문서 중 하나를 삽입하거나 삭제할 수 있는 메뉴 기반 프로그램을 만듭니다. 이 랩의 주요 목적은 가장 일반적인 응답 코드 중 일부를 사용하는 방법과 이를 앱의 오류 처리 코드에서 사용하는 방법을 소개하는 것입니다. 여러 응답 코드에 대한 오류 처리를 코딩하지만, 두 가지 유형의 조건만 트리거할 것입니다. 또한 오류 처리는 복잡한 작업을 수행하지 않으며, 응답 코드에 따라 화면에 메시지를 표시하거나 10초간 기다린 후 작업을 한 번 더 재시도하는 정도입니다.

> **[배경지식: SDK에서의 예외 처리(Exception Handling)]**
> 견고한(robust) 애플리케이션을 구축하기 위한 가장 기본적이면서도 중요한 원칙은 예외 상황을 적절하게 처리하는 것입니다. Azure Cosmos DB와 같은 분산 시스템과 통신할 때는 다양한 오류가 발생할 수 있습니다.
> - **영구적 오류 (Permanent Errors):** 재시도해도 성공할 수 없는 오류입니다. 예를 들어, 이미 존재하는 ID로 문서를 생성하려는 시도(409 Conflict)나 권한 없는 작업 시도(403 Forbidden)가 있습니다. 이 경우, 애플리케이션은 작업을 중단하고 사용자에게 명확한 원인을 알려줘야 합니다.
> - **일시적 오류 (Transient Errors):** 잠시 후에 다시 시도하면 성공할 가능성이 있는 일시적인 오류입니다. 예를 들어, 네트워크 문제(408 Request Timeout)나 일시적인 서비스 과부하(503 Service Unavailable), 그리고 할당된 처리량 초과(429 Too Many Requests)가 있습니다. 이 경우, 애플리케이션은 즉시 실패 처리하는 대신, 잠시 대기한 후 자동으로 작업을 재시도하는 로직(Retry Pattern)을 구현해야 합니다.
>
> 이 랩에서는 .NET SDK가 제공하는 `CosmosException`을 사용하여 이러한 오류들을 어떻게 구분하고 처리하는지 실습합니다.

## 개발 환경 준비

이 랩을 진행하는 환경에 **DP-420**에 대한 랩 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 엽니다.

1.  **Visual Studio Code**를 시작합니다.

    > &#128221; Visual Studio Code 인터페이스에 익숙하지 않다면, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

2.  명령 팔레트(command palette)를 열고 **Git: Clone**을 실행하여 `https://github.com/microsoftlearning/dp-420-cosmos-db-dev` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

    > &#128161; **CTRL+SHIFT+P** 키보드 단축키를 사용하여 명령 팔레트를 열 수 있습니다.

3.  리포지토리가 복제되면, **Visual Studio Code**에서 선택한 로컬 폴더를 엽니다.

## Azure Cosmos DB for NoSQL 계정 생성하기

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다. Azure Cosmos DB for NoSQL 계정 프로비저닝이 완료되면, 엔드포인트(endpoint)와 키(key)를 검색할 수 있습니다. 엔드포인트와 키를 사용하여 프로그래밍 방식으로 Azure Cosmos DB for NoSQL 계정에 연결합니다. 이 정보는 Azure SDK for .NET이나 다른 SDK의 연결 문자열에 사용됩니다.

1.  새 웹 브라우저 창이나 탭에서 Azure portal (``portal.azure.com``)로 이동합니다.

2.  구독과 연결된 Microsoft 자격 증명을 사용하여 포털에 로그인합니다.

3.  **+ 리소스 만들기**를 선택하고, *Cosmos DB*를 검색한 후, 다음 설정을 사용하여 새로운 **Azure Cosmos DB for NoSQL** 계정 리소스를 생성합니다. 나머지 모든 설정은 기본값으로 둡니다:

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

7.  브라우저 창을 최소화하되 닫지는 마십시오. 다음 단계에서 백그라운드 워크로드를 시작한 후 몇 분 뒤에 Azure portal로 돌아올 것입니다.

## .NET 스크립트에 Microsoft.Azure.Cosmos 라이브러리 가져오기

.NET CLI는 사전 구성된 패키지 피드에서 패키지를 가져오는 [add package][docs.microsoft.com/dotnet/core/tools/dotnet-add-package] 명령을 포함합니다. .NET 설치는 NuGet을 기본 패키지 피드로 사용합니다.

1.  **Visual Studio Code**의 **탐색기(Explorer)** 창에서 **26-sdk-troubleshoot** 폴더로 이동합니다.

2.  **26-sdk-troubleshoot** 폴더의 컨텍스트 메뉴를 열고 **통합 터미널에서 열기(Open in Integrated Terminal)**를 선택하여 새 터미널 인스턴스를 엽니다.

    > &#128221; 이 명령은 시작 디렉터리가 이미 **26-sdk-troubleshoot** 폴더로 설정된 터미널을 엽니다.

3.  다음 명령을 사용하여 NuGet에서 [Microsoft.Azure.Cosmos][nuget.org/packages/microsoft.azure.cosmos/3.49.0] 패키지를 추가합니다:

    ```
    dotnet add package Microsoft.Azure.Cosmos --version 3.49.0
    ```

## 스크립트를 실행하여 문서 삽입 및 삭제를 위한 메뉴 기반 옵션 생성하기

애플리케이션을 실행하기 전에 Azure Cosmos DB 계정에 연결해야 합니다.

1.  **Visual Studio Code**의 **탐색기** 창에서 **26-sdk-troubleshoot** 폴더로 이동합니다.

2.  **Program.cs** 코드 파일을 엽니다.

3.  **endpoint**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **endpoint** 값으로 업데이트합니다.
  
    ```csharp
    private static readonly string endpoint = "<cosmos-endpoint>";
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이라면, C# 문은 **private static readonly string endpoint = "https&shy;://dp420.documents.azure.com:443/";** 가 됩니다.

4.  **key**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **key** 값으로 업데이트합니다.

    ```csharp
    private static readonly string key = "<cosmos-key>";
    ```

    > &#128221; 예를 들어, 키가 **fDR2ci9QgkdkvERTQ==** 라면, C# 문은 **private static readonly string key = "fDR2ci9QgkdkvERTQ==";** 가 됩니다.

5.  파일을 저장합니다.

6.  [dotnet run][docs.microsoft.com/dotnet/core/tools/dotnet-run] 명령을 사용하여 프로젝트를 빌드하고 실행합니다:

    ```
    dotnet run
    ```
    > &#128221; 이것은 매우 간단한 프로그램입니다. 아래와 같이 5개의 옵션이 있는 메뉴를 표시합니다. 두 개의 옵션은 미리 정의된 문서를 삽입하고, 두 개는 미리 정의된 문서를 삭제하며, 한 개는 프로그램을 종료하는 옵션입니다.

    >```
    >1) Add Document 1 with id = '0C297972-BE1B-4A34-8AE1-F39E6AA3D828'
    >2) Add Document 2 with id = 'AAFF2225-A5DD-4318-A6EC-B056F96B94B7'
    >3) Delete Document 1 with id = '0C297972-BE1B-4A34-8AE1-F39E6AA3D828'
    >4) Delete Document 2 with id = 'AAFF2225-A5DD-4318-A6EC-B056F96B94B7'
    >5) Exit
    >Select an option:
    >```

## 문서 삽입 및 삭제 시간

1.  **1**을 선택하고 **ENTER**를 눌러 첫 번째 문서를 삽입합니다. 프로그램은 첫 번째 문서를 삽입하고 다음 메시지를 반환합니다.

    ```
    Insert Successful.
    Document for customer with id = '0C297972-BE1B-4A34-8AE1-F39E6AA3D828' Inserted.
    Press [ENTER] to continue
    ```

2.  다시 **1**을 선택하고 **ENTER**를 눌러 첫 번째 문서를 삽입합니다. 이번에는 프로그램이 예외와 함께 비정상적으로 종료(crash)됩니다. 오류 스택을 살펴보면 프로그램 실패의 원인을 찾을 수 있습니다. 오류 스택에서 추출한 메시지에서 알 수 있듯이, 처리되지 않은 예외 "Conflict (409)"를 마주하게 됩니다.

    ```
    Unhandled exception. Microsoft.Azure.Cosmos.CosmosException : Response status code does not indicate success: Conflict (409);
    ```

3.  문서를 삽입하고 있으므로, 문서 생성 시 반환되는 일반적인 [create document status codes][/rest/api/cosmos-db/create-a-document#status-codes] 목록을 검토해야 합니다. 이 코드의 설명은 *새 문서에 제공된 ID가 기존 문서에 의해 이미 사용되었습니다* 입니다. 잠시 전에 동일한 문서를 생성하는 메뉴 옵션을 실행했기 때문에 이는 명백합니다.

4.  스택을 더 자세히 살펴보면, 이 예외가 100번째 줄에서 호출되었고, 이는 다시 64번째 줄에서 호출되었음을 알 수 있습니다.

    ```
    at Program.CreateDocument1(Container Customer) in C:\Git\dp-420-cosmos-db-dev\26-sdk-troubleshoot\Program.cs:line 100   
   at Program.CompleteTaskOnCosmosDB(String consoleinputcharacter, Container container) in C:\Git\dp-420-cosmos-db-dev\26-sdk-troubleshoot\Program.cs:line 64
    ```

5.  100번째 줄을 검토하면, 예상대로 오류가 *CreateItemAsync* 작업으로 인해 발생했음을 알 수 있습니다.

    ```C#
        ItemResponse<customerInfo> response = await Customer.CreateItemAsync<customerInfo>(customer, new PartitionKey(customerID));
    ```

6.  또한 100~103번째 줄을 검토하면 이 코드에 오류 처리가 전혀 없다는 것이 명백합니다. 이를 수정해야 합니다.

    ```C#
        ItemResponse<customerInfo> response = await Customer.CreateItemAsync<customerInfo>(customer, new PartitionKey(customerID));
        Console.WriteLine("Insert Successful.");
        Console.WriteLine("Document for customer with id = '" + customerID + "' Inserted.");
    ```

7.  오류 처리 코드가 무엇을 해야 할지 결정해야 합니다. [create document status codes][/rest/api/cosmos-db/create-a-document#status-codes]를 검토하여 이 작업에 대한 모든 가능한 상태 코드에 대한 오류 처리 코드를 만들 수 있습니다. 이 랩에서는 이 목록에서 상태 코드 403과 409만 고려할 것입니다. 반환되는 다른 모든 상태 코드는 시스템 오류 메시지만 표시합니다.

    > &#128221; 이 랩에서는 403 예외에 대한 오류 처리 작업을 코딩하지만, 403 예외를 생성하지는 않을 것입니다.

8.  **CompleteTaskOnCosmosDB**라는 함수에 대한 오류 처리를 추가해 봅시다. **Main** 함수의 **45**번째 줄에 있는 **while** 루프를 찾아 **CompleteTaskOnCosmosDB** 호출을 오류 처리 코드로 감쌉니다. **47**번째 줄의 **CompleteTaskOnCosmosDB** 문을 아래 코드로 대체합니다. 이 새 코드에서 가장 먼저 주목할 점은 **catch** 블록에서 **CosmosException** 클래스 타입의 예외를 캡처한다는 것입니다. 이 클래스는 Azure Cosmos DB 서비스로부터의 요청 완료 상태 코드를 반환하는 **StatusCode** 속성을 포함합니다. **StatusCode** 속성은 **System.Net.HttpStatusCode** 타입이므로, 이 값을 사용하여 .NET [HTTP Status Code][dotnet/api/system.net.httpstatuscode]의 필드 이름과 비교할 수 있습니다.

    > **[Note: `CosmosException`과 `StatusCode`]**
    > - **`CosmosException`**: Cosmos DB .NET SDK에서 서비스와 관련된 모든 오류는 이 예외 클래스로 캡슐화되어 전달됩니다. 일반적인 `Exception` 대신 이 특정 예외 타입을 `catch`하는 것이 중요합니다.
    > - **`StatusCode`**: `CosmosException` 객체의 가장 중요한 속성입니다. `System.Net.HttpStatusCode` 열거형(enum) 타입으로, 실패의 원인을 나타내는 HTTP 상태 코드(예: `HttpStatusCode.Conflict`, `HttpStatusCode.NotFound`)를 담고 있습니다. 이를 통해 개발자는 오류 유형에 따라 분기 처리를 할 수 있습니다. 이 랩에서는 편의상 `e.StatusCode.ToString()`을 사용해 문자열로 비교합니다.

    ```C#
        try
        {
            await CompleteTaskOnCosmosDB(consoleinputcharacter, CustomersDB_Customer_container);
        }
        catch (CosmosException e)
        {
                    switch (e.StatusCode.ToString())
                    {
                        case ("Conflict"):
                            Console.WriteLine("Insert Failed. Response Code (409).");
                            Console.WriteLine("Can not insert a duplicate partition key, customer with the same ID already exists."); 
                            break;
                        case ("Forbidden"):
                            Console.WriteLine("Response Code (403).");
                            Console.WriteLine("The request was forbidden to complete. Some possible reasons for this exception are:");
                            Console.WriteLine("Firewall blocking requests.");
                            Console.WriteLine("Partition key exceeding storage.");
                            Console.WriteLine("Non-data operations are not allowed.");
                            break;
                        default:
                            Console.WriteLine(e.Message);
                            break;
                    }
        }
    ```

9.  파일을 저장하고, 프로그램이 비정상 종료되었으므로 메뉴 프로그램을 다시 실행해야 합니다. 다음 명령을 실행하세요:

    ```
    dotnet run
    ```
 
10. 다시 **1**을 선택하고 **ENTER**를 눌러 첫 번째 문서를 삽입합니다. 이번에는 프로그램이 비정상 종료되지 않고 무슨 일이 일어났는지에 대한 더 사용자 친화적인 메시지를 받게 됩니다.

    ```
    Insert Failed. 
    Response Code (409).
    Can not insert a duplicate partition key, customer with the same ID already exists.
    ```

11. 이 코드는 *403*과 *409* 예외에 대한 오류 처리를 추가했습니다. 이제 몇 가지 일반적인 통신 유형의 예외에 대한 코드를 추가해 봅시다. 세 가지 일반적인 통신 유형의 예외는 *429*, *503*, *408* 또는 각각 너무 많은 요청, 서비스 사용 불가, 요청 시간 초과입니다. *66*번째 줄 근처에 **default** 문이 있어야 합니다. 이전 **break;** 문 바로 뒤, **default** 문 바로 앞에 아래 코드를 추가하세요. 이 코드는 이러한 통신 예외 중 하나라도 발견되면 10초간 기다린 후 문서를 한 번 더 삽입하려고 시도합니다. 아래 코드를 추가해 봅시다:

    > **[Note: 과도적 오류 처리 및 재시도 패턴 (Transient Fault Handling and Retry Pattern)]**
    > 429(TooManyRequests), 503(ServiceUnavailable), 408(RequestTimeout)은 일시적인 문제일 가능성이 높은 **과도적 오류(transient errors)**입니다. 이러한 오류를 받았을 때 즉시 실패 처리하는 대신, 잠시 기다렸다가 작업을 재시도하는 것은 분산 시스템에서 안정성을 높이는 핵심적인 패턴입니다. 이 코드에서는 `Task.Delay`를 사용해 간단한 재시도 로직을 구현합니다. 실제 프로덕션 코드에서는 보통 지수 백오프(exponential backoff)와 같은 더 정교한 재시도 전략을 사용합니다.

    ```C#
                        case ("TooManyRequests"):
                        case ("ServiceUnavailable"):
                        case ("RequestTimeout"):
                            // Check if the issues are related to connectivity and if so, wait 10 seconds to retry.
                            await Task.Delay(10000); // Wait 10 seconds
                            try
                            {
                                Console.WriteLine("Try one more time...");
                                await CompleteTaskOnCosmosDB(consoleinputcharacter, CustomersDB_Customer_container);
                            }
                            catch (CosmosException e2)
                            {
                                Console.WriteLine("Insert Failed. " + e2.Message);
                                Console.WriteLine("Can not insert a duplicate partition key, Connectivity issues encountered.");
                                break;
                            }
                            break;
    ```

    > &#128221; 이 랩에서는 429, 503 또는 408 예외가 발생했을 때 수행할 작업을 코딩하지만, 이러한 유형의 예외로 오류를 생성하지는 않을 것입니다.

12. 이제 **Main** 함수는 다음과 같이 보일 것입니다:

    ```C#
        public static async Task Main(string[] args)
        {
            // ... (client, database, container 생성 코드 생략) ...
            while((consoleinputcharacter = Console.ReadLine()) != "5") 
            {
                 try
                 {
                     await CompleteTaskOnCosmosDB(consoleinputcharacter, CustomersDB_Customer_container);
                 }
                 catch (CosmosException e)
                 {
                     switch (e.StatusCode.ToString())
                     {
                        case ("Conflict"):
                            // ...
                            break;
                        case ("Forbidden"):
                            // ...
                            break;
                        case ("TooManyRequests"):
                        case ("ServiceUnavailable"):
                        case ("RequestTimeout"):
                            // ... (재시도 로직)
                            break;
                        default:
                            Console.WriteLine(e.Message);
                            break;
                     }
                }
                // ... (메뉴 출력 코드 생략) ...
            }
        }
    ```

13. **CreateDocument2** 함수도 위의 변경 사항에 의해 수정될 것입니다.

14. 마지막으로, **DeleteDocument1**과 **DeleteDocument2** 함수도 **CreateDocument1** 함수와 유사하게 적절한 오류 처리 코드로 교체해야 합니다. 이 함수들은 **CreateItemAsync** 대신 **DeleteItemAsync**를 사용한다는 점 외에 유일한 차이점은 [deletes status codes][/rest/api/cosmos-db/delete-a-document]가 삽입 상태 코드와 다르다는 것입니다. 삭제의 경우, 문서가 없음을 나타내는 **404** 상태 코드만 신경 씁니다. **CompleteTaskOnCosmosDB** 함수 호출의 오류 처리에 추가 case를 업데이트해 봅시다. **Main** 함수에서 **default** case 위에 다음 코드를 추가해야 합니다:

    ```C#
                    case ("NotFound"):
                        Console.WriteLine("Delete Failed. Response Code (404).");
                        Console.WriteLine("Can not delete customer, customer not found.");
                        break;         
    ```

15. 파일을 저장합니다.

16. 모든 함수 수정을 마친 후, 모든 메뉴 옵션을 여러 번 테스트하여 예외 발생 시 앱이 비정상 종료되지 않고 메시지를 반환하는지 확인하세요. 앱이 비정상 종료되면 오류를 수정하고 다음 명령을 다시 실행하세요:

    ```
    dotnet run
    ```

17. 엿보지 마세요. 하지만 다 마치면, `Main` 코드는 다음과 같이 보일 것입니다.

    ```C#
        public static async Task Main(string[] args)
        {
            // ... (client, database, container 생성 코드 생략) ...
    
            while((consoleinputcharacter = Console.ReadLine()) != "5") 
            {
                    try
                    {
                        await CompleteTaskOnCosmosDB(consoleinputcharacter, CustomersDB_Customer_container);
                    }
                    catch (CosmosException e)
                    {
                        // [코드 설명]
                        // Cosmos DB 작업에서 발생하는 예외를 잡아서 처리합니다.
                        switch (e.StatusCode.ToString())
                        {
                            // 409 Conflict: 이미 존재하는 ID로 문서를 생성하려고 할 때 발생
                            case ("Conflict"):
                                Console.WriteLine("Insert Failed. Response Code (409).");
                                Console.WriteLine("Can not insert a duplicate partition key, customer with the same ID already exists."); 
                                break;
                            // 403 Forbidden: 권한 부족, 방화벽 차단 등 접근이 금지되었을 때 발생
                            case ("Forbidden"):
                                Console.WriteLine("Response Code (403).");
                                Console.WriteLine("The request was forbidden to complete. Some possible reasons for this exception are:");
                                Console.WriteLine("Firewall blocking requests.");
                                Console.WriteLine("Partition key exceeding storage.");
                                Console.WriteLine("Non-data operations are not allowed.");
                                break;
                            // 429, 503, 408: 일시적인 통신 또는 처리량 문제
                            case ("TooManyRequests"):
                            case ("ServiceUnavailable"):
                            case ("RequestTimeout"):
                                // 10초 대기 후 한 번 더 재시도하는 패턴 구현
                                await Task.Delay(10000); 
                                try
                                {
                                    Console.WriteLine("Try one more time...");
                                    await CompleteTaskOnCosmosDB(consoleinputcharacter, CustomersDB_Customer_container);
                                }
                                catch (CosmosException e2) // 재시도도 실패하면 최종 실패 처리
                                {
                                    Console.WriteLine("Insert Failed. " + e2.Message);
                                    Console.WriteLine("Can not insert a duplicate partition key, Connectivity issues encountered.");
                                    break;
                                }
                                break;    
                            // 404 Not Found: 존재하지 않는 문서를 삭제하려고 할 때 발생
                            case ("NotFound"):
                                Console.WriteLine("Delete Failed. Response Code (404).");
                                Console.WriteLine("Can not delete customer, customer not found.");
                                break; 
                            // 그 외 모든 예외는 기본 메시지 출력
                            default:
                                Console.WriteLine(e.Message);
                                break;
                        }
                    }
                // ... (메뉴 출력 코드 생략) ...
            }
        }
    ```

## 결론

가장 초보 개발자라도 모든 코드에 적절한 오류 처리를 추가해야 한다는 것을 알고 있습니다. 이 코드의 오류 처리는 간단하지만, 코드에서 견고한 오류 처리 솔루션을 만드는 데 필요한 Azure Cosmos DB 예외 구성 요소에 대한 기본 사항을 제공했을 것입니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/core/tools/dotnet-add-package]: https://docs.microsoft.com/dotnet/core/tools/dotnet-add-package
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/microsoft.azure.cosmos/3.22.1]: https://www.nuget.org/packages/Microsoft.Azure.Cosmos/3.22.1
[/rest/api/cosmos-db/create-a-document#status-codes]:https://docs.microsoft.com/rest/api/cosmos-db/create-a-document#status-codes
[dotnet/api/system.net.httpstatuscode]:https://docs.microsoft.com/dotnet/api/system.net.httpstatuscode?view=net-6.0
[/rest/api/cosmos-db/delete-a-document]:https://docs.microsoft.com/rest/api/cosmos-db/delete-a-document#status-codes
