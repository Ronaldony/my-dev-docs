# Windows 사용자 영역 파일 손실 진단 및 Reset 기반 복구

## 1. 목적과 범위

Windows 환경에서 원격 화면 지연, 배경화면 소실, 작업표시줄 고정 항목 소실, 터미널 외형 변화, 개발 도구 탐색 실패가 동시에 관찰된 사고를 분석하고, 원인 분리부터 데이터 보존과 Windows Reset까지 실제 수행한 복구 흐름을 정리한다.

이 문서의 범위는 다음과 같다.

- Windows Event Log와 PowerShell 실행 기록을 이용한 사고 시점·원인 추적
- 커널·GPU·RDP·네트워크 오류와 사용자 파일 손실의 원인 분리
- PowerShell에서 `cmd.exe`를 호출할 때 발생한 인용부호 해석 오류 분석
- 사용자 프로필 상태와 사용자별 파일·앱·터미널 설정 손실 검증
- VSS, System Restore, Windows Backup, File History, USN Journal을 이용한 rollback 가능성 평가
- 별도 물리 디스크를 이용한 복구 전 데이터 백업과 무결성 검증
- Windows 11 Enterprise의 Cloud Reset 수행과 초기화 후 기준선 검증
- 재초기화 과정에서 발생한 WinRE/Reset 문제의 복구

실제로 수행하지 않은 USB 클린 설치나 이후 재구성 계획은 포함하지 않는다.

---

## 2. 전체 조사 흐름

초기에는 개발 작업 직후 Windows 이상이 체감되어 두 현상을 하나의 원인으로 볼 가능성이 있었다. 실제 분석에서는 시간 순서와 증거를 기준으로 원인을 분리했다.

~~~mermaid
flowchart TD
    A["복합 증상 발생"] --> B["발생 시점 재검증"]
    B --> C["System / Application / RDP 이벤트 분석"]
    C --> D["기존 GPU·RDP·네트워크 장애 이력 분리"]
    C --> E["PowerShell 실행 기록 역추적"]

    E --> F["삭제 명령 인용부호 오류 확인"]
    F --> G["사용자별 파일·설정 손실 검증"]

    G --> H["Rollback source 조사"]
    H --> I["VSS / Restore / Backup / USN 한계 확인"]
    I --> J["별도 물리 디스크로 데이터 백업"]
    J --> K["SHA-256 및 NTFS stream 검증"]
    K --> L["Windows Cloud Reset"]
    L --> M["초기화 후 OS·드라이버·데이터 검증"]
    M --> N["프로필 명명 문제로 재초기화"]
    N --> O["OOBE 도달 및 최종 상태 확인"]

    D --> P["원격 연결 불안정성은 별도 원인 후보로 유지"]
~~~

핵심은 **파일 손실 사고와 기존 원격 연결·드라이버 불안정성을 동일 원인으로 단정하지 않은 것**이다.

---

## 3. 초기 증상과 시간 순서 재구성

### 3.1 관찰된 증상

동일한 Windows 세션에서 다음 현상이 복합적으로 관찰됐다.

- 원격 화면의 jitter 또는 반복적인 연결 끊김
- 기존 배경화면이 표시되지 않음
- 작업표시줄에서 기존 고정 항목 대부분이 사라짐
- 명령 프롬프트가 이전과 다른 Console Host 형태로 표시됨
- 기존에 사용하던 일부 개발 도구가 PATH에서 발견되지 않음

초기에는 개발 workspace 경로 정리 직후 발생한 것으로 인식됐지만, 실제 시각에는 착오 가능성이 있었기 때문에 커밋 시각·PowerShell 실행 기록·Event Log를 별도로 대조했다.

### 3.2 개발 작업 이전에 존재한 시스템 장애

시스템 중단 로그에서는 개발 workspace 정리보다 앞선 시점에 다음 장애가 확인됐다.

~~~text
NVIDIA GPU TDR / Bus Reset
        ↓
Remote Desktop Indirect Display Driver 오류
        ↓
Realtek 무선 네트워크 오류
        ↓
0x133 DPC_WATCHDOG_VIOLATION
        ↓
재부팅
~~~

확인된 주요 이벤트는 다음과 같다.

