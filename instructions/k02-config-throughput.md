# Azure portal을 사용하여 Azure Cosmos DB for NoSQL의 처리량 구성

Azure Cosmos DB for NoSQL을 시작할 때 가장 먼저 이해해야 할 중요한 것 중 하나는 처리량(throughput)을 구성하는 것입니다. Azure Cosmos DB for NoSQL 컨테이너를 생성하려면 먼저 계정(account), 그 다음 데이터베이스(database)를 순서대로 만들어야 합니다.

이 실습에서는 **Data Explorer**를 사용하여 다양한 방법으로 처리량을 프로비저닝합니다. 데이터베이스 및 컨테이너 수준에서 수동(manually) 또는 자동 확장(autoscale)을 사용하여 처리량을 프로비저닝하게 됩니다.

> **배경지식: 처리량(Throughput)과 RU(Request Unit)**
>
> Azure Cosmos DB에서 처리량은 초당 요청 단위(Request Units per second, RU/s)로 측정됩니다. RU는 데이터베이스 작업(읽기, 쓰기, 쿼리 등)을 수행하는 데 필요한 시스템 리소스(CPU, IOPS, 메모리)를 추상화한 단위입니다. 모든 작업에는 RU 비용이 발생하며, 이는 작업의 복잡성에 따라 달라집니다.
>
> 처리량을 프로비저닝하는 것은 애플리케이션에 필요한 성능을 보장하고 비용을 예측하는 핵심적인 과정입니다. 처리량을 너무 낮게 설정하면 요청이 제한(throttled)되어 오류가 발생할 수 있고, 너무 높게 설정하면 불필요한 비용이 발생할 수 있습니다.

## Serverless 계정 생성

먼저 `Serverless` 계정을 만드는 간단한 작업부터 시작하겠습니다. `Serverless` 모드에서는 모든 것이 서버리스로 관리되므로 구성할 것이 많지 않습니다. 데이터베이스와 컨테이너를 만들 때 처리량을 전혀 프로비저닝할 필요가 없습니다. 계정을 생성하는 과정을 통해 이를 확인하게 될 것입니다.

> **Note: Serverless 용량 모드**
>
> `Serverless` 모드는 예측할 수 없거나 간헐적인 트래픽이 있는 워크로드에 적합합니다. 사용한 작업에 대한 RU만큼만 비용을 지불하므로, 처리량을 미리 프로비저닝할 필요가 없습니다. 개발, 테스트 또는 사용량이 적은 애플리케이션에 이상적입니다.

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
| **Capacity mode** | **Serverless 선택** |

7.  **Review + Create**를 선택하여 **Review + Create** 탭으로 이동한 다음, **Create**를 선택합니다.

    > 📝 Azure Cosmos DB for NoSQL 계정을 사용하는 데 10-15분 정도 걸릴 수 있습니다.

8.  **Deployment** 창을 관찰합니다. 배포가 완료되면 창에 **Deployment successful** 메시지가 표시됩니다.

9.  **Deployment** 창에서 **Go to resource**를 선택합니다.

10. **Azure Cosmos DB account** 창의 리소스 메뉴에서 **Data Explorer**를 선택합니다.

11. **Data Explorer** 창에서 **New Container**를 확장한 다음 **New Database**를 선택합니다.

12. **New Database** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | `cosmicworks` |

13. **Data Explorer** 창으로 돌아와 계층 구조 내에서 **cosmicworks** 데이터베이스 노드를 확인합니다.

14. **Data Explorer** 창에서 **New Container**를 선택합니다.

15. **New Container** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | *Use existing* | *cosmicworks* |
| **Container id** | `products` |
| **Partition key** | `/category/name` |

16. **Data Explorer** 창으로 돌아와 **cosmicworks** 데이터베이스 노드를 확장하고 계층 구조 내에서 **products** 컨테이너 노드를 확인합니다.

17. Azure portal의 **Home**으로 돌아갑니다.

## Provisioned throughput 계정 생성

이제 더 전통적인 구성 옵션을 가진 `Provisioned throughput` 계정을 만들어 보겠습니다. 이 유형의 계정은 다양한 구성 옵션을 제공하며, 때로는 압도적으로 느껴질 수도 있습니다. 여기서는 가능한 데이터베이스와 컨테이너 조합의 몇 가지 예를 살펴보겠습니다.

> **Note: Provisioned Throughput 용량 모드**
>
> `Provisioned throughput` 모드는 예측 가능한 성능이 필요한 프로덕션 워크로드에 적합합니다. 초당 RU/s로 처리량을 지정하며, 이 처리량은 애플리케이션에 대해 예약됩니다. 수동(Manual) 또는 자동 확장(Autoscale) 두 가지 방식으로 프로비저닝할 수 있습니다.
> - **Manual**: 고정된 RU/s를 할당합니다. 안정적이고 예측 가능한 워크로드에 적합합니다.
> - **Autoscale**: 최소 및 최대 RU/s 범위를 설정하면, Azure Cosmos DB가 트래픽에 따라 이 범위 내에서 처리량을 자동으로 조정합니다. 변동이 심한 워크로드에 비용 효율적입니다.

