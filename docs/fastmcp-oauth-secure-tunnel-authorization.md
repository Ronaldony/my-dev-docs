# FastMCP OAuth·Secure MCP Tunnel 연결 및 도구 권한 검증

## 1. 목적과 기록 범위

기존 FastMCP Gateway에 사용자 OAuth 인증을 적용하고, ChatGPT가 Secure MCP Tunnel을 통해 비공개 MCP에 접근하는 구성을 연결·검증했다. 공개 HTTPS는 OAuth에 필요한 경로에만 제공하고, MCP 본체는 loopback에 유지했다.

[다중 MCP Gateway 및 Headless 연결 문서](fastmcp-multi-upstream-headless-gateway.md)의 provider·namespace·Tool Search 구성을 전제로 한다. 이 문서는 그 구성 위에 추가한 인증·권한·외부 연결과 관련 이슈를 다룬다. 기존 문서의 서비스 호출 성공을 이번 OAuth 적용 후의 성공으로 합산하지 않는다.

**확인된 최종 범위는 사용자 OAuth 재연결, 현재 public surface 3개(`search_tools`, `get_tool_schema`, `call_tool`)의 metadata 반영, provider-aware `search → schema → call` 원격 호출, cross-provider 거부 및 stdio 기반 응용프로그램 읽기 성공까지다.** HTTP provider의 원격 MCP 호출도 확인했지만 해당 최종 호출은 request-context 조회였으므로 headless 응용프로그램 데이터 검증과 구분한다. 실제 원격 쓰기 거부와 만료 후 refresh는 아직 완료 범위에 포함하지 않는다.

근거는 해당 작업의 설정·구현 조회, launcher와 Tunnel 진단, 운영 프로세스의 인증·도구 목록 로그, 테스트 종료 코드 및 Git 상태다. 이번 문서화에서 운영 서버를 다시 실행한 것은 아니다. 환경 고유값은 역할로 치환했으며, 아래 `<…>` 표현은 실제 입력으로 바꿔야 하는 자리표시자다.

## 2. 구성요소와 연결 경계

사용한 핵심 기술은 Windows PowerShell, FastMCP 4.0.5, MCP Streamable HTTP, `OIDCProxy`·`OAuthProxy`, `JWTVerifier`, `AuthMiddleware`, Tool Search, Auth0, OpenID Connect, OAuth authorization code·PKCE, CIMD, Protected Resource Metadata, OpenAI Secure MCP Tunnel, Harpoon, Tailscale Funnel, Windows Credential Manager 및 Git이다. 버전은 작업 당시 확인값이며 다른 버전의 동작 보장이 아니다.

~~~mermaid
flowchart LR
    Client["ChatGPT MCP / OAuth client"]
    Browser["사용자 브라우저"]
    Secure["OpenAI Secure MCP Tunnel"]
    Agent["로컬 tunnel-client"]
    MCP["비공개 FastMCP /mcp"]
    Public["Tailscale Funnel: 공개 OAuth 경로"]
    Proxy["FastMCP OIDCProxy"]
    IdP["Auth0"]
    Policy["subject / scope 권한 검사"]
    Providers["Blender / Unity ProxyProviders"]

    Client <--> Secure
    Secure <--> Agent
    Agent <--> MCP
    Client --> Browser
    Browser <--> Public
    Client -->|"공개 token endpoint"| Public
    Public <--> Proxy
    Proxy <--> IdP
    MCP --> Policy
    Policy --> Providers
~~~

FastMCP의 MCP endpoint와 OAuth route는 같은 로컬 HTTP listener에서 처리했다. OAuth를 위해 별도의 FastMCP 전용 포트를 추가한 구성이 아니다. Tunnel의 health/admin listener는 MCP listener와 별도이며, 공개 OAuth 주소는 Funnel이 해당 로컬 route로 전달했다.

| 구성요소 | 이 구성에서 맡은 책임 |
|---|---|
| ChatGPT 앱·플러그인 | OAuth client 정보 제공, 사용자 연결, 도구 정의 수집 및 호출 |
| Secure MCP Tunnel | 등록된 Tunnel ID와 로컬 MCP target 사이의 전달 |
| Harpoon | Tunnel의 OAuth discovery에 필요한 로컬 metadata/resource target 처리 |
| Tailscale Funnel | 브라우저와 OAuth client가 접근할 공개 HTTPS 경로 제공 |
| FastMCP OAuth proxy | Auth0 연결, downstream OAuth 처리, token 검증과 resource 연계 |
| Auth0 | 사용자 로그인, API audience와 scope를 포함한 upstream 인증 |
| FastMCP 권한 정책 | 검증된 subject의 허용 여부와 도구별 scope 검사 |

