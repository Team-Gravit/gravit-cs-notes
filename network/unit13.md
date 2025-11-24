## 무선 네트워크(Wireless Network)

**무선 네트워크**는 유선 케이블 없이 전파나 적외선 등의 전자기파를 통해 데이터를 송수신하는 네트워크이다.

<br>

### 1. 무선 네트워크의 특징

**신호 세기 감소**

- 전파는 거리나 장애물(벽, 유리 등)에 의해 쉽게 약해짐
- 거리 제곱에 비례하여 세기가 감소하며 실내에서는 반사·흡수로 인한 손실도 큼

<br>

**외부 노이즈 영향**

- 주변 기기나 다른 채널의 간섭(Interference)이나 잡음(Noise)에 영향을 받음
- 유선보다 안정성이 낮고 전송 품질이 변동되기 쉬움

<br>

**이동성(Mobility)**

- 사용자가 이동하면서도 네트워크 연결을 유지할 수 있음

<br>

### 2. 무선 네트워크 구성 요소

- **무선 단말(Host)**: 스마트폰, 노트북, IoT 기기 등 데이터를 송수신하는 주체
- **기지국/AP(Access Point)**: 무선 단말과 유선 네트워크를 연결하는 중계 역할
- **무선 링크(Wireless Link)**: 전파를 통해 데이터를 전달하는 통신 경로

<br>

### 3. 무선 네트워크 구성 방식

**인프라스트럭처 방식(Infrastructure Mode)**

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/network/unit13/image1.png" width="100%">

- 무선 단말들이 **기지국(AP)** 또는 **액세스 포인트**를 통해 네트워크에 접속함
- 모든 데이터 통신은 **AP**나 기지국을 경유함
- 중앙 장치가 연결을 관리하므로 안정적이고 유선 네트워크와 연동이 용이함
- 예시: Wi-Fi(공유기), LTE, 5G

<br>

**애드혹 방식(Ad-hoc Mode)**

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/network/unit13/image2.png" width="100%">

- 기지국 없이 무선 단말들이 **서로 직접(Peer-to-Peer)** 연결되어 통신함
- 네트워크는 필요할 때마다 임시로 형성됨
- 별도 인프라 없이 빠르게 네트워크 구성이 가능함
- 예시: Bluetooth, Wi-Fi Direct

<br>

### 4. 무선 네트워크 종류

| 구분                                         | 통신 범위    | 특징 / 예시                            |
| -------------------------------------------- | ------------ | -------------------------------------- |
| **WPAN**(Wireless Personal Area Network)     | 수 m         | 개인 기기 간 연결, Bluetooth, ZigBee   |
| **WLAN**(Wireless Local Area Network)        | 수십~수백 m  | 가정·사무실 AP 기반 Wi-Fi(IEEE 802.11) |
| **WMAN**(Wireless Metropolitan Area Network) | 수 km        | 도시권 네트워크, WiMAX                 |
| **WWAN**(Wireless Wide Area Network)         | 수십~수백 km | 셀룰러 기반 이동통신, LTE, 5G          |

<br>

### 5. IEEE 802.11 표준

**IEEE 802.11**은 무선 LAN(WLAN)의 **물리 계층**과 **매체 접근 제어 계층**을 정의하는 표준이다.

<br>

**주요 특징**

- **CSMA/CA**(Carrier Sense Multiple Access with Collision Avoidance) 방식으로 충돌 회피
- **RTS/CTS**(Request to Send/Clear to Send)로 숨은 단말(Hidden Terminal) 문제 완화
- **ACK**(Acknowledgment)로 전송 성공 여부 확인

<br>

**주요 표준**

| 표준                  | 주파수 대역 | 최대 속도   | 특징                            |
| --------------------- | ----------- | ----------- | ------------------------------- |
| **802.11b**           | 2.4GHz      | 11Mbps      | 초기 표준, 간섭에 취약          |
| **802.11a**           | 5GHz        | 54Mbps      | 고속 전송, 전파 도달거리 짧음   |
| **802.11g**           | 2.4GHz      | 54Mbps      | b와 호환, 속도 향상             |
| **802.11n**           | 2.4/5GHz    | 600Mbps     | MIMO 다중 안테나 기술 도입      |
| **802.11ac**(Wi-Fi 5) | 5GHz        | 1Gbps 이상  | 고속 전송, 대용량 스트리밍 적합 |
| **802.11ax**(Wi-Fi 6) | 2.4/5GHz    | 10Gbps 근접 | OFDMA·MU-MIMO, 다수 단말 최적화 |

<br>

### 6. CSMA/CA

**CSMA/CA**는 여러 단말이 하나의 매체를 공유하면서 충돌을 회피하는 방식이다. 주로 Wi-Fi(802.11)에서 사용된다.

<br>

**CSMA/CD vs CSMA/CA**

- **CSMA/CD**(유선 이더넷): 충돌 발생 시 실시간 감지 후 중단 및 재전송
- **CSMA/CA**(무선 LAN): 무선 환경에서는 송신 중 충돌 감지가 어려우므로 **사전에 충돌을 회피**함

<br>

**Hidden Node 문제**

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/network/unit13/image3.png" width="100%">

무선 환경에서 **A와 C가 서로의 탐지 범위 밖**에 있지만 **B와는 통신 가능**한 경우, A와 C는 서로의 존재를 모르므로 충돌이 발생할 수 있다.

