# Azure Data Factory를 사용하여 기존 데이터 마이그레이션

Azure Data Factory에서 Azure Cosmos DB는 데이터 수집을 위한 원본(source) 및 데이터 출력을 위한 대상(sink)으로 지원됩니다.

이 실습에서는 유용한 명령줄 유틸리티를 사용하여 Azure Cosmos DB를 채운 다음, Azure Data Factory를 사용하여 한 컨테이너에서 다른 컨테이너로 데이터의 일부를 이동합니다.

> **배경지식: Azure Data Factory (ADF)와 ETL**
>
> **Azure Data Factory**는 클라우드 기반 데이터 통합 서비스로, 다양한 데이터 저장소에서 데이터를 이동하고 변환하는 데이터 기반 워크플로우를 생성하고 스케줄링할 수 있게 해줍니다.
>
> **ETL**은 **Extract(추출), Transform(변환), Load(적재)**의 약자로, 데이터 웨어하우징 및 데이터 마이그레이션의 핵심 프로세스입니다.
> 1.  **Extract**: 원본 시스템(예: 관계형 데이터베이스, 파일 등)에서 데이터를 추출합니다.
> 2.  **Transform**: 추출된 데이터를 비즈니스 요구사항에 맞게 정제, 결합, 재구성하는 등의 변환 작업을 수행합니다.
> 3.  **Load**: 변환된 데이터를 대상 시스템(예: Azure Cosmos DB, Azure Synapse Analytics 등)에 적재합니다.
>
> 이 실습에서는 ADF를 사용하여 Cosmos DB의 `products` 컨테이너에서 데이터를 **추출(Extract)**하고, 필요한 필드만 선택하고 이름을 변경하여 구조를 **변환(Transform)**한 후, `flatproducts` 컨테이너에 **적재(Load)**하는 ETL 파이프라인을 만듭니다.

## Azure Cosmos DB for NoSQL 계정 생성 및 데이터 시딩

명령줄 유틸리티를 사용하여 초당 **4,000 요청 단위(RU/s)**로 **cosmicworks** 데이터베이스와 **products** 컨테이너를 생성합니다. 생성 후에는 처리량을 400 RU/s로 조정합니다.

`products` 컨테이너와 함께, 이 실습의 마지막에 수행할 ETL 변환 및 로드 작업의 대상이 될 **flatproducts** 컨테이너를 수동으로 생성합니다.

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
    1.  **PRIMARY CONNECTION STRING** 필드. 이 **connection string** 값은 나중에 이 연습에서 사용됩니다.

7.  나중에 다시 사용할 것이므로 브라우저 탭을 열어 둡니다.

8.  **Visual Studio Code**를 시작합니다.

    > 📝 Visual Studio Code 인터페이스에 익숙하지 않은 경우, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

9.  **Visual Studio Code**에서 **Terminal** 메뉴를 열고 **New Terminal**을 선택하여 새 터미널 인스턴스를 엽니다.

10. [cosmicworks][nuget.org/packages/cosmicworks] 명령줄 도구를 컴퓨터에 전역으로 설치합니다.

    ```powershell
    dotnet tool install --global CosmicWorks --version 2.3.1
    ```

    > 💡 이 명령은 완료하는 데 몇 분 정도 걸릴 수 있습니다. 이전에 이 도구의 최신 버전을 이미 설치한 경우 'Tool 'cosmicworks' is already installed'라는 경고 메시지가 출력됩니다.
    >
    > **Note: `cosmicworks` 도구**
    >
    > `cosmicworks`는 샘플 데이터를 Azure Cosmos DB 계정에 빠르고 쉽게 채우기 위해 만들어진 .NET 기반의 명령줄 유틸리티입니다. 이 도구를 사용하면 실습에 필요한 데이터베이스, 컨테이너, 항목들을 수동으로 만들 필요 없이 자동으로 생성할 수 있습니다.

11. 다음 명령줄 옵션으로 `cosmicworks`를 실행하여 Azure Cosmos DB 계정에 데이터를 시딩합니다.

