# Azure Cosmos DB for NoSQL SDK를 사용하여 문서 생성 및 업데이트

[Microsoft.Azure.Cosmos.Container][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container] 클래스는 Azure Cosmos DB for NoSQL 컨테이너 내에서 항목(items)을 생성, 검색, 업데이트 및 삭제하기 위한 멤버 메서드 집합을 포함합니다. 이 메서드들은 함께 NoSQL API 컨테이너 내의 다양한 항목에 걸쳐 가장 일반적인 "CRUD" 작업을 수행합니다.

이 실습에서는 SDK를 사용하여 Azure Cosmos DB for NoSQL 컨테이너 내의 항목에 대한 일상적인 CRUD 작업을 수행합니다.

> **배경지식: CRUD 및 Point Operations**
>
> **CRUD**는 데이터베이스 작업의 네 가지 기본 기능인 **Create(생성)**, **Read(읽기)**, **Update(업데이트)**, **Delete(삭제)**를 나타내는 약어입니다. 이 실습에서는 SDK를 통해 이 네 가지 작업을 모두 수행합니다.
>
> **Point Operation(점 작업)**은 항목의 고유 `id`와 `partitionKey`를 사용하여 단일 항목을 대상으로 하는 작업을 의미합니다. 이는 가장 효율적인 작업 유형으로, 특정 항목을 직접 찾아가기 때문에 대규모 컨테이너에서도 매우 빠르고 낮은 RU 비용으로 실행됩니다. 이 실습에서 다루는 `CreateItemAsync`, `ReadItemAsync`, `UpsertItemAsync`, `DeleteItemAsync`는 모두 점 작업에 해당합니다.

## 개발 환경 준비

이 실습을 진행하는 환경에 **DP-420**에 대한 실습 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 여세요.

1.  **Visual Studio Code**를 시작합니다.

    > 📝 Visual Studio Code 인터페이스에 익숙하지 않은 경우, [Visual Studio Code 시작 설명서][code.visualstudio.com/docs/getstarted]를 검토하세요.

2.  명령 팔레트를 열고 **Git: Clone**을 실행하여 ``https://github.com/microsoftlearning/dp-420-cosmos-db-dev`` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

    > 💡 **CTRL+SHIFT+P** 키보드 단축키를 사용하여 명령 팔레트를 열 수 있습니다.

3.  리포지토리가 복제되면, 선택한 로컬 폴더를 **Visual Studio Code**에서 엽니다.

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

    > 📝 실습 환경에서는 새 리소스 그룹 생성을 막는 제약이 있을 수 있습니다. 그럴 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

4.  이 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

5.  새로 생성된 **Azure Cosmos DB** 계정 리소스로 이동하여 **Keys** 창으로 이동합니다.

6.  이 창에는 SDK에서 계정에 연결하는 데 필요한 연결 정보와 자격 증명이 포함되어 있습니다. 특히 다음을 확인합니다.
    1.  **URI** 필드. 이 **endpoint** 값은 나중에 이 연습에서 사용됩니다.
    2.  **PRIMARY KEY** 필드. 이 **key** 값은 나중에 이 연습에서 사용됩니다.

7.  **Visual Studio Code**로 다시 전환합니다.

## SDK에서 Azure Cosmos DB for NoSQL 계정에 연결

새로 생성된 계정의 자격 증명을 사용하여 SDK 클래스로 연결하고 새로운 데이터베이스와 컨테이너 인스턴스를 생성합니다. 그런 다음, Data Explorer를 사용하여 인스턴스가 Azure portal에 존재하는지 확인합니다.

1.  **Visual Studio Code**의 **Explorer** 창에서 **06-sdk-crud** 폴더로 이동합니다.

2.  **06-sdk-crud** 폴더의 컨텍스트 메뉴를 열고 **Open in Integrated Terminal**을 선택하여 새 터미널 인스턴스를 엽니다.

    > 📝 이 명령은 시작 디렉토리가 이미 **06-sdk-crud** 폴더로 설정된 터미널을 엽니다.

3.  다음 명령을 사용하여 NuGet에서 [Microsoft.Azure.Cosmos][nuget.org/packages/microsoft.azure.cosmos/3.22.1] 패키지를 추가합니다.

    ```powershell
    dotnet add package Microsoft.Azure.Cosmos --version 3.49.0
    ```

4.  [dotnet build][docs.microsoft.com/dotnet/core/tools/dotnet-build] 명령을 사용하여 프로젝트를 빌드합니다.

    ```powershell
    dotnet build
    ```

5.  통합 터미널을 닫습니다.

