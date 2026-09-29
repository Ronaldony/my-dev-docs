# Proxmox VM 인터넷 Egress 및 절전 독립성 구성

## 1. 목적과 전제

물리 Windows 시스템과 여러 Linux VM이 동일한 유선 LAN에 연결된 환경에서, VM의 인터넷 통신이 Windows 시스템의 전원 상태에 종속되는 문제가 확인되었다.

최종 구성의 목표는 다음과 같다.

- VM의 기본 게이트웨이를 항상 켜져 있는 Proxmox Host로 통일한다.
- Proxmox Host가 VM LAN의 IPv4 라우팅과 NAT를 담당한다.
- 인터넷 uplink는 Host의 별도 네트워크 인터페이스를 사용한다.
- Windows 시스템은 동일 LAN의 일반 endpoint 및 Wake-on-LAN 대상일 뿐, VM의 게이트웨이 역할을 하지 않는다.
- Windows가 S3 절전 상태에 들어가도 VM 인터넷 통신은 유지되어야 한다.
- Host와 Guest 재부팅 이후에도 설정이 자동 복구되어야 한다.
- VM LAN prefix를 실제 bridge LAN 범위와 일치하도록 통일한다.

구성에서 사용된 핵심 기술은 Proxmox VE, Linux bridge, ifupdown2, Linux IPv4 forwarding, iptables legacy, MASQUERADE, Netplan, QEMU Guest Agent, SSH, Wake-on-LAN이다.

---

## 2. 초기 네트워크 상태 확인

먼저 Host와 Guest를 변경하지 않은 상태에서 네트워크 경로를 확인했다.

Host에서는 다음 상태가 확인되었다.

- VM LAN은 Linux bridge에 연결되어 있었다.
- 물리 Ethernet 인터페이스는 bridge port로 사용되고 있었다.
- 인터넷 기본 경로는 별도의 Wi-Fi uplink를 사용하고 있었다.
- IPv4 forwarding은 비활성 상태였다.
- nftables ruleset은 비어 있었다.
- iptables는 legacy backend를 사용하고 있었다.
- FORWARD 기본 정책은 ACCEPT였다.
- NAT table에는 기존 POSTROUTING 규칙이 없었다.
- Proxmox firewall 서비스는 실행 중이었지만 정책 상태는 비활성 상태였다.

Guest에서는 다음 상태가 확인되었다.

- VM과 Host LAN bridge 사이의 통신은 정상적이었다.
- VM에서 Windows endpoint와 Proxmox Host 모두 도달 가능했다.
- 기본 게이트웨이는 Windows endpoint를 가리키고 있었다.
- 공인 IP, DNS, HTTPS 통신은 실패했다.
- 네트워크는 정적 Netplan 구성으로 관리되고 있었다.
- cloud-init은 비활성 marker에 의해 네트워크 구성을 다시 생성하지 않는 상태였다.

따라서 문제 범위는 VM의 L2 연결이나 bridge 자체가 아니라, **기본 게이트웨이 이후의 인터넷 egress 경로**로 좁혀졌다.

---

## 3. 일시적 Egress Pilot

영구 설정을 변경하기 전에 한 대의 VM을 대상으로 runtime-only pilot을 진행했다.

Host에서는 다음 두 가지 변경만 일시 적용했다.

1. IPv4 forwarding 활성화
2. VM LAN에서 인터넷 uplink로 나가는 트래픽에 MASQUERADE 적용

개념적인 NAT 규칙은 다음과 같다.

```bash
iptables -t nat -A POSTROUTING \
  -s <VM_LAN_CIDR> \
  -o <UPLINK_INTERFACE> \
  -j MASQUERADE
```

Guest에서는 기본 게이트웨이를 runtime에서만 Proxmox Host의 LAN 주소로 변경했다.

```text
기존 기본 게이트웨이
    ↓
Proxmox Host LAN 주소
```

pilot 검증은 다음 순서로 수행했다.

1. Proxmox Host LAN 도달 여부
2. 동일 LAN의 Windows endpoint 도달 여부
3. 공인 IP ICMP 통신
4. DNS 이름 해석
5. HTTPS 접속
6. Ubuntu APT repository 갱신

모든 항목이 성공했다.

이 결과로 다음 구조가 실제로 동작함을 확인했다.

```text
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
별도 Internet uplink
   │
   ▼
Internet
```

---

## 4. Host Egress 영구화

pilot 성공 후 Host 설정을 영구화했다.

### 4.1 IPv4 forwarding

sysctl 영구 설정을 사용하여 부팅 후에도 IPv4 forwarding이 활성화되도록 구성했다.

```text
net.ipv4.ip_forward = 1
```

