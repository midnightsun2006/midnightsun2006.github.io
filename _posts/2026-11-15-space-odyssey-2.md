---
title: "[Paper Review] Space Odyssey: An Experimental Software Security Analysis of Satellites - week 2"
date: 2026-11-15 23:43:21 +0900
categories: [Paper Review, Satellite Security]
tags: [satellite, space-security, firmware, moonlight]
media_subpath: /assets/img/posts/space-odyssey-2

---

## 이번 주차 - week 2
2장 Satellite Context와 3장 Satellite Firmware Threats를 읽고 이해하였다.

## 2. Satellite Context

위성은 지구 같은 천체 주위를 돌도록 우주에 배치한 물체다. 하루 열 번 넘게 약 200도의 온도 변화를 겪고, 진공에 가까우며, 우주 방사선에 노출되는 환경에서 동작해야 한다. 그래도 통신, 지구 관측, 연구처럼 우주에서만 제공할 수 있는 서비스가 있어 위성을 쏘아 올린다.

대부분의 위성은 LEO(Low Earth Orbit, 250-2000km)에 있고, 목적에 따라 GEO(Geostationary Orbit, 35,786km) 같은 다른 궤도를 쓰기도 한다.

### 2.1 Satellite Operations

위성 운용은 세 segment로 나뉜다.

| Segment | 구성 | 역할 |
|---|---|---|
| Ground Segment | 운영자, GS | 위성 수명 내내 운용을 담당한다. 위성에 명령을 보내고 상태를 받는다. |
| Space Segment | 위성 한 기 또는 constellation 전체 | 서비스를 제공한다. 위성끼리 ISL(Inter-Satellite Link)로 통신하기도 한다. |
| User Segment | VSAT, GPS 수신기 같은 단말 | Space Segment가 제공하는 서비스를 받는다. |

#### Ground Segment

운영자는 Ground Station(GS)으로 위성과 통신한다. 주고받는 데이터는 방향에 따라 이름이 다르다.

| 이름 | 방향 | 내용 |
|---|---|---|
| Telecommand (TC) | GS -> 위성 (uplink) | 위성에 내리는 명령 |
| Telemetry (TM) | 위성 -> GS (downlink) | 위성의 상태, 오류, 각종 지표 |

TC는 space protocol을 쓰며, 논문은 TC와 TM을 합쳐 TC/TM traffic이라고 부른다.

#### Space Segment

위성은 로켓으로 궤도에 올라간 뒤 orbital deployment phase를 거쳐 지상국과 통신을 시작한다. 이후 정상 서비스를 수행하는 단계를 nominal operations phase라고 한다.

#### User Segment

지구 관측 위성처럼 지상국하고만 통신하는 위성은 User Segment가 없다.

### 2.2 Satellite Architecture

<figure>
  <img src="1.png" alt="">
  <figcaption style="text-align:center; font-size:0.8em; color:gray;">
    Space Odyssey (IEEE S&amp;P 2023), Figure 2
  </figcaption>
</figure>

위성의 구성 요소는 payload와 bus로 나뉜다.

#### payload와 bus

보안 분야에서 payload는 exploit payload, bus는 I2C, CAN, SPI 같은 data bus를 뜻하지만, 위성 공학에서는 뜻이 다르다. 이 논문만의 용어가 아니라 NASA, ESA 문서에서도 같은 뜻으로 쓴다.

| 용어 | 보안 분야 | 위성 공학 |
|---|---|---|
| Payload | exploit payload, 패킷의 데이터 부분 | 위성이 임무를 수행하는 장비. 예: 지구 관측용 고해상도 카메라, 통신용 무선 장비 |
| Bus | I2C, CAN, SPI 같은 data bus | 위성을 유지하고 운용하는 구성 요소 전체. COM + CDHS + ADCS + EPS + ... |

bus는 승객을 태우는 차량, payload는 승객이다. 승객이 관광객이면 관광버스, 군인이면 군용버스가 되듯이 payload가 카메라면 관측위성, 통신 장비면 통신위성이 된다. 실제로 제조사는 bus platform을 먼저 개발하고 payload만 바꿔 가며 위성을 만들기도 한다. 유럽 문헌에서는 bus를 platform이라고 부르기도 한다.

