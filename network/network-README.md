# 네트워크 (Network)

> 컴퓨터 간 데이터를 주고받는 원리. 백엔드 개발의 근간이 되는 핵심 지식.

## 챕터 목록

1. [OSI 7계층](./01-osi-7-layer.md)
   - OSI 7계층 각 역할
   - 데이터 캡슐화 / 역캡슐화
   - PDU (Protocol Data Unit)
   - OSI 7계층 vs TCP/IP 4계층

2. [TCP & UDP](./02-tcp-udp.md)
   - TCP 특징, 세그먼트 구조
   - 3-Way Handshake (연결 수립)
   - 4-Way Handshake (연결 해제)
   - 흐름 제어 (Stop-and-Wait, Sliding Window)
   - 혼잡 제어 (Slow Start, Congestion Avoidance)
   - UDP 특징, TCP vs UDP 비교

3. [HTTP 심화](./03-http.md)
   - HTTP 특징 (Stateless, 비연결성)
   - 요청/응답 구조
   - HTTP 메서드, 멱등성
   - 상태 코드 (2xx ~ 5xx)
   - 헤더, 쿠키/세션
   - HTTP/1.0 → 1.1 → 2.0 → 3.0 변화

4. [HTTPS & DNS](./04-https-dns.md)
   - 대칭키 / 공개키 (비대칭키) 암호화
   - HTTPS 동작 원리
   - TLS Handshake (1.2 / 1.3)
   - SSL 인증서, 인증서 체인
   - DNS 구조, 레코드 타입
   - DNS 조회 과정 (재귀 조회)
   - 브라우저에 URL 입력 시 전체 흐름