| 계층 | 확인된 이벤트 |
|---|---|
| GPU | `nvlddmkm` Event ID 153, BusReset TDR |
| RDP 그래픽 | DriverFrameworks-UserMode Event ID 10121, `RdpIdd.dll` |
| 무선 네트워크 | `RtlWlanu` Event ID 5002 |
| 커널 | BugCheck `0x133 DPC_WATCHDOG_VIOLATION` |

또한 Terminal Services 로그에는 workspace 정리보다 이전부터 Event ID 40의 세션 종료가 반복적으로 존재했고, `0x80070079` 및 TCP read/write timeout 계열 오류가 관찰됐다.

따라서 다음 사실을 분리했다.

- **확인된 사실:** 원격 연결·GPU·네트워크 불안정성은 workspace 정리 이전에도 존재했다.
- **해석:** 파일 손실 사고가 모든 원격 연결 문제의 최초 원인이라고 볼 근거는 없다.

---

## 4. PowerShell 실행 기록에서 삭제 사고 원인 추적

### 4.1 Windows PowerShell Event ID 400

Windows PowerShell 로그의 Event ID 400에는 당시 실행된 `HostApplication` 전체 명령이 기록되어 있었다.

문제가 된 패턴은 다음과 같은 형태였다.

~~~powershell
cmd /c "rmdir /s /q \"<LEGACY_WORKSPACE>\""
~~~

의도는 이전 workspace 하나만 재귀적으로 제거하는 것이었다.

