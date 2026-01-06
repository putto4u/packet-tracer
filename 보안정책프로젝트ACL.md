# Chapter 5. Advanced Security Policy Implementation (Best Practice ACLs)

## 1. 개요 및 보안 강화 전략

본 섹션에서는 단순히 서버 접근을 제어하는 것을 넘어, 엔터프라이즈 네트워크에서 필수적으로 적용되는 **표준 보안 모델(Standard Security Models)**을 적용합니다. 이를 통해 포트폴리오의 수준을 높이고, 실무 지식을 어필할 수 있습니다.

### 1.1 적용할 주요 보안 정책 (Best Practices)

1. **관리자 접근 제어 (Management Access Control):** 오직 관리자 PC(또는 특정 관리 VLAN)에서만 네트워크 장비(Telnet/SSH)에 접속할 수 있도록 제한합니다.
2. **안티 스푸핑 (Anti-Spoofing):** 외부(인터넷)에서 사설 IP 대역을 달고 들어오는 위조된 패킷을 엣지(Edge) 단계에서 차단합니다.
3. **지사-본사 중요 부서 격리 (Branch-HQ Segmentation):** 물리적으로 떨어진 지사(Branch)에서 본사의 핵심 부서(전략기획실 등)로의 직접 접근을 차단합니다.
4. **ICMP 제어 (Stealth Mode):** 외부에서 내부망 구조를 스캐닝(Ping Sweep, Traceroute) 하지 못하도록 차단합니다.

---

## 2. 시나리오별 구현 (Scenario Implementation)

### 2.1 관리자 접근 제어 (Management Access ACL)

**목적:** 해커나 내부 사용자가 무단으로 라우터/스위치 콘솔에 접속하는 것을 방지합니다.
**가정:** `VLAN 50 (192.168.50.0/24)`을 관리자(Admin) 네트워크로 지정합니다.

**적용 대상:** **모든 장비 (R1~R4, MSW1~2, L3_Branch)**
*(대표적으로 R1과 MSW1에 적용하는 예시를 보여줍니다.)*

```bash
! -- R1 (Edge Router) 설정 --
R1(config)# ip access-list standard ADMIN_ONLY
R1(config-std-nacl)# permit 192.168.50.0 0.0.0.255
R1(config-std-nacl)# deny any
R1(config-std-nacl)# exit

! -- VTY 라인(원격 접속)에 적용 --
R1(config)# line vty 0 4
R1(config-line)# access-class ADMIN_ONLY in
R1(config-line)# transport input ssh
R1(config-line)# exit

! -- MSW1 (Core Switch) 설정 --
MSW1(config)# ip access-list standard ADMIN_ONLY
MSW1(config-std-nacl)# permit 192.168.50.0 0.0.0.255
MSW1(config-std-nacl)# deny any
MSW1(config-std-nacl)# exit

MSW1(config)# line vty 0 15
MSW1(config-line)# access-class ADMIN_ONLY in
MSW1(config-line)# transport input ssh
MSW1(config-line)# exit

```

### 2.2 엣지 보안: Anti-Spoofing (RFC 2827/BCP 38)

**목적:** 인터넷(ISP) 방향에서 들어오는 패킷의 출발지 IP가 '우리 내부 사설 IP'일 수 없습니다. 만약 그렇다면 이는 IP 스푸핑 공격이므로 엣지 라우터에서 즉시 드랍해야 합니다.

**적용 대상:** **HQ Edge Router (R1, R2)의 WAN 인터페이스 (Inbound)**

```bash
! -- R1 설정 --
R1(config)# ip access-list extended BLOCK_SPOOFING

! -- 1. 자신의 사설 IP 대역을 달고 들어오는 패킷 차단 --
R1(config-ext-nacl)# deny ip 192.168.0.0 0.0.255.255 any
R1(config-ext-nacl)# deny ip 172.16.0.0 0.0.15.255 any
R1(config-ext-nacl)# deny ip 10.0.0.0 0.255.255.255 any
R1(config-ext-nacl)# deny ip 127.0.0.0 0.255.255.255 any

! -- 2. 나머지 정상 트래픽 허용 --
R1(config-ext-nacl)# permit ip any any
R1(config-ext-nacl)# exit

! -- 3. WAN 인터페이스(Inbound) 적용 --
R1(config)# interface GigabitEthernet0/0
R1(config-if)# ip access-group BLOCK_SPOOFING in
R1(config-if)# exit

```

