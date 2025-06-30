---
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
# Azure Cosmos DB for NoSQL 계정 생성

Azure Cosmos DB를 깊이 있게 배우기 전에, 가장 많이 사용하게 될 리소스를 생성하는 기본 사항을 파악하는 것이 중요합니다. 대부분의 시나리오에서 계정, 데이터베이스, 컨테이너 및 항목을 능숙하게 생성할 수 있어야 합니다. 실제 시나리오에서는 모든 리소스가 올바르게 생성되었는지 테스트하기 위해 몇 가지 기본 쿼리를 "준비"해 두는 것이 좋습니다.

이 실습에서는 NoSQL용 API를 사용하여 새로운 Azure Cosmos DB 계정을 생성합니다. 그런 다음 Data Explorer를 사용하여 데이터베이스, 컨테이너 및 두 개의 항목을 만듭니다. 마지막으로, 생성한 항목에 대해 데이터베이스를 쿼리합니다.

> **배경지식: Azure Cosmos DB 리소스 계층 구조**
>
> Azure Cosmos DB는 다음과 같은 계층적 리소스 모델을 가집니다. 이 구조를 이해하는 것은 리소스를 관리하고 구성하는 데 중요합니다.
>
> 1.  **Account (계정)**: Azure Cosmos DB의 최상위 리소스입니다. 전역적으로 고유한 이름을 가지며, 데이터베이스의 관리 단위입니다. 여기에서 지역, 용량 모드, 일관성 수준 등을 구성합니다.
> 2.  **Database (데이터베이스)**: 컨테이너의 논리적 그룹입니다. 하나의 계정 아래에 여러 데이터베이스를 만들 수 있습니다.
> 3.  **Container (컨테이너)**: 항목(items)의 스키마에 구애받지 않는 집합입니다. 관계형 데이터베이스의 '테이블'과 유사한 개념입니다. 처리량(throughput)과 파티셔닝(partitioning)이 컨테이너 수준에서 구성됩니다.
> 4.  **Item (항목)**: 컨테이너에 저장되는 개별 문서입니다. 일반적으로 JSON 형식으로 표현됩니다. 관계형 데이터베이스의 '행'과 유사합니다.

## 새로운 Azure Cosmos DB 계정 생성

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다.

1.  새 웹 브라우저 창 또는 탭에서 Azure portal (``portal.azure.com``)로 이동합니다.

2.  구독과 연결된 Microsoft 자격 증명을 사용하여 포털에 로그인합니다.

3.  **Azure services** 카테고리에서 **Create a resource**를 선택한 다음, **Azure Cosmos DB**를 선택합니다.

    > 💡 또는 **☰** 메뉴를 확장하고, **All Services**를 선택한 후 **Databases** 카테고리에서 **Azure Cosmos DB**를 선택하고 **Create**를 클릭할 수도 있습니다.

4.  **Select API option** 창의 **Azure Cosmos DB for NoSQL** 섹션에서 **Create** 옵션을 선택합니다.

5.  **Create Azure Cosmos DB Account** 창에서 **Basics** 탭을 확인합니다.

6.  **Basics** 탭에서 각 설정에 다음 값을 입력합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Workload Type** | **Learning** |
| **Subscription** | **기존 Azure 구독을 사용합니다.** *모든 리소스는 리소스 그룹에 속해야 하며, 모든 리소스 그룹은 구독에 속해야 합니다.* |
| **Resource Group** | **기존 리소스 그룹을 사용하거나 새로 만듭니다.** *모든 리소스는 리소스 그룹에 속해야 합니다.* |
| **Account Name** | **전역적으로 고유한 이름을 입력합니다.** *이 이름은 요청에 대한 DNS 주소의 일부로 사용됩니다. 포털에서 이름의 고유성을 실시간으로 확인합니다.* |
| **Location** | **사용 가능한 지역을 선택합니다.** *데이터베이스가 초기에 호스팅될 지리적 지역을 선택합니다.* |
| **Capacity mode** | **Provisioned throughput** |
| **Apply Free Tier Discount** | **Do Not Apply** |

7.  **Review + Create**를 선택하여 **Review + Create** 탭으로 이동한 다음, **Create**를 선택합니다.

    > 📝 Azure Cosmos DB for NoSQL 계정을 사용하는 데 10-15분 정도 걸릴 수 있습니다.

8.  **Deployment** 창을 관찰합니다. 배포가 완료되면 창에 **Deployment successful** 메시지가 표시됩니다.

9.  **Deployment** 창에서 **Go to resource**를 선택합니다.

## Data Explorer를 사용하여 새 데이터베이스 및 컨테이너 생성

**Data Explorer**는 Azure portal에서 Azure Cosmos DB for NoSQL 데이터베이스와 컨테이너를 관리하는 주요 도구입니다. 이 실습에서 사용할 기본 데이터베이스와 컨테이너를 생성합니다.

1.  **Azure Cosmos DB account** 창의 리소스 메뉴에서 **Data Explorer**를 선택합니다.

2.  **Data Explorer** 창에서 **New Container**를 선택합니다.

3.  **New Container** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | `cosmicworks` |
| **Share throughput across containers** | *Unchecked* |
| **Container id** | `products` |
| **Partition key** | `/categoryId` |
| **Container throughput (autoscale)** | *Manual* |
| **RU/s** | `400` |

