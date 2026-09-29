# Proxmox VM 인터넷 Egress 및 절전 독립성 구성

## 1. 목적

동일한 LAN에 Proxmox Host, Linux VM, Windows endpoint가 함께 연결된 환경에서 VM의 인터넷 경로가 Windows 시스템의 전원 상태에 종속되어 있었다.

최종 구성의 목표는 다음과 같다.

- Linux VM의 기본 게이트웨이를 항상 동작하는 Proxmox Host로 통일한다.
- Proxmox Host가 VM LAN의 IPv4 forwarding과 NAT를 담당한다.
- Host의 별도 uplink를 통해 인터넷으로 egress한다.
- Windows endpoint는 일반 LAN endpoint와 Wake-on-LAN 대상 역할만 수행한다.
- Windows가 S3 절전 상태여도 VM 인터넷 통신이 유지되어야 한다.
- Host와 Guest 재부팅 후에도 egress 구성이 자동 복구되어야 한다.
- VM LAN의 subnet prefix를 실제 bridge LAN 범위와 일치시킨다.

사용한 핵심 기술은 Proxmox VE, Linux bridge, ifupdown2, Linux IPv4 forwarding, iptables legacy, MASQUERADE, Netplan, QEMU Guest Agent, SSH, Wake-on-LAN이다.

---

## 2. 초기 네트워크 상태 확인

변경 전 Host와 Guest의 네트워크 경로를 먼저 확인했다.

### 2.1 Host 상태

확인된 상태는 다음과 같았다.

- VM LAN은 Linux bridge에 연결되어 있었다.
- 물리 Ethernet 인터페이스는 bridge port로 사용되고 있었다.
- 인터넷 기본 경로는 별도의 Wi-Fi uplink를 사용하고 있었다.
- IPv4 forwarding은 비활성 상태였다.
- nftables ruleset은 비어 있었다.
- iptables는 legacy backend를 사용하고 있었다.
- FORWARD 기본 정책은 ACCEPT였다.
- NAT table에는 기존 POSTROUTING 규칙이 없었다.
- Proxmox firewall 서비스는 실행 중이었지만 정책 자체는 비활성 상태였다.

### 2.2 Guest 상태

대표 VM에서 다음 상태가 확인되었다.

- VM과 Proxmox Host 사이의 LAN 통신은 정상적이었다.
- 동일 LAN의 Windows endpoint에도 도달할 수 있었다.
- 기본 게이트웨이는 Windows endpoint를 가리키고 있었다.
- 공인 IP 통신은 실패했다.
- DNS 이름 해석은 실패했다.
- HTTPS 접속도 실패했다.
- 네트워크는 정적 Netplan 구성으로 관리되고 있었다.
- cloud-init의 네트워크 재생성 기능은 비활성화되어 있었다.

따라서 문제는 Linux bridge나 L2 연결 자체가 아니라, **Guest의 기본 게이트웨이 이후 인터넷 egress 경로가 존재하지 않는 상태**로 좁혀졌다.

---

## 3. Firewall 및 NAT 구조 확인

영구 변경 전 Host의 forwarding 및 firewall 상태를 확인했다.

~~~text
IPv4 forwarding
    disabled

nftables
    no active ruleset

iptables
    legacy backend
    FORWARD policy = ACCEPT
    no existing NAT POSTROUTING rule
~~~

이 상태에서는 별도의 forwarding 차단 규칙이 존재하지 않았기 때문에, pilot에서 필요한 최소 변경은 다음 두 가지였다.

1. Host IPv4 forwarding 활성화
2. VM LAN에서 uplink 방향으로 나가는 트래픽에 MASQUERADE 적용

개념적인 NAT 규칙은 다음과 같다.

~~~bash
iptables -t nat -A POSTROUTING \
  -s <VM_LAN_CIDR> \
  -o <UPLINK_INTERFACE> \
  -j MASQUERADE
~~~

