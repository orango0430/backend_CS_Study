# 스레드 (Thread)

> 스레드는 프로세스 안의 실행 흐름. 백엔드 서버가 요청을 동시에 처리하는 핵심 단위이다.

## 1. 스레드란?

**정의**: 프로세스 안에서 실행되는 경량 실행 단위. 프로세스의 자원을 공유하면서 독립적으로 실행된다.

```mermaid
graph TB
    subgraph 프로세스["프로세스 (하나의 JVM)"]
        SHARED["공유 자원<br/>Code / Data / Heap"]
        T1["스레드 1<br/>Stack | Register | PC"]
        T2["스레드 2<br/>Stack | Register | PC"]
        T3["스레드 3<br/>Stack | Register | PC"]
        SHARED --- T1
        SHARED --- T2
        SHARED --- T3
    end

    style 프로세스 fill:#e8f4f8
    style SHARED fill:#fff3cd
    style T1 fill:#d4edda
    style T2 fill:#d4edda
    style T3 fill:#d4edda
```

프로세스 1개 = 스레드 1개 이상. 프로세스는 스레드의 컨테이너이다.

## 2. 스레드 메모리 구조

### 공유 vs 독립 영역

```mermaid
graph TB
    PROC["프로세스"]
    PROC --> CODE["Code 영역<br/>프로그램 코드<br/>🔗 스레드끼리 공유"]
    PROC --> DATA["Data 영역<br/>전역/static 변수<br/>🔗 스레드끼리 공유"]
    PROC --> HEAP["Heap 영역<br/>new로 생성한 객체<br/>🔗 스레드끼리 공유"]
    PROC --> T1S["Thread 1 전용<br/>Stack | Register | PC"]
    PROC --> T2S["Thread 2 전용<br/>Stack | Register | PC"]
    PROC --> T3S["Thread 3 전용<br/>Stack | Register | PC"]

    style PROC fill:#0f3460,color:#fff
    style CODE fill:#fff3cd
    style DATA fill:#fff3cd
    style HEAP fill:#fff3cd
    style T1S fill:#d4edda
    style T2S fill:#d4edda
    style T3S fill:#d4edda
```

**핵심**: Code/Data/Heap은 공유, Stack만 스레드마다 별도.

### 왜 Stack만 분리하는가?

```mermaid
graph TB
    subgraph 이유["Stack을 분리하는 이유"]
        R1["각 스레드는 독립적인<br/>함수 호출 흐름을 가짐"]
        R2["스레드 1이 methodA() 실행 중<br/>스레드 2는 methodB() 실행 중"]
        R3["각자의 지역 변수,<br/>매개변수, 리턴 주소가<br/>서로 다름"]
    end

    style 이유 fill:#e8f4f8
```

```mermaid
graph LR
    subgraph Thread1["스레드 1의 Stack"]
        T1_F1["main() 프레임"]
        T1_F2["handleRequest() 프레임"]
        T1_F3["findUser() 프레임"]
    end
    subgraph Thread2["스레드 2의 Stack"]
        T2_F1["main() 프레임"]
        T2_F2["handleRequest() 프레임"]
        T2_F3["createOrder() 프레임"]
    end

    style Thread1 fill:#d4edda
    style Thread2 fill:#e8f4f8
```

## 3. 프로세스 vs 스레드 비교

```mermaid
graph LR
    subgraph 프로세스방식["멀티 프로세스"]
        P1["프로세스 1<br/>Code|Data|Heap|Stack"]
        P2["프로세스 2<br/>Code|Data|Heap|Stack"]
        P1 -.->|"IPC 필요<br/>(느림)"| P2
    end

    subgraph 스레드방식["멀티 스레드"]
        subgraph 공유영역["공유: Code|Data|Heap"]
            T1["Thread 1<br/>Stack"]
            T2["Thread 2<br/>Stack"]
        end
        T1 -->|"메모리 공유<br/>(빠름)"| T2
    end

    style 프로세스방식 fill:#ffe4e1
    style 스레드방식 fill:#d4edda
    style 공유영역 fill:#fff3cd
```

