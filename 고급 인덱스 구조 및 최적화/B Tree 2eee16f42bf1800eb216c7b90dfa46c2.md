# B Tree

[https://velog.io/@emplam27/자료구조-그림으로-알아보는-B-Tree](https://velog.io/@emplam27/%EC%9E%90%EB%A3%8C%EA%B5%AC%EC%A1%B0-%EA%B7%B8%EB%A6%BC%EC%9C%BC%EB%A1%9C-%EC%95%8C%EC%95%84%EB%B3%B4%EB%8A%94-B-Tree)

## 1. Definition

B-Tree는 노드 하나에 여러가지 키를 가지고 있고, 이진트리와 달리 자식 노드가 2개 이상 가질 수 있는 트리이다.

- 차수를 M 이라 가정하자
- 키의 갯수 : (M/2) - 1 개 부터 M-1개의 키를 가질 수 있다.
- 모든 리프 노드들은 항상 같은 depth에 존재한다.

![image.png](B%20Tree/image.png)

## 2. Search