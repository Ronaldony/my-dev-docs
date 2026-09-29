# Proxmox Host 관리 접근 복구 및 USB Wi-Fi Uplink 영구화

## 1. 목적과 최종 구성

유선 관리망만 구성된 Proxmox VE Host에 USB Wi-Fi uplink를 추가하고, 재부팅 후에도 무선 인증·DHCP·기본 경로가 자동 복구되도록 구성했다.

작업 중에는 Host의 기존 IPv4 관리 주소로 접근할 수 없고 root 비밀번호도 확인되지 않는 상태였기 때문에, 먼저 관리 접근을 복구한 뒤 임시 인터넷 경로를 확보하고 Wi-Fi를 단계적으로 검증했다.

최종 구성의 목표는 다음과 같다.

- Proxmox Host의 유선 Linux bridge는 관리망 전용으로 유지한다.
- USB Wi-Fi 인터페이스가 인터넷 uplink 역할을 담당한다.
- WPA 인증은 인터페이스 전용 systemd `wpa_supplicant` 서비스가 담당한다.
- IPv4 주소와 기본 경로는 DHCP로 획득한다.
- 유선 bridge에는 인터넷용 default gateway를 두지 않는다.
- 재부팅 후에도 Wi-Fi 연결, DHCP 주소, default route가 자동 복구되어야 한다.
- 구축 과정에서 사용한 임시 Windows ICS 경로와 임시 주소는 최종 구성에서 제거한다.

사용한 핵심 기술은 Proxmox VE, GNU GRUB, Linux bridge, ifupdown2, systemd, wpa_supplicant, DHCP, Linux routing, Windows Internet Connection Sharing(ICS), `iw`, `rfkill`이다.

---

## 2. 초기 네트워크 상태 확인

초기 구조는 다음과 같았다.

~~~mermaid
flowchart LR
    N[Notebook] <-->|Wi-Fi| W[Windows endpoint]
    W <-->|Ethernet| P[Proxmox Host]
    P -. "USB Wi-Fi\n미설정" .-> AP[Wireless AP]
    AP --> I[Internet]
~~~

확인된 상태는 다음과 같았다.

- Windows endpoint와 Proxmox Host 사이의 Ethernet link LED는 점등되었다.
- Windows Ethernet은 link up 상태였지만 Proxmox 관리 주소와 다른 subnet에 있었다.
- Proxmox 콘솔에 표시된 과거 IPv4 관리 주소는 실제 활성 주소와 일치하지 않았다.
- IPv4 ARP와 TCP/8006 접근은 실패했다.
- 동일 Ethernet segment에서 Proxmox 계열 가상 NIC와 Host의 IPv6 link-local neighbor는 탐지되었다.
- Host의 IPv6 link-local 주소에 대해 TCP/8006과 HTTPS 응답이 확인되었다.

따라서 Proxmox 서비스 자체가 중단된 것이 아니라, **IPv4 관리 경로와 인증 접근을 먼저 복구해야 하는 상태**로 좁혀졌다.

---

## 3. 로컬 관리 접근 복구

### 3.1 GRUB를 이용한 일회성 root shell 진입

물리 콘솔에서 GNU GRUB 메뉴가 확인되었다.

부팅 엔트리의 kernel command line을 일회성으로 편집하여 다음 옵션을 추가했다.

~~~text
init=/bin/bash
~~~

이 옵션은 정상 부팅 시 PID 1로 실행되는 init/systemd 대신 `/bin/bash`를 직접 실행하도록 kernel에 지시한다.

중요한 특성은 다음과 같다.

- GRUB의 `e` 편집은 해당 1회 부팅에만 적용된다.
- boot configuration 파일 자체를 변경하지 않는다.
- 재부팅하면 `init=/bin/bash` 옵션은 자동으로 사라진다.
- 최소 환경의 root shell을 확보할 수 있다.

### 3.2 root 비밀번호 재설정

초기 root filesystem은 쓰기 제한 상태일 수 있으므로 먼저 읽기/쓰기 가능 상태로 remount했다.

