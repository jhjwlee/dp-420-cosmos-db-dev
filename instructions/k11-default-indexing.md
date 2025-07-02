# 포털을 사용하여 Azure Cosmos DB for NoSQL 컨테이너의 기본 인덱스 정책 검토

모든 Azure Cosmos DB 컨테이너는 컨테이너 내의 항목(item)을 어떻게 인덱싱할지 서비스에 지시하는 인덱싱 정책(indexing policy)을 가지고 있습니다. 기본적으로 이 인덱싱 정책은 모든 항목의 모든 속성(property)을 인덱싱합니다. 이 기본 인덱싱 정책 덕분에 프로젝트 시작 단계에서 인덱싱, 성능, 관리에 대해 고민할 필요 없이 Azure Cosmos DB를 빠르고 쉽게 시작할 수 있습니다.

이 랩에서는 Data Explorer를 사용하여 몇몇 컨테이너의 기본 인덱스 정책을 관찰하고 조작해 볼 것입니다.

> **[배경지식: 인덱싱 정책(Indexing Policy)]**
> 인덱싱 정책은 Azure Cosmos DB가 컨테이너의 데이터를 어떻게 인덱싱할지 정의하는 규칙 집합입니다. 어떤 속성을 인덱스에 포함할지(Included Paths), 제외할지(Excluded Paths), 인덱싱 모드(일관된 인덱싱 또는 없음) 등을 설정할 수 있습니다. 잘 설계된 인덱싱 정책은 쿼리 성능을 크게 향상시키고, RU(Request Unit) 비용을 절감하는 데 매우 중요합니다. 기본 정책은 모든 경로(`/*`)를 인덱싱하여 개발 초기 단계에서 편의성을 제공하지만, 프로덕션 환경에서는 쿼리 패턴에 맞게 최적화하는 것이 좋습니다.

## Azure Cosmos DB for NoSQL 계정 생성하기

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다. Azure Cosmos DB for NoSQL 계정 프로비저닝이 완료되면, 엔드포인트(endpoint)와 키(key)를 검색하여 Azure SDK for .NET이나 원하는 다른 SDK를 사용해 계정에 연결할 수 있습니다.

1.  새 웹 브라우저 창이나 탭에서 Azure portal (``portal.azure.com``)로 이동합니다.

2.  구독과 연결된 Microsoft 자격 증명을 사용하여 포털에 로그인합니다.

3.  **+ 리소스 만들기**를 선택하고, *Cosmos DB*를 검색한 후, 다음 설정을 사용하여 새로운 **Azure Cosmos DB for NoSQL** 계정 리소스를 생성합니다. 나머지 모든 설정은 기본값으로 둡니다.

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
    1.  **PRIMARY CONNECTION STRING** 필드를 확인합니다. 이 **connection string** 값은 이 연습의 뒷부분에서 사용됩니다.

> **[Note: Connection String]**
> 연결 문자열(Connection String)은 데이터베이스에 프로그래매틱하게 접근하기 위한 모든 정보를 담고 있는 문자열입니다. 일반적으로 계정의 주소(Endpoint), 데이터베이스 이름, 그리고 인증을 위한 키(Key)나 암호를 포함합니다. 애플리케이션은 이 문자열 하나만으로 데이터베이스에 안전하게 연결할 수 있습니다.

## Azure Cosmos DB for NoSQL 계정에 데이터 시딩(Seed)하기

[cosmicworks][nuget.org/packages/cosmicworks] 명령줄 도구는 모든 Azure Cosmos DB for NoSQL 계정에 샘플 데이터를 배포합니다. 이 도구는 오픈 소스이며 NuGet을 통해 사용할 수 있습니다. 이 도구를 Azure Cloud Shell에 설치한 후 데이터베이스를 시딩하는 데 사용할 것입니다.

1.  **Visual Studio Code**를 시작합니다.

2.  **Visual Studio Code**에서 **터미널(Terminal)** 메뉴를 열고 **새 터미널(New Terminal)**을 선택하여 새 터미널 인스턴스를 엽니다.

    > &#128221; Visual Studio Code 인터페이스에 익숙하지 않다면, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

3.  머신에서 전역적으로 사용할 수 있도록 [cosmicworks][nuget.org/packages/cosmicworks] 명령줄 도구를 설치합니다.

    ```
    dotnet tool install --global CosmicWorks --version 2.3.1
    ```
  
    > &#128161; 이 명령은 완료하는 데 몇 분이 걸릴 수 있습니다. 이전에 이 도구의 최신 버전을 이미 설치한 경우, `Tool 'cosmicworks' is already installed`라는 경고 메시지가 출력됩니다.

4.  다음 명령줄 옵션을 사용하여 `cosmicworks`를 실행하여 Azure Cosmos DB 계정을 시딩합니다.

| **옵션** | **값** |
| ---: | :--- |
| **-c** | *이 랩의 앞부분에서 확인한 connection string 값* |
| **--number-of-employees** | *달리 지정하지 않으면 `cosmicworks` 명령은 데이터베이스에 각각 1000개와 200개의 항목을 가진 `employees`와 `products` 컨테이너를 채웁니다.* |

    ```powershell
    cosmicworks -c "connection-string" --number-of-employees 0 --disable-hierarchical-partition-keys
    ```
    > **[코드 설명]**
    > 이 명령어는 `cosmicworks` 도구를 사용하여 Cosmos DB 계정에 데이터를 채웁니다.
    > - `cosmicworks`: 도구 실행 명령어입니다.
    > - `-c "connection-string"`: `-c` 또는 `--connection-string` 옵션은 연결할 Cosmos DB 계정의 `PRIMARY CONNECTION STRING`을 지정합니다.
    > - `--number-of-employees 0`: 이 랩에서는 `products` 데이터만 필요하므로, `employees` 항목은 생성하지 않도록 `0`으로 설정합니다.
    > - `--disable-hierarchical-partition-keys`: 계층적 파티션 키 기능을 비활성화합니다. 이 랩에서는 단일 파티션 키를 사용합니다.

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이고 키가 **fDR2ci9QgkdkvERTQ==** 이라면, 명령어는 다음과 같습니다:
    > ``cosmicworks -c "AccountEndpoint=https://dp420.documents.azure.com:443/;AccountKey=fDR2ci9QgkdkvERTQ==" --number-of-employees 0 --disable-hierarchical-partition-keys``

5.  **cosmicworks** 명령이 계정에 데이터베이스, 컨테이너, 그리고 항목들로 채우는 것을 완료할 때까지 기다립니다.

6.  통합 터미널을 닫습니다.

## 기본 인덱싱 정책 보기 및 조작하기

코드, 포털 또는 도구를 통해 컨테이너를 생성할 때, 별도로 지정하지 않으면 인덱싱 정책은 지능적인 기본값으로 설정됩니다. 이 기본 인덱싱 정책을 관찰하고 정책을 변경해 보겠습니다.

1.  웹 브라우저로 돌아갑니다.

2.  **Azure Cosmos DB** 계정 리소스 내에서 **Data Explorer** 창으로 이동합니다.

3.  **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장한 다음, **NOSQL API** 탐색 트리 내의 새로운 **products** 컨테이너 노드를 확인합니다.

4.  **NOSQL API** 탐색 트리에서 **products** 컨테이너 노드를 선택한 다음, **New SQL Query**를 선택합니다.

5.  편집기 영역의 내용을 삭제합니다.

6.  **name**이 **HL Headset**과 동일한 모든 문서를 반환하는 새 SQL 쿼리를 작성합니다:

    ```sql
    SELECT * FROM p WHERE p.name = 'HL Headset'
    ```
    > **[코드 설명]**
    > - `SELECT * FROM p`: `p`라는 별칭으로 지정된 `products` 컨테이너에서 모든 속성(`*`)을 선택합니다.
    > - `WHERE p.name = 'HL Headset'`: `name` 속성의 값이 'HL Headset'인 항목만 필터링합니다.

7.  **Execute Query**를 선택합니다.

8.  쿼리 결과를 관찰합니다.

9.  **Query** 탭에서 **Query Stats**를 선택합니다.

10. **Query Statistics** 섹션 내의 **Request Charge** 필드 값을 관찰합니다.

> **[Note: Request Charge & RU]**
> Request Charge(요청 요금)는 쿼리나 데이터베이스 작업이 소비한 처리량을 RU(Request Unit) 단위로 측정한 값입니다. RU는 CPU, 메모리, IOPS(초당 입출력 작업 수)를 추상화한 성능 단위입니다. 쿼리가 효율적일수록(즉, 인덱스를 잘 활용할수록) 더 적은 RU를 소비하므로 비용이 절감됩니다.

> &#128221; 현재 모든 경로가 인덱싱되어 있으므로 이 쿼리는 비교적 효율적이어야 합니다.

11. **NOSQL API** 탐색 트리의 **products** 컨테이너 노드 내에서 **Settings**를 선택합니다.

12. **Indexing Policy** 섹션에서 기본 인덱싱 정책을 관찰합니다:

    ```json
    {
      "indexingMode": "consistent",
      "automatic": true,
      "includedPaths": [
        {
          "path": "/*"
        }
      ],
      "excludedPaths": [
        {
          "path": "/\"_etag\"/?"
        }
      ]
    }
    ```
    > **[코드 설명: 기본 인덱싱 정책]**
    > - `"indexingMode": "consistent"`: 모든 쓰기 작업에 대해 인덱스가 동기적으로 업데이트됩니다. 쿼리는 항상 최신 데이터를 반영합니다.
    > - `"automatic": true"`: 항목이 생성, 수정 또는 삭제될 때 인덱스가 자동으로 업데이트됩니다.
    > - `"includedPaths"`: 인덱스에 포함될 경로를 지정합니다. `{"path": "/*"}`는 모든 속성(모든 경로)을 인덱싱하라는 의미의 와일드카드입니다.
    > - `"excludedPaths"`: 인덱스에서 제외할 경로를 지정합니다. `{"path": "/\"_etag\"/?"}`는 Cosmos DB의 내부 시스템 속성인 `_etag`를 인덱싱에서 제외합니다. `_etag`는 낙관적 동시성 제어(optimistic concurrency control)에 사용됩니다.

    > &#128221; 이 기본 정책은 **_etag**를 제외한 모든 가능한 경로를 인덱싱합니다.

13. 편집기 내에서 인덱싱 정책의 내용을 **/price** 경로만 인덱싱하도록 교체합니다:

    ```json
    {
      "indexingMode": "consistent",
      "automatic": true,
      "includedPaths": [
        {
          "path": "/price/?"
        }
      ],
      "excludedPaths": [
        {
          "path": "/*"
        }
      ]
    }
    ```
    > **[코드 설명: 수정된 인덱싱 정책]**
    > - `"includedPaths": [{"path": "/price/?"}]`: 이제 `price` 속성만 인덱스에 포함됩니다. `?` 와일드카드는 해당 경로의 스칼라 값(문자열, 숫자 등)만 인덱싱하라는 의미입니다.
    > - `"excludedPaths": [{"path": "/*"}]`: `includedPaths`에 명시적으로 포함되지 않은 다른 모든 경로(`/*`)를 인덱스에서 제외합니다. 이 규칙 때문에 `name` 속성은 더 이상 인덱싱되지 않습니다.

14. **Save**를 선택하여 변경 사항을 저장합니다.

15. **New SQL Query**를 선택합니다.

16. 편집기 영역의 내용을 삭제합니다.

17. **name**이 **HL Headset**과 동일한 모든 문서를 반환하는 새 SQL 쿼리를 다시 작성합니다:

    ```sql
    SELECT * FROM p WHERE p.name = 'HL Headset'
    ```

18. **Execute Query**를 선택합니다.

19. 쿼리 결과를 관찰합니다.

20. **Query** 탭에서 **Query Stats**를 선택합니다.

21. **Query Statistics** 섹션 내의 **Request Charge** 필드 값을 관찰합니다.

> &#128221; 이제 **name** 속성이 인덱싱되지 않았으므로, 요청 요금(Request Charge)이 증가했습니다.
> **[Note]** 쿼리 엔진은 `name` 필터 조건에 맞는 항목을 찾기 위해 인덱스를 사용할 수 없으므로, 컨테이너의 모든 항목을 스캔(scan)해야 합니다. 이 스캔 작업은 인덱스를 사용하는 것보다 훨씬 더 많은 리소스(RU)를 소모하므로 요청 요금이 크게 증가합니다.

22. 편집기 영역의 내용을 삭제합니다.

23. **price**가 **$3,000**보다 큰 모든 문서를 반환하는 새 SQL 쿼리를 작성합니다:

    ```sql
    SELECT * FROM p WHERE p.price > 3000
    ```
    > **[코드 설명]**
    > `WHERE p.price > 3000`: `price` 속성의 값이 `3000`보다 큰 항목만 필터링합니다. 이 쿼리는 범위(range) 쿼리입니다.

24. **Execute Query**를 선택합니다.

25. 쿼리 결과를 관찰합니다.

26. **Query** 탭에서 **Query Stats**를 선택합니다.

27. **Query Statistics** 섹션 내의 **Request Charge** 필드 값을 관찰합니다. 이 쿼리는 `price` 속성이 인덱싱되어 있으므로 이전 `name` 쿼리보다 훨씬 효율적이고 RU 비용이 낮을 것입니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
[nuget.org/packages/cosmicworks]: https://www.nuget.org/packages/cosmicworks/
