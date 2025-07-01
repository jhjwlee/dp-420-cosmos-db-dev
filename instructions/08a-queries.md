이전 실습에서 대량으로 생성한 25,000개의 제품 데이터를 활용하여, Azure Cosmos DB for NoSQL 쿼리를 사용해 봅니다.

---

# Azure Cosmos DB for NoSQL 쿼리

| | |
| :--- | :--- |
| **시간** | 약 1시간 |
| **모듈** | 10개 단위 |
| **수준** | 중급 |
| **대상** | 개발자 |
| **제품** | Azure Cosmos DB |

이 모듈에서는 SQL(Structured Query Language)과 유사한 쿼리 언어를 사용하여 Azure Cosmos DB for NoSQL의 데이터를 조회하는 방법을 배웁니다. 이전 실습에서 `products` 컨테이너에 대량으로 삽입한 25,000개의 문서를 사용하여 실제 데이터를 쿼리하는 경험을 쌓게 됩니다.

### 학습 목표

이 모듈을 완료하면 다음을 수행할 수 있습니다.

*   기본적인 SQL 쿼리 생성 및 실행
*   쿼리 결과 투영(Projection) 및 형태 변형
*   쿼리에서 형식 검사 함수 사용
*   다양한 기본 제공 함수(문자열, 수학, 배열)를 활용한 쿼리 작성

### 사전 요구 사항

*   이전 "Move multiple documents in bulk with the Azure Cosmos DB for NoSQL SDK" 실습을 완료하여 `cosmicworks` 데이터베이스와 25,000개의 항목이 포함된 `products` 컨테이너가 준비된 Azure Cosmos DB for NoSQL 계정.
*   Azure Portal 사용 경험.

---

## 소개 (2분)

데이터를 저장하는 것만큼이나 중요한 것은 저장된 데이터를 효과적으로 검색하고 활용하는 것입니다. Azure Cosmos DB for NoSQL은 개발자에게 익숙한 SQL 구문을 제공하여, 스키마가 없는 JSON 문서에서도 강력하고 유연한 쿼리를 실행할 수 있게 해줍니다.

이 실습에서는 Azure Portal의 **Data Explorer**를 주 도구로 사용합니다. Data Explorer는 코드를 작성하지 않고도 쿼리를 신속하게 테스트하고, 결과를 확인하며, 요청 비용(RU)을 측정할 수 있는 훌륭한 환경을 제공합니다.

우리는 25,000개의 제품 데이터가 담긴 컨테이너를 대상으로 `SELECT`, `WHERE`와 같은 기본 구문부터 시작하여, 결과의 형태를 바꾸는 투영(projection), 특정 데이터 타입만 필터링하는 형식 검사, 그리고 복잡한 로직을 처리하는 내장 함수까지 단계별로 학습할 것입니다.

## NoSQL 쿼리 언어 이해 (4분)

Azure Cosmos DB for NoSQL의 쿼리 언어는 관계형 데이터베이스에서 널리 사용되는 ANSI SQL과 많은 유사점을 공유하지만, NoSQL 데이터 모델의 특성에 맞게 조정되었습니다.

**배경지식: SQL for NoSQL의 특징**

*   **계층적 데이터 작업**: 관계형 데이터베이스의 평평한 테이블 구조와 달리, Cosmos DB는 중첩된 객체와 배열을 포함하는 JSON 문서를 다룹니다. 쿼리 언어는 이러한 계층 구조 내의 속성에 접근할 수 있습니다. (`c.category.name` 과 같이 `.` 표기법 사용)
*   **스키마 유연성**: 모든 문서가 동일한 구조를 가질 필요가 없습니다. 따라서 쿼리 작성 시 특정 속성이 존재하지 않을 가능성을 염두에 두어야 합니다. (`IS_DEFINED`와 같은 함수가 유용한 이유)
*   **컨테이너 내 `JOIN`**: 관계형 데이터베이스처럼 여러 테이블(컨테이너)을 `JOIN`하는 기능은 지원하지 않습니다. 대신, 단일 문서 내의 배열 요소를 소스와 결합하는 **자체 조인(self-join)**만 가능합니다.
*   **대/소문자 구분**: 대부분의 키워드(`SELECT`, `FROM` 등)는 대/소문자를 구분하지 않지만, **속성 이름**(예: `c.price`)은 JSON 문서에 정의된 대로 정확히 일치해야 합니다.

가장 기본적인 쿼리 구조는 다음과 같습니다.

```sql
SELECT <property-list>
FROM <container-alias>
WHERE <filter-expression>
ORDER BY <sort-expression>
```

> **Note: `FROM c`의 의미**
>
> 쿼리에서 `FROM c` 또는 `FROM products`와 같은 표현을 보게 될 것입니다. 여기서 `c`나 `products`는 현재 쿼리 중인 컨테이너를 가리키는 **별칭(alias)**입니다. 어떤 이름을 사용해도 무방하지만, 관례적으로 짧은 별칭인 `c` (container의 약자)를 많이 사용합니다. `FROM` 절은 필수이며, 컨테이너 내의 각 항목을 반복하는 소스로 생각할 수 있습니다.

이제 Azure Portal로 이동하여 실제 쿼리를 시작해 봅시다.

1.  Azure Portal에서 이전에 생성한 Azure Cosmos DB 계정으로 이동합니다.
2.  왼쪽 메뉴에서 **Data Explorer**를 선택합니다.
3.  **NOSQL API** 섹션에서 `cosmicworks` 데이터베이스와 `products` 컨테이너를 확장합니다.
4.  `products` 컨테이너를 선택하고, 상단 메뉴에서 **New SQL Query**를 클릭합니다.

이제 쿼리를 작성하고 실행할 준비가 되었습니다.

## NoSQL로 쿼리 만들기 (7분)

가장 기본적인 쿼리부터 시작하여 필터링과 정렬을 추가해 보겠습니다.

1.  **모든 항목 선택 (`SELECT *`)**

    쿼리 편집기에 다음 쿼리를 입력하고 **Execute Query**를 클릭합니다.

    ```sql
    SELECT * FROM c
    ```

    **결과**: 컨테이너의 모든 항목(기본적으로 100개)이 JSON 형식으로 `Results` 탭에 표시됩니다. `*`는 각 문서의 모든 속성을 반환하라는 의미입니다.

2.  **조건부 필터링 (`WHERE`)**

    이제 특정 조건을 만족하는 항목만 찾아보겠습니다. `WHERE` 절은 불필요한 데이터를 걸러내는 데 사용됩니다.

    *   **가격 기준 필터링**: 가격(`price`)이 950달러보다 비싼 제품을 찾아봅시다.

        ```sql
        SELECT *
        FROM c
        WHERE c.price > 950
        ```
        **결과**: `price` 속성 값이 950을 초과하는 제품들만 반환됩니다.

    *   **특정 카테고리 필터링**: 이전 실습에서 `Bogus` 라이브러리는 임의의 카테고리 이름을 생성했습니다. 그 중 하나를 사용하여 필터링해 보겠습니다. "Shoes" 카테고리에 속한 제품을 찾아봅시다.

        ```sql
        SELECT *
        FROM c
        WHERE c.categoryId = "Shoes"
        ```
        **결과**: `categoryId`가 "Shoes"인 제품 목록이 나타납니다.

3.  **결과 정렬 (`ORDER BY`)**

    `ORDER BY` 절을 사용하여 결과를 특정 속성 기준으로 정렬할 수 있습니다.

    *   **가격이 높은 순으로 정렬**: 제품을 가격이 비싼 순서대로 정렬해 보겠습니다. `DESC` 키워드는 내림차순(descending)을 의미합니다.

        ```sql
        SELECT *
        FROM c
        WHERE c.price > 950
        ORDER BY c.price DESC
        ```
        **결과**: 950달러 이상의 제품들이 가격이 가장 높은 것부터 차례로 표시됩니다. 오름차순(ascending)으로 정렬하려면 `ASC`를 사용하거나 생략하면 됩니다(기본값은 오름차순).

> **Note: `ORDER BY`와 인덱싱**
>
> `ORDER BY`를 사용한 쿼리는 기본적으로 인덱싱 정책에 해당 경로가 포함되어 있어야 효율적으로 작동합니다. 단일 속성에 대한 정렬은 대부분 기본 인덱싱 정책으로 처리되지만, 여러 속성으로 정렬하려면(예: `ORDER BY c.categoryId, c.price`) 복합 인덱스(composite index)가 필요할 수 있습니다.

## 쿼리 결과 투영 (10분)

때로는 문서의 전체 내용이 아닌 특정 필드만 필요하거나, 결과의 구조를 새롭게 만들고 싶을 때가 있습니다. 이를 **투영(Projection)**이라고 합니다.

1.  **특정 속성 선택**

    `SELECT *` 대신 필요한 속성의 이름만 명시하여 네트워크 대역폭과 RU 비용을 절약할 수 있습니다. 제품의 이름(`name`)과 가격(`price`)만 선택해 봅시다.

    ```sql
    SELECT c.name, c.price
    FROM c
    WHERE c.price > 950
    ORDER BY c.price DESC
    ```
    **결과**: 각 결과 항목은 이제 `name`과 `price` 두 개의 속성만 가진 JSON 객체입니다.

2.  **별칭(Alias) 사용**

    `AS` 키워드를 사용하여 결과의 속성 이름을 바꿀 수 있습니다.

    ```sql
    SELECT c.name AS ProductName, c.price AS Cost
    FROM c
    WHERE c.price > 950
    ORDER BY c.price DESC
    ```
    **결과**: 속성 이름이 `name`, `price` 대신 `ProductName`, `Cost`로 변경되어 반환됩니다.

3.  **새로운 JSON 객체 생성**

    쿼리 결과로 완전히 새로운 구조의 JSON 객체를 즉석에서 만들 수 있습니다.

    ```sql
    SELECT { "product": c.name, "details": { "cost": c.price, "category": c.categoryId } }
    FROM c
    WHERE c.price > 950
    ```
    **결과**: 각 결과 항목이 지정된 중첩 구조를 가진 단일 JSON 객체로 반환됩니다. `$1`이라는 기본 이름으로 래핑되어 표시될 수 있습니다.

4.  **`VALUE` 키워드 사용**

    결과를 JSON 객체로 래핑하지 않고 값 자체의 배열로 받고 싶을 때 `VALUE` 키워드를 사용합니다. 이는 특정 속성값 목록만 깔끔하게 추출할 때 유용합니다.

    *   가격이 950달러 이상인 모든 제품의 이름만 배열로 반환해 봅시다.

        ```sql
        SELECT VALUE c.name
        FROM c
        WHERE c.price > 950
        ```
        **결과**: `["Refined Steel Computer", "Handcrafted Concrete Pants", ...]` 와 같이 제품 이름 문자열로만 구성된 배열이 반환됩니다.

## 쿼리에서 형식 검사 구현 (9분)

스키마가 없는 NoSQL 데이터베이스에서는 문서마다 속성이 누락되거나 데이터 타입이 다를 수 있습니다. 형식 검사 함수는 이러한 상황에서 쿼리의 안정성을 높여줍니다.

> **배경지식: 형식 검사의 중요성**
>
> 예를 들어, 일부 제품에는 `tags`라는 배열 속성이 있지만 다른 제품에는 없을 수 있습니다. `tags` 배열의 길이를 비교하는 쿼리를 모든 문서에 실행하면, `tags` 속성이 없는 문서에서 오류가 발생할 수 있습니다. 형식 검사 함수를 사용하면 `tags` 속성이 배열 타입으로 존재하는 문서만 안전하게 필터링할 수 있습니다.

1.  **속성 존재 여부 확인 (`IS_DEFINED`)**

    `IS_DEFINED` 함수는 문서에 특정 속성 경로가 정의되어 있는지 여부를 확인합니다. `price` 속성이 정의된 제품만 세어봅시다.

    ```sql
    SELECT VALUE COUNT(1)
    FROM c
    WHERE IS_DEFINED(c.price)
    ```
    **결과**: `price` 속성을 가진 문서의 총 개수(25000)가 반환됩니다.

2.  **데이터 타입 확인 함수**

    다양한 `IS_*` 함수를 사용하여 속성의 타입을 검사할 수 있습니다.
    *   `IS_STRING`, `IS_NUMBER`, `IS_BOOLEAN`
    *   `IS_ARRAY`, `IS_OBJECT`, `IS_NULL`

    *   가격이 숫자 타입이고 980달러보다 큰 제품의 이름을 찾아봅시다.

        ```sql
        SELECT VALUE c.name
        FROM c
        WHERE IS_NUMBER(c.price) AND c.price > 980
        ```
        **결과**: 조건을 만족하는 제품 이름의 배열이 반환됩니다. 이는 데이터가 오염되어 `price`가 ` "990" `과 같은 문자열로 저장된 경우를 배제하는 안전한 쿼리입니다.

## 기본 제공 함수 사용 (실습 위주로 15분)

Azure Cosmos DB는 쿼리를 더욱 강력하게 만들어주는 풍부한 내장 함수 라이브러리를 제공합니다. 문자열, 수학, 배열 함수를 중심으로 몇 가지를 실습해 보겠습니다.

1.  **문자열 함수**

    *   `UPPER`, `LOWER`: 문자열을 대/소문자로 변환합니다.
    *   `CONTAINS`: 특정 문자열을 포함하는지 확인합니다.
    *   `STARTSWITH`, `ENDSWITH`: 특정 문자열로 시작하거나 끝나는지 확인합니다.

    *   이름에 "Awesome"이라는 단어가 포함된 제품을 찾아봅시다.

        ```sql
        SELECT VALUE c.name
        FROM c
        WHERE CONTAINS(c.name, "Awesome")
        ```

    *   제품 이름을 모두 대문자로 변환하여 반환해 봅시다.

        ```sql
        SELECT VALUE UPPER(c.name)
        FROM c
        WHERE c.price < 20
        ```

2.  **수학 함수**

    *   `ABS`, `ROUND`, `CEILING`, `FLOOR`, `TRUNC` 등 표준적인 수학 함수를 지원합니다.

    *   가격이 20달러 미만인 제품의 이름, 원래 가격, 반올림된 가격을 함께 조회해 봅시다.

        ```sql
        SELECT c.name, c.price, ROUND(c.price) AS RoundedPrice
        FROM c
        WHERE c.price < 20
        ```

3.  **배열 함수**

    배열 함수를 테스트하기 위해 먼저 일부 데이터에 배열을 추가해야 합니다.

    *   **데이터 준비**:
        1.  `SELECT * FROM c WHERE c.price < 20` 쿼리를 실행합니다.
        2.  결과에서 아무 문서나 하나 선택합니다. 문서의 `id`를 복사해 둡니다.
        3.  문서 편집기에서 마지막 속성 뒤에 쉼표(`,`)를 추가하고 다음 `tags` 배열을 추가합니다.
            `"tags": ["new", "sale", "special-offer"]`
        4.  **Save**를 클릭하여 문서를 업데이트합니다. 다른 문서 1~2개에도 다른 태그 배열을 추가해 보세요. (예: `"tags": ["popular", "sale"]`)

    *   `ARRAY_CONTAINS`: 배열이 특정 값을 포함하는지 확인합니다.
    *   `ARRAY_LENGTH`: 배열의 요소 개수를 반환합니다.

    *   'sale' 태그를 포함하는 모든 제품을 찾아봅시다.

        ```sql
        SELECT c.name, c.tags
        FROM c
        WHERE ARRAY_CONTAINS(c.tags, "sale")
        ```

    *   태그가 3개인 제품을 찾아봅시다.

        ```sql
        SELECT c.name, c.tags
        FROM c
        WHERE ARRAY_LENGTH(c.tags) = 3
        ```

4.  **Intra-document `JOIN`**

    `JOIN` 키워드를 사용하여 문서 내의 배열을 각 요소별로 분해하여 쿼리할 수 있습니다.

    *   `tags` 배열을 분해하여 각 제품과 태그를 개별 행으로 표시해 봅시다.

        ```sql
        SELECT p.name, t AS Tag
        FROM products p
        JOIN t IN p.tags
        ```
        **결과**: 각 제품이 가진 태그의 수만큼 여러 행으로 나뉘어 표시됩니다. 예를 들어 한 제품에 태그가 3개 있다면, 그 제품에 대해 3개의 결과 행이 생성됩니다. 이는 배열의 각 요소를 기준으로 필터링하거나 집계할 때 매우 유용합니다.

---

이것으로 Azure Cosmos DB for NoSQL 쿼리 실습을 마칩니다. 여러분은 이제 기본 쿼리부터 고급 함수 활용까지 다양한 쿼리를 작성하고 실행할 수 있는 역량을 갖추게 되었습니다.
