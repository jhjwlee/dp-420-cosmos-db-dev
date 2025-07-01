# Azure Cosmos DB for NoSQL SDK를 사용하여 여러 문서를 대량으로 이동

대량 작업(bulk operation)을 수행하는 방법을 배우는 가장 쉬운 방법은 클라우드의 Azure Cosmos DB for NoSQL 계정에 많은 문서를 푸시하려고 시도하는 것입니다. SDK의 대량 기능을 사용하면 [System.Threading.Tasks][docs.microsoft.com/dotnet/api/system.threading.tasks] 네임스페이스의 약간의 도움으로 이 작업을 수행할 수 있습니다.

이 실습에서는 NuGet의 [Bogus][nuget.org/packages/bogus/33.1.1] 라이브러리를 사용하여 가상의 데이터를 생성하고 이를 Azure Cosmos DB 계정에 배치합니다.

> **배경지식: 대량 실행 (Bulk Execution)**
>
> **대량 실행(Bulk Execution)**은 다수의 점 작업(point operations)을 동시에 실행하여 처리량을 극대화하는 기능입니다. SDK는 내부적으로 이러한 작업들을 그룹화하여 서비스로 보내고, 요청이 제한(throttled)되거나 다른 일시적인 오류가 발생할 경우 자동으로 재시도합니다.
>
> **Transactional Batch와의 차이점:**
>
> | 기능 | **Transactional Batch (트랜잭션 일괄 처리)** | **Bulk Execution (대량 실행)** |
> | :--- | :--- | :--- |
> | **목표** | 데이터 일관성 (원자성 보장) | 처리량 극대화 (최고의 성능) |
> | **원자성** | **보장됨** (모두 성공 또는 모두 실패) | **보장되지 않음** (개별 작업으로 처리됨) |
> | **파티션 키** | 모든 항목이 **동일한** 논리 파티션 키를 가져야 함 | 모든 항목이 **다른** 논리 파티션 키를 가질 수 있음 |
> | **오류 처리**| 일괄 처리 전체가 실패하고 롤백됨 | SDK가 실패한 개별 작업을 자동으로 재시도함 |
> | **활성화** | `container.CreateTransactionalBatch()` 사용 | `CosmosClientOptions.AllowBulkExecution = true` 설정 |
>
> 이 실습에서는 **Bulk Execution**을 사용하여 서로 다른 파티션 키를 가질 수 있는 수많은 항목을 가능한 한 빨리 데이터베이스에 삽입하는 방법을 배웁니다.

## 개발 환경 준비

이 실습을 진행하는 환경에 **DP-420**에 대한 실습 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 여세요.

1.  **Visual Studio Code**를 시작합니다.
2.  명령 팔레트를 열고 **Git: Clone**을 실행하여 ``https://github.com/microsoftlearning/dp-420-cosmos-db-dev`` GitHub 리포지토리를 복제합니다.
3.  리포지토리가 복제되면, 선택한 로컬 폴더를 **Visual Studio Code**에서 엽니다.

## Azure Cosmos DB for NoSQL 계정 생성 및 SDK 프로젝트 구성

1.  새 웹 브라우저 창에서 Azure portal (``portal.azure.com``)로 이동합니다.
2.  포털에 로그인합니다.
3.  다음 설정으로 새로운 **Azure Cosmos DB for NoSQL** 계정 리소스를 생성합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Workload Type** | **Learning** |
| **Subscription** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Account Name** | *전역적으로 고유한 이름 입력* |
| **Location** | *사용 가능한 지역 선택* |
| **Capacity mode** | *Provisioned throughput* |
| **Apply Free Tier Discount** | *Do Not Apply* |

4.  배포가 완료될 때까지 기다립니다.
5.  생성된 **Azure Cosmos DB** 계정 리소스에서 **Keys** 창으로 이동하여 **URI**(endpoint)와 **PRIMARY KEY**(key)를 확인합니다.
6.  계정 리소스 내에서 **Data Explorer** 창으로 이동합니다.
7.  **New Container**를 선택하고 다음 설정으로 새 컨테이너를 만듭니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | *Create new* | `cosmicworks` |
| **Share throughput across containers**| *Do not select* |
| **Container id** | `products` |
| **Partition key** | `/categoryId` |
| **Container throughput**| *Autoscale* | `4000` |

    > **Note: 처리량 설정**
    >
    > 대량 삽입 작업은 많은 RU를 소모하므로, 이 실습에서는 처리량을 `4000` RU/s로 비교적 높게 설정합니다. Autoscale을 사용하면 실제 사용량에 따라 처리량이 자동으로 조정됩니다.