| 구분 | 프로세스 | 스레드 |
|------|---------|--------|
| 메모리 | 완전히 독립 | 공유 (Stack만 별도) |
| 생성 비용 | 크다 (메모리 전체 할당) | 작다 (Stack만 할당) |
| 통신 | IPC 필요 (복잡, 느림) | 메모리 공유 (간단, 빠름) |
| 컨텍스트 스위칭 | 느림 (메모리 재매핑 필요) | 빠름 (공유 영역 유지) |
| 안정성 | 한 프로세스 죽어도 독립 | 한 스레드 죽으면 전체 죽음 ⚠️ |
| 데이터 공유 | 어려움 | 쉬움 (단, 동시성 문제 주의) |

### 한 스레드가 죽으면?

```mermaid
graph TB
    subgraph 프로세스_격리["프로세스 격리"]
        PA["프로세스 A 💀"] -.->|영향 없음| PB["프로세스 B ✅"]
    end

    subgraph 스레드_공유["스레드 공유"]
        TA["스레드 A 💀<br/>예외 미처리"] -->|"프로세스 전체 영향"| TB["스레드 B 💀"]
        TA -->|"프로세스 전체 영향"| TC["스레드 C 💀"]
    end

    style 프로세스_격리 fill:#d4edda
    style 스레드_공유 fill:#ffe4e1
```

> **실무 주의**: 스프링에서 한 스레드의 예외가 처리되지 않으면 JVM 전체가 불안정해질 수 있다. 그래서 `@ExceptionHandler`, `@ControllerAdvice`가 중요한 것이다.

## 4. 동시성 문제 ⚠️

스레드는 메모리를 공유해서 편하지만, 여러 스레드가 같은 데이터를 동시에 건드리면 문제가 생긴다.

### Race Condition (경쟁 조건)

```mermaid
sequenceDiagram
    participant T1 as 스레드 1
    participant MEM as 공유 변수<br/>count = 0
    participant T2 as 스레드 2

    T1->>MEM: 읽기: count = 0
    T2->>MEM: 읽기: count = 0
    Note over T1: 0 + 1 = 1 계산
    Note over T2: 0 + 1 = 1 계산
    T1->>MEM: 쓰기: count = 1
    T2->>MEM: 쓰기: count = 1
    Note over MEM: ❌ 예상: 2, 실제: 1
```

```java
int count = 0;

// Thread 1                    // Thread 2
count++;                       count++;
// 읽기(0) → 증가 → 쓰기(1)   // 읽기(0) → 증가 → 쓰기(1)

// 예상: count = 2
// 실제: count = 1 (두 스레드가 동시에 0을 읽었기 때문)
```

### 실무 예시 - 재고 오버셀링

```mermaid
sequenceDiagram
    participant T1 as 스레드 A (사용자 1)
    participant DB as DB (재고 = 1)
    participant T2 as 스레드 B (사용자 2)

    T1->>DB: SELECT 재고 조회
    DB->>T1: 재고 = 1 (있음!)
    T2->>DB: SELECT 재고 조회
    DB->>T2: 재고 = 1 (있음!)
    T1->>DB: UPDATE 재고 = 0 (차감)
    T2->>DB: UPDATE 재고 = -1 (차감)
    Note over DB: ❌ 재고 -1<br/>오버셀링 발생!
```

### Deadlock (교착 상태)

두 스레드가 서로가 가진 자원을 기다리며 **영원히 멈추는** 상황.

```mermaid
graph LR
    TA["스레드 A<br/>🔒 자원 1 보유"] -->|"자원 2 필요<br/>(대기 ⏳)"| RES2["🔒 자원 2"]
    TB["스레드 B<br/>🔒 자원 2 보유"] -->|"자원 1 필요<br/>(대기 ⏳)"| RES1["🔒 자원 1"]
    RES1 --> TA
    RES2 --> TB

    style TA fill:#ffe4e1
    style TB fill:#ffe4e1
    style RES1 fill:#fff3cd
    style RES2 fill:#fff3cd
```

### 실무 예시 - 계좌 이체 데드락