*(R2에도 동일하게 적용)*

### 2.3 지사-본사 중요 부서 격리 (Inter-Site Security)

**목적:** 지사(Branch)는 물리적 보안이 본사보다 취약할 수 있습니다. 지사 PC(`192.170.x.x`)가 해킹당하더라도 본사의 핵심인 '전략기획실(`VLAN 30`)'에는 접근하지 못하게 해야 합니다.

**적용 대상:** **HQ Core Switch (MSW1, MSW2)**
**적용 위치:** **VLAN 30 (전략기획실) 인터페이스 Outbound** (전략기획실로 들어가는 마지막 관문)

```bash
! -- MSW1 설정 --
MSW1(config)# ip access-list extended PROTECT_STRATEGY

! -- 1. 지사 대역(192.170.0.0/16 전체)에서의 접근 차단 --
MSW1(config-ext-nacl)# deny ip 192.170.0.0 0.0.255.255 192.168.30.0 0.0.0.255

! -- 2. 나머지 통신 허용 (인터넷, 타 부서 등) --
MSW1(config-ext-nacl)# permit ip any any
MSW1(config-ext-nacl)# exit

! -- 3. VLAN 30 인터페이스 적용 --
MSW1(config)# interface Vlan30
MSW1(config-if)# ip access-group PROTECT_STRATEGY out
MSW1(config-if)# exit

```

*(MSW2에도 동일하게 적용)*

### 2.4 ICMP 제어 (Reconnaissance Protection)

**목적:** 외부에서 우리 내부망으로 Ping(Echo Request)을 보내 네트워크 구조를 파악하거나, Smurf 공격을 하는 것을 방지합니다. 단, 내부에서 외부로 나가는 Ping의 응답(Echo Reply)은 허용해야 합니다.

**적용 대상:** **HQ Edge Router (R1, R2)의 WAN 인터페이스 (Inbound)**

```bash
! -- R1 설정 (기존 BLOCK_SPOOFING ACL 수정) --
R1(config)# ip access-list extended BLOCK_SPOOFING

! (앞서 설정한 스푸핑 차단 룰들 아래에 추가)
! -- 1. 외부에서 들어오는 Ping 요청(Echo) 차단 --
R1(config-ext-nacl)# deny icmp any any echo

! -- 2. 외부에서 들어오는 Traceroute 차단 (UDP 포트 범위) --
R1(config-ext-nacl)# deny udp any any range 33434 33523

! -- 3. 나머지(내부가 요청한 Ping의 응답 등) 허용 --
R1(config-ext-nacl)# permit ip any any

```

---

## 3. 포트폴리오/보고서용 요약 (Report Summary)

이 프로젝트에서는 단순한 네트워크 연결을 넘어, 실제 엔터프라이즈 환경에서 요구되는 **다계층 보안 아키텍처(Defense-in-Depth)**를 구현하였습니다.

1. **Management Plane Security:** `Standard ACL`을 `VTY Line`에 적용하여 승인된 관리자(VLAN 50) 외의 관리 접속 시도를 원천 차단함.
2. **Data Plane Security (Edge):** `Anti-Spoofing ACL`을 적용하여 외부 위조 패킷 유입을 차단하고 DDoS 공격 위험을 경감함.
3. **Data Plane Security (Internal):** `Extended ACL`을 통해 물리적으로 취약할 수 있는 지사(Branch) 네트워크가 본사의 핵심 자산(전략기획실)에 접근하는 것을 제어함.
4. **Reconnaissance Protection:** ICMP 및 Traceroute 패킷 제어를 통해 내부 네트워크 토폴로지 정보가 외부로 유출되는 것을 방지함.

이러한 정책들은 네트워크의 가용성(Availability)뿐만 아니라 기밀성(Confidentiality)과 무결성(Integrity)을 보장하는 핵심 요소입니다.

Next Level : AWS VPC 생성 및 Site-to-Site VPN 연결 설정
