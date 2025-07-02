# Azure Monitor를 사용하여 Azure Cosmos DB for NoSQL 계정 분석하기

Azure Monitor는 Azure 리소스를 모니터링하기 위한 완벽한 기능 세트를 제공하는 Azure의 풀스택(full stack) 모니터링 서비스입니다. Azure Cosmos DB는 Azure Monitor를 사용하여 모니터링 데이터를 생성합니다. Azure Monitor는 Cosmos DB의 메트릭(metrics)과 원격 분석 데이터(telemetry data)를 수집합니다.

이 랩에서는 Azure Cosmos DB 컨테이너에 대해 시뮬레이션된 워크로드(workload)를 실행하고, 이 워크로드가 Azure Cosmos DB 계정에 미치는 영향을 분석해 봅니다.

> **[배경지식: Azure Monitor와 Cosmos DB]**
> Azure Monitor는 모든 Azure 서비스의 성능 및 상태 데이터를 수집, 분석, 대응하는 중앙 허브 역할을 합니다. Cosmos DB의 경우, Azure Monitor는 두 가지 주요 유형의 데이터를 수집합니다.
> 1.  **메트릭 (Metrics)**: 특정 시점의 숫자 값 데이터입니다. 예를 들어, 초당 요청 단위(Request Units per second), 요청 수, 서버 지연 시간 등이 있습니다. 이 데이터는 가볍고 실시간 경고에 적합합니다.
> 2.  **진단 로그 (Diagnostic Logs)**: 더 풍부하고 상세한 이벤트 데이터입니다. 예를 들어, 특정 쿼리의 전문(full text), 요청 요금(Request Charge), 세션 정보 등이 포함됩니다. Log Analytics를 사용하여 이 데이터를 복잡하게 쿼리하고 분석할 수 있습니다.
>
> 이 랩에서는 주로 **메트릭(Metrics)**과, 메트릭을 기반으로 사전 구성된 대시보드인 **인사이트(Insights)**를 사용하여 성능을 분석하는 방법에 초점을 맞춥니다.

## 개발 환경 준비

이 랩을 진행하는 환경에 **DP-420**에 대한 랩 코드 리포지토리를 아직 복제하지 않았다면, 다음 단계에 따라 복제하세요. 그렇지 않다면, 이전에 복제한 폴더를 **Visual Studio Code**에서 엽니다.

1.  **Visual Studio Code**를 시작합니다.

    > &#128221; Visual Studio Code 인터페이스에 익숙하지 않다면, [Visual Studio Code 시작 가이드][code.visualstudio.com/docs/getstarted]를 검토하세요.

2.  명령 팔레트(command palette)를 열고 **Git: Clone**을 실행하여 `https://github.com/microsoftlearning/dp-420-cosmos-db-dev` GitHub 리포지토리를 원하는 로컬 폴더에 복제합니다.

    > &#128161; **CTRL+SHIFT+P** 키보드 단축키를 사용하여 명령 팔레트를 열 수 있습니다.

3.  리포지토리가 복제되면, **Visual Studio Code**에서 선택한 로컬 폴더를 엽니다.

## Azure Cosmos DB for NoSQL 계정 생성하기

Azure Cosmos DB는 여러 API를 지원하는 클라우드 기반 NoSQL 데이터베이스 서비스입니다. 처음으로 Azure Cosmos DB 계정을 프로비저닝할 때, 계정이 지원할 API(예: **Mongo API** 또는 **NoSQL API**)를 선택하게 됩니다. Azure Cosmos DB for NoSQL 계정 프로비저닝이 완료되면, 엔드포인트(endpoint)와 키(key)를 검색할 수 있습니다. 엔드포인트와 키를 사용하여 프로그래밍 방식으로 Azure Cosmos DB for NoSQL 계정에 연결합니다. 이 정보는 Azure SDK for .NET이나 다른 SDK의 연결 문자열에 사용됩니다.

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
| **Limit the total amount of throughput that can be provisioned on this account** | *Uncheck* |

> **[Note]** "Limit the total amount of throughput..." 옵션을 선택 해제하는 이유는 이 랩의 스크립트가 여러 컨테이너를 생성하고 높은 처리량(throughput)을 할당하기 때문입니다. 이 옵션이 선택되어 있으면 계정 전체에 프로비저닝할 수 있는 총 RU/s가 제한되어 랩 진행에 문제가 발생할 수 있습니다.

> &#128221; 랩 환경에서는 새 리소스 그룹 생성이 제한될 수 있습니다. 이 경우, 미리 생성된 기존 리소스 그룹을 사용하세요.