Tunnel 연결에 쓰는 runtime API key와 MCP 사용자 OAuth는 별개의 인증이다. Tunnel이 연결되거나 health 검사가 통과했다는 이유만으로 사용자에게 도구 접근 권한이 생기지는 않는다.

## 3. 외부 IdP와 권한 구성

### 3.1 두 OAuth client 관계

~~~mermaid
flowchart LR
    C["ChatGPT: downstream OAuth client"]
    G["FastMCP: downstream OAuth server / upstream OAuth client"]
    A["Auth0: upstream authorization server"]
    C <-->|"CIMD · authorization code · PKCE"| G
    G <-->|"Regular Web Application 자격정보"| A
~~~

Auth0에는 API와 Regular Web Application을 구성했다. API Identifier는 upstream token의 audience를 식별하고, Application은 FastMCP가 Auth0와 통신할 client 자격정보 및 callback을 정의했다.

| 항목 | 적용 방식 |
|---|---|
| Auth0 API | Identifier와 `mcp.read`, `mcp.write` 권한 구성 |
| Auth0 Application | FastMCP용 Regular Web Application, Client ID·Client Secret, callback 등록 |
| OIDC discovery | Auth0의 `/.well-known/openid-configuration` 사용 |
| 로그인 scope | `openid`, `offline_access` 처리 및 Allow Offline Access 설정 |
| ChatGPT client 등록 | CIMD 사용; Auth0 Application 자격정보를 ChatGPT client 자격정보와 혼동하지 않음 |
| 비밀정보 | Client Secret과 proxy signing key를 Windows Credential Manager를 통해 로드 |

Callback도 둘이다. Auth0에 등록하는 callback은 **공개 FastMCP OAuth origin의 `/auth/callback`**이고, FastMCP가 마지막 authorization code를 돌려보내는 대상은 **ChatGPT가 제공한 callback**이다. 두 주소를 서로 대신 사용하지 않았다.

### 3.2 인증과 도구 권한의 분리

검증된 token의 `subject`를 허용 목록과 비교하는 전역 `AuthMiddleware`를 적용했다. 그와 별개로 도구에 scope 검사를 부여했다.

| 정책 | 적용한 검사 |
|---|---|
| 사용자 허용 | token과 subject가 존재하고 정확한 subject가 allowlist에 포함됨 |
| 읽기 도구 | `require_scopes`로 `mcp.read` 확인 |
| 쓰기 도구 | `mcp.read`와 `mcp.write`를 모두 확인 |
| 분류 불명 | 허용하지 않음; 확인된 override 또는 annotation으로 정책 결정 |

도구 정책은 namespace 적용 후 이름을 사용했다. Scope 정책 transform을 provider-aware Tool Search보다 먼저 구성하고, 전역 subject 검사는 `search_tools`, `get_tool_schema`, `call_tool`을 포함한 public 도구 목록에도 적용했다. 검색 후보를 만들기 전에 권한이 적용된 catalog를 얻고, schema 조회와 실제 호출에서도 같은 권한 경로를 유지했다. 직접 노출 여부와 실행 권한은 다른 개념이다.

`openid`·`offline_access`는 로그인·세션 관련 scope다. 두 값의 존재를 MCP 읽기·쓰기 권한으로 대신 취급하지 않았다. Refresh를 위한 설정이 준비된 상태와 실제 token 만료 후 refresh 성공은 구분했다.

## 4. 공개 OAuth와 비공개 resource 구성

### 4.1 URL의 역할 분리

| 역할 | 역할 치환 예시 또는 설정 의미 |
|---|---|
| 로컬 MCP target | `http://127.0.0.1:<gateway-port>/mcp` |
| 로컬 Protected Resource Metadata | 같은 origin의 `/.well-known/oauth-protected-resource/mcp` |
| 공개 OAuth origin | `https://<oauth-public-host>`; `base_url`에 대응 |
| 공개 OAuth callback | `https://<oauth-public-host>/auth/callback` |
| FastMCP canonical resource | `resource_base_url`과 MCP path로 정해지는 로컬 resource |
| ChatGPT가 요청한 Tunnel resource | 해당 Tunnel의 client-facing resource URL |
| Auth0 API audience | Auth0에 이미 등록한 API Identifier; 위 접속 주소와 별도로 유지 |
| Auth0 issuer | OIDC discovery·JWT 검증의 발급자; 공개 FastMCP OAuth origin과 구분 |