1.  **Azure services** 카테고리에서 **Create a resource**를 선택한 다음, **Azure Cosmos DB**를 선택합니다.

    > 💡 또는 **☰** 메뉴를 확장하고, **All Services**를 선택한 후 **Databases** 카테고리에서 **Azure Cosmos DB**를 선택하고 **Create**를 클릭할 수도 있습니다.

2.  **Select API option** 창의 **Azure Cosmos DB for NoSQL** 섹션에서 **Create** 옵션을 선택합니다.

3.  **Create Azure Cosmos DB Account** 창에서 **Basics** 탭을 확인합니다.

4.  **Basics** 탭에서 각 설정에 다음 값을 입력합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Workload Type** | **Learning** |
| **Subscription** | **기존 Azure 구독을 사용합니다.** |
| **Resource Group** | **기존 리소스 그룹을 사용하거나 새로 만듭니다.** |
| **Account Name** | **전역적으로 고유한 이름을 입력합니다.** |
| **Location** | **사용 가능한 지역을 선택합니다.** |
| **Capacity mode** | **Provisioned throughput** |
| **Apply Free Tier Discount** | **Do Not Apply** |
| **Limit the total amount of throughput that can be provisioned on this account** | **Unchecked** |

5.  **Review + Create**를 선택하여 **Review + Create** 탭으로 이동한 다음, **Create**를 선택합니다.

    > 📝 Azure Cosmos DB for NoSQL 계정을 사용하는 데 10-15분 정도 걸릴 수 있습니다.

6.  **Deployment** 창을 관찰합니다. 배포가 완료되면 창에 **Deployment successful** 메시지가 표시됩니다.

7.  **Deployment** 창에서 **Go to resource**를 선택합니다.

8.  **Azure Cosmos DB account** 창의 리소스 메뉴에서 **Data Explorer**를 선택합니다.

9.  **Data Explorer** 창에서 **New Container**를 확장한 다음 **New Database**를 선택합니다.

10. **New Database** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | `nothroughputdb` |
| **Provision throughput** | *Unchecked* |

> **Note: 처리량 프로비저닝 위치**
>
> 처리량은 **데이터베이스 수준** 또는 **컨테이너 수준**에서 프로비저닝할 수 있습니다.
> - **컨테이너 수준**: 각 컨테이너에 대해 전용 처리량을 할당합니다. 성능 보장이 중요한 컨테이너에 적합합니다. 이 시나리오에서는 데이터베이스 자체에는 처리량을 프로비저닝하지 않습니다.
> - **데이터베이스 수준**: 데이터베이스에 처리량을 할당하고, 해당 데이터베이스 내의 모든 컨테이너가 이 처리량을 공유합니다. 개별 컨테이너의 사용량이 적거나 워크로드가 분산되어 있을 때 비용 효율적입니다.

11. **Data Explorer** 창으로 돌아와 계층 구조 내에서 **nothroughputdb** 데이터베이스 노드를 확인합니다.

12. **Data Explorer** 창에서 **New Container**를 선택합니다.

13. **New Container** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다. 이 단계에서는 데이터베이스에 처리량이 할당되지 않았으므로, 컨테이너에 반드시 처리량을 할당해야 합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | *Use existing* | *nothroughputdb* |
| **Container id** | `requiredthroughputcontainer` |
| **Partition key** | `/primarykey` |
| **Container throughput** | *Manual* |
| **RU/s** | `400` |

14. **Data Explorer** 창으로 돌아와 **nothroughputdb** 데이터베이스 노드를 확장하고 계층 구조 내에서 **requiredthroughputcontainer** 컨테이너 노드를 확인합니다.

15. **Data Explorer** 창에서 **New Container**를 확장한 다음 **New Database**를 선택합니다. 이번에는 데이터베이스 수준에서 처리량을 프로비저닝합니다.

16. **New Database** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | `manualthroughputdb` |
| **Provision throughput** | *Checked* |
| **Database throughput** | *Manual* |
| **RU/s** | `400` |

17. **Data Explorer** 창으로 돌아와 계층 구조 내에서 **manualthroughputdb** 데이터베이스 노드를 확인합니다.

18. **Data Explorer** 창에서 **New Container**를 선택합니다.

19. **New Container** 팝업에서 각 설정에 다음 값을 입력하고 **OK**를 선택합니다. 이 컨테이너는 데이터베이스의 공유 처리량을 사용하는 대신, 자체 전용 처리량을 갖게 됩니다.

| **Setting** | **Value** |
| :--- | :--- |
| **Database id** | *Use existing* | *manualthroughputdb* |
| **Container id** | `childcontainer` |
| **Partition key** | `/primarykey` |
| **Provision dedicated throughput for this container** | *Checked* |
| **Container throughput** | *Manual* |
| **RU/s** | `1000` |

20. **Data Explorer** 창으로 돌아와 **manualthroughputdb** 데이터베이스 노드를 확장하고 계층 구조 내에서 **childcontainer** 컨테이너 노드를 확인합니다.

