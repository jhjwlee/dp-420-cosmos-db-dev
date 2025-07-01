
# 07.1 - Azure Cosmos DB for NoSQL의 Vector Search 활성화
---
Azure Cosmos DB for NoSQL은 고차원 벡터를 모든 규모에서 효율적이고 정확하게 저장하고 쿼리하도록 설계된 효율적인 벡터 인덱싱 및 검색 기능을 제공합니다. 이 기능을 활용하려면, 계정에서 *Vector Search for NoSQL API* 기능을 사용하도록 설정해야 합니다.

이 실습에서는 Azure Cosmos DB for NoSQL 계정을 생성하고 여기에 `Vector Search` 기능을 활성화하여, 데이터베이스를 벡터 저장소(vector store)로 사용할 수 있도록 준비합니다.

> **배경지식: 벡터(Vector)와 벡터 검색(Vector Search)**
>
> **벡터 (Vector) 또는 임베딩 (Embedding):**
> 텍스트, 이미지, 오디오와 같은 복잡한 데이터를 기계가 이해할 수 있는 숫자 배열로 변환한 것입니다. 예를 들어, "강아지"라는 단어와 "개"라는 단어는 의미가 매우 유사하므로, 벡터 공간(vector space)에서 서로 가까운 위치에 표현됩니다. 이러한 숫자 배열을 생성하는 과정을 **임베딩(embedding)**이라고 하며, 보통 OpenAI의 `text-embedding-3-small`과 같은 AI 모델을 사용합니다.
>
> **벡터 검색 (Vector Search):**
> 기존의 키워드 기반 검색(예: "노트북"이라는 단어가 포함된 문서 찾기)과 달리, **의미 기반 검색(semantic search)**을 수행합니다. 사용자의 쿼리를 벡터로 변환한 다음, 데이터베이스에 저장된 수많은 벡터 중에서 이 쿼리 벡터와 가장 "유사한" (가까운) 벡터들을 찾아냅니다. 이를 통해 "가볍고 성능 좋은 휴대용 컴퓨터"라고 검색해도 "노트북"이나 "랩탑"이 포함된 결과를 찾을 수 있습니다. 이 유사도 검색을 **ANN(Approximate Nearest Neighbor, 근사 근접 이웃)** 검색이라고도 합니다.

## 개발 환경 준비

아직 **Build copilots with Azure Cosmos DB**에 대한 실습 코드 리포지토리를 복제하고 로컬 환경을 설정하지 않았다면, [로컬 실습 환경 설정](00-setup-lab-environment.md) 지침을 참조하여 설정하십시오.

## Azure Cosmos DB for NoSQL 계정 생성

