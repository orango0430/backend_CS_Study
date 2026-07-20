# HTTPS & DNS

> HTTPS는 HTTP에 보안(TLS)을 더한 프로토콜, DNS는 도메인을 IP로 변환하는 시스템.

## 1. 대칭키 & 공개키 (비대칭키)

HTTPS를 이해하려면 먼저 암호화 방식을 알아야 한다.

### 대칭키 암호화

**하나의 키**로 암호화와 복호화를 모두 수행.

```mermaid
sequenceDiagram
    participant A as 송신자
    participant B as 수신자

    Note over A,B: 같은 키(🔑)를 공유
    A->>A: 🔑 키로 암호화
    A->>B: 암호문 전송 (X#@$!%)
    B->>B: 🔑 같은 키로 복호화
    Note over B: 원본 데이터 획득
```

| 장점 | 단점 |
|------|------|
| 암복호화 속도 빠름 | 키를 어떻게 안전하게 전달하나? (키 배송 문제) |
| 구현 간단 | 통신 상대마다 다른 키 필요 |

대표 알고리즘: AES, DES, 3DES

### 공개키 (비대칭키) 암호화

**공개키**와 **개인키**, 2개의 키를 사용. 공개키로 암호화하면 개인키로만 복호화 가능.

```mermaid
sequenceDiagram
    participant A as 송신자
    participant B as 수신자

    Note over B: 🔓 공개키 / 🔐 개인키 보유
    B->>A: 🔓 공개키 공개
    A->>A: 🔓 공개키로 암호화
    A->>B: 암호문 전송 (X#@$!%)
    B->>B: 🔐 개인키로 복호화
    Note over B: 원본 데이터 획득
    Note over A,B: 개인키는 절대 공개하지 않음!
```

```mermaid
graph TB
    subgraph 핵심["공개키 암호화 핵심"]
        PUB["🔓 공개키<br/>(누구나 알 수 있음)"]
        PRI["🔐 개인키<br/>(소유자만 보유)"]
        PUB -->|"공개키로 암호화"| ENC["암호문"]
        ENC -->|"개인키로만 복호화"| DEC["원본"]
        PRI --> DEC
    end

    style PUB fill:#e8f4f8
    style PRI fill:#ffe4e1
    style ENC fill:#fff3cd
    style DEC fill:#d4edda
```

| 장점 | 단점 |
|------|------|
| 키 배송 문제 해결 | 속도 느림 (대칭키 대비) |
| 디지털 서명 가능 | 연산 비용 높음 |

대표 알고리즘: RSA, ECDSA

### 하이브리드 방식 (실제 HTTPS)

```mermaid
graph LR
    subgraph 실제["HTTPS = 공개키 + 대칭키 조합"]
        STEP1["1. 공개키로<br/>대칭키를 암호화해서 전달"]
        STEP2["2. 이후 통신은<br/>대칭키로 암호화 (빠름)"]
        STEP1 --> STEP2
    end

    style 실제 fill:#d4edda
```

공개키 방식으로 **대칭키를 안전하게 교환**한 뒤, 이후 통신은 **빠른 대칭키 방식**으로 하는 것. 두 방식의 장점만 취한다.

## 2. HTTPS란?

**HTTP + TLS(SSL)** = HTTPS. HTTP 통신을 암호화하여 데이터를 보호한다.

```mermaid
graph LR
    subgraph HTTP_PLAIN["HTTP"]
        H1["평문 데이터<br/>누구나 볼 수 있음 ❌"]
    end
    subgraph HTTPS_ENC["HTTPS"]
        H2["암호화된 데이터<br/>중간에 탈취해도 해독 불가 ✅"]
    end

    style HTTP_PLAIN fill:#ffe4e1
    style HTTPS_ENC fill:#d4edda
```

### HTTPS가 보호하는 것

