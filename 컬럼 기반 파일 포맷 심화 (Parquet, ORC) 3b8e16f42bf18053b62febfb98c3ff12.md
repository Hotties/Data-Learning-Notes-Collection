# 컬럼 기반 파일 포맷 심화 (Parquet, ORC)

### 1. Row-oriented vs. Column-oriented (물리적 저장 구조의 차이)

데이터를 디스크나 메모리에 쓸 때, 어떤 순서로 배치하느냐에 따라 시스템의 성능이 극단적으로 갈립니다.

- **Row-oriented (행 기반 포맷, 예: CSV, JSON, Avro):**
    - 구조: 첫 번째 행의 모든 컬럼, 두 번째 행의 모든 컬럼 순으로 연속해서 저장합니다. (`[id1, name1, age1], [id2, name2, age2]`)
    - 장점: 특정 행의 모든 정보를 한 번에 읽어야 하는 OLTP(트랜잭션)에 유리합니다.
- **Column-oriented (열 기반 포맷, 예: Parquet, ORC):**
    - 구조: 같은 컬럼의 데이터끼리 모아서 연속해서 저장합니다. (`[id1, id2, id3], [name1, name2, name3], [age1, age2, age3]`)
    - 장점: 특정 컬럼들만 골라서 분석하는 OLAP(집계/분석)에 압도적으로 유리합니다.

OLTP와 OLAP의 차이점 : [https://aws.amazon.com/ko/compare/the-difference-between-olap-and-oltp/](https://aws.amazon.com/ko/compare/the-difference-between-olap-and-oltp/)

### 2. 컬럼 포맷의 핵심 무기: Projection Pushdown과 Vectorized Reading

#### A. Projection Pushdown (열 단위 필터링)

쿼리에서 요청한 컬럼만 디스크에서 읽어옵니다.

- *효과:* 디스크 I/O 대역폭을 극적으로 아낄 수 있습니다. 100GB짜리 파일이라도 필요한 컬럼이 전체의 10%라면, 실제 읽는 데이터는 훨씬 줄어듭니다.

Projection Pushdown과 Predicate Pushdown : [https://m.blog.naver.com/simula/223938077873](https://m.blog.naver.com/simula/223938077873)

#### B. Vectorized Reading (벡터화된 읽기)과 SIMD

컬럼(행) 기반 포맷은 데이터 타입이 완전히 동일한 값들이 연속해서 배열 형태로 저장되어 있습니다.

- 현대 CPU는 **SIMD (Single Instruction, Multiple Data)** 명령어를 지원하여, 한 번의 CPU 사이클로 여러 데이터를 동시에 처리할 수 있습니다.(참고 : [https://rhksgml78.tistory.com/304](https://rhksgml78.tistory.com/304))
- 벡터화된 리더(Vectorized Reader)는 데이터를 메모리 버퍼에 덩어리(Batch)로 한 번에 올려놓고 CPU 캐시 효율을 극대화하여 연산 속도를 수 배 이상 높입니다.

### 3. 고효율 압축과 인코딩 (Encoding & Compression)

컬럼 기반 포맷이 작은 용량을 차지하면서도 압축률이 압도적으로 높은 이유는 **동일한 타입의 데이터가 연속해서 모여 있기 때문**입니다. 내부적으로 다음과 같은 저수준 인코딩 기법이 적용됩니다.

1. **Dictionary Encoding (사전 인코딩):**
    - 중복이 많은 문자열 컬럼(예: 국가명 `KR`, `US`, `KR` ...)을 숫자로 변환합니다. (`KR -> 1, US -> 2`) 이후에는 가벼운 정수 배열로 처리합니다.
2. **Run-Length Encoding (RLE):**
    - 연속으로 같은 값이 반복될 때, 그 횟수와 값으로 압축합니다. (예: `5, 5, 5, 5, 5` -> `(5, 5개)`)
3. **Bit-Packing:**
    - 숫자의 크기가 작을 때, 32비트나 64비트 대신 실제 필요한 비트 수(예: 3비트)만 사용하여 공간을 촘촘히 압축합니다.

이러한 인코딩을 거친 뒤 Snappy나 Gzip 같은 범용 압축 알고리즘을 한 번 더 거치기 때문에, CSV 대비 파일 크기가 **1/3 ~ 1/5 수준**으로 줄어듭니다.

### 4. Parquet의 물리적 내부 파일 구조

Parquet 파일은 단순히 데이터만 들어있는 것이 아니라, 내부에 메타데이터(Metadata)를 체계적으로 품고 있습니다.

- **File:** 최상위 파일 단위.
- **Row Group:** 데이터를 논리적으로 나눈 단위 (보통 128MB ~ 1GB). 이 안에서 행들이 나뉩니다.
- **Column Chunk:** Row Group 안의 특정 컬럼 데이터 모음.
- **Page:** Column Chunk를 더 쪼갠 최소 독해 단위 (보통 1MB 내외).
- **File Footer (파일 푸터):** **가장 중요한 부분입니다!** 파일의 끝에 위치하며, 각 Row Group의 컬럼별 Min / Max 통계값(Statistics)과 메타데이터가 기록되어 있습니다.
    - *Predicate Pushdown 효과:* 쿼리에 `WHERE age > 50`이 들어오면, 스파크는 데이터를 실제로 열어보지도 않고 **Footer의 Min/Max 값만 확인**하여 해당 범위에 데이터가 아예 없는 Row Group은 통째로 건너뜁니다(Row Group Skipping).

![image.png](%EC%BB%AC%EB%9F%BC%20%EA%B8%B0%EB%B0%98%20%ED%8C%8C%EC%9D%BC%20%ED%8F%AC%EB%A7%B7%20%EC%8B%AC%ED%99%94%20(Parquet,%20ORC)/image.png)