# Apache Spark 아키텍쳐와 내부 실행 원리

스파크의 가장fundamental한 추상화 단위는 RDD입니다. 이름에 담긴 세 가지 단어가 스파크 아키텍처의 핵심 철학을 보여줍니다.

- **Resilient (탄력성/내결함성):** 데이터가 유실되어도 처음부터 다시 계산(Recomputation)할 수 있는 구조입니다. (Hadoop이 데이터를 디스크에 복제하여 fault tolerance를 해결했다면, 스파크는 **Lineage**를 통해 재계산으로 해결합니다.)
- **Distributed (분산):** 데이터가 클러스터 내 여러 노드에 걸쳐 파티셔닝되어 있습니다.
- **Dataset (데이터셋):** 구조화되거나 비구조화된 데이터의 모음입니다.

#### RDD의 3대 핵심 특성 (CS 관점):

1. **불변성 (Immutability):** RDD는 생성 후 변경할 수 없습니다. 변형(Transformation)을 가하면 새로운 RDD가 생성되며, 이는 함수형 프로그래밍의 철학을 따릅니다.
2. **지연 연산 (Lazy Evaluation):** `map`, `filter` 같은 Transformation 연산은 즉시 실행되지 않고, 연산의 계보(Lineage Graph)에 기록만 됩니다. `count`, `collect`, `save` 같은 **Action** 연산이 호출되는 순간 비로소 DAG가 생성되고 실행됩니다. (이를 통해 옵티마이저는 전체 쿼리를 최적화할 기회를 얻습니다.)
3. **Lineage (계보):** RDD가 어떤 과정을 거쳐 만들어졌는지 기록한 의존성 그래프입니다. 노드가 다운되면 이 Lineage를 거슬러 올라가 유실된 파티션만 정확히 다시 계산합니다.

### 2. Spark Cluster 컴포넌트의 역할

스파크 애플리케이션이 실행될 때 클러스터 내부에서는 크게 세 가지 역할이 상호작용합니다.

| **컴포넌트** | **역할 및 설명** | **컴퓨터 공학적 비유** |
| --- | --- | --- |
| **Driver (드라이버)** | 사용자 코드를 실행하여 SparkSession을 만들고, 전체 논리적/물리적 계획을 수립한 뒤 Task를 Executor로 배포함. | **지휘자 / 컨트롤러** |
| **Executor (익스큐터)** | 드라이버가 보낸 Task를 받아 실제로 데이터를 처리하고, 결과를 메모리나 디스크에 캐싱함. 노드당 보통 여러 개가 뜸. | **작업자 / 워커 스레드** |
| **Cluster Manager** | 클러스터의 자원(CPU, 메모리)을 관리하고 드라이버의 요청에 따라 Executor를 할당함. (YARN, Kubernetes, Mesos 등) | **운영체제 프로세스 스케줄러** |

![image.png](Apache%20Spark%20%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90%EC%99%80%20%EB%82%B4%EB%B6%80%20%EC%8B%A4%ED%96%89%20%EC%9B%90%EB%A6%AC/image.png)

### 3. DAG 스케줄러와 Stage 분할 (Narrow vs. Wide Dependency)

[DAG 알고리즘과 위상정렬](Apache%20Spark%20%EC%95%84%ED%82%A4%ED%85%8D%EC%B3%90%EC%99%80%20%EB%82%B4%EB%B6%80%20%EC%8B%A4%ED%96%89%20%EC%9B%90%EB%A6%AC/DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC%203b4e16f42bf1809fbbdae6c658c7b0db.md)

스파크 성능 최적화의 핵심은 **네트워크 통신(Shuffle)을 언제, 얼마나 줄이느냐**에 달렸습니다. 이를 결정하는 것이 RDD 간의 의존성 타입입니다.

#### ① Narrow Dependency (좁은 의존성)

- 부모 RDD의 파티션 하나가 오직 **하나의 자식 RDD 파티션**에만 영향을 주는 경우 (`map`, `filter`, `union` 등).
- **특징:** 데이터가 다른 노드로 이동할 필요가 없습니다(Network Shuffle 없음). 메모리 내에서 파이프라인(Pipeline)으로 묶여 한 번에 처리될 수 있어 매우 빠릅니다.

#### ② Wide Dependency (넓은 의존성)

- 부모 RDD의 파티션 데이터가 **여러 자식 RDD 파티션으로 쪼개져 섞여야 하는** 경우 (`groupByKey`, `reduceByKey`, `join` 등).
- **특징:** 데이터가 클러스터 전역의 네트워크를 타고 이동해야 합니다. 이 지점에서 스파크는 연산을 멈추고 Stage를 분할(Stage Boundary)합니다. 이 과정을 셔플(Shuffle)이라고 부릅니다.

shuffle이란 : [https://firststep-de.tistory.com/54](https://firststep-de.tistory.com/54)

Dependency 참고 : [https://github.com/rohgar/scala-spark-4/wiki/Wide-vs-Narrow-Dependencies](https://github.com/rohgar/scala-spark-4/wiki/Wide-vs-Narrow-Dependencies)

⚠️ **DE의 성능 팁:** Wide Dependency는 디스크 I/O와 네트워크 대역폭을 동시에 소모하는 성능 병목의 주범입니다. (예: `groupByKey` 대신 `reduceByKey`를 쓰라는 이유가 바로 네트워크로 전송되는 데이터 양을 줄이기 위함입니다.)

### 4. DataFrame/Dataset과 Catalyst Optimizer (현대 스파크)

초창기 스파크는 RDD 기반의 로우레벨 API 위주였지만, 현재는 최적화 엔진인 **Catalyst Optimizer**가 내장된 **DataFrame / Dataset** API가 표준입니다.

- **Catalyst Optimizer:** SQL 및 DataFrame 연산 쿼리를 받아 논리적 계획(Logical Plan)을 세우고, 불필요한 열을 미리 떨궈내거나(Predicate Pushdown) 조인 순서를 바꾸는 등의 비용 기반 최적화(CBO)를 수행한 뒤 물리적 계획으로 컴파일합니다.
- **Tungsten Engine:** 자바 객체 오버헤드를 줄이기 위해 메모리를 직접 관리(Off-heap memory management)하고, CPU 캐시 효율을 극대화하는 코드 생성(Whole-stage code generation) 기술을 적용하여 하드웨어 성능을 극한으로 끌어냅니다.