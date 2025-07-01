# Azure Cosmos DB for NoSQL SDK를 사용하여 여러 점 작업을 일괄 처리

[TransactionalBatch][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatch]와 [TransactionalBatchResponse][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatchresponse] 클래스는 함께 작업을 단일 논리적 단계로 구성하고 분해하는 핵심입니다. 이 클래스들을 사용하여 여러 작업을 수행하는 코드를 작성하고, 서버 측에서 성공적으로 완료되었는지 확인할 수 있습니다.

이 실습에서는 SDK를 사용하여 두 개의 항목을 단일 논리 단위로 생성하려는 두 가지 이중 항목 작업을 수행합니다.

> **배경지식: 트랜잭션 일괄 처리 (Transactional Batch)**
>
> **Transactional Batch**는 여러 점 작업(create, upsert, replace, delete 등)을 하나의 그룹으로 묶어 원자적(atomic)으로 실행하는 기능입니다.
>
> -   **원자성 (Atomicity)**: 일괄 처리 내의 모든 작업은 **전부 성공**하거나 **전부 실패**합니다. 일부만 성공하는 경우는 없습니다. 이를 통해 데이터 일관성을 보장할 수 있습니다. 예를 들어, 한 계좌에서 돈을 인출하고 다른 계좌에 입금하는 두 작업을 하나의 일괄 처리로 묶으면, 둘 중 하나만 실패하더라도 모든 작업이 롤백되어 데이터가 불일치 상태에 빠지는 것을 방지할 수 있습니다.
> -   **성능**: 여러 작업을 단일 네트워크 요청으로 서버에 보내므로, 각 작업을 개별적으로 보낼 때보다 네트워크 오버헤드가 줄어들어 성능이 향상됩니다.
> -   **제약 조건**: 가장 중요한 제약은 일괄 처리 내의 **모든 항목이 동일한 논리적 파티션에 속해야 한다**는 것입니다. 즉, 모든 항목이 동일한 `partitionKey` 값을 가져야 합니다. 이 실습에서 이 제약 조건을 직접 확인하게 됩니다.

## 개발 환경 준비

이 실습을 진행하는 환경에 **DP-420**에 대한 실습 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 여세요.

1.  **Visual Studio Code**를 시작합니다.

    > 📝 Visual Studio Code 인터페이스에 익숙하지 않은 경우, [Getting Started documentation][code.visualstudio.com/docs/getstarted]을 검토하세요.

2.  명령 팔레트를 열고 **Git: Clone**을 실행하여 ``https://github.com/microsoftlearning/dp-420-cosmos-db-dev`` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

    > 💡 **CTRL+SHIFT+P** 키보드 단축키를 사용하여 명령 팔레트를 열 수 있습니다.

3.  리포지토리가 복제되면, 선택한 로컬 폴더를 **Visual Studio Code**에서 엽니다.

## Azure Cosmos DB for NoSQL 계정 생성 및 SDK 프로젝트 구성

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

7.  **Visual Studio Code**로 돌아갑니다.

8.  **Explorer** 창에서 **07-sdk-batch** 폴더로 이동합니다.

9.  **07-sdk-batch** 폴더 내의 **script.cs** 코드 파일을 엽니다.

10. **endpoint**라는 이름의 `string` 변수를 찾습니다. 값을 이전에 생성한 Azure Cosmos DB 계정의 **endpoint**로 설정합니다.

    ```csharp
    string endpoint = "<cosmos-endpoint>";
    ```

11. **key**라는 이름의 `string` 변수를 찾습니다. 값을 이전에 생성한 Azure Cosmos DB 계정의 **key**로 설정합니다.

    ```csharp
    string key = "<cosmos-key>";
    ```

12. **script.cs** 코드 파일을 **저장**합니다.

13. **07-sdk-batch** 폴더의 컨텍스트 메뉴를 열고 **Open in Integrated Terminal**을 선택하여 새 터미널 인스턴스를 엽니다.

14. 다음 명령을 사용하여 NuGet에서 [Microsoft.Azure.Cosmos][nuget.org/packages/microsoft.azure.cosmos/3.22.1] 패키지를 추가합니다.

    ```powershell
    dotnet add package Microsoft.Azure.Cosmos --version 3.49.0
    ```

15. [dotnet build][docs.microsoft.com/dotnet/core/tools/dotnet-build] 명령을 사용하여 프로젝트를 빌드합니다.

    ```powershell
    dotnet build
    ```

16. 통합 터미널을 닫습니다.

## 트랜잭션 일괄 처리 생성하기

먼저, 두 개의 가상 제품을 만드는 간단한 트랜잭션 일괄 처리를 생성해 보겠습니다. 이 일괄 처리는 닳은 안장(worn saddle)과 녹슨 핸들바(rusty handlebar)를 동일한 "used accessories" 카테고리 식별자로 컨테이너에 삽입합니다. 두 항목 모두 동일한 논리 파티션 키를 가지므로 성공적인 일괄 처리가 보장됩니다.