> **Note: Partition Key와 RU/s**
>
> - **Partition Key**: 컨테이너 내에서 데이터를 논리적 파티션으로 분할하는 데 사용되는 항목의 속성입니다. 파티션 키를 잘 선택하는 것은 확장성과 성능에 매우 중요합니다. Azure Cosmos DB는 파티션 키 값을 사용하여 데이터를 여러 물리적 파티션에 분산시켜 워크로드를 균등하게 분배합니다.
> - **RU/s (Request Units per second)**: 처리량을 나타내는 단위입니다. `400 RU/s`는 컨테이너에 초당 400 요청 단위를 처리할 수 있는 성능을 할당한다는 의미입니다. 이는 Cosmos DB의 최소 프로비저닝 처리량입니다.

4.  **Data Explorer** 창으로 돌아와 **cosmicworks** 데이터베이스 노드를 확장하고 계층 구조 내에서 **products** 컨테이너 노드를 확인합니다.

## Data Explorer를 사용하여 새 항목 생성

**Data Explorer**는 Azure Cosmos DB for NoSQL 컨테이너에서 항목을 쿼리, 생성 및 관리하는 다양한 기능도 포함하고 있습니다. Data Explorer에서 원시 JSON을 사용하여 두 개의 기본 항목을 생성합니다.

1.  **Data Explorer** 창에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 확장한 다음, **Items**를 선택합니다.

2.  명령 모음에서 **New Item**을 선택하고, 편집기에서 자리 표시자 JSON 항목을 다음 내용으로 바꿉니다.

    ```json
    {
      "id": "2E8B4F39-4D46-484B-A23B-23D923B41A3B",
      "categoryId": "4F34E180-384D-42FC-AC10-FEC30227577F",
      "categoryName": "Components, Pedals",
      "sku": "PD-R563",
      "name": "ML Road Pedal",
      "price": 62.09
    }
    ```
    > **Note: 항목(Item)의 구조**
    >
    > 모든 항목은 고유한 `id` 속성을 가져야 합니다. `id`를 제공하지 않으면 Cosmos DB가 자동으로 GUID를 생성하여 할당합니다. `partitionKey`로 지정된 속성(이 경우 `categoryId`)도 반드시 포함되어야 합니다.

3.  명령 모음에서 **Save**를 선택하여 첫 번째 JSON 항목을 추가합니다.

4.  **Items** 탭으로 돌아와 명령 모음에서 **New Item**을 선택합니다. 편집기에서 자리 표시자 JSON 항목을 다음 내용으로 바꿉니다.

    ```json
    {
      "id": "A1D8A34A-4C26-4444-9B13-50478E441E7E",
      "categoryId": "75BF1ACB-168D-469C-9AA3-1FD26BB4EA4C",
      "categoryName": "Bikes, Touring Bikes",
      "sku": "BK-T18Y-44",
      "name": "Touring-3000 Yellow, 44",
      "price": 742.35
    }
    ```

5.  명령 모음에서 **Save**를 선택하여 두 번째 JSON 항목을 추가합니다.

6.  **Items** 탭에서 **Items** 창에 있는 두 개의 새 항목을 관찰합니다.

## Data Explorer를 사용하여 기본 쿼리 실행

마지막으로, **Data Explorer**에는 쿼리를 실행하고, 결과를 관찰하며, 초당 요청 단위(RU/s) 측면에서 영향을 측정하는 데 사용되는 내장 쿼리 편집기가 있습니다.

1.  **Data Explorer** 창에서 **New SQL Query**를 선택합니다.

2.  쿼리 탭에서 **Execute Query**를 선택하여 필터 없이 모든 항목을 선택하는 표준 쿼리를 확인합니다. 기본 쿼리는 `SELECT * FROM c` 입니다.

3.  편집기 영역의 내용을 삭제합니다.

4.  자리 표시자 쿼리를 다음 내용으로 바꿉니다.

    ```sql
    SELECT * FROM products p WHERE p.price > 500
    ```

    > 📝 이 쿼리는 **price**가 $500보다 큰 모든 항목을 선택합니다.
    >
    > **Note: NoSQL용 SQL API**
    >
    > Azure Cosmos DB for NoSQL은 익숙한 SQL 구문을 사용하여 JSON 문서를 쿼리할 수 있는 기능을 제공합니다. 이를 통해 개발자들은 복잡한 NoSQL 쿼리 언어를 새로 배울 필요 없이 데이터를 쉽게 조회할 수 있습니다.

5.  **Execute Query**를 선택합니다.

6.  쿼리 결과를 관찰합니다. 단일 JSON 항목과 그 모든 속성을 포함해야 합니다.

7.  **Query** 탭에서 **Query Stats**를 선택합니다.

8.  **Query** 탭에서 **Query Statistics** 섹션 내의 **Request Charge** 필드 값을 관찰합니다.

    > 📝 일반적으로 컨테이너 크기가 작을 때 이 간단한 쿼리의 요청 비용은 2~3 RU/s 사이입니다. 이 값은 쿼리를 실행하는 데 소모된 RU의 양을 나타냅니다.

9.  웹 브라우저 창 또는 탭을 닫습니다.

---
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