처음 공개 OAuth 주소로 사용하려던 도메인은 DNS 조회에서 `NXDOMAIN`을 반환했고 HTTPS 접근도 실패했다. 공개 OAuth 주소와 callback은 Tailscale Funnel의 유효한 HTTPS host로 변경했다.

**같은 문자열이 Auth0 API Identifier에도 쓰였다는 이유만으로 API를 재생성하지는 않았다.** 이 구성에서 audience는 API 식별값으로 유지했고, 브라우저가 방문할 OAuth 주소는 실제 DNS·HTTPS가 동작하는 주소로 분리했다. 공개 접속 주소의 DNS 실패와 audience 식별값의 정합성은 다른 검사였다.

### 4.2 경로 단위 공개

Funnel은 OAuth 동의·인증·callback·token 및 필요한 metadata 경로를 로컬 FastMCP listener로 전달했다. MCP 본체인 `/mcp`는 Funnel에 노출하지 않고 Secure MCP Tunnel만 사용했다.

확인된 경계는 다음과 같다.

~~~text
로컬 /mcp, 인증정보 없음
→ 401 + WWW-Authenticate

공개 OAuth metadata
→ HTTP 200

공개 Funnel host의 /mcp
→ HTTP 404

Secure MCP Tunnel의 MCP target
→ 로컬 /mcp
~~~

이는 당시 확인한 경로의 공개 범위다. 단일 404 응답을 모든 경로·방화벽·다른 서비스의 보안 검증으로 확대하지 않는다.

## 5. Launcher와 실제 runtime 정합성

### 5.1 환경변수 우선순위

전용 PowerShell launcher가 `.env`에서 관리 대상 접두사의 설정을 읽고 자식 FastMCP 프로세스에 전달하도록 구성했다. 기존 로더는 Process 환경변수가 없을 때만 값을 설정해서, 같은 셸에서 재실행하면 이전 값이 남았다.

최종 로딩 방식은 **`.env`에 존재하는 관리 대상 키를 매 실행 덮어쓰는 것**이었다.

~~~powershell
[Environment]::SetEnvironmentVariable($name, $value, 'Process')
~~~

FastMCP 실행에는 `--skip-env`를 유지했다. `.env` 로딩을 launcher 한 곳에서 담당하기 위한 구성이다. 임의의 모든 OS 환경변수를 덮어쓰는 방식으로 일반화하지 않으며, `.env`에서 삭제한 키의 잔존값 자동 정리까지 구현·검증했다는 의미도 아니다.

일부러 이전 resource/audience 값을 Process 환경에 넣고 launcher를 실행한 뒤, **실제 listener의 `WWW-Authenticate`와 metadata가 `.env`의 새 값으로 바뀌는지** 확인했다. 별도 셸의 preflight 성공만으로 runtime 반영을 판정하지 않았다.

### 5.2 프로세스와 포트

MCP listener와 Tunnel health/admin listener의 점유 PID, 실행 파일, command line 및 프로필을 대조했다. 기존 Tunnel과의 중복 실행 때문에 health 포트 bind가 실패한 문제를 launcher의 중복 실행 방지·소유 프로세스 정리로 보완했다.

실패 시 자신이 시작한 Gateway 프로세스 트리를 정리하는 흐름과 재실행을 확인했다. 최종 점검에서는 MCP·health 포트가 각각 listen 중이었고 `/readyz`는 HTTP 200이었다. 특정 PID나 포트 번호 자체가 재사용할 설정값은 아니므로 문서에는 고정하지 않았다.

## 6. Discovery와 resource binding 연결

### 6.1 로컬 resource와 공개 authorization server

FastMCP의 실제 route를 확인해 필요한 Protected Resource Metadata 경로를 보완했다. 최종적으로 로컬 `/mcp`의 401 challenge는 로컬 metadata를 가리키고, 그 metadata는 로컬 resource와 공개 OAuth authorization server를 구분했다.

다음은 실제 구성 관계를 역할로 치환한 예시다.