#### bus-payload separation

bus는 payload 없이도 동작하도록 설계되지만, payload는 bus에 의존한다. 논문은 이를 bus-payload separation이라고 부른다. 제어도 bus -> payload 방향으로만 이뤄진다.

논문은 bus를 중심으로 분석한다. payload와 달리 bus를 장악하면 위성 전체를 제어할 수 있기 때문이다.

#### 2.2.1 Command and Data Handling System (CDHS)

bus의 중심으로, 위성을 관리하고 모든 기능을 제어한다.

- OBC(On-Board Controller): MCU와 메모리로 이루어진 computing platform. 지상의 임베디드 기기와 비슷하다.
- OBSW(On-Board Software): OBC에서 실행되는 소프트웨어. 이 논문의 주된 분석 대상이다. 보통 RTOS 기반의 remote-control server로 구현된다.

OBSW가 하는 일은 다음과 같다.

- TC/TM traffic 처리
- data storage
- command scheduling
- autonomous action
- program code update

OBSW도 소프트웨어이므로 일반적인 소프트웨어 결함이 있을 수 있고, 공격자는 이를 이용해 CDHS의 제어권을 탈취할 수 있다.

#### 2.2.2 Communications Module (COM)

GS와의 통신을 담당한다. 안테나와 radio로 이루어지고, decoding, protocol implementation, access protection을 처리하는 computing setup이 붙기도 한다.

COM은 보통 TC/TM traffic만 처리한다. 촬영 영상 같은 대용량 payload data의 downlink는 별도의 더 강력한 radio가 맡는 경우가 많다.

| | COM | payload downlink radio |
|---|---|---|
| 전송 대상 | TC, TM | payload data (영상, 관측 데이터 등) |
| 데이터 양 | 적음 | 많음 |
| 방향 | uplink, downlink | 주로 downlink |

COM은 TC/TM 처리를 위해 CDHS와 바로 연결되어 있어 공격자의 주된 진입점이 된다. COM의 protocol implementation은 first line of defense가 될 수 있어 보안상 중요하다.

#### 2.2.3 Attitude Determination and Control System (ADCS)

위성의 attitude, 즉 위성이 어느 방향을 향하는지를 측정하고 조정한다. 안테나는 지구를, 태양전지판은 태양을 향하게 맞춘다.

- detumbling: 발사체에서 분리된 직후 위성은 회전하는데, 이 회전을 자동으로 멈추는 과정이다. 회전하는 동안에는 안테나 방향이 계속 바뀌므로 첫 통신 링크를 맺으려면 detumbling이 필요하다.
- AOCS(Attitude and Orbit Control System): ADCS에 thruster를 더해 궤도도 조금 바꿀 수 있는 시스템이다.

| | ADCS | AOCS |
|---|---|---|
| 제어 대상 | attitude | attitude + 궤도 |
| 장치 예시 | reaction wheel, magnetorquer | ADCS + thruster |

AOCS가 있는 위성은 다른 물체와 충돌할 수 있어 cyber-physical system이 된다. 충돌은 orbital chain reaction으로 이어질 수 있어 보안상 중요하다.

#### 2.2.4 Power Supply (EPS)

EPS(Electrical Power System)는 위성의 전원 장치다. 태양전지판으로 발전하고, 지구 그림자에 들어갈 때를 대비해 배터리에 저장한다.

배터리가 완전히 방전되면 위성을 다시 가동할 수 없다. 그래서 배터리를 완전히 방전시키는 deep-draining은 위성을 영구적으로 무력화하려는 공격자가 노릴 수 있다.

#### 2.2.5 Payload

payload에는 임무 장비와 함께 CDHS와 비슷한 장치가 탑재되는 경우가 많다.

- PDHS(Payload Data Handling System): payload의 데이터 처리 시스템. PLCOM에서 제어 트래픽을 받거나 임무 장비의 데이터를 처리한다.
- PLCOM(Payload Communication Module): payload에 달린 수신기. PL은 PayLoad를 뜻한다.

