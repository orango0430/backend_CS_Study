# HTTP (HyperText Transfer Protocol)

> 웹에서 클라이언트와 서버가 통신하는 프로토콜. 요청-응답 구조로 동작한다.

## 1. HTTP란?

```mermaid
graph LR
    C["클라이언트<br/>(브라우저)"] -->|"HTTP 요청"| S["서버"]
    S -->|"HTTP 응답"| C

    style C fill:#e8f4f8
    style S fill:#d4edda
```

### HTTP 특징

```mermaid
graph TB
    HTTP["HTTP 특징"]
    HTTP --> CS["클라이언트-서버 구조<br/>요청/응답으로 동작"]
    HTTP --> STATELESS["무상태 (Stateless)<br/>각 요청은 독립적<br/>이전 요청 기억 안 함"]
    HTTP --> CONN["비연결성 (Connectionless)<br/>응답 후 연결 끊음<br/>(HTTP/1.0 기준)"]

    style HTTP fill:#0f3460,color:#fff
    style CS fill:#e8f4f8
    style STATELESS fill:#fff3cd
    style CONN fill:#e8f4f8
```

### Stateless (무상태)

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    C->>S: 요청 1: 로그인 (id=user1)
    S->>C: 응답 1: 로그인 성공

    C->>S: 요청 2: 마이페이지 보여줘
    S->>C: 응답 2: 너 누구야? ❌
    Note over S: 이전 요청을 기억 못 함
    Note over C,S: → 쿠키/세션/토큰으로 해결
```

서버가 클라이언트의 상태를 저장하지 않으므로, 매 요청마다 필요한 정보를 모두 포함해야 한다.

## 2. HTTP 요청/응답 구조

### HTTP 요청 (Request)

```mermaid
graph TB
    subgraph 요청["HTTP 요청 구조"]
        RL["요청 라인<br/>GET /members/1 HTTP/1.1"]
        HD["헤더<br/>Host: example.com<br/>Content-Type: application/json<br/>Authorization: Bearer xxx"]
        BLANK["빈 줄"]
        BODY["본문 (Body)<br/>{name: 김사자, age: 25}"]
    end

    RL --> HD --> BLANK --> BODY

    style 요청 fill:#e8f4f8
    style RL fill:#d4edda
```

### HTTP 응답 (Response)

```mermaid
graph TB
    subgraph 응답["HTTP 응답 구조"]
        SL["상태 라인<br/>HTTP/1.1 200 OK"]
        HD2["헤더<br/>Content-Type: application/json<br/>Content-Length: 50"]
        BLANK2["빈 줄"]
        BODY2["본문 (Body)<br/>{id: 1, name: 김사자}"]
    end

    SL --> HD2 --> BLANK2 --> BODY2

    style 응답 fill:#d4edda
    style SL fill:#e8f4f8
```

## 3. HTTP 메서드 ⭐

클라이언트가 서버에게 **어떤 동작**을 원하는지 나타내는 방법.

```mermaid
graph TB
    METHODS["HTTP 메서드"]
    METHODS --> GET["GET<br/>조회"]
    METHODS --> POST["POST<br/>생성"]
    METHODS --> PUT["PUT<br/>전체 수정"]
    METHODS --> PATCH["PATCH<br/>부분 수정"]
    METHODS --> DELETE["DELETE<br/>삭제"]

    style METHODS fill:#0f3460,color:#fff
    style GET fill:#d4edda
    style POST fill:#fff3cd
    style PUT fill:#e8f4f8
    style PATCH fill:#e8f4f8
    style DELETE fill:#ffe4e1
```

| 메서드 | 역할 | 요청 Body | 멱등성 | 안전성 |
|--------|------|-----------|--------|--------|
| **GET** | 리소스 조회 | 없음 | ✅ | ✅ |
| **POST** | 리소스 생성 | 있음 | ❌ | ❌ |
| **PUT** | 리소스 전체 수정 | 있음 | ✅ | ❌ |
| **PATCH** | 리소스 부분 수정 | 있음 | ❌ | ❌ |
| **DELETE** | 리소스 삭제 | 없음 | ✅ | ❌ |

### 멱등성 (Idempotent)

같은 요청을 여러 번 보내도 **결과가 동일**한 성질.

```mermaid
graph LR
    subgraph 멱등["멱등 (GET, PUT, DELETE)"]
        I1["GET /members/1"] --> I2["결과: 김사자"]
        I3["GET /members/1"] --> I4["결과: 김사자 (동일)"]
    end
    subgraph 비멱등["비멱등 (POST)"]
        N1["POST /members"] --> N2["결과: 회원 1 생성"]
        N3["POST /members"] --> N4["결과: 회원 2 생성 (다름!)"]
    end

    style 멱등 fill:#d4edda
    style 비멱등 fill:#ffe4e1
```

### 안전성 (Safe)

서버의 **상태를 변경하지 않는** 메서드. GET, HEAD, OPTIONS만 안전하다.

## 4. HTTP 상태 코드 ⭐

서버가 요청을 어떻게 처리했는지 알려주는 3자리 숫자.

```mermaid
graph TB
    STATUS["HTTP 상태 코드"]
    STATUS --> S1XX["1xx 정보<br/>요청 처리 중"]
    STATUS --> S2XX["2xx 성공 ✅<br/>요청 정상 처리"]
    STATUS --> S3XX["3xx 리다이렉션<br/>추가 동작 필요"]
    STATUS --> S4XX["4xx 클라이언트 에러 ❌<br/>요청 잘못됨"]
    STATUS --> S5XX["5xx 서버 에러 💀<br/>서버 처리 실패"]

    style STATUS fill:#0f3460,color:#fff
    style S2XX fill:#d4edda
    style S3XX fill:#fff3cd
    style S4XX fill:#ffe4e1
    style S5XX fill:#ff6b6b,color:#fff
```

### 자주 쓰이는 상태 코드

| 코드 | 이름 | 설명 |
|------|------|------|
| **200** | OK | 요청 성공 |
| **201** | Created | 리소스 생성 성공 |
| **204** | No Content | 성공, 응답 본문 없음 |
| **301** | Moved Permanently | 영구 이동 (URL 변경) |
| **302** | Found | 임시 이동 |
| **304** | Not Modified | 캐시 사용 (수정 없음) |
| **400** | Bad Request | 잘못된 요청 (문법 오류 등) |
| **401** | Unauthorized | 인증 필요 (로그인 안 됨) |
| **403** | Forbidden | 권한 없음 (인가 실패) |
| **404** | Not Found | 리소스 없음 |
| **405** | Method Not Allowed | 허용되지 않는 메서드 |
| **500** | Internal Server Error | 서버 내부 오류 |
| **502** | Bad Gateway | 게이트웨이 오류 |
| **503** | Service Unavailable | 서비스 이용 불가 |

### 401 vs 403 차이

```mermaid
graph LR
    subgraph 401["401 Unauthorized"]
        A1["로그인 안 한 사용자"]
        A2["→ 인증(Authentication) 실패"]
        A3["→ 로그인 해주세요"]
    end
    subgraph 403["403 Forbidden"]
        B1["로그인은 했지만"]
        B2["→ 인가(Authorization) 실패"]
        B3["→ 접근 권한 없음"]
    end

    style 401 fill:#fff3cd
    style 403 fill:#ffe4e1
```

## 5. HTTP 헤더

요청/응답에 **부가 정보**를 담는 영역.

### 주요 요청 헤더

| 헤더 | 설명 | 예시 |
|------|------|------|
| Host | 요청 대상 서버 | `Host: www.example.com` |
| Content-Type | 본문 데이터 타입 | `Content-Type: application/json` |
| Accept | 원하는 응답 데이터 타입 | `Accept: application/json` |
| Authorization | 인증 정보 | `Authorization: Bearer {token}` |
| Cookie | 쿠키 전송 | `Cookie: sessionId=abc123` |
| User-Agent | 클라이언트 정보 | `User-Agent: Mozilla/5.0...` |

### 주요 응답 헤더

| 헤더 | 설명 | 예시 |
|------|------|------|
| Content-Type | 응답 데이터 타입 | `Content-Type: application/json` |
| Content-Length | 응답 본문 크기 | `Content-Length: 1024` |
| Set-Cookie | 쿠키 설정 | `Set-Cookie: sessionId=abc123` |
| Location | 리다이렉트 위치 | `Location: /new-page` |
| Cache-Control | 캐시 정책 | `Cache-Control: max-age=3600` |

## 6. 쿠키와 세션

HTTP는 Stateless라서 상태를 기억하지 못한다. 이를 보완하기 위해 쿠키와 세션을 사용한다.

### 쿠키 (Cookie)

**클라이언트(브라우저)** 에 저장되는 작은 데이터.

```mermaid
sequenceDiagram
    participant C as 브라우저
    participant S as 서버

    C->>S: 로그인 요청 (id, pw)
    S->>C: 응답 + Set-Cookie: sessionId=abc123
    Note over C: 브라우저가 쿠키 저장

    C->>S: 다음 요청 + Cookie: sessionId=abc123
    Note over S: 쿠키로 사용자 식별
    S->>C: 응답 (로그인 상태 유지)
```

### 세션 (Session)

**서버** 에 저장되는 사용자 상태 정보. 세션 ID만 쿠키로 주고받는다.

```mermaid
graph LR
    subgraph 클라이언트["브라우저"]
        COOKIE["쿠키<br/>sessionId=abc123"]
    end
    subgraph 서버["서버 메모리"]
        SESSION["세션 저장소<br/>abc123 → {userId: 1, role: ADMIN}"]
    end

    COOKIE -->|"sessionId 전송"| SESSION

    style 클라이언트 fill:#e8f4f8
    style 서버 fill:#d4edda
```

### 쿠키 vs 세션

| 구분 | 쿠키 | 세션 |
|------|------|------|
| 저장 위치 | 클라이언트 (브라우저) | 서버 |
| 보안 | 상대적으로 취약 (변조 가능) | 안전 (서버에 저장) |
| 용량 | 4KB 제한 | 제한 없음 (서버 메모리) |
| 속도 | 빠름 (로컬 저장) | 상대적으로 느림 (서버 조회) |
| 만료 | 설정 가능 (Expires) | 서버 설정에 따름 |

## 7. HTTP 버전별 차이 ⭐

### HTTP/1.0

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    C->>S: TCP 연결 (3-way handshake)
    C->>S: GET /page.html
    S->>C: 응답
    Note over C,S: 연결 끊김!

    C->>S: TCP 연결 (또 handshake)
    C->>S: GET /style.css
    S->>C: 응답
    Note over C,S: 또 끊김!

    C->>S: TCP 연결 (또 handshake)
    C->>S: GET /image.png
    S->>C: 응답
    Note over C,S: 매번 연결/해제 반복 (비효율)
```

### HTTP/1.1 — Keep-Alive ⭐

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    C->>S: TCP 연결 (3-way handshake)
    Note over C,S: 연결 유지 (Keep-Alive)

    C->>S: GET /page.html
    S->>C: 응답
    C->>S: GET /style.css
    S->>C: 응답
    C->>S: GET /image.png
    S->>C: 응답

    Note over C,S: 하나의 연결로 여러 요청 처리!
```

하지만 HTTP/1.1에는 **HOL Blocking (Head-of-Line Blocking)** 문제가 있다.

```mermaid
graph TB
    subgraph HOL["HOL Blocking 문제"]
        R1["요청 1: page.html (처리 중...)"]
        R2["요청 2: style.css ⏳ 대기"]
        R3["요청 3: image.png ⏳ 대기"]
        R1 --> R2 --> R3
    end

    style HOL fill:#ffe4e1
```

앞선 요청이 느리면 뒤의 요청이 모두 **대기**해야 한다.

### HTTP/2 — 멀티플렉싱 ⭐

```mermaid
graph LR
    subgraph HTTP1["HTTP/1.1 (순차)"]
        H1_1["요청1 → 응답1"]
        H1_2["요청2 → 응답2"]
        H1_3["요청3 → 응답3"]
        H1_1 --> H1_2 --> H1_3
    end
    subgraph HTTP2["HTTP/2 (병렬)"]
        H2_1["요청1 → 응답1"]
        H2_2["요청2 → 응답2"]
        H2_3["요청3 → 응답3"]
    end

    style HTTP1 fill:#ffe4e1
    style HTTP2 fill:#d4edda
```

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as 서버

    C->>S: 하나의 TCP 연결

    par 동시 전송
        C->>S: Stream 1: GET /page.html
        C->>S: Stream 2: GET /style.css
        C->>S: Stream 3: GET /image.png
    end

    par 동시 응답
        S->>C: Stream 2: style.css (먼저 완료)
        S->>C: Stream 1: page.html
        S->>C: Stream 3: image.png
    end

    Note over C,S: HOL Blocking 해결!
```

HTTP/2 주요 특징:
- **멀티플렉싱**: 하나의 연결에서 여러 요청/응답 동시 처리
- **헤더 압축**: HPACK 방식으로 헤더 크기 축소
- **서버 푸시**: 클라이언트가 요청하지 않은 리소스를 미리 전송
- **바이너리 프레이밍**: 텍스트가 아닌 바이너리로 전송 (파싱 효율 증가)

### HTTP/3 — QUIC (UDP 기반)

```mermaid
graph TB
    subgraph HTTP2스택["HTTP/2"]
        H2["HTTP/2"]
        TLS2["TLS 1.2+"]
        TCP2["TCP"]
    end
    subgraph HTTP3스택["HTTP/3"]
        H3["HTTP/3"]
        QUIC["QUIC<br/>(TLS 1.3 내장)"]
        UDP3["UDP"]
    end

    style HTTP2스택 fill:#e8f4f8
    style HTTP3스택 fill:#d4edda
```

| 구분 | HTTP/2 | HTTP/3 |
|------|--------|--------|
| 전송 계층 | TCP | **UDP (QUIC)** |
| 연결 수립 | TCP handshake + TLS handshake | **QUIC handshake (1-RTT)** |
| HOL Blocking | TCP 레벨에서 여전히 존재 | **완전 해결** (스트림 독립) |
| 연결 마이그레이션 | IP 바뀌면 재연결 | **Connection ID로 유지** |

HTTP/3가 UDP를 쓰는 이유: TCP는 프로토콜 자체를 수정할 수 없지만(OS 커널에 내장), UDP 위에 **QUIC**이라는 새 프로토콜을 올려서 TCP의 장점(신뢰성)을 구현하면서 단점(HOL Blocking)은 해결할 수 있다.

### 버전 비교 요약

| 버전 | 핵심 특징 | 한계 |
|------|----------|------|
| HTTP/1.0 | 요청마다 연결/해제 | 매번 handshake (느림) |
| HTTP/1.1 | Keep-Alive (연결 유지) | HOL Blocking |
| HTTP/2 | 멀티플렉싱, 헤더 압축 | TCP 레벨 HOL Blocking |
| HTTP/3 | QUIC(UDP 기반), 1-RTT | 아직 보급 진행 중 |

## 핵심 요약

| 개념 | 핵심 |
|------|------|
| HTTP 특징 | Stateless, 요청-응답 구조 |
| 메서드 | GET(조회), POST(생성), PUT(전체수정), PATCH(부분수정), DELETE(삭제) |
| 멱등성 | 같은 요청 여러 번 = 같은 결과 (GET, PUT, DELETE) |
| 상태 코드 | 2xx 성공, 3xx 리다이렉션, 4xx 클라이언트 에러, 5xx 서버 에러 |
| 쿠키/세션 | Stateless 보완 (쿠키=클라이언트, 세션=서버) |
| HTTP/1.1 | Keep-Alive, HOL Blocking 문제 |
| HTTP/2 | 멀티플렉싱으로 HOL Blocking 해결 |
| HTTP/3 | QUIC(UDP 기반), 연결 수립 1-RTT |
