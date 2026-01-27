# 부하 테스트 + 최적화 관련

## 부하테스트 전체 구조 

![이미지 1](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076860/load-test/abrikk0prljzzoqnwrig.avif)

## 부하 테스트 환경 설정 관련

- 커널 수준 리소스 제한 활용 


## cgroup 설정 관련
### 컨테이너별 자원 할당
- 핵심 그룹 (blog-loadtest, mysql-container)
  - 서비스의 메인 로직과 데이터 처리를 담당하므로 가장 높은 우선순위 부여.
  - AWS t3.small 환경의 하드웨어 제약을 가정하여 저사양에서의 안정성 검증 목적

- 지원 그룹 (prometheus, grafana)
  - 성능 측정의 신뢰성을 위해 지표 수집이 끊기지 않는 것을 목표로 할당

- 보조 그룹 (my_nginx_server, exporter)
  - 트래픽 중계 및 메트릭 노출 등 단순 반복 작업을 수행하는 요소
  - 전체 시스템 자원 낭비를 방지하기 위해 구동 가능한 최소한의 자원만 할당

| **컨테이너**        | **역할**    | **CPU Limit (상한)** | **Memory Limit (상한)** | **비고**             |
|---------------------| ----------- | ------- | ------------ | -------------------- |
| **blog-loadtest**   | Java 앱     | 1.25    | 2GB          | 메인 테스트 대상     |
| **mysql-container** | DB          | 1.25    | 2GB          | 쿼리 및 I/O 처리     |
| **prometheus**      | 지표 수집   | 1.0     | 1.5GB        | 수집 누락 방지       |
| **grafana**         | 시각화      | 0.5     | 1GB          | 원활한 그래프 렌더링 |
| **my_nginx_server** | 프록시      | 0.4     | 512MB        | 트래픽 중계          |
| **exporter 계열**     | 메트릭 노출 | 0.2     | 128MB        | 지표 데이터 제공     |

#### 제한 확인 체크 (docker stats)

```bash
docker stats
```



![이미지 2](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076875/load-test/wa6jzqb2bnbsaklx1gcl.avif)

---

## 더미 데이터 관련

### 더미 데이터 주입 시 사용한 Spring batch 코드

