# 운영체제 (Operating System)

> 백엔드 서버가 요청을 처리하는 방식을 이해하기 위한 운영체제 지식

## 챕터 목록

1. [CPU](./01-cpu.md)
   - CPU 기본 구조 (ALU, CU, 레지스터)
   - 명령어 사이클, 파이프라이닝
   - 클럭 속도, 코어, 하이퍼스레딩
   - CPU 스케줄링 (FCFS, SJF, Round Robin, Priority)
   - 컨텍스트 스위칭
   - CPU 바운드 vs I/O 바운드
   - 톰캣 스레드 풀, WebFlux와의 연결

2. [메모리 (Memory)](./02-memory.md)
   - 메모리 계층 구조, Redis가 빠른 이유
   - 캐시 메모리, 캐시 지역성
   - 가상 메모리, 페이지 폴트, 페이지 교체 알고리즘
   - JVM 메모리 구조 (Heap, Stack, Method Area)
   - 가비지 컬렉션 (GC), Stop-The-World
   - 메모리 누수 원인과 해결

3. [프로세스 (Process)](./03-process.md)
   - 프로그램 vs 프로세스
   - 프로세스 메모리 구조 (Code, Data, Heap, Stack)
   - 프로세스 상태 전이
   - PCB (Process Control Block)
   - 프로세스 생성 (fork, exec)
   - IPC (파이프, 소켓, 공유 메모리, 메시지 큐)
   - MSA와 프로세스 격리, Docker 컨테이너

4. [스레드 (Thread)](./04-thread.md)
   - 프로세스 vs 스레드 비교
   - 스레드 메모리 구조 (공유 vs 독립)
   - 동시성 문제 (Race Condition, Deadlock)
   - 동기화 방법 (Mutex, Semaphore, Monitor)
   - 스레드 풀과 톰캣 요청 처리 흐름
   - @Transactional 동시성, DB 락 (비관적/낙관적)
   - 스프링의 Stateless 설계
