# TCP & UDP

> 전송 계층(4계층)의 두 핵심 프로토콜. TCP는 신뢰성, UDP는 속도를 우선한다.

## 1. TCP (Transmission Control Protocol)

**연결 지향적**, **신뢰성 있는** 데이터 전송 프로토콜.

### 특징

```mermaid
graph TB
    TCP["TCP 특징"]
    TCP --> CONN["연결 지향<br/>통신 전 연결 수립 필요"]
    TCP --> RELIABLE["신뢰성 보장<br/>데이터 손실 시 재전송"]
    TCP --> ORDER["순서 보장<br/>보낸 순서대로 도착"]
    TCP --> FLOW["흐름 제어<br/>수신자 처리 속도에 맞춤"]
    TCP --> CONG["혼잡 제어<br/>네트워크 과부하 방지"]

    style TCP fill:#0f3460,color:#fff
    style CONN fill:#d4edda
    style RELIABLE fill:#d4edda
    style ORDER fill:#d4edda
    style FLOW fill:#d4edda
    style CONG fill:#d4edda
```

### TCP 세그먼트 구조

```mermaid
graph TB
    subgraph TCP_HEADER["TCP 헤더 (20바이트)"]
        SRC_PORT["출발지 포트 (16bit)"]
        DST_PORT["목적지 포트 (16bit)"]
        SEQ["시퀀스 번호 (32bit)<br/>데이터 순서 관리"]
        ACK_NUM["ACK 번호 (32bit)<br/>다음에 기대하는 번호"]
        FLAGS["플래그: SYN, ACK, FIN, RST, PSH, URG"]
        WINDOW["윈도우 크기 (16bit)<br/>수신 가능 버퍼 크기"]
        CHECKSUM["체크섬 (16bit)<br/>오류 검출"]
    end

    style TCP_HEADER fill:#e8f4f8
```

### 주요 플래그

| 플래그 | 의미 | 사용 시점 |
|--------|------|----------|
| **SYN** | 연결 요청 | 연결 수립 시 (3-way handshake) |
| **ACK** | 확인 응답 | 데이터 수신 확인 |
| **FIN** | 연결 종료 | 연결 해제 시 (4-way handshake) |
| **RST** | 연결 강제 초기화 | 비정상 종료 |
| **PSH** | 즉시 전달 | 버퍼링 없이 바로 전달 |
| **URG** | 긴급 데이터 | 우선 처리 필요 |

## 2. TCP 3-Way Handshake (연결 수립) ⭐

TCP 통신을 시작하기 전 **연결을 수립**하는 과정. 3번의 패킷 교환이 필요하다.

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    Note over C: CLOSED
    Note over S: LISTEN (대기 중)

    C->>S: ① SYN (seq=100)
    Note over C: SYN_SENT
    Note over C,S: "나 연결하고 싶어"

    S->>C: ② SYN + ACK (seq=200, ack=101)
    Note over S: SYN_RECEIVED
    Note over C,S: "알겠어, 나도 연결하고 싶어"

    C->>S: ③ ACK (ack=201)
    Note over C: ESTABLISHED
    Note over S: ESTABLISHED
    Note over C,S: "좋아, 연결 완료!"

    C->>S: 데이터 전송 시작
```

### 각 단계 설명

| 단계 | 방향 | 플래그 | 의미 |
|------|------|--------|------|
| ① | 클라이언트 → 서버 | SYN | 연결 요청 |
| ② | 서버 → 클라이언트 | SYN + ACK | 요청 수락 + 나도 연결 원함 |
| ③ | 클라이언트 → 서버 | ACK | 확인, 연결 수립 완료 |

### 왜 2-way가 아니라 3-way인가?

```mermaid
graph TB
    subgraph 2way["2-Way (부족한 이유)"]
        A1["클라이언트 → SYN → 서버"] --> A2["서버 → ACK → 클라이언트"]
        A2 --> A3["서버는 클라이언트가<br/>ACK를 받았는지 모름 ❌"]
    end

    subgraph 3way["3-Way (올바른 방식)"]
        B1["클라이언트 → SYN"] --> B2["서버 → SYN+ACK"]
        B2 --> B3["클라이언트 → ACK"]
        B3 --> B4["양쪽 모두 연결 확인 ✅"]
    end

    style 2way fill:#ffe4e1
    style 3way fill:#d4edda