| bus | payload | 역할 |
|---|---|---|
| CDHS | PDHS | 명령 처리, 데이터 처리 |
| COM | PLCOM | 위성 밖과 통신 |

PLCOM도 통신 장비인데 payload에 속하는 이유는 bus와 payload를 통신 여부가 아니라 존재 목적으로 나누기 때문이다. COM이 없으면 위성을 운용할 수 없지만, PLCOM은 임무 수행용이라 없어도 COM으로 위성을 운용할 수 있다. 통신위성처럼 사용자 신호를 받아 중계하는 경우에는 수신기가 곧 임무 장비다.

payload는 위성마다 맞춤 제작되기 때문에 PDHS와 PLCOM의 명칭은 문헌마다 다를 수 있다. PDHS가 없는 위성도 있는데, 소형 큐브샛에서는 CDHS가 payload 장비를 바로 제어하기도 한다.

#### 2.2 정리

내가 파악한 위성 구조와 TC 흐름은 다음과 같다.

<figure>
  <img src="2.png" alt="">
</figure>

- CDHS는 ADCS와 EPS에도 명령을 내린다. ADCS와 EPS는 자체 마이크로컨트롤러로 detumbling, 배터리 보호 같은 기본 동작을 처리하지만, 목표 attitude 지정이나 장비 전원 제어 같은 명령은 CDHS에서 받는다. 따라서 CDHS를 장악하면 ADCS와 EPS도 조종할 수 있다.
- CDHS와 PDHS는 서로 다른 컴퓨터이고 Bus-Payload Link로 연결된다.
- GS에서 payload로 명령을 보내는 경로는 두 가지다.
  - GS -> COM -> CDHS -> PDHS: CDHS가 payload용 TC를 PDHS로 넘긴다.
  - GS -> PLCOM -> PDHS: bus를 거치지 않는다.

### 2.3 Satellite Communication Protocols

TC/TM traffic은 space protocol로 전달된다. 이런 프로토콜을 발표하는 주요 기관은 여러 우주 기관이 모여 표준을 정하는 CCSDS(Consultative Committee for Space Data Systems)다.

CCSDS는 기관 이름이지만, 논문은 CCSDS가 발표한 프로토콜 모음을 줄여서 CCSDS protocol이라고 부른다. 인터넷 프로토콜 모음을 TCP/IP라고 부르는 것과 비슷하다.

CCSDS 표준은 OSI 모델의 모든 계층을 다루고, 계층마다 선택지가 여러 개 있다. 논문에서 언급하는 프로토콜은 두 가지다.

- SDLS(Space Data Link Security): data link 계층. 인증과 암호화를 위한 security extension을 구현한다.
- SPP(Space Packet Protocol): network 계층

TC/TM을 주고받는 데 프로토콜 하나면 될 것 같지만, 웹 통신이 HTTP, TCP, IP, Ethernet을 쌓아 동작하듯 위성 통신도 계층마다 맡는 일이 다르다.

| 계층 | CCSDS 프로토콜 예시 | 역할 |
|---|---|---|
| physical | RF and Modulation 표준 | 주파수, 변조 방식 |
| data link (하위) | TC Synchronization and Channel Coding | 오류 정정, 동기화. 우주 채널은 잡음이 많아 필요하다 |
| data link | TC Space Data Link Protocol, SDLS | 프레임 구성, 재전송, 인증과 암호화 |
| network | SPP | APID로 위성 안의 목적지 애플리케이션 지정 |
| 상위 | CFDP 등 | 파일 전송 등 |

계층마다 선택지가 여러 개인 이유는 저궤도 큐브샛과 심우주 탐사선처럼 임무 환경이 다르기 때문이다. uplink(TC)와 downlink(TM)도 요구 조건이 달라 서로 다른 data link 프로토콜을 쓴다.

## 3. Satellite Firmware Threats

3장에서는 위성 펌웨어에 대한 위협을 3계층 threat taxonomy로 정리한다.

### Falco & Boschetti의 선행 연구와의 관계

Falco와 Boschetti의 선행 연구(*A Security Risk Taxonomy for Commercial Space Missions*, ASCEND 2021)는 우주 임무 중 생길 수 있는 위험을 분류했다. 논문은 그중 environmental, physical, digital-cyber-technical 위험을 예로 든다.

