# 메모리 (Memory)

> 데이터를 저장하고 CPU가 접근하는 공간. 계층 구조와 관리 방식을 이해하면 백엔드 성능 최적화의 기반이 된다.

## 1. 메모리 계층 구조

속도와 용량은 반비례한다. 빠를수록 비싸고 용량이 작다.

```mermaid
graph TB
    REG[레지스터<br/>~1ns 이하<br/>수 KB] --> L1[L1 캐시<br/>~1ns<br/>32~64KB]
    L1 --> L2[L2 캐시<br/>~5ns<br/>256KB~1MB]
    L2 --> L3[L3 캐시<br/>~20ns<br/>8~64MB]
    L3 --> RAM[RAM<br/>~100ns<br/>8~64GB]
    RAM --> SSD[SSD<br/>~0.1ms<br/>256GB~4TB]
    SSD --> HDD[HDD<br/>~10ms<br/>1~20TB]

    style REG fill:#ff6b6b,color:#fff
    style L1 fill:#ff8787
    style L2 fill:#ffa8a8
    style L3 fill:#ffc9c9
    style RAM fill:#fff3cd
    style SSD fill:#d4edda
    style HDD fill:#cce5ff
```

### 속도 차이 체감

| 비유 | 구간 | 속도 차이 |
|------|------|----------|
| 책상 위 메모지 → 방 안 책장 | 레지스터 → RAM | 100배 느림 |
| 방 안 책장 → 동네 도서관 | RAM → SSD | 1,000배 느림 |
| 방 안 책장 → 다른 도시 도서관 | RAM → HDD | 100,000배 느림 |

### 백엔드 연결 - Redis가 왜 빠른가?

```mermaid
graph LR
    subgraph Redis["Redis (In-Memory DB)"]
        R[데이터 요청] --> RAM2[RAM에서 읽음<br/>~100ns ⚡]
    end
    subgraph MySQL["MySQL (디스크 기반 DB)"]
        M[데이터 요청] --> SSD2[SSD에서 읽음<br/>~0.1ms]
    end

    style Redis fill:#d4edda
    style MySQL fill:#fff3cd
```

```mermaid
graph TB
    COMPARE[같은 데이터 조회 성능 비교]
    COMPARE --> REDIS_T["Redis: ~0.1ms"]
    COMPARE --> MYSQL_T["MySQL (메모리): ~1ms"]
    COMPARE --> MYSQL_D["MySQL (디스크): ~10ms"]

    style COMPARE fill:#0f3460,color:#fff
    style REDIS_T fill:#d4edda
    style MYSQL_T fill:#fff3cd
    style MYSQL_D fill:#ffe4e1
```

RAM vs SSD = 속도 차이 1,000배 이상. Redis가 빠른 이유가 바로 이것이다.

### 실무 활용 패턴

```mermaid
sequenceDiagram
    participant C as 클라이언트
    participant S as Spring Boot
    participant R as Redis (캐시)
    participant DB as MySQL

    C->>S: 상품 조회 요청
    S->>R: Redis에 캐시 있나?
    alt 캐시 히트 ⚡
        R->>S: 캐시된 데이터 반환
        S->>C: 즉시 응답 (~1ms)
    else 캐시 미스
        S->>DB: DB 조회
        DB->>S: 데이터 반환
        S->>R: Redis에 캐시 저장
        S->>C: 응답 (~10ms)
    end
```

## 2. 캐시 메모리

CPU가 RAM에 직접 접근하면 느려서, 자주 쓰는 데이터를 캐시에 올려두는 것이다.

### 캐시 히트 vs 캐시 미스

```mermaid
flowchart TD
    REQ[CPU: 데이터 필요] --> CHECK{L1 캐시에<br/>있는가?}
    CHECK -->|있음| HIT[캐시 히트 ⚡<br/>~1ns에 반환]
    CHECK -->|없음| L2CHECK{L2 캐시에<br/>있는가?}
    L2CHECK -->|있음| L2HIT[L2에서 반환<br/>~5ns]
    L2CHECK -->|없음| L3CHECK{L3 캐시에<br/>있는가?}
    L3CHECK -->|있음| L3HIT[L3에서 반환<br/>~20ns]
    L3CHECK -->|없음| MISS[캐시 미스 ❌<br/>RAM에서 가져옴<br/>~100ns]
    MISS --> SAVE[캐시에 저장]
    SAVE --> RET[다음번엔<br/>캐시에서 반환]

    style HIT fill:#d4edda
    style L2HIT fill:#e8f4f8
    style L3HIT fill:#fff3cd
    style MISS fill:#ffe4e1
```