8.  **Visual Studio Code**로 돌아옵니다.
9.  **Explorer** 창에서 **08-sdk-bulk** 폴더로 이동합니다.
10. **script.cs** 파일을 엽니다.
11. **endpoint**와 **key** 변수에 Azure Portal에서 복사한 값을 설정합니다.
12. **script.cs** 파일을 **저장**합니다.
13. **08-sdk-bulk** 폴더의 터미널에서 다음 명령을 실행합니다.

    ```powershell
    dotnet add package Microsoft.Azure.Cosmos --version 3.49.0
    # Bogus 라이브러리 추가 (프로젝트 파일에 이미 포함되어 있을 수 있음)
    dotnet add package Bogus
    dotnet build
    ```
14. 통합 터미널을 닫습니다.

## 25,000개 문서 대량 삽입하기

"과감하게" 많은 문서를 삽입하여 이 기능이 어떻게 작동하는지 확인해 보겠습니다. 내부 테스트에서 이 작업은 실습 가상 머신과 Azure Cosmos DB for NoSQL 계정이 지리적으로 비교적 가까우면 약 1~2분이 소요될 수 있습니다.

### **C# 코드**

1.  **script.cs** 파일의 편집기 탭으로 돌아갑니다.

2.  **AllowBulkExecution** 속성을 **true**로 설정한 [CosmosClientOptions][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclientoptions]의 새 인스턴스 **options**를 생성합니다.

    ```csharp
    CosmosClientOptions options = new () 
    { 
        AllowBulkExecution = true 
    };
    ```

3.  **endpoint**, **key**, **options** 변수를 생성자 매개변수로 전달하여 **CosmosClient** 클래스의 새 인스턴스 **client**를 생성합니다.

    ```csharp
    CosmosClient client = new (endpoint, key, options); 
    ```

4.  **client** 변수의 [GetContainer][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient.getcontainer] 메서드를 사용하여 데이터베이스 이름(*cosmicworks*)과 컨테이너 이름(*products*)으로 기존 컨테이너를 검색합니다.

    ```csharp
    Container container = client.GetContainer("cosmicworks", "products");
    ```

5.  다음 샘플 코드를 사용하여 NuGet에서 가져온 Bogus 라이브러리의 **Faker** 클래스로 **25,000**개의 가상 제품을 생성합니다.

    ```csharp
    List<Product> productsToInsert = new Faker<Product>()
        .StrictMode(true)
        .RuleFor(o => o.id, f => Guid.NewGuid().ToString())
        .RuleFor(o => o.name, f => f.Commerce.ProductName())
        .RuleFor(o => o.price, f => Convert.ToDouble(f.Commerce.Price(max: 1000, min: 10, decimals: 2)))
        .RuleFor(o => o.categoryId, f => f.Commerce.Department(1))
        .Generate(25000);
    ```

    > 💡 **Bogus** 라이브러리는 사용자 인터페이스 애플리케이션 테스트를 위한 가상 데이터를 설계하는 데 사용되는 오픈 소스 라이브러리이며, 대량 가져오기/내보내기 애플리케이션 개발 방법을 배우는 데 매우 유용합니다.

6.  **Task** 타입의 새로운 제네릭 **List<>** **concurrentTasks**를 생성합니다.

    ```csharp
    List<Task> concurrentTasks = new List<Task>();
    ```

7.  이 애플리케이션의 앞부분에서 생성된 제품 목록을 반복하는 `foreach` 루프를 생성합니다.

    ```csharp
    foreach(Product product in productsToInsert)
    {
    }
    ```

8.  `foreach` 루프 내에서, 제품을 Azure Cosmos DB for NoSQL에 비동기적으로 삽입하는 **Task**를 생성하고, 파티션 키를 명시적으로 지정한 후 이 작업을 **concurrentTasks**라는 작업 목록에 추가합니다.

    ```csharp
    concurrentTasks.Add(
        container.CreateItemAsync(product, new PartitionKey(product.categoryId))
    );   
    ```
    > **Note: 동시성 관리**
    >
    > 여기서 핵심은 각 `CreateItemAsync` 호출이 즉시 실행되지 않고 `Task` 객체로 반환된다는 점입니다. 이 `Task`들을 리스트에 모아두고, 루프가 끝난 후 `Task.WhenAll`을 사용하여 모든 작업을 동시에 시작하고 완료될 때까지 기다립니다. 이 방식을 통해 SDK는 내부적으로 대량 실행 최적화를 수행하여 여러 요청을 효율적으로 처리합니다.

9.  `foreach` 루프 후에 **concurrentTasks** 변수에 대해 **Task.WhenAll**의 결과를 비동기적으로 기다립니다.

    ```csharp
    await Task.WhenAll(concurrentTasks);
    ```