4.  이 작업을 계속하기 전에 배포 작업이 완료될 때까지 기다립니다.

5.  새로 생성된 **Azure Cosmos DB** 계정 리소스로 이동하여 **Keys** 창으로 이동합니다.

6.  이 창에는 SDK에서 계정에 연결하는 데 필요한 연결 정보와 자격 증명이 포함되어 있습니다. 특히 다음을 확인하세요:
    1.  **URI** 필드를 확인합니다. 이 **endpoint** 값은 이 연습의 뒷부분에서 사용됩니다.
    2.  **PRIMARY KEY** 필드를 확인합니다. 이 **key** 값은 이 연습의 뒷부분에서 사용됩니다.

7.  브라우저 창을 최소화하되 닫지는 마십시오. 다음 단계에서 백그라운드 워크로드를 시작한 후 몇 분 뒤에 Azure portal로 돌아올 것입니다.

## .NET 스크립트에 Microsoft.Azure.Cosmos 및 Newtonsoft.Json 라이브러리 가져오기

.NET CLI는 사전 구성된 패키지 피드에서 패키지를 가져오는 [add package][docs.microsoft.com/dotnet/core/tools/dotnet-add-package] 명령을 포함합니다. .NET 설치는 NuGet을 기본 패키지 피드로 사용합니다.

1.  **Visual Studio Code**의 **탐색기(Explorer)** 창에서 **25-monitor** 폴더로 이동합니다.

2.  **25-monitor** 폴더의 컨텍스트 메뉴를 열고 **통합 터미널에서 열기(Open in Integrated Terminal)**를 선택하여 새 터미널 인스턴스를 엽니다.

    > &#128221; 이 명령은 시작 디렉터리가 이미 **25-monitor** 폴더로 설정된 터미널을 엽니다.

3.  다음 명령을 사용하여 NuGet에서 [Microsoft.Azure.Cosmos][nuget.org/packages/microsoft.azure.cosmos/3.49.0] 패키지를 추가합니다:

    ```
    dotnet add package Microsoft.Azure.Cosmos --version 3.49.0
    ```

4.  다음 명령을 사용하여 NuGet에서 [Newtonsoft.Json][nuget.org/packages/Newtonsoft.Json/13.0.3] 패키지를 추가합니다:

    ```
    dotnet add package Newtonsoft.Json --version 13.0.3
    ```

## 스크립트를 실행하여 컨테이너와 워크로드 생성하기

이제 워크로드를 실행하여 Azure Cosmos DB 계정 사용량을 모니터링할 준비가 되었습니다. 실행할 스크립트는 백그라운드에서 세 개의 컨테이너를 생성하고 일부 데이터를 해당 컨테이너에 로드합니다. 그런 다음 스크립트는 여러 사용자 애플리케이션이 Azure Cosmos DB 계정에 접근하는 것을 시뮬레이션하기 위해 일부 SQL 쿼리를 무작위로 실행합니다.

1.  **Visual Studio Code**의 **탐색기** 창에서 **25-monitor** 폴더로 이동합니다.

2.  **Program.cs** 코드 파일을 엽니다.

3.  **endpoint**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **endpoint** 값으로 업데이트합니다.
  
    ```csharp
    private static readonly string endpoint = "<cosmos-endpoint>";
    ```

    > &#128221; 예를 들어, 엔드포인트가 **https&shy;://dp420.documents.azure.com:443/** 이라면, C# 문은 **private static readonly string endpoint = "https&shy;://dp420.documents.azure.com:443/";** 가 됩니다.

4.  **key**라는 기존 변수를 이전에 생성한 Azure Cosmos DB 계정의 **key** 값으로 업데이트합니다.

    ```csharp
    private static readonly string key = "<cosmos-key>";
    ```

    > &#128221; 예를 들어, 키가 **fDR2ci9QgkdkvERTQ==** 라면, C# 문은 **private static readonly string key = "fDR2ci9QgkdkvERTQ==";** 가 됩니다.

5.  **Program.cs** 파일을 저장합니다.

6.  *통합 터미널*로 돌아갑니다.

