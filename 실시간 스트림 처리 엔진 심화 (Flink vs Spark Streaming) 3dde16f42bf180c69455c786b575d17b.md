# 실시간 스트림 처리 엔진 심화 (Flink vs. Spark Streaming)

### 1. Micro-batching vs. Native Continuous Processing (근본적인 아키텍처 차이)

스트림 처리 엔진을 고를 때 가장 먼저 마주하는 분기점은 데이터를 처리하는 방식(Execution Model)의 차이입니다.

- **Spark Structured Streaming (마이크로 배치 모델):**
    - **원리:** 진짜 실시간처럼 보이지만, 내부적으로는 아주 짧은 시간 단위(예: 100ms ~ 1초)의 미니 배치(Micro-batch)로 데이터를 잘라서 처리합니다.
    - **장점:** 기존 Spark의 강력한 배치 엔진(Catalyst, Tungsten)과 최적화 기법을 그대로 물려받아 대규모 처리량(Throughput)과 안정성이 뛰어납니다.
    - **단점:** 마이크로 배치 단위로 끊기 때문에 물리적 지연 시간(Latency)을 수십 밀리초 이하로 줄이기는 어렵습니다.
    
    ```jsx
    [ 실시간 데이터 스트림 (Unbounded) ]
      ---> [ 데이터 수집 버퍼 ] ---> ( ⏱️ 100ms 타이머 째깍! )
                                      |
                                      v
                            [ Micro-Batch #1 생성 ]
                                      |
                                      v
                            [ Spark Driver / Optimizer ]
                                      |
                                      v
                            [ Executor에서 병렬 처리 ] ---> [ 결과 저장 ]
    ```
    
- **Apache Flink (네이티브 컨티뉴어스 프로세싱 / 이벤트 기반 모델):**
    - **원리:** 데이터가 시스템에 들어오는 즉시(Record-by-record) 파이프라인을 타고 흘러가며 처리되는 **True Real-time (Event-driven)** 엔진입니다.
    - **장점:** 지연 시간이 밀리초(Millisecond) 단위로 매우 짧으며, 복잡한 실시간 패턴 매칭(Complex Event Processing, CEP)에 압도적으로 유리합니다.

```jsx
[ 실시간 데이터 스트림 ] 
  ---> (Record 1) ---> [ Operator A ] ---> (Record 1 변환) ---> [ Operator B ] ---> [ 결과 출력 ]
  ---> (Record 2) ---> [ Operator A ] ---> (Record 2 변환) ---> [ Operator B ] ---> [ 결과 출력 ]
  ---> (Record 3) ---> [ Operator A ] ---> (Record 3 변환) ---> [ Operator B ] ---> [ 결과 출력 ]
```

### 2. 스트림 처리의 꽃: 상태 관리 (State Management)

배치(Batch) 처리는 입력 데이터만 알면 결과를 내지만, 스트림(Stream) 처리는 "지난 1시간 동안의 누적 사용자 방문 수"나 **"현재까지의 세션 상태"** 같은 상태(State)를 메모리나 저장소에 유지하고 있어야 합니다.

- **Stateful Stream Processing:**
    - 스트림의 각 이벤트가 들어올 때마다 이전 상태를 조회하고 업데이트한 뒤 결과를 뿜어내는 구조입니다.
- **문제점:** 서버가 갑자기 죽으면 메모리에 들고 있던 '상태(State)'가 날아갑니다. 실시간 스트리밍 시스템에서 장애가 나도 상태가 유실되지 않도록 보장하는 것이 엔진의 핵심 기술입니다.

### 3. 분산 상태의 복구: Chandy-Lamport 알고리즘과 체크포인팅

Apache Flink는 분산 환경에서 상태를 안전하게 백업하고 복구하기 위해 **Chandy-Lamport 분산 스냅샷 알고리즘**을 변형한 **Asynchronous Barrier Snapshotting (ABS)** 방식을 사용합니다.

1. **Checkpoint Barrier (체크포인트 배리어):**
    - 스트림 데이터 흐름 사이에 특수한 신호(Barrier)를 주기적으로 주입합니다. 이 배리어는 데이터와 함께 흘러갑니다.
2. **Asynchronous Snapshot (비동기 스냅샷):**
    - 배리어가 연산자(Operator)에 도달하면, 연산자는 현재 자신들의 메모리 상태(State)를 멈추지 않고 비동기적으로 분산 스토리지(S3, HDFS 등)에 백업합니다.
3. **Fault Tolerance (장애 복구):**
    - 노드가 죽었다가 살아나면, 가장 최근에 성공한 체크포인트 시점으로 상태를 복원(Rollback)하고, 오프셋을 재조정하여 데이터 유실이나 중복 없이 정확히 복구합니다.
    
    관련 링크 : https://www.alibabacloud.com/blog/597985
    

### 4. 실시간 윈도우(Windowing) 연산

끝없이 들어오는 무한한(Unbounded) 데이터 스트림을 유한한 단위로 묶어서 집계하려면 **윈도우(Window)** 개념이 필수적입니다.

- **Tumbling Window (텀블링 윈도우):** 서로 겹치지 않는 고정 크기의 윈도우 (예: 10분 단위로 끊어서 집계).

![image.png](%EC%8B%A4%EC%8B%9C%EA%B0%84%20%EC%8A%A4%ED%8A%B8%EB%A6%BC%20%EC%B2%98%EB%A6%AC%20%EC%97%94%EC%A7%84%20%EC%8B%AC%ED%99%94%20(Flink%20vs%20Spark%20Streaming)/image.png)

- **Sliding Window (슬라이딩 윈도우):** 서로 겹칠 수 있는 윈도우 (예: 10분 동안의 데이터를 1분마다 갱신하여 계산).

![image.png](%EC%8B%A4%EC%8B%9C%EA%B0%84%20%EC%8A%A4%ED%8A%B8%EB%A6%BC%20%EC%B2%98%EB%A6%AC%20%EC%97%94%EC%A7%84%20%EC%8B%AC%ED%99%94%20(Flink%20vs%20Spark%20Streaming)/image%201.png)

- **Session Window (세션 윈도우):** 사용자의 활동 간격을 기준으로 동적으로 묶이는 윈도우 (예: 유저가 30분 동안 아무 행동이 없으면 세션 종료).

![image.png](%EC%8B%A4%EC%8B%9C%EA%B0%84%20%EC%8A%A4%ED%8A%B8%EB%A6%BC%20%EC%B2%98%EB%A6%AC%20%EC%97%94%EC%A7%84%20%EC%8B%AC%ED%99%94%20(Flink%20vs%20Spark%20Streaming)/image%202.png)