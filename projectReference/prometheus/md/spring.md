# Prometheus spring 관련 지표

![spring-1](../metricImage/spring/spring-1.png)

### 시스템 기본 정보

- **Uptime** 
  - 애플리케이션  구동 시간
- **Start time**
  - 해당 인스턴스가 최초로 실행된 시점


### 메모리 사용 현황

- **Heap Used**
  - JVM 힙 메모리의 **할당량** 중 사용량

- **Non-Heap Used**
  - Metaspace 등 힙 이외의 JVM 메모리 사용량


### 파일 시스템 자원

- **Process Open Files**
  - 애플리케이션이 현재 점유 중인 **파일 디스크립터**(File Descriptor) 수
  - 재사용 자원 사용이 적을 경우 이 지표가 튈나? 
    - (나중에 확인해보기)
  

## Load Average

- **Load Average**는 시스템의 **전체적인 부하 수준**을 나타내는 지표

- 실행 중이거나 대기 중인 프로세스의 평균 수를 의미
- Load Average가 높다면 **CPU 연산이 너무 많거나, 디스크/네트워크 병목으로 인해 프로세스들이 멈춰 있는 상황** 중 하나

### 해석 기준: CPU 코어 수

- 이 수치는 절대적인 것이 아니라 시스템의 **CPU Core 수**와 비교해서 해석 필요

- **기준점**: 코어 1개당 Load 1.0이 "풀 가동" 상태를 의미
- **사용자 차트 분석**: 현재 시스템의 `CPU Core Size`는 **2**
  - **Load 2.0 미만**
    - 시스템이 여유롭거나 감당 가능한 수준
  - **Load 2.0 초과**
    - 대기열이 발생하기 시작한 과부하 상태



![spring-2](../metricImage/spring/spring-2.png)



## 참고 사진

![참고 사진](../metricImage/spring/참고 사진.webp)



### 힙(Heap) 영역 분석

- **G1 Eden Space**
  - 메모리 사용량이 `Committed` 라인까지 차오르다 급격히 떨어지는 지점은 Minor GC가 발생하여 살아남은 객체가 Survivor 영역으로 이동했음을 의미
    - 살아남은 건, 당장은 GC 대상이 아니라는 의미
    - *commit* - 할당 이라는 의미로 쓰이는 듯
  - 19:55분 이후 주기가 짧아지는 것은 객체 생성 속도가 빨라졌음을 나타낼수도
- **G1 Survivor Space**
  - Eden 영역에서 살아남은 객체들이 머무는 공간
  - 19:50분부터 20:00분 사이에 사용량(`Used`)이 크게 요동치는 것은 부하로 인해 생존 객체 수가 일시적으로 급증
- **G1 Old Gen**
  - 오랜 기간 살아남은 객체가 저장
  - 부하 테스트 과정에서 처리량이 많아짐에 따라 일부 객체가 Survivor 영역에서 Old 영역으로 승격(Promotion)되었음을 의미

### 비힙(Non-Heap) 영역 분석

- **Metaspace**
  - 클래스, 메서드 등의 메타데이터를 저장하는 영역
  - 일정 수준에서 변동 없이 직선을 유지하고 있음
    - 이는 런타임 중에 새로운 클래스가 동적으로 로드되거나 누수되는 현상이 없을 수도
- **CodeHeap (non-nmethods, profiled, non-profiled)**
  - JIT(Just-In-Time) 컴파일러가 생성한 **네이티브 코드**를 저장하는 영역
  -  `profiled nmethods` 수치가 27.3MiB 정도로 유지되는 등 전체적으로 안정적인 상태
- **Compressed Class Space**
  -  클래스 포인터 압축을 위한 공간





![spring-3](../metricImage/spring/spring-3.png)



### Threads 지표

- **Live Threads**
  - 현재 애플리케이션 내에서 실행 중인 전체 스레드 수
  - 유입된 요청을 처리하기 위해 톰캣(Tomcat) 등의 WAS 스레드 풀이 확장되었음을 나타냄
- **Daemon Threads**
  - JVM 내부의 백그라운드 작업을 수행하는 스레드
  - Live 스레드 수의 상승 폭과 거의 일치
  - 증가한 작업 스레드 대부분이 데몬 스레드로 설정되어 관리되는 거 같음

## Memory Allocate/Promote

- **Allocated**
  - Eden 영역에 새로운 객체가 할당되는 속도
  - 이는 트래픽 증가에 따라 비즈니스 로직 처리를 위한 객체 생성이 단시간에 집중되었음을 의미
- **Promoted** 
  - Young 영역에서 살아남아 Old 영역으로 이동한 객체의 양
  - 객체 할당량(Allocated)의 급증에 비해 승격량(Promoted)은 매우 미미한 수준을 유지

### Metaspace (Non-Heap) 지표

- 클래스, 메서드, 상수 풀 등의 메타데이터를 저장하는 영역
- 애플리케이션 실행 중에 새로운 클래스가 동적으로 계속 로드되거나, 클래스 로더와 관련된 메모리 누수가 발생하지 않는 상태



![spring-4](../metricImage/spring/spring-4.png)



## 클래스 로딩 지표 (Classes Loaded / Unloaded)

- **Classes Loaded**

  - 현재 JVM에 로드되어 있는 클래스의 총 개수
  - 그래프가 수평선(Flat)을 유지하는 것은 애플리케이션 구동 초기 이후 추가적인 클래스 로딩이 발생하지 않았음을 의미
  
- **Classes Unloaded**
  -  제거된 클래스가 전혀 없음을 나타냄
  -  런타임 중에 동적으로 클래스를 생성하고 소멸시키는 로직(예: 리플렉션의 과도한 사용이나 동적 스크립트 엔진)이 없는 상태