7.  [dotnet run][docs.microsoft.com/dotnet/core/tools/dotnet-run] 명령을 사용하여 프로젝트를 빌드하고 실행합니다:

    ```
    dotnet run
    ```
    > &#128221; 이 스크립트의 첫 부분은 세 개의 컨테이너를 생성하고 데이터를 로드하며, 약 2분 정도 소요됩니다. 일부 속도 제한(rate limiting) 이벤트를 시뮬레이션하기 위해, 스크립트는 프로비저닝된 처리량을 400 RU/s로 설정합니다. 그러면 "***Creating simulated background workload, wait 5-10 minutes and go to the next step of the exercise.***"라는 메시지가 표시됩니다. Azure 리소스는 모니터링 데이터를 Azure Monitor에 비동기적으로 업로드하므로, Azure Monitor Metrics 및 Insights에서 진단 데이터를 받기 시작하려면 잠시 기다려야 합니다. 5-10분 후에 다음 단계로 이동하세요. 추가 진단 데이터를 수집하고 싶다면, 5-10분 후에 스크립트를 중지할 필요 없이 랩이 끝날 때까지 기다렸다가 중지해도 됩니다.

    > &#128221; 컴파일러가 스크립트가 많은 작업을 동기적으로 실행하고 작업의 응답을 기다리지 않는 것을 감지하기 때문에 노란색 경고가 몇 개 표시될 것입니다. 여러 SQL 스크립트를 동시에 실행하는 것이 예상된 동작이므로 이 경고는 무시해도 됩니다.

## Azure Monitor를 사용하여 Azure Cosmos DB 계정 사용량 분석하기

이 연습의 이 부분에서는 브라우저로 돌아가 일부 Azure Monitor Insight 및 Metric 보고서를 검토합니다.

### Azure Monitor Metrics 보고서

1.  앞서 최소화했던 열린 브라우저 창으로 돌아갑니다. 닫았다면 새 창을 열고 portal.azure.com에서 Azure Cosmos DB 계정 페이지로 이동합니다.

2.  Azure Cosmos DB 왼쪽 메뉴의 *Monitoring* 아래에서 **Metrics**를 선택합니다. **Scope**와 **Metric Namespace** 필드가 올바른 정보로 미리 채워져 있는 것을 확인할 수 있습니다. 다음 단계에서는 몇 가지 **Metric** 옵션과 *Add filter* 및 *Apply splitting* 기능을 살펴보겠습니다.

3.  기본적으로 *Metrics* 섹션은 지난 24시간 동안의 진단 정보를 보여줍니다. 이전 단계에서 생성한 워크로드 동안의 메트릭을 살펴보기 위해 더 세분화해야 합니다. 오른쪽 상단 모서리에서 ***Local time: Last 24 hours (Automatic)***라고 표시된 버튼을 선택하면 여러 시간 범위 옵션이 있는 창이 나타납니다. ***Last 30 minutes*** 라디오 버튼을 선택하고 **Apply** 버튼을 선택합니다. 필요한 경우 *Custom* 라디오 버튼을 선택하고 시작 및 종료 날짜와 시간을 선택하여 훨씬 더 세분화할 수 있습니다.

4.  이제 진단 차트에 적합한 시간 범위를 설정했으므로 몇 가지 메트릭을 살펴보겠습니다. 일반적인 메트릭부터 시작하겠습니다. *Metric* 드롭다운에서 **Total Request Units**를 선택합니다. 기본적으로 이 메트릭은 RU의 총합(Sum)으로 표시됩니다. 또는 Aggregation 드롭다운을 avg 또는 max로 변경할 수 있습니다. 두 집계를 확인한 후, 다음 단계를 위해 다시 *Sum*으로 설정합니다.

5.  이 메트릭은 Azure Cosmos DB 계정에서 사용된 요청 단위의 양에 대한 좋은 아이디어를 제공합니다. 그러나 현재 차트는 계정에 여러 데이터베이스나 컨테이너가 있을 때 문제를 파악하는 데 도움이 되지 않을 수 있습니다. 이를 변경해 보겠습니다. RU 소비가 데이터베이스별로 어떻게 이루어졌는지 검토해 보겠습니다. 차트 제목 아래 메뉴에서 **Apply splitting**을 선택하고, **Values** 드롭다운에서 **DatabaseName**을 선택한 다음, 차트의 아무 곳이나 클릭하여 변경 사항을 적용합니다. **Split by = DatabaseName** 버튼이 이제 차트 바로 위에 위치하게 됩니다.

> **[Note: Splitting (분할)]**
> 분할 기능은 메트릭을 특정 **차원(dimension)**별로 세분화하여 보여주는 강력한 기능입니다. `Total Request Units`라는 단일 값을 `DatabaseName` 차원으로 분할하면, 어떤 데이터베이스가 RU를 많이 소비하는지 한눈에 파악할 수 있습니다.