~~~json
{
  "resource": "http://127.0.0.1:<gateway-port>/mcp",
  "authorization_servers": ["https://<oauth-public-host>/"]
}
~~~

Tunnel은 로컬 Protected Resource Metadata와 공개 Authorization Server Metadata를 조회했다. PKCE S256, CIMD 및 authorize/token/registration endpoint를 확인했다. 이 배포에서는 신뢰하는 loopback HTTP target에 대해 `HARPOON_ALLOW_PLAINTEXT_HTTP`를 명시적으로 사용했다.

그 결과 resource와 metadata source에 대응하는 Harpoon target 두 개가 자동 등록됐다. 이 개수는 당시 구성의 관측값이지 모든 배포의 필수 개수가 아니다. 로컬 HTTP 허용을 공개 HTTP 사용이나 resource 검증 해제로 확대하지 않았다.

### 6.2 Tunnel resource alias

Discovery가 진행된 뒤 실제 authorization 요청에서는 다음 불일치가 발생했다.

~~~text
ChatGPT가 보낸 resource = Tunnel의 client-facing resource URL
FastMCP가 기대한 resource = 로컬 canonical MCP resource
→ invalid_target: Resource does not match this server
~~~

당시 Tunnel의 resource rewrite와 FastMCP의 `OAuthProxy.authorize()` 검증을 대조했다. 공개 OAuth endpoint에 도착한 요청에는 로컬 주소가 아니라 Tunnel resource가 들어왔다.

FastMCP proxy subclass에서 **승인된 해당 Tunnel resource alias만 기존 canonical resource로 변환**한 뒤 원래 `authorize()`에 전달하도록 보완했다. Auth0 audience나 로컬 metadata 구조를 다시 바꾸지 않았다.

~~~mermaid
flowchart LR
    R["요청 resource"] --> A{"허용 alias와 정규화 후 일치?"}
    A -->|"일치"| C["로컬 canonical resource로 변환"]
    A -->|"불일치"| K["요청값 유지"]
    C --> V["기존 FastMCP authorize 검증"]
    K --> V
    V -->|"유효"| Consent["consent 진행"]
    V -->|"다른 resource"| Reject["invalid_target"]
~~~

비교에는 당시 FastMCP의 `normalize_resource_url`을 사용했다. Query·fragment·trailing slash 처리가 포함된 **정규화 후 일치**이며 원문 byte 단위 일치나 임의 host/prefix 허용과는 다르다. Alias 설정은 HTTPS URL인지도 검사했다.

허용 alias의 로컬·공개 `/authorize` 요청은 `/consent`로 진행했고, 다른 host의 resource는 `invalid_target`을 반환했다. 이후 사용자도 실제 ChatGPT 재연결 성공을 보고했다. 이 보완은 확인한 authorization 단계의 호환 처리이며, 모든 버전의 token·refresh·alias 조합에 대한 보안 검증을 의미하지 않는다.

## 7. 인증된 도구 목록과 subject 불일치 진단

### 7.1 실제 응답을 기준으로 문제 계층 구분

OAuth 연결 성공 뒤에도 플러그인에 도구가 보이지 않았다. `tools/list`가 `outcome=ok`였다는 기록만으로는 도구가 반환됐는지 알 수 없었다.

Gateway middleware에 `tool_list_catalog` 진단을 추가해 도구 수·이름과 nested 여부를 기록했다. Tool Search의 내부 목록과 최상위 요청을 구분하고, 시각·운영 listener PID·trace ID·RPC ID를 맞춰 해당 요청만 확인했다.

다른 플러그인의 `remote_*` 목록, middleware를 우회한 로컬 probe, 별도 테스트 프로세스의 도구 7개/74개 결과는 최종 ChatGPT 응답의 증거에서 제외했다. 최종적으로 **운영 프로세스의 인증 요청이 최상위 `tools/list`에서 실제로 0개를 반환**한 것을 확인했다.

### 7.2 인증 context 비교

추가 진단은 token·subject의 존재, subject 허용 여부, scope, component와 trace를 기록했다. 이어서 provider prefix와 subject의 SHA-256 앞 12자리 지문을 비교했다. Token·secret·authorization code·subject 원문은 진단 로그에 넣지 않았다. 지문은 비교용이며 완전한 익명화를 보장하는 값으로 취급하지 않는다.

아래는 동일 요청의 로그에서 필요한 필드만 추린 결과다.

