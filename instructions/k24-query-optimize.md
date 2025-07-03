# 특정 쿼리에 맞게 Azure Cosmos DB for NoSQL 컨테이너의 인덱싱 정책 최적화하기

Azure Cosmos DB for NoSQL 계정을 계획할 때, 가장 자주 사용되는 쿼리를 파악하고 있으면 쿼리가 가능한 한 최상의 성능을 발휘하도록 인덱싱 정책을 조정하는 데 도움이 됩니다.

이 랩에서는 Data Explorer를 사용하여 기본 인덱싱 정책과 복합 인덱스(composite index)가 포함된 인덱싱 정책으로 SQL 쿼리를 테스트해 보겠습니다.

> **[배경지식: 쿼리 최적화와 인덱싱 정책]**
> Azure Cosmos DB에서 쿼리 성능은 요청 단위(Request Units, RU)로 측정됩니다. 쿼리를 실행하는 데 필요한 RU가 적을수록 성능이 좋고 비용이 저렴합니다. 인덱싱 정책은 쿼리 엔진이 데이터를 효율적으로 찾을 수 있도록 돕는 가장 중요한 도구입니다.
> - **기본 인덱싱 정책:** 모든 속성을 인덱싱하여 개발 초기에는 편리하지만, 불필요한 인덱스 때문에 쓰기 성능 저하와 저장 공간 낭비를 유발할 수 있습니다.
> - **사용자 지정 인덱싱 정책:** 자주 사용하는 쿼리 패턴에 맞춰 필요한 속성만 인덱싱하거나, 이 랩에서 다룰 복합 인덱스를 추가하여 특정 유형의 쿼리(특히 `ORDER BY` 절이 있는 쿼리) 성능을 극적으로 향상시킬 수 있습니다.

## Azure Cosmos DB for NoSQL 계정 생성하기

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다. Azure Cosmos DB for NoSQL 계정 프로비저닝이 완료되면, 엔드포인트(endpoint)와 키(key)를 검색하여 Azure SDK for .NET이나 원하는 다른 SDK를 사용해 계정에 연결할 수 있습니다.

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

5.  새로 생성된 **Azure Cosmos DB** 계정 리소스로 이동하여 **Data Explorer** 창으로 이동합니다.

6.  **Data Explorer** 창에서 **New Container**를 확장한 다음 **New Database**를 선택합니다.

7.  **New Database** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *``cosmicworks``* |
| **Provision throughput** | enabled |
| **Database throughput** | **Manual** |
| **Database Required RU/s** | ``1000`` |

8.  **Data Explorer** 창으로 돌아와서 계층 구조 내의 **cosmicworks** 데이터베이스 노드를 확인합니다.

9.  **Data Explorer** 창에서 **New Container**를 선택합니다.

10. **New Container** 팝업에서 각 설정에 다음 값을 입력한 다음 **OK**를 선택합니다:

| **설정** | **값** |
| --: | :-- |
| **Database id** | *Use existing* &vert; *cosmicworks* |
| **Container id** | *``products``* |
| **Partition key** | *``/category/name``* |

11. **Data Explorer** 창으로 돌아와서 **cosmicworks** 데이터베이스 노드를 확장한 다음 계층 구조 내의 **products** 컨테이너 노드를 확인합니다.

12. 리소스 블레이드에서 **Keys** 창으로 이동합니다.

13. 이 창에는 SDK에서 계정에 연결하는 데 필요한 연결 정보와 자격 증명이 포함되어 있습니다. 특히 다음을 확인하세요:
    1.  **PRIMARY CONNECTION STRING** 필드를 확인합니다. 이 **connection string** 값은 이 연습의 뒷부분에서 사용됩니다.

14. **Visual Studio Code**를 엽니다.

## Azure Cosmos DB for NoSQL 계정에 샘플 데이터 시딩하기

