# FastMCP 기반 다중 MCP Gateway 및 Headless 도구 연결 구성

## 1. 목적

여러 MCP 서버를 하나의 FastMCP Gateway 뒤에 통합하고, 최종 MCP 클라이언트가 필요한 도구만 효율적으로 발견·호출할 수 있는 구조를 구성했다.

최종 구성의 목표는 다음과 같다.

- 서로 다른 transport를 사용하는 MCP 서버를 하나의 Gateway에서 통합한다.
- 각 upstream MCP의 원본 도구 정의를 ProxyProvider를 통해 자동 발견한다.
- namespace를 적용해 서로 다른 MCP의 도구 이름 충돌을 방지한다.
- upstream 대상은 명시적인 app/provider 식별자로 먼저 한정하고, 해당 provider 내부에서만 Tool Search를 수행한다.
- 초기 목록은 검색·스키마 조회·호출 진입점으로 제한하고, 필요한 도구 정보만 단계적으로 가져온다.
- headless로 실행되는 응용프로그램의 MCP bridge를 실제 서비스 데이터까지 연결한다.
- 하나의 upstream 장애가 다른 upstream의 호출을 막지 않도록 연결을 분리한다.
- localhost의 Gateway를 인증된 외부 연결 계층을 통해 원격 MCP 클라이언트와 연결한다.
- Gateway 처리와 실제 upstream RPC를 구분해 기록한다.
- 최종 검증은 도구 목록 조회가 아니라 실제 응용프로그램 데이터 호출까지 확인한다.

사용한 핵심 기술은 FastMCP 4, MCP, ProxyProvider, ProxyClient, StdioTransport, Streamable HTTP, Namespace, BM25 Tool Search, FastMCP Middleware, Python logging, Windows Job Object, Blender MCP, Unity MCP, Secure MCP Tunnel이다.

---

## 2. 전체 구성

최종 구조는 FastMCP Gateway가 여러 MCP 서버를 한 endpoint로 집계하고, 외부 MCP 클라이언트가 인증된 연결 경로를 통해 Gateway를 사용하는 형태다.

~~~mermaid
flowchart TB
    Client["MCP Client"]
    Tunnel["Authenticated external connection"]
    Gateway["FastMCP Gateway"]

    Route["app/provider routing"]
    Search["BM25 discovery / schema lookup"]
    ProxyA["ProxyProvider A"]
    ProxyB["ProxyProvider B"]

    MCPA["stdio MCP server"]
    MCPB["HTTP MCP server"]

    BridgeA["Application bridge"]
    BridgeB["Application bridge"]

    AppA["Headless application A"]
    AppB["Headless application B"]

    Client <--> Tunnel
    Tunnel <--> Gateway
    Gateway --> Route
    Route --> Search
    Search --> ProxyA
    Search --> ProxyB
    ProxyA <--> MCPA
    ProxyB <--> MCPB
    MCPA <--> BridgeA
    MCPB <--> BridgeB
    BridgeA <--> AppA
    BridgeB <--> AppB
~~~

핵심 경계는 다음과 같다.

| 구성요소 | 역할 |
|---|---|
| FastMCP Gateway | upstream 집계, app/provider routing, namespace 적용, 단계적 도구 discovery·schema 조회, 호출 중계, Gateway 로그 |
| ProxyProvider / ProxyClient | upstream MCP discovery와 실제 MCP 호출 중계 |
| stdio MCP | Gateway가 transport의 일부로 MCP 자식 프로세스를 실행하고 표준 입출력으로 통신 |
| HTTP MCP | 별도로 실행 중인 MCP endpoint에 Gateway가 연결 |
| 응용프로그램 bridge | MCP 요청을 실제 응용프로그램 내부 API 또는 상태로 전달 |
| 외부 연결 계층 | localhost Gateway를 인증된 원격 MCP 클라이언트와 연결 |

응용프로그램 자체의 설치·라이선스·bridge 준비는 Gateway 책임과 분리했다. Gateway는 MCP transport와 도구 라우팅을 담당하고, 응용프로그램 런처는 해당 응용프로그램과 bridge의 실행을 담당한다.

---

## 3. FastMCP Gateway 기본 구성

### 3.1 독립 실행환경 구성

Gateway는 다른 MCP 서버의 Python 환경과 분리된 독립 실행환경으로 구성했다.

확인된 최종 기준은 다음과 같다.