~~~bash
mount -o remount,rw /
passwd root
sync
exec /sbin/init
~~~

각 명령의 역할은 다음과 같다.

| 명령 | 역할 |
|---|---|
| `mount -o remount,rw /` | root filesystem을 쓰기 가능한 상태로 전환 |
| `passwd root` | root 계정 비밀번호 변경 |
| `sync` | 메모리의 변경 내용을 block device에 반영 |
| `exec /sbin/init` | 임시 bash 대신 정상 init 흐름으로 전환 |

정상 부팅 후 root 계정으로 콘솔 로그인이 성공하여 관리 접근이 복구되었다.

---

## 4. 실제 Host 네트워크 구성 확인

관리 접근 복구 후 Host의 link, address, route, USB device와 `/etc/network/interfaces`를 확인했다.

확인된 구조는 다음과 같았다.

~~~mermaid
flowchart TB
    E[Physical Ethernet NIC] --> B[Linux bridge]
    B --> M[Management LAN]
    U[USB Wi-Fi NIC] --> D[rtw_8822bu driver]
    D -. "아직 uplink 미구성" .-> AP[Wireless AP]
~~~

검증 환경의 USB Wi-Fi 장치는 Realtek RTL88x2BU 계열이었고 Linux에서는 `rtw_8822bu` driver로 인식되었다.

네트워크 인터페이스까지 생성되어 있었으므로 문제는 USB 인식이나 kernel driver 부재가 아니라, **무선 인증과 IP/routing 구성이 아직 없는 상태**였다.

---

## 5. 임시 인터넷 경로 확보

Wi-Fi 설정 도구를 설치하고 검증하려면 Host에 먼저 인터넷 경로가 필요했다.

Windows endpoint의 Wi-Fi 인터넷 연결을 Ethernet으로 공유하도록 ICS를 활성화했다.

전환 구조는 다음과 같았다.

~~~mermaid
flowchart LR
    I[Internet] --> AP[Wireless AP]
    AP <-->|Wi-Fi| W[Windows endpoint]
    W -->|ICS / NAT| E[Windows Ethernet]
    E <-->|Temporary private LAN| B[Proxmox bridge]
    B --> P[Proxmox Host]
~~~

Proxmox bridge에는 ICS subnet의 임시 IPv4 주소를 추가하고, Windows Ethernet 주소를 temporary default gateway로 사용했다.

이 단계의 설정은 permanent configuration을 변경하기 전에 연결성을 검증하기 위한 runtime pilot이었다.

검증 순서는 다음과 같았다.

1. Windows ICS gateway와 같은 subnet에 Host 주소 추가
2. default route를 ICS gateway 방향으로 설정
3. 공인 IPv4 ICMP 확인
4. DNS 이름 해석 확인
5. HTTPS 접속 확인

공인 IP 통신과 DNS가 정상화되어 package 설치가 가능한 상태를 확보했다.

---

## 6. Wi-Fi 도구 및 무선 장치 검증

필요한 도구는 다음과 같다.

~~~text
wpa_supplicant
wpa_cli
iw
rfkill
~~~

`wpa_supplicant`와 `wpa_cli`는 이미 존재했고, `iw`와 `rfkill`을 추가 설치했다.

설치 후 다음 항목을 확인했다.

- USB Wi-Fi interface가 `managed` mode로 표시됨
- `rfkill`의 soft block과 hard block이 모두 해제 상태
- 주변 AP scan 성공
- target 5 GHz SSID 탐지 성공

개념적인 확인 명령은 다음과 같다.

~~~bash
iw dev
rfkill list
ip link set <WIFI_IF> up
iw dev <WIFI_IF> scan
~~~

---

## 7. WPA 인증 검증

영구 설정 전에 runtime-only 방식으로 인증을 검증했다.

### 7.1 임시 wpa_supplicant 구성

비밀번호를 shell history나 화면에 직접 노출하지 않도록 입력한 뒤 `wpa_passphrase`로 임시 설정을 생성했다.

~~~bash
read -s WIFI_PSK
wpa_passphrase '<WIFI_SSID>' "$WIFI_PSK" > /run/wpa-usb.conf
unset WIFI_PSK
chmod 600 /run/wpa-usb.conf
~~~