| 위험 | 예시 |
|---|---|
| Environmental | 우주 방사선, 태양 폭풍, 우주 파편 |
| Physical | 위성요격 미사일, 발사 전 하드웨어 조작 |
| Digital-cyber-technical | 해킹, 악성 TC, 데이터 탈취 |

이 논문은 digital-cyber-technical 위험을 high-level attacker goal로 삼고, 그 아래에 펌웨어 수준의 component와 interface를 붙였다. 선행 연구가 위험 전반을 넓게 분류했다면, 이 논문은 그중 digital-cyber-technical 위험을 펌웨어 수준까지 세분화했다.

<figure>
  <img src="3.png" alt="">
</figure>

### 3.1 Threat Taxonomy

<figure>
  <img src="4.png" alt="">
  <figcaption style="text-align:center; font-size:0.8em; color:gray;">
    Space Odyssey (IEEE S&amp;P 2023), Figure 3
  </figcaption>
</figure>

taxonomy는 세 layer로 이루어지고, 논문은 위에서 아래로 설명한다.

| Layer | 다루는 내용 | 예시 |
|---|---|---|
| Control layer | 공격자의 최종 목표 | Seizure of Control, Denial of Service |
| Components layer | 목표를 이루려고 노려야 하는 component | CDHS, PDHS |
| Interface layer | component에 닿기 위해 거쳐야 하는 입구 | COM Rx, TC Fetcher |

Figure 3에서 실선은 계층 관계(bus는 위성의 일부, CDHS는 bus의 일부)를, 점선 화살표는 공격 경로(CDHS에 TC를 보내려면 먼저 COM에 접근해야 함)를 나타낸다.

배터리를 완전히 방전시키는 공격을 예로 들면 다음과 같다.

| Layer | 예시 |
|---|---|
| Control layer | bus의 Denial of Service (배터리 완전 방전) |
| Components layer | CDHS를 장악해 EPS에 위험한 명령 전송 |
| Interface layer | COM Rx로 악성 TC 주입 |

### 3.1.1 Control Layer

공격자의 최종 목표를 정하고, 목표를 bus에서 이루는지 payload에서 이루는지 구분한다. bus는 payload를 제어하므로 bus에서 payload에 대한 목표를 이룰 수 있지만, 반대는 안 된다. bus-payload separation은 payload 장비의 고장이 bus로 번지지 않게 하려는 안전상의 이유에서 나왔다.

| 목표 | bus | payload |
|---|---|---|
| Denial of Service/Control | Denial of Service, Denial of Control | Denial of Service만 |
| Malicious Data Interaction | control data | payload data |
| Seizure of Control | arbitrary code execution 필요 | 해당 없음 |

#### Denial of Service/Control

DoS는 현재 위성에 대한 가장 흔한 공격 방식으로, 위성의 availability를 위협한다.

- Denial of Service: 위성이 임무(촬영, 통신 등)를 못 하게 만든다. bus와 payload 양쪽에서 가능하다.
- Denial of Control: 운영자가 위성에 명령을 내리지 못하게 만든다. 명령 경로인 GS -> COM -> CDHS가 모두 bus에 있으므로 bus를 통해서만 가능하다.

#### Malicious Data Interaction

위성 데이터를 빼내거나 조작한다.

| | control data | payload data |
|---|---|---|
| 위치 | bus (주로 CDHS) | payload |
| 내용 | flight-critical data(attitude, 궤도, command schedule 등), access protection secret(TC 인증 키 등) | 임무 결과물(촬영 영상, 측정값 등) |
| 접근 조건 | bus를 장악해야 함 | payload만 장악해도 됨 |

bus의 데이터는 모두 위성 제어와 관련되므로 control data라고 부른다. 논문은 데이터를 저장 위치가 아니라 누구의 데이터인지로 구분한다. 그래서 payload data가 downlink 전에 bus 저장 장치에 잠시 있더라도 payload data다.

#### Seizure of Control

위성의 제어권을 완전히 빼앗는다.