### 캐시 지역성 (Locality)
CPU가 다음에 무엇을 접근할지 예측하는 원리.

```mermaid
graph TB
    LOC[캐시 지역성]
    LOC --> TEMP[시간적 지역성<br/>Temporal Locality]
    LOC --> SPAT[공간적 지역성<br/>Spatial Locality]
    TEMP --> TEMP_EX["최근 접근한 데이터를<br/>곧 또 접근할 가능성 높음<br/>예: 루프 안에서 같은 변수 반복 사용"]
    SPAT --> SPAT_EX["접근한 데이터 근처를<br/>곧 접근할 가능성 높음<br/>예: 배열을 순서대로 접근"]

    style LOC fill:#0f3460,color:#fff
    style TEMP fill:#d4edda
    style SPAT fill:#e8f4f8
```

### 실제 코드 성능 차이

```java
int[][] arr = new int[1000][1000];

// ✅ 행 우선 순회 - 공간적 지역성 활용 (빠름)
for (int i = 0; i < 1000; i++)
    for (int j = 0; j < 1000; j++)
        sum += arr[i][j]; // 메모리 연속 접근 → 캐시 히트율 높음

// ❌ 열 우선 순회 - 캐시 미스 연발 (느림)
for (int i = 0; i < 1000; i++)
    for (int j = 0; j < 1000; j++)
        sum += arr[j][i]; // 메모리 불연속 접근 → 캐시 미스 연발
```

### 왜 행 우선이 빠른가?

자바 2차원 배열은 메모리에 **행 단위로 연속** 저장된다.

```mermaid
graph LR
    subgraph 메모리배치["실제 메모리 배치"]
        M1["[0][0]"] --> M2["[0][1]"] --> M3["[0][2]"] --> M4["[0][3]"]
        M4 --> M5["[1][0]"] --> M6["[1][1]"] --> M7["[1][2]"] --> M8["[1][3]"]
    end

    style 메모리배치 fill:#f0f0f0
```

```mermaid
graph LR
    subgraph 행우선["행 우선 순회 (빠름 ✅)"]
        A1["[0][0]"] --> A2["[0][1]"] --> A3["[0][2]"] --> A4["[0][3]"]
    end
    subgraph 열우선["열 우선 순회 (느림 ❌)"]
        B1["[0][0]"] -.-> B2["[1][0]"] -.-> B3["[2][0]"] -.-> B4["[3][0]"]
    end

    style 행우선 fill:#d4edda
    style 열우선 fill:#ffe4e1
```

행 우선은 메모리를 순서대로 읽어서 캐시 히트율이 높고, 열 우선은 메모리를 건너뛰며 읽어서 캐시 미스가 연발된다.

## 3. 가상 메모리

RAM이 부족할 때 SSD/HDD 일부를 RAM처럼 사용하는 기술.

### 핵심 아이디어

```mermaid
graph LR
    subgraph 가상["프로세스가 보는 가상 주소"]
        V1[0x0000]
        V2[0x1000]
        V3[0x2000]
        V4[0x3000]
    end

    subgraph 물리["실제 물리 메모리 (RAM)"]
        P1[프레임 3]
        P2[프레임 7]
        P3[프레임 1]
    end

    subgraph 디스크["SSD (스왑 영역)"]
        D1[스왑 공간]
    end

    V1 --> P1
    V2 --> P2
    V3 --> D1
    V4 --> P3

    style 가상 fill:#e8f4f8
    style 물리 fill:#d4edda
    style 디스크 fill:#fff3cd
```

각 프로세스는 자신만의 가상 주소 공간을 가지며, OS가 가상 주소를 실제 물리 주소로 변환한다. RAM에 다 못 올리는 페이지는 SSD(스왑)에 저장된다.

### 페이지 테이블

```mermaid
graph LR
    VA[가상 주소<br/>0x2000] --> PT[페이지 테이블]
    PT --> CHECK{RAM에<br/>있는가?}
    CHECK -->|있음| PA[물리 주소 변환<br/>바로 접근 ⚡]
    CHECK -->|없음| PF[페이지 폴트 발생 ❌]

    style VA fill:#e8f4f8
    style PT fill:#fff3cd
    style PA fill:#d4edda
    style PF fill:#ffe4e1
```