```mermaid
sequenceDiagram
    participant TA as 스레드 A<br/>(A→B 이체)
    participant ACC1 as 계좌 1 (🔒)
    participant ACC2 as 계좌 2 (🔒)
    participant TB as 스레드 B<br/>(B→A 이체)

    TA->>ACC1: 계좌 1 락 획득 🔒
    TB->>ACC2: 계좌 2 락 획득 🔒
    TA->>ACC2: 계좌 2 락 요청... ⏳
    TB->>ACC1: 계좌 1 락 요청... ⏳
    Note over TA,TB: 💀 서로 영원히 대기<br/>= DEADLOCK
```

### Deadlock 발생 조건 (4가지 모두 만족 시)

```mermaid
graph TB
    DL["Deadlock 발생 조건<br/>(4가지 모두 만족 시)"]
    DL --> C1["1. 상호 배제<br/>Mutual Exclusion<br/>자원을 한 스레드만 점유"]
    DL --> C2["2. 점유와 대기<br/>Hold and Wait<br/>자원 잡은 채로 다른 자원 대기"]
    DL --> C3["3. 비선점<br/>No Preemption<br/>강제로 뺏을 수 없음"]
    DL --> C4["4. 순환 대기<br/>Circular Wait<br/>대기 관계가 원형"]

    style DL fill:#ff6b6b,color:#fff
    style C1 fill:#ffe4e1
    style C2 fill:#ffe4e1
    style C3 fill:#ffe4e1
    style C4 fill:#ffe4e1
```

하나라도 깨면 데드락이 발생하지 않는다. 실무에서는 주로 **순환 대기를 깨는 방법**(자원 순서 고정)을 사용한다.

### Deadlock 해결 - 자원 순서 고정

```mermaid
graph LR
    subgraph 데드락["❌ 데드락 발생"]
        A1["스레드 A: 계좌1 → 계좌2"]
        B1["스레드 B: 계좌2 → 계좌1"]
    end
    subgraph 해결["✅ 순서 고정으로 해결"]
        A2["스레드 A: 계좌1 → 계좌2"]
        B2["스레드 B: 계좌1 → 계좌2"]
    end

    style 데드락 fill:#ffe4e1
    style 해결 fill:#d4edda
```

## 5. 동기화 방법

### Mutex (Mutual Exclusion)

한 번에 하나의 스레드만 접근 가능하게 하는 잠금 장치.

```mermaid
sequenceDiagram
    participant T1 as 스레드 1
    participant LOCK as synchronized (lock)
    participant T2 as 스레드 2

    T1->>LOCK: 락 획득 🔒
    Note over T1,LOCK: count++ 실행 (안전)
    T2->>LOCK: 락 요청... ⏳ 대기
    T1->>LOCK: 락 해제 🔓
    LOCK->>T2: 락 획득 🔒
    Note over T2,LOCK: count++ 실행 (안전)
    T2->>LOCK: 락 해제 🔓
```

```java
// 자바에서 Mutex → synchronized 키워드
synchronized (lock) {
    count++; // 한 스레드씩 순서대로 실행
}
```

### Semaphore

N개의 스레드까지 동시 접근을 허용하는 방식.

```mermaid
graph TB
    SEM["Semaphore (permits = 3)"]
    SEM --> T1["스레드 1 ✅ 접근 중"]
    SEM --> T2["스레드 2 ✅ 접근 중"]
    SEM --> T3["스레드 3 ✅ 접근 중"]
    SEM -.-> T4["스레드 4 ⏳ 대기"]
    SEM -.-> T5["스레드 5 ⏳ 대기"]

    style SEM fill:#0f3460,color:#fff
    style T1 fill:#d4edda
    style T2 fill:#d4edda
    style T3 fill:#d4edda
    style T4 fill:#ffe4e1
    style T5 fill:#ffe4e1
```

```java
Semaphore sem = new Semaphore(3); // 동시에 3개까지 허용

sem.acquire(); // 허가 획득 (permits--)
try {
    // 작업 수행
} finally {
    sem.release(); // 허가 반환 (permits++)
}
```

### Mutex vs Semaphore

```mermaid
graph LR
    subgraph Mutex방식["Mutex (화장실 1칸)"]
        M_LOCK["🔒 1명만 입장"]
        M_T1["사용 중 🚶"]
        M_T2["대기 ⏳"]
        M_T3["대기 ⏳"]
    end

    subgraph Semaphore방식["Semaphore (화장실 3칸)"]
        S_LOCK["🔒 3명까지 입장"]
        S_T1["사용 중 🚶"]
        S_T2["사용 중 🚶"]
        S_T3["사용 중 🚶"]
        S_T4["대기 ⏳"]
    end

    style Mutex방식 fill:#e8f4f8
    style Semaphore방식 fill:#d4edda
```