6.  **06-sdk-crud** 폴더 내의 **script.cs** 코드 파일을 엽니다.

7.  **endpoint**라는 이름의 `string` 변수를 찾습니다. 값을 이전에 생성한 Azure Cosmos DB 계정의 **endpoint**로 설정합니다.

    ```csharp
    string endpoint = "<cosmos-endpoint>";
    ```

    > 📝 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/**이면 C# 문은 **string endpoint = "https&shy;://dp420.documents.azure.com:443/";**가 됩니다.

8.  **key**라는 이름의 `string` 변수를 찾습니다. 값을 이전에 생성한 Azure Cosmos DB 계정의 **key**로 설정합니다.

    ```csharp
    string key = "<cosmos-key>";
    ```

    > 📝 예를 들어, 키가 **fDR2ci9QgkdkvERTQ==**이면 C# 문은 **string key = "fDR2ci9QgkdkvERTQ==";**가 됩니다.

### **C# 코드**

9.  **client** 변수의 `CreateDatabaseIfNotExistsAsync` 메서드를 비동기적으로 호출하여 생성하려는 새 데이터베이스 이름(**cosmicworks**)을 전달하고, 결과를 **Database** 타입의 변수에 저장합니다.

    ```csharp
    Database database = await client.CreateDatabaseIfNotExistsAsync("cosmicworks");
    ```

10. **database** 변수의 `CreateContainerIfNotExistsAsync` 메서드를 비동기적으로 호출하여 **cosmicworks** 데이터베이스 내에 생성하려는 새 컨테이너 이름(**products**), 파티션 키 경로(**/categoryId**), 처리량(**400**)을 전달하고, 결과를 **Container** 타입의 변수에 저장합니다.

    ```csharp
    Container container = await database.CreateContainerIfNotExistsAsync("products", "/categoryId", 400);    
    ```
    > **Note: `...IfNotExistsAsync` 메서드**
    >
    > `CreateDatabaseIfNotExistsAsync` 및 `CreateContainerIfNotExistsAsync`와 같은 메서드는 멱등성(idempotent)을 가집니다. 즉, 여러 번 실행해도 결과가 동일합니다. 데이터베이스나 컨테이너가 이미 존재하면 새로 만들지 않고 기존 인스턴스를 반환하므로, 애플리케이션 초기화 코드에서 안전하게 사용할 수 있습니다.

11. 완료되면 코드 파일은 이제 다음을 포함해야 합니다.

    ```csharp
    using System;
    using Microsoft.Azure.Cosmos;

    string endpoint = "<cosmos-endpoint>";
    string key = "<cosmos-key>";

    CosmosClient client = new CosmosClient(endpoint, key);
    
    Database database = await client.CreateDatabaseIfNotExistsAsync("cosmicworks");
    
    Container container = await database.CreateContainerIfNotExistsAsync("products", "/categoryId", 400);
    ```

---

### **Python 코드 (추가)**

C#과 동일한 작업을 수행하는 Python 코드입니다. **06-sdk-crud** 폴더에 **script.py** 파일을 만들고 다음 코드를 추가합니다.

1.  터미널에서 Python용 SDK를 설치합니다. `pip install azure-cosmos`

2.  **script.py** 파일에 코드를 작성합니다.

    ```python
    from azure.cosmos import CosmosClient, PartitionKey

    endpoint = "<cosmos-endpoint>"
    key = "<cosmos-key>"

    # CosmosClient 인스턴스 생성
    client = CosmosClient(url=endpoint, credential=key)

    # 데이터베이스 생성 또는 가져오기
    database_name = "cosmicworks"
    database = client.create_database_if_not_exists(id=database_name)

    # 컨테이너 생성 또는 가져오기
    container_name = "products"
    partition_key_path = "/categoryId"
    container = database.create_container_if_not_exists(
        id=container_name, 
        partition_key=PartitionKey(path=partition_key_path),
        offer_throughput=400
    )

    print(f"Database '{database.id}' and container '{container.id}' are ready.")
    ```

---

12. **script.cs** (또는 **script.py**) 코드 파일을 **저장**합니다.

13. **Visual Studio Code**에서 **06-sdk-crud** 폴더의 컨텍스트 메뉴를 열고 **Open in Integrated Terminal**을 선택하여 새 터미널 인스턴스를 엽니다.

14. [dotnet run][docs.microsoft.com/dotnet/core/tools/dotnet-run] 명령을 사용하여 프로젝트를 빌드하고 실행합니다.

    ```powershell
    dotnet run
    ```
    Python의 경우: `python script.py`

15. 통합 터미널을 닫습니다.

16. 웹 브라우저 창으로 전환합니다.