### 4.2 NAT lifecycle

별도의 방화벽 프레임워크나 추가 persistent package를 도입하지 않고, 기존 ifupdown2 네트워크 lifecycle에 NAT 적용과 제거를 연결했다.

NAT helper는 다음 특성을 갖도록 구성했다.

- VM LAN CIDR과 인터넷 uplink를 명시적으로 제한
- POSTROUTING MASQUERADE 규칙만 관리
- 동일 규칙의 중복 추가 방지
- uplink 활성화 시 규칙 적용
- uplink 종료 시 규칙 제거
- 상태 확인 기능 제공

ifupdown2의 uplink interface lifecycle에서는 다음 구조를 사용한다.

```text
uplink up
  → NAT helper apply

uplink down
  → NAT helper remove
```

기존 Host firewall 체계를 대체하지 않고, VM egress에 필요한 최소 NAT 기능만 추가했다.

---

## 5. Guest 기본 게이트웨이 영구 전환

Host egress가 안정적으로 동작한 뒤 Guest 설정을 순차적으로 변경했다.

먼저 pilot VM의 Netplan 원본을 백업하고, 주소·DNS·기타 route는 유지한 채 **default route의 gateway만** Proxmox Host로 변경했다.

```yaml
routes:
  - to: default
    via: <PROXMOX_HOST_LAN_IP>
```

적용 순서는 다음과 같았다.

1. Netplan 후보 설정 생성
2. 변경 diff 확인
3. `netplan generate` 검증
4. `netplan apply`
5. SSH 연결 확인
6. QEMU Guest Agent 확인
7. 공인 IP, DNS, HTTPS 확인
8. Guest reboot
9. reboot 후 동일 항목 재검증

pilot VM의 reboot persistence가 확인된 뒤 나머지 VM에 동일 방식으로 순차 적용했다.

각 VM은 독립적으로 검증했으며, 실패 시 해당 VM만 원래 Netplan으로 되돌릴 수 있도록 rollback 기준을 유지했다.

---

## 6. Windows S3 독립성 검증

VM의 인터넷 경로가 Windows 시스템에서 완전히 분리되었는지 확인하기 위해 S3 절전 상태에서 acceptance test를 수행했다.

Windows가 S3 상태에 진입한 뒤 다음 상태가 확인되었다.

- RDP 포트는 닫힘
- Ethernet carrier는 유지
- Proxmox Host와 VM은 계속 동작
- 모든 running VM의 기본 게이트웨이는 Proxmox Host 유지
- 모든 running VM의 공인 IP 통신 성공
- DNS 이름 해석 성공
- HTTPS 통신 성공

즉 Windows 전원 상태와 VM 인터넷 egress 사이의 의존성이 제거되었다.

이후 기존 Wake-on-LAN 경로를 사용해 Magic Packet을 전송했고, Windows가 정상적으로 복귀한 뒤 RDP와 Remote Desktop 연결이 다시 가능함을 확인했다.

Wake-on-LAN 자체의 신규 구축이 아니라, **기존 WOL 경로가 변경된 VM egress 구조와 충돌하지 않는지에 대한 회귀 검증**으로 수행했다.

---

## 7. Host 재부팅 Persistence 검증

Host 재부팅 전 다음 상태를 기준선으로 기록했다.

- IPv4 forwarding 활성
- NAT 규칙 활성
- NAT lifecycle hook 존재
- 자동 시작 VM과 비자동 시작 VM의 전원 상태 구분
- Remote MCP와 VM lifecycle 서비스 활성

Host를 정상 재부팅한 뒤 다음 항목을 다시 확인했다.

- boot ID 변경
- IPv4 forwarding 자동 복구
- Remote MCP 자동 복구
- VM lifecycle 서비스 자동 복구
- 자동 시작 VM의 running 상태 복구
- 비자동 시작 VM의 stopped 상태 유지
- Guest 기본 게이트웨이 유지
- SSH 및 QEMU Guest Agent 정상
- 공인 IP, DNS, HTTPS 통신 정상

비특권 자동화 계정에서는 iptables 상태를 직접 조회할 권한이 없기 때문에 NAT helper의 상태 출력만으로는 판정하지 않았다.

대신 Guest에서 실제 인터넷 통신이 성공하는지 확인하여 NAT 복구 여부를 기능적으로 검증했고, root 권한으로 최종 NAT rule이 정확히 한 개 존재함을 별도로 확인했다.

---

## 8. VM LAN Prefix 정규화

egress 작업 이후 일부 VM이 실제 LAN보다 넓은 `/16` prefix를 사용하고 있음을 확인했다.