6.  훨씬 낫습니다. 이제 어떤 데이터베이스가 대부분의 작업을 수행하는지 알 수 있습니다. 이 정보는 좋지만, 어떤 컨테이너가 모든 작업을 수행하는지는 알 수 없습니다. **Split by = DatabaseName** 버튼을 선택하여 분할 조건을 변경하고 *Values* 드롭다운에서 **CollectionName**을 선택합니다. 좋습니다. 이제 **customer** 및 **salesOrder** 컬렉션에 대한 데이터가 있어야 합니다. 이 차트에는 한 가지 문제가 있습니다. **salesOrder** 컬렉션은 **database-v2**와 **database-v3** 두 데이터베이스에 모두 존재하므로, 이 값은 두 데이터베이스에 있는 해당 컬렉션 이름의 집계입니다.

7.  간단히 수정할 수 있습니다. **Add filter** 버튼을 선택하고, *Property* 드롭다운에서 **DatabaseName**을 선택한 다음, *Values*에서 **database-V3**를 선택합니다.

8.  두 가지 메트릭을 더 살펴보겠습니다. 기존 차트를 편집하거나 원한다면 새 차트를 만들 수 있습니다. 차트 위에서 *Azure Cosmos DB 계정 이름*과 **Total Request Unit** 레이블이 있는 버튼을 선택합니다. *Metric* 드롭다운에서 **Total Requests**를 선택합니다. 사용 가능한 집계는 *Count*뿐임을 확인하세요.

9.  여기서 두 가지 주요 필터가 다양한 유형의 문제를 해결하는 데 도움이 될 수 있습니다. Property가 **StatusCode**인 필터를 추가하고(다른 유형의 세부 정보를 가진 유사한 필터는 **Status**임), *Values*로 **200**과 **429**를 선택합니다. 분할(Split)을 StatusCode를 사용하도록 변경합니다. 상태 200(성공적인 요청)에 비해 엄청난 양의 429(속도 제한 요청)가 있음을 주목하세요. 429 예외는 스크립트가 초당 수천 개의 요청을 보내는 동안 프로비저닝된 처리량을 400 RU/s로 설정했기 때문에 발생했습니다. *성공적인 요청에 비해 이렇게 많은 429 예외가 발생하는 것은 프로덕션 환경에서 정상이 아니어야 합니다. 프로덕션 환경에서 건강한 Azure Cosmos DB 계정에서는 429 예외가 드물게 발생해야 합니다*. 유사한 문제 해결 방식으로 **Total Request Units**에 대해 **StatusCode** 또는 **Status** *Properties*를 사용할 수도 있습니다.

> **[Note: StatusCode 429]**
> `HTTP 429 Too Many Requests`는 클라이언트가 할당된 처리량(RU/s)보다 더 많은 요청을 보내고 있음을 나타내는 상태 코드입니다. 이것은 **스로틀링(throttling)** 또는 **속도 제한(rate limiting)**이라고도 합니다. 이 랩에서는 의도적으로 이 상황을 만들어 모니터링하는 방법을 배우지만, 실제 운영 환경에서는 429 에러가 지속적으로 발생한다면 RU를 증설하거나 쿼리/데이터 모델을 최적화해야 한다는 강력한 신호입니다.

10. **Total Request**를 계속 살펴보되, 분할을 **OperationType**으로 변경해 봅시다. 이 속성은 어떤 읽기 또는 쓰기 작업이 작업의 대부분을 차지하는지 결정하는 데 도움이 됩니다. 다시 말하지만, 이 속성은 **Total Request Units**에 대해서도 유사하게 사용될 수 있습니다.

11. **Total Request Units**에서 했던 것처럼, 다른 필터와 분할 옵션을 선택하며 실험해 보세요.

12. 이 연습에서 마지막으로 살펴볼 메트릭은 **Normalized RU Consumption** 메트릭입니다. 분할을 **PartitionKeyRangeId**로 변경합니다. 이 메트릭은 어떤 파티션 키 범위의 사용량이 더 높은지(warmer) 식별하는 데 도움이 됩니다. 이 메트릭은 파티션 키 범위에 대한 처리량의 편중(skew)을 보여줍니다. *Metric* 드롭다운에서 해당 메트릭을 선택하세요. 이 차트는 이제 지속적으로 100% **Normalized RU Consumption**에 도달하는 매우 비정상적인 시스템을 보여줄 것입니다.

> **[Note: Normalized RU Consumption과 Hot Partition]**
> `Normalized RU Consumption`은 단일 물리적 파티션(PartitionKeyRangeId)이 프로비저닝된 처리량을 얼마나 소모하고 있는지를 0-100% 사이의 값으로 나타내는 가장 중요한 메트릭 중 하나입니다. 특정 파티션의 이 값이 지속적으로 100%에 가깝다면, 이는 **핫 파티션(hot partition)**이 발생했음을 의미합니다. 즉, 전체 처리량은 여유가 있더라도 특정 파티션에 요청이 집중되어 해당 파티션에서만 스로틀링(429 에러)이 발생할 수 있습니다. 이는 파티션 키 설계가 잘못되었을 가능성을 시사합니다.