### 페이지 폴트 동작 과정

```mermaid
sequenceDiagram
    participant P as 프로세스
    participant MMU as MMU<br/>(메모리 관리 장치)
    participant RAM as RAM
    participant SSD as SSD (스왑)

    P->>MMU: 가상 주소 0x2000 접근
    MMU->>MMU: 페이지 테이블 확인
    Note over MMU: RAM에 없음!
    MMU->>MMU: ⚠️ 페이지 폴트 발생
    MMU->>RAM: 안 쓰는 페이지 선택
    RAM->>SSD: 선택된 페이지 → SSD로 내보냄 (Swap Out)
    SSD->>RAM: 필요한 페이지 → RAM으로 올림 (Swap In)
    MMU->>MMU: 페이지 테이블 갱신
    RAM->>P: 데이터 반환 (느림)
```

### 페이지 교체 알고리즘

RAM이 꽉 찼을 때 어떤 페이지를 내보낼지 결정하는 알고리즘.

```mermaid
graph TB
    ALG[페이지 교체 알고리즘]
    ALG --> FIFO["FIFO<br/>먼저 들어온 것 먼저 교체<br/>단순하지만 비효율"]
    ALG --> LRU["LRU (Least Recently Used)<br/>가장 오래 안 쓴 것 교체<br/>⭐ 가장 많이 사용"]
    ALG --> OPT["Optimal<br/>앞으로 가장 안 쓸 것 교체<br/>이론적 최적 (구현 불가)"]
    ALG --> LFU["LFU (Least Frequently Used)<br/>가장 적게 쓴 것 교체"]

    style ALG fill:#0f3460,color:#fff
    style LRU fill:#d4edda
    style FIFO fill:#e8f4f8
    style OPT fill:#f0f0f0
    style LFU fill:#e8f4f8
```

> **백엔드 연결**: Redis의 캐시 만료 정책도 LRU를 사용한다. `maxmemory-policy allkeys-lru`

**서버 연결**: 서버 RAM이 부족하면 스왑 발생 → 응답 시간이 갑자기 느려지는 원인 → 실무에서 서버 메모리 모니터링이 중요한 이유

### 스왑 발생 시 성능 영향

```mermaid
graph LR
    subgraph 정상["RAM 충분 (정상)"]
        N1[요청] --> N2[RAM에서 읽기<br/>~100ns] --> N3[응답 1ms ✅]
    end
    subgraph 스왑["RAM 부족 (스왑 발생)"]
        S1[요청] --> S2[SSD에서 읽기<br/>~0.1ms] --> S3[응답 100ms ❌]
    end

    style 정상 fill:#d4edda
    style 스왑 fill:#ffe4e1
```

## 4. 자바 메모리 구조

자바는 JVM 위에서 돌아가기 때문에 JVM이 메모리를 관리한다.

```mermaid
graph TB
    JVM[JVM 메모리]
    JVM --> MA["Method Area<br/>클래스 정보, static 변수<br/>모든 스레드 공유"]
    JVM --> HEAP["Heap<br/>new로 생성한 객체<br/>GC가 관리<br/>모든 스레드 공유"]
    JVM --> S1["Stack - Thread 1<br/>지역 변수, 매개변수<br/>스레드마다 독립"]
    JVM --> S2["Stack - Thread 2<br/>지역 변수, 매개변수<br/>스레드마다 독립"]
    JVM --> PC["PC Register<br/>현재 실행 위치<br/>스레드마다 독립"]

    style JVM fill:#0f3460,color:#fff
    style HEAP fill:#fff3cd
    style MA fill:#e8f4f8
    style S1 fill:#d4edda
    style S2 fill:#d4edda
    style PC fill:#f0f0f0
```

### 공유 vs 독립 영역

```mermaid
graph TB
    subgraph 공유["모든 스레드가 공유하는 영역"]
        MA2[Method Area]
        HEAP2[Heap]
    end
    subgraph 독립["스레드마다 독립인 영역"]
        ST1[Stack]
        PC1[PC Register]
        NS1[Native Method Stack]
    end

    style 공유 fill:#fff3cd
    style 독립 fill:#d4edda
```

> **동시성 문제**: 공유 영역(Heap)에 여러 스레드가 동시 접근하면 Race Condition 발생 가능

### 스택 vs 힙 실제 동작