17. **Azure Cosmos DB** 계정 리소스 내에서 **Data Explorer** 창으로 이동합니다.

18. **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장한 다음, **NOSQL API** 탐색 트리 내에서 새로운 **products** 컨테이너 노드를 관찰합니다.

## SDK를 사용하여 항목에 대한 생성 및 읽기 점 작업 수행

이제 Microsoft.Azure.Cosmos.Container 클래스의 비동기 메서드 집합을 사용하여 NoSQL API 컨테이너 내의 항목에 대한 일반적인 작업을 수행합니다. 이러한 작업은 모두 C#의 작업 비동기 프로그래밍 모델을 사용하여 수행됩니다.

1.  **Visual Studio Code**로 돌아갑니다. **06-sdk-crud** 폴더 내의 **product.cs** 코드 파일을 엽니다.

    > 📝 **script.cs** 파일의 편집기를 닫지 마세요.

2.  이 코드 파일 내의 **Product** 클래스를 관찰합니다. 이 클래스는 이 컨테이너에 저장되고 조작될 제품 항목을 나타냅니다.

3.  **script.cs** 파일의 편집기 탭으로 돌아갑니다.

### **C# 코드**

4.  다음 속성을 가진 **saddle**이라는 이름의 **Product** 타입의 새 객체를 만듭니다.

| Property | Value |
| :--- | :--- |
| **id** | `706cd7c6-db8b-41f9-aea2-0e0c7e8eb009` |
| **categoryId** | `9603ca6c-9e28-4a02-9194-51cdb7fea816` |
| **name** | *Road Saddle* |
| **price** | `45.99d` |
| **tags** | `{ tan, new, crisp }` |

    
    ```csharp
    Product saddle = new()
    {
        id = "706cd7c6-db8b-41f9-aea2-0e0c7e8eb009",
        categoryId = "9603ca6c-9e28-4a02-9194-51cdb7fea816",
        name = "Road Saddle",
        price = 45.99d,
        tags = new string[]
        {
            "tan",
            "new",
            "crisp"
        }
    };
    ```

5.  **container** 변수의 제네릭 [CreateItemAsync<>][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.createitemasync] 메서드를 비동기적으로 호출하여 **saddle** 변수를 메서드 매개변수로 전달하고 **Product**를 제네릭 타입으로 사용합니다.

    ```csharp
    await container.CreateItemAsync<Product>(saddle);
    ```

...*이후, 코드를 실행하고 항목이 생성되었는지 확인한 다음, 생성 코드를 삭제하고 읽기 코드를 추가하는 과정이 이어집니다.*

다음은 생성 및 읽기 작업을 합친 최종 코드입니다.

14. **id**와 **categoryId** 변수를 정의합니다.

    ```csharp
    string id = "706cd7c6-db8b-41f9-aea2-0e0c7e8eb009";
    string categoryId = "9603ca6c-9e28-4a02-9194-51cdb7fea816";
    ```
15. **categoryId**를 사용하여 **PartitionKey** 객체를 생성합니다.

    ```csharp
    PartitionKey partitionKey = new (categoryId);
    ```

16. **container** 변수의 제네릭 [ReadItemAsync<>][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.readitemasync] 메서드를 비동기적으로 호출하여 **id**와 **partitionkey** 변수를 전달하고, **Product**를 제네릭 타입으로 사용하여 결과를 **Product** 타입의 **saddle** 변수에 저장합니다.

    ```csharp
    Product saddle = await container.ReadItemAsync<Product>(id, partitionKey);
    ```

17. 정적 **Console.WriteLine** 메서드를 호출하여 서식화된 출력 문자열을 사용하여 saddle 객체를 인쇄합니다.

    ```csharp
    Console.WriteLine($"[{saddle.id}]\t{saddle.name} ({saddle.price:C})");
    ```

---

### **Python 코드 (추가)**

Python에서는 딕셔너리를 사용하여 항목을 표현합니다. **script.py**에 다음 코드를 추가합니다.