> &#128221; 한 번에 여러 차트를 보고 싶다면, 차트 이름 위의 **+ New Chart** 옵션을 클릭하세요.

> &#128221; 메트릭을 직접 저장할 수는 없지만, 기존 대시보드를 사용하거나 새로 만들어서 차트의 오른쪽 상단 모서리에 있는 **Pin to dashboard** 버튼을 클릭하여 이 차트를 추가할 수 있습니다. 버튼을 클릭하고 **Create new** 탭을 선택한 다음, *DP-420 labs*라는 이름을 지정하고 **Create and pin**을 클릭합니다. 개인 대시보드를 보려면 왼쪽 상단 모서리의 포털 메뉴로 이동하여 Azure 리소스 옵션에서 Dashboard를 선택합니다. 대시보드가 처음 나타나는 데 몇 분이 걸릴 수 있습니다.

> &#128221; 차트를 공유하는 또 다른 방법은 Share 드롭다운을 클릭하여 Excel 파일로 다운로드하거나 Copy link 옵션을 사용하는 것입니다.

### Azure Monitor Insights 보고서

Azure Monitor Metrics 진단 보고서를 미세 조정하는 데 시간을 할애해야 할 수 있습니다. Cosmos DB Insights는 Azure Cosmos DB 리소스의 전반적인 성능, 실패, 운영 상태에 대한 뷰를 제공합니다. 이러한 Insight 차트는 Metric 차트와 유사한 사전 빌드된 차트입니다. 그 중 몇 가지를 살펴보겠습니다.

1.  Azure Cosmos DB 왼쪽 메뉴의 *Monitoring* 아래에서 **Insights**를 선택합니다. Overview부터 Management Options까지 여러 탭이 있음을 확인할 수 있습니다. 이 **Insight** 차트 중 몇 가지를 살펴보겠습니다. 첫 번째 탭인 Overview 탭은 가장 일반적으로 사용할 수 있는 차트의 요약을 제공합니다. 예를 들어, 총 요청 수, 데이터 및 인덱스 사용량, 429 예외 및 정규화된 RU 소비량과 같은 차트가 있습니다. 이전 섹션에서 이러한 차트의 대부분을 보았습니다.

2.  차트 상단에서 **Time Range**를 지정할 수 있으므로, 이 연습의 워크로드를 평가하기 위해 *15* 또는 *30*분을 선택합니다.

3.  *각* 차트의 오른쪽 상단 모서리에는 ***Open Metric Explorer*** 옵션이 있습니다. **Total Requests** 차트에 대해 **Open Metric Explorer** 옵션을 선택해 봅시다. 이 옵션을 선택하면 이전에 검토했던 Metric 보고서로 이동하는 것을 확인할 수 있습니다. Metric Explorer를 여는 장점은 차트의 상당 부분이 이미 우리를 위해 만들어져 있다는 것입니다.

4.  Metric 차트의 오른쪽 상단에 있는 **X**를 선택하여 Insights 페이지로 돌아갑니다.

5.  Throughput 탭을 선택합니다. 이 차트들은 처리량 문제를 파악하는 데 좋습니다. 특히 **Normalized RU Consumption (%) By PartitionKeyRangeID** 차트에 주목하세요. 이 차트는 핫 파티션을 감지하는 데 사용될 수 있습니다.

6.  Requests 탭을 선택합니다. 이 차트들은 계정이 경험한 제한 이벤트의 수(429 vs. 200)를 분석하거나 작업 유형별 요청 수를 검토하는 데 좋습니다.

7.  Storage 탭을 선택합니다. 이 차트들은 컬렉션의 성장과 데이터 및 인덱스 사용량을 보여줍니다.

8.  System 탭을 선택합니다. 애플리케이션이 계정의 메타데이터를 자주 생성, 삭제 또는 쿼리하는 경우 429 예외가 발생할 수 있습니다. 이 차트들은 빈번한 메타데이터 접근이 429 예외의 원인인지 판단하는 데 도움이 됩니다. 또한 메타데이터 요청의 상태를 확인할 수 있습니다.

### Azure Monitor Insights 보고서 (오타 수정: 작업 정리)

1.  Program이 아직 실행 중이라면, Visual Studio Code 명령 터미널로 돌아갑니다.

2.  통합 터미널을 닫습니다.

3.  **Visual Studio Code**를 닫습니다.