### **C# 코드**

1.  **script.cs** 파일의 편집기 탭으로 돌아갑니다.

2.  고유 식별자 **0120**, 이름 **Worn Saddle**, 카테고리 식별자 **9603ca6c-9e28-4a02-9194-51cdb7fea816**을 가진 **saddle**이라는 **Product** 변수를 생성합니다.

    ```csharp
    Product saddle = new("0120", "Worn Saddle", "9603ca6c-9e28-4a02-9194-51cdb7fea816");
    ```

3.  고유 식별자 **012A**, 이름 **Rusty Handlebar**, 카테고리 식별자 **9603ca6c-9e28-4a02-9194-51cdb7fea816**을 가진 **handlebar**라는 **Product** 변수를 생성합니다.

    ```csharp
    Product handlebar = new("012A", "Rusty Handlebar", "9603ca6c-9e28-4a02-9194-51cdb7fea816");
    ```

4.  생성자 매개변수로 **9603ca6c-9e28-4a02-9194-51cdb7fea816**을 전달하여 **partitionKey**라는 **PartitionKey** 타입의 변수를 생성합니다.

    ```csharp
    PartitionKey partitionKey = new ("9603ca6c-9e28-4a02-9194-51cdb7fea816");
    ```

5.  **container** 변수의 [CreateTransactionalBatch][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.createtransactionalbatch] 메서드를 호출하고 **partitionkey** 변수를 전달합니다. fluent 구문을 사용하여 [CreateItem<>][docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatch.createitem] 제네릭 메서드를 호출하여 **saddle**과 **handlebar** 변수를 개별 작업으로 생성할 항목으로 전달하고, 결과를 **batch**라는 **TransactionalBatch** 타입의 변수에 저장합니다.

    ```csharp
    TransactionalBatch batch = container.CreateTransactionalBatch(partitionKey)
        .CreateItem<Product>(saddle)
        .CreateItem<Product>(handlebar);
    ```

6.  `using` 문 내에서 **batch** 변수의 **ExecuteAsync** 메서드를 비동기적으로 호출하고 결과를 **response**라는 **TransactionalBatchResponse** 타입의 변수에 저장합니다.

    ```csharp
    using TransactionalBatchResponse response = await batch.ExecuteAsync();
    ```

7.  정적 **Console.WriteLine** 메서드를 호출하여 **response** 변수의 **StatusCode** 속성 값을 출력합니다.

    ```csharp
    Console.WriteLine($"Status:\t{response.StatusCode}");
    ```

---

### **Python 코드 (추가)**

Python에서는 C#의 fluent 구문 대신, 작업 목록을 만들어 `execute_transactional_batch` 메서드에 전달합니다.

1.  **07-sdk-batch** 폴더에 **script.py** 파일을 만들고 기본 연결 설정을 추가합니다. (이전 실습 참조)

2.  다음 코드를 **script.py**에 추가합니다.

    ```python
    from azure.cosmos.exceptions import CosmosHttpResponseError
    
    # 두 항목 모두 동일한 categoryId를 가짐
    saddle = {
        'id': '0120',
        'name': 'Worn Saddle',
        'categoryId': '9603ca6c-9e28-4a02-9194-51cdb7fea816'
    }
    handlebar = {
        'id': '012A',
        'name': 'Rusty Handlebar',
        'categoryId': '9603ca6c-9e28-4a02-9194-51cdb7fea816'
    }
    
    # 일괄 처리할 작업 목록 생성
    operations = [
        ('create', saddle),
        ('create', handlebar)
    ]
    
    partition_key = '9603ca6c-9e28-4a02-9194-51cdb7fea816'
    
    try:
        # 트랜잭션 일괄 처리 실행
        result = container.execute_transactional_batch(operations=operations, partition_key=partition_key)
        # Python SDK는 성공 시 예외를 발생시키지 않고 결과를 반환합니다.
        # HTTP 상태 코드를 직접 확인하려면 response 헤더를 봐야하지만,
        # 보통은 성공 여부를 예외 발생 유무로 판단합니다.
        print(f"Status:\tSuccess (Batch operation completed)")
        # print(f"Result: {result}")
    
    except CosmosHttpResponseError as e:
        print(f"Status:\t{e.status_code}")
        print(f"Message:\t{e.message}")
    
    ```
    > **Note: Python의 트랜잭션 일괄 처리**
    >
    > Python SDK는 C#의 `TransactionalBatch` 빌더 클래스와 다른 접근 방식을 사용합니다.
    > - **작업 목록**: `(작업_유형, 항목_본문)` 형태의 튜플 리스트를 만듭니다. 작업 유형은 `'create'`, `'upsert'`, `'replace'`, `'delete'` 등이 될 수 있습니다.
    > - **`execute_transactional_batch`**: 이 메서드는 작업 목록과 `partition_key`를 인수로 받아 일괄 처리를 실행합니다. 성공하면 결과 리스트를 반환하고, 실패하면 `CosmosHttpResponseError` 예외를 발생시킵니다.