| 구분 | Mutex | Semaphore |
|------|-------|-----------|
| 동시 접근 | 1개만 | N개까지 |
| 자바 구현 | `synchronized` | `Semaphore` 클래스 |
| 비유 | 화장실 1칸 | 화장실 N칸 |
| 사용 사례 | 공유 자원 보호 | 커넥션 풀, 스레드 풀 제한 |

### Monitor

자바 객체마다 내부적으로 가진 락. `synchronized`가 이 방식을 사용한다.

```mermaid
graph TB
    subgraph Monitor["Monitor (자바 객체 내장 락)"]
        ENTRY["Entry Set<br/>락 대기 큐"]
        OWNER["Owner<br/>현재 락 소유 스레드"]
        WAIT["Wait Set<br/>조건 대기 큐<br/>(wait() 호출 시)"]
    end

    ENTRY -->|"락 획득"| OWNER
    OWNER -->|"wait() 호출"| WAIT
    WAIT -->|"notify() 받으면"| ENTRY
    OWNER -->|"작업 완료,<br/>락 해제"| ENTRY

    style Monitor fill:#e8f4f8
    style ENTRY fill:#fff3cd
    style OWNER fill:#d4edda
    style WAIT fill:#ffe4e1
```

```java
synchronized (monitor) {
    while (!조건) {
        monitor.wait();    // Wait Set으로 이동
    }
    // 작업 수행
    monitor.notify();     // Wait Set의 스레드 하나를 깨움
}
```

## 6. 스레드 풀 (Thread Pool)

스레드를 매번 생성/삭제하면 비용이 크다. 미리 만들어두고 재사용하는 것이 스레드 풀이다.

### 왜 스레드 풀이 필요한가?

```mermaid
graph LR
    subgraph 풀없음["스레드 풀 없이 (비효율)"]
        R1["요청 1"] --> C1["스레드 생성 → 작업 → 스레드 삭제"]
        R2["요청 2"] --> C2["스레드 생성 → 작업 → 스레드 삭제"]
        R3["요청 3"] --> C3["스레드 생성 → 작업 → 스레드 삭제"]
    end

    subgraph 풀있음["스레드 풀 사용 (효율)"]
        R4["요청 1"] --> T1P["스레드 A (재사용)"]
        R5["요청 2"] --> T2P["스레드 B (재사용)"]
        R6["요청 3"] --> T1P
    end

    style 풀없음 fill:#ffe4e1
    style 풀있음 fill:#d4edda
```

### 스레드 풀 동작 방식

```mermaid
sequenceDiagram
    participant REQ as 요청들
    participant Q as 작업 큐
    participant POOL as 스레드 풀<br/>(200개)
    participant T as 스레드

    REQ->>Q: 요청 1 도착
    REQ->>Q: 요청 2 도착
    REQ->>Q: 요청 3 도착
    Q->>POOL: 유휴 스레드 요청
    POOL->>T: 스레드 할당
    T->>T: 작업 수행
    T->>POOL: 작업 완료, 스레드 반납
    Note over POOL: 스레드 삭제 X<br/>다음 작업에 재사용 ♻️
    Q->>POOL: 다음 작업 요청
    POOL->>T: 같은 스레드 재할당
```

### 스레드 풀 크기와 성능

```mermaid
graph TB
    SIZE["스레드 풀 크기 설정"]
    SIZE --> SMALL["너무 작으면<br/>요청이 큐에 쌓여 대기<br/>응답 지연 ❌"]
    SIZE --> OPTIMAL["적절하면<br/>효율적 자원 활용<br/>안정적 응답 ✅"]
    SIZE --> LARGE["너무 크면<br/>컨텍스트 스위칭 비용 증가<br/>메모리 낭비 ❌"]

    style SIZE fill:#0f3460,color:#fff
    style SMALL fill:#ffe4e1
    style OPTIMAL fill:#d4edda
    style LARGE fill:#ffe4e1
```