```python
# 생성할 항목 정의 (Python 딕셔너리 사용)
saddle_item = {
    'id': '706cd7c6-db8b-41f9-aea2-0e0c7e8eb009',
    'categoryId': '9603ca6c-9e28-4a02-9194-51cdb7fea816',
    'name': 'Road Saddle',
    'price': 45.99,
    'tags': ['tan', 'new', 'crisp']
}

# 항목 생성
container.create_item(body=saddle_item)
print(f"Created item with id: {saddle_item['id']}")

# 생성된 항목 읽기
item_id = "706cd7c6-db8b-41f9-aea2-0e0c7e8eb009"
partition_key_value = "9603ca6c-9e28-4a02-9194-51cdb7fea816"

saddle = container.read_item(item=item_id, partition_key=partition_key_value)

# 읽어온 항목 정보 출력
# Python의 f-string을 사용하여 C#의 형식화된 출력과 유사하게 만듭니다.
print(f"[{saddle['id']}]\t{saddle['name']} (${saddle['price']:.2f})")
```
> **Note: Python SDK CRUD 메서드**
>
> - `create_item(body=...)`: 새 항목을 생성합니다. `body` 매개변수에 딕셔너리 형태의 항목을 전달합니다.
> - `read_item(item=..., partition_key=...)`: `id`와 `partitionKey`를 사용하여 단일 항목을 읽습니다.

---

*실습 지시에 따라 코드를 실행하고 결과를 확인합니다.*

## SDK를 사용하여 업데이트 및 삭제 점 작업 수행

SDK를 배울 때, 온라인 Azure Cosmos DB SDK 계정이나 에뮬레이터를 사용하여 항목을 업데이트하고, 작업을 수행하고 변경 사항이 적용되었는지 확인하면서 Data Explorer와 원하는 IDE를 오가는 것은 드문 일이 아닙니다. 여기서는 SDK를 사용하여 항목을 업데이트하고 삭제하면서 바로 그 작업을 수행합니다.

*... Data Explorer에서 항목을 확인하는 단계를 따릅니다 ...*

### **C# 코드**

1.  **script.cs** 파일로 돌아갑니다.

2.  `ReadItemAsync`로 읽어온 `saddle` 객체의 속성을 변경합니다.

    ```csharp
    saddle.price = 32.55d;
    saddle.name = "Road LL Saddle";
    ```

3.  **container** 변수의 제네릭 [UpsertItemAsync<>][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.upsertitemasync] 메서드를 비동기적으로 호출하여 수정된 **saddle** 변수를 전달합니다.

    ```csharp
    await container.UpsertItemAsync<Product>(saddle);
    ```
    > **Note: `UpsertItemAsync` vs `ReplaceItemAsync`**
    >
    > - `UpsertItemAsync`: "Update or Insert"의 줄임말입니다. 지정된 `id`의 항목이 있으면 **업데이트**하고, 없으면 **새로 생성**합니다.
    > - `ReplaceItemAsync`: 지정된 `id`의 항목이 있을 때만 **업데이트**합니다. 항목이 없으면 오류가 발생합니다.

*...코드를 실행하고 Data Explorer에서 변경 사항을 확인한 후, 업데이트 코드를 삭제하고 삭제 코드를 추가하는 과정으로 이어집니다.*

4.  업데이트 코드 대신 다음 삭제 코드를 추가합니다.

    ```csharp
    await container.DeleteItemAsync<Product>(id, partitionKey);
    ```

### **Python 코드 (추가)**

Python에서도 동일한 업데이트 및 삭제 작업을 수행할 수 있습니다.

```python
# 이전에 읽어온 'saddle' 딕셔너리 사용
# 항목 속성 업데이트
saddle['price'] = 32.55
saddle['name'] = 'Road LL Saddle'

# 항목 업데이트(Upsert)
container.upsert_item(body=saddle)
print(f"Upserted item with id: {saddle['id']}")

# 업데이트된 항목 다시 읽어서 확인
updated_saddle = container.read_item(item=item_id, partition_key=partition_key_value)
print(f"Updated name: {updated_saddle['name']}, Updated price: ${updated_saddle['price']:.2f}")

# 항목 삭제
container.delete_item(item=item_id, partition_key=partition_key_value)
print(f"Deleted item with id: {item_id}")
```
> **Note: Python SDK CRUD 메서드 (계속)**
>
> - `upsert_item(body=...)`: 항목을 업데이트하거나 생성합니다.
> - `delete_item(item=..., partition_key=...)`: `id`와 `partitionKey`를 사용하여 단일 항목을 삭제합니다.

---

*실습의 나머지 단계를 따라 코드를 실행하고 Data Explorer에서 항목이 비어 있는지 확인한 후, 브라우저와 Visual Studio Code를 닫습니다.*

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.createitemasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.createitemasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.deleteitemasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.deleteitemasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.readitemasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.readitemasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.upsertitemasync]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.upsertitemasync
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.partitionkey]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.partitionkey
[docs.microsoft.com/dotnet/core/tools/dotnet-build]: https://docs.microsoft.com/dotnet/core/tools/dotnet-build
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/microsoft.azure.cosmos/3.22.1]: https://www.nuget.org/packages/Microsoft.Azure.Cosmos/3.22.1
