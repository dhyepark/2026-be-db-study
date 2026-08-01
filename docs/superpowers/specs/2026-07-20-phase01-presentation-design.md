# Phase01 MySQL 면접 대비 발표 디자인

## 목적

MySQL 내부 동작과 인덱스를 처음 학습하는 컴퓨터공학 4학년이 1~5주차 개념을 하나의 성능 최적화 흐름으로 이해하고, DB 면접에서 실행 계획을 근거로 답할 수 있게 한다.

발표가 끝났을 때 청중은 “MySQL은 어떻게 덜 읽고, 덜 정렬하고, 테이블 접근을 줄이는가?”라는 질문에 `EXPLAIN`과 인덱스 구조를 연결해 답해야 한다.

## 산출물

- 형식: PowerPoint `.pptx`
- 분량: 본문 29장 + 부록 3장, 총 32장
- 예상 발표 시간: 50~60분
- 화면 비율과 시각 시스템: Simple Light Mode retained reference 유지
- 소스: `phase01` Markdown 7개, 포함된 이미지, MySQL 8.x 공식 문서
- 발표자 지원: 핵심 슬라이드에 면접 답변 한 문장과 발표자 노트 포함

## 핵심 원칙

1. `type`은 점수가 아니라 접근 방식이다. `rows`, `filtered`, `key`, `Extra`와 함께 본다.
2. `possible_keys → key → key_len → ref`는 하나의 인덱스 선택 이야기로 읽는다.
3. `rows × filtered / 100`은 다음 연산으로 넘어갈 예상 행 수이며 실제값은 `EXPLAIN ANALYZE`로 확인한다.
4. `Extra`는 좋고 나쁜 라벨이 아니라 접근 이후 추가 작업을 설명한다.
5. `Using index`, `Using index condition`, `Using index for group-by`를 서로 다른 의미로 설명한다.
6. 모든 주차는 “읽기량과 추가 작업을 줄인다”는 동일한 축으로 연결한다.

## 내러티브와 슬라이드 구조

### 도입 — 2장

1. `MySQL은 어떻게 덜 읽는가?`
2. `다섯 주차는 하나의 실행 계획으로 연결된다`

### 1주차 — 구조와 실행 흐름 — 2장

3. MySQL 엔진과 InnoDB의 역할 분리
4. Parser → Preprocessor → Optimizer → Execution Engine → Storage Engine

### EXPLAIN 집중 해설 — 11장

5. `EXPLAIN`과 `EXPLAIN ANALYZE`의 차이
6. 추정값인데도 `EXPLAIN`을 보는 이유: 옵티마이저가 추정값으로 계획을 고른다
7. `EXPLAIN → EXPLAIN ANALYZE` 검증 흐름: 예상과 실제의 차이를 개선 단서로 사용한다
8. 한 줄을 `table → type → key 계열 → rows/filtered → Extra` 순서로 읽기
9. `ALL`과 `index`: 테이블 전체와 인덱스 전체 스캔
10. `range`, `ref`, `eq_ref`, `const`: 발생 조건과 SQL 예시
11. `possible_keys`, `key`, `key_len`, `ref`의 관계
12. `rows × filtered`와 조인 반복 비용
13. `Using where`, `Using index`, `Using index condition`
14. `Using filesort`, `Using temporary`, `Using index for group-by`
15. `EXPLAIN ANALYZE`의 `actual time`, `rows`, `loops`

### 2주차 — B+Tree와 복합 인덱스 — 3장

16. B+Tree 페이지 구조와 높은 fan-out
17. Index Range Scan의 seek → scan → row lookup
18. 복합 인덱스 왼쪽 접두사, 첫 range 이후 컬럼, SARGability

### 3주차 — 옵티마이저·정렬·조인 — 4장

19. 선택도와 전체 비용으로 `ALL`도 선택될 수 있다
20. 인덱스 순서와 `Using filesort`
21. Single-pass와 two-pass filesort
22. 드라이빙 테이블, Nested Loop Join, `LIMIT` 조기 종료

### 4주차 — GROUP BY와 DISTINCT — 3장

23. `WHERE → GROUP BY → HAVING`으로 입력 행을 먼저 줄인다
24. Tight Index Scan과 Loose Index Scan
25. `DISTINCT`, `Using temporary`, `Using index for group-by (scanning)`

### 5주차 — InnoDB와 커버링 — 2장

26. 클러스터드 인덱스와 세컨더리 인덱스의 이중 탐색
27. 커버링 인덱스와 ICP 비교

### 종합과 면접 — 2장

28. 하나의 쿼리를 다섯 주차 기준으로 판독
29. 면접에서 실행 계획을 설명하는 다섯 문장

### 부록 — 3장

30. 드물게 보이는 `type`: `system`, `ref_or_null`, `index_merge`, 서브쿼리 계열
31. Extra 용어표와 오해 방지
32. 면접 빠른 질문 12개와 한 문장 답변

## 이미지 사용 계획

- 구조: `01-mysql-server-architecture.png`, `01-mysql-query-execution-flow.png`
- B+Tree: `02_B-tree.jpg`
- 접근 방식: `03_01`, `03_02`, `03_03`, `03_04`, `03_08` EXPLAIN 캡처
- 그룹핑: `03_10`, `03_13`, `03_14`, `03_17` 캡처
- 커버링/ICP: `05_covering_index_*`, `05_optimizer_index_selection.png`
- 캡처는 전체 화면 장식이 아니라 관련 열을 읽을 수 있는 크기로 자르거나 두 화면을 같은 배율로 비교한다.
- 잘린 캡처에는 쿼리와 확인 가능한 열만 표시하며 누락된 값을 추정하지 않는다.
- 깨진 `03_11`과 Loose Index Scan 증거가 아닌 `03_12`는 근거 이미지로 사용하지 않는다.

## 정확성 보정

- `Using index`는 커버링이며 일반적인 “인덱스 사용” 표시가 아니다.
- `Using index condition`은 ICP이며 통과한 레코드의 테이블 접근은 남는다.
- `Using filesort`는 별도 정렬이며 반드시 디스크 정렬이라는 뜻은 아니다.
- `Using temporary`는 내부 임시 테이블이며 반드시 디스크 테이블이라는 뜻은 아니다.
- `filtered`는 버려지는 비율이 아니라 조건을 통과할 예상 비율이다.
- `id`를 같은 SELECT 안의 물리적 실행 순서로 단정하지 않는다.
- `possible_keys=NULL`이어도 정렬·커버링·풀 인덱스 스캔 목적으로 `key`가 선택될 수 있다.

## 템플릿 적용

Simple Light Mode의 26개 원본 슬라이드를 전부 검사하고, 각 출력 슬라이드를 가장 가까운 원본 레이아웃에 매핑한다. 원본의 Aptos 계열 타이포그래피, 흑백 팔레트, 여백, 푸터와 페이지 번호를 유지한다. 이미지 설명은 원본의 half-image, comparison, grid, checklist, data-table 레이아웃을 우선 사용한다.

## 완료 기준

- 총 32장, 50~60분 분량
- 주차별 핵심 내용이 독립적으로 보이면서 전체 최적화 흐름으로 연결됨
- `type`, `key` 계열, `rows/filtered`, `Extra`, `EXPLAIN ANALYZE`를 면접 수준으로 설명
- 실제 캡처의 `type`, `key`, `rows`, `Extra`가 발표 화면에서 읽힘
- `Using index` 계열 세 표현을 혼동하지 않음
- 모든 슬라이드의 렌더링, 오버플로, 빈 placeholder, 템플릿 fidelity 검사 통과
