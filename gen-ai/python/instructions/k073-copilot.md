# 07.3 - Python과 Azure Cosmos DB for NoSQL로 코파일럿 빌드하기

Python의 다재다능한 프로그래밍 능력과 Azure Cosmos DB의 확장 가능한 NoSQL 데이터베이스 및 벡터 검색 기능을 활용하여, 복잡한 워크플로를 간소화하는 강력하고 효율적인 AI 코파일럿(copilot)을 만들 수 있습니다.

이 실습에서는 Python과 Azure Cosmos DB for NoSQL을 사용하여 코파일럿을 빌드합니다. Azure 서비스(Azure OpenAI 및 Azure Cosmos DB)와의 상호 작용에 필요한 엔드포인트를 제공하는 백엔드 API와, 코파일럿과의 사용자 상호 작용을 용이하게 하는 프론트엔드 UI를 생성합니다. 이 코파일럿은 Cosmic Works 사용자가 자전거 관련 제품을 관리하고 찾는 데 도움을 주는 어시스턴트 역할을 합니다. 구체적으로, 코파일럿은 사용자가 제품 카테고리에 할인을 적용 및 제거하고, 사용 가능한 제품 유형을 알리기 위해 제품 카테고리를 조회하며, 벡터 검색을 사용하여 제품에 대한 유사도 검색을 수행할 수 있도록 합니다.

![Streamlit을 사용하여 Python으로 개발된 UI, Python으로 작성된 백엔드 API, 그리고 Azure Cosmos DB 및 Azure OpenAI와의 상호 작용을 보여주는 고수준 코파일럿 아키텍처 다이어그램](media/07-copilot-high-level-architecture-diagram.png)

> **배경지식: 프론트엔드와 백엔드 분리 아키텍처**
>
> 이 실습에서는 UI(프론트엔드)와 비즈니스 로직(백엔드 API)을 분리하는 현대적인 웹 애플리케이션 아키텍처를 채택합니다.
>
> *   **프론트엔드 (UI)**: **Streamlit**을 사용하여 만듭니다. 사용자가 직접 보고 상호작용하는 부분으로, 사용자 입력을 받아 백엔드 API로 전송하고 그 결과를 화면에 표시하는 역할만 담당합니다.
> *   **백엔드 (API)**: **FastAPI**를 사용하여 만듭니다. 실제 작업이 이루어지는 부분으로, Azure Cosmos DB에서 데이터를 조회/수정하고, Azure OpenAI 모델을 호출하는 등의 모든 비즈니스 로직과 외부 서비스 연동을 처리합니다.
>
> **분리의 장점:**
> 1.  **모듈성 및 유지보수성**: UI와 백엔드를 독립적으로 개발하고 업데이트할 수 있습니다. 예를 들어, UI 디자인을 변경해도 백엔드 로직에는 영향을 주지 않습니다.
> 2.  **확장성**: 사용자가 많아지면 UI 서버와 백엔드 API 서버를 각각 독립적으로 확장(scale-out)하여 부하를 분산시킬 수 있습니다.
> 3.  **보안**: 민감한 정보(예: 데이터베이스 연결 정보, API 키)를 백엔드에만 보관하고, UI에는 노출하지 않아 보안을 강화할 수 있습니다.
> 4.  **기술 유연성**: 프론트엔드와 백엔드를 서로 다른 기술 스택으로 개발할 수도 있습니다.

> ⚠️ 이 모듈의 이전 실습들은 이 실습의 전제 조건입니다. 아직 완료하지 않은 실습이 있다면, 계속하기 전에 먼저 완료해 주십시오. 이전 실습들은 이 실습에 필요한 인프라와 스타터 코드를 제공합니다.

## 백엔드 API 구축

코파일럿을 위한 백엔드 API는 복잡한 데이터를 처리하고, 실시간 통찰력을 제공하며, 다양한 서비스와 원활하게 연결하는 능력을 강화하여 상호 작용을 더욱 동적이고 유익하게 만듭니다. 코파일럿용 API를 구축하기 위해 FastAPI Python 라이브러리를 사용합니다.