`/run`은 runtime filesystem이므로 이 파일은 reboot persistence를 갖지 않는다.

### 7.2 연결 확인

~~~bash
wpa_supplicant -B -i <WIFI_IF> -c /run/wpa-usb.conf
iw dev <WIFI_IF> link
~~~

정상 연결 시 다음 상태를 확인했다.

~~~text
Connected to <AP_BSSID>
SSID: <WIFI_SSID>
~~~

---

## 8. 이슈: SSID 대소문자 불일치

### 8.1 이슈 내용과 발생 시점

AP scan에서는 target SSID가 보였지만 `wpa_supplicant`가 계속 연결 대상이 없다고 판단했다.

debug log에는 다음 흐름이 반복되었다.

~~~text
SSID '<target>'
skip - SSID mismatch
No suitable network found
~~~

### 8.2 발생 원인 추적

실제 AP의 SSID와 임시 WPA 설정에 입력한 SSID를 비교한 결과 한 문자의 대소문자가 달랐다.

SSID는 case-sensitive string이므로 시각적으로 유사해 보여도 다른 network identifier로 처리된다.

### 8.3 해결 과정

실제 AP에 표시된 SSID 문자열을 그대로 사용하여 WPA 설정을 다시 생성했다.

### 8.4 검증

`wpa_supplicant`에서 association과 key negotiation이 완료되었고 `iw dev <WIFI_IF> link`에서 연결된 SSID가 확인되었다.

### 8.5 결과

driver나 WPA cipher 문제가 아니라 **SSID 문자열 불일치**가 원인이었으며, 정확한 SSID 적용 후 인증이 정상화되었다.

---

## 9. DHCP 및 Wi-Fi 인터넷 경로 검증

WPA 연결 성공 후 DHCP client를 통해 IPv4 주소를 획득했다.

~~~bash
dhclient <WIFI_IF>
ip -br addr show <WIFI_IF>
ip route
~~~

DHCP 주소는 정상적으로 할당되었지만, 최초에는 Wi-Fi default route가 생성되지 않았다.

---

## 10. 이슈: 기존 default route와 Wi-Fi route 충돌

### 10.1 이슈 내용과 발생 시점

Wi-Fi 인터페이스로 DHCP 주소를 받은 뒤 AP gateway에는 도달할 수 있었지만, Wi-Fi interface를 지정한 공인 IP ping은 실패했다.

route 조회 결과 외부 목적지에 대해 Wi-Fi gateway가 next hop으로 사용되지 않고 있었다.

### 10.2 발생 원인 추적

임시 ICS 구성에서 생성한 기존 default route가 이미 존재했다.

DHCP lease에는 Wi-Fi router 정보가 포함되어 있었지만, 활성 route table에서는 기존 default route가 우선하고 있었다.

단순히 interface만 지정하는 것은 gateway route를 생성하지 않는다.

~~~text
Interface binding
    ≠
Gateway selection
~~~

### 10.3 해결 과정

먼저 단일 공인 IP에 대해 host route를 추가하여 Wi-Fi gateway를 명시적으로 통과시키는 pilot을 수행했다.

~~~text
Public IP
   ↓
Wi-Fi gateway
   ↓
USB Wi-Fi interface
   ↓
Internet
~~~

pilot 성공 후 Wi-Fi default route와 임시 ICS default route에 metric을 부여했다.

~~~text
Wi-Fi default route        lower metric
Temporary ICS route        higher metric
~~~

기존 metric 없는 route가 남아 있을 때는 해당 route를 제거하여 우선순위 충돌을 해소했다.

### 10.4 검증

`ip route get <PUBLIC_IP>`에서 다음 구조가 확인되었다.

~~~text
<PUBLIC_IP> via <WIFI_GATEWAY> dev <WIFI_IF> src <WIFI_DHCP_IP>
~~~

공인 IPv4 ping, DNS lookup, HTTPS request가 모두 성공했다.

### 10.5 결과

