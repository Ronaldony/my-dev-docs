# 인프라 셋업 작업 로그 저장소

이 저장소는 특정 프로젝트의 작업 로그를 보관하는 공간으로서, 실제 구축·설정·연결·검증 과정에서 확인된 기술 구조와 원리를 문서로 정리하기 위한 저장소입니다.

## 문서화 원칙

- 특정 저장소명, 프로젝트명, 사용자명, 절대 경로, 로컬 폴더 구조 등 특정 환경에 종속된 표현은 배제합니다.
- 문서를 단순 텍스트 설명만으로 구성하지 않고, 구조도·흐름도·관계도·인포그래픽 등 이해를 돕는 시각적 표현을 적극적으로 포함합니다.
- 툴, 프로그램, 라이브러리, 프로토콜, 기술 표준 등 재현에 필요한 기술 명칭은 정확하게 기록합니다.
- 개별 파일 수정 내역보다 구성요소의 역할, 연결 관계, 동작 방식, 설정 목적을 중심으로 설명합니다.
- 문서는 실제로 수행되고 최종 반영된 작업만을 대상으로 합니다.
- 폐기된 방안, 미반영 계획, 단순 아이디어, 향후 개선 제안은 최종 구성과 직접 관련되지 않는 한 포함하지 않습니다.
- 확인된 사실과 작성자의 해석을 구분합니다.
- 다른 환경에서도 동일한 구조를 이해하거나 재구성할 수 있도록 필요한 기술 정보를 충분히 남깁니다.

## 문서 구성

문서는 실제 작업 순서에 따라 시간 흐름이 자연스럽게 이어지도록 구성합니다.

일반적인 흐름은 다음과 같습니다.

1. 구성 목적과 전제
2. 핵심 구성요소와 역할
3. 설정 및 연결 방식
4. 동작 구조
5. 검증 방법과 결과
6. 주요 이슈와 해결 과정

복잡한 구성이나 동작 흐름은 텍스트만으로 설명하기보다 구조도·흐름도·관계도·인포그래픽 등 적절한 시각적 표현을 함께 사용하여 이해하기 쉽게 정리합니다.

예상치 못한 문제를 기록할 때는 다음 순서를 사용합니다.

이슈 내용과 발생 시점 → 발생 원인 추적 → 해결 과정 → 검증 → 결과

## 작성 기준

문서의 목표는 특정 환경을 복제하는 것이 아니라, 해당 작업에서 확인된 기술적 구조와 적용 원리를 보존하는 것입니다.

따라서 다음과 같은 정보는 가능한 한 일반화합니다.

- 로컬 경로와 폴더명
- 사용자 계정명
- 저장소 및 프로젝트 고유 이름
- 환경에만 의미가 있는 파일 배치

반대로 다음 정보는 재현성을 위해 구체적으로 기록합니다.

- 사용 기술과 표준
- 구성요소 간 의존 관계
- 설정 목적과 주요 동작 조건
- 통신 및 연결 방식
- 검증 절차와 확인 결과
- 실제 문제의 원인과 해결 방법

## 문서 목록

- [Proxmox VM 인터넷 Egress 및 절전 독립성 구성](docs/proxmox-vm-internet-egress.md) — Proxmox Host 기반 NAT, Guest gateway 전환, Windows S3 독립성, 재부팅 persistence, VM LAN prefix 정규화 과정을 정리합니다.
- [FastMCP 기반 다중 MCP Gateway 및 Headless 도구 연결 구성](docs/fastmcp-multi-upstream-headless-gateway.md) — ProxyProvider 기반 다중 upstream 통합, namespace·Tool Search, headless 응용프로그램 bridge, Secure MCP Tunnel, upstream RPC 로깅과 실제 E2E 검증 과정을 정리합니다.
- [Proxmox Host 관리 접근 복구 및 USB Wi-Fi Uplink 영구화](docs/proxmox-host-usb-wifi-uplink.md) — GRUB 기반 root 접근 복구, Windows ICS 임시 egress, USB Wi-Fi 인증·DHCP·routing 검증, systemd와 ifupdown2 역할 분리, 재부팅 persistence 검증 과정을 정리합니다.
- [Windows 사용자 영역 파일 손실 진단 및 Reset 기반 복구](docs/windows-user-profile-loss-reset-recovery.md) — PowerShell/CMD 삭제 명령 인용 오류 분석, Event Log 기반 원인 분리, rollback source 평가, 물리 디스크 분리 백업, Windows Cloud Reset 및 재검증 과정을 정리합니다.

## 기여

문서를 추가하거나 수정할 때는 [CONTRIBUTING.md](CONTRIBUTING.md)의 기준을 따릅니다.
