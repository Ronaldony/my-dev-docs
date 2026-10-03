# FastMCP Tool Routing 후속 Reliability 검증 및 회귀 기준선

## 1. 목적과 기록 범위

`fastmcp-mcp-tool-routing.md`에서 확정한 provider-aware Tool Routing 구성을 대상으로, 실제 운영 환경에서의 안정성·복구성·동시성·회귀 특성을 확인하는 **후속 reliability 작업**을 수행했다. OAuth·Secure MCP Tunnel은 이 routing 경로의 실제 인증·재시작·세션 지속성을 검증하기 위한 운영 조건으로 함께 확인했다.

이 문서는 기존 Routing 설계를 확장하거나 다시 설계하는 문서가 아니다. 다음 기존 문서를 변경하거나 대체하지 않고, 특히 `fastmcp-mcp-tool-routing.md` 작업 이후 수행한 **reliability 검증과 실제 안정성 이슈 수정**만 별도로 기록한다.

- [FastMCP MCP Tool Routing 및 단계적 스키마 조회 구성](fastmcp-mcp-tool-routing.md)
- [FastMCP OAuth·Secure MCP Tunnel 연결 및 도구 권한 검증](fastmcp-oauth-secure-tunnel-authorization.md)

이번 작업에서 새로 확인한 범위는 다음과 같다.

- provider별 Tool Search 품질
- 대표 읽기 작업의 실제 실행
- OAuth state 영속성과 Gateway/Tunnel 재시작
- refresh 실패·rotation 경계
- 병렬 요청과 context 분리
- dynamic catalog 변경
- 실제 ChatGPT → Tunnel → Gateway E2E
- 버전·행동 회귀 기준선
- GPT Tool Calling 반복 관찰

기존 transport, OAuth 설계, ProxyProvider 구조를 다시 설계하지 않았으며, 확인된 문제에 대해서만 최소 수정했다.

---

## 2. 검증 대상 구조

후속 검증 시점의 public surface는 다음 세 도구였다.

~~~text
search_tools
get_tool_schema
call_tool
~~~

실제 요청 경로는 다음과 같다.

~~~mermaid
flowchart TD
    Client["ChatGPT / MCP Client"]
    Tunnel["Secure MCP Tunnel"]
    OAuth["OAuth / subject / scope"]
    Search["search_tools(app, query)"]
    Filter["provider hard filter"]
    Rank["BM25"]
    Schema["get_tool_schema(app, names)"]
    Call["call_tool(app, name, arguments)"]
    Proxy["ProxyProvider"]
    A["stdio upstream"]
    B["HTTP upstream"]
    AppA["Application A"]
    AppB["Application B"]

    Client --> Tunnel
    Tunnel --> OAuth
    OAuth --> Search
    Search --> Filter
    Filter --> Rank
    Rank --> Schema
    Schema --> Call
    Call --> Proxy
    Proxy --> A
    Proxy --> B
    A --> AppA
    B --> AppB
~~~

검증에서는 검색 결과만 보는 것으로 끝내지 않고, 가능한 경우 실제 응용프로그램 데이터까지 반환되는지 확인했다.

---

## 3. Tool Search 품질 검증

provider 후보 제한 이후 BM25 검색 품질을 한국어·영어 질의로 다시 측정했다.

### 3.1 Golden Query Set

총 84개 질의를 사용했다.

정규화 적용 전 결과:

~~~text
전체 84
Top-1 37
Top-3 48
Top-5 48
No-hit 36

한국어 42
Top-1 3
Top-3 6
Top-5 6
No-hit 36

영어 42
Top-1 34
Top-3 42
Top-5 42
No-hit 0
~~~

한국어 질의의 scene, object, modifier, find, validate, compile 등 실제 사용 표현에 대해 선택적 normalization과 alias를 적용했다.

적용 후 결과:

~~~text
전체 84
Top-1 69
Top-3 84
Top-5 84
No-hit 0

한국어 42
Top-1 35
Top-3 42
Top-5 42
No-hit 0

영어 42
Top-1 34
Top-3 42
Top-5 42
No-hit 0
~~~

기존 영어 결과의 순위 하락은 관측되지 않았다.

### 3.2 Holdout 표현

Golden Query Set에 포함하지 않은 16개 표현으로 별도 확인했다.

초기 결과는 Top-5 12/16이었다. 실패한 표현은 scene, modifier, find, compile·validate 계열의 자연어 변형이었다.

필요한 동의어를 최소 범위로 보강한 뒤 같은 holdout에서 Top-5 16/16을 확인했다.

이 수치는 해당 유한 질의 집합의 결과이며 일반 자연어 전체에 대한 100% 정확도를 의미하지 않는다.

---

## 4. 대표 실행 검증

live catalog에서 확인된 도구 수는 총 74개였다.

~~~text
Provider A: 26
Provider B: 48
Total: 74
~~~

대표 읽기 작업은 반드시 다음 흐름으로 확인했다.

~~~text
search
→ selected schema
→ call
→ actual upstream
→ application result
~~~

검증한 대표 범위는 다음과 같다.

### stdio provider

- scene/object summary
- 현재 파일 경로·저장 상태
- 누락된 외부 파일 확인
- 사용자 매뉴얼 검색
- Python API 문서 조회

### HTTP provider

- active scene 조회
- GameObject 검색
- Console error/warning 조회

대표 8개 실제 실행은 모두 search, schema, call 단계에서 오류 없이 완료됐다.

HTTP 기반 headless 응용프로그램에서는 active scene이 비어 있고 project 내부 pipeline 상태 파일 접근 거부 로그가 함께 확인됐다. 이 현상은 Gateway routing 실패와 분리해 응용프로그램 fixture/runtime 문제로 취급했다.

---

## 5. OAuth lifecycle 및 재시작 검증

### 5.1 실행 설정 정합성

OAuth preflight는 설정이 현재 프로세스 환경에 주입되지 않은 셸에서 실행하면 필수 값이 없는 것으로 판정됐다.

실제 launcher와 동일하게 관리 대상 환경을 주입한 상태에서는 다음 항목이 모두 준비 상태였다.

- OAuth public base URL
- resource base URL
- OIDC discovery URL
- client ID
- token audience
- subject allowlist
- resource forwarding
- client secret 존재
- JWT signing key 존재
- OAuth route
- CIMD

따라서 preflight 실패와 운영 설정 실패를 동일하게 취급하지 않았다.

### 5.2 OAuth state 영속성

FastMCP 4.0.5의 OAuth proxy 기본 저장 방식을 확인했다.

별도 `client_storage`가 없을 때 signing key에서 파생한 key로 암호화된 파일 저장소를 사용한다.

~~~text
stable signing key
        ↓
derived JWT key
        ↓
derived storage encryption key
        ↓
encrypted OAuth proxy store
~~~

운영 상태에서 확인한 한 시점의 저장 상태는 다음과 같았다.

~~~text
OAuth state files: 52
registered clients: 2
JTI mappings: 19
refresh metadata: 9
upstream token sets: 9
~~~

이 숫자는 고정 설정값이 아니라 당시 운영 상태의 관측값이다.

### 5.3 Gateway/Tunnel 재시작

Gateway를 중단하고 재기동하는 과정에서 처음에는 독립적으로 새 프로세스를 시작하는 방식이 listener 복구에 실패했다.

Tunnel launcher가 Gateway lifecycle까지 소유하는 구조였기 때문에, 이후 공식 launcher 경로를 사용해 stale Tunnel과 Gateway를 함께 정리하고 새 stack을 기동했다.

재기동 후 확인 결과:

- Gateway listener 정상
- Tunnel health HTTP 200
- OAuth metadata HTTP 200
- OAuth store 파일 수 유지
- 등록 client/JTI/refresh/upstream state 유지

### 5.4 실제 ChatGPT 세션 재사용

재시작 뒤 현재 ChatGPT 연결에서 별도 재로그인 없이 다음을 수행했다.

~~~text
search_tools
→ get_tool_schema
→ call_tool
~~~

Blender와 Unity 양쪽에서 실제 읽기 작업이 성공했다.

따라서 이번 검증에서는 stable signing key와 persisted OAuth state가 유지된 경우 Gateway/Tunnel 재시작 후 기존 ChatGPT 인증 세션을 계속 사용할 수 있음을 확인했다.

---

## 6. Refresh 실패와 rotation 경계

실제 사용자 refresh token을 의도적으로 파괴하지 않기 위해 FastMCP OAuthProxy의 실제 구현을 메모리 기반 격리 fixture에서 검증했다.

확인 결과는 다음과 같다.

| 시나리오 | 결과 |
|---|---|
| 존재하지 않는 refresh token | 조회 실패, 재인증 경로 |
| upstream refresh endpoint 오류 | `invalid_grant` |
| refresh 실패 뒤 기존 mapping | 유지 |
| 정상 refresh | 새 access/refresh token 발급 |
| 이전 refresh JTI | 폐기 |
| 이전 refresh metadata | 폐기 |
| 새 refresh token | 조회 가능 |
| 필수 credential 누락 | provider 생성 단계에서 fail-closed |

FastMCP의 access token 검증 경로에서는 upstream token 만료 시각과 threshold를 확인하고, refresh token이 있을 때 transparent refresh를 시도하는 구조도 확인했다.

이 결과는 실제 IdP access token을 강제로 만료시킨 장기 세션 시험과는 구분한다.

---

## 7. 병렬 요청에서 발견한 stdio transport 충돌

### 7.1 이슈 내용

Blender와 Unity 요청을 병렬 실행하는 과정에서 stdio provider에서 세션 충돌이 재현됐다.

오류의 핵심은 하나의 stdio transport가 서로 다른 client session에 동시에 사용되고 있다는 것이었다.

~~~text
one StdioTransport instance
        ↓
multiple ProxyClient sessions
        ↓
live session conflict
        ↓
provider list failure
        ↓
valid tool not found
~~~

### 7.2 발생 원인

ProxyProvider의 client factory는 매 호출마다 새 ProxyClient를 만들었지만, 내부에는 동일한 StdioTransport 인스턴스를 전달하고 있었다.

FastMCP의 stdio transport는 동시에 다른 session option으로 공유하는 구조를 지원하지 않았다.

### 7.3 해결

ProxyClient 생성 시마다 새 StdioTransport를 함께 생성하도록 변경했다.

~~~text
client factory call
        ↓
new ProxyClient
        ↓
new StdioTransport
~~~

ProxyProvider, namespace, logging 구조는 변경하지 않았다.

### 7.4 검증

수정 후 다음을 확인했다.

- Blender/Unity 병렬 검색 성공
- Blender/Unity 병렬 실제 call 성공
- OAuth read/write/outsider fixture 병렬 실행 시 권한 context 혼입 없음
- RPC ID 전부 고유
- 한 실행 trace에서 서로 다른 provider의 실제 `tools/call` 혼입 0건

fresh transport 생성 여부를 자동 회귀 테스트에도 추가했다.

---

## 8. Dynamic catalog state 검증

upstream tool catalog가 실행 중 변경되는 상황을 fixture로 확인했다.

~~~text
catalog A
  └─ alpha

runtime switch

catalog B
  └─ beta
~~~

변경 뒤 다음을 확인했다.

- 새 tool이 search에서 발견됨
- 제거된 tool이 search에서 사라짐
- 제거된 tool의 schema 조회 거부
- 새 tool의 schema 조회 성공
- 새 tool의 call 성공

이 검증도 자동 회귀 테스트에 포함했다.

---

## 9. 실제 ChatGPT E2E 확대 검증

현재 ChatGPT 연결을 사용해 한국어·영어 요청, 두 provider, 읽기형과 보수적으로 write scope로 분류된 도구를 섞어 확인했다.

대표 시나리오:

- Blender 파일 저장 상태와 경로
- Blender scene object 목록
- Blender manual 검색
- Unity active scene
- Unity Console error/warning
- Blender API 문서

동시 요청과 의도적인 오류 요청도 포함했다.

검증 구간의 public 호출은 총 21건이었다.

~~~text
search_tools: 8
get_tool_schema: 5
call_tool: 8
~~~

결과:

- 정상 시나리오 19건 성공
- cross-provider 잘못된 호출 1건 upstream 전 거부
- unknown app 요청 1건 upstream 전 거부
- 실제 upstream `tools/call` 7건
- 실제 cross-provider 실행 0건
- public trace RPC ID 충돌 0건
- 해당 요청의 authorization context는 모두 허용 상태

Tool Search와 schema 조회 과정에서는 authorization-filtered 전체 visible catalog를 만들기 위해 양쪽 provider의 `tools/list`가 발생할 수 있었다.

따라서 cross-provider leak 판단은 `tools/list`가 아니라 **실제 upstream `tools/call`이 잘못된 provider로 나갔는지**를 기준으로 했다.

---

## 10. Version regression 기준선

후속 업그레이드에서 동작 변화를 빠르게 확인할 수 있도록 version과 behavior를 함께 고정했다.

당시 확인한 기준:

| 항목 | 기준 |
|---|---|
| CPython | 3.12.14 |
| FastMCP | 4.0.5 |
| Secure MCP Tunnel client | 0.0.15 계열 |
| stdio upstream MCP | 1.0.2 |
| Unity Editor | 6000.6.0f1 |
| live catalog | 74개 |
| provider별 catalog | 26 / 48 |
| public surface | 3개 routing wrapper |
| Tool Search | 84/84 Top-3 |

Tunnel binary는 version 문자열뿐 아니라 Git SHA까지 확인했고, upstream도 Git commit 또는 package lock hash를 함께 기록해 단순 이름 기반 버전 추정을 피했다.

회귀 suite는 다음을 한 번에 검사했다.

- runtime version
- Tunnel binary version
- upstream version/hash
- 전체 pytest
- Tool Search 품질
- live Blender/Unity integration
- catalog count
- public surface
- 실제 upstream call marker

최종 suite 결과:

~~~text
16 checks
16 passed
0 failed
~~~

provider 강제 장애나 실제 ChatGPT E2E처럼 운영 상태를 의도적으로 흔드는 시험은 매 회귀 실행에 넣지 않고 별도 외부 검증으로 구분했다.

### 10.1 자동 테스트

catalog refresh와 stdio fresh transport 회귀를 추가한 뒤 전체 pytest 결과는 다음과 같았다.

~~~text
38 passed, 2 skipped
~~~

skip은 성공으로 합산하지 않았다.

---

## 11. GPT Tool Calling 반복 관찰

자연어 요청에 대해 ChatGPT가 provider와 tool을 선택하는 방식도 실제 연결에서 관찰했다.

### 11.1 Protocol-controlled baseline

처음 6개 시나리오는 `search → schema → call` 순서를 명시적으로 적용해 확인했다.

~~~text
6 / 6 scenario pass
expected tool Top-1: 5 / 5
full schema: 0
cross-provider leak: 0
authorization denial: 0
~~~

이 실행은 protocol 순서를 통제한 calibration 성격이므로 독립 blind model sample로 취급하지 않았다.

### 11.2 Blind multi-turn 세션 A/B/C

별도 ChatGPT 대화 세션 3개에서 기대값을 실행 뒤에만 확인하도록 같은 6개 요청을 반복했다.

각 세션의 엄격한 rubric 점수는 2/6이었다.

그러나 실패 원인을 분해하면 다음과 같다.

- 실행이 필요한 실제 작업: 12/12 완료
- 기대 provider/tool 선택: 15/15 정확
- 관측 가능한 기대 search 결과: 모두 Top-1
- 모호한 destructive 요청 no-call: 3/3
- cross-provider 실제 실행: 0
- authorization 문제: 0
- full schema 요청: 0

strict score가 낮아진 주된 이유는 다음 두 가지였다.

1. `get_tool_schema`를 매 case에서 반드시 다시 호출해야 한다는 rubric을 모델이 자주 생략했다.
2. 한 대화 안의 앞선 provider/tool context를 다음 case에서 재사용했다.

특히 `현재 씬 상태 확인` 요청은 바로 앞에서 Unity Console을 확인한 같은 대화에 있었다. 따라서 Unity를 선택한 동작을 provider의 근거 없는 임의 선택이라고 단정하지 않았다.

또 두 provider를 함께 확인하는 후속 요청에서는 직전 case에서 이미 발견한 Unity scene tool을 다시 검색하지 않은 사례가 있었다.

따라서 이번 관찰에서는 다음을 분리했다.

~~~text
task / tool selection correctness
≠
strict protocol-step adherence
~~~

낮은 strict rubric 점수만으로 Gateway routing 결함이라고 판단하지 않았고, 이 결과를 근거로 추가 Gateway 변경도 하지 않았다.

---

## 12. 남은 검증 경계

다음 항목은 이번 작업에서 완료로 주장하지 않는다.

| 항목 | 상태 |
|---|---|
| 실제 IdP access token 강제 만료 후 refresh | 실제 사용자 세션에서는 수행하지 않음 |
| 사용자 OAuth authorization 취소 후 복구 | 수행하지 않음 |
| 다른 실제 사용자·축소 scope의 ChatGPT 부정 시험 | fixture와 구분해 미수행 |
| 파괴적 write 작업의 실제 응용프로그램 실행 | 의도적으로 제외 |
| 장기간 운영 후 refresh token 자연 만료 | 시간 경과 시험 미수행 |
| HTTP headless fixture의 pipeline 상태 파일 접근 거부 | Gateway routing과 분리된 runtime 이슈로 남음 |

---

## 13. 최종 상태

이번 후속 검증에서 확인된 최종 구조는 다음과 같다.

~~~mermaid
flowchart TD
    Client["ChatGPT"]
    Tunnel["Secure MCP Tunnel"]
    Auth["OAuth / authorization"]
    Search["Provider-aware Tool Search"]
    Schema["Selected schema"]
    Call["Routed call"]
    Proxy["ProxyProvider"]
    Stdio["Fresh stdio client + transport"]
    Http["HTTP client"]
    Apps["Applications"]

    Client --> Tunnel
    Tunnel --> Auth
    Auth --> Search
    Search --> Schema
    Schema --> Call
    Call --> Proxy
    Proxy --> Stdio
    Proxy --> Http
    Stdio --> Apps
    Http --> Apps
~~~

확인된 범위는 다음과 같다.

- provider hard routing과 BM25 ranking 분리
- 한국어·영어 Tool Search 품질 검증
- 실제 Blender·Unity 읽기 실행
- OAuth state 영속성
- Gateway/Tunnel 재시작 후 세션 재사용
- refresh 실패·rotation fail-closed 검증
- stdio 동시성 결함 수정
- dynamic catalog 갱신
- 실제 ChatGPT E2E와 trace 검증
- version/behavior regression 기준선
- GPT Tool Calling 반복 관찰과 평가 경계 구분

이 문서는 위 후속 검증 결과만 기록하며, 기존 Routing 설계 문서와 OAuth 구성 문서의 내용을 현재 상태에 맞춰 병합하거나 재작성하지 않는다.
