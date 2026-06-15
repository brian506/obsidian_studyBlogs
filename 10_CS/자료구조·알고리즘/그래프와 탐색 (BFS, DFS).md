# 그래프와 탐색 (BFS, DFS)

## 그래프 (Graph)
- **정점(Vertex/Node) + 간선(Edge)** 의 집합
- 종류: **방향/무방향**, **가중치/비가중치**, **순환/비순환(DAG)**, 연결/비연결

## 표현 방식

### 인접 행렬 (Adjacency Matrix)
- 2D 배열 `adj[i][j] = 1`이면 i→j 간선 있음
- 공간 O(V²), 간선 존재 확인 O(1)
- **간선이 많을 때(dense)** 유리

### 인접 리스트 (Adjacency List)
- 각 정점마다 연결된 정점들의 리스트
- 공간 O(V + E), 간선 존재 확인 O(degree)
- **희소 그래프(sparse)** 표준. 실무 대부분 이걸 씀

```java
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i <= V; i++) adj.add(new ArrayList<>());
adj.get(u).add(v);
```

## DFS (Depth-First Search) — 깊이 우선
한 방향으로 깊게 들어갔다가 막히면 되돌아옴.

```java
void dfs(int u) {
    visited[u] = true;
    for (int v : adj.get(u)) {
        if (!visited[v]) dfs(v);
    }
}
```
- **구현**: 재귀(암묵 스택) 또는 명시적 스택
- 시간 **O(V + E)**, 공간 O(V) (재귀 깊이)
- 용도: **사이클 검출**, 위상 정렬, 연결 요소 카운트, 백트래킹

## BFS (Breadth-First Search) — 너비 우선
가까운 노드부터 차근차근 넓혀감.

```java
void bfs(int start) {
    Queue<Integer> q = new ArrayDeque<>();
    q.offer(start); visited[start] = true;
    while (!q.isEmpty()) {
        int u = q.poll();
        for (int v : adj.get(u)) {
            if (!visited[v]) { visited[v] = true; q.offer(v); }
        }
    }
}
```
- **구현**: 큐
- 시간 **O(V + E)**, 공간 O(V)
- 용도: **최단 경로 (가중치 없는 그래프)**, 레벨 탐색

## DFS vs BFS

| | DFS | BFS |
| --- | --- | --- |
| 자료구조 | 스택(재귀) | 큐 |
| 메모리 | 보통 적음 (깊이만) | 보통 많음 (한 레벨 전체) |
| 최단 경로 (비가중치) | ❌ 보장 X | ✅ 보장 |
| 사이클·연결요소·위상정렬 | ✅ 적합 | 가능하지만 DFS가 자연스러움 |
| 백트래킹 | ✅ 자연스러움 | ❌ |

## 최단 경로 알고리즘
| 알고리즘 | 가중치 | 음수 가중치 | 시간 |
| --- | --- | --- | --- |
| **BFS** | 없음 (모두 1) | 없음 | O(V + E) |
| **Dijkstra** | 양수만 | ❌ | O((V + E) log V) — 힙 사용 |
| **Bellman-Ford** | 음수 가능 | ✅ (음수 사이클 검출) | O(VE) |
| **Floyd-Warshall** | 가능 | ✅ | O(V³) — 모든 쌍 최단 경로 |

## 위상 정렬 (Topological Sort)
**DAG(방향 비순환 그래프)** 에서 선후 관계를 만족하는 정렬.
- 작업 스케줄링, 빌드 의존성 (Gradle), 강의 선수 과목
- 방법 1: **DFS + 후위 순서 역순**
- 방법 2: **진입 차수 0인 노드부터 BFS (Kahn)**

## MST (Minimum Spanning Tree)
모든 정점을 연결하면서 가중치 합이 최소인 트리.
- **Kruskal**: 간선 정렬 + Union-Find로 사이클 회피 — O(E log E)
- **Prim**: 시작 정점에서 최소 간선 확장 — 우선순위 큐로 O((V+E) log V)

## Union-Find (Disjoint Set)
- 원소가 어느 집합인지 빠르게 확인 + 두 집합 합치기
- 최적화(경로 압축 + 랭크): **거의 O(1)** (정확히 O(α(N)), α는 아커만 역함수)
- 용도: Kruskal, 네트워크 연결성 판정

## 면접 빈출 질문
1. **DFS와 BFS 차이와 각 용도?** → 스택 vs 큐. DFS는 사이클·백트래킹, BFS는 비가중치 최단 경로
2. **DFS를 재귀로 구현 시 주의점?** → 깊이가 깊으면 **StackOverflow** — 명시적 스택 또는 BFS 고려
3. **인접 행렬과 인접 리스트 차이?** → 공간 O(V²) vs O(V+E). 희소 그래프는 리스트가 효율적
4. **BFS가 비가중치 최단 경로를 보장하는 이유?** → 가까운 거리부터 레벨 단위로 방문 → 처음 도달한 시점이 최단
5. **가중치 있는 최단 경로는?** → 양수만이면 Dijkstra, 음수 있으면 Bellman-Ford
6. **위상 정렬은 언제 쓰나?** → 빌드 의존성, 작업 순서 결정 (DAG 필수)
7. **Union-Find의 시간 복잡도?** → 경로 압축 + 랭크 시 거의 O(1) (amortized)

## 응용
- **소셜 네트워크 친구 추천** = BFS (2단계 친구)
- **마이크로서비스 의존성** = DAG + 위상 정렬
- **네트워크 라우팅** = Dijkstra
- **Kafka의 토픽 종속 관계, 빌드 시스템** = 위상 정렬
