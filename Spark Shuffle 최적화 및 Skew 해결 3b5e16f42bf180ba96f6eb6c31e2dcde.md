# Spark Shuffle 최적화 및 Skew 해결

### 1. Shuffle은 왜 비용이 큰가? (OS & Hardware 관점)

셔플은 **Map-side(쓰기) -> Network(전송) -> Reduce-side(읽기)** 과정을 거칩니다.

1. **Disk I/O:** 맵 작업 완료 후 중간 데이터를 로컬 디스크에 씁니다(Spilling to disk).
2. **Network I/O:** 셔플 서비스를 통해 다른 익스큐터(Executor)로 네트워크 패킷을 전송합니다.
3. **Serialization:** 데이터를 전송하기 위해 객체를 바이트 단위로 직렬화(Serialization)하고, 다시 역직렬화하는 CPU 연산이 발생합니다.

이 과정에서 **디스크 쓰기/읽기**와 **네트워크 대역폭**이 점유되므로, 셔플 횟수를 줄이는 것이 스파크 튜닝의 제1원칙입니다.

### 2. Shuffle 최적화 전략 (The Optimization Checklist)

#### ① Broadcast Join (대소 테이블 조인)

가장 강력한 최적화 기법입니다. 조인할 두 테이블 중 하나가 Executor의 메모리에 다 올라갈 만큼 작다면, 셔플 없이 모든 노드로 작은 테이블을 복사(Broadcast)해서 조인을 수행합니다.

- **아키텍처적 이점:** Wide Dependency가 Narrow Dependency로 바뀝니다(네트워크 셔플 제거).
- **고려사항:** `spark.sql.autoBroadcastJoinThreshold` 설정을 적절히 조절해야 합니다.
- Broadcast join 참고자료 : [https://velog.io/@hyungki26/Broadcast-Join-Optimization-완전-가이드-Databricks에서-조인-성능을-극대화하는-법](https://velog.io/@hyungki26/Broadcast-Join-Optimization-%EC%99%84%EC%A0%84-%EA%B0%80%EC%9D%B4%EB%93%9C-Databricks%EC%97%90%EC%84%9C-%EC%A1%B0%EC%9D%B8-%EC%84%B1%EB%8A%A5%EC%9D%84-%EA%B7%B9%EB%8C%80%ED%99%94%ED%95%98%EB%8A%94-%EB%B2%95)

#### ② Map-side Aggregation (셔플 전 미리 집계)

`groupByKey` 대신 `reduceByKey` 또는 `aggregateByKey`를 사용하십시오.

- **왜?** `groupByKey`는 셔플 후에 모든 데이터를 모아서 집계하지만, `reduceByKey`는 셔플 전에 각 파티션(Map-side)에서 미리 로컬 컴바이너(Combiner)를 통해 데이터를 압축합니다. 이는 네트워크를 타고 넘어가는 데이터 양을 획기적으로 줄여줍니다.
- groupByKey와 reduceByKey의 차이 : [https://seamless.tistory.com/101](https://seamless.tistory.com/101)

#### ③ 파티셔닝 전략 (Partitioning)

- 조인 시 `join`할 키들이 이미 동일한 파티셔너(Partitioner)로 파티셔닝되어 있다면, 셔플을 건너뛸 수 있습니다. `partitionBy`를 사용하여 데이터를 미리 정렬/분할해 두는 습관이 필요합니다.

### 3. 데이터 스큐(Data Skew)와 해결책

데이터 스큐는 "특정 파티션에만 데이터가 몰리는 현상"입니다. 분산 시스템의 고질적인 문제인 "Straggler(지연자) 문제"를 일으킵니다. (예: 99개의 태스크는 1초 만에 끝나는데, 마지막 1개의 태스크가 1시간 동안 도는 상황).

#### 왜 스큐가 일어나는가?

- 데이터의 분포가 균등하지 않음(Zipf 분포). 예를 들어, 'null' 값이나 특정 인기 상품(예: '아이폰') ID가 너무 많아 한 파티션에 다 모이는 경우.
- 

#### 해결 전략: Salting (소금 뿌리기)

Skew가 발생한 키(예: '아이폰')에 임의의 번호(0~N)를 붙여서 데이터를 강제로 쪼개는 기법입니다.

1. **스큐된 키에 접두사(Salt) 추가:** '아이폰' -> '아이폰_1', '아이폰_2', ... '아이폰_N'
2. **조인 상대 테이블 확장:** 상대 테이블은 '아이폰' 키를 N배 복제하여 조인할 수 있게 만듭니다.
3. **조인 실행:** 분산된 키들이 각자 다른 파티션에서 처리되므로 특정 노드에 과부하가 걸리지 않습니다.
4. **최종 집계:** 조인 결과에서 Salt를 제거하고 다시 집계합니다.

salting 참고자료 : [https://gyuhoonk.github.io/spark-salting](https://gyuhoonk.github.io/spark-salting)