# FastMCP MCP Tool Routing 및 단계적 스키마 조회 구성

## 1. 목적과 기록 범위

다중 upstream FastMCP Gateway에서 도구 검색 공간을 provider 단위로 분리하고, 클라이언트가 필요한 도구의 스키마만 단계적으로 가져오도록 Tool Routing을 보완했다.

이 문서는 다음 두 기존 구성 위에서 수행한 **도구 discovery·schema 조회·호출 경로의 변경과 검증**만 다룬다.

- [다중 MCP Gateway 및 Headless 연결 구성](fastmcp-multi-upstream-headless-gateway.md)
- [OAuth·Secure MCP Tunnel 연결 및 도구 권한 검증](fastmcp-oauth-secure-tunnel-authorization.md)

기존 transport, ProxyProvider 연결, OAuth/OIDC 구조, Secure MCP Tunnel 자체는 이번 작업에서 재설계하지 않았다.

최종 목표는 다음과 같다.

- 클라이언트가 먼저 대상 app/provider를 명시한다.
- BM25는 선택한 provider의 도구만 후보로 사용한다.
- 검색 결과에서 전체 JSON schema를 매번 반환하지 않는다.
- 선택한 도구만 별도 schema 조회 후 호출한다.
- schema 조회와 실제 호출에서도 같은 provider ownership과 OAuth 권한을 다시 확인한다.
- 숨겨진 upstream 도구의 직접 호출로 routing 경계를 우회하지 못하게 한다.
- 한 provider 장애가 다른 provider의 검색·호출에 영향을 주지 않도록 하고, 복구 후 Gateway 재시작 없이 다시 사용할 수 있게 한다.

작업 당시 Gateway 실행환경은 CPython 3.12 계열과 FastMCP 4.0.5였다.

---

## 2. 기존 검색 구조에서 확인한 문제

기존 구성은 여러 ProxyProvider의 도구를 namespace로 구분한 뒤 하나의 BM25 검색 공간에서 검색했다.

~~~text
Provider A tools ─┐
                  ├─ global searchable catalog → BM25 → top N
Provider B tools ─┘
~~~

namespace는 이름 충돌을 방지하지만 **검색 대상을 특정 provider로 제한하는 기능은 아니다.**

실제 검색에서 한 응용프로그램을 의도한 질의에도 다른 provider의 도구가 상위 결과에 섞일 수 있었다. app 이름을 query 문자열에 포함하는 방식도 ranking 힌트일 뿐 hard routing이 아니므로, 관련성이 높은 다른 provider 도구가 결과에 포함될 가능성이 남는다.

또한 FastMCP Tool Search는 discovery surface를 줄이는 기능이지 접근 제어 자체가 아니다. 검색 결과에서 숨긴 원본 도구도 이름을 알고 있으면 기본 transform의 직접 호출 경로로 접근할 수 있으므로, provider 경계를 만들려면 **검색·schema 조회·호출·직접 접근을 같은 정책으로 묶어야 했다.**

---

## 3. 최종 구조

최종 public surface는 응용프로그램별 도구를 직접 고정 노출하지 않고 세 진입점으로 제한했다.

~~~text
search_tools
get_tool_schema
call_tool
~~~

전체 흐름은 다음과 같다.

~~~mermaid
flowchart TD
    Client["MCP Client"] --> Search["search_tools(app, query, detail=brief)"]
    Search --> Auth["권한이 적용된 tool catalog"]
    Auth --> Route["app → provider/namespace hard filter"]
    Route --> Rank["BM25 ranking"]
    Rank --> Brief["compact candidate list"]
    Brief --> Schema["get_tool_schema(app, names)"]
    Schema --> Owner1["provider ownership + 권한 재확인"]
    Owner1 --> Selected["selected schema"]
    Selected --> Call["call_tool(app, name, arguments)"]
    Call --> Owner2["provider ownership + 권한 재확인"]
    Owner2 --> Proxy["ProxyProvider"]
    Proxy --> Upstream["upstream MCP"]
~~~

핵심 원칙은 **provider 선택은 routing이고, BM25는 선택된 provider 안의 ranking**이라는 점이다.

---

## 4. 안정적인 app ID와 namespace

각 upstream에는 클라이언트가 사용할 안정적인 app ID와 Gateway 도구 이름에 사용할 namespace의 대응을 둔다.

개념적으로 다음과 같다.

~~~text
app: application_a
namespace: application_a

app: application_b
namespace: application_b
~~~

실제 provider 이름의 길이에 따라 자동으로 약어를 만드는 규칙은 사용하지 않았다. 이름 길이는 routing 의미와 무관하고, 자동 축약은 식별자 충돌이나 의미 손실을 만들 수 있기 때문이다.

app ID는 명시적으로 정한 안정 식별자로 유지하고, namespace는 기존처럼 도구 이름 충돌 방지와 소속 식별에 사용한다.

현재 구현의 ownership 검사는 등록된 **app ID → namespace 대응과 namespaced tool prefix**를 사용한다. 이 계약은 현재 Gateway 구조에서 검증한 방식이며, 중첩 namespace나 별도의 provider ownership metadata를 쓰는 다른 구현에 그대로 적용되는 보편 규칙으로 취급하지 않는다.

---

## 5. BM25 이전 Provider Hard Routing

검색 순서는 다음과 같다.

~~~text
OAuth / authorization 적용
        ↓
현재 사용자가 볼 수 있는 catalog
        ↓
app ID 검증
        ↓
해당 app의 namespace 도구만 후보로 제한
        ↓
BM25 ranking
        ↓
top N 반환
~~~

중요한 점은 **전체 catalog에서 BM25를 수행한 다음 결과를 provider로 후처리하지 않는 것**이다.

후처리 방식에서는 다른 provider 도구가 top N을 먼저 차지해 대상 provider의 관련 도구가 ranking 단계에서 이미 탈락할 수 있다. 따라서 provider 후보 제한을 BM25 이전에 수행했다.

존재하지 않는 app ID를 요청할 때는 전체 catalog 검색으로 fallback하지 않고 오류로 처리한다.

---

## 6. Progressive Disclosure

기존 검색은 상위 검색 결과의 전체 tool definition을 반환했다. 복잡한 upstream 도구는 description과 input/output schema가 커서, 후보 선택에 필요하지 않은 정보까지 클라이언트 context로 전달됐다.

최종 구성은 검색과 schema 조회를 분리했다.

### 6.1 검색

기본 호출은 다음 형태다.

~~~text
search_tools(app, query, detail=brief)
~~~

detail 수준은 세 단계로 사용했다.

| detail | 반환 목적 |
|---|---|
| `brief` | 후보 선택을 위한 tool name과 압축된 설명 |
| `detailed` | 파라미터 이름·타입·필수 여부 중심의 compact schema |
| `full` | 전체 MCP tool definition |

일반 검색에서는 `brief`를 기본값으로 사용한다.

### 6.2 선택 도구 schema 조회

후보를 고른 뒤 필요한 도구만 조회한다.

~~~text
get_tool_schema(
    app,
    names,
    detail=detailed
)
~~~

복잡한 입력 제약을 compact schema만으로 판단하기 어려운 경우에만 `detail=full`을 사용한다.

### 6.3 호출

최종 실행은 다음 형태다.

~~~text
call_tool(
    app,
    name,
    arguments
)
~~~

따라서 일반적인 discovery와 호출 순서는 다음과 같다.

~~~mermaid
sequenceDiagram
    participant C as MCP Client
    participant G as FastMCP Gateway
    participant U as Upstream MCP

    C->>G: search_tools(app, query)
    G-->>C: brief candidates
    C->>G: get_tool_schema(app, selected names)
    G-->>C: selected tool schema
    C->>G: call_tool(app, name, arguments)
    G->>U: tools/call
    U-->>G: result
    G-->>C: result
~~~

---

## 7. Provider Ownership과 직접 호출 차단

routing 경계는 검색에만 적용하지 않았다.

### 7.1 schema 조회

