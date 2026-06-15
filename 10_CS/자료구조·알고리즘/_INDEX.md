# 자료구조·알고리즘 — 면접 대비 인덱스

## 핵심 노트
- [[시간·공간 복잡도 (Big-O)]] — Big-O/Ω/Θ, amortized, 평균 vs 최악
- [[배열과 연결 리스트]] — ArrayList vs LinkedList, 캐시 친화성
- [[스택·큐·덱]] — LIFO/FIFO, ArrayDeque 권장
- [[해시 테이블]] — 해시 충돌, equals·hashCode, ConcurrentHashMap
- [[트리와 이진 탐색 트리 (BST, AVL, RB)]] — 균형 트리, B+ Tree, 순회
- [[힙과 우선순위 큐]] — Min/Max Heap, Top-K, Dijkstra
- [[그래프와 탐색 (BFS, DFS)]] — 인접 리스트, 최단경로, 위상정렬
- [[정렬 알고리즘]] — 퀵/병합/힙/Tim, 안정성·in-place

## 빈출 주제 우선순위
1. **HashMap 동작 원리** (충돌, resize, equals/hashCode 계약)
2. **DFS vs BFS** + 각 용도
3. **퀵 정렬 vs 병합 정렬** 비교
4. **균형 트리(Red-Black)와 B+ Tree** — Java TreeMap, DB 인덱스
5. **힙과 우선순위 큐** — Dijkstra, Top-K
6. **시간 복잡도 표기** (평균/최악, amortized)
7. **ArrayList vs LinkedList** — 캐시 친화

## 코딩 테스트 자주 쓰는 Java 컬렉션
```
ArrayList         — 기본 배열
ArrayDeque        — 스택/큐/덱 통합 (LinkedList보다 빠름)
HashMap / HashSet — 해시
TreeMap / TreeSet — 정렬·range query
PriorityQueue     — 힙
```

## 연결되는 다른 영역
- **DB 인덱스 (B+ Tree)** → [[10_CS/데이터베이스/인덱스/인덱스 구조 (B-Tree 자료구조)]]
- **운영체제 캐시 지역성** → [[10_CS/운영체제/캐시 메모리와 지역성]]
- **분산 큐 (Kafka)** → [[30_인프라·운영/Kafka]]
- **해시 = Redis 자료구조** → [[30_인프라·운영/Redis]]
