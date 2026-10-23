# DAG 알고리즘과 위상정렬

## 1. DAG (Directed Acyclic Graph) 개념

- **정의**: 방향성(Directed)이 존재하고, 순환(`Acyclic`, 사이클)이 존재하지 않는 그래프를 의미합니다.
- **핵심 특징**:
    - **방향성**: 간선(Edge)에 방향이 있어 한쪽 방향으로만 이동할 수 있습니다.
    - **비순환성**: 어떤 정점에서 출발해 간선을 따라 다시 자기 자신으로 돌아오는 경로(사이클)가 절대 존재하지 않습니다.
- **주요 활용**:
    - 작업 순서 결정 (예: 대학 선수과목 체계)
    - 패키지 의존성 관리 (예: `npm`, `pip`)
    - 빌드 시스템 (예: `Make`, `Gradle`)

## 2. 위상 정렬 (Topological Sort) 개념

- **정의**: DAG의 모든 정점을 **방향성에 거스르지 않도록 순서대로 나열하는 것**입니다.
- **조건**: 그래프가 반드시 DAG(방향 비순환 그래프)여야 합니다. 사이클이 존재하면 선후 관계를 정의할 수 없으므로 위상 정렬을 수행할 수 없습니다.
- **특징**:
    - 정점 A에서 정점 B로 가는 간선이 존재한다면, 위상 정렬 결과에서 정점 A는 정점 B보다 **항상 먼저** 등장합니다.
    - 선후 관계가 명확하지 않은 독립된 정점들에 따라 여러 개의 결과(정렬 경우의 수)가 존재할 수 있습니다.

## 2-1. DFS 기반 위상 정렬의 핵심 원리

1. **3가지 상태 관리 (`visited` 배열)**:
    - `0` (미방문): 아직 탐색하지 않은 노드
    - `1` (방문 중): 현재 DFS 경로상에 있는 노드 (**사이클 감지용**)
    - `2` (방문 완료): 탐색이 완전히 끝난 노드
2. **후위 순회와 역순 배치**:
    - DFS를 수행하면서 더 이상 갈 수 없는 가장 깊은(후행) 노드부터 처리하고, 함수가 종료될 때 결과 리스트에 추가합니다.
    - 이렇게 얻은 결과 리스트를 역순(Reverse)으로 뒤집으면 올바른 위상 정렬 순서가 됩니다.
3. **사이클 감지**:
    - 탐색 중인 노드가 다시 `1`(방문 중) 상태인 노드를 만나면 사이클(순환)이 존재하는 것이므로 위상 정렬이 불가능함을 판단할 수 있습니다.
    

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%201.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%202.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%203.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%204.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%205.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%206.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%207.png)

![image.png](DAG%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98%EA%B3%BC%20%EC%9C%84%EC%83%81%EC%A0%95%EB%A0%AC/image%208.png)

이미지 출처 : [https://yonghwankim-dev.tistory.com/222](https://yonghwankim-dev.tistory.com/222)

## 2-2. 파이썬 예제 코드 (DFS 활용)

```
def topological_sort_dfs(v_count, edges):
# 1. 인접 리스트 생성
graph = [[] for _ in range(v_count + 1)]
for u, v in edges:
graph[u].append(v)

# 2. 방문 상태 배열 초기화 (0: 미방문, 1: 방문 중, 2: 방문 완료)
visited = [0] * (v_count + 1)
result = []
has_cycle = False

def dfs(node):
    nonlocal has_cycle
    if has_cycle:
        return

    visited[node] = 1  # 현재 경로에 포함 (방문 중)

    for next_node in graph[node]:
        # 다음 노드가 이미 현재 경로에 포함되어 있다면 사이클 발생
        if visited[next_node] == 1:
            has_cycle = True
            return
        # 아직 방문하지 않은 노드라면 DFS 재귀 호출
        elif visited[next_node] == 0:
            dfs(next_node)

    visited[node] = 2  # 탐색 완료
    result.append(node)  # 후위 순회 순서대로 결과에 추가

# 3. 모든 노드에 대해 DFS 수행 (비연결 그래프 고려)
for i in range(1, v_count + 1):
    if visited[i] == 0:
        dfs(i)
        if has_cycle:
            return "사이클이 발생하여 위상 정렬을 수행할 수 없습니다."

# 4. 후위 순회 결과를 뒤집어야 올바른 위상 정렬 순서가 됨
result.reverse()
return result
```