- CPython 3.12 계열
- FastMCP 4.0.5
- FastMCP 외부 upstream 패키지를 Gateway 환경에 직접 설치하지 않음
- upstream의 자체 실행환경과 Gateway 실행환경을 분리

이 분리는 upstream 의존성 충돌을 줄이기 위한 것이다. stdio MCP가 별도의 프로젝트 환경을 사용하더라도 Gateway는 해당 환경을 transport 명령으로 호출할 뿐, upstream 패키지를 자신의 환경으로 가져오지 않는다.

### 3.2 서로 다른 transport 연결

두 종류의 upstream 연결 방식을 함께 사용했다.

~~~text
FastMCP Gateway
├─ ProxyProvider
│  └─ StdioTransport
│     └─ upstream MCP process
└─ ProxyProvider
   └─ Streamable HTTP
      └─ already-running upstream MCP endpoint
~~~

stdio 방식은 MCP 서버 프로세스 생성 자체가 transport의 일부다. 반면 HTTP 방식에서는 MCP 서버가 먼저 실행되어 있어야 하며 Gateway는 endpoint에 접속한다.

### 3.3 Namespace 적용

여러 upstream의 도구를 하나의 Gateway에 병합할 때 각 provider에 namespace를 적용했다.

개념적으로 도구 이름은 다음과 같이 변환된다.

~~~text
Upstream A: read_status
    ↓ namespace
Gateway: service_a_read_status

Upstream B: read_status
    ↓ namespace
Gateway: service_b_read_status
~~~

이를 통해 동일한 원본 도구 이름을 가진 MCP가 함께 연결되어도 클라이언트가 대상을 명확히 구분할 수 있다.

정책과 호출은 namespace 적용 이후의 Gateway 도구 이름을 기준으로 처리했다.

---

## 4. Provider-aware Tool Routing과 단계적 스키마 조회

최종 구성에서는 **어느 upstream을 대상으로 할지 결정하는 routing**과 **그 provider 안에서 관련 도구를 찾는 ranking**을 분리했다.

namespace는 도구 이름 충돌을 방지하고 소속을 식별하는 데 사용한다. 별도로 안정적인 app ID와 namespace의 대응을 유지하고, 클라이언트가 app을 명시하면 해당 provider의 namespaced 도구만 BM25 후보에 포함한다. app 이름 길이에 따른 자동 약어화는 사용하지 않았다.

초기 `tools/list`의 public surface는 응용프로그램 도구를 직접 고정 노출하지 않고 다음 세 진입점으로 제한했다.

~~~text
search_tools
get_tool_schema
call_tool
~~~

~~~mermaid
flowchart TD
    C["MCP Client"] --> S["search_tools(app, query, detail=brief)"]
    S --> V["권한이 적용된 전체 catalog"]
    V --> R["app → namespace 후보 hard filter"]
    R --> B["BM25 ranking"]
    B --> K["compact candidate list"]
    K --> G["get_tool_schema(app, names)"]
    G --> O["provider ownership + 권한 재검증"]
    O --> X["selected schema"]
    X --> T["call_tool(app, name, arguments)"]
    T --> P["provider ownership + 권한 재검증"]
    P --> U["ProxyProvider → upstream MCP"]
~~~

### 4.1 Provider 후보를 BM25 이전에 제한

기존 전체 catalog BM25 검색에서는 특정 응용프로그램을 질의에 적어도 다른 upstream의 공통 용어가 점수를 얻을 수 있었다. 최종 구성은 검색 문자열에 app 이름을 덧붙이는 대신 다음 순서로 처리한다.

~~~text
인증·권한이 적용된 catalog
        ↓
app 식별자 검증
        ↓
app에 대응하는 namespace 도구만 후보로 제한
        ↓
BM25 ranking
        ↓
상위 결과 반환
~~~

따라서 app 선택은 검색 힌트가 아니라 routing 조건이다. 존재하지 않는 app을 요청하면 전체 검색으로 fallback하지 않고 오류로 처리한다.

현재 구현의 ownership 검사는 **등록된 app ID → namespace 대응과 namespaced tool prefix**를 기준으로 한다. 이 방식은 현재 provider 구성이 명확한 환경에서 검증한 계약이며, 중첩 namespace나 별도 ownership metadata를 사용하는 다른 구현에 그대로 일반화하지 않는다.