> ⚠️ 백엔드 API는 이전 실습에서 `python/07-build-copilot/api/app` 폴더의 `main.py` 파일에 추가한 코드를 기반으로 합니다. 아직 이전 실습을 완료하지 않았다면, 계속하기 전에 완료해 주십시오.

1.  Visual Studio Code에서 **Build copilots with Azure Cosmos DB** 학습 모듈의 실습 코드 리포지토리를 복제한 폴더를 엽니다.

2.  **Explorer** 창에서 **python/07-build-copilot/api/app** 폴더로 이동하여 그 안에 있는 `main.py` 파일을 엽니다.

3.  `main.py` 파일 상단의 기존 `import` 문 아래에 다음 코드를 추가하여 FastAPI를 사용한 비동기 작업을 수행하는 데 사용할 라이브러리를 가져옵니다.

    ```python
    from contextlib import asynccontextmanager
    from fastapi import FastAPI
    import json
    ```

4.  생성할 `/chat` 엔드포인트가 요청 본문(request body)으로 데이터를 받을 수 있도록, 프로젝트의 *models* 모듈에 정의된 `CompletionRequest` 객체를 통해 콘텐츠를 전달할 것입니다. 파일 상단의 `from models import Product` import 문을 업데이트하여 `models` 모듈의 `CompletionRequest` 클래스를 포함시킵니다. import 문은 이제 다음과 같아야 합니다.

    ```python
    from models import Product, CompletionRequest
    ```

5.  Azure OpenAI 서비스에서 생성한 채팅 완료 모델의 배포 이름이 필요합니다. 이를 제공하기 위해 Azure OpenAI 구성 변수 블록의 맨 아래에 변수를 생성합니다.

    ```python
    COMPLETION_DEPLOYMENT_NAME = 'gpt-4o'
    ```
    채팅 완료 배포 이름이 다른 경우, 변수에 할당된 값을 적절히 업데이트하십시오.

6.  Azure Cosmos DB 및 Identity SDK는 해당 서비스와 비동기적으로 작업하기 위한 메서드를 제공합니다. 이 클래스들은 API의 여러 함수에서 사용될 것이므로, 각각의 전역 인스턴스를 생성하여 동일한 클라이언트를 여러 메서드에서 공유할 수 있도록 합니다. Cosmos DB 구성 변수 블록 아래에 다음 전역 변수 선언을 삽입합니다.

    ```python
    # Create a global async Cosmos DB client
    cosmos_client = None
    # Create a global async Microsoft Entra ID RBAC credential
    credential = None
    ```

7.  파일에서 다음 코드를 삭제합니다. 이 기능은 다음 단계에서 정의할 `lifespan` 함수로 이동될 것입니다.

    ```python
    # Enable Microsoft Entra ID RBAC authentication
    credential = DefaultAzureCredential()
    ```

8.  `CosmosClient`와 `DefaultAzureCredential` 클래스의 싱글톤(singleton) 인스턴스를 생성하기 위해, FastAPI의 `lifespan` 객체를 활용합니다. 이 메서드는 API 앱의 생명주기 동안 해당 클래스들을 관리합니다. `lifespan`을 정의하기 위해 다음 코드를 삽입합니다.

    ```python
    @asynccontextmanager
    async def lifespan(app: FastAPI):
        global cosmos_client
        global credential
        # Create an async Microsoft Entra ID RBAC credential
        credential = DefaultAzureCredential()
        # Create an async Cosmos DB client using Microsoft Entra ID RBAC authentication
        cosmos_client = CosmosClient(url=AZURE_COSMOSDB_ENDPOINT, credential=credential)
        yield
        await cosmos_client.close()
        await credential.close()
    ```
    > **Note: FastAPI의 `lifespan` 이벤트**
    > `lifespan`은 FastAPI 애플리케이션의 시작과 종료 시점에 실행되는 특별한 작업입니다.
    > *   **시작 시**: 애플리케이션이 요청을 처리하기 전에, `yield` 이전의 코드가 실행됩니다. 여기서는 `CosmosClient`와 `DefaultAzureCredential` 객체를 한 번만 생성하여 전역 변수에 할당합니다. 이렇게 하면 모든 API 요청이 이 객체들을 재사용하여 효율성을 높입니다 (싱글톤 패턴).
    > *   **종료 시**: 애플리케이션이 종료될 때, `yield` 이후의 코드가 실행됩니다. 여기서는 사용된 리소스(클라이언트 및 자격 증명 세션)를 안전하게 닫습니다.

9.  다음 코드를 사용하여 FastAPI 클래스의 인스턴스를 생성합니다. 이 코드는 `lifespan` 함수 아래에 삽입되어야 합니다.

    ```python
    app = FastAPI(lifespan=lifespan)
    ```

10. 다음으로, API의 엔드포인트를 명시합니다. `api_status` 메서드는 API의 루트 URL에 연결되어 API가 올바르게 실행 중임을 보여주는 상태 메시지 역할을 합니다. `/chat` 엔드포인트는 이 연습의 뒷부분에서 구축할 것입니다.

    ```python
    @app.get("/")
    async def api_status():
        """Display a status message for the API"""
        return {"status": "ready"}
    
    @app.post('/chat')
    async def generate_chat_completion(request: CompletionRequest):
        """Generate a chat completion using the Azure OpenAI API."""
        raise NotImplementedError("The chat endpoint is not implemented yet.")
    ```

11. 파일 맨 아래의 main guard 블록을 덮어써서, 파일이 명령줄에서 실행될 때 `uvicorn` ASGI(Asynchronous Server Gateway Interface) 웹 서버를 시작하도록 합니다.

    ```python
    if __name__ == "__main__":
        import uvicorn
        uvicorn.run(app, host="0.0.0.0", port=8000)
    ```

12. `main.py` 파일을 저장합니다. 이전 연습에서 추가한 `generate_embeddings` 및 `upsert_product` 메서드를 포함하여 전체 파일이 구성됩니다.
13. API를 빠르게 테스트하기 위해 Visual Studio Code에서 새 통합 터미널 창을 열고 `az login`으로 로그인한 후 가상 환경을 활성화합니다.
14. 터미널 프롬프트에서 `api/app`으로 디렉토리를 변경한 다음, 다음 명령을 실행하여 FastAPI 웹 앱을 실행합니다.

    ```bash
    uvicorn main:app
    ```
15. 웹 브라우저에서 <http://127.0.0.1:8000>로 이동합니다. 브라우저 창에 `{"status":"ready"}` 메시지가 표시되면 API가 실행 중인 것입니다.
16. URL 끝에 `/docs`를 추가하여 API의 Swagger UI로 이동합니다: <http://127.0.0.1:8000/docs>.
    > 📝 Swagger UI는 OpenAPI 사양에서 생성된 API 엔드포인트를 탐색하고 테스트하기 위한 대화형 웹 기반 인터페이스입니다.
17. Visual Studio Code로 돌아가 관련 통합 터미널 창에서 **CTRL+C**를 눌러 API 앱을 중지합니다.

## Azure Cosmos DB의 제품 데이터 통합

Azure Cosmos DB의 데이터를 활용하여 코파일럿은 복잡한 워크플로를 간소화하고 사용자가 효율적으로 작업을 완료하도록 도울 수 있습니다. 이제 코파일럿이 제품 카테고리에 할인을 적용할 수 있도록 하는 함수를 추가합니다.

1.  코파일럿은 `apply_discount`라는 비동기 함수를 사용하여 지정된 카테고리의 제품에 할인을 추가하고 제거합니다. `main.py` 파일의 맨 아래 근처, `upsert_product` 함수 아래에 다음 함수 코드를 삽입합니다.

    ```python
    async def apply_discount(discount: float, product_category: str) -> str:
        """Apply a discount to products in the specified category."""
        # Load the CosmicWorks database
        database = cosmos_client.get_database_client(DATABASE_NAME)
        # Retrieve the product container
        container = database.get_container_client(CONTAINER_NAME)
    
        query_results = container.query_items(
            query = """
            SELECT * FROM Products p WHERE CONTAINS(LOWER(p.category_name), LOWER(@product_category))
            """,
            parameters = [
                {"name": "@product_category", "value": product_category}
            ]
        )
    
        # Apply the discount to the products
        async for item in query_results:
            item['discount'] = discount
            item['sale_price'] = item['price'] * (1 - discount) if discount > 0 else item['price']
            await container.upsert_item(item)
    
        return f"A {discount*100}% discount was successfully applied to {product_category}." if discount > 0 else f"Discounts on {product_category} removed successfully."
    ```

2.  다음으로, `get_category_names`라는 두 번째 함수를 추가합니다. 코파일럿은 이 함수를 호출하여 제품에 할인을 적용하거나 제거할 때 사용 가능한 제품 카테고리가 무엇인지 파악합니다.

    ```python
    async def get_category_names() -> list:
        """Retrieve the names of all product categories."""
        # Load the CosmicWorks database
        database = cosmos_client.get_database_client(DATABASE_NAME)
        # Retrieve the product container
        container = database.get_container_client(CONTAINER_NAME)
        # Get distinct product categories
        query_results = container.query_items(
            query = "SELECT DISTINCT VALUE p.category_name FROM Products p"
        )
        categories = []
        async for category in query_results:
            categories.append(category)
        return list(categories)
    ```

3.  `main.py` 파일을 저장합니다.

## 채팅 엔드포인트 구현

백엔드 API의 `/chat` 엔드포인트는 프론트엔드 UI가 Azure OpenAI 모델 및 내부 Cosmic Works 제품 데이터와 상호 작용하는 인터페이스 역할을 합니다.

1.  `main.py` 파일에서 이전에 추가한 `/chat` 엔드포인트 스텁을 찾습니다.
2.  `raise NotImplementedError(...)` 줄을 삭제하고, 이제 엔드포인트 구현을 시작합니다.
3.  먼저 **시스템 프롬프트(system prompt)**를 제공합니다. 이 프롬프트는 코파일럿의 "페르소나"를 정의하고, 사용자와 상호 작용하는 방식, 질문에 응답하는 방식, 사용 가능한 함수를 활용하여 작업을 수행하는 방법을 지시합니다.

    ```python
    # Define the system prompt that contains the assistant's persona.
    system_prompt = """
    You are an intelligent copilot for Cosmic Works designed to help users manage and find bicycle-related products.
    You are helpful, friendly, and knowledgeable, but can only answer questions about Cosmic Works products.
    If asked to apply a discount:
        - Apply the specified discount to all products in the specified category. If the user did not provide you with a discount percentage and a product category, prompt them for the details you need to apply a discount.
        - Discount amounts should be specified as a decimal value (e.g., 0.1 for 10% off).
    If asked to remove discounts from a category:
        - Remove any discounts applied to products in the specified category by setting the discount value to 0.
    """
    ```

4.  다음으로, 시스템 프롬프트, 채팅 기록의 메시지, 그리고 들어오는 사용자 메시지를 추가하여 LLM에 보낼 메시지 배열을 만듭니다.

    ```python
    # Provide the copilot with a persona using the system prompt.
    messages = [{"role": "system", "content": system_prompt }]
    
    # Add the chat history to the messages list
    for message in request.chat_history[-request.max_history:]:
        messages.append(message)
    
    # Add the current user message to the messages list
    messages.append({"role": "user", "content": request.message})
    ```

5.  코파일럿이 Azure Cosmos DB의 데이터와 상호 작용하기 위해 정의한 함수를 사용할 수 있도록, **"도구(tools)"** 모음을 정의해야 합니다. LLM은 실행의 일부로 이러한 도구를 호출합니다. `apply_discount`와 `get_category_names` 메서드에 대한 함수 정의를 포함하는 다음 코드를 제공하여 `tools` 배열을 만듭니다.

    ```python
    # Define function calling tools
    tools = [
        # ... apply_discount 및 get_category_names에 대한 JSON 정의 ...
    ]
    ```
    > **Note: 함수 호출 (Function Calling)**
    > 함수 호출은 LLM이 단순히 텍스트를 생성하는 것을 넘어, 외부 시스템(이 경우 우리 API의 Python 함수)과 상호 작용할 수 있게 해주는 강력한 기능입니다.
    > 1.  사용자 요청과 함께 우리가 정의한 `tools` 목록을 LLM에 전달합니다.
    > 2.  LLM은 사용자 요청을 이해하고, 이를 처리하기 위해 어떤 `tool`(함수)을 어떤 인자(`arguments`)로 호출해야 할지 결정하여 응답합니다.
    > 3.  우리 코드는 LLM의 응답을 받아 실제로 해당 Python 함수를 실행합니다.
    > 4.  함수 실행 결과를 다시 LLM에 전달하여, 최종적으로 사용자에게 보여줄 자연어 응답을 생성하도록 합니다.

6.  채팅 완료 모델에 요청하기 위한 비동기 Azure OpenAI 클라이언트를 생성합니다.

7.  채팅 엔드포인트는 함수 호출을 활용하기 위해 Azure OpenAI에 두 번의 호출을 합니다. 첫 번째는 Azure OpenAI 클라이언트에 도구에 대한 접근을 제공합니다.

8.  이 첫 번째 호출의 응답에는 LLM이 요청에 응답하는 데 필요하다고 판단한 도구나 함수에 대한 정보가 포함됩니다. 함수 호출 출력을 처리하는 코드를 포함해야 합니다.

9.  Azure Cosmos DB에서 보강된 데이터로 요청을 완료하려면, 완료(completion)를 생성하기 위해 Azure OpenAI에 두 번째 요청을 보내야 합니다.

10. 마지막으로, 완료 응답을 UI에 반환합니다.

11. `main.py` 파일을 저장합니다.

## 간단한 채팅 UI 빌드

Streamlit UI는 사용자가 코파일럿과 상호 작용할 수 있는 인터페이스를 제공합니다.

1.  UI는 `python/07-build-copilot/ui` 폴더에 있는 `index.py` 파일을 사용하여 정의됩니다.
2.  `index.py` 파일을 열고 시작하기 위해 파일 상단에 import 문을 추가합니다.
3.  `index.py` 파일 내에 정의된 Streamlit 페이지를 구성합니다.
4.  UI는 `requests` 라이브러리를 사용하여 API에 정의한 `/chat` 엔드포인트를 호출함으로써 백엔드 API와 상호 작용합니다.
5.  애플리케이션 호출의 진입점인 `main` 함수를 정의합니다.
6.  마지막으로, 파일 끝에 main guard 블록을 추가합니다.
7.  `index.py` 파일을 저장합니다.

## UI를 통해 코파일럿 테스트

1.  API 앱을 시작하기 위해 Visual Studio Code에서 해당 앱의 열려 있는 통합 터미널 창으로 돌아가 다음을 입력합니다.

    ```bash
    uvicorn main:app
    ```

2.  새 통합 터미널 창을 열고 `python/07-build-copilot`으로 디렉토리를 변경하여 Python 환경을 활성화한 다음, `ui` 폴더로 디렉토리를 변경하고 다음을 실행하여 UI 앱을 시작합니다.

    ```bash
    python -m streamlit run index.py
    ```

3.  UI가 브라우저 창에서 자동으로 열리지 않으면, 새 브라우저 탭이나 창을 열고 <http://localhost:8501>로 이동하여 UI를 엽니다.

4.  UI의 채팅 프롬프트에 "Apply discount"를 입력하고 메시지를 보냅니다. 코파일럿은 할인율과 제품 카테고리와 같은 추가 정보를 요청하는 응답을 해야 합니다.

5.  사용 가능한 카테고리를 이해하기 위해, 코파일럿에게 제품 카테고리 목록을 제공하도록 요청합니다.

6.  다음으로, 코파일럿에게 모든 의류 제품에 15% 할인을 적용하도록 요청합니다.

7.  Azure portal에서 Azure Cosmos DB 계정을 열고, **Data Explorer**를 선택한 다음, `Products` 컨테이너에 대해 "clothing" 카테고리의 모든 제품을 보는 쿼리를 실행하여 가격 할인이 적용되었는지 확인할 수 있습니다.

8.  Visual Studio Code로 돌아가 해당 앱을 실행 중인 터미널 창에서 **CTRL+C**를 눌러 API 앱을 중지합니다. UI는 계속 실행해도 됩니다.

## 벡터 검색 통합

지금까지 코파일럿에 제품 할인을 적용하는 작업을 수행할 수 있는 능력을 부여했지만, 아직 데이터베이스에 저장된 제품에 대한 지식은 없습니다. 이 작업에서는 특정 품질을 가진 제품을 요청하고 데이터베이스 내에서 유사한 제품을 찾을 수 있도록 하는 벡터 검색 기능을 추가합니다.

1.  `api/app` 폴더의 `main.py` 파일로 돌아가 Azure Cosmos DB 계정의 `Products` 컨테이너에 대해 벡터 검색을 수행하는 메서드를 제공합니다.

    ```python
    async def vector_search(query_embedding: list, num_results: int = 3, similarity_score: float = 0.25):
        """Search for similar product vectors in Azure Cosmos DB"""
        # ... (Azure Cosmos DB에 대한 벡터 검색 쿼리 실행) ...
        query = """
        SELECT TOP @num_results p.name, p.description, ..., VectorDistance(p.embedding, @query_embedding) AS similarity_score
        FROM Products p
        WHERE VectorDistance(p.embedding, @query_embedding) > @similarity_score
        ORDER BY VectorDistance(p.embedding, @query_embedding)
        """
        # ...
    ```
    > **Note: `VectorDistance` 함수**
    > `VectorDistance`는 Azure Cosmos DB for NoSQL에 내장된 함수로, 쿼리 시 두 벡터 간의 유사도를 계산합니다.
    > *   `VectorDistance(p.embedding, @query_embedding)`: 저장된 각 제품의 `embedding` 벡터와 사용자의 쿼리로부터 생성된 `@query_embedding` 벡터 사이의 거리를 계산합니다.
    > *   `WHERE ... > @similarity_score`: 계산된 유사도 점수가 특정 임계값보다 높은 항목만 필터링합니다. (코사인 유사도의 경우 1에 가까울수록 유사함)
    > *   `ORDER BY ...`: 유사도 점수를 기준으로 결과를 정렬하여 가장 유사한 항목을 먼저 반환합니다.

2.  다음으로, `get_similar_products`라는 메서드를 생성합니다. 이 메서드는 LLM이 데이터베이스에 대해 벡터 검색을 수행하는 데 사용할 함수 역할을 합니다. 이 함수는 들어오는 사용자 메시지에 대해 임베딩을 생성하고, 데이터베이스에 저장된 벡터와 비교하기 위해 `vector_search` 함수를 호출합니다.

3.  LLM이 새 함수를 사용할 수 있도록, 이전에 생성한 `tools` 배열을 업데이트하여 `get_similar_products` 메서드에 대한 함수 정의를 추가해야 합니다.

4.  또한, 새 함수의 출력을 처리하는 코드를 추가해야 합니다. 함수 호출 출력을 처리하는 코드 블록에 `elif` 조건을 추가합니다.

5.  마지막으로, 벡터 검색을 수행하는 방법에 대한 지침을 제공하기 위해 시스템 프롬프트 정의를 업데이트해야 합니다.

6.  `main.py` 파일을 저장합니다.

## 벡터 검색 기능 테스트

1.  Visual Studio Code에서 해당 앱의 열려 있는 통합 터미널 창에서 다음을 실행하여 API 앱을 다시 시작합니다.

    ```bash
    uvicorn main:app
    ```

2.  UI를 실행 중인 브라우저 창으로 돌아가 채팅 프롬프트에 다음을 입력합니다.

    ```
    Tell me about the mountain bikes in stock
    ```
    이 질문은 검색과 일치하는 몇 가지 제품을 반환합니다.

3.  "Show me durable pedals," "Provide a list of 5 stylish jerseys," "Give me details about all gloves suitable for warm weather riding"과 같은 다른 검색을 시도해 보십시오. 마지막 두 쿼리의 경우, 제품에 이전에 적용한 15% 할인 및 판매 가격이 포함되어 있음을 관찰하십시오.