```java
public void createMember() {
    int id = 1;                    // Stack에 저장
    String name = "김사자";         // Stack에 참조값, Heap에 실제 문자열
    Member member = new Member();  // Stack에 참조값, Heap에 실제 객체
}
// 메서드 종료 → Stack 프레임 자동 해제
// Heap의 member는 GC가 나중에 정리
```

```mermaid
graph TB
    subgraph Stack["Stack (createMember 프레임)"]
        V1["id = 1 (값 직접 저장)"]
        V2["name = 0x200 (참조값)"]
        V3["member = 0x100 (참조값)"]
    end
    subgraph Heap["Heap"]
        O1["0x100: Member 객체<br/>name: 김사자<br/>major: 컴공"]
        O2["0x200: String 객체<br/>value: 김사자"]
    end
    V3 -->|참조| O1
    V2 -->|참조| O2

    style Stack fill:#d4edda
    style Heap fill:#fff3cd
```

### 참조값 이해

```mermaid
graph LR
    subgraph Stack
        A["member | 0x100"]
    end
    subgraph Heap
        B["0x100: Member<br/>name: 김사자<br/>major: 컴공"]
    end
    A -->|참조| B

    style Stack fill:#d4edda
    style Heap fill:#fff3cd
```

`member` 변수 자체에는 `0x100` 같은 주소값이 들어있고, 그 주소가 가리키는 Heap 공간에 실제 객체 데이터가 있다.

### 두 변수가 같은 객체를 참조하면?

```java
Member a = new Member("김사자");
Member b = a; // 같은 주소를 복사
b.setName("박사자");
System.out.println(a.getName()); // "박사자" (같은 객체)
```

```mermaid
graph LR
    subgraph Stack
        VA["a | 0x100"]
        VB["b | 0x100"]
    end
    subgraph Heap
        OBJ["0x100: Member<br/>name: 박사자"]
    end
    VA -->|참조| OBJ
    VB -->|참조| OBJ

    style Stack fill:#d4edda
    style Heap fill:#fff3cd
```

### 스택 프레임 동작 과정

```mermaid
sequenceDiagram
    participant JVM as JVM
    participant Stack as Stack
    participant Heap as Heap

    JVM->>Stack: main() 프레임 push
    JVM->>Stack: createMember() 프레임 push
    Stack->>Heap: new Member() → Heap에 객체 생성
    Note over Stack: id, name, member 저장
    JVM->>Stack: createMember() 프레임 pop (자동 해제)
    Note over Heap: Member 객체는 아직 남아있음
    Note over Heap: → GC가 나중에 정리
    JVM->>Stack: main() 프레임 pop
```

### 스택 오버플로우

```java
public void infinite() {
    infinite(); // 스택 프레임이 계속 쌓임
}
// → StackOverflowError 발생
```

```mermaid
graph TB
    subgraph Stack["Stack 영역 (크기 제한)"]
        F1["infinite() 프레임 #1"]
        F2["infinite() 프레임 #2"]
        F3["infinite() 프레임 #3"]
        F4["infinite() 프레임 #4"]
        F5["... 계속 쌓임"]
        F6["💥 StackOverflowError"]
    end
    F1 --> F2 --> F3 --> F4 --> F5 --> F6

    style Stack fill:#ffe4e1
    style F6 fill:#ff6b6b,color:#fff
```

## 5. 가비지 컬렉션 (GC) ⭐

자바는 개발자가 메모리를 직접 해제 안 해도 된다. JVM의 GC가 자동으로 처리.

### C/C++ vs Java 메모리 관리 비교

```mermaid
graph LR
    subgraph C["C/C++ (수동 관리)"]
        C1["malloc() / new"] --> C2["개발자가 직접<br/>free() / delete"]
        C2 --> C3["실수하면<br/>메모리 누수 💀"]
    end
    subgraph JAVA["Java (자동 관리)"]
        J1["new 객체 생성"] --> J2["GC가 자동으로<br/>안 쓰는 객체 정리"]
        J2 --> J3["개발자는<br/>비즈니스 로직에 집중 ✅"]
    end

    style C fill:#ffe4e1
    style JAVA fill:#d4edda
```

### GC 대상 판단 - Reachability