### 4.2 Progressive disclosure

`search_tools`는 기본 `detail=brief`에서 도구 이름과 압축된 설명만 반환한다. 후보를 고른 뒤 `get_tool_schema`로 선택한 도구의 파라미터를 가져온다.

- `brief`: 후보 선택용 이름·짧은 설명
- `detailed`: 파라미터 이름·타입·필수 여부 중심의 compact schema
- `full`: 전체 MCP tool schema

일반 호출 흐름은 다음과 같다.

~~~text
search_tools(app, query)
        ↓
후보 도구 선택
        ↓
get_tool_schema(app, names, detail=detailed)
        ↓
필요하면 detail=full
        ↓
call_tool(app, name, arguments)
~~~

이 구조의 목적은 search 단계에서 상위 여러 도구의 전체 JSON schema를 매번 반환하지 않는 것이다.

### 4.3 호출 경계

`get_tool_schema`와 `call_tool`에서도 app과 namespaced tool의 소속을 다시 확인한다. 다른 provider의 이름을 app과 조합하면 upstream 호출 전에 거부한다.

Tool Search transform이 활성화된 경로에서는 숨겨진 원본 upstream 도구의 직접 호출도 허용하지 않고, 검증된 `call_tool(app, ...)` 경로로만 실행되게 했다. 따라서 discovery와 execution이 같은 provider 경계를 공유한다.

검색은 실행 권한을 대신하지 않는다. OAuth가 활성화된 구성에서는 권한이 적용된 catalog를 먼저 얻은 뒤 provider filter를 적용하고, schema 조회와 실제 호출에서도 동일 권한 경로를 유지한다.

### 4.4 전달량과 지연의 trade-off

같은 provider-aware 구현과 같은 live upstream에서 5회 반복 median으로 `search_tools(detail=full)`과 기본 `brief + 선택 도구 detailed schema`를 비교했다. 이는 이전 소프트웨어 리비전과의 비교가 아니라 **현재 구현 안에서 discovery 표현 방식을 비교한 측정**이다.

| 대표 질의 | full search | brief search | 선택 schema | brief+schema | full payload | brief+schema payload | 감소 |
|---|---:|---:|---:|---:|---:|---:|---:|
| stdio upstream의 scene/object 조회 | 약 896 ms | 약 894 ms | 약 890 ms | 약 1,785 ms | 3,325 B | 1,275 B | 61.7% |
| stdio upstream의 Python 실행 검색 | 약 911 ms | 약 901 ms | 약 893 ms | 약 1,794 ms | 7,314 B | 1,418 B | 80.6% |
| HTTP upstream의 scene/prefab 검색 | 약 902 ms | 약 898 ms | 약 898 ms | 약 1,796 ms | 13,417 B | 2,542 B | 81.1% |
| HTTP upstream의 animation/build 검색 | 약 890 ms | 약 895 ms | 약 894 ms | 약 1,789 ms | 16,735 B | 2,402 B | 85.6% |

compact rendering 자체는 search latency를 의미 있게 줄이지 않았다. provider catalog 조회 비용이 지배적이었다. 한 도구를 선택하는 표준 흐름에서는 public 호출이 `search + call` 2회에서 `search + schema + call` 3회로 늘고 discovery 지연도 증가하지만, 전달되는 schema 데이터는 크게 줄었다.

따라서 이 변경은 latency 최적화가 아니라 **모델 context와 불필요한 schema 노출을 줄이기 위한 trade-off**로 적용했다.

---

## 5. Headless HTTP MCP 연결

HTTP 기반 MCP는 MCP 서버와 실제 응용프로그램 프로세스가 별도로 존재하는 구조였다.

최종 연결은 다음과 같다.

~~~text
FastMCP Gateway
    ⇅ Streamable HTTP
MCP server
    ⇅ application bridge
Headless application process
    ⇅
project / scene / console state
~~~

응용프로그램은 GUI Editor를 열지 않고 batch/headless 모드로 실행했다. 내부 bootstrap이 MCP 서버의 bridge endpoint에 접속하고, 연결이 완료된 뒤 프로젝트 범위 도구가 활성화되는 구조를 확인했다.

### 5.1 준비 상태 판정

단순히 MCP HTTP port가 열렸는지만 확인하지 않았다.

다음 두 조건을 모두 확인한 뒤 준비 완료로 판정했다.