| **Option** | **Value** |
| :--- | :--- |
| **-c** | *이 실습의 앞부분에서 확인한 connection string 값* |
| **--number-of-employees** | *`cosmicworks` 명령은 별도로 지정하지 않는 한, 각각 1000개와 200개의 항목을 가진 employees와 products 컨테이너로 데이터베이스를 채웁니다.* |

    ```powershell
    cosmicworks -c "connection-string" --number-of-employees 0 --disable-hierarchical-partition-keys
    ```

    > 📝 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/**이고 키가 **fDR2ci9QgkdkvERTQ==**라면, 명령은 다음과 같습니다:
    > ``cosmicworks -c "AccountEndpoint=https://dp420.documents.azure.com:443/;AccountKey=fDR2ci9QgkdkvERTQ==" --number-of-employees 0 --disable-hierarchical-partition-keys``

12. **cosmicworks** 명령이 계정에 데이터베이스, 컨테이너, 항목을 채울 때까지 기다립니다.

13. 통합 터미널을 닫습니다.

14. 웹 브라우저로 다시 전환하고, 새 탭을 열어 Azure portal (``portal.azure.com``)로 이동합니다.

15. **Resource groups**를 선택하고, 이 실습의 앞부분에서 생성하거나 확인한 리소스 그룹을 선택한 다음, 이 실습에서 생성한 **Azure Cosmos DB account** 리소스를 선택합니다.

16. **Azure Cosmos DB** 계정 리소스 내에서 **Data Explorer** 창으로 이동합니다.

17. **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 확장한 다음, **Items**를 선택합니다.

18. **products** 컨테이너에 있는 다양한 JSON 항목을 관찰하고 선택합니다. 이것들은 이전 단계에서 사용한 명령줄 도구로 생성된 항목들입니다.

19. **Scale** 노드를 선택합니다. **Scale** 탭에서 **Manual**을 선택하고, **required throughput** 설정을 **4000 RU/s**에서 **400 RU/s**로 업데이트한 다음, 변경 사항을 **Save**합니다.

20. **Data Explorer** 창에서 **New Container**를 선택합니다.

21. **New Container** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | *Use existing* | *cosmicworks* |
| **Container id** | `flatproducts` |
| **Partition key** | `/category` |

22. **Data Explorer** 창으로 돌아와 **cosmicworks** 데이터베이스 노드를 확장하고 계층 구조 내에서 **flatproducts** 컨테이너 노드를 확인합니다.

23. Azure portal의 **Home**으로 돌아갑니다.

## Azure Data Factory 리소스 생성

Azure Cosmos DB for NoSQL 리소스가 준비되었으므로, 이제 Azure Data Factory 리소스를 생성하고 한 NoSQL API 컨테이너에서 다른 컨테이너로 데이터를 한 번 이동하는 데 필요한 모든 구성 요소와 연결을 구성하여 데이터를 추출, 변환 및 로드합니다.

1.  **+ Create a resource**를 선택하고, *Data Factory*를 검색한 후, 다음 설정으로 새로운 **Data Factory** 리소스를 생성합니다. 나머지 설정은 모두 기본값으로 둡니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Subscription** | *기존 Azure 구독* |
| **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
| **Name** | *전역적으로 고유한 이름 입력* |
| **Region** | *사용 가능한 지역 선택* |
| **Version** | *V2* |

    > 📝 실습 환경에서는 새 리소스 그룹 생성을 막는 제약이 있을 수 있습니다. 그럴 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

2.  이 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

3.  새로 생성된 **Data Factory** 리소스로 이동하여 **Launch studio**를 선택합니다.

    > 💡 또는 (``adf.azure.com/home``)으로 이동하여 새로 생성한 Data Factory 리소스를 선택한 다음 홈 아이콘을 선택할 수도 있습니다.

4.  홈 화면에서 **Ingest** 옵션을 선택하여 대규모 데이터 복사 작업을 한 번 수행하는 빠른 마법사를 시작하고 마법사의 **Properties** 단계로 이동합니다.

5.  마법사의 **Properties** 단계부터 시작하여, **Task type** 섹션에서 **Built-in copy task**를 선택합니다.

6.  **Task cadence or task schedule** 섹션에서 **Run once now**를 선택한 다음 **Next**를 선택하여 마법사의 **Source** 단계로 이동합니다.

7.  마법사의 **Source** 단계에서, **Source type** 목록에서 **Azure Cosmos DB for NoSQL**을 선택합니다.

8.  **Connection** 섹션에서 **+ New connection**을 선택합니다.

    > **Note: ADF 연결(Linked Service)**
    >
    > Azure Data Factory에서 `Connection`(UI 용어) 또는 `Linked Service`(기본 개념)는 외부 데이터 저장소나 컴퓨팅 서비스에 대한 연결 정보를 정의합니다. 여기서는 Cosmos DB 계정에 연결하기 위한 정보를 설정합니다.

9.  **New connection (Azure Cosmos DB for NoSQL)** 팝업에서 다음 값으로 새 연결을 구성한 다음 **Create**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Name** | `CosmosSqlConn` |
| **Connect via integration runtime** | *AutoResolveIntegrationRuntime* |
| **Authentication method** | *Account key* | *Connection string* |
| **Account selection method** | *From Azure subscription* |
| **Azure subscription** | *기존 Azure 구독* |
| **Azure Cosmos DB account name** | *이 실습의 앞부분에서 선택한 기존 Azure Cosmos DB 계정 이름* |
| **Database name** | *cosmicworks* |

10. **Source data store** 섹션으로 돌아와, **Source tables** 섹션 내에서 **Use query**를 선택합니다.

11. **Table name** 목록에서 **products**를 선택합니다.

12. **Query** 편집기에서 기존 내용을 삭제하고 다음 쿼리를 입력합니다. 이 쿼리는 데이터를 평탄화(flatten)하는 변환 역할을 합니다.

    ```sql
    SELECT 
        p.name, 
        p.categoryName as category, 
        p.price 
    FROM 
        products p
    ```
    > **Note: 데이터 변환**
    >
    > 원본 `products` 컨테이너의 항목들은 `categoryName`과 같은 필드 이름을 가집니다. 이 쿼리는 `p.categoryName as category` 구문을 사용하여 필드 이름을 `category`로 변경합니다. 또한, `name`, `categoryName`, `price` 필드만 선택하여 필요한 데이터만 추출합니다. 이것이 이 ETL 프로세스의 'T(Transform)'에 해당합니다.

13. **Preview data**를 선택하여 쿼리의 유효성을 테스트합니다. **Next**를 선택하여 마법사의 **Destination** 단계로 이동합니다.

14. 마법사의 **Destination** 단계에서, **Destination type** 목록에서 **Azure Cosmos DB for NoSQL**을 선택합니다.

15. **Connection** 목록에서 **CosmosSqlConn**을 선택합니다.

16. **Target** 목록에서 **flatproducts**를 선택한 다음 **Next**를 선택하여 마법사의 **Settings** 단계로 이동합니다.

17. 마법사의 **Settings** 단계에서, **Task name** 필드에 `FlattenAndMoveData`를 입력합니다.

18. 나머지 필드는 모두 기본 빈 값으로 두고 **Next**를 선택하여 마법사의 마지막 단계로 이동합니다.

19. 마법사에서 선택한 단계의 **Summary**를 검토한 다음 **Next**를 선택합니다.

20. 배포의 여러 단계를 관찰합니다. 배포가 완료되면 **Finish**를 선택합니다.

21. **Azure Cosmos DB account**가 있는 브라우저 탭으로 돌아가 **Data Explorer** 창으로 이동합니다.

22. **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **flatproducts** 컨테이너 노드를 선택한 다음, **New SQL Query**를 선택합니다.

23. 편집기 영역의 내용을 삭제합니다.

24. **name**이 **HL Headset**과 동일한 모든 문서를 반환하는 새 SQL 쿼리를 작성합니다.

    ```sql
    SELECT 
        p.name, 
        p.category, 
        p.price 
    FROM
        flatproducts p
    WHERE
        p.name = 'HL Headset'
    ```

25. **Execute Query**를 선택합니다.

26. 쿼리 결과를 관찰합니다. `categoryName`이 아닌 `category` 필드를 가진, 평탄화된(flattened) 구조의 데이터가 성공적으로 마이그레이션된 것을 확인할 수 있습니다.

27. 웹 브라우저 창 또는 탭을 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[nuget.org/packages/cosmicworks]: https://www.nuget.org/packages/cosmicworks

---