이 단계에서는 영구 설정을 변경하지 않았다.

---

## 4. Runtime-only Egress Pilot

영구 설정 전 한 대의 VM을 대상으로 runtime-only pilot을 수행했다.

### 4.1 Host runtime 변경

Host에서는 다음 두 설정만 일시 적용했다.

~~~text
IPv4 forwarding = enabled

POSTROUTING
  source: <VM_LAN_CIDR>
  output: <UPLINK_INTERFACE>
  action: MASQUERADE
~~~

변경 전 상태를 기록하고, NAT 규칙과 forwarding 값을 원래 상태로 되돌릴 수 있는 rollback 절차도 함께 준비했다.

### 4.2 Guest runtime gateway 변경

대표 VM의 기본 게이트웨이를 재부팅 없이 일시적으로 Proxmox Host의 LAN 주소로 변경했다.

~~~text
Before
Guest
  → Windows endpoint
  → Internet unavailable

Pilot
Guest
  → Proxmox Host
  → Host uplink
  → Internet
~~~

### 4.3 Pilot 검증

다음 순서로 확인했다.

1. Proxmox Host LAN 도달
2. Windows endpoint 도달
3. 공인 IP ICMP 통신
4. DNS 이름 해석
5. HTTPS 접속
6. Ubuntu APT repository 접근

모든 항목이 성공했다.

이 결과로 다음 구조가 실제로 동작함을 확인했다.

~~~text
Linux VM
   │
   │ default route
   ▼
Proxmox Host / Linux bridge
   │
   │ IPv4 forwarding
   ▼
iptables POSTROUTING MASQUERADE
   │
   ▼
Host Internet uplink
   │
   ▼
Internet
~~~

---

## 5. Host Egress 영구화

pilot 성공 후 Host 설정을 영구화했다.

### 5.1 IPv4 forwarding 영구화

sysctl drop-in을 사용하여 Host 부팅 후에도 IPv4 forwarding이 활성화되도록 구성했다.

~~~text
net.ipv4.ip_forward = 1
~~~

### 5.2 NAT helper

MASQUERADE 규칙은 root-owned helper로 관리하도록 구성했다.

helper의 역할은 다음과 같다.

- VM LAN CIDR과 uplink를 명시적으로 제한
- POSTROUTING MASQUERADE 규칙 적용
- 동일 규칙 중복 추가 방지
- 규칙 제거
- root 권한에서 상태 확인

개념적인 동작은 다음과 같다.

~~~text
apply
  └─ NAT rule이 없으면 1회 추가

remove
  └─ 해당 NAT rule 제거

status
  └─ 현재 rule 존재 여부 확인
~~~

### 5.3 ifupdown2 lifecycle 연결

별도의 persistent firewall 패키지를 추가하지 않고 기존 ifupdown2 interface lifecycle에 NAT helper를 연결했다.

~~~text
uplink up
  → NAT helper apply

uplink down
  → NAT helper remove
~~~

따라서 uplink가 재구성되거나 Host가 재부팅되어도 NAT 규칙이 network lifecycle에 맞춰 다시 적용된다.

---

## 6. Guest 기본 게이트웨이 영구 전환

Host egress가 정상 동작한 뒤 Guest 설정을 순차적으로 변경했다.

### 6.1 Pilot VM 영구 전환

먼저 대표 VM의 Netplan 원본을 백업했다.

주소, DNS, 기타 route는 유지하고 default route의 gateway만 Proxmox Host로 변경했다.

~~~yaml
routes:
  - to: default
    via: <PROXMOX_HOST_LAN_IP>
~~~

적용 순서는 다음과 같았다.

1. 원본 Netplan 백업
2. 변경 후보 생성
3. diff로 gateway 외 변경 여부 확인
4. Netplan 구문 생성 검증
5. 설정 적용
6. SSH 재접속 확인
7. QEMU Guest Agent 확인
8. 공인 IP, DNS, HTTPS 확인
9. Guest reboot
10. reboot 후 동일 항목 재검증

pilot VM에서 reboot persistence까지 확인한 뒤 원래 전원 상태로 복원했다.

### 6.2 나머지 VM 순차 적용

나머지 running VM은 한 대씩 순차적으로 변경했다.

각 VM마다 다음 절차를 반복했다.

~~~text
backup
  ↓
gateway 변경
  ↓
netplan generate
  ↓
netplan apply
  ↓
SSH / QGA 확인
  ↓
Public IP / DNS / HTTPS 확인
~~~

각 VM은 독립적으로 rollback할 수 있도록 원본 Netplan을 유지한 상태에서 진행했다.

모든 대상 VM의 default gateway가 Proxmox Host로 통일되었다.

---

## 7. Windows S3 독립성 검증

VM의 인터넷 경로가 Windows 시스템과 완전히 분리되었는지 확인하기 위해 Windows를 실제 S3 상태로 전환했다.

### 7.1 S3 진입 확인

Windows가 절전 상태에 진입한 뒤 다음 상태가 확인되었다.

- RDP 포트는 닫혔다.
- Ethernet carrier는 유지되었다.
- Proxmox Host는 계속 동작했다.
- VM도 계속 동작했다.

### 7.2 S3 상태에서 VM egress 확인

Windows가 계속 S3 상태인 동안 모든 running VM에서 다음을 확인했다.

- default gateway가 Proxmox Host를 유지
- 공인 IP 통신 성공
- DNS 이름 해석 성공
- HTTPS 통신 성공

즉 VM 인터넷 egress는 Windows의 전원 상태와 완전히 분리되었다.

### 7.3 Wake-on-LAN 회귀 검증

기존 Wake-on-LAN 경로를 사용해 Windows를 다시 깨웠다.

확인 결과:

- Magic Packet 전송 성공
- Ethernet carrier 유지
- RDP 서비스 복귀
- Remote Desktop 연결 복귀
- VM 인터넷 egress 계속 정상

Wake-on-LAN 기능을 새로 구축한 것이 아니라, **기존 WOL 구조가 변경된 egress 설계와 충돌하지 않는지 확인하는 regression test**로 수행했다.

---

## 8. Host Reboot Persistence 검증

Host 재부팅 후에도 egress 구조가 자동 복구되는지 검증했다.

### 8.1 재부팅 전 기준선

다음 상태를 기록했다.

- IPv4 forwarding 활성
- NAT rule 활성
- NAT lifecycle hook 존재
- 자동 시작 VM과 비자동 시작 VM의 전원 상태
- Remote MCP 서비스 활성
- VM lifecycle 서비스 활성

### 8.2 재부팅 후 확인

Host를 정상 재부팅한 뒤 다음을 확인했다.

- boot ID 변경
- IPv4 forwarding 자동 복구
- Remote MCP 자동 복구
- VM lifecycle 서비스 자동 복구
- 자동 시작 VM의 running 상태 복구
- 비자동 시작 VM의 stopped 상태 유지
- Guest default gateway 유지
- SSH 정상
- QEMU Guest Agent 정상
- 공인 IP 통신 정상
- DNS 이름 해석 정상
- HTTPS 접속 정상

### 8.3 NAT 상태 검증 방식

비특권 자동화 계정에서는 iptables 조회 권한이 없기 때문에 NAT helper의 상태 출력만으로 복구 여부를 판정하지 않았다.

대신 두 단계로 검증했다.

~~~text
Guest 기능 검증
  → 실제 Internet / DNS / HTTPS 성공

root 검증
  → 동일 NAT rule이 정확히 1개 존재하는지 확인
~~~

이를 통해 Host reboot 이후에도 NAT가 정상적으로 자동 복구됨을 확인했다.

---

## 9. VM LAN Prefix 정규화

egress 전환 완료 후 일부 VM에서 실제 LAN보다 넓은 subnet prefix가 사용되고 있음을 확인했다.

### 9.1 확인된 상태

일부 VM에서는 다음과 같은 connected route가 생성되어 있었다.

~~~text
Guest address
  <VM_IP>/16

Connected route
  192.168.0.0/16
~~~

실제 VM LAN은 /24 범위였기 때문에, Guest가 필요 이상으로 넓은 네트워크를 직접 연결된 LAN으로 인식하고 있었다.

### 9.2 Canary 적용

먼저 한 VM을 canary로 선정했다.

변경 범위는 subnet prefix 하나로 제한했다.

~~~text
/16
 ↓
/24
~~~

주소, default gateway, DNS, 기타 route는 변경하지 않았다.

적용 후 다음을 확인했다.

- Guest 주소가 /24로 표시
- connected route가 VM LAN /24로 변경
- Proxmox Host 도달
- Windows endpoint 도달
- peer VM 도달
- 공인 IP 통신
- DNS
- HTTPS
- QEMU Guest Agent

모든 항목이 성공했다.

### 9.3 나머지 VM 적용

동일한 절차를 나머지 /16 VM에 순차 적용했다.

각 VM마다 원본 Netplan을 백업하고, prefix 변경 외 다른 diff가 없는지 확인한 뒤 적용했다.

최종적으로 모든 관리 대상 VM에서 다음 상태가 일관되게 확인되었다.

~~~text
Guest LAN prefix
  /24

Connected route
  <VM_LAN_CIDR>

Default gateway
  <PROXMOX_HOST_LAN_IP>
~~~

---

## 10. 최종 구조

최종 네트워크 구조는 다음과 같다.

~~~text
                           Internet
                              ▲
                              │
                       Host uplink
                              ▲
                              │
                  iptables MASQUERADE
                              ▲
                              │
                    IPv4 forwarding
                              ▲
                              │
                    Proxmox Host
                 Linux bridge /24 LAN
                    ▲               ▲
                    │               │
              Linux VM group   Windows endpoint
              gateway = Host    WOL / RDP target
~~~

구성요소의 역할은 다음과 같이 분리된다.

| 구성요소 | 역할 |
|---|---|
| Proxmox Host | VM default gateway, IPv4 forwarding, NAT |
| Linux bridge | VM과 물리 LAN의 L2 연결 |
| Host uplink | 인터넷 egress |
| Linux VM | Proxmox Host를 default gateway로 사용 |
| Windows endpoint | 일반 LAN endpoint, S3 및 WOL 대상 |
| Netplan | Guest 정적 네트워크 구성 |
| ifupdown2 | Host interface lifecycle과 NAT helper 연결 |
| iptables | VM LAN source NAT |
| QEMU Guest Agent | Guest 상태 및 reboot 검증 |
| SSH | Guest 네트워크 설정 및 동작 검증 |

Windows endpoint는 VM 인터넷 경로에 포함되지 않는다.

---

## 11. 최종 데이터 흐름

### VM 인터넷 통신

~~~text
Linux VM
  │
  │ default route
  ▼
Proxmox Host bridge address
  │
  │ IPv4 forwarding
  ▼
POSTROUTING MASQUERADE
  │
  ▼
Host uplink
  │
  ▼
Internet
~~~

### Windows 절전 중

~~~text
Windows endpoint
  └─ S3

Linux VM
  │
  ▼
Proxmox Host
  │
  ▼
Internet

Windows 상태와 무관하게 유지
~~~

### Host 재부팅 후

~~~text
Host boot
  │
  ├─ sysctl → IPv4 forwarding 활성화
  │
  ├─ ifupdown2 → uplink 구성
  │                └─ NAT helper apply
  │
  └─ VM lifecycle
       ├─ onboot VM → running
       └─ non-onboot VM → stopped 유지
~~~

---

## 12. 주요 이슈와 해결 과정

### 이슈 1. LAN 통신은 되지만 VM 인터넷이 동작하지 않음

**이슈 내용과 발생 시점**

VM은 동일 LAN의 Proxmox Host와 Windows endpoint에는 도달할 수 있었지만 공인 IP, DNS, HTTPS 통신은 실패했다.

**발생 원인 추적**

Guest default gateway가 Windows endpoint를 가리키고 있었다.

동시에 Proxmox Host에서는 IPv4 forwarding이 비활성 상태였고, VM LAN을 uplink로 변환하는 NAT 규칙도 존재하지 않았다.

**해결 과정**

- Proxmox Host를 VM default gateway로 전환
- Host IPv4 forwarding 활성화
- VM LAN에서 uplink 방향으로 MASQUERADE 적용
- runtime pilot 성공 후 Host와 Guest 설정 영구화

**검증**

공인 IP, DNS, HTTPS, Ubuntu APT repository까지 단계적으로 확인했다.

**결과**

VM 인터넷 egress가 정상화되었고 Windows의 전원 상태와 분리되었다.

---

### 이슈 2. 비특권 계정의 NAT 상태 확인 결과가 실제 상태와 다름

**이슈 내용과 발생 시점**

Host reboot 후 비특권 자동화 계정에서 NAT helper의 상태를 조회했을 때 inactive처럼 보이는 결과가 확인되었다.

**발생 원인 추적**

비특권 계정은 Host iptables 상태를 직접 조회할 권한이 없었다.

따라서 helper status 결과만으로 실제 NAT rule 존재 여부를 판단할 수 없었다.

**해결 과정**

- Guest에서 실제 인터넷 egress를 기능적으로 검증
- root 권한으로 NAT rule을 직접 확인

**검증**

Guest의 공인 IP, DNS, HTTPS 통신이 모두 성공했고, root 조회에서는 동일한 MASQUERADE rule이 정확히 한 개 존재했다.

**결과**

NAT persistence는 정상으로 확인되었으며, Host firewall 상태 검증은 root 권한과 Guest 기능 검증을 함께 사용하는 방식으로 확정했다.

---

### 이슈 3. 일부 VM의 subnet prefix 불일치

**이슈 내용과 발생 시점**

egress rollout 이후 일부 VM에서 실제 bridge LAN보다 넓은 connected route가 확인되었다.

**발생 원인 추적**

해당 VM의 정적 Netplan 주소가 /16 prefix로 설정되어 있었다.

**해결 과정**

- 한 VM에서 /16 → /24 canary 적용
- 주소, gateway, DNS는 유지
- LAN 및 인터넷 검증 후 나머지 대상 VM에 순차 적용

**검증**

Host, Windows endpoint, peer VM, 공인 IP, DNS, HTTPS, QEMU Guest Agent를 확인했다.

**결과**

모든 관리 대상 VM의 LAN prefix와 connected route가 실제 /24 LAN 범위로 통일되었다.

---

## 13. 최종 검증 기준

최종 구성은 다음 항목을 모두 통과했다.

| 검증 항목 | 결과 |
|---|---|
| Host IPv4 forwarding | 정상 |
| Host NAT | 정상 |
| Host reboot 후 NAT 자동 복구 | 정상 |
| VM default gateway | Host로 통일 |
| VM LAN prefix | /24로 통일 |
| Guest SSH | 정상 |
| QEMU Guest Agent | 정상 |
| 공인 IP 통신 | 정상 |
| DNS | 정상 |
| HTTPS | 정상 |
| APT repository 접근 | 정상 |
| Guest reboot persistence | 정상 |
| Host reboot persistence | 정상 |
| Windows S3 중 VM 인터넷 | 정상 |
| Wake-on-LAN 회귀 검증 | 정상 |
| 비자동 시작 VM 전원 상태 유지 | 정상 |

이 검증 결과를 기준으로 최종 구성을 확정했다.