`get_tool_schema`는 요청한 app과 tool name의 provider 소속을 확인한다. 다른 provider의 namespaced tool을 지정하면 schema를 반환하지 않는다.

### 7.2 call_tool

`call_tool`도 app ID와 tool ownership을 다시 대조한다.

~~~text
app = application_a
name = application_b_some_tool
→ provider mismatch
→ upstream 호출 전 거부
~~~

synthetic routing tool 자체를 `call_tool`로 다시 호출하는 것도 허용하지 않는다.

### 7.3 숨겨진 upstream tool 직접 호출

Tool Search가 활성화된 최종 경로에서는 원본 upstream tool을 초기 목록에서 숨기는 것만으로 끝내지 않고, 직접 이름을 알고 호출하는 경로도 차단했다.

검증된 routed call 내부에서만 해당 provider의 원본 tool lookup이 가능하도록 하여 다음 경로를 강제했다.

~~~text
search / schema discovery
        ↓
call_tool(app, ...)
        ↓
provider ownership validation
        ↓
upstream tool
~~~

---

## 8. OAuth Authorization과 결합

OAuth가 활성화된 Gateway에서는 provider routing보다 먼저 현재 사용자에게 허용된 tool catalog가 결정된다.

~~~text
전체 upstream catalog
        ↓
subject / scope authorization
        ↓
authorization-filtered catalog
        ↓
app/provider hard routing
        ↓
search / schema / call
~~~

따라서 app routing이 OAuth 권한을 대신하지 않으며, 반대로 OAuth 권한이 맞아도 다른 provider로 routing된 tool을 실행할 수 없다.

실제 HTTP authorization fixture에서 다음을 확인했다.

- 읽기 scope만 가진 요청에는 쓰기 tool이 search 결과에 포함되지 않음
- 같은 쓰기 tool의 schema 조회도 거부됨
- `call_tool`을 통한 실행도 거부됨
- 쓰기 scope가 있는 요청은 같은 routed call 경로에서 정상 실행됨

---

## 9. ChatGPT Tool Metadata 갱신

서버의 synthetic tool schema를 다음처럼 변경한 뒤 기존 ChatGPT 세션에는 이전 metadata가 한동안 남아 있었다.

~~~text
기존
search_tools(query)
call_tool(name, arguments)

변경 후
search_tools(app, query, detail)
get_tool_schema(app, names, detail)
call_tool(app, name, arguments)
~~~

서버 자체가 새 schema를 사용하고 있는지는 이전 형식의 호출에서 필수 `app` 인자가 없다는 validation error가 발생하는 것으로 먼저 확인했다.

이후 ChatGPT의 Tool metadata를 새로고침하자 세 public tool의 최신 schema가 다시 discovery되었다.

최종 원격 검증에서는 다음 순서를 실제로 수행했다.

~~~text
search_tools(app=...)
→ get_tool_schema(app=...)
→ call_tool(app=...)
~~~

같은 세션에서 다른 provider tool의 schema를 요청했을 때는 provider mismatch로 거부됐다.

서버 schema 변경과 클라이언트 metadata 반영은 같은 상태가 아니므로, public tool signature를 변경한 뒤에는 최종 MCP 클라이언트에서 tool metadata를 다시 확인해야 한다.

---

## 10. Live 통합 및 검색 품질 검증

provider-aware routing 적용 후 두 upstream을 함께 연결한 live catalog에서 namespaced 도구를 다시 확인했다.

당시 catalog는 총 74개였고, 한 provider에서 26개, 다른 provider에서 48개가 발견됐다. 이 숫자는 해당 버전과 연결 상태의 관측값이며 고정된 합격 기준은 아니다.

최종 public surface는 다음 세 도구다.

~~~text
search_tools
get_tool_schema
call_tool
~~~

실제 검증에서는 다음을 확인했다.