그러나 PowerShell에서 백슬래시 `\`는 큰따옴표의 이스케이프 문자가 아니다. 실제 PowerShell parser로 해당 실행 문자열을 **데이터로만 파싱**한 결과, `cmd`의 인수는 다음과 같이 분리됐다.

~~~text
argument 0: cmd
argument 1: /c
argument 2: rmdir /s /q \
argument 3: <LEGACY_WORKSPACE>\
~~~

즉, 삭제 대상 앞에 **독립된 `\` 인수**가 생성됐다.

### 4.2 안전한 전달 검증

과거 삭제 명령을 재실행하지 않고 `rmdir`을 `echo`로 치환해 native argument 전달만 확인했다.

~~~text
ARGUMENTS_ONLY \ <LEGACY_WORKSPACE>\
~~~

따라서 PowerShell parser 단계뿐 아니라 `cmd.exe` 경계에서도 독립된 `\`이 전달되는 것을 확인했다.

Windows 명령 셸에서 `\`은 현재 드라이브의 root를 의미할 수 있다. `rmdir /s /q`와 결합되면 의도한 workspace 외의 root tree를 삭제 대상으로 해석할 위험이 생긴다.

### 4.3 사실과 해석

**확인된 사실**

- PowerShell 실행 로그에 잘못 인용된 `cmd /c rmdir` 호출이 남아 있었다.
- parser와 echo-only 검증에서 독립된 `\` 인수가 확인됐다.
- 동일 시점 이후 사용자별 파일과 응용프로그램 데이터가 광범위하게 사라진 정황이 확인됐다.

**확정하지 못한 부분**

- 당시 `cmd.exe`의 정확한 current drive/current directory를 독립적인 감사 기록으로 복원하지 못했다.
- 파일별 삭제 성공 목록은 확보하지 못했다.

따라서 **독립된 root 인수와 사용자 파일 손실의 결합은 강한 원인 설명력을 갖지만, 삭제된 모든 파일을 명령 단위로 전수 입증한 것은 아니다.**

---

## 5. 사용자 프로필 손실 검증

### 5.1 임시 프로필 가설 제외

`Win32_UserProfile`을 조회한 결과 현재 프로필은 정상 로드 상태였다.

~~~text
Loaded = True
Status = 0
~~~

전형적인 User Profile Service 임시 프로필 오류 이벤트도 확인되지 않았다.

따라서 문제는 **다른 임시 프로필로 로그인한 것**보다, 정상 프로필 내부의 일부 파일·설정이 사라진 상황에 가까웠다.

### 5.2 레지스트리는 남고 파일은 사라진 부분 손상

다음과 같은 조합이 확인됐다.

| 영역 | 확인 결과 |
|---|---|
| Documents / Downloads | 사용자별 known folder 경로는 등록되어 있으나 실제 디렉터리가 없음 |
| 사용자 Git 설정 | 사용자별 설정 파일이 없음 |
| 사용자 설치 앱 | uninstall 등록은 남아 있으나 설치 디렉터리와 실행 파일이 없음 |
| Windows Terminal | 패키지는 설치 상태지만 사용자 설정 파일과 실행 alias가 없음 |
| 배경화면 | 레지스트리가 참조하는 이미지 파일이 없음 |
| Theme cache | 사고 이후 새로 생성된 빈 theme 관련 디렉터리 확인 |
| 작업표시줄 | Taskband 데이터 일부는 남아 있으나 사용자 pinned directory가 비어 있음 |

이 구조는 **registry hive 전체가 사라진 것이 아니라, profile 내부의 일반 파일과 사용자별 앱 데이터가 부분적으로 삭제된 상황**을 설명한다.

### 5.3 현재 존재하는 파일을 사고 생존 증거로 보지 않음

사고 직후 한 개발 도구가 다시 존재한다는 이유로 삭제되지 않았다고 판단할 수 없었다.

PowerShell 실행 기록에서 해당 도구의 설치 스크립트가 **사고 이후 다시 실행된 것**을 확인했고, 현재 실행 파일의 생성 시각도 그 이후였다.

따라서 사고 분석에서는 다음 원칙을 적용했다.

> 현재 존재 여부와 사고 당시 생존 여부를 분리하고, 설치·생성 시각과 실행 기록을 함께 확인한다.

---

## 6. Rollback 가능성 평가

Windows 전체를 사고 이전 시점으로 되돌릴 수 있는지 다음 항목을 순차적으로 확인했다.

~~~mermaid
flowchart LR
    A["System Restore"] -->|없음| B["VSS Shadow Copy"]
    B -->|없음| C["Windows Backup catalog"]
    C -->|없음| D["WindowsImageBackup / File History"]
    D -->|없음| E["USN Journal"]
    E -->|사고 시점 기록 없음| F["원본 파일 undelete 가능성만 잔존"]
    F --> G["데이터 보존 후 OS 재구성"]
~~~

### 6.1 확인 결과

| 복구 수단 | 결과 |
|---|---|
| System Restore point | 현재 0개 |
| VSS shadow copy | 현재 0개 |
| `wbadmin get versions` | 사용 가능한 Windows backup 없음 |
| `WindowsImageBackup` | 확인되지 않음 |
| File History | 사용 가능한 구성·백업 확인되지 않음 |
| Recycle Bin | 사용자 복원 메타데이터 없음 |
| USN Journal | 사고 이후 시점의 기록만 보존 |
| Windows.old | 없음 |

System Restore 관련 이벤트에서는 과거 restore point 생성 성공 이력과 이후 shadow copy 저장 한도 때문에 오래된 복사본이 삭제되거나 생성이 중단된 이력이 확인됐다.

즉 **복원 기능이 처음부터 꺼져 있던 것이 아니라, 실제 사고 분석 시점에는 사용할 수 있는 과거 snapshot이 남아 있지 않았다.**

### 6.2 USN Journal의 한계

관리자 권한으로 USN Journal을 export했지만 보존 범위의 첫 시각이 사고 이후였다.

따라서 USN Journal은 다음 용도로는 사용할 수 없었다.

- 사고 시점 삭제 파일 전체 목록 복원
- 삭제 파일 내용 복원
- 삭제 명령이 성공한 각 파일의 전수 입증

OS 볼륨은 SSD였고 NTFS TRIM notification이 허용된 상태였으므로, 삭제 파일의 raw recovery 가능성은 파일별로 보장할 수 없었다.

---

## 7. 복구 전 데이터 보존

### 7.1 물리 디스크 분리 전략

OS 재구성 전에 복구 대상 데이터와 백업 사본을 서로 다른 물리 디스크에 배치했다.

~~~mermaid
flowchart LR
    OS["Physical Disk A<br/>Windows OS volume"]
    DATA["Physical Disk B<br/>Persistent data"]
    BACKUP["Physical Disk C<br/>Verified backup"]

    DATA -->|"Robocopy"| BACKUP
    OS -. "Reset target only" .-> OS
    DATA -. "Preserve" .-> DATA
    BACKUP -. "Preserve" .-> BACKUP
~~~

보존 대상은 인증서 관련 데이터, 개발 workspace, 작업 데이터였다.

### 7.2 Robocopy 복사

실제 복사에는 다음 특성을 사용했다.

~~~text
/E
/COPY:DAT
/DCOPY:DAT
/SJ
/SL
/XJ
/R:2
/W:1
~~~

목적은 다음과 같다.

- 빈 디렉터리를 포함해 하위 트리 복사
- file data/attribute/time 보존
- directory data/attribute/time 보존
- junction/symbolic link를 target tree로 확장하지 않고 link 자체로 취급
- 재시도 횟수 제한

### 7.3 무결성 검증

단순 복사 완료 코드만 신뢰하지 않고 별도 검증을 수행했다.

검증 항목:

- source/destination 상대 경로 목록
- 일반 파일 수
- 파일 byte length
- 모든 일반 파일 SHA-256
- NTFS named data stream 목록·크기·SHA-256
- junction/link target
- 검증 전후 source inventory 변화 여부

최종 full snapshot에서는 모든 대상 일반 파일과 named data stream이 일치했고, 검증 도중 source tree가 바뀌지 않은 것을 확인했다.

### 7.4 변경 중인 source 처리

첫 검증에서는 개발 프로세스의 build output과 SQLite WAL/SHM 파일이 계속 변경되어 hash mismatch가 발생했다.

이에 따라 다음 순서로 처리했다.

1. 변경 작업 중지
2. 전체 source를 다시 복사
3. SQLite database는 online backup API로 별도 일관 snapshot 생성
4. `quick_check` 결과 `ok` 확인
5. full file hash 검증 재실행
6. source inventory가 검증 전후 동일한지 확인

이 절차를 통해 **복사 성공과 시점 일관성을 분리해 검증**했다.

---

## 8. 첫 번째 Windows Cloud Reset

Rollback source가 없고 사용자 영역 손실 범위가 넓었기 때문에 Windows 자체를 새 기준선으로 재구성했다.

Settings의 Reset this PC에서 실제 적용한 옵션은 다음과 같다.

~~~text
Remove everything
        +
Cloud download
        +
Delete files from Windows drive only
        +
Do not clean the drive
~~~

데이터 및 백업용 물리 디스크는 reset target에서 제외했다.

### 8.1 초기화 완료 확인

초기화 후 다음을 직접 확인했다.

- Windows 설치 시각이 초기화 시점으로 갱신됨
- 새 사용자 프로필이 생성됨
- 기존 사용자 프로필 경로가 사라짐
- Windows 11 Enterprise 활성화 상태 유지
- OS 볼륨의 free space가 초기화 전보다 크게 증가
- Reset 관련 process가 종료됨
- Persistent data volume과 backup volume이 그대로 존재함
- Reset log에서 OOBE와 user logon 완료 확인

---

## 9. 초기화 후 기준선 검증

### 9.1 OS·스토리지·PnP

초기화 직후 기준선은 다음과 같았다.

- 모든 물리 디스크: Healthy / Online
- 모든 주요 volume: Healthy / OK
- Present PnP device 중 비정상 상태: 0
- Windows Defender 실시간 보호: 활성
- Windows Firewall: Domain / Private / Public 활성
- Windows Update 관련 service: 동작 상태

Defender signature는 초기 이미지에 포함된 오래된 상태였기 때문에 OS 정상성과 signature 최신성은 별개로 판단했다.

### 9.2 GPU·네트워크 드라이버

초기화 이후 GPU와 주요 네트워크 장치는 Device Status가 `OK`였고, 직후 Event Log에서 다음 오류는 재발하지 않았다.

- `nvlddmkm`
- Display / DxgKrnl
- RDP indirect display
- Realtek WLAN driver error

다만 일부 드라이버 버전은 초기화 이전과 동일했다. 따라서 **Reset 성공이 드라이버 문제 자체를 교체·해결했다는 증거는 아니다.**

### 9.3 두 Wi-Fi adapter의 경로 차이

두 개의 Wi-Fi adapter가 동일 subnet에 동시에 연결되고 둘 다 default gateway를 갖는 상태를 확인했다.

- 낮은 interface metric을 가진 adapter가 기본 경로로 우선 선택됨
- 두 adapter 모두 짧은 ping에서는 packet loss 0%
- 한 adapter는 1~4 ms 범위로 안정적
- 다른 adapter에서는 100 ms 이상의 latency spike가 관찰됨

따라서 원격 화면 jitter 분석에서는 **OS 파일 손실과 별도로 네트워크 adapter 선택과 latency variation을 추적해야 한다**는 근거가 남았다.

### 9.4 RDP

Windows Reset 후 Remote Desktop Services는 기본 비활성 상태였다.

~~~text
TermService = Stopped
fDenyTSConnections = 1
~~~

따라서 초기화 직후에는 실제 Windows RDP live session을 이용한 재현 시험을 수행하지 않았다.

---

## 10. 개발 데이터와 실행환경 분리 검증

Persistent data volume의 프로젝트 데이터는 유지됐지만, OS 볼륨에 의존하던 실행환경은 그대로 사용할 수 없었다.

대표적으로 기존 Python virtual environment의 `pyvenv.cfg`가 삭제된 이전 사용자 profile 아래의 Python runtime을 참조하고 있었다.

실행 결과는 다음과 같은 형태였다.

~~~text
uv trampoline failed to spawn Python child process
entity not found
~~~

반면 Node.js workspace의 package metadata, `node_modules`, workspace junction, native module은 정상적으로 보존된 것을 확인했다.

이 검증에서 확인된 원리는 다음과 같다.

~~~text
Persistent project data
        ≠
Portable execution environment
~~~

특히 Python virtual environment처럼 interpreter 절대 위치를 내부에 기록하는 환경은 OS/profile 재생성 후 재구성이 필요하다.

---

## 11. 사용자 프로필 이름 문제와 두 번째 Reset

첫 Cloud Reset의 OOBE에서 조직 계정으로 바로 로그인하면서 사용자 profile directory가 로컬 언어 기반 이름으로 생성됐다.

기술적으로 Windows 자체는 정상 동작했지만 개발 환경의 경로 일관성을 위해 영문 local profile을 먼저 생성하려는 요구가 발생했다.

이를 위해 두 번째 Reset을 진행했다.

### 11.1 Reset 진입 실패

초기에는 `SystemSettingsAdminFlows.exe FeaturedResetPC`를 직접 호출하면 Reset UI가 즉시 다음 오류를 표시했다.

~~~text
There was a problem resetting your PC.
No changes were made.
~~~

### 11.2 원인 추적

다음 항목을 확인했다.

- DISM `/CheckHealth`: component store corruption 없음
- DISM `/ScanHealth`: corruption 없음
- SFC `/verifyonly`: integrity violation 없음
- WinRE: Enabled
- recovery partition: 정상 존재
- ReAgent.xml: recovery partition을 정상 참조

OS component corruption은 확인되지 않았다.

### 11.3 WinRE 재등록

Reset path의 recovery registration을 다시 구성하기 위해 다음 작업을 수행했다.

~~~cmd
reagentc /disable
reagentc /enable
reagentc /info
~~~

재등록은 성공했고 WinRE용 BCD identifier가 새로 생성됐다.

이후 직접 executable 호출 대신 **Settings → System → Recovery → Reset this PC**의 정상 UI 경로를 사용했다.

### 11.4 두 번째 Reset 설정

두 번째 Reset에서도 다음 설정을 다시 확인한 뒤 실행했다.

~~~text
Remove apps and files
Do not clean the drive
Delete files from Windows drive only
Cloud download and reinstall Windows
~~~

최종 확인 화면에서 reset을 실행했고 Cloud download 진행 상태를 확인했다.

두 번째 초기화는 OOBE까지 도달했다.

---

## 12. 주요 이슈 정리

### 12.1 삭제 명령의 인용부호 오류

**이슈 내용과 발생 시점**

workspace 경로 정리 과정에서 사용자별 파일·앱·설정이 광범위하게 사라졌다.

**발생 원인 추적**

PowerShell Event ID 400의 `HostApplication`을 분석해 `cmd /c rmdir` 문자열을 복원했다. PowerShell에서 `\"`를 사용한 결과 parser가 독립된 `\` 인수를 생성하는 것을 확인했다.