1. MCP HTTP 서버가 요청을 받을 수 있음
2. headless 응용프로그램 bridge가 MCP 서버에 등록됨

bridge 연결 후에는 응용프로그램 Console 조회와 현재 scene 조회를 실제 FastMCP 경로로 실행했다.

### 5.2 프로세스 수명

headless 응용프로그램과 MCP 서버를 시작한 런처가 자신이 소유한 프로세스만 정리하도록 Windows Job Object의 KILL_ON_JOB_CLOSE 동작을 사용했다.

Job Object에 포함되지 못한 프로세스에는 PID와 실행 파일 이름을 다시 확인한 뒤 fallback tree cleanup을 수행하도록 구성했다.

이는 프로세스 소유권을 명확히 하면서 다른 세션에서 실행한 동일 프로그램을 잘못 종료하지 않기 위한 구조다.

---

## 6. Upstream 통합 및 장애 분리 검증

Gateway의 기본 연결을 구성한 뒤 두 upstream을 함께 연결해 catalog와 호출 경로를 확인했다.

확인된 동작은 다음과 같다.

- 두 upstream의 도구가 각각의 namespace 아래에서 발견되었다.
- Tool Search가 숨겨진 upstream 도구를 검색할 수 있었다.
- call_tool이 검색된 실제 upstream 도구를 호출했다.
- HTTP MCP를 통해 응용프로그램 Console 및 scene 정보를 읽을 수 있었다.
- stdio MCP의 문서 도구와 실제 응용프로그램 의존 도구를 호출할 수 있었다.

### 6.1 장애 분리와 무재시작 복구

provider-aware routing 적용 후 양쪽 연결을 다시 중단·복구하며 격리를 검증했다.

첫 번째 경로에서는 stdio upstream이 의존하는 headless 응용프로그램 bridge를 중지했다. 그 동안 HTTP provider의 검색과 실제 MCP 호출은 계속 성공했고 Gateway listener는 유지되었다. bridge를 다시 실행한 뒤 Gateway를 재시작하지 않고 stdio provider의 검색과 응용프로그램 조회가 복구되었다.

두 번째 경로에서는 HTTP 기반 headless 응용프로그램을 종료해 launcher가 소유한 HTTP MCP server까지 함께 정리되는 상태를 만들었다. 이 동안 stdio provider의 검색과 실제 응용프로그램 조회는 계속 성공했다. HTTP MCP와 응용프로그램을 다시 기동한 뒤에도 Gateway 재시작 없이 해당 provider의 검색과 호출이 복구되었다.

~~~mermaid
flowchart TD
    FailA["Provider/bridge A unavailable"] --> Keep["FastMCP Gateway remains running"]
    Keep --> B["Provider B search/call remains available"]
    RecoverA["A restarted"] --> Same["same Gateway process"]
    Same --> ABack["A search/call recovers"]
~~~

이 검증으로 한 provider 또는 그 응용프로그램 bridge의 장애가 다른 provider의 정상 경로를 막지 않고, 복구 후 Gateway 재시작 없이 다시 사용할 수 있음을 확인했다. bridge 장애와 MCP transport 자체의 장애는 같은 현상으로 취급하지 않고 각각 관찰된 범위로 구분했다.

---

## 7. 원격 MCP 클라이언트 연결

Gateway는 localhost에 유지하고, 외부 MCP 클라이언트 연결은 별도의 인증된 tunnel 계층을 사용했다.

~~~mermaid
flowchart LR
    Remote["Remote MCP Client"]
    Secure["Secure MCP Tunnel"]
    Local["localhost FastMCP Gateway"]
    Providers["ProxyProviders"]
    Services["Upstream MCPs"]

    Remote <--> Secure
    Secure <--> Local
    Local <--> Providers
    Providers <--> Services
~~~

터널 실행 전에는 다음을 확인했다.

- 터널 프로필이 로컬 FastMCP endpoint를 가리키는지
- Runtime API key가 프로세스 환경에서 로드되는지
- 터널 진단 명령이 로컬 MCP 접근을 성공으로 판정하는지
- health와 readiness endpoint가 정상 상태인지

최종적으로 원격 MCP 클라이언트에서 Tool Search를 수행하고, 검색된 HTTP 기반 응용프로그램 도구를 call_tool로 호출해 실제 Console 메시지를 읽었다.