10. 내장 **Console.WriteLine** 정적 메서드를 사용하여 콘솔에 **Bulk tasks complete**라는 정적 메시지를 출력합니다.

    ```csharp
    Console.WriteLine("Bulk tasks complete");
    ```

---

### **Python 코드 (추가)**

Python에서는 `asyncio` 라이브러리를 사용하여 유사한 동시성 패턴을 구현할 수 있습니다.

1.  필요한 라이브러리를 설치합니다.

    ```bash
    pip install azure-cosmos Faker asyncio
    ```

2.  **08-sdk-bulk** 폴더에 **script.py** 파일을 만들고 다음 코드를 추가합니다.

    ```python
    import asyncio
    from faker import Faker
    from azure.cosmos.aio import CosmosClient as CosmosClientAio
    from azure.cosmos import PartitionKey

    endpoint = "<cosmos-endpoint>"
    key = "<cosmos-key>"

    async def main():
        # Python에서 대량 실행을 하려면 비동기 클라이언트를 사용해야 합니다.
        # Bulk execution is implicitly enabled in the Python SDK aio client.
        async with CosmosClientAio(url=endpoint, credential=key) as client:
            database = await client.create_database_if_not_exists("cosmicworks")
            container = await database.create_container_if_not_exists(
                id="products",
                partition_key=PartitionKey(path="/categoryId"),
                offer_throughput=4000
            )
            
            print("Generating fake data...")
            fake = Faker()
            products_to_insert = [
                {
                    'id': str(fake.uuid4()),
                    'name': fake.ecommerce_name(),
                    'price': float(fake.pydecimal(left_digits=3, right_digits=2, positive=True, min_value=10, max_value=1000)),
                    'categoryId': fake.ecommerce_category()
                } for _ in range(25000)
            ]
            
            print("Starting bulk insert...")
            tasks = []
            for product in products_to_insert:
                # 각 아이템 생성 작업을 비동기 작업으로 추가
                tasks.append(container.create_item(product))
            
            # 모든 작업을 동시에 실행
            await asyncio.gather(*tasks)
            
            print("Bulk tasks complete")

    if __name__ == "__main__":
        asyncio.run(main())

    ```
    > **Note: Python의 대량 실행**
    >
    > - **`azure.cosmos.aio`**: Python SDK에서 대량 실행을 활용하려면 비동기 I/O를 지원하는 `aio` 클라이언트를 사용해야 합니다. 비동기 클라이언트에서는 대량 실행이 기본적으로 활성화되어 있어 별도의 옵션 설정이 필요 없습니다.
    > - **`asyncio.gather`**: C#의 `Task.WhenAll`과 유사한 역할을 합니다. 여러 비동기 작업(coroutine)을 리스트로 받아 동시에 실행하고 모든 작업이 완료될 때까지 기다립니다.

---

11. **script.cs** (또는 **script.py**) 파일을 **저장**하고 실행합니다.
12. 애플리케이션은 조용히 실행되며, 완료되기까지 약 1~2분이 소요됩니다.

## 결과 관찰

이제 Azure Cosmos DB에 25,000개의 항목을 보냈으니 Data Explorer로 가서 확인해 보겠습니다.

1.  웹 브라우저로 돌아가 **Data Explorer** 창으로 이동합니다.
2.  **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장한 다음, **products** 컨테이너 노드를 관찰합니다.
3.  **products** 노드를 확장하고 **Items** 노드를 선택하여 컨테이너 내의 항목 목록을 관찰합니다.
4.  **products** 컨테이너 노드를 선택하고 **New SQL Query**를 선택합니다.
5.  편집기 영역의 내용을 삭제합니다.
6.  대량 작업으로 생성된 모든 문서의 수를 반환하는 새 SQL 쿼리를 작성합니다.

    ```sql
    SELECT VALUE COUNT(1) FROM c
    ```
    *수정: `SELECT COUNT(1) FROM items` -> `SELECT VALUE COUNT(1) FROM c` 로 변경하여 숫자만 반환하도록 함.*

7.  **Execute Query**를 선택합니다.
8.  컨테이너의 항목 수를 관찰합니다. 25,000에 가까운 숫자가 표시되어야 합니다.
9.  웹 브라우저 창 또는 탭을 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient.getcontainer]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclient.getcontainer
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclientoptions]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.cosmosclientoptions
[docs.microsoft.com/dotnet/api/system.threading.tasks]: https://docs.microsoft.com/dotnet/api/system.threading.tasks
[docs.microsoft.com/dotnet/core/tools/dotnet-build]: https://docs.microsoft.com/dotnet/core/tools/dotnet-build
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/bogus/33.1.1]: https://www.nuget.org/packages/bogus/33.1.1
