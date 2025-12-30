## 단일 L3 스위치 기반 VLAN 구성 및 스위치 간 연결

 Cisco 3560 L3 스위치를 Core로 두고, 하위에 2960 L2 스위치 2대를 연결하여 VLAN 10과 VLAN 20으로 네트워크를 격리하는 구성

---

### 1. 네트워크 구성 개요

* **Core Switch (L3):** Cisco 3560 (VLAN 간 라우팅 및 게이트웨이 역할)
* **Access Switch (L2):** Cisco 2960 (2대, 각 VLAN 전용 연결)
* **VLAN 정보:**
* **VLAN 10:** 192.168.10.0/24 (Gateway: 192.168.10.1)
* **VLAN 20:** 192.168.20.0/24 (Gateway: 192.168.20.1)


* **연결 포트:**
* 3560 (Fa0/1) <---> 2960-A (Fa0/1) [Trunk]
* 3560 (Fa0/2) <---> 2960-B (Fa0/1) [Trunk]
* 2960-A (Fa0/2) <---> PC 1 [Access VLAN 10]
* 2960-B (Fa0/2) <---> PC 2 [Access VLAN 20]



---

### 2. Core Switch (3560) 설정

L3 스위치에서는 VLAN 생성, 트렁크 포트 설정, 그리고 서로 다른 VLAN 간 통신을 위한 **IP Routing** 활성화가 핵심입니다.

```bash
Switch> enable
Switch# configure terminal
Switch(config)# hostname Core-3560

! 1. VLAN 생성
Core-3560(config)# vlan 10
Core-3560(config-vlan)# name Sales
Core-3560(config-vlan)# exit
Core-3560(config)# vlan 20
Core-3560(config-vlan)# name Engineering
Core-3560(config-vlan)# exit

! 2. L3 라우팅 활성화 (VLAN 간 통신 허용)
Core-3560(config)# ip routing

! 3. SVI(Switched Virtual Interface) 설정 - 각 VLAN의 게이트웨이
Core-3560(config)# interface vlan 10
Core-3560(config-if)# ip address 192.168.10.1 255.255.255.0
Core-3560(config-if)# no shutdown
Core-3560(config-if)# exit

Core-3560(config)# interface vlan 20
Core-3560(config-if)# ip address 192.168.20.1 255.255.255.0
Core-3560(config-if)# no shutdown
Core-3560(config-if)# exit

! 4. 트렁크 포트 설정 (2960 연결용)
! 3560은 encapsulation dot1q 명령어를 먼저 입력해야 트렁크 설정이 가능함
Core-3560(config)# interface range fastEthernet 0/1 - 2
Core-3560(config-if-range)# switchport trunk encapsulation dot1q
Core-3560(config-if-range)# switchport mode trunk
Core-3560(config-if-range)# exit

```

---

### 3. Access Switch A (2960-A) 설정

VLAN 10을 사용하는 첫 번째 L2 스위치 설정입니다.

```bash
Switch> enable
Switch# configure terminal
Switch(config)# hostname Access-2960A

! 1. VLAN 생성
Access-2960A(config)# vlan 10
Access-2960A(config-vlan)# exit

! 2. 업링크 포트 설정 (Core 스위치 연결)
Access-2960A(config)# interface fastEthernet 0/1
Access-2960A(config-if)# switchport mode trunk
Access-2960A(config-if)# exit

! 3. PC 연결 포트 설정 (VLAN 10 할당)
Access-2960A(config)# interface fastEthernet 0/2
Access-2960A(config-if)# switchport mode access
Access-2960A(config-if)# switchport access vlan 10
Access-2960A(config-if)# spanning-tree portfast
Access-2960A(config-if)# exit

```

---

### 4. Access Switch B (2960-B) 설정

VLAN 20을 사용하는 두 번째 L2 스위치 설정입니다.

```bash
Switch> enable
Switch# configure terminal
Switch(config)# hostname Access-2960B

! 1. VLAN 생성
Access-2960B(config)# vlan 20
Access-2960B(config-vlan)# exit

! 2. 업링크 포트 설정 (Core 스위치 연결)
Access-2960B(config)# interface fastEthernet 0/1
Access-2960B(config-if)# switchport mode trunk
Access-2960B(config-if)# exit

! 3. PC 연결 포트 설정 (VLAN 20 할당)
Access-2960B(config)# interface fastEthernet 0/2
Access-2960B(config-if)# switchport mode access
Access-2960B(config-if)# switchport access vlan 20
Access-2960B(config-if)# spanning-tree portfast
Access-2960B(config-if)# exit

```

---

### 5. PC 네트워크 설정 및 확인

| 장비 | IP 주소 | 서브넷 마스크 | 기본 게이트웨이 |
| --- | --- | --- | --- |
| **PC 1** | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| **PC 2** | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |

> **실전 팁:** > 3560 스위치에서 `ip routing` 명령어를 생략하면 VLAN 간의 통신(Inter-VLAN Routing)이 이루어지지 않습니다. 설정 후 반드시 `show ip route` 명령어로 라우팅 테이블이 생성되었는지 확인하십시오.

Next: VLAN 통신 문제 해결 및 트러블슈팅 조치 방법