<br>

**RTS/CTS 방식으로 문제 보완**

<img src="https://raw.githubusercontent.com/Team-Gravit/gravit-images/main/network/unit13/image4.png" width="100%">

1. **채널 감지 + DIFS 대기**: 송신 노드가 채널이 조용한지 확인하고 **DIFS** 시간만큼 대기
2. **RTS 전송**: **RTS** 프레임으로 채널 사용 기간을 알림
3. **CTS 응답**: 수신 노드가 **CTS** 프레임으로 응답, 다른 노드들은 **NAV** 설정
4. **데이터 전송**: **SIFS** 후 데이터 프레임 전송
5. **ACK 전송**: 수신 노드가 **ACK**로 수신 확인
6. **경쟁 윈도우**: **NAV** 종료 후 랜덤 대기 시간을 두고 다시 경쟁

<br>

### 7. 무선랜 보안 기술

**인증 기술**

- **SSID**(Service Set Identifier): AP를 식별하는 네트워크 이름
- **MAC 주소 필터링**: MAC 주소 기반 접속 제어(위조에 취약)
- **EAP**(Extensible Authentication Protocol): 사용자 인증 프레임워크

<br>

**암호화 및 보안 표준**

- **WEP**(Wired Equivalent Privacy): 초기 암호화 기술, 보안 취약점이 많음
- **TKIP**(Temporal Key Integrity Protocol): WEP 개선 방식
- **CCMP**(Counter Mode with CBC-MAC Protocol): AES 블록 암호화 사용
- **WPA/WPA2/WPA3**(Wi-Fi Protected Access):
  - **WPA**: TKIP 기반
  - **WPA2**: AES/CCMP 기반
  - **WPA3**: 최신 표준, SAE 인증 및 GCMP-256 암호화

<br>

## 이동 통신(Mobile Communication)

**이동 통신**은 사용자가 이동 중에도 통신이 가능하도록 하는 무선 통신 기술이다. 단말기가 한 셀에서 다른 셀로 이동해도 **통화나 데이터 연결이 끊기지 않게** 한다.

<br>

### 1. 이동 통신 기본 개념

**셀(Cell)**

- 지역을 여러 **셀**로 나누어 각 셀마다 기지국을 설치함
- **주파수 재사용**이 가능하여 효율적인 대역 활용이 가능함

<br>

**기지국(Base Station)**

- 단말기와 직접 무선 신호를 주고받는 장비
- 각 셀에 설치되어 접속, 송수신, 전력 제어 등을 담당함

<br>

**기지국 제어기(BSC/RNC/MME)**

- 여러 기지국을 관리하며 단말 이동 시 연결을 유지하도록 제어함
- 이동성 관리, 핸드오프, 자원 할당 등을 담당함

<br>

**코어 네트워크(Core Network)**

- 통신망의 중심으로 음성·데이터 트래픽을 인터넷이나 다른 통신망으로 연결함

<br>

### 2. 이동 통신 핵심 기능

**위치 등록(Location Registration)**

- 단말기가 새로운 셀에 진입하면 네트워크에 위치 정보를 등록함
- 네트워크는 이 정보로 단말을 찾고 적절한 기지국으로 연결함

<br>

**핸드오프(Handoff/Handover)**

- 사용자가 한 셀에서 다른 셀로 이동할 때 통신이 끊기지 않도록 **연결을 인접 셀로 전환**함
- **Hard Handoff**: 기존 연결을 끊고 새 셀로 연결(Break-Before-Make)
- **Soft Handoff**: 두 셀과 동시 연결(Make-Before-Break, CDMA 계열)

<br>

**이동성 관리(Mobility Management)**

- 단말 위치를 지속적으로 추적하여 이동 중에도 통신을 유지함
- 위치 관리 + 핸드오프 관리로 구성됨

<br>

**로밍(Roaming)**

- 사용자가 자신의 통신 사업자 영역을 벗어나 다른 사업자 망을 이용해 통신함
- 홈 네트워크와 방문 네트워크가 협력하여 인증 및 과금 정보를 교환함

<br>

### 3. 이동 통신 구조

| 구분          | 구성 요소               | 설명                                              |
| ------------- | ----------------------- | ------------------------------------------------- |
| **무선 구간** | 단말기, 기지국          | 무선 주파수로 신호 송수신                         |
| **제어 구간** | 기지국 제어기           | 무선 자원 관리, 핸드오프, 인증                    |
| **코어 구간** | MSC, SGSN, GGSN, EPC 등 | 전체 통신 제어, 데이터 라우팅, 외부 네트워크 연결 |

<br>

### 4. 이동 통신 세대별 발전

| 세대        | 주요 기술            | 특징                             |
| ----------- | -------------------- | -------------------------------- |
| **1G**      | 아날로그 음성        | 음성 전용, 보안 취약             |
| **2G**      | GSM, CDMA            | 디지털 음성, SMS                 |
| **3G**      | WCDMA, HSPA          | 음성+데이터 통합, 인터넷 접속    |
| **4G(LTE)** | OFDMA                | All-IP 기반, 고속 데이터 통신    |
| **5G**      | mmWave, Massive MIMO | 초고속·초저지연·초연결, IoT 대응 |