```mermaid
graph TB
    HTTPS_PROT["HTTPS가 보호하는 3가지"]
    HTTPS_PROT --> CONF["기밀성 (Confidentiality)<br/>데이터 암호화<br/>→ 도청 방지"]
    HTTPS_PROT --> INTEG["무결성 (Integrity)<br/>데이터 변조 감지<br/>→ 위변조 방지"]
    HTTPS_PROT --> AUTH["인증 (Authentication)<br/>서버 신원 확인<br/>→ 피싱 방지"]

    style HTTPS_PROT fill:#0f3460,color:#fff
    style CONF fill:#d4edda
    style INTEG fill:#d4edda
    style AUTH fill:#d4edda
```

## 3. SSL/TLS 인증서

서버가 "나는 진짜 이 도메인의 주인이다"를 증명하는 **디지털 인증서**.

### 인증서 발급 과정

```mermaid
sequenceDiagram
    participant S as 서버 (example.com)
    participant CA as 인증 기관 (CA)

    S->>S: 공개키/개인키 쌍 생성
    S->>CA: 인증서 발급 요청<br/>(도메인, 공개키 포함)
    CA->>CA: 도메인 소유 검증
    CA->>CA: CA의 개인키로 서명
    CA->>S: SSL 인증서 발급
    Note over S: 인증서에 포함된 것:<br/>도메인, 서버 공개키,<br/>CA 서명, 유효기간
```

### 인증서 신뢰 체인

```mermaid
graph TB
    ROOT["Root CA<br/>(최상위 인증 기관)<br/>브라우저에 내장"]
    INTER["Intermediate CA<br/>(중간 인증 기관)"]
    CERT["서버 인증서<br/>(example.com)"]

    ROOT -->|"서명"| INTER
    INTER -->|"서명"| CERT

    style ROOT fill:#0f3460,color:#fff
    style INTER fill:#e8f4f8
    style CERT fill:#d4edda
```

브라우저는 Root CA 목록을 내장하고 있어서, 인증서 체인을 따라 올라가며 신뢰 여부를 확인한다.

## 4. TLS Handshake ⭐

HTTPS 연결 시 클라이언트와 서버가 **암호화 방식을 합의**하고 **대칭키를 교환**하는 과정.

### TLS 1.2 Handshake

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    Note over C,S: TCP 3-Way Handshake 완료 후

    C->>S: ① ClientHello<br/>(지원 암호 목록, 랜덤 값)
    S->>C: ② ServerHello<br/>(선택한 암호, 랜덤 값)
    S->>C: ③ 서버 인증서 전송<br/>(공개키 포함)
    S->>C: ④ ServerHelloDone

    C->>C: 인증서 검증 (CA 서명 확인)
    C->>C: Pre-Master Secret 생성
    C->>S: ⑤ Pre-Master Secret<br/>(서버 공개키로 암호화)

    Note over C,S: 양쪽 모두 동일한 대칭키 생성<br/>(랜덤 값 + Pre-Master Secret)

    C->>S: ⑥ ChangeCipherSpec + Finished
    S->>C: ⑦ ChangeCipherSpec + Finished

    Note over C,S: ✅ 이후 모든 통신은 대칭키로 암호화
```

### TLS 1.3 Handshake (개선)

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    C->>S: ClientHello + 키 공유 정보
    Note over C,S: 1-RTT (왕복 1번)
    S->>C: ServerHello + 인증서 + Finished
    C->>S: Finished

    Note over C,S: ✅ 암호화 통신 시작
```

### TLS 1.2 vs 1.3

| 구분 | TLS 1.2 | TLS 1.3 |
|------|---------|---------|
| Handshake | 2-RTT | **1-RTT** |
| 0-RTT 재연결 | 미지원 | **지원** |
| 암호 스위트 | 다양 (일부 취약) | 안전한 것만 허용 |
| 성능 | 상대적 느림 | 빠름 |

### 전체 HTTPS 연결 과정

```mermaid
graph TB
    A["1. TCP 3-Way Handshake<br/>(연결 수립)"] --> B["2. TLS Handshake<br/>(암호화 합의, 대칭키 교환)"]
    B --> C["3. 암호화된 HTTP 통신<br/>(대칭키로 암호화)"]
    C --> D["4. 연결 종료<br/>(TCP 4-Way Handshake)"]

    style A fill:#e8f4f8
    style B fill:#fff3cd
    style C fill:#d4edda
    style D fill:#e8f4f8
```

## 5. HTTP vs HTTPS

| 구분 | HTTP | HTTPS |
|------|------|-------|
| 포트 | 80 | 443 |
| 암호화 | 없음 (평문) | TLS 암호화 |
| 인증서 | 불필요 | SSL 인증서 필요 |
| 속도 | 빠름 | 약간 느림 (handshake 비용) |
| URL | `http://` | `https://` |
| 보안 | 도청/변조 가능 | 도청/변조 불가 |

## 6. DNS (Domain Name System) ⭐

도메인 이름을 **IP 주소**로 변환하는 시스템. 인터넷의 전화번호부.

```mermaid
graph LR
    DOMAIN["www.google.com<br/>(사람이 읽기 쉬움)"]
    DNS["DNS"]
    IP["142.250.196.100<br/>(컴퓨터가 이해)"]

    DOMAIN -->|"DNS 조회"| DNS
    DNS -->|"IP 반환"| IP

    style DNS fill:#0f3460,color:#fff
    style DOMAIN fill:#e8f4f8
    style IP fill:#d4edda
```

### 도메인 구조

```mermaid
graph LR
    subgraph 도메인구조["www.example.com 구조"]
        ROOT[". (루트)"]
        TLD[".com<br/>Top-Level Domain"]
        SLD["example<br/>Second-Level Domain"]
        SUB["www<br/>서브도메인"]
    end

    ROOT --> TLD --> SLD --> SUB

    style ROOT fill:#0f3460,color:#fff
    style TLD fill:#e8f4f8
    style SLD fill:#fff3cd
    style SUB fill:#d4edda
```

도메인은 오른쪽에서 왼쪽으로 읽는다: `. → com → example → www`

### DNS 서버 종류

```mermaid
graph TB
    DNS_TYPES["DNS 서버 종류"]
    DNS_TYPES --> RECUR["재귀 DNS 서버<br/>(Recursive Resolver)<br/>클라이언트 요청을 대신 조회"]
    DNS_TYPES --> ROOT_S["루트 DNS 서버<br/>13개 존재<br/>TLD 서버 위치를 알려줌"]
    DNS_TYPES --> TLD_S["TLD DNS 서버<br/>.com, .net, .kr 등 관리<br/>권한 서버 위치를 알려줌"]
    DNS_TYPES --> AUTH["권한 DNS 서버<br/>(Authoritative)<br/>실제 IP 주소를 보유"]

    style DNS_TYPES fill:#0f3460,color:#fff
    style RECUR fill:#d4edda
    style ROOT_S fill:#e8f4f8
    style TLD_S fill:#fff3cd
    style AUTH fill:#ffc9c9
```

## 7. DNS 조회 과정 ⭐

브라우저에 `www.example.com`을 입력했을 때 IP를 찾는 전체 과정.

```mermaid
sequenceDiagram
    participant B as 브라우저
    participant LC as 로컬 캐시
    participant RR as 재귀 DNS 서버<br/>(ISP 제공)
    participant ROOT as 루트 DNS
    participant TLD as TLD DNS (.com)
    participant AUTH as 권한 DNS<br/>(example.com)

    B->>LC: ① 로컬 캐시에 있나?
    LC-->>B: 없음

    B->>RR: ② www.example.com의 IP?
    RR->>ROOT: ③ .com은 어디서 관리?
    ROOT->>RR: ④ .com → TLD 서버 주소

    RR->>TLD: ⑤ example.com은 어디서 관리?
    TLD->>RR: ⑥ example.com → 권한 서버 주소

    RR->>AUTH: ⑦ www.example.com의 IP?
    AUTH->>RR: ⑧ IP = 93.184.216.34

    RR->>B: ⑨ IP = 93.184.216.34
    Note over B: 캐시에 저장 (TTL 동안)
    B->>B: ⑩ 해당 IP로 HTTP 요청
```

### DNS 캐싱

매번 이 과정을 거치면 느리니까, 여러 단계에서 **캐싱**을 한다.

```mermaid
graph TB
    CACHE["DNS 캐시 단계 (조회 순서)"]
    CACHE --> C1["1. 브라우저 캐시<br/>가장 먼저 확인"]
    CACHE --> C2["2. OS 캐시<br/>(/etc/hosts 등)"]
    CACHE --> C3["3. 라우터 캐시"]
    CACHE --> C4["4. ISP DNS 서버 캐시<br/>(재귀 DNS)"]
    CACHE --> C5["5. 캐시에 없으면<br/>루트부터 재귀 조회"]

    C1 --> C2 --> C3 --> C4 --> C5

    style CACHE fill:#0f3460,color:#fff
    style C1 fill:#d4edda
    style C2 fill:#d4edda
    style C3 fill:#e8f4f8
    style C4 fill:#e8f4f8
    style C5 fill:#fff3cd
```

각 캐시에는 **TTL(Time To Live)** 이 설정되어 있어서, 일정 시간이 지나면 캐시가 만료되고 다시 조회한다.

### DNS 레코드 타입

| 타입 | 설명 | 예시 |
|------|------|------|
| **A** | 도메인 → IPv4 주소 | example.com → 93.184.216.34 |
| **AAAA** | 도메인 → IPv6 주소 | example.com → 2606:2800:... |
| **CNAME** | 도메인 → 다른 도메인 (별칭) | www.example.com → example.com |
| **MX** | 메일 서버 지정 | example.com → mail.example.com |
| **NS** | 네임서버 지정 | example.com → ns1.example.com |
| **TXT** | 텍스트 정보 | SPF, DKIM 등 인증용 |

## 8. 브라우저에 URL을 입력하면? (전체 흐름)

```mermaid
sequenceDiagram
    participant U as 사용자
    participant B as 브라우저
    participant DNS as DNS
    participant S as 서버

    U->>B: https://www.example.com 입력

    Note over B,DNS: 1단계: DNS 조회
    B->>DNS: www.example.com의 IP?
    DNS->>B: IP = 93.184.216.34

    Note over B,S: 2단계: TCP 연결
    B->>S: TCP 3-Way Handshake
    S->>B: 연결 수립

    Note over B,S: 3단계: TLS Handshake
    B->>S: TLS Handshake (암호화 합의)
    S->>B: 인증서 + 대칭키 교환

    Note over B,S: 4단계: HTTP 요청/응답
    B->>S: GET / HTTP/1.1 (암호화)
    S->>B: HTTP 200 OK + HTML (암호화)

    Note over B: 5단계: 렌더링
    B->>U: 웹 페이지 표시
```

## 핵심 요약

| 개념 | 핵심 |
|------|------|
| 대칭키 | 하나의 키로 암복호화, 빠르지만 키 배송 문제 |
| 공개키 | 공개키/개인키 쌍, 느리지만 키 배송 문제 해결 |
| HTTPS | HTTP + TLS, 공개키로 대칭키 교환 후 대칭키로 통신 |
| TLS Handshake | 암호화 합의 + 대칭키 교환 과정 |
| 인증서 | CA가 서명한 서버 신원 증명서 |
| DNS | 도메인 → IP 변환 시스템 |
| DNS 조회 | 캐시 → 재귀 서버 → 루트 → TLD → 권한 서버 |
| 전체 흐름 | DNS 조회 → TCP → TLS → HTTP → 렌더링 |