이후 stdio 기반 응용프로그램 도구도 같은 원격 경로에서 호출해 실제 데이터블록 정보를 읽었다.

따라서 다음 전체 경로가 실제로 검증되었다.

~~~text
Remote MCP Client
→ Secure Tunnel
→ FastMCP Gateway
→ search_tools(app, query)
→ get_tool_schema(app, names)
→ call_tool(app, name, arguments)
→ ProxyProvider
→ upstream MCP
→ application bridge
→ headless application
→ result
~~~

---

## 8. Headless stdio MCP 연결

stdio MCP 서버는 자체적으로 실행되면서 로컬 TCP bridge를 통해 headless 응용프로그램과 연결되는 구조였다.

~~~text
FastMCP Gateway
    ⇅ MCP / stdio
stdio MCP server
    ⇅ local TCP
application MCP extension
    ⇅
headless application
~~~

응용프로그램은 background 모드와 온라인 접근 허용 옵션을 사용하고, MCP 확장이 제공하는 CLI command를 실행하는 방식으로 기동했다.

최종적으로 다음을 확인했다.

- GUI 창 없이 background 프로세스로 실행됨
- MCP TCP bridge가 localhost에서 listen
- stdio MCP가 bridge에 접속
- FastMCP를 통해 실제 scene/object 정보 조회 성공
- 원격 MCP 클라이언트를 통한 실제 데이터 조회 성공

---

## 9. FastMCP 자체 로그와 upstream RPC 로그

Gateway의 일반 요청 로그와 실제 upstream MCP 통신은 서로 다른 계층이므로 두 종류의 로그로 구분했다.

### 9.1 Gateway 처리 로그

FastMCP Middleware에서 논리적인 Gateway 요청을 기록했다.

주요 필드는 다음과 같다.

- timestamp
- request trace ID
- RPC ID
- MCP method
- tool name
- duration
- outcome
- MCP error 여부

### 9.2 실제 upstream RPC 로그

Gateway Middleware만으로는 캐시·Tool Search 내부 처리와 실제 upstream 송신을 구분할 수 없다.

따라서 ProxyClient가 사용하는 session의 send_request 경로를 계측해 실제 upstream RPC를 별도로 기록했다.

~~~text
Client request
    │
    ├─ Gateway dispatch log
    │
    └─ Tool Search / routing
            │
            └─ actual upstream send_request
                    │
                    └─ upstream RPC log
~~~

같은 호출 흐름은 trace ID로 연관시키고, 각각의 요청/응답 쌍은 RPC ID로 구분했다.

### 9.3 Payload 비기록 원칙

로그에는 기본적으로 다음 내용을 남기지 않았다.

- tool arguments 전체
- result body
- 인증 header
- API key
- exception 문자열 전체

예외 메시지나 프레임워크 경고에도 입력 payload가 포함될 수 있으므로 warning/error 텍스트도 그대로 복사하지 않고 오류 종류와 코드 중심으로 기록했다.

로그는 UTF-8 JSON Lines 형식으로 저장하고, 파일당 2 MiB에서 rotation하며 이전 로그 3개를 보존하도록 구성했다.

### 9.4 버전 의존 지점

실제 upstream RPC 기록은 FastMCP 4.0.5의 ProxyClient transport option과 session class 확장 지점을 사용했다.

따라서 FastMCP 버전을 변경할 경우 다음 항목을 다시 검증해야 한다.

- ProxyClient가 사용하는 session class 연결 방식
- send_request hook이 실제 upstream RPC 아래에서 실행되는지
- 기존 protocol forwarding 동작이 보존되는지
- Tool Search만 수행했을 때 tools/call 로그가 잘못 생성되지 않는지

---

## 10. 최종 검증

provider-aware routing과 progressive disclosure 적용 후 자동 테스트, live upstream 통합, 원격 OAuth 클라이언트, 장애 복구 및 성능 측정을 다시 수행했다.

### 10.1 자동 테스트

최종 회귀 suite 결과는 다음과 같다.

~~~text
35 passed, 2 skipped
~~~

skip은 실행 조건이 없는 선택 시험으로 남겼고 성공으로 합산하지 않았다.

검증 범위에는 다음 내용이 포함되었다.