[Spring Batch git hub 코드 링크](https://github.com/Kimheojin/spring-batch-preprocessing?tab=readme-ov-file#2-dummydatajob-%EB%B6%80%ED%95%98-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EB%8C%80%EB%B9%84-%EB%8D%B0%EC%9D%B4%ED%84%B0-%EC%A0%81%EC%9E%AC)

### 더미데이터 주요 테이블 row 갯수 

| post_cnt | post_tag_cnt | tag_cnt | category_cnt |
| :--- | :--- | :--- | :--- |
| 3376000 | 5064604 | 50 | 100 |

![이미지 3](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076886/load-test/slfbm5vfnqlvmlyecyqr.avif)

## 부하 테스트 및 주요 수정 사항

- nginx keep alive (nginx - Spring)을 통한 서버 부하 감소
- 서버 spec 대비 과도한 데이터 부하 환경으로 인한 반 정규화를 통한 최적화
- connection 갯수 조정

### nginx keep alive 설정 적용 (nginx - spring)

#### 문제 상황

- K6 에서 100 ops 정도의 낮은 부하시 에도 Tomcat 의 쓰레드 수가 과하게 활성화
- 기존 nginx - Spring 간 Connection 미 재사용으로 인해 발생 추정

![이미지 4](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076899/load-test/i6hiqshyfs17t6sb7my6.avif)

##### 최적화 전 nginx 설정

```
server {
    listen 443 ssl;
    server_name heojineee.ddnsking.com;

    location /api {
        # 백엔드 서버로 직접 연결
        proxy_pass http://blog-loadtest:9003;
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

#### 최적화 후 nginx 설정
- **TCP 핸드셰이크 비용 절감**
  - 매 요청마다 발생하는 연결(3-way) 및 종료(4-way) 과정을 생략하여 네트워크 지연 시간 단축
  - 단시간에 많은 요청이 몰릴 때 발생하는 OS 포트 고갈(Ephemeral Port Exhaustion) 문제 방지

- **연결 재사용 (Persistent Connection)**
  - Nginx와 Spring Boot 사이에 연결된 소켓을 끊지 않고 유지
  - 요청마다 소켓을 새로 생성하는 부하를 줄여 서버 자원을 효율적으로 사용


```
# 백엔드 연결 풀 설정
upstream blog_backend {
    server blog-loadtest:9003;
    keepalive 32; # 유지할 idle 커넥션 수
    # 주의: keepalive 수는 예상되는 동시 접속 트래픽과 WAS의 Thread Pool 사이즈를 고려하여 설정해야 합니다.
}

server {
    listen 443 ssl;
    server_name heojineee.ddnsking.com;

    location /api {
        proxy_pass http://blog_backend;
        
        # Keep-alive 활성화를 위한 필수 설정
        proxy_http_version 1.1; # HTTP/1.0은 Keep-alive를 지원하지 않으므로 1.1 강제
        proxy_set_header Connection ""; # Client가 보낸 'close' 헤더를 무시하고 Upstream과 연결 유지
        
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 주요 변경사항 상세 분석

- **Upstream Keep-alive**
  - 단순한 연결 유지를 넘어 Spring 서버로 보내는 연결을 캐싱하여 재사용함
  - 설정값(32)은 각 Nginx 워커 프로세스가 유지할 수 있는 유휴(Idle) 연결의 최대 개수를 의미함

- **proxy_http_version 1.1**
  - 연결 유지가 기본 사양인 HTTP/1.1을 명시적으로 사용함
  - 설정하지 않을 경우 Nginx는 기본값인 1.0을 사용하여 매 요청마다 연결을 끊게 됨

- **proxy_set_header Connection ""**
  - 클라이언트가 보낸 연결 종료 요청(Connection: close)이 백엔드까지 전달되지 않도록 헤더를 초기화함
  - Nginx와 Spring 사이의 소켓이 불필요하게 닫히는 것을 방지하여 연결 유지 효율을 높임

#### 성능 개선 결과

![이미지 5](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076912/load-test/mtwcvwhpmkwo1ccojqze.avif)

- Tomcat Thread 관리 효율화
  - Current Threads 수치가 최대치(60)에서 안정 수치 (22 ~ 25) 로 약 60% 감소
  
- 동일한 초당 부하량 (100ops) 을 유지하면서도 서버 자원 가용상 향상 기대

### 반 정규화를 통한 쿼리 최적화

#### 문제 상황

- K6를 통한 부하 환경에서 카테고리별 게시글 수 조회 시 337만 건의 데이터를 실시간 집계함에 따라 
  - CPU 점유율 128% 초과 및 메모리 94% 점유 발생. 초당 100회 요청(100 OPS) 처리 불가.
  - 또한 과도한 쓰레드 점유 상황 발생 

![이미지 6](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076925/load-test/ssvlathwfa6roopwqk0l.avif)

![이미지 7](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076944/load-test/mstkje1ahesqls4bnynh.avif)


#### 원인 상세 분석

- **대량 데이터 처리 비용 과다**
  - `COUNT(*)` 쿼리는 인덱스가 존재하더라도 조건에 부합하는 모든 레코드를 전수 스캔해야 함
  - 데이터가 300만 건을 상회할 경우 과도한 CPU 연산과 디스크 I/O를 발생시켜 DB 성능 저하 유발

- **스레드 차단(Thread Blocking) 및 장애 전파**
  - DB 응답이 지연되는 동안 Spring Boot의 워커 스레드가 커넥션을 장기간 점유하며 대기 상태로 유지됨
  - 이는 스레드 풀 고갈로 이어져, 해당 쿼리와 무관한 다른 요청들까지 처리하지 못하는 연쇄 장애(Cascading Failure) 발생 가능성 증대

#### 반정규화(De-normalization) 최적화 전략

> "조회 성능 향상을 위해 쓰기 성능과 데이터 정합성 관리 비용을 지불함"

- **전략: 카운트 필드 추가**
  - `Category` 테이블에 `post_count` 컬럼을 추가하여 게시글 수를 미리 저장
  - 조회 시 별도의 연산 없이 저장된 값을 즉시 반환($O(1)$)하여 DB 부하를 근본적으로 해결

- **Trade-off (고려사항)**
  - **장점**: 데이터 규모와 관계없이 일정한 응답 속도를 보장하며 DB CPU 부하가 거의 없음
  - **단점**: 게시글 작성 및 삭제 시마다 `Category` 테이블의 `UPDATE`가 수반되어 쓰기 트랜잭션 비용 증가
  - **정합성 관리**: 동시성 이슈로 인해 실제 게시글 수와 차이가 발생할 수 있으므로, 주기적인 배치(Batch) 작업을 통한 데이터 동기화(Sync)가 필요함

#### 적용 쿼리 및 로직

```aiexclude
-- 1. Category 엔티티에 컬럼 추가
ALTER TABLE category ADD COLUMN post_count INT DEFAULT 0;

-- 2. 기존 데이터 업데이트 (1회성)
UPDATE category c SET c.post_count = (
    SELECT COUNT(*) FROM post p 
    WHERE p.category_id = c.category_id AND p.status = 'PUBLISHED'
);

-- 3. 조회 쿼리
SELECT category_id, category_name, post_count, priority 
FROM category 
ORDER BY priority ASC, category_name ASC;
```
![이미지 8](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076957/load-test/ymb8cjcyknjyoaqo3xkm.avif)

#### 최적화 결과 확인

- **CPU 및 메모리 안정화 (위 이미지)**
  - 반정규화 적용 후 JOIN 연산 제거로 인해 CPU 점유율이 안정권으로 하락
  - 메모리 사용 효율 개선

![이미지 9](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076968/load-test/afpy3ir40b12w9ggv7ed.avif)

- **TPS 처리량 향상 (위 이미지)**
  - 쿼리 비용 감소로 인해 초당 처리 가능한 요청 수(TPS)가 목표치에 근접하게 상승

---

### connection 수 개선

#### 문제 상황

![이미지 10](https://res.cloudinary.com/dtrxriyea/image/upload/v1769076995/load-test/ma9p0jmybhowakzkt8ur.avif)

- **현상**: 500 ops 부하 테스트 시 목표 처리량을 달성하지 못함
- **원인**: `HikariCP Pending Connections` 지표가 급증하는 것으로 보아, DB 커넥션 풀 고갈로 인한 병목 현상 확인

#### 해결 방안 및 결과

### 적용 후

![이미지 11](https://res.cloudinary.com/dtrxriyea/image/upload/v1769077008/load-test/euohzrpjfjvmnuqbglzw.avif)

#### HikariCP 커넥션 풀(Connection Pool) 최적화

- **조치: 최대 풀 크기(maximum-pool-size) 상향 조정**
  - 제한된 서버 자원($1.25$ vCPU)을 고려하여 기본값인 $10$에서 $20$으로 변경

- **이론적 근거 및 산정 공식**
  - HikariCP 및 주요 DB 진영에서 권장하는 공식 참고:
    $$connections = ((\text{core\_count} \times 2) + \text{effective\_spindle\_count})$$
  - 무분별한 확장보다는 CPU 코어 수와 디스크 I/O 효율을 고려한 수치 산출



- **과도한 설정 지양 (Why not 100?)**
  - 커넥션 수가 코어 수에 비해 지나치게 많으면, OS 스케줄러가 스레드 간 컨텍스트 스위칭(Context Switching)을 처리하느라 실제 쿼리 연산보다 CPU 오버헤드가 더 커짐

- **설정 값 도출 과정**
  - **기본값 (10)**: 부하 발생 시 커넥션 획득 대기열(Pending)이 형성되어 응답 시간 지연 발생
  - **최적화 (20)**: $1.25$ Core 환경에서 컨텍스트 스위칭 부하를 억제하면서 대기 시간을 해소하는 적정 지점으로 확인

- **결과**
  - 커넥션 획득 지연 없이 안정적인 처리량(Throughput) 확보 및 자원 사용 효율 극대화

```yaml
  # DB 커넥션 풀 수치 조정 (1.25 Core 환경)
  datasource:
    hikari:
      maximum-pool-size: 20 # 20으로 조정: 동시 트래픽 수용량 증대
      minimum-idle: 10 # 최소 유휴 커넥션 확보
      connection-timeout: 5000 # 5초 내 연결 실패 시 에러 (빠른 실패 전략)
      pool-name: HikariCP-Performance
```



## k6 부하 테스트 지표 비교 (stats_denormalized_500.js)

> **주요 지표 관련**
>
> - **TPS (Transactions Per Second)**: 초당 처리된 트랜잭션 수. 시스템의 실제 처리 용량(Throughput)을 의미
> - **p(95) 응답 시간**: 전체 요청 중 95%가 이 시간 이내에 완료됨을 의미
> - **Dropped Iterations**: 부하 생성기(Client)가 목표 부하를 맞추려 했으나, 서버가 응답하지 않아 생성을 포기한 요청 수

| 지표 항목 | 이전 테스트 (실패) | 현재 테스트 (개선) | 비고 |
| :--- | :--- | :--- | :--- |
| **Target TPS** | 500 iters/s | 500 iters/s | 동일 설정 |
| **Actual TPS** | 449.44 iters/s | 469.43 iters/s | 처리량 약 4.4% 향상 |
| **p(95) 응답 시간** | 1.14s | 87.66ms | 약 13배 개선 (성능 핵심) |
| **Max 응답 시간** | 26.51s | 2.13s | 지연 시간 대폭 감소 |
| **Dropped Iterations** | 5250건 | 146건 | 누락 요청 급감 |
| **Max 사용 VU** | 300 (제한 도달) | 164 | 리소스 사용 효율 증가 |
| **Thresholds 결과** | ✗ 실패 | ✓ 성공 | 모든 기준 충족 |



---
## 기타 링크

- [온프레미스 서버 스펙 정리](server-specifications.md)
- [README 이동](https://github.com/Kimheojin/spring-blog-backend/blob/main/README.md)