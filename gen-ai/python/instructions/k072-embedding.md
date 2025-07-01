

# 07.2 - Azure OpenAI로 벡터 임베딩을 생성하고 Azure Cosmos DB for NoSQL에 저장하기

Azure OpenAI는 `text-embedding-ada-002`, `text-embedding-3-small`, `text-embedding-3-large` 모델을 포함한 OpenAI의 고급 언어 모델에 대한 접근을 제공합니다. 이러한 모델 중 하나를 활용하여 텍스트 데이터의 벡터 표현(vector representation)을 생성하고, 이를 Azure Cosmos DB for NoSQL과 같은 벡터 저장소(vector store)에 저장할 수 있습니다. 이는 효율적이고 정확한 유사도 검색을 용이하게 하여, 코파일럿(copilot)이 관련 정보를 검색하고 문맥적으로 풍부한 상호작용을 제공하는 능력을 크게 향상시킵니다.

이 실습에서는 Azure OpenAI 서비스를 생성하고 임베딩 모델을 배포합니다. 그런 다음, Python 코드를 사용하여 각각의 Python SDK로 Azure OpenAI 및 Cosmos DB 클라이언트를 생성하고, 제품 설명의 벡터 표현을 생성하여 데이터베이스에 기록합니다.

> ⚠️ 이 모듈의 이전 실습은 이 실습의 전제 조건입니다. 아직 해당 실습을 완료하지 않았다면, 계속하기 전에 먼저 완료해 주십시오. 이전 실습은 이 실습에 필요한 인프라를 제공합니다.

> **배경지식: 엔드투엔드 벡터화 프로세스**
>
> 이 실습은 생성형 AI 애플리케이션, 특히 검색 증강 생성(Retrieval-Augmented Generation, RAG) 패턴의 핵심 단계를 다룹니다. 전체적인 흐름은 다음과 같습니다.
>
> 1.  **데이터 준비**: 제품 설명과 같은 원본 텍스트 데이터를 준비합니다.
> 2.  **임베딩 생성 (Vectorization)**: Azure OpenAI의 임베딩 모델(`text-embedding-3-small`)을 호출하여 각 텍스트 조각을 숫자 배열, 즉 **벡터(vector)**로 변환합니다. 이 벡터는 텍스트의 "의미"를 담고 있습니다.
> 3.  **데이터 저장**: 원본 데이터와 함께 생성된 벡터를 Azure Cosmos DB for NoSQL 컨테이너에 저장합니다. 이제 Cosmos DB는 **벡터 저장소(vector store)** 역할을 하게 됩니다.
>
> 이 과정을 통해 데이터베이스는 단순한 텍스트가 아닌, 의미적으로 검색할 수 있는 상태가 됩니다. 다음 실습에서는 이 저장된 벡터를 활용하여 의미 기반 검색을 수행하게 됩니다.

## Azure OpenAI 서비스 생성

Azure OpenAI는 OpenAI의 강력한 언어 모델에 대한 REST API 접근을 제공합니다. 이러한 모델은 콘텐츠 생성, 요약, 이미지 이해, 의미 검색, 자연어-코드 변환 등 특정 작업에 쉽게 적용될 수 있습니다.

1.  새 웹 브라우저 창이나 탭에서 Azure portal (``portal.azure.com``)로 이동합니다.

2.  구독과 연결된 Microsoft 자격 증명을 사용하여 포털에 로그인합니다.

3.  **Create a resource**를 선택하고, *Azure OpenAI*를 검색한 다음, 다음 설정으로 새로운 **Azure OpenAI** 리소스를 생성합니다. 나머지 설정은 모두 기본값으로 둡니다.

    | Setting | Value |
    | --- | --- |
    | **Subscription** | *기존 Azure 구독* |
    | **Resource group** | *기존 리소스 그룹을 선택하거나 새로 생성* |
    | **Region** | [지원 지역 목록](https://learn.microsoft.com/azure/ai-services/openai/concepts/models?tabs=python-secure%2Cglobal-standard%2Cstandard-embeddings#tabpanel_3_standard-embeddings)에서 `text-embedding-3-small` 모델을 지원하는 사용 가능한 지역을 선택합니다. |
    | **Name** | *전역적으로 고유한 이름 입력* |
    | **Pricing Tier** | *Standard 0 선택* |

    > 📝 실습 환경에서는 새 리소스 그룹 생성을 막는 제약이 있을 수 있습니다. 그럴 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

4.  다음 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

## 임베딩 모델 배포

Azure OpenAI를 사용하여 임베딩을 생성하려면, 먼저 서비스 내에 원하는 임베딩 모델의 인스턴스를 배포해야 합니다.

1.  Azure portal (``portal.azure.com``)에서 새로 생성한 Azure OpenAI 서비스로 이동합니다.

2.  Azure OpenAI 서비스의 **Overview** 페이지에서, 도구 모음의 **Go to Azure AI Foundry portal** 링크를 선택하여 **Azure AI Foundry**를 시작합니다.
    > **Note: Azure AI Foundry**
    > Azure AI Foundry(이전의 Azure AI Studio)는 Azure에서 AI 모델을 빌드, 관리 및 배포하기 위한 통합 포털입니다. 여기에서 모델 배포, 플레이그라운드 테스트, 미세 조정 등의 작업을 수행할 수 있습니다.

3.  Azure AI Foundry의 왼쪽 메뉴에서 **Deployments**를 선택합니다.

4.  **Model deployments** 페이지에서 **Deploy model**을 선택하고 드롭다운에서 **Deploy base model**을 선택합니다.

5.  모델 목록에서 `text-embedding-3-small`을 선택합니다.

    > 💡 추론 작업(inference tasks) 필터를 사용하여 *Embeddings* 모델만 표시하도록 목록을 필터링할 수 있습니다.
    >
    > 📝 `text-embedding-3-small` 모델이 보이지 않으면, 현재 해당 모델을 지원하지 않는 Azure 지역을 선택했을 수 있습니다. 이 경우, 이 실습에서는 `text-embedding-ada-002` 모델을 사용할 수 있습니다. 두 모델 모두 1536차원의 벡터를 생성하므로, Azure Cosmos DB의 `Products` 컨테이너에 정의한 컨테이너 벡터 정책을 변경할 필요가 없습니다.

6.  **Confirm**을 선택하여 모델을 배포합니다.

7.  Azure AI Foundry의 **Model deployments** 페이지에서, `text-embedding-3-small` 모델 배포의 **Name**을 기록해 두십시오. 이 값은 나중에 이 연습에서 필요합니다.

## 채팅 완료 모델 배포

임베딩 모델 외에도, 코파일럿을 위한 채팅 완료 모델이 필요합니다. OpenAI의 `gpt-4o` 대규모 언어 모델을 사용하여 코파일럿의 응답을 생성할 것입니다.

1.  Azure AI Foundry의 **Model deployments** 페이지에서, 다시 **Deploy model** 버튼을 선택하고 드롭다운에서 **Deploy base model**을 선택합니다.

2.  목록에서 **gpt-4o** 채팅 완료 모델을 선택합니다.

3.  **Confirm**을 선택하여 모델을 배포합니다.

4.  Azure AI Foundry의 **Model deployments** 페이지에서, `gpt-4o` 모델 배포의 **Name**을 기록해 두십시오. 이 값은 나중에 이 연습에서 필요합니다.

## Cognitive Services OpenAI User RBAC 역할 할당

사용자 ID가 Azure OpenAI 서비스와 상호 작용할 수 있도록 하려면, 계정에 **Cognitive Services OpenAI User** 역할을 할당할 수 있습니다. Azure OpenAI 서비스는 Azure 리소스에 대한 개별 액세스를 관리하기 위한 인증 시스템인 Azure 역할 기반 액세스 제어(Azure RBAC)를 지원합니다. Azure RBAC를 사용하면 특정 프로젝트의 필요에 따라 다른 팀원에게 다른 수준의 권한을 할당할 수 있습니다.

> **Note: 키 없는 인증 (Keyless Authentication)과 RBAC**
>
> 이 실습에서는 보안 강화를 위해 API 키를 코드에 직접 하드코딩하는 대신, 역할 기반 접근 제어(RBAC)와 Microsoft Entra ID(구 Azure Active Directory)를 사용한 키 없는 인증 방식을 사용합니다.
>
> - **RBAC**: 사용자, 그룹 또는 서비스 주체에게 특정 Azure 리소스에 대한 권한(예: 읽기, 쓰기, 데이터 접근)을 부여하는 체계입니다.
> - **`Cognitive Services OpenAI User` 역할**: 이 역할을 할당받은 ID는 Azure OpenAI 모델을 호출하고 사용할 수 있는 권한을 가집니다.
> - **`DefaultAzureCredential` (Python SDK)**: 코드에서 사용할 클래스로, 환경 변수, 관리 ID, Azure CLI 로그인 정보 등 다양한 방법을 순서대로 시도하여 자동으로 인증 자격 증명을 찾아줍니다. 이를 통해 코드에서 민감한 키를 제거하여 보안을 강화하고 관리를 용이하게 합니다.

1.  Azure portal (``portal.azure.com``)에서 Azure OpenAI 리소스로 이동합니다.

2.  왼쪽 탐색 창에서 **Access Control (IAM)**을 선택합니다.

3.  **Add**를 선택한 다음, **Add role assignment**를 선택합니다.

4.  **Role** 탭에서 **Cognitive Services OpenAI User** 역할을 선택한 다음, **Next**를 선택합니다.

5.  **Members** 탭에서, 사용자, 그룹 또는 서비스 주체에 대한 액세스 할당을 선택하고 **Select members**를 선택합니다.

6.  **Select members** 대화 상자에서, 자신의 이름이나 이메일 주소를 검색하고 자신의 계정을 선택합니다.

7.  **Review + assign** 탭에서, **Review + assign**을 선택하여 역할을 할당합니다.

## Python 가상 환경 생성

Python의 가상 환경은 깨끗하고 조직화된 개발 공간을 유지하는 데 필수적이며, 개별 프로젝트가 다른 프로젝트와 격리된 자체 종속성 집합을 가질 수 있도록 합니다. 이는 다른 프로젝트 간의 충돌을 방지하고 개발 워크플로우의 일관성을 보장합니다.

1.  Visual Studio Code를 사용하여 **Build copilots with Azure Cosmos DB** 학습 모듈의 실습 코드 리포지토리를 복제한 폴더를 엽니다.
2.  Visual Studio Code에서 새 터미널 창을 열고 `python/07-build-copilot` 폴더로 디렉토리를 변경합니다.
3.  터미널 프롬프트에서 다음 명령을 실행하여 `.venv`라는 이름의 가상 환경을 생성합니다.

    ```bash
    python -m venv .venv 
    ```

4.  아래 표에서 OS 및 셸에 맞는 명령을 선택하고 터미널 프롬프트에서 실행하여 가상 환경을 활성화합니다.

    | Platform | Shell | Command to activate virtual environment |
    | --- | --- | --- |
    | POSIX | bash/zsh | `source .venv/bin/activate` |
    | | fish | `source .venv/bin/activate.fish` |
    | | csh/tcsh | `source .venv/bin/activate.csh` |
    | | pwsh | `.venv/bin/Activate.ps1` |
    | Windows | cmd.exe | `.venv\Scripts\activate.bat` |
    | | PowerShell | `.venv\Scripts\Activate.ps1` |

5.  `requirements.txt`에 정의된 라이브러리를 설치합니다.

    ```bash
    pip install -r requirements.txt
    ```
    > **Note: `requirements.txt` 파일**
    > 이 파일에는 현재 프로젝트에 필요한 모든 Python 라이브러리와 그 버전이 명시되어 있습니다. `pip install -r` 명령을 사용하면 이 파일에 나열된 모든 라이브러리를 한 번에 설치할 수 있어, 환경 설정을 간편하고 일관되게 만들어 줍니다.
    
    이 실습에서 사용할 주요 라이브러리는 다음과 같습니다.
    
    | Library | Description |
    | --- | --- |
    | `azure-cosmos` | Azure Cosmos DB for Python SDK |
    | `azure-identity`| 키 없는 인증을 위한 Azure Identity SDK |
    | `openai` | Azure OpenAI REST API에 접근하기 위한 라이브러리 |
    | `fastapi` | API 빌드를 위한 웹 프레임워크 |
    | `streamlit` | 대화형 웹 앱을 만들기 위한 프레임워크 |

## 텍스트 벡터화를 위한 Python 함수 추가

Python용 Azure OpenAI SDK는 텍스트 데이터에 대한 임베딩을 생성하는 데 사용할 수 있는 동기 및 비동기 클래스를 모두 제공합니다. 이 기능은 Python 코드의 함수로 캡슐화할 수 있습니다.

1.  Visual Studio Code의 **Explorer** 창에서 `python/07-build-copilot/api/app` 폴더로 이동하여 그 안에 있는 `main.py` 파일을 엽니다.

2.  Python용 비동기 Azure OpenAI SDK를 사용하기 위해 `main.py` 파일의 맨 위에 다음 코드를 추가하여 라이브러리를 가져옵니다.

    ```python
    from openai import AsyncAzureOpenAI
    ```

3.  Azure 인증 및 이전에 사용자 ID에 할당한 Entra ID RBAC 역할을 사용하여 Azure OpenAI 및 Cosmos DB에 비동기적으로 액세스할 것입니다. `openai` import 문 아래에 다음 줄을 추가하여 `azure-identity` 라이브러리에서 필요한 클래스를 가져옵니다.

    ```python
    from azure.identity.aio import DefaultAzureCredential, get_bearer_token_provider
    ```

4.  Azure OpenAI API 버전과 엔드포인트를 저장할 변수를 생성하고, `<AZURE_OPENAI_ENDPOINT>` 토큰을 Azure OpenAI 서비스의 엔드포인트 값으로 바꿉니다. 또한, 임베딩 모델 배포 이름을 위한 변수도 생성합니다. 파일의 `import` 문 아래에 다음 코드를 삽입합니다.

    ```python
    # Azure OpenAI configuration
    AZURE_OPENAI_ENDPOINT = "<AZURE_OPENAI_ENDPOINT>"
    AZURE_OPENAI_API_VERSION = "2024-05-01-preview" # 최신 버전 확인 가능
    EMBEDDING_DEPLOYMENT_NAME = "text-embedding-3-small"
    ```
    > 📝 `EMBEDDING_DEPLOYMENT_NAME`은 Azure AI Foundry에서 `text-embedding-3-small` 모델을 배포한 후 확인한 **Name** 값입니다.

5.  Python용 Azure Identity SDK의 `DefaultAzureCredential` 클래스를 사용하여 Microsoft Entra ID RBAC 인증을 통해 Azure OpenAI 및 Azure Cosmos DB에 액세스하기 위한 비동기 자격 증명을 생성합니다. 변수 선언 아래에 다음 코드를 삽입합니다.

    ```python
    # Enable Microsoft Entra ID RBAC authentication
    credential = DefaultAzureCredential()
    ```

6.  임베딩 생성을 처리하기 위해, Azure OpenAI 클라이언트를 사용하여 임베딩을 생성하는 함수를 추가하는 다음 코드를 삽입합니다.

    ```python
    async def generate_embeddings(text: str):
        """Generates embeddings for the provided text."""
        # Create an async Azure OpenAI client
        async with AsyncAzureOpenAI(
            api_version = AZURE_OPENAI_API_VERSION,
            azure_endpoint = AZURE_OPENAI_ENDPOINT,
            azure_ad_token_provider = get_bearer_token_provider(credential, "https://cognitiveservices.azure.com/.default")
        ) as client:
            response = await client.embeddings.create(
                input = text,
                model = EMBEDDING_DEPLOYMENT_NAME
            )
            return response.data[0].embedding
    ```
    > **Note: 비동기(Async) 프로그래밍**
    >
    > `async def`, `async with`, `await` 키워드는 Python의 비동기 프로그래밍 구문입니다. Azure OpenAI API 호출과 같은 네트워크 작업은 시간이 걸릴 수 있습니다. 비동기 방식을 사용하면, 하나의 API 호출이 완료되기를 기다리는 동안 프로그램이 다른 작업을 처리할 수 있어 전반적인 효율성과 응답성이 향상됩니다.

7.  `main.py` 파일의 전체 내용은 이제 다음과 유사해야 합니다. (아래 코드는 예시이며 단계별로 코드가 추가됩니다.)
8.  `main.py` 파일을 저장합니다.

## 임베딩 함수 테스트

`main.py` 파일의 `generate_embeddings` 함수가 올바르게 작동하는지 확인하기 위해, 파일 끝에 몇 줄의 코드를 추가하여 직접 실행할 수 있도록 합니다. 이 코드를 통해 명령줄에서 `generate_embeddings` 함수를 실행하고 임베딩할 텍스트를 전달할 수 있습니다.

1.  `main.py` 파일의 맨 아래에 `generate_embeddings`를 호출하는 **main guard** 블록을 추가합니다.

    ```python
    if __name__ == "__main__":
        import asyncio
        import sys
     
        async def main():
            print(await generate_embeddings(sys.argv[1]))
            # Close open async credential sessions
            await credential.close()
         
        asyncio.run(main())
    ```
    > **Note: `if __name__ == "__main__"`**
    > 이 블록은 파이썬에서 "main guard" 또는 "entry point"라고 불립니다. 이 코드는 스크립트가 직접 실행될 때만 실행되고, 다른 스크립트에서 모듈로 가져올 때는 실행되지 않도록 보장합니다. 코드를 정리하고 재사용 가능하게 만드는 좋은 습관입니다.

2.  `main.py` 파일을 저장합니다.
3.  Visual Studio Code에서 새 통합 터미널 창을 엽니다.
4.  Azure OpenAI로 요청을 보내는 API를 실행하기 전에, `az login` 명령을 사용하여 Azure에 로그인해야 합니다. 터미널 창에서 다음을 실행합니다.

    ```bash
    az login
    ```
5.  브라우저에서 로그인 프로세스를 완료합니다.
6.  터미널 프롬프트에서 `python/07-build-copilot`으로 디렉토리를 변경하고, 가상 환경을 활성화합니다.
7.  `api/app`으로 디렉토리를 변경한 후 다음 명령을 실행합니다.

    ```bash
    python main.py "Hello, world!"
    ```
9.  터미널 창의 출력을 확인합니다. "Hello, world!" 문자열의 벡터 표현인 부동 소수점 숫자 배열이 표시되어야 합니다. 다음과 유사한 출력이 나타납니다.

    ```bash
    [-0.019184619188308716, -0.025279032066464424, -0.0017195191467180848, 0.01884828321635723...]
    ```

## Azure Cosmos DB에 데이터를 쓰기 위한 함수 빌드

Python용 Azure Cosmos DB SDK를 사용하여 데이터베이스에 문서를 업서트(upsert)하는 함수를 생성할 수 있습니다. 업서트 작업은 일치하는 레코드를 찾으면 업데이트하고, 없으면 새 레코드를 삽입합니다.

1.  Visual Studio Code에서 열려 있는 `main.py` 파일로 돌아가, 파일의 기존 `import` 문 바로 아래에 다음 줄을 삽입하여 Python용 Azure Cosmos DB SDK에서 비동기 `CosmosClient` 클래스를 가져옵니다.

    ```python
    from azure.cosmos.aio import CosmosClient
    ```

2.  `api/app` 폴더의 *models* 모듈에서 `Product` 클래스를 참조하는 또 다른 import 문을 추가합니다. `Product` 클래스는 Cosmic Works 데이터셋의 제품 형태를 정의합니다.

    ```python
    from models import Product
    ```

3.  이전에 삽입한 Azure OpenAI 변수 아래에 Azure Cosmos DB와 관련된 구성 값을 포함하는 새로운 변수 그룹을 `main.py` 파일에 추가합니다. `<AZURE_COSMOSDB_ENDPOINT>` 토큰을 Azure Cosmos DB 계정의 엔드포인트로 바꾸십시오.

    ```python
    # Azure Cosmos DB configuration
    AZURE_COSMOSDB_ENDPOINT = "<AZURE_COSMOSDB_ENDPOINT>"
    DATABASE_NAME = "CosmicWorks"
    CONTAINER_NAME = "Products"
    ```

4.  `main.py` 파일의 `generate_embeddings` 함수 아래에 다음 코드를 삽입하여 Cosmos DB에 문서를 업서트(업데이트 또는 삽입)하는 `upsert_product` 함수를 추가합니다.

    ```python
    async def upsert_product(product: Product):
        """Upserts the provided product to the Cosmos DB container."""
        # Create an async Cosmos DB client
        async with CosmosClient(url=AZURE_COSMOSDB_ENDPOINT, credential=credential) as client:
            # Load the CosmicWorks database
            database = client.get_database_client(DATABASE_NAME)
            # Retrieve the product container
            container = database.get_container_client(CONTAINER_NAME)
            # Upsert the product
            await container.upsert_item(product)
    ```
    > **Note: `upsert_item` 함수**
    > `upsert`는 "update or insert"의 줄임말입니다. `id`를 기준으로 문서가 이미 존재하면 내용을 업데이트하고, 존재하지 않으면 새로 삽입합니다. 이 함수 덕분에 스크립트를 여러 번 실행해도 중복 오류가 발생하지 않고 데이터가 최신 상태로 유지됩니다.

5.  `main.py` 파일을 저장합니다.

## 샘플 데이터 벡터화

이제 `generate_embeddings`와 `upsert_product` 함수를 함께 테스트할 준비가 되었습니다. 이를 위해, `if __name__ == "__main__"` main guard 블록을 GitHub에서 Cosmic Works 제품 정보가 포함된 샘플 데이터 파일을 다운로드하고, 각 제품의 `description` 필드를 벡터화한 다음, Azure Cosmos DB 데이터베이스의 `Products` 컨테이너에 문서를 업서트하는 코드로 덮어쓸 것입니다.

> 📝 이 접근 방식은 Azure OpenAI로 생성하고 Azure Cosmos DB에 임베딩을 저장하는 기술을 시연하기 위해 사용됩니다. 그러나 실제 시나리오에서는 Azure Cosmos DB 변경 피드에 의해 트리거되는 Azure Function을 사용하는 것과 같은 더 견고한 접근 방식이 기존 및 새 문서에 임베딩을 추가하는 데 더 적합합니다.

1.  `main.py` 파일에서 `if __name__ == "__main__":` 코드 블록을 다음으로 덮어씁니다.

    ```python
    if __name__ == "__main__":
        import asyncio
        from models import Product
        import requests
     
        async def main():
            product_raw_data = "https://raw.githubusercontent.com/solliancenet/microsoft-learning-path-build-copilots-with-cosmos-db-labs/refs/heads/main/data/07/products.json?v=1"
            products = [Product(**data) for data in requests.get(product_raw_data).json()]
     
            # Call the generate_embeddings function for each product description.
            for product in products:
                print(f"Generating embeddings for product: {product.name}", end="\r")
                product.embedding = await generate_embeddings(product.description)
                await upsert_product(product.model_dump())
     
            print("\nAll products with vectorized descriptions have been upserted to the Cosmos DB container.")
            # Close open credential sessions
            await credential.close()
     
        asyncio.run(main())
    ```

2.  Visual Studio Code의 열려 있는 통합 터미널 프롬프트에서 다음 명령을 사용하여 `main.py` 파일을 다시 실행합니다.

    ```bash
    python main.py
    ```

3.  코드 실행이 완료될 때까지 기다립니다. 모든 제품에 벡터화된 설명이 추가되어 Cosmos DB 컨테이너에 업서트되었다는 메시지가 표시됩니다. 제품 데이터셋의 295개 레코드에 대한 벡터화 및 데이터 업서트 프로세스를 완료하는 데 최대 10~15분이 소요될 수 있습니다. 모든 제품이 삽입되지 않은 경우, 위 명령을 사용하여 `main.py`를 다시 실행하여 나머지 제품을 추가할 수 있습니다.

## Cosmos DB에서 업서트된 샘플 데이터 검토

1.  Azure portal (``portal.azure.com``)로 돌아가 Azure Cosmos DB 계정으로 이동합니다.
2.  왼쪽 탐색 메뉴에서 **Data Explorer**를 선택합니다.
3.  **CosmicWorks** 데이터베이스와 **Products** 컨테이너를 확장한 다음, 컨테이너 아래의 **Items**를 선택합니다.
4.  컨테이너 내의 여러 문서를 무작위로 선택하고 `embedding` 필드가 벡터 배열로 채워져 있는지 확인합니다.