이 사이트의 **Build copilots with Azure Cosmos DB** 실습을 위해 이미 Azure Cosmos DB for NoSQL 계정을 생성했다면, 해당 계정을 이 실습에서도 사용할 수 있으며 [다음 섹션](#enable-vector-search-for-nosql-api)으로 건너뛸 수 있습니다. 그렇지 않은 경우, [Azure Cosmos DB 설정](../../common/instructions/00-setup-cosmos-db.md) 지침을 참조하여 실습 모듈 전반에 걸쳐 사용할 Azure Cosmos DB for NoSQL 계정을 생성하고, 사용자 ID에 **Cosmos DB Built-in Data Contributor** 역할을 할당하여 계정의 데이터를 관리할 수 있는 액세스 권한을 부여하십시오.

> **Note: Cosmos DB Built-in Data Contributor 역할**
> 이 역할은 Azure Active Directory(Azure AD) ID(사용자, 그룹, 서비스 주체 등)가 Azure Cosmos DB 계정의 데이터(데이터베이스, 컨테이너, 항목)에 접근하고 수정할 수 있도록 허용하는 데이터 평면(data plane) 권한을 부여합니다. 관리 평면(management plane, 계정 자체의 설정 변경 등) 권한과는 분리되어 있어 보안에 유리합니다.

## Enable Vector Search for NoSQL API

이 작업에서는 Azure CLI를 사용하여 Azure Cosmos DB 계정에서 *Vector Search for NoSQL API* 기능을 활성화합니다.

1.  [Azure portal](https://portal.azure.com)의 도구 모음에서 **Cloud Shell**을 엽니다.

    ![Azure portal의 도구 모음에서 Cloud Shell 아이콘이 강조 표시됨](media/07-azure-portal-toolbar-cloud-shell.png)

2.  Cloud Shell 프롬프트에서, `<SUBSCRIPTION_ID>` 자리 표시자 토큰을 이 실습에 사용하는 구독의 ID로 바꾸어 `az account set -s <SUBSCRIPTION_ID>`를 실행하여 후속 명령에 실습용 구독이 사용되도록 합니다.

3.  Azure Cloud Shell에서 다음 명령을 실행하여 *Vector Search for NoSQL API* 기능을 활성화합니다. `<RESOURCE_GROUP_NAME>`과 `<COSMOS_DB_ACCOUNT_NAME>` 토큰을 각각 리소스 그룹 이름과 Azure Cosmos DB 계정 이름으로 바꾸십시오.

    ```bash
    az cosmosdb update \
      --resource-group <RESOURCE_GROUP_NAME> \
      --name <COSMOS_DB_ACCOUNT_NAME> \
      --capabilities EnableNoSQLVectorSearch
    ```
    > **Note: `--capabilities EnableNoSQLVectorSearch`**
    > `Vector Search`는 현재 미리보기(preview) 기능으로 제공될 수 있으므로, 계정 수준에서 명시적으로 기능을 활성화해야 합니다. 이 `az cosmosdb update` 명령은 `--capabilities` 플래그를 사용하여 `EnableNoSQLVectorSearch`라는 기능을 계정에 추가합니다.

4.  Cloud Shell을 종료하기 전에 명령이 성공적으로 실행될 때까지 기다립니다.

5.  Cloud Shell을 닫습니다.

## 벡터 호스팅을 위한 데이터베이스 및 컨테이너 생성

1.  [Azure portal](https://portal.azure.com)의 Azure Cosmos DB 계정 왼쪽 메뉴에서 **Data Explorer**를 선택한 다음, **New Container**를 선택합니다.

2.  **New Container** 대화 상자에서:
    1.  **Database id** 아래에서 **Create new**를 선택하고 데이터베이스 ID 필드에 "CosmicWorks"를 입력합니다.
    2.  **Container id** 상자에 "Products"라는 이름을 입력합니다.
    3.  **/category_id**를 **Partition key**로 할당합니다.

        ![위에 지정된 New Container 설정이 대화 상자에 입력된 스크린샷](media/07-azure-cosmos-db-new-container.png)

    4.  **New Container** 대화 상자의 맨 아래로 스크롤하여 **Container Vector Policy**를 확장하고 **Add vector embedding**을 선택합니다.

    5.  **Container Vector Policy** 설정 섹션에서 다음을 설정합니다.

        | Setting | Value |
        | --- | --- |
        | **Path** | */embedding*을 입력합니다. |
        | **Data type** | *float32*를 선택합니다. |
        | **Distance function** | *cosine*을 선택합니다. |
        | **Dimensions** | OpenAI의 `text-embedding-3-small` 모델이 생성하는 차원 수와 일치하도록 *1536*을 입력합니다. |
        | **Index type** | *diskANN*을 선택합니다. |
        | **Quantization byte size** | 이 필드는 비워 둡니다. |
        | **Indexing search list size**| 기본값인 *100*을 그대로 사용합니다. |

        ![위에 지정된 Container Vector Policy가 New Container 대화 상자에 입력된 스크린샷](media/07-azure-cosmos-db-container-vector-policy.png)

    > **Note: 벡터 정책(Vector Policy) 설정 설명**
    >
    > *   **Path**: 각 JSON 문서 내에서 벡터(임베딩) 배열이 저장될 속성의 경로입니다. 여기서는 모든 문서에 `/embedding`이라는 필드가 있을 것으로 예상합니다.
    > *   **Data type**: 벡터를 구성하는 숫자의 데이터 타입입니다. `float32`는 임베딩에 널리 사용되는 형식입니다.
    > *   **Distance function**: 두 벡터 간의 유사성을 계산하는 방법입니다.
    >     *   **`cosine` (코사인 유사도)**: 두 벡터 사이의 각도를 측정하여 방향의 유사성을 판단합니다. 텍스트 의미 검색에 가장 일반적으로 사용됩니다.
    >     *   **`l2` (유클리드 거리)**: 두 벡터 끝점 사이의 직선 거리를 측정합니다. 이미지 인식 등에서 사용됩니다.
    >     *   **`dot` (내적)**: 두 벡터의 크기와 방향을 모두 고려합니다.
    > *   **Dimensions**: 각 벡터 배열의 길이(차원 수)입니다. 이 값은 임베딩을 생성하는 데 사용한 AI 모델의 출력 차원과 **정확히 일치해야 합니다.** `text-embedding-3-small`은 1536차원 벡터를 생성합니다.
    > *   **Index type**: 벡터 검색을 위해 사용할 인덱스의 종류입니다.
    >     *   **`diskANN`**: Microsoft에서 개발한 고성능 근사 근접 이웃(ANN) 인덱스로, 대규모 데이터셋에 대해 속도와 정확성의 균형을 잘 맞춘 최신 기술입니다.
    >     *   **`flat` (미래 지원)**: 모든 벡터를 스캔하여 정확한 결과를 찾지만, 데이터가 많아지면 매우 느려집니다.
    >     *   **`quantizedFlat`**: 벡터를 압축하여 메모리 사용량을 줄인 flat 인덱스입니다.
    > *   **Quantization**: 벡터를 압축하여 메모리 및 스토리지 사용량을 줄이는 기술입니다. 약간의 정확도 손실을 감수하고 성능을 높일 때 사용합니다. 여기서는 비활성화합니다.

    6.  **OK**를 선택하여 데이터베이스와 컨테이너를 생성합니다.

    7.  계속 진행하기 전에 컨테이너가 생성될 때까지 기다립니다. 컨테이너가 준비되는 데 몇 분 정도 걸릴 수 있습니다.