- app A 검색에서 app A namespace 결과만 반환
- app B 검색에서 app B namespace 결과만 반환
- selected schema의 detailed 조회
- 필요한 경우 full schema 조회
- app A로 app B schema 요청 거부
- app A로 app B call 요청 거부
- stdio provider를 통한 실제 응용프로그램 상태 조회 성공
- HTTP provider를 통한 실제 scene·Console 조회 성공
- 원격 OAuth MCP 클라이언트에서 같은 `search → schema → call` 경로 성공
- Blender와 Unity 요청을 동시에 실행해도 provider 실행 경로가 섞이지 않음

### 10.1 Tool Search 품질

검색 품질은 provider별 Golden Query Set으로 별도 측정했다.

~~~text
전체 질의: 84
Top-3: 84 / 84
Top-5: 84 / 84
no-hit: 0

한국어 질의: 42
Top-3: 42 / 42

영어 질의: 42
Top-3: 42 / 42
~~~

한국어 normalization 적용 전에는 한국어 질의의 no-hit가 다수 발생했다. scene, object, modifier, find, validate, compile 등 실제 검증에서 필요했던 동의어를 선택적으로 보강한 뒤 기존 영어 결과의 regression 없이 위 결과를 확인했다.

Golden Query Set과 별도로 만든 16개 holdout 표현도 Top-5 16/16을 기록했다. 이는 해당 유한 질의 집합의 결과이며 일반 자연어 전체에 대한 100% 정확도를 의미하지 않는다.

### 10.2 실제 ChatGPT → Tunnel → Gateway E2E

현재 ChatGPT 연결에서 실제 public tool을 사용해 다음 경로를 검증했다.

~~~text
ChatGPT
→ Secure MCP Tunnel
→ FastMCP Gateway
→ search_tools
→ get_tool_schema
→ call_tool
→ ProxyProvider
→ upstream MCP
→ application result
~~~

확대 검증 구간에서는 public 호출 21건을 관측했다. 정상 요청 19건은 성공했고, 나머지 2건은 의도적으로 수행한 cross-provider 호출과 unknown app 요청이 upstream 실행 전에 거부된 경우였다.

실제 upstream `tools/call`은 7건이었고 Blender와 Unity 양쪽에서 응용프로그램 데이터를 반환했다. 해당 실행 trace에서 cross-provider 실제 실행은 0건이었고 RPC ID 충돌도 관측되지 않았다.

---

## 11. Provider 장애 격리와 무재시작 복구

새 routing 적용 후 장애 격리를 양방향으로 다시 검증했다.

### 11.1 stdio 쪽 bridge 중단

stdio provider가 의존하는 headless 응용프로그램 bridge를 중지했다.

~~~text
stdio application bridge unavailable
        ↓
HTTP provider search 정상
HTTP provider 실제 MCP call 정상
Gateway 유지
~~~

bridge를 다시 실행한 뒤 Gateway를 재시작하지 않고 stdio provider의 검색과 실제 응용프로그램 조회가 복구됐다.

### 11.2 HTTP provider 중단

HTTP 기반 headless 응용프로그램을 종료했고, launcher가 소유한 HTTP MCP server도 함께 정리되는 상태를 만들었다.

~~~text
HTTP MCP / application unavailable
        ↓
stdio provider search 정상
stdio provider 실제 application call 정상
Gateway 유지
~~~

HTTP MCP와 응용프로그램을 다시 기동한 뒤에도 Gateway 재시작 없이 search와 call이 복구됐다.

이 검증 동안 Gateway listener 프로세스는 유지됐다. 따라서 한 provider 또는 bridge의 장애가 다른 provider의 정상 경로를 막지 않았고, 복구 후 Gateway 재기동 없이 다시 discovery·호출할 수 있음을 확인했다.

bridge 장애와 MCP transport 자체의 장애는 같은 시험으로 합산하지 않고 실제 중단 범위에 맞춰 구분했다.

---

## 12. Discovery 전달량과 지연 측정

progressive disclosure의 목적은 search latency 자체를 줄이는 것이 아니라 **후보 선택 단계에서 불필요한 schema 전달을 줄이는 것**이었다.

같은 현재 구현과 같은 live upstream에서 5회 반복 median으로 `detail=full` search와 기본 `brief + 선택 도구 detailed schema`를 비교했다.

이 측정은 **이전 소프트웨어 리비전과 현재 리비전의 성능 비교가 아니다.** 같은 provider-aware 구현에서 discovery 표현 방식을 비교한 결과다.

| 대표 질의 | full search | brief search | selected schema | brief+schema | full payload | brief+schema payload | 감소 |
|---|---:|---:|---:|---:|---:|---:|---:|
| stdio scene/object | 약 896 ms | 약 894 ms | 약 890 ms | 약 1,785 ms | 3,325 B | 1,275 B | 61.7% |
| stdio Python execution | 약 911 ms | 약 901 ms | 약 893 ms | 약 1,794 ms | 7,314 B | 1,418 B | 80.6% |
| HTTP scene/prefab | 약 902 ms | 약 898 ms | 약 898 ms | 약 1,796 ms | 13,417 B | 2,542 B | 81.1% |
| HTTP animation/build | 약 890 ms | 약 895 ms | 약 894 ms | 약 1,789 ms | 16,735 B | 2,402 B | 85.6% |

compact rendering 자체는 search latency를 의미 있게 줄이지 않았다. provider catalog를 얻는 비용이 지배적이었다.

한 도구를 선택하는 표준 workflow는 `search → schema → call`로 public call이 하나 늘어나지만, 실제 전달되는 discovery payload는 대표 질의에서 약 61.7%~85.6% 감소했다. 이 결과를 token 감소율로 직접 환산하지 않았다.

---

## 13. 동시성 이슈와 회귀 검증

### 13.1 stdio transport 공유 충돌

동시 Blender/Unity 요청을 검증하는 과정에서 stdio provider에서 실제 세션 충돌이 재현됐다.

~~~text
one StdioTransport instance
        ↓
multiple ProxyClient sessions
        ↓
live session already in use
        ↓
catalog lookup failure
        ↓
valid tool reported as unknown/unauthorized
~~~

원인은 ProxyProvider의 client factory가 여러 client에 같은 `StdioTransport` 인스턴스를 재사용한 것이었다.

해결은 구조를 바꾸지 않고 **client factory가 호출될 때마다 새 StdioTransport를 생성**하도록 최소 수정했다.

수정 후 다음을 확인했다.

- Blender/Unity 병렬 search와 call 성공
- OAuth read/write/outsider context를 병렬 실행해 권한 context 혼입 없음
- 각 RPC ID 고유
- 한 실행 trace에서 Blender와 Unity 실제 `tools/call` 혼입 0건
- fresh transport 생성 여부를 자동 회귀 테스트로 고정

### 13.2 Catalog state

동적 fixture에서 upstream catalog를 한 도구 집합에서 다른 집합으로 교체했다.

검증 결과는 다음과 같다.

- 새 tool이 다음 search에서 발견됨
- 제거된 tool이 search에서 사라짐
- 제거된 tool의 schema 조회 거부
- 새 tool의 schema 조회 및 call 성공

이 검증도 자동 회귀 테스트에 포함했다.

### 13.3 자동 테스트

최종 전체 pytest 결과는 다음과 같다.

~~~text
38 passed, 2 skipped
~~~

skip은 선택 조건이 충족되지 않은 시험으로 남겼으며 성공으로 합산하지 않았다.

주요 검증 범위는 다음과 같다.

- public surface 3개 고정
- app/provider별 검색 결과 격리
- 기본 brief 결과에 전체 `inputSchema` 미노출
- selected schema의 detailed/full 반환
- unknown app 거부
- cross-provider schema/call 거부
- 숨겨진 upstream tool 직접 호출 차단
- OAuth authorization-filtered catalog 보존
- stdio transport 동시성 회귀
- catalog 변경 후 discovery 갱신
- stdio 및 HTTP upstream 실제 검색·호출
- logging·cancellation·rotation 회귀

---

## 14. 버전 회귀 기준선

현재 동작을 이후 업그레이드와 비교할 수 있도록 실행 버전과 행동 검증을 한 suite로 묶었다.