## 다이렉트 버퍼

- **지표 의미**
  - JVM 힙(Heap) 외부의 네이티브 메모리를 직접 사용하는 버퍼
  - 주로 NIO(Non-blocking I/O) 통신 시 데이터 복사 비용을 줄여 네트워크 성능을 높이기 위해 사용

## 매핑된 버퍼

- **지표 의미**
  - 메모리 맵 파일(Memory-mapped files)을 처리하기 위해 할당된 영역 
  - 파일 I/O 성능 향상을 위해 파일을 메모리에 직접 매핑할 때 사용
  
- **상태 분석**
  - 현재 그래프상 수치가 0에 수렴하며 변동이 없음
  - 이는 해당 부하 테스트 시나리오에서 대용량 파일 입출력이나 mmap을 사용하는 작업은 포함되지 않았음을 뜻하는 듯



![spring-5](../metricImage/spring/spring-5.png)



## JVM Statistics - GC

- **GC Count (G1 Evacuation Pause)**
  - 객체 생성량이 많아져 Eden 영역이 빠르게 찼음을 의미

- **GC Stop the World Duration**
  - duration - 소요 시간 (사실 지속이 나한텐 익숙)
  - GC로 인해 애플리케이션이 멈춘 시간


## HikariCP Statistics

- **Connections Size** 
  - 현재 설정된 전체 커넥션 풀의 크기가 15개임을 나타냄

- **Connections (Active / Idle / Pending)**
  - **Active**
    - 실제 DB 쿼리를 수행 중인 커넥션
    - 부하 시점에 최대치인 15개에 도달하여 풀이 가득 참
  - **Pending**
    - 커넥션을 할당받기 위해 대기 중인 스레드 수
    - 이는 애플리케이션의 처리 능력이 DB 연결 한계에 부딪혔음을 보여주는 핵심 지표
  - **Idle**
    - 사용되지 않고 대기 중인 커넥션
    - 부하 시점에는 0으로 떨어졌다가 부하 종료 후 다시 15로 복구 된듯
- **Connection Timeout Count**
  - 커넥션을 대기하던 스레드가 설정된 타임아웃 시간 내에 커넥션을 획득하지 못해 에러가 발생한 횟수
  - 총 3건의 요청이 DB 연결 실패로 인해 사용자에게 에러를 반환했을 가능성이 있음




![spring-6](../metricImage/spring/spring-6.png)



### Connection Pool 지표

DB 커넥션 효율성을 나타내는 지표들입니다.

- **Connection Creation Time**
  - 새로운 DB 커넥션을 생성하는 데 걸리는 시간
  - 그래프상 약 4.61ms로 유지되다가 특정 시점에 변동이 발생
  - 이 수치가 급증하면 DB 서버 부하 또는 네트워크 지연을 의심 하는 것도 나쁘지 않을듯
- **Connection Usage Time**
  - 애플리케이션이 커넥션을 빌려가서 실제 사용(Query 실행 등)하고 반납할 때까지의 시간
  - 비즈니스 로직의 처리 속도나 쿼리 성능을 반영
- **Connection Acquire Time**
  - 애플리케이션이 Pool에서 커넥션을 빌리기 위해 대기한 시간
  - **핵심 지표** 
    - 이 시간이 길어지면 Pool에 가용한 커넥션이 없어 스레드가 차단(Blocking)되고 있다는 신호

## Jetty Statistics (사실 근데 Tomcat 사용)

서블릿 컨테이너(WAS)의 스레드 설정 및 요청 처리 상태

- **Thread Config Min / Max**:
  - Jetty가 사용할 최소/최대 스레드 개수 설정값입니다. 현재 **N/A**로 표시된 것은 메트릭 수집 설정이 누락되었거나 활성화되지 않았음을 나타냄
- **HTTP Codes (Throughput)**:
  - 초당 처리되는 HTTP 응답 코드별 빈도(ops/s)
  - **2xx (Success)**
    - 파란색 선으로 표시되며, 현재 시스템의 처리량(Throughput)을 보여줌
    - 19:50~20:00 사이에 요청이 집중적으로 발생했음을 알 수 있음
  - **4xx/5xx (Errors)**
    - 클라이언트 및 서버 에러 발생 여부를 모니터링
    - 현재는 거의 발생하지 않는 안정적인 상태





![spring-7](../metricImage/spring/spring-7.png)



### Requests/second (초당 요청 수)

- 애플리케이션의 **처리량(Throughput)**을 의미

### Requests duration (요청 처리 시간)

사용자가 요청을 보낸 후 응답을 받기까지 걸리는 **지연 시간(Latency)**을 의미

- **Average(녹색)**
  - 평균 응답 시간은 매우 낮게 유지되고 있음

- **Maximum(노란색)**
  - 두 번째 트래픽 피크 시점에 최대 응답 시간이 **3.70초**까지 급증

- **의미**
  - 대부분의 요청은 빠르게 처리되나, 부하가 몰릴 때 특정 요청들이 심각하게 느려지는 '꼬리 지연(Tail Latency)' 현상이 발생하고 있음


### TOP 10 (상위 엔드포인트)

어떤 API가 서버 부하의 주범인지 보여주는 리스트

- **대상**
  -  `/api/categories/stats/denormalized` 엔드포인트가 가장 많은 요청을 처리하고 있음

- **수치**
  - 해당 API의 Max 값이 428인 것으로 보아, 위에서 본 Requests/second의 피크 수치와 일치

- **의미**
  - 현재 서버 부하와 응답 지연의 직접적인 원인이 이 특정 API에 집중되어 있음을 시사