- public surface가 `search_tools`, `get_tool_schema`, `call_tool` 세 도구로 제한됨
- app별 BM25 후보 격리
- brief 검색에서 전체 `inputSchema`가 노출되지 않음
- selected schema의 detailed/full 조회
- 잘못된 app 및 cross-provider schema/call 거부
- 숨겨진 원본 upstream 도구의 직접 호출 차단
- namespace 일관성
- OAuth scope가 적용된 catalog에서 schema와 call 우회 방지
- HTTP/stdio upstream 검색·호출
- Gateway ↔ upstream RPC trace 연관
- payload 미기록, 연결 실패·cancellation·rotation 등 기존 로깅 회귀

### 10.2 Live upstream과 원격 클라이언트

live 통합 검증에서 두 provider의 전체 catalog가 namespaced 상태로 확인되었고, public surface는 synthetic tool 세 개만 반환했다.

실제 원격 OAuth 클라이언트에서는 다음을 다시 확인했다.

- stdio 기반 provider를 대상으로 compact search 성공
- 선택한 stdio 도구의 detailed schema 조회 성공
- 같은 app으로 실제 응용프로그램 상태 조회 성공
- HTTP provider를 대상으로 compact search 성공
- HTTP MCP의 request-context 도구 호출 성공
- stdio app을 지정한 상태에서 HTTP 도구 schema 조회 거부
- stdio app을 지정한 상태에서 HTTP 도구 call 거부
- 서버 schema 변경 후 클라이언트 metadata를 새로고침하면 세 public tool의 최신 입력 스키마가 반영됨

HTTP provider의 request-context 성공은 MCP provider 경로의 복구를 입증하지만, 그 호출만으로 headless 응용프로그램 데이터까지 읽었다고 확대하지 않는다. HTTP headless application의 scene/Console 데이터 연결은 별도의 기존 live 통합 검증에서 확인했다.

### 10.3 장애와 성능

- 한 provider/bridge가 내려가도 다른 provider의 검색·실호출이 계속 동작했다.
- 중단한 provider를 복구하면 Gateway를 재시작하지 않고 검색·호출이 다시 성공했다.
- 동일 현재 구현에서 full discovery와 brief+selected schema를 비교했을 때 대표 질의의 discovery payload는 약 61.7%~85.6% 감소했다.
- 대신 schema 조회 한 번이 추가되므로 단일 도구 선택 workflow의 discovery latency는 증가했다.

최종 성공 판정은 `tools/list`나 포트 상태만이 아니라 **provider 경계, 실제 upstream 호출, 응용프로그램 결과, 장애 후 복구 및 권한 거부가 의도대로 동작하는지**를 함께 기준으로 했다.

---

## 11. 주요 이슈

### 11.1 Headless 응용프로그램 라이선스 오류

#### 이슈 내용과 발생 시점

HTTP 기반 응용프로그램을 batch/headless 모드로 실행하는 과정에서 프로세스가 라이선스 오류와 함께 종료되었다.

#### 발생 원인 추적

MCP 서버와 bridge 코드 자체는 정상적으로 준비되어 있었지만, headless 실행에 필요한 응용프로그램 라이선스가 현재 환경에서 활성 상태가 아니었다.

#### 해결 과정

응용프로그램의 정식 인증 절차를 완료한 뒤 동일 headless launcher를 다시 실행했다.

#### 검증

- headless 프로세스가 정상 유지됨
- MCP 서버에 application bridge가 등록됨
- Console 및 scene 도구 호출 성공

#### 결과

MCP 연결 문제가 아니라 **headless 응용프로그램 실행 전제인 라이선스 상태 문제**였음을 확인했다.

---

### 11.2 Microsoft Store 응용프로그램의 headless 실행 실패

#### 이슈 내용과 발생 시점

Microsoft Store로 설치된 응용프로그램을 headless MCP bridge로 실행할 때 패키지 내부 WindowsApps 실행 파일을 직접 호출하는 방식이 비대화형 환경에서 정상 기동되지 않았다.

직접 실행에서는 프로세스 생성 실패와 접근 거부가 확인되었고, AppX activation을 통한 별도 시도에서도 사용할 수 있는 headless bridge가 만들어지지 않았다.

#### 발생 원인 추적

설치 자체가 없는 것은 아니었으며 Windows에는 해당 응용프로그램의 App Execution Alias가 등록되어 있었다.

별칭을 통해 background 실행을 probe했을 때 실제 응용프로그램이 GUI 없이 정상 실행되었다.