**해결 과정**

과거 명령을 재실행하지 않고 parser와 echo-only 방식으로 인수 전달만 검증했다. 이후 손상된 OS volume을 직접 수리하기보다 데이터 보존과 Windows Reset을 수행했다.

**검증**

사용자 프로필은 정상 로드됐지만 사용자별 파일·응용프로그램 데이터가 사라진 정황을 확인했다. 삭제 명령의 native argument에도 독립된 `\`이 존재했다.

**결과**

광범위 사용자 영역 손실의 가장 강한 원인으로 삭제 명령 인용 오류를 식별했다. 단, 당시 `cmd.exe`의 exact current drive는 별도 감사 로그로 복원하지 못했다.

---

### 12.2 Rollback source 부재

**이슈 내용과 발생 시점**

사고 이전 상태로 COW snapshot 또는 시스템 이미지 rollback을 시도할 수 있는지 확인해야 했다.

**발생 원인 추적**

VSS, System Restore, Windows Backup, File History, Recycle Bin, USN Journal을 조사했다.

**해결 과정**

관리자 권한으로 shadow copy, restore point, backup catalog, journal retention 범위를 확인하고 관련 로그를 별도 물리 디스크에 보존했다.

**검증**

사용 가능한 snapshot·backup은 없었고 USN Journal도 사고 이후 시점부터만 남아 있었다.

**결과**

완전 rollback이 가능한 local source가 없다고 판단하고, 현재 보존 가능한 데이터를 검증 백업한 뒤 OS를 재구성하는 방식으로 전환했다.

---

### 12.3 변경 중인 개발 데이터의 백업

**이슈 내용과 발생 시점**

첫 백업 검증 중 source tree와 SQLite WAL 파일이 계속 변경되어 hash mismatch가 발생했다.

**발생 원인 추적**

build output, test output, WAL/SHM 등 런타임 변경 파일이 복사와 검증 사이에 갱신됐다.

**해결 과정**

작업을 중지한 뒤 full copy를 다시 수행하고 SQLite online backup과 전체 SHA-256 검증을 실행했다.

**검증**

모든 대상 regular file과 NTFS named data stream이 일치했고 검증 전후 source inventory가 동일했다.

**결과**

OS Reset 전에 시점 일관성이 검증된 독립 backup을 확보했다.

---

### 12.4 Reset this PC 진입 오류

**이슈 내용과 발생 시점**

두 번째 초기화 시 직접 Reset admin flow를 실행하면 옵션 화면에 들어가기 전에 초기화 실패 메시지가 표시됐다.

**발생 원인 추적**

DISM, SFC, WinRE 상태, recovery partition, ReAgent 구성을 순서대로 확인했다.

**해결 과정**

WinRE를 disable/enable 방식으로 재등록하고 BCD 연결을 갱신했다. 이후 Settings의 지원되는 Reset UI 경로로 다시 진입했다.

**검증**

Reset 옵션 화면에서 `Remove everything`, `Cloud download`, Windows volume only, no drive cleaning 설정을 읽어 확인했고 최종 reset 실행 후 download 진행 상태를 확인했다.

**결과**

Reset UI가 정상 동작했고 두 번째 초기화가 OOBE까지 진행됐다.

---

## 13. 최종 확인 상태

이 대화에서 실제 확인된 최종 상태는 다음과 같다.

- 사용자 영역 파일 손실과 기존 원격 연결 불안정성을 별도 문제로 분리함
- 잘못 인용된 PowerShell → `cmd.exe` 삭제 명령을 사용자 영역 손실의 핵심 원인으로 식별함
- 사고 이전 local rollback source는 확보되지 않음
- persistent data를 별도 물리 디스크에 복사하고 파일·stream 단위 무결성을 검증함
- 첫 Windows 11 Enterprise Cloud Reset이 완료됨
- 초기화 후 OS, disk, volume, PnP 기준선이 정상임을 확인함
- OS 재생성 후 기존 virtual environment의 절대 runtime 의존성이 깨지는 것을 확인함
- 사용자 profile directory 명명 문제 때문에 두 번째 Cloud Reset을 수행함
- 두 번째 Reset 과정의 WinRE/Reset 진입 문제를 재등록으로 복구함
- 두 번째 Reset은 OOBE까지 도달함
- 원격 연결 불안정성은 초기 파일 손실과 별개의 원인 후보로 남음

따라서 이 사고에서 가장 중요한 기술적 교훈은 **파일 삭제 명령의 shell boundary와 quoting을 검증하지 않은 상태에서 재귀 삭제를 실행하면, 의도한 workspace 범위를 넘어 시스템·사용자 영역에 영향을 줄 수 있다는 점**과, **복구 전에는 OS 수리보다 먼저 증거와 별도 물리 디스크 백업을 확보해야 한다는 점**이다.