### 자바의 스레드 풀

```java
// 자바 기본 스레드 풀
ExecutorService pool = Executors.newFixedThreadPool(10);

pool.submit(() -> {
    System.out.println("작업 실행: " + Thread.currentThread().getName());
});

pool.shutdown(); // 모든 작업 완료 후 종료
```

## 7. 자바의 동시성 제어 ⭐

### synchronized vs Lock vs Atomic

```mermaid
graph TB
    SYNC["자바 동시성 제어 방법"]
    SYNC --> S1["synchronized<br/>키워드 기반<br/>간단, 자동 해제"]
    SYNC --> S2["ReentrantLock<br/>명시적 lock/unlock<br/>tryLock, 타임아웃 지원"]
    SYNC --> S3["Atomic 클래스<br/>CAS 기반 (락 없이)<br/>AtomicInteger 등"]

    style SYNC fill:#0f3460,color:#fff
    style S1 fill:#e8f4f8
    style S2 fill:#fff3cd
    style S3 fill:#d4edda
```

```java
// 1. synchronized
synchronized (this) {
    count++;
}

// 2. ReentrantLock
Lock lock = new ReentrantLock();
lock.lock();
try {
    count++;
} finally {
    lock.unlock(); // 반드시 해제
}

// 3. AtomicInteger (락 없이 안전)
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // 원자적 증가
```

## 8. 백엔드 개발자 관점 연결

### 스프링 부트 요청 처리 = 스레드 풀 활용

```mermaid
sequenceDiagram
    participant Client as 클라이언트
    participant Tomcat as 톰캣 스레드 풀<br/>(기본 200개)
    participant Thread as 할당된 스레드
    participant Controller as Controller
    participant Service as Service
    participant DB as DB

    Client->>Tomcat: HTTP 요청
    Tomcat->>Thread: 유휴 스레드 할당
    Thread->>Controller: @GetMapping 실행
    Controller->>Service: @Service 메서드 호출
    Service->>DB: JPA findById() (I/O 대기)
    Note over Thread: ⏳ DB 응답 대기<br/>스레드 블로킹 상태
    DB->>Service: 결과 반환
    Service->>Controller: 응답 DTO 반환
    Controller->>Thread: ResponseEntity 반환
    Thread->>Tomcat: 스레드 반납 ♻️
    Tomcat->>Client: HTTP 응답 전송
```

### 톰캣 스레드 풀 설정

```yaml
# application.yml
server:
  tomcat:
    threads:
      max: 200        # 최대 스레드 수
      min-spare: 10   # 최소 유지 스레드 수
    max-connections: 8192
    accept-count: 100  # 대기 큐 크기
```

```mermaid
graph TB
    REQ["요청 도착"]
    REQ --> CHECK1{스레드<br/>풀에 여유?}
    CHECK1 -->|있음| ASSIGN["스레드 할당<br/>즉시 처리 ⚡"]
    CHECK1 -->|없음| CHECK2{대기 큐에<br/>여유?}
    CHECK2 -->|있음| QUEUE["대기 큐에 저장<br/>순서 대기 ⏳"]
    CHECK2 -->|없음| REJECT["❌ 503 에러<br/>요청 거부"]

    style ASSIGN fill:#d4edda
    style QUEUE fill:#fff3cd
    style REJECT fill:#ff6b6b,color:#fff
```

### @Transactional과 동시성 문제

```java
@Transactional
public void deductStock(Long productId) {
    Product p = repository.findById(productId);
    p.decreaseStock(1); // 재고 차감
    repository.save(p);
}
// ⚠️ 여러 스레드가 동시 호출하면 Race Condition!
```

### 해결 - DB 락 종류

```mermaid
graph TB
    LOCK["DB 락으로 동시성 해결"]
    LOCK --> OPT["낙관적 락<br/>Optimistic Lock"]
    LOCK --> PES["비관적 락<br/>Pessimistic Lock"]

    OPT --> OPT_D["버전 번호로 충돌 감지<br/>충돌 시 재시도<br/>@Version 사용"]
    PES --> PES_D["SELECT FOR UPDATE<br/>읽을 때부터 잠금<br/>충돌 원천 차단"]

    style LOCK fill:#0f3460,color:#fff
    style OPT fill:#d4edda
    style PES fill:#e8f4f8
```