USB Wi-Fi, WPA 인증, DHCP와 AP NAT는 모두 정상이고 문제는 **기존 default route가 Wi-Fi 경로보다 우선하던 routing 상태**였음이 확인되었다.

---

## 11. Wi-Fi 인증 영구화

runtime 검증 후 WPA 설정을 persistent 위치로 이동했다.

구성 파일은 인터페이스 전용 systemd template unit의 규칙에 맞춰 다음 형태로 관리했다.

~~~text
/etc/wpa_supplicant/wpa_supplicant-<WIFI_IF>.conf
~~~

구성 개념은 다음과 같다.

~~~text
ctrl_interface=/run/wpa_supplicant
update_config=0
country=<REGULATORY_DOMAIN>

network={
    ssid="<WIFI_SSID>"
    psk=<HASHED_PSK>
}
~~~

plain-text PSK 주석은 제거하고 파일 permission을 root 전용으로 제한했다.

---

## 12. 이슈: ifupdown2의 legacy wpa_supplicant hook 실패

### 12.1 이슈 내용과 발생 시점

초기 영구화 방식에서는 `/etc/network/interfaces`에 `wpa-conf`를 지정했다.

`ifreload -a`와 `ifup --verbose` 실행 시 다음 단계에서 실패했다.

~~~text
/etc/network/if-pre-up.d/wpasupplicant
returned 1
~~~

### 12.2 발생 원인 추적

Wi-Fi driver와 WPA 인증은 이미 수동으로 정상 동작하고 있었으므로 wireless stack 자체의 실패는 아니었다.

실패는 ifupdown2가 legacy `if-pre-up.d/wpasupplicant` hook을 호출하는 integration 지점에 한정되어 있었다.

### 12.3 해결 과정

역할을 분리했다.

~~~mermaid
flowchart TB
    S[systemd] --> W["wpa_supplicant@<WIFI_IF>.service"]
    W --> A[WPA authentication / association]
    F[ifupdown2] --> D[DHCP address and route]
~~~

`/etc/network/interfaces`에서는 `wpa-conf` 항목을 제거하고 DHCP 관리만 남겼다.

~~~text
auto <WIFI_IF>
iface <WIFI_IF> inet dhcp
~~~

WPA 인증은 다음 interface-specific service가 담당하도록 했다.

~~~text
wpa_supplicant@<WIFI_IF>.service
~~~

### 12.4 검증

서비스를 enable/start한 뒤 다음 상태가 확인되었다.

~~~text
Active: active (running)
WPA: Key negotiation completed
CTRL-EVENT-CONNECTED
~~~

`iw dev <WIFI_IF> link`에서도 target SSID association이 확인되었다.

### 12.5 결과

최종 구성에서는 **systemd가 WPA lifecycle을 담당하고 ifupdown2가 DHCP를 담당하는 구조**로 분리되었다.

---

## 13. 이슈: wpa_supplicant country 설정 오타

### 13.1 이슈 내용과 발생 시점

interface-specific systemd service의 최초 기동이 exit code 255로 실패했다.

### 13.2 발생 원인 추적

journal에서 regulatory domain 설정 key에 오타가 확인되었다.

~~~text
unknown global field
Invalid configuration line
~~~

### 13.3 해결 과정

잘못된 key를 올바른 `country=<COUNTRY_CODE>` 형식으로 수정했다.

### 13.4 검증

서비스가 `active (running)` 상태가 되었고 association, WPA key negotiation, connection event가 연속으로 확인되었다.

### 13.5 결과

systemd unit이나 driver 문제가 아니라 **wpa_supplicant configuration syntax 오류**였음이 확인되었다.

---

## 14. 최종 persistent 네트워크 구조

최종 역할 분리는 다음과 같다.

~~~mermaid
flowchart TB
    AP[Wireless AP / Router]
    WIFI[USB Wi-Fi Interface]
    WPA[systemd wpa_supplicant]
    DHCP[ifupdown2 + DHCP]
    HOST[Proxmox Host]
    BR[Linux Bridge]
    ETH[Physical Ethernet]
    MGMT[Wired Management Network]
    NET[Internet]

    WPA -->|Authenticate| WIFI
    AP <-->|802.11| WIFI
    WIFI --> DHCP
    DHCP --> HOST
    AP --> NET

    HOST --> BR
    BR --> ETH
    ETH --> MGMT
~~~

핵심 `/etc/network/interfaces` 구조는 다음과 같다.

~~~text
auto lo
iface lo inet loopback

iface <ETHERNET_IF> inet manual

auto <WIFI_IF>
iface <WIFI_IF> inet dhcp

auto <BRIDGE_IF>
iface <BRIDGE_IF> inet static
        address <MANAGEMENT_IP>/<PREFIX>
        bridge-ports <ETHERNET_IF>
        bridge-stp off
        bridge-fd 0

source /etc/network/interfaces.d/*
~~~

구조적 원칙은 다음과 같다.

- Wi-Fi interface는 Linux bridge의 port로 사용하지 않는다.
- Wi-Fi는 Host uplink 역할만 담당한다.
- 유선 bridge는 관리망을 담당한다.
- 유선 bridge에는 인터넷용 default gateway를 두지 않는다.
- Wi-Fi DHCP가 default route를 제공한다.
- WPA 인증은 `wpa_supplicant@<WIFI_IF>.service`가 담당한다.

서비스는 boot 시 자동 시작하도록 enable 상태로 구성했다.

---

## 15. 재부팅 persistence 검증

Host reboot 후 다음 순서로 검증했다.

~~~bash
systemctl is-active wpa_supplicant@<WIFI_IF>.service
iw dev <WIFI_IF> link
ip -br addr show <WIFI_IF>
ip route
ping -c 3 <PUBLIC_IP>
curl -4I https://deb.debian.org
~~~

확인된 결과는 다음과 같다.

- interface-specific `wpa_supplicant` service: active
- target SSID: 자동 재연결
- Wi-Fi interface: DHCP IPv4 자동 획득
- default route: Wi-Fi router 방향으로 자동 구성
- 공인 IPv4 ICMP: packet loss 0%
- HTTPS: HTTP 200 응답

따라서 **재부팅 이후에도 인증 → DHCP → routing → 인터넷 연결이 자동 복구됨**을 확인했다.

---

## 16. 임시 ICS 구성 제거

Wi-Fi persistence 검증 완료 후 Windows ICS는 더 이상 Host 인터넷 연결에 필요하지 않았다.

최종 정리 방향은 다음과 같다.

~~~text
Windows endpoint
   │
   │ Ethernet management LAN
   ▼
Proxmox Linux bridge
   │
   └─ management only

Proxmox USB Wi-Fi
   │
   └─ default Internet uplink
~~~

구축 과정에서 bridge에 추가했던 ICS subnet의 임시 주소와 fallback route를 제거하고, 유선 Ethernet은 기존 관리 subnet만 유지했다.

이렇게 하면 Host 인터넷 경로와 관리 경로가 분리된다.

---

## 17. 최종 검증 기준

동일 구조를 재구성할 때는 다음 기준을 모두 만족해야 한다.

| 계층 | 검증 항목 | 성공 기준 |
|---|---|---|
| Hardware | USB Wi-Fi device 인식 | USB device와 network interface 확인 |
| Driver | kernel driver | `ethtool -i`에서 driver 확인 |
| RF | radio block | soft/hard block 모두 해제 |
| 802.11 | AP scan | target SSID 탐지 |
| WPA | authentication | key negotiation 및 connected event |
| IPv4 | DHCP | Wi-Fi subnet 주소 획득 |
| Routing | default route | Wi-Fi gateway가 기본 경로 |
| DNS | name resolution | 외부 hostname 해석 성공 |
| L4/TLS | HTTPS | 외부 HTTPS 2xx 응답 |
| Persistence | reboot | 위 상태가 자동 복구 |

최종적으로 Wi-Fi uplink와 유선 관리망을 분리함으로써, 임시 중계 장치 없이 Proxmox Host가 독립적으로 인터넷에 접근하고 재부팅 후에도 동일 상태를 유지하는 구성이 완성되었다.