```mermaid
flowchart TD
    ROOT["GC Root<br/>(Stack 변수, static 변수 등)"]
    ROOT --> OBJ1["객체 A<br/>참조됨 ✅"]
    OBJ1 --> OBJ2["객체 B<br/>참조됨 ✅"]
    ROOT --> OBJ3["객체 C<br/>참조됨 ✅"]

    OBJ4["객체 D<br/>참조 없음 ❌"] --> FREE1["GC 수거 대상"]
    OBJ5["객체 E<br/>참조 없음 ❌"] --> FREE2["GC 수거 대상"]

    style ROOT fill:#0f3460,color:#fff
    style OBJ1 fill:#d4edda
    style OBJ2 fill:#d4edda
    style OBJ3 fill:#d4edda
    style OBJ4 fill:#ffe4e1
    style OBJ5 fill:#ffe4e1
    style FREE1 fill:#cce5ff
    style FREE2 fill:#cce5ff
```

GC Root에서 참조 체인을 따라갈 수 있는 객체는 살아남고, 참조 체인이 끊긴 객체는 수거 대상이 된다.

### Heap 세대별 구조

```mermaid
graph TB
    HEAP[Heap]
    HEAP --> YOUNG[Young Generation<br/>새로 생성된 객체<br/>Minor GC 대상]
    HEAP --> OLD[Old Generation<br/>오래 살아남은 객체<br/>Major GC 대상]

    YOUNG --> EDEN["Eden<br/>객체 최초 생성 장소"]
    YOUNG --> S0["Survivor 0<br/>Minor GC 후 생존 객체"]
    YOUNG --> S1["Survivor 1<br/>S0에서 다시 생존한 객체"]

    style HEAP fill:#0f3460,color:#fff
    style YOUNG fill:#d4edda
    style OLD fill:#fff3cd
    style EDEN fill:#e8f4f8
    style S0 fill:#e8f4f8
    style S1 fill:#e8f4f8
```

### 왜 세대를 나누는가? (Generational Hypothesis)

```mermaid
graph LR
    subgraph 관찰["관찰된 사실"]
        F1["대부분의 객체는<br/>금방 죽는다 (약 95%)"]
        F2["오래 살아남은 객체는<br/>계속 살아남는다"]
    end
    subgraph 전략["GC 전략"]
        S1A["Young 영역은 자주,<br/>빠르게 청소 (Minor GC)"]
        S2A["Old 영역은 가끔,<br/>신중하게 청소 (Major GC)"]
    end
    F1 --> S1A
    F2 --> S2A

    style 관찰 fill:#e8f4f8
    style 전략 fill:#d4edda
```

### GC 동작 과정

```mermaid
sequenceDiagram
    participant App as 애플리케이션
    participant Eden as Eden
    participant S0 as Survivor 0
    participant S1 as Survivor 1
    participant Old as Old Generation

    App->>Eden: new 객체 생성
    App->>Eden: new 객체 생성
    App->>Eden: new 객체 생성
    Note over Eden: Eden 꽉 참!

    Note over Eden,S0: 🔄 Minor GC 발생
    Eden->>S0: 살아있는 것만 S0로 이동
    Note over Eden: 죽은 객체 일괄 제거 (빠름 ⚡)

    Note over S0,S1: 🔄 다음 Minor GC
    S0->>S1: 살아있으면 S1으로 이동<br/>(age +1)

    Note over S1,Old: age가 임계값 초과
    S1->>Old: Old Generation으로 승격 (Promotion)

    Note over Old: Old도 꽉 참!
    Note over App,Old: 🔴 Major GC (Full GC)<br/>Stop-The-World ⚠️<br/>모든 스레드 멈춤
```

### Stop-The-World ⚠️

GC 실행 중 모든 애플리케이션 스레드가 정지된다.

```mermaid
gantt
    title Stop-The-World 영향
    dateFormat X
    axisFormat %s

    section 애플리케이션
    요청 처리 :active, a1, 0, 3
    STW 정지 :crit, a2, 3, 5
    요청 처리 :active, a3, 5, 10

    section GC
    대기 :done, g1, 0, 3
    GC 실행 :crit, g2, 3, 5
    대기 :done, g3, 5, 10
```

- API 응답이 수백ms ~ 수초 지연될 수 있음
- 실무에서 GC 튜닝이 중요한 이유
- 객체를 불필요하게 많이 생성하면 GC 부하 증가

### GC 알고리즘 비교