~~~text
token_present=true
subject_present=true
subject_allowed=false
scopes=[mcp.read, mcp.write, offline_access, openid]
tool_count=0
nested=false
~~~

Scope가 부족한 상태가 아니었다. 실제 subject의 provider는 `google-oauth2`, 허용 목록의 provider는 `auth0`였으며 지문도 달랐다.

~~~mermaid
flowchart TD
    Login["Auth0에서 Google identity로 로그인"] --> Actual["실제 subject: google-oauth2 계열"]
    Config["허용 subject: Auth0 DB identity 계열"] --> Check{"정확한 subject 일치?"}
    Actual --> Check
    Check -->|"불일치"| Filter["전역 AuthMiddleware에서 모두 제외"]
    Filter --> Empty["tools/list: 0개"]
    Approval["사용자 승인 후 실제 Google subject로 교체"] --> Match["subject_allowed=true"]
    Match --> Listed["tools/list: 7개"]
~~~

### 7.3 승인된 identity 적용

사용자가 실제 Google identity 사용을 선택한 뒤, 검증된 인증 context에서 확인한 정확한 subject로 허용 목록 한 항목을 교체했다. Provider 전체 허용, 검증 생략 또는 거절된 사용자의 자동 등록으로 해결하지 않았다.

원문은 로컬에서만 다뤘다. 작업 중 일회성 평문 캡처를 사용한 뒤 캡처 파일과 임시 코드를 제거했다. 이 캡처 방식은 최종 운영 기능으로 남기지 않았다. 제거 후 **디스크 코드·파일 정리**는 확인했지만, 이미 실행 중인 프로세스가 제거된 코드를 다시 로드했는지는 마지막 기록만으로 확인되지 않는다.

허용 목록 변경 후 재기동된 운영 프로세스의 실제 요청에서 subject와 allowlist 지문이 일치하고 `subject_allowed=true`였으며, 당시 `always_visible` 정책 기준 `tools/list`가 7개를 반환했다. 따라서 도구 0개 현상의 확인된 원인은 **로그인 identity와 allowlist의 불일치**다. 이후 provider-aware routing을 적용하면서 application 도구 직접 고정 노출을 제거해 현재 public surface는 3개의 synthetic tool로 바뀌었다. 두 시점의 도구 개수를 같은 정책의 결과로 혼동하지 않는다.

## 8. 검증 방법과 결과

| 검증 대상 | 확인 방법 | 기록된 결과와 한계 |
|---|---|---|
| 로컬 MCP 인증 요구 | 인증정보 없는 `/mcp` 요청 | 401 및 `WWW-Authenticate` 확인 |
| 공개 OAuth 경로 | metadata HTTP 조회 | 필요한 metadata의 200 응답 확인 |
| MCP 비공개 경계 | Funnel host의 `/mcp` 조회 | 404 확인; 전체 네트워크 보안 검사와는 구분 |
| 실제 설정 반영 | 오래된 Process 값 주입 후 launcher 재실행 | listener의 challenge·resource가 새 설정으로 바뀜 |
| Tunnel discovery | doctor, OAuth 상태 및 Harpoon target 확인 | 필수 진단 통과, 해당 구성의 자동 target 확인 |
| Resource alias | 허용 alias와 다른 host로 `/authorize` 요청 | 허용값은 consent, 다른 host는 `invalid_target` |
| Subject 권한 복구 | 같은 운영 요청의 context와 catalog 대조 | `subject_allowed=false`/0개에서 `true`/당시 7개로 전환 |
| 현재 public surface | 인증된 최상위 `tools/list`와 클라이언트 metadata 대조 | `search_tools`, `get_tool_schema`, `call_tool` 3개 |
| Progressive discovery | 실제 ChatGPT에서 search → schema → call | compact search, selected schema, stdio 응용프로그램 읽기 성공 |
| Provider 경계 | 다른 app의 schema/call을 의도적으로 요청 | upstream 전송 전에 provider mismatch로 거부 |
| HTTP provider 원격 호출 | provider-aware search와 request-context call | MCP provider 경로 성공; application data 성공으로 확대하지 않음 |
| 자동 테스트 | 인증·routing·HTTP fixture·로깅을 포함한 전체 pytest | 35 passed, 2 skipped |
| 장애 복구 | provider/bridge 중단·재기동 중 Gateway 유지 | 다른 provider 정상, 복구 후 Gateway 재시작 없이 재호출 성공 |
| 최종 listener 상태 | Gateway·upstream listener 확인 | 검증 종료 시 모두 listen 상태 |
| 코드·임시물 정리 | 임시 시험 파일과 Git 상태 확인 | 일회성 산출물 제거, Git clean 확인 |

