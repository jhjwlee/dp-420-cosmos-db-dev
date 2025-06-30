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