```mermaid
graph LR
    subgraph Serial["Serial GC"]
        SG["싱글 스레드<br/>작은 서버용<br/>STW 김"]
    end
    subgraph Parallel["Parallel GC"]
        PG["멀티 스레드<br/>처리량 중시<br/>STW 중간"]
    end
    subgraph G1["G1 GC ⭐"]
        G1G["Java 9+ 기본<br/>Region 기반<br/>STW 최소화"]
    end
    subgraph ZGC["ZGC"]
        ZG["Java 15+<br/>대용량 Heap<br/>STW 1ms 미만"]
    end

    style Serial fill:#ffe4e1
    style Parallel fill:#fff3cd
    style G1 fill:#d4edda
    style ZGC fill:#e8f4f8
```

| 알고리즘 | Java 버전 | STW 시간 | 적합한 환경 |
|---------|----------|---------|-----------|
| Serial GC | 전체 | 길다 | 클라이언트, 작은 Heap |
| Parallel GC | 전체 | 중간 | 배치 처리, 처리량 중시 |
| G1 GC ⭐ | 9+ 기본 | 짧다 | 일반 서버 (가장 많이 사용) |
| ZGC | 15+ | 극히 짧다 (< 1ms) | 대용량 Heap, 저지연 서비스 |

## 6. 메모리 누수 (Memory Leak)

GC가 있어도 메모리 누수가 발생할 수 있다. 참조가 끊기지 않으면 GC가 지우지 못한다.

```mermaid
graph LR
    subgraph 정상["정상 - 참조 끊김"]
        A1["member = null"] -.->|참조 없음| O1[객체]
        O1 -->|GC 수거| FREE1["메모리 해제 ✅"]
    end

    subgraph 누수["누수 - 참조 안 끊김"]
        A2["static Map"] -->|참조 유지| O2[객체]
        O2 -->|GC 못 지움| LEAK["메모리 계속 증가 ❌"]
    end

    style 정상 fill:#d4edda
    style 누수 fill:#ffe4e1
```

### 메모리 누수 시 서버 상태 변화

```mermaid
graph LR
    T1["서버 시작<br/>Heap 사용 30%"] --> T2["운영 1일<br/>Heap 사용 50%"]
    T2 --> T3["운영 3일<br/>Heap 사용 70%"]
    T3 --> T4["운영 7일<br/>Heap 사용 90%"]
    T4 --> T5["💥 OutOfMemoryError<br/>서버 다운"]

    style T1 fill:#d4edda
    style T2 fill:#fff3cd
    style T3 fill:#ffa8a8
    style T4 fill:#ff8787
    style T5 fill:#ff6b6b,color:#fff
```

### 백엔드에서 흔한 메모리 누수 원인

```mermaid
graph TB
    LEAK[메모리 누수 원인 Top 4]
    LEAK --> L1["1. static 컬렉션<br/>추가만 하고 삭제 안 함"]
    LEAK --> L2["2. 이벤트 리스너<br/>등록 후 해제 안 함"]
    LEAK --> L3["3. DB 커넥션 / 스트림<br/>close() 호출 안 함"]
    LEAK --> L4["4. 무제한 캐시<br/>TTL 없이 계속 쌓음"]

    style LEAK fill:#0f3460,color:#fff
    style L1 fill:#ffe4e1
    style L2 fill:#ffe4e1
    style L3 fill:#ffe4e1
    style L4 fill:#ffe4e1
```

### 올바른 자원 관리 예시

```java
// ❌ 나쁜 예 - close 안 함
public void readFile() {
    InputStream is = new FileInputStream("data.txt");
    // 예외 발생 시 close 안 됨 → 메모리 누수
}

// ✅ 좋은 예 - try-with-resources
public void readFile() {
    try (InputStream is = new FileInputStream("data.txt")) {
        // 자동으로 close() 호출
    }
}
```

## 핵심 요약

| 개념 | 핵심 내용 | 백엔드 연결 |
|------|----------|------------|
| 메모리 계층 | 빠를수록 비싸고 작다 | Redis가 빠른 이유 (RAM vs SSD) |
| 캐시 지역성 | 시간적/공간적 지역성 | 코드 최적화, 배열 순회 방향 |
| 가상 메모리 | SSD를 RAM처럼 사용 | 서버 스왑 발생 시 성능 저하 원인 |
| JVM 메모리 | Heap(공유) + Stack(독립) | 객체 생명주기, 동시성 문제의 근원 |
| GC | 안 쓰는 객체 자동 정리 | STW로 인한 응답 지연, GC 튜닝 |
| 메모리 누수 | 참조 안 끊기면 GC 무력화 | 서버가 점점 느려지다 OOM 발생 |