최종 public tool 목록은 다음과 같다.

~~~text
search_tools
get_tool_schema
call_tool
~~~

이전의 Blender namespace 5개 + wrapper 2개 구성은 subject 불일치 해결 직후의 직접 노출 정책이었다. 최종 provider-aware 구성에서는 application tool을 `always_visible`로 고정하지 않는다.

현재 원격 호출 흐름은 다음과 같다.

~~~text
ChatGPT
→ Secure MCP Tunnel
→ 인증된 FastMCP Gateway
→ search_tools(app, query, detail=brief)
→ get_tool_schema(app, names, detail=detailed|full)
→ call_tool(app, name, arguments)
→ provider ownership + scope 확인
→ ProxyProvider
→ upstream MCP
~~~

서버의 Tool schema를 변경한 직후 기존 ChatGPT 세션에는 이전 metadata가 남아 있었다. 연결의 Tool metadata를 새로고침한 뒤 `get_tool_schema`를 포함한 최신 3-tool schema가 반영됐고, 위 흐름을 실제 클라이언트에서 다시 확인했다.

---

## 9. 주요 이슈와 해결 과정

### 9.1 공개 OAuth 주소의 DNS 실패

**이슈 내용과 발생 시점:** 브라우저 callback에 사용할 공개 도메인을 확인하는 과정에서 DNS·접속 실패가 발생했다.

**발생 원인 추적:** `NXDOMAIN`과 이름 해석 실패를 확인했다. 방문해야 할 OAuth endpoint로 사용할 수 없는 주소였다.

**해결 과정:** Tailscale Funnel의 유효한 공개 HTTPS origin을 OAuth base/callback에 적용하고, MCP 본체는 Tunnel에 유지했다. Auth0 API Identifier는 별도 audience로 보존했다.

**검증:** 공개 metadata의 200 응답과 공개 `/mcp`의 404 응답을 확인했다.

**결과:** 공개 OAuth 접근과 비공개 MCP 전달 경로를 분리했다.

### 9.2 Tunnel health 포트 충돌

**이슈 내용과 발생 시점:** launcher 재실행에서 `health_listener` bind가 실패하고 Gateway 정리 경로로 종료됐다.

**발생 원인 추적:** health 포트의 실제 점유 PID와 command line·프로필을 확인해 기존 Tunnel 프로세스와의 충돌을 구분했다.

**해결 과정:** 중복 실행 방지와 소유 프로세스 정리 흐름을 보완했다. 다른 용도의 Tunnel이나 응용프로그램은 종료 대상으로 합치지 않았다.

**검증:** 재실행의 `health_listener` 통과, 필수 doctor 검사와 listener/readiness를 확인했다.

**결과:** 해당 실행 충돌을 해소했으며, 이후 OAuth 실패와는 별개 문제로 분리했다.

### 9.3 설정 수정과 runtime 불일치

**이슈 내용과 발생 시점:** `.env`를 수정했는데도 실제 metadata는 이전 공개 resource를 광고했다.

**발생 원인 추적:** launcher는 기존 Process 값을 보존했고 FastMCP는 `--skip-env`로 실행됐다. 별도 preflight 프로세스와 운영 launcher의 환경값이 달랐다.

**해결 과정:** 전용 launcher의 관리 대상 키를 `.env` 값으로 매 실행 덮어썼다.

**검증:** 이전 값을 의도적으로 주입한 뒤 실제 `/mcp` challenge와 로컬 metadata를 확인했다. 로컬 resource 및 metadata source가 잡히고 Harpoon target 두 개가 등록됐다.

**결과:** 수정 파일과 운영 프로세스의 실제 설정이 일치했다. 이전 `unsupported channel "harpoon"` 현상도 이 단계의 target 미등록 상태와 연결해 구분했다.

### 9.4 Authorization resource 불일치

**이슈 내용과 발생 시점:** discovery 이후 `/authorize`에서 `invalid_target`이 발생했다.

**발생 원인 추적:** 요청의 Tunnel resource와 FastMCP의 로컬 canonical resource가 달랐다.

