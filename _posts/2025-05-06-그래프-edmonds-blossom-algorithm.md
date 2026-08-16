---
title: "[그래프] Edmonds' Blossom Algorithm"
date: 2025-05-06 20:52:50 +0900
categories: [Algorithm, 그래프]
tags: []
---

### **Edmonds' Blossom Algorithm (에드먼즈 블라썸 알고리즘)**

일반적인 무방향 그래프에서 최대 매칭을 구하는 알고리즘 (General Maximum Unweighted Matching Algorithm)

#### **용어 정리**

**Matching (매칭)**: 정점들이 서로 겹치지 않도록 선택된 간선들의 집합, 어떠한 정점도 차수가 1보다 클 수 없음

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-01.png)

**Maximum Matching (최대 매칭)**: 가능한 가장 많은 간선이 포함된 매칭

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-02.png)

**Exposed Vertex (노출 정점)**: 어떤 매칭에 속하지 않은 정점

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-03.png)

**Alternating Path (교차 경로)**: 어떤 매칭 M에 대해, 매칭에 속한 간선과 매칭에 속하지 않은 간선이 번갈아가며 속하는 규칙이 반복되는 경로

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-04.png)

**Augmenting Path (증가 경로)**: 어떤 Alternating Path의 양 끝 점이 Exposed Vertex인 경로

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-05.png)

**Blossom**: Augmenting Path를 탐색하는 도중에 발견되는 홀수 사이클(2k+1)로, 사이클 내의 정확히 k개의 간선이 매칭에 포함되어 있고, 이로 인해 탐색이 막히는 구조

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-06.png)

#### **사전 지식**

- Augmenting path를 찾으면, Matching을 뒤집어 Matching의 크기를 1 키울 수 있음

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-07.png)

- 그래프에 Augmenting path가 더 이상 없으면, Maximum Matching 달성

- 그래프에서 찾은 Blossom을 B, B를 하나의 정점으로 압축한 새로운 그래프를 G', 이에 파생된 새로운 Matching을 M'이라 할 때,

G'이 M'을 이용하여 Augmenting Path를 구할 수 있으면 G에서도 M을 이용하여 Augmenting Path를 구할 수 있음

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-08.png)

1. G와 M이 있는 상태

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-09.png)

2. 찾은 Blossom B를 하나의 정점으로 압축하여 새로운 그래프 G'과 새로운 Matching M'을 생성

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-10.png)

3. Augmenting Path를 찾고, Matching을 뒤집어 크기를 늘림

![](/assets/img/posts/2025-05-06-그래프-edmonds-blossom-algorithm-11.png)

4. 기존 그래프에 적용

#### **알고리즘**

1. Exposed Vertex에서 BFS 시작   
2. BFS 중 Augmenting Path 찾기

3. Augmenting Path 탐색 중 홀수 사이클 만나면 Blossom 압축 후 계속 탐색   
4. Augmenting Path 발견 시 Matching을 뒤집음→ 크기 증가  
5. 모든 정점에 대하여 1 - 4의 단계를 반복하여 Maximum Matching 달성

#### **증명**