따라서 보호된 패키지 내부 EXE의 직접 경로를 실행 대상으로 사용하는 방식이 현재 실행환경에 적합하지 않음을 확인했다.

추가로 headless 프로필에는 MCP 확장이 활성화되어 있지 않아 MCP CLI command를 사용할 수 없는 상태도 확인했다.

#### 해결 과정

1. Windows에 등록된 App Execution Alias를 실행 경로로 사용
2. 비대화형 셸에서 빠질 수 있는 Windows known-folder 환경변수를 보완
3. 공식 MCP 확장을 현재 응용프로그램 프로필에 설치·활성화
4. background + online access + MCP CLI command 조합으로 실행

#### 검증

- background mode 확인
- GUI main window 없음 확인
- MCP TCP bridge listen 확인
- FastMCP를 통한 실제 object/scene 조회 성공
- 런처 종료 시 소유 bridge 프로세스 정리 확인
- 재실행 성공
- 원격 MCP 클라이언트에서 실제 데이터 조회 성공

#### 결과

Microsoft Store 설치를 제거하거나 WindowsApps ACL을 변경하지 않고, **공식 App Execution Alias와 MCP 확장을 이용하는 방식으로 headless bridge를 정상화**했다.

---

### 11.3 Gateway 로그와 실제 upstream RPC의 구분

#### 이슈 내용과 발생 시점

FastMCP의 일반 요청 로그만으로는 call_tool 요청이 실제로 어느 upstream 도구까지 전달되었는지 명확히 추적할 수 없었다.

#### 발생 원인 추적

Gateway Middleware는 클라이언트 요청과 내부 Tool Search/라우팅을 관찰하지만, provider cache 아래에서 실제 MCP 서버로 전송된 RPC와 동일한 계층은 아니다.

따라서 Middleware 로그만으로 실제 upstream 전송을 판정하면 캐시 응답이나 내부 호출을 송신으로 오인할 수 있었다.

#### 해결 과정

ProxyClient가 실제 MCP session을 만드는 지점을 확인하고, session의 send_request 경로를 계측했다.

Gateway 요청과 upstream RPC가 동일한 trace ID를 공유하도록 하고, 요청별 RPC ID와 실행 시간을 별도로 기록했다.

#### 검증

- search_tools만 실행할 때 upstream tools/call이 기록되지 않음
- call_tool 실행 시 실제 원본 도구 이름으로 upstream tools/call 기록
- Gateway와 upstream 기록이 동일 trace ID로 연결됨
- 성공, MCP error, exception, cancellation 구분 확인
- 입력 payload와 결과 body가 로그에 포함되지 않음

#### 결과

Gateway 처리와 실제 MCP 통신을 분리하면서도 하나의 호출 흐름으로 추적할 수 있게 되었다.

---

## 12. 최종 동작 원리 요약

최종 구조에서 핵심 요청 흐름은 다음과 같다.

~~~mermaid
flowchart TD
    A["Client tools/list"] --> B["Public tools + search_tools + call_tool"]
    B --> C{"Tool directly visible?"}

    C -- Yes --> D["Gateway tool call"]
    C -- No --> E["search_tools"]
    E --> F["Tool name + schema"]
    F --> G["call_tool"]

    D --> H["Namespace routing"]
    G --> H

    H --> I["ProxyProvider"]
    I --> J["stdio or HTTP MCP"]
    J --> K["Application bridge"]
    K --> L["Headless application"]

    L --> M["Application result"]
    M --> J
    J --> I
    I --> N["Gateway result"]
    N --> O["Client"]

    H -. trace .-> P["Gateway dispatch log"]
    I -. trace .-> Q["Upstream RPC log"]
~~~

구성의 핵심은 다음 네 가지다.

1. **FastMCP의 provider 기능을 사용해 원본 MCP 스키마를 재작성하지 않고 통합한다.**
2. **namespace와 Tool Search로 여러 MCP의 도구를 충돌 없이 필요한 만큼만 노출한다.**
3. **응용프로그램 bridge의 실제 데이터까지 호출해 연결 성공을 검증한다.**
4. **Gateway 처리와 실제 upstream RPC를 분리해 기록하되 하나의 trace로 연결한다.**

이 구조를 통해 서로 다른 transport와 headless 응용프로그램을 사용하는 MCP들을 하나의 FastMCP Gateway 아래에서 일관된 검색·호출 모델로 사용할 수 있음을 확인했다.