bus가 제공하는 TC는 CDHS의 OBSW가 처리하도록 미리 만들어 둔 명령 목록이다. 공격자가 인증을 뚫어 TC를 보낼 수 있게 되어도 목록에 있는 동작만 할 수 있다. 그래서 위성을 완전히 장악하려면 bus에서 arbitrary code execution이 가능한 취약점을 찾아야 한다.

payload에서 Seizure of Control이 해당 없는 이유는 제어가 bus -> payload 방향으로만 이뤄지기 때문이다. payload를 장악해도 payload만 가질 뿐이고, 위성 전체를 가지려면 bus를 장악해야 한다.

Seizure of Control은 위성 소유자만의 문제가 아니다. 위성에 thruster가 있다면 공격자는 Kessler Syndrome을 일으킬 수 있다. Kessler Syndrome은 위성 하나가 충돌해 생긴 파편이 다른 위성과 부딪혀 그 위성을 부수고 새 파편을 만드는 연쇄 반응이다. 시뮬레이션에 따르면 이 파편 때문에 수십 년 동안 우주에 접근하지 못할 수도 있다. 논문은 위성 해킹 한 번이 이런 결과를 낳을 수 있는데도 보안 커뮤니티가 이를 거의 다루지 않는다고 지적한다.

### 3.1.2 Components Layer

components layer는 2.2절의 위성 component에 Bus-Payload Link와 UDHS(Untrusted Data Handling System)를 더한 것이다. 각 component는 별개의 하드웨어일 수도 있고 하나의 펌웨어 이미지에 합쳐져 있을 수도 있지만, 여기서는 기능을 기준으로 나눈다.

bus와 payload는 component를 묶은 단위이고, component는 그 안의 CDHS, PDHS 같은 개별 요소다. ADCS와 EPS는 CDHS를 통해 제어되므로 따로 component로 두지 않는다.

#### 위협 분류 기준

각 component의 위협은 다음 네 기준으로 나눈다.

| 기준 | 설명 |
|---|---|
| (1) Integrity | 데이터와 control flow를 공격자가 바꿀 수 없어야 한다. |
| (2) Stability | Integrity의 하위 범주. 설계상 원래 있는 기능이지만, 공격자가 쓸 수 있으면 위성 운용을 위태롭게 하는 위협이다. |
| (3) Availability | 필요할 때 기능을 쓸 수 있어야 한다. |
| (4) Confidentiality | secret과 내부 정보가 유출되지 않아야 한다. |

component마다 네 기준을 하나씩 대입해 위협에 이름을 붙인다. 이렇게 하면 위협 유형을 빠뜨리지 않고 검토할 수 있고, 실제 위성에서 찾은 취약점을 같은 이름으로 분류할 수 있다.

#### COM / PLCOM

COM은 radio로 TC를 받고, PLCOM은 payload data나 payload용 TC를 받는다. 논문은 펌웨어 공격만 다루므로 둘의 위협을 Bypass Access Control 하나로 묶는다. Man-in-the-Middle 같은 RF 위협, timing side-channel 같은 cryptographic 위협, TC를 demodulation하거나 decoding하는 microcode를 노리는 위협이 여기에 들어간다.

COM과 PLCOM은 공격자가 지나가는 입구다. 공격자는 access control을 뚫어 위성이 자신의 패킷을 받아들이게 만들고, 실제로 장악하는 대상은 그 뒤에서 패킷을 파싱하는 CDHS나 PDHS다.

#### CDHS

CDHS는 TC를 받으면 펌웨어에서 그 TC에 연결된 함수를 실행해 위성에 즉시 또는 예약된 동작을 수행한다.

| 기준 | 위협 | 내용 |
|---|---|---|
| (1) Integrity | Vulnerable TCs | TC 처리 중의 buffer overflow, format string, use-after-free 같은 memory corruption 취약점. private data 접근, control flow hijacking, 정보 유출로 이어진다. |
| (2) Stability | Dangerous TCs | 보내기만 하면 access control, firmware update, control flow, critical data를 장악할 수 있는 TC. 주로 디버깅용으로 남아 있다. |
| (3) Availability | TC Suppression | 위성이 TC를 처리하지 못하게 만드는 위협. course correction처럼 시간이 중요한 동작이 있어 TC 처리는 항상 가능해야 한다. |
| (4) Confidentiality | Control Data Leak | TC로 access control secret 등을 유출하는 위협. 이를 통해 더 높은 권한을 얻으면 위성 전체가 장악될 수 있다. |

논문은 디버깅용 arbitrary memory write TC나 access control secret을 바꾸는 TC는 구현하지 말고, firmware image에 접근할 때는 signed image 같은 검증을 한 겹 더 두라고 권고한다. 논문은 이를 defense in depth라고 부른다.

#### Bus-Payload Link

payload가 bus와 상호작용하는 통로다. payload를 제어하려면 payload 장비 전원 제어 같은 bus의 monitoring, control 기능이 필요할 수 있기 때문이다.

논문은 이 링크를 신뢰 수준이 다른 계층 사이의 API 같은 접점이라고 표현한다. bus는 설계상 위성 전체를 지휘하고, payload는 이미 장악되었거나 신뢰할 수 없는 코드를 돌리고 있을 수 있다. user process가 kernel 기능을 syscall로만 호출하는 구조와 비슷하다. syscall에 취약점이 있으면 privilege escalation이 일어나듯, 이 링크가 허술하면 payload 장악이 bus 장악으로 이어진다.

bus가 장악된 경우는 따로 다루지 않는다. bus는 원래 payload를 제어할 수 있으므로, 그 시점에 공격자는 이미 위성 전체를 가진 것이다.

| 기준 | 위협 | 내용 |
|---|---|---|
| (1) Integrity | Vulnerable TC Interface | 링크는 자체 TC 없이 CDHS의 TC를 TC interface로 쓴다. 이 interface도 Vulnerable TC처럼 취약할 수 있다. |
| (2) Stability | Critical TC Interface | 장악되었을 수 있는 payload에 너무 넓은 명령 기능을 열어 주는 것. 예: unlimited power management, 무선 통신이 불가능하도록 attitude 변경 |
| (3) Availability | Link Suppression | PDHS나 UDHS가 여러 개일 때 하나가 다른 쪽의 링크 사용을 막는 것 |
| (4) Confidentiality | Payload Data Leak | 한 payload component가 다른 payload component 전용 정보를 빼내는 것 |

payload가 내리는 명령은 payload 장비에만 영향을 주고, bus가 언제든 되돌릴 수 있어야 한다.

Critical TC와 Dangerous TC의 차이는 다음과 같다.

| | Critical TC | Dangerous TC |
|---|---|---|
| 정체 | 위성 운용에 필요한 정상 명령 | 운용에 꼭 필요하지 않거나 너무 강력한 명령 |
| 예시 | attitude 변경, 전원 제어 | 디버깅용 arbitrary memory write, 인증 키 변경 |
| 문제 | payload에 열어 준 것 | 존재 자체 |
| 올바른 설계 | bus만 사용 | 없애거나 추가 인증, 검증 요구 |

Link Suppression은 한 위성 안에 payload 컴퓨터가 여러 개 있을 때의 이야기다. 예를 들어 카메라 담당 PDHS와 선박 신호 수신 담당 PDHS가 같은 통신선으로 bus와 연결되어 있을 때, 하나가 해킹당해 통신선을 쓰레기 데이터로 채우면 다른 PDHS도 bus와 통신할 수 없다. CAN이라면 한 노드가 가장 높은 우선순위 ID로 메시지를 계속 보내 다른 노드가 전송하지 못하게 만들 수 있다. payload component가 하나뿐이면 자기 자신만 막는 셈이라 링크의 위협으로 보지 않는다.

#### PDHS

payload의 데이터 처리 시스템이다. 임무에 따라 PLCOM에서 온 패킷으로 payload 명령도 처리한다. payload 기능을 제어하거나 User Segment에서 온 신뢰할 수 없는 데이터를 처리한다.

PDHS는 어떤 computing task든 할 수 있어 위성마다 하는 일이 다르다. CDHS는 TC 처리라는 역할이 정해져 있어 위협을 구체적으로 정할 수 있지만, PDHS는 범용 컴퓨터에 가까워질수록 위협이 일반적인 시스템 보안 위협과 같아진다. 그래서 논문은 PDHS의 위협 범주를 일부러 넓게 잡는다.

| 기준 | 위협 | 내용 |
|---|---|---|
| (1) Integrity | Vulnerable Data Processing | Vulnerable TC와 비슷한 모든 소프트웨어 취약점 |
| (2) Stability | Dangerous Payload Command | PDHS가 CDHS처럼 payload 명령을 처리할 때만 해당. Dangerous TC와 비슷하다 |
| (3) Availability | Payload Service Denial | 들어오는 패킷 처리를 막아 payload 서비스를 중단시키는 것 |
| (4) Confidentiality | Service Info Leak | 서비스 정보 유출 |

#### UDHS

UDHS는 2.2절에는 없고 3.1.2절에서 처음 나오는 component로, payload 사용자의 untrusted code를 위성에서 실행하는 isolated environment다.

- SataaS(Satellite-as-a-Service): 위성을 직접 만들지 않고 남의 위성 자원을 빌려 쓰는 서비스
- orbital cloud computing: 위성 위의 컴퓨터를 클라우드 서버처럼 써서 데이터를 우주에서 처리하는 개념

두 경우 모두 외부인이 만든 코드가 위성에서 실행되므로, 이를 PDHS의 일반 운용과 분리할 isolated environment가 필요하다. 기존 위성 구조에는 없던 component지만 논문은 이런 흐름을 반영해 추가했고, 사례 연구 대상 위성에서도 확인했다고 한다.

PDHS와 UDHS는 서로를 포함할 수 있다.

- PDHS 안에 UDHS가 있는 경우: PDHS가 OS를 돌리면서 isolated process 하나에서 외부 코드를 실행하고, 같은 환경에서 payload 데이터 처리도 한다. PC에서 내 프로그램을 돌리면서 VM으로 남의 프로그램을 돌리는 것과 비슷하다.
- UDHS 안에 PDHS가 있는 경우: isolated environment 안의 외부 프로그램이 수신 장비의 데이터를 받아 파싱한다. 공격자 입장에서는 이 프로그램이 PDHS와 같은 역할을 하므로 논문은 이를 UDHS-PDHS라고 부른다.

<figure>
  <img src="5.png" alt="">
</figure>

UDHS-PDHS는 PDHS와 같은 위협을 받지만, 뚫리더라도 공격자는 isolated environment 안에 있으므로 주 PDHS까지 가려면 isolation을 한 번 더 깨야 한다.

| 기준 | 위협 | 내용 |
|---|---|---|
| (1)+(2) Integrity + Stability | Escape to PDHS | isolation을 깨고 PDHS로 탈출 |
| (3) Availability | - | availability 의무는 UDHS-PDHS에만 있어 다루지 않음 |
| (4) Confidentiality | Side Channel Attacks | isolated host environment에서 정보를 빼냄 |

### 3.1.3 Interface Layer

taxonomy의 가장 아래 layer로, component가 다른 component나 위성 밖과 주고받을 때 거치는 입구다. 여기서 layer는 OSI 같은 통신 계층이 아니라 taxonomy의 세 번째 단계라는 뜻이다.

| 종류 | 이름 | 연결 대상 |
|---|---|---|
| external interface | External Rx (COM Rx, PLCOM Rx 등) | component - 위성 밖(GS 등) |
| internal interface | Data Fetcher (TC Fetcher, PD Fetcher, UD Fetcher 등) | component - component |

- 모든 interface는 parent component를 하나만 가진다. 예: COM Rx의 parent는 COM
- component 하나는 interface를 여러 개 가질 수 있다. 예: CDHS는 COM용과 Bus-Payload Link용 TC Fetcher를 따로 가질 수 있다.

COM Rx는 위성 밖 신호가 처음 들어오는 수신 지점이고, COM은 그 뒤에서 decoding과 access control을 하는 component다.

#### External Rx

위성 밖에서 데이터를 받는 radio 또는 optical receiver다. 이 모델에서 External Rx는 소프트웨어 없이 하드웨어로만 동작한다고 본다. 공격할 소프트웨어가 없으니 남는 위협은 jamming 같은 electromagnetic/RF 위협뿐이고, 이는 펌웨어 공격을 다루는 이 논문의 범위 밖이라 분석하지 않는다. 대신 공격자가 어디로 들어오는지 표시하는 데 쓰고, 4장의 attacker model에서 활용한다.