이 상태에서는 connected route가 실제 bridge LAN 범위보다 넓게 생성되었다.

```text
기존 일부 VM
192.168.x.x/16
→ connected route: 192.168.0.0/16

실제 VM LAN
→ /24
```

주소, 기본 게이트웨이, DNS는 그대로 유지하고 prefix만 `/24`로 정규화했다.

적용은 다음 순서로 진행했다.

1. 한 VM을 canary로 선정
2. Netplan 원본 백업
3. `/16 → /24` 한 항목만 변경
4. Netplan 생성 및 적용
5. Host LAN 통신 확인
6. Windows endpoint 통신 확인
7. peer VM 통신 확인
8. 인터넷, DNS, HTTPS 확인
9. QEMU Guest Agent 확인
10. 나머지 대상 VM에 동일 적용

최종적으로 모든 관리 대상 VM에서 connected route가 동일한 VM LAN `/24` 범위로 통일되었다.

---

## 9. 최종 동작 구조

최종 구조는 다음과 같다.

```text
                    Internet
                       ▲
                       │
               별도 Host uplink
                       ▲
                       │
             iptables MASQUERADE
                       ▲
                       │
             IPv4 forwarding
                       ▲
                       │
               Proxmox Host
              Linux bridge /24
                ▲            ▲
                │            │
        Linux VM group    Windows endpoint
        gateway = Host     WOL / RDP target
```

핵심 역할은 명확히 분리된다.

- **Proxmox Host**: VM 기본 게이트웨이, IPv4 forwarding, NAT
- **Linux bridge**: VM과 물리 LAN의 L2 연결
- **Internet uplink**: Host 외부 egress
- **Linux VM**: Host를 default gateway로 사용
- **Windows endpoint**: 같은 LAN의 일반 endpoint, S3 및 WOL 대상
- **QEMU Guest Agent / SSH**: Guest 상태 확인과 관리

Windows endpoint가 정지하거나 절전 상태에 들어가더라도 VM egress 경로에는 포함되지 않는다.

---

## 10. 주요 이슈와 해결 과정

### 이슈 1. LAN 통신은 되지만 VM 인터넷이 동작하지 않음

**이슈 내용과 발생 시점**

VM은 동일 LAN의 Host 및 Windows endpoint와 통신할 수 있었지만 공인 IP, DNS, HTTPS 접속은 실패했다.

**발생 원인 추적**

Guest 기본 게이트웨이가 Windows endpoint를 가리키고 있었고, 해당 endpoint는 VM의 지속적인 인터넷 gateway 역할을 수행하도록 구성된 장비가 아니었다.

동시에 Proxmox Host는 IPv4 forwarding이 비활성 상태였고 VM LAN에 대한 NAT도 존재하지 않았다.

**해결 과정**

- Proxmox Host를 VM default gateway로 사용
- IPv4 forwarding 활성화
- VM LAN에서 별도 인터넷 uplink 방향으로 MASQUERADE 적용
- Guest default route를 Host로 변경

**검증**

공인 IP, DNS, HTTPS, APT repository를 단계적으로 확인했다.

**결과**

VM 인터넷 egress가 정상화되었고 Windows의 전원 상태와 분리되었다.

### 이슈 2. 일부 VM의 subnet prefix 불일치

**이슈 내용과 발생 시점**

egress rollout 이후 일부 Guest에서 실제 bridge LAN보다 넓은 connected route가 확인되었다.

**발생 원인 추적**

해당 Guest의 정적 Netplan 주소가 `/16`으로 구성되어 있었다.

**해결 과정**

주소와 gateway는 유지하고 prefix만 실제 LAN 범위인 `/24`로 변경했다.

**검증**

Host, Windows endpoint, peer VM, 인터넷, DNS, HTTPS, QEMU Guest Agent를 확인했다.

**결과**

모든 관리 대상 VM의 LAN prefix와 connected route가 `/24`로 일관되게 정리되었다.

---

## 11. 검증 기준 요약

최종 완료 기준은 다음과 같다.

- Host reboot 후 IPv4 forwarding 유지
- Host reboot 후 NAT 자동 복구
- 모든 managed VM의 default gateway가 Host를 가리킴
- 모든 managed VM의 LAN prefix가 `/24`
- SSH 및 QEMU Guest Agent 정상
- 공인 IP 통신 정상
- DNS 정상
- HTTPS 정상
- APT repository 접근 정상
- Windows S3 상태에서도 VM 인터넷 정상
- Wake-on-LAN 이후 Windows 복귀 정상
- 비자동 시작 VM의 전원 상태가 Host reboot 후에도 유지

이 기준을 모두 통과하여 최종 구성을 확정했다.