---

8.  **script.cs** (또는 **script.py**) 파일을 **저장**하고 실행합니다.
9.  터미널에서 출력을 관찰합니다. 상태 코드는 **OK**(C#) 또는 성공 메시지(Python)여야 합니다.

## 오류가 발생하는 트랜잭션 일괄 처리 생성하기

이제 의도적으로 오류를 발생시키는 트랜잭션 일괄 처리를 만들어 보겠습니다. 이 일괄 처리는 서로 다른 논리 파티션 키를 가진 두 항목을 삽입하려고 시도합니다. "used accessories" 카테고리의 깜박이는 스트로브 라이트와 "pristine accessories" 카테고리의 새 헬멧을 생성할 것입니다. 정의에 따라 이것은 잘못된 요청이어야 하며, 이 트랜잭션을 수행할 때 오류를 반환해야 합니다.

### **C# 코드**

1.  **script.cs** 파일에서 이전 일괄 처리 코드를 삭제합니다.

2.  고유 식별자 **012B**, 이름 **Flickering Strobe Light**, 카테고리 식별자 **9603ca6c-9e28-4a02-9194-51cdb7fea816**을 가진 **light**라는 **Product** 변수를 생성합니다.

    ```csharp
    Product light = new("012B", "Flickering Strobe Light", "9603ca6c-9e28-4a02-9194-51cdb7fea816");
    ```

3.  고유 식별자 **012C**, 이름 **New Helmet**, 카테고리 식별자 **0feee2e4-687a-4d69-b64e-be36afc33e74**을 가진 **helmet**이라는 **Product** 변수를 생성합니다. *<- 파티션 키 값이 다름!*

    ```csharp
    Product helmet = new("012C", "New Helmet", "0feee2e4-687a-4d69-b64e-be36afc33e74");
    ```

4.  이전과 동일하게 `partitionKey`를 정의하고 `TransactionalBatch`를 구성한 후 `ExecuteAsync`를 호출합니다.

    ```csharp
    PartitionKey partitionKey = new ("9603ca6c-9e28-4a02-9194-51cdb7fea816");
    
    TransactionalBatch batch = container.CreateTransactionalBatch(partitionKey)
        .CreateItem<Product>(light)
        .CreateItem<Product>(helmet);
    
    using TransactionalBatchResponse response = await batch.ExecuteAsync();
    
    Console.WriteLine($"Status:\t{response.StatusCode}");
    ```

### **Python 코드 (추가)**

**script.py**에서 이전 코드를 수정하여 두 번째 항목의 `categoryId`를 다르게 설정합니다.

```python
# light와 helmet은 서로 다른 categoryId를 가짐
light = {
    'id': '012B',
    'name': 'Flickering Strobe Light',
    'categoryId': '9603ca6c-9e28-4a02-9194-51cdb7fea816'
}
helmet = {
    'id': '012C',
    'name': 'New Helmet',
    'categoryId': '0feee2e4-687a-4d69-b64e-be36afc33e74' # <-- 다른 파티션 키
}

operations = [
    ('create', light),
    ('create', helmet)
]

# CreateTransactionalBatch에 제공된 파티션 키와 helmet의 파티션 키가 일치하지 않음
partition_key = '9603ca6c-9e28-4a02-9194-51cdb7fea816'

try:
    result = container.execute_transactional_batch(operations=operations, partition_key=partition_key)
    print("Status:\tSuccess (This should not happen)")

except CosmosHttpResponseError as e:
    # 예상된 오류!
    print(f"Status:\t{e.status_code}")
    print(f"Message:\t{e.message}")
```

---

5.  **script.cs** (또는 **script.py**) 파일을 **저장**하고 실행합니다.
6.  터미널에서 출력을 관찰합니다. 상태 코드는 **Bad Request** 또는 **Conflict**여야 합니다. 이는 트랜잭션 내의 모든 항목이 트랜잭션 일괄 처리와 동일한 파티션 키 값을 공유하지 않았기 때문에 발생했습니다.
7.  통합 터미널과 **Visual Studio Code**를 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.createtransactionalbatch]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.container.createtransactionalbatch
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatch]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatch
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatch.createitem]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatch.createitem
[docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatchresponse]: https://docs.microsoft.com/dotnet/api/microsoft.azure.cosmos.transactionalbatchresponse
[docs.microsoft.com/dotnet/core/tools/dotnet-build]: https://docs.microsoft.com/dotnet/core/tools/dotnet-build
[docs.microsoft.com/dotnet/core/tools/dotnet-run]: https://docs.microsoft.com/dotnet/core/tools/dotnet-run
[nuget.org/packages/microsoft.azure.cosmos/3.22.1]: https://www.nuget.org/packages/Microsoft.Azure.Cosmos/3.22.1