demodulation용 microcode처럼 수신 경로에 펌웨어가 있다면, 그 버그로 인증을 건너뛸 수 있으므로 Rx가 아니라 COM의 일부로 보고 Bypass Access Control에 포함한다.

또한 component가 파싱하는 structured data를 받는 receiver만 다룬다. 방사선 측정 장비나 온도계는 bit-exact structured data를 받을 수 없어 제외한다.

#### Data Fetcher

한 component에서 데이터를 가져와 parent component에 넘기는 internal interface다. Data 자리에는 TC처럼 구체적인 이름을 넣는다. 받은 트래픽을 넘기기만 하므로 Integrity와 Availability 위협만 받는다. Confidentiality를 침해하려면 반대 방향의 return channel이 필요하기 때문이다.

| 기준 | 위협 | 내용 |
|---|---|---|
| (1)+(2) Integrity + Stability | Data Injection / Data Alteration | 데이터를 끼워 넣거나 바꾼다. 예: TC를 하나 보냈는데 공격자가 하나를 더 끼워 넣음 |
| (3) Availability | Data Flooding | 메모리가 적어 보통 ring buffer를 쓰는데, 패킷을 쏟아부어 처리되지 않은 패킷을 계속 밀어낸다 |
| (3) Availability | Forward Suppression | 과부하 말고 다른 방식으로 전달을 막는다 |

### 3.1 정리

Figure 3의 내용을 표로 정리하면 다음과 같다.

| Component | (1) Integrity | (2) Stability | (3) Availability | (4) Confidentiality |
|---|---|---|---|---|
| COM / PLCOM | Bypass Access Control | | | |
| CDHS | Vulnerable TCs | Dangerous TCs | TC Suppression | Control Data Leak |
| Bus-Payload Link | Vuln. TC Interface | Critical TC Interface | Link Suppression | Payload Data Leak |
| PDHS | Vuln. Data Processing | Dangerous Payload Cmd. | Payload Service Denial | Service Info Leak |
| UDHS | Escape to PDHS | | - | Side Channel Attacks |
| Data Fetcher | Data Injection / Alteration | | Data Flooding, Forward Suppression | - |

COM / PLCOM은 네 기준을 Bypass Access Control 하나로 묶고, UDHS와 Data Fetcher는 Integrity와 Stability를 하나로 묶는다.

## week 2 정리

### 한 줄 요약

위성은 운용을 맡는 bus와 임무를 맡는 payload로 나뉘고, 제어가 bus -> payload 방향으로만 이뤄지므로 bus, 그중 CDHS를 장악하면 위성 전체를 장악할 수 있다. 논문은 이 구조를 바탕으로 위성 펌웨어 위협을 control layer, components layer, interface layer의 3계층 taxonomy로 정리했다.

### 읽으면서 헷갈렸던 부분

- payload와 bus: 보안 분야의 exploit payload, data bus와 뜻이 다른 위성 용어다.
- PLCOM: 통신 장비지만 임무 수행용이라 payload에 속한다.
- Critical TC와 Dangerous TC: Critical TC는 payload에 열어 준 것이, Dangerous TC는 존재 자체가 문제다.
- UDHS: 2.2절의 위성 구조에는 없고 3.1.2절에서 추가된 component다.

### 다음 주차 계획

week 3에서는 4장과 5장을 읽을 예정이다.

4장에서는 논문이 가정하는 attacker model을 알아볼 예정이다. 공격자가 위성에 대해 어떤 지식과 접근 수준을 가진다고 보는지, 공격자 유형을 어떻게 나누는지 살펴볼 예정이다.

5장에서는 위성 3기(ESTCube-1, OPS-SAT, Flying Laptop)의 사례 연구를 읽을 예정이다. 이번 주에 정리한 taxonomy가 각 위성의 threat model에 어떻게 적용되는지, 어떤 취약점이 발견되었는지를 중심으로 정리할 예정이다.