확인된 기준은 다음과 같다.

| 항목 | 기준 |
|---|---|
| Python | CPython 3.12.14 |
| FastMCP | 4.0.5 |
| Secure MCP Tunnel client | 0.0.15 계열, 실행 binary Git SHA까지 확인 |
| stdio upstream MCP | 1.0.2 및 Git commit 확인 |
| HTTP upstream | package lock의 Git hash와 Unity Editor 6000.6.0f1 확인 |
| live catalog | 74개: provider A 26개, provider B 48개 |
| public surface | `search_tools`, `get_tool_schema`, `call_tool` |
| 검색 품질 | 84/84 Top-3, regression 0 |

회귀 suite는 버전 확인, 전체 pytest, 검색 품질, live integration, catalog 수, public surface 및 두 provider의 실제 call 성공을 한 번에 검사했다.

최종 실행에서는 16개 검사 항목이 모두 통과했다. provider 강제 장애 시험과 실제 ChatGPT E2E처럼 서비스 상태를 의도적으로 흔드는 검증은 자동 suite에서 매번 실행하지 않고 이미 확인된 외부 검증으로 분리했다.

---

## 15. GPT Tool Calling 관찰

현재 ChatGPT 연결에서 자연어 요청이 provider와 tool을 선택하는 방식도 실제로 관찰했다.

먼저 protocol 순서를 명시적으로 통제한 6개 시나리오에서는 모두 기대 흐름을 완료했다. 이후 기대값을 실행 후에만 확인하도록 한 별도 대화 세션 A/B/C에서 동일한 6개 요청을 반복했다.

세 blind multi-turn 실행에서 확인된 사실은 다음과 같다.

- 실행이 필요한 작업 12/12 완료
- 기대 provider/tool 선택 15/15 정확
- 실제 관측 가능한 기대 검색 결과는 모두 Top-1
- 모호한 destructive 요청은 3/3 no-call
- cross-provider 실제 실행 0건
- authorization 문제 0건
- full schema 요청 0건

반면 엄격한 per-case rubric에서는 각 blind run이 2/6이었다. 주요 원인은 `get_tool_schema`를 매번 다시 호출하도록 요구한 기준을 모델이 자주 생략했고, 한 대화 안의 이전 provider/tool context를 다음 요청에서 재사용했기 때문이다.

특히 `현재 씬 상태` 요청은 바로 앞의 Unity Console 요청과 같은 대화에 있었으므로 Unity를 선택한 행동을 **provider의 임의 선택이라고 단정할 수 없다.** 또한 두 provider를 동시에 확인하는 후속 요청에서는 직전 discovery 정보를 재사용한 사례가 있었다.

따라서 이 결과에서는 **task/tool 선택 정확도와 protocol-step 준수를 분리해서 해석**했다. 낮은 strict score만으로 Gateway routing 결함이라고 판단하지 않았고, 이 관찰을 근거로 추가 Gateway 변경도 하지 않았다.

---

## 16. 최종 상태

최종 discovery와 실행 계약은 다음과 같다.

~~~text
MCP Client
    ↓
search_tools(app, query, detail=brief)
    ↓
provider-scoped candidates
    ↓
get_tool_schema(app, selected names)
    ↓
selected schema
    ↓
call_tool(app, name, arguments)
    ↓
provider ownership + authorization
    ↓
ProxyProvider
    ↓
fresh provider client / transport
    ↓
upstream MCP
~~~

완료된 범위는 **provider-aware discovery, 단계적 schema 조회, provider 소속 검증, OAuth 권한 보존, 직접 호출 우회 차단, 실제 upstream 호출, 장애 격리·복구, 동시성 수정, catalog 갱신, 검색 품질 측정, 버전 회귀 기준선 및 실제 GPT Tool Calling 관찰**이다.

이 구조에서 namespace는 이름 충돌 방지와 소속 식별을 담당하고, app ID는 routing을 담당하며, BM25는 이미 선택된 provider 내부에서만 ranking을 수행한다.