명령줄 유틸리티를 사용하여 **cosmicworks** 데이터베이스와 **products** 컨테이너를 생성합니다. 그런 다음 이 도구는 터미널 창에서 실행 중인 변경 피드 프로세서를 통해 관찰할 항목 세트를 생성합니다.

1.  **Visual Studio Code**에서 **터미널(Terminal)** 메뉴를 열고 **새 터미널(New Terminal)**을 선택하여 새 터미널을 엽니다.

2.  머신에서 전역적으로 사용할 수 있도록 [cosmicworks][nuget.org/packages/cosmicworks] 명령줄 도구를 설치합니다.

    ```
    dotnet tool install --global CosmicWorks --version 2.3.1
    ```

    > &#128161; 이 명령은 완료하는 데 몇 분이 걸릴 수 있습니다. 이전에 이 도구의 최신 버전을 이미 설치한 경우, `Tool 'cosmicworks' is already installed`라는 경고 메시지가 출력됩니다.

3.  다음 명령줄 옵션을 사용하여 cosmicworks를 실행하여 Azure Cosmos DB 계정을 시딩합니다:

| **옵션** | **값** |
| ---: | :--- |
| **-c** | *이 랩의 앞부분에서 확인한 connection string 값* |
| **--number-of-employees** | *달리 지정하지 않으면 cosmicworks 명령은 데이터베이스에 각각 1000개와 200개의 항목을 가진 employees와 products 컨테이너를 채웁니다.* |

    ```powershell
    cosmicworks -c "connection-string" --number-of-employees 0 --disable-hierarchical-partition-keys
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이고 키가 **fDR2ci9QgkdkvERTQ==** 라면, 명령어는 다음과 같습니다:
    > ``cosmicworks -c "AccountEndpoint=https://dp420.documents.azure.com:443/;AccountKey=fDR2ci9QgkdkvERTQ==" --number-of-employees 0 --disable-hierarchical-partition-keys``

4.  **cosmicworks** 명령이 계정에 데이터베이스, 컨테이너, 그리고 항목들로 채우는 것을 완료할 때까지 기다립니다.

5.  통합 터미널을 닫습니다.

6.  **Visual Studio Code**를 닫고 브라우저로 돌아갑니다.

## SQL 쿼리를 실행하고 요청 단위 요금 측정하기

인덱싱 정책을 수정하기 전에, 먼저 몇 가지 샘플 SQL 쿼리를 실행하여 기준 요청 단위 요금(RU로 표현)을 얻습니다.

1.  **Azure Cosmos DB** 계정 리소스 내에서 **Data Explorer** 창으로 이동합니다.

2.  **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 선택한 다음, **New SQL Query**를 선택합니다.

3.  기본 쿼리를 실행하려면 **Execute Query**를 선택합니다:

    ```sql
    SELECT * FROM c
    ```

4.  쿼리 결과를 관찰합니다. RU 단위의 요청 단위 요금을 보려면 **Query Stats**를 선택합니다.

5.  편집기 영역의 내용을 삭제합니다.

6.  모든 문서에서 세 가지 값을 반환하는 새 SQL 쿼리를 작성합니다:

    ```sql
    SELECT 
        p.name,
        p.category,
        p.price
    FROM
        products p    
    ```

7.  **Execute Query**를 선택합니다.

8.  쿼리의 결과와 통계를 관찰합니다. 요청 단위 요금은 첫 번째 쿼리와 거의 동일합니다.

9.  편집기 영역의 내용을 삭제합니다.

10. **categoryName**으로 정렬된 모든 문서에서 세 가지 값을 반환하는 새 SQL 쿼리를 작성합니다:

    ```sql
    SELECT 
        p.name,
        p.category,
        p.price
    FROM
        products p
    ORDER BY
        p.category DESC
    ```

11. **Execute Query**를 선택합니다.

12. 쿼리의 결과와 통계를 관찰합니다. **ORDER BY** 절 때문에 요청 단위 요금이 증가한 것을 볼 수 있습니다.

> **[Note: ORDER BY와 RU]**
> `ORDER BY` 절이 있는 쿼리는 단순히 데이터를 필터링하는 것보다 더 많은 리소스를 필요로 합니다. 쿼리 엔진은 결과를 정렬하기 위해 추가적인 계산을 수행해야 하므로 RU 소비량이 증가합니다. 이 비용은 특히 복합 인덱스가 없을 때 더 두드러집니다.

## 인덱싱 정책에 복합 인덱스 생성하기

이제 여러 속성을 사용하여 항목을 정렬해야 하는 경우 복합 인덱스(composite index)를 만들어야 합니다. 이 작업에서는 카테고리 이름과 실제 이름으로 항목을 정렬하기 위한 복합 인덱스를 생성합니다.

> **[배경지식: 복합 인덱스 (Composite Index)]**
> 복합 인덱스는 **두 개 이상의 속성**을 결합하여 하나의 인덱스로 만드는 것입니다. 이는 **여러 속성에 대한 `ORDER BY` 절**이 포함된 쿼리의 성능을 최적화하는 데 필수적입니다.
>
> 예를 들어, `ORDER BY p.category DESC, p.price ASC`와 같은 쿼리를 효율적으로 처리하려면, `category`와 `price` 두 속성을 모두 포함하고, 쿼리와 동일한 정렬 순서(`descending`, `ascending`)를 가지는 복합 인덱스가 필요합니다. 복합 인덱스가 없으면 Cosmos DB는 이 쿼리를 실행하지 못하고 오류를 반환합니다.

1.  **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 선택한 다음, **New SQL Query**를 선택합니다.

2.  편집기 영역의 내용을 삭제합니다.

3.  결과를 먼저 **category**로 내림차순 정렬하고, 그 다음 **price**로 오름차순 정렬하는 새 SQL 쿼리를 작성합니다:

    ```sql
    SELECT 
        p.name,
        p.category,
        p.price
    FROM
        products p
    ORDER BY
        p.category DESC,
        p.price ASC
    ```

4.  **Execute Query**를 선택합니다.

5.  쿼리는 **The order by query does not have a corresponding composite index that it can be served from** 라는 오류와 함께 실패해야 합니다.

6.  **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 확장한 다음, **Settings**를 선택합니다.

7.  **Settings** 탭에서 **Indexing Policy** 섹션으로 이동합니다.

8.  기본 인덱싱 정책을 관찰합니다:

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

9.  인덱싱 정책을 이 수정된 JSON 객체로 교체한 다음 변경 사항을 **Save**합니다:

    ```json
    {
      "indexingMode": "consistent",
      "automatic": true,
      "includedPaths": [
        {
          "path": "/*"
        }
      ],
      "excludedPaths": [],
      "compositeIndexes": [
        [
          {
            "path": "/category",
            "order": "descending"
          },
          {
            "path": "/price",
            "order": "ascending"
          }
        ]
      ]
    }
    ```
    > **[코드 설명: 복합 인덱스 추가]**
    > - `"compositeIndexes": [...]`: 인덱싱 정책에 복합 인덱스를 정의하는 섹션을 추가했습니다.
    > - `[[...]]`: 복합 인덱스는 배열의 배열로 정의됩니다. 각 내부 배열이 하나의 복합 인덱스를 나타냅니다.
    > - `{"path": "/category", "order": "descending"}`: 복합 인덱스의 첫 번째 속성은 `/category`이며, 정렬 순서는 `descending`입니다.
    > - `{"path": "/price", "order": "ascending"}`: 복합 인덱스의 두 번째 속성은 `/price`이며, 정렬 순서는 `ascending`입니다.
    > **중요:** 이 복합 인덱스의 속성 순서와 정렬 순서는 `ORDER BY` 절의 순서와 정확히 일치해야 합니다.

10. **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 선택한 다음, **New SQL Query**를 선택합니다.

11. 편집기 영역의 내용을 삭제합니다.

12. 결과를 먼저 **categoryName**으로 내림차순 정렬하고, 그 다음 **price**로 오름차순 정렬하는 새 SQL 쿼리를 다시 작성합니다:

    ```sql
    SELECT 
        p.name,
        p.category,
        p.price
    FROM
        products p
    ORDER BY
        p.category DESC,
        p.price ASC
    ```

13. **Execute Query**를 선택합니다.

14. 쿼리의 결과와 통계를 관찰합니다. 이번에는 쿼리가 완료되었으므로, RU 요금을 다시 검토할 수 있습니다. 복합 인덱스 덕분에 RU 비용이 이전의 단일 속성 `ORDER BY` 쿼리보다 훨씬 낮아졌을 가능성이 높습니다.

15. 편집기 영역의 내용을 삭제합니다.

16. 결과를 먼저 **category**로 내림차순, 그 다음 **name**으로 오름차순, 마지막으로 **price**로 오름차순 정렬하는 새 SQL 쿼리를 작성합니다:

    ```sql
    SELECT 
        p.name,
        p.category,
        p.price
    FROM
        products p
    ORDER BY
        p.category DESC,
        p.name ASC,
        p.price ASC
    ```

17. **Execute Query**를 선택합니다.

18. 쿼리는 **The order by query does not have a corresponding composite index that it can be served from** 라는 오류와 함께 다시 실패해야 합니다. 이는 3개 속성에 대한 복합 인덱스가 없기 때문입니다.

19. **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 확장한 다음, 다시 **Settings**를 선택합니다.

20. **Settings** 탭에서 **Indexing Policy** 섹션으로 이동합니다.

21. 인덱싱 정책을 이 수정된 JSON 객체로 교체한 다음 변경 사항을 **Save**합니다:

    ```json
    {
      "indexingMode": "consistent",
      "automatic": true,
      "includedPaths": [
        {
          "path": "/*"
        }
      ],
      "excludedPaths": [],
      "compositeIndexes": [
        [
          {
            "path": "/category",
            "order": "descending"
          },
          {
            "path": "/price",
            "order": "ascending"
          }
        ],
        [
          {
            "path": "/category",
            "order": "descending"
          },
          {
            "path": "/name",
            "order": "ascending"
          },
          {
            "path": "/price",
            "order": "ascending"
          }
        ]
      ]
    }
    ```
    > **[코드 설명]**
    > 이제 `compositeIndexes` 배열에 두 개의 복합 인덱스 정의가 포함됩니다. 첫 번째는 이전과 동일하고, 두 번째는 `(category DESC, name ASC, price ASC)` 쿼리를 지원하기 위해 새로 추가된 것입니다.

22. **Data Explorer**에서 **cosmicworks** 데이터베이스 노드를 확장하고, **products** 컨테이너 노드를 선택한 다음, **New SQL Query**를 선택합니다.

23. 편집기 영역의 내용을 삭제합니다.

24. 결과를 먼저 **category**로 내림차순, 그 다음 **name**으로 오름차순, 마지막으로 **price**로 오름차순 정렬하는 새 SQL 쿼리를 다시 작성합니다:

    ```sql
    SELECT 
        p.name,
        p.category,
        p.price
    FROM
        products p
    ORDER BY
        p.category DESC,
        p.name ASC,
        p.price ASC
    ```

25. **Execute Query**를 선택합니다.

26. 쿼리의 결과와 통계를 관찰합니다. 이번에는 쿼리가 완료되었으므로, RU 요금을 다시 검토할 수 있습니다.

27. 웹 브라우저 창이나 탭을 닫습니다.

[code.visualstudio.com/docs/getstarted]: https://code.visualstudio.com/docs/getstarted/tips-and-tricks