**해결 과정:** 승인된 Tunnel alias를 정규화·대조한 경우에만 canonical resource로 변환해 기존 검증에 전달했다.

**검증:** 허용 alias의 consent 진행, 다른 host의 거부 및 사용자 재연결 성공을 확인했다.

**결과:** resource 검사 자체를 해제하지 않고 해당 authorization 경로를 연결했다.

### 9.5 인증 후 도구 0개

**이슈 내용과 발생 시점:** OAuth 연결·`tools/list` 처리 성공에도 ChatGPT에 사용할 도구가 없었다.

**발생 원인 추적:** 실제 최상위 응답이 0개였고, scope가 있어도 전역 subject 검사에서 모든 도구가 거절됐다. Auth0 DB identity와 Google identity의 불일치를 확인했다.

**해결 과정:** 사용자 승인 후 정확한 Google subject로 allowlist를 교체하고 재기동했다.

**검증:** 같은 운영 요청에서 `subject_allowed=true`, 일치하는 지문 및 도구 7개 반환을 확인했다.

**결과:** 서버 측 권한 필터에 의한 빈 목록이 해소됐다. HTTP/RPC 성공과 허용된 도구 목록 반환을 별도로 판정했다.

### 9.6 내용 차이 없는 Git 수정 표시

**이슈 내용과 발생 시점:** 임시 코드 제거 뒤 대상 파일은 `M`으로 표시됐지만 일반 diff가 비어 있었다.

**발생 원인 추적:** HEAD/index/정규화된 worktree blob이 같고 `diff --quiet`는 0이었다. `core.autocrlf=true`와 함께 index의 파일 크기·시각 정보와 실제 파일 정보가 달랐다. LF/CRLF 변환에 따른 크기 차이와 일치하는 상태였다.

**해결 과정:** `update-index --really-refresh`만으로 해소되지 않아, 내용 동일성을 확인한 해당 파일만 `git add`로 재등록했다. 저장소 전체 줄바꿈 설정이나 사용자 변경을 일괄 초기화하지 않았다.

**검증:** worktree diff와 staged diff가 모두 0이고 `git status`가 clean인 것을 확인했다. 전체 pytest도 exit 0이었다.

**결과:** 코드 변경 없이 index 상태가 정리됐으므로 추가 코드 커밋은 만들지 않았다.

## 10. 완료 범위와 남은 검증 경계

이 작업에서는 **공개 OAuth 구성 → Tunnel discovery → resource alias 처리 → subject 권한 복구 → provider-aware public surface → 원격 search/schema/call**까지 연결했다. 마지막 상태에서 Gateway와 두 upstream도 다시 기동된 상태로 확인했다.

완료된 현재 경계는 다음과 같다.

- 인증된 ChatGPT 연결에서 3개의 public tool metadata 확인
- app별 compact search와 selected schema 조회
- app/provider ownership을 벗어난 schema/call 거부
- stdio provider의 실제 응용프로그램 읽기 호출
- HTTP provider의 인증된 MCP 호출
- provider/bridge 장애 중 다른 provider 유지 및 무재시작 복구
- 전체 회귀 suite 35 passed, 2 skipped
- routing·progressive disclosure 적용 후 Git clean

다음은 이번 기록만으로 완료를 주장하지 않는 범위다.

| 경계 | 기록상 상태 |
|---|---|
| 현재 ChatGPT 연결의 실제 쓰기 권한 거부 | OAuth fixture의 scope 거부는 검증했지만 최종 원격 클라이언트의 별도 쓰기 부정 시험은 수행하지 않음 |
| 변경 후 다른 계정·축소 scope의 실제 ChatGPT 거부 | 이전 subject 불일치와 fixture는 존재하나 현재 최종 policy의 별도 원격 부정 시험과 구분 |
| Token 만료 후 refresh와 장기 세션 유지 | 관련 설정은 유지했으나 실제 만료 후 refresh 실증 근거 없음 |
| 현재 provider-aware 경로의 HTTP headless 응용프로그램 데이터 | provider search와 MCP request-context 호출은 성공; application scene/Console의 이전 검증과 구분 |

따라서 이 문서의 현재 종료 상태는 **OAuth 연결과 인증된 provider-aware tool discovery/call의 정상화**다. 목록·MCP debug 호출을 모든 쓰기 권한·세션 수명·응용프로그램 작업의 성공으로 확대하지 않는다.