```mermaid
sequenceDiagram
    participant T1 as 스레드 A
    participant DB as DB
    participant T2 as 스레드 B

    Note over T1,T2: 비관적 락 (Pessimistic Lock)
    T1->>DB: SELECT ... FOR UPDATE 🔒
    Note over DB: 해당 행 잠금
    T2->>DB: SELECT ... FOR UPDATE
    Note over T2: ⏳ 대기 (락 해제까지)
    T1->>DB: UPDATE 재고 = 0
    T1->>DB: COMMIT → 🔓 락 해제
    DB->>T2: 락 획득 🔒
    T2->>DB: SELECT → 재고 = 0
    Note over T2: 재고 없음 → 차감 안 함
    T2->>DB: COMMIT → 🔓 락 해제
    Note over DB: ✅ 오버셀링 방지!
```

```mermaid
sequenceDiagram
    participant T1 as 스레드 A
    participant DB as DB (version=1)
    participant T2 as 스레드 B

    Note over T1,T2: 낙관적 락 (Optimistic Lock)
    T1->>DB: SELECT (재고=1, version=1)
    T2->>DB: SELECT (재고=1, version=1)
    T1->>DB: UPDATE SET 재고=0, version=2<br/>WHERE version=1
    Note over DB: ✅ 성공 (version 1→2)
    T2->>DB: UPDATE SET 재고=0, version=2<br/>WHERE version=1
    Note over DB: ❌ 실패! version이 이미 2<br/>OptimisticLockException
    Note over T2: 재시도 또는 실패 처리
```

| 방식 | 구현 | 적합한 상황 |
|------|------|-----------|
| 비관적 락 | `@Lock(PESSIMISTIC_WRITE)` | 충돌 많은 경우 (재고 차감) |
| 낙관적 락 | `@Version` 필드 | 충돌 적은 경우 (게시글 수정) |

### 스프링의 스레드 안전한 설계

```mermaid
graph TB
    subgraph 안전["스프링 빈 = 싱글톤 + Stateless ✅"]
        BEAN["@Service UserService<br/>(인스턴스 1개)"]
        T1A["스레드 1"] --> BEAN
        T2A["스레드 2"] --> BEAN
        T3A["스레드 3"] --> BEAN
        Note1["상태(필드)를 가지지 않으면<br/>동시 접근해도 안전"]
    end

    subgraph 위험["Stateful 빈 ❌"]
        BEAN2["@Service 에 필드 변수<br/>int count = 0"]
        WARN["⚠️ 여러 스레드가<br/>같은 필드 수정 → Race Condition"]
    end

    style 안전 fill:#d4edda
    style 위험 fill:#ffe4e1
```

```java
// ✅ 안전 - Stateless (필드에 상태 없음)
@Service
public class UserService {
    private final UserRepository userRepository; // 주입받은 의존성만

    public User findUser(Long id) {
        return userRepository.findById(id); // 지역 변수만 사용
    }
}

// ❌ 위험 - Stateful (필드에 상태 있음)
@Service
public class CounterService {
    private int count = 0; // 공유 상태! → Race Condition 위험

    public void increment() {
        count++; // 여러 스레드가 동시에 접근 가능
    }
}
```

## 핵심 요약

| 개념 | 핵심 내용 | 백엔드 연결 |
|------|----------|------------|
| 스레드 | 프로세스 안의 실행 흐름 | 톰캣이 요청마다 스레드 할당 |
| 메모리 공유 | Heap 공유, Stack 독립 | Stateless 설계가 중요한 이유 |
| Race Condition | 동시 접근 → 데이터 불일치 | 재고 오버셀링, 좋아요 카운트 |
| Deadlock | 서로 락 대기 → 영원히 멈춤 | DB 트랜잭션 교착, 자원 순서 고정 |
| 동기화 | Mutex, Semaphore, Monitor | synchronized, @Lock |
| 스레드 풀 | 미리 생성 & 재사용 | 톰캣 기본 200개, 커넥션 풀 |
| DB 락 | 낙관적/비관적 락 | @Version, @Lock(PESSIMISTIC_WRITE) |