```

3-way가 필요한 이유는 **양방향 통신 가능**을 확인하기 위해서다. 클라이언트→서버, 서버→클라이언트 양쪽 모두 확인해야 한다.

## 3. TCP 4-Way Handshake (연결 해제) ⭐

TCP 연결을 **종료**하는 과정. 4번의 패킷 교환이 필요하다.

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    Note over C,S: ESTABLISHED (연결 중)

    C->>S: ① FIN
    Note over C: FIN_WAIT_1
    Note over C,S: "나 이제 끊을게"

    S->>C: ② ACK
    Note over S: CLOSE_WAIT
    Note over C: FIN_WAIT_2
    Note over C,S: "알겠어, 잠깐만"

    Note over S: 남은 데이터 전송 완료...

    S->>C: ③ FIN
    Note over S: LAST_ACK
    Note over C,S: "나도 준비 됐어, 끊자"

    C->>S: ④ ACK
    Note over C: TIME_WAIT
    Note over S: CLOSED
    Note over C: 일정 시간 후 CLOSED
```

### 왜 4-way인가? (3-way로 안 되는 이유)

```mermaid
graph TB
    subgraph 이유["FIN과 ACK를 분리하는 이유"]
        R1["클라이언트가 FIN 보냄"]
        R2["서버: ACK는 바로 보냄"]
        R3["서버: 아직 보낼 데이터가<br/>남아있을 수 있음 ⏳"]
        R4["서버: 데이터 다 보낸 후<br/>FIN을 보냄"]
        R1 --> R2 --> R3 --> R4
    end

    style 이유 fill:#fff3cd
```

서버가 ACK와 FIN을 동시에 보낼 수 없는 이유는, ACK를 보낸 후에도 **아직 전송할 데이터가 남아있을 수 있기 때문**이다.

### TIME_WAIT 상태

```mermaid
graph LR
    FIN_ACK["④ ACK 전송"] --> TW["TIME_WAIT<br/>(일반적으로 2MSL 대기)"]
    TW --> CLOSED["CLOSED"]

    style TW fill:#fff3cd
    style CLOSED fill:#d4edda
```

TIME_WAIT가 필요한 이유:
1. **마지막 ACK 유실 대비**: 서버가 ACK를 못 받으면 FIN을 재전송함. 이때 응답하려면 아직 연결이 살아있어야 함
2. **지연 패킷 처리**: 이전 연결의 패킷이 새 연결에 영향을 주지 않도록 대기

## 4. TCP 흐름 제어 (Flow Control)

**수신자의 처리 속도**에 맞춰 송신자의 전송 속도를 조절하는 기법.

### Stop-and-Wait

```mermaid
sequenceDiagram
    participant S as 송신자
    participant R as 수신자

    S->>R: 패킷 1 전송
    Note over S: ⏳ ACK 올 때까지 대기
    R->>S: ACK 1
    S->>R: 패킷 2 전송
    Note over S: ⏳ ACK 올 때까지 대기
    R->>S: ACK 2
    S->>R: 패킷 3 전송
```

- 한 번에 하나씩 보내고 확인
- 단순하지만 **매우 비효율적** (대기 시간 낭비)

### Sliding Window ⭐

```mermaid
sequenceDiagram
    participant S as 송신자
    participant R as 수신자

    Note over R: 윈도우 크기 = 4
    S->>R: 패킷 1
    S->>R: 패킷 2
    S->>R: 패킷 3
    S->>R: 패킷 4
    Note over S: 윈도우만큼 연속 전송!
    R->>S: ACK 5 (4번까지 받음)
    Note over S: 윈도우 슬라이드
    S->>R: 패킷 5
    S->>R: 패킷 6
    S->>R: 패킷 7
    S->>R: 패킷 8
```

```mermaid
graph LR
    subgraph 윈도우["슬라이딩 윈도우 (크기=4)"]
        P1["✅ 1"] --> P2["✅ 2"] --> P3["✅ 3"] --> P4["✅ 4"]
        P4 --> P5["⏳ 5"] --> P6["⏳ 6"] --> P7["⏳ 7"] --> P8["⏳ 8"]
    end

    style P1 fill:#d4edda
    style P2 fill:#d4edda
    style P3 fill:#d4edda
    style P4 fill:#d4edda
    style P5 fill:#fff3cd
    style P6 fill:#fff3cd
    style P7 fill:#fff3cd
    style P8 fill:#fff3cd
```

- ACK를 기다리지 않고 **윈도우 크기만큼 연속 전송**
- 수신자가 ACK에 윈도우 크기를 담아 보내면, 송신자가 속도를 조절
- Stop-and-Wait보다 **훨씬 효율적**

## 5. TCP 혼잡 제어 (Congestion Control)

**네트워크 전체의 혼잡 상태**에 따라 전송 속도를 조절하는 기법.

> 흐름 제어 = 수신자 기준 / 혼잡 제어 = 네트워크 기준

### 혼잡 제어 알고리즘

```mermaid
graph LR
    subgraph 혼잡제어["혼잡 제어 단계"]
        SS["Slow Start<br/>지수적 증가<br/>(1→2→4→8...)"]
        CA["Congestion Avoidance<br/>선형 증가<br/>(8→9→10→11...)"]
        FR["Fast Recovery<br/>혼잡 감지 시<br/>빠른 복구"]
    end
    SS -->|"임계값 도달"| CA
    CA -->|"패킷 손실"| FR
    FR -->|"복구 완료"| CA

    style SS fill:#d4edda
    style CA fill:#fff3cd
    style FR fill:#ffe4e1
```

### Slow Start → Congestion Avoidance 과정

```mermaid
graph TB
    subgraph 과정["혼잡 윈도우 변화"]
        T1["cwnd=1<br/>1개 전송"]
        T2["cwnd=2<br/>2개 전송"]
        T3["cwnd=4<br/>4개 전송"]
        T4["cwnd=8<br/>임계값 도달!"]
        T5["cwnd=9<br/>선형 증가"]
        T6["cwnd=10"]
        T7["💥 혼잡 감지<br/>(패킷 손실)"]
        T8["cwnd 절반으로<br/>다시 시작"]

        T1 --> T2 --> T3 --> T4 --> T5 --> T6 --> T7 --> T8
    end

    style T1 fill:#d4edda
    style T2 fill:#d4edda
    style T3 fill:#d4edda
    style T4 fill:#fff3cd
    style T5 fill:#fff3cd
    style T6 fill:#fff3cd
    style T7 fill:#ff6b6b,color:#fff
    style T8 fill:#ffe4e1
```

| 단계 | 동작 | 윈도우 변화 |
|------|------|-----------|
| Slow Start | 지수적 증가 (2배씩) | 1→2→4→8→16... |
| Congestion Avoidance | 선형 증가 (+1씩) | 16→17→18→19... |
| 혼잡 감지 | 패킷 손실 발생 | 윈도우 크기 절반으로 축소 |
| Fast Recovery | 빠른 복구 | 절반에서 선형 증가 재시작 |

## 6. UDP (User Datagram Protocol)

**비연결형**, **신뢰성 없는** 데이터 전송 프로토콜. 빠르고 가볍다.

### 특징

```mermaid
graph TB
    UDP_FEAT["UDP 특징"]
    UDP_FEAT --> NOCONN["비연결<br/>handshake 없음"]
    UDP_FEAT --> NOREL["비신뢰성<br/>손실되어도 재전송 X"]
    UDP_FEAT --> NOORDER["순서 보장 X<br/>보낸 순서대로 안 올 수 있음"]
    UDP_FEAT --> FAST["빠름<br/>오버헤드 적음"]
    UDP_FEAT --> BROAD["브로드캐스트 지원<br/>1:N 전송 가능"]

    style UDP_FEAT fill:#0f3460,color:#fff
    style NOCONN fill:#e8f4f8
    style NOREL fill:#e8f4f8
    style NOORDER fill:#e8f4f8
    style FAST fill:#e8f4f8
    style BROAD fill:#e8f4f8
```

### UDP 데이터그램 구조

```mermaid
graph TB
    subgraph UDP_HEADER["UDP 헤더 (8바이트)"]
        U_SRC["출발지 포트 (16bit)"]
        U_DST["목적지 포트 (16bit)"]
        U_LEN["길이 (16bit)"]
        U_CHK["체크섬 (16bit)"]
    end

    style UDP_HEADER fill:#e8f4f8
```

TCP 헤더(20바이트)보다 **훨씬 작다**(8바이트). 연결 관리, 순서, 재전송 같은 기능이 없어서 가볍다.

### UDP 전송 방식

```mermaid
sequenceDiagram
    participant S as 송신자
    participant R as 수신자

    S->>R: 데이터 1 전송
    S->>R: 데이터 2 전송
    S->>R: 데이터 3 전송
    Note over S: ACK 안 기다림!
    Note over R: 2번이 유실되어도<br/>송신자는 모름
```

### UDP 사용 사례

| 사용 사례 | 이유 |
|----------|------|
| 실시간 스트리밍 (유튜브, 넷플릭스) | 약간의 손실보다 지연이 더 문제 |
| 온라인 게임 | 실시간성이 중요 |
| VoIP (인터넷 전화) | 지연 최소화 |
| DNS 조회 | 짧은 데이터, 빠른 응답 필요 |
| IoT 센서 데이터 | 가벼운 프로토콜 필요 |

## 7. TCP vs UDP 비교 ⭐

```mermaid
graph LR
    subgraph TCP영역["TCP"]
        TCP_BOX["신뢰성 ✅<br/>연결 지향 ✅<br/>순서 보장 ✅<br/>느림 ❌<br/>헤더 20바이트"]
    end
    subgraph UDP영역["UDP"]
        UDP_BOX["신뢰성 ❌<br/>비연결 ❌<br/>순서 보장 ❌<br/>빠름 ✅<br/>헤더 8바이트"]
    end

    style TCP영역 fill:#d4edda
    style UDP영역 fill:#e8f4f8
```

| 구분 | TCP | UDP |
|------|-----|-----|
| 연결 | 연결 지향 (3-way handshake) | 비연결 |
| 신뢰성 | 보장 (재전송) | 미보장 |
| 순서 | 보장 | 미보장 |
| 속도 | 상대적으로 느림 | 빠름 |
| 헤더 크기 | 20바이트 | 8바이트 |
| 흐름 제어 | 있음 | 없음 |
| 혼잡 제어 | 있음 | 없음 |
| 전송 방식 | 1:1 (유니캐스트) | 1:1, 1:N, N:N |
| 사용 예 | HTTP, 이메일, 파일 전송 | 스트리밍, DNS, 게임 |

### 비유

```mermaid
graph LR
    subgraph TCP비유["TCP = 등기 우편 📮"]
        T1["배달 확인"]
        T2["분실 시 재배송"]
        T3["순서대로 도착"]
        T4["느리지만 확실"]
    end
    subgraph UDP비유["UDP = 전단지 뿌리기 📄"]
        U1["받았는지 확인 안 함"]
        U2["분실되면 끝"]
        U3["순서 상관없음"]
        U4["빠르고 대량 전송"]
    end

    style TCP비유 fill:#d4edda
    style UDP비유 fill:#e8f4f8
```

## 8. TCP 상태 다이어그램

TCP 연결의 전체 상태 전이.

```mermaid
stateDiagram-v2
    [*] --> CLOSED
    CLOSED --> LISTEN: 서버 대기
    CLOSED --> SYN_SENT: SYN 전송 (클라이언트)

    LISTEN --> SYN_RECEIVED: SYN 수신, SYN+ACK 전송
    SYN_SENT --> ESTABLISHED: SYN+ACK 수신, ACK 전송
    SYN_RECEIVED --> ESTABLISHED: ACK 수신

    ESTABLISHED --> FIN_WAIT_1: FIN 전송 (능동 종료)
    ESTABLISHED --> CLOSE_WAIT: FIN 수신 (수동 종료)

    FIN_WAIT_1 --> FIN_WAIT_2: ACK 수신
    FIN_WAIT_2 --> TIME_WAIT: FIN 수신, ACK 전송
    CLOSE_WAIT --> LAST_ACK: FIN 전송
    LAST_ACK --> CLOSED: ACK 수신

    TIME_WAIT --> CLOSED: 2MSL 타임아웃
```

## 핵심 요약

| 개념 | 핵심 |
|------|------|
| TCP | 신뢰성, 연결 지향, 순서 보장 |
| 3-Way Handshake | SYN → SYN+ACK → ACK (연결 수립) |
| 4-Way Handshake | FIN → ACK → FIN → ACK (연결 해제) |
| 흐름 제어 | 수신자 속도에 맞춤 (Sliding Window) |
| 혼잡 제어 | 네트워크 혼잡에 맞춤 (Slow Start) |
| UDP | 빠름, 비연결, 비신뢰성 |
| TCP vs UDP | 신뢰성 vs 속도 트레이드오프 |
