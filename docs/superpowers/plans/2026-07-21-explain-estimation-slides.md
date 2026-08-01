# EXPLAIN 추정값 해설 슬라이드 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 기존 30장 MySQL 면접 대비 덱의 5번과 6번 사이에 EXPLAIN 추정값의 의미와 검증 절차를 설명하는 2장을 추가한다.

**Architecture:** 기존 Simple Light Mode retained-reference 작업 공간과 artifact-tool 빌더를 재사용한다. frame map에 source slide 12의 4-card 레이아웃과 source slide 15의 4-step 레이아웃을 삽입하고, 기존 6~30번을 두 칸 이동시켜 32장 starter와 최종 PPTX를 다시 만든다.

**Tech Stack:** PowerPoint, `@oai/artifact-tool`, Node.js ES modules, Presentations template-following scripts, bundled presentation renderer

## Global Constraints

- 최종 파일: `outputs/phase01-mysql-interview-guide.pptx`
- 총 32장, 50~60분 발표
- 기존 30장 내용과 Simple Light Mode의 타이포그래피·여백·페이지 번호를 유지
- 6번은 “왜 추정값을 보는가”, 7번은 “예상과 실제를 어떻게 검증하는가”만 담당
- `rows=100 → actual rows=100,000` 예시와 `possible_keys=idx_status → key=NULL → type=ALL` 예시를 포함
- 32장 모두 발표자 노트를 유지
- 빈 placeholder, 텍스트 overflow, 의도하지 않은 overlap을 허용하지 않음

---

### Task 1: 32장 template frame map과 starter 생성

**Files:**
- Modify in external scratch: `tmp/template-frame-map.json`
- Create in external scratch: `tmp/expand-frame-map-to-32.mjs`
- Regenerate in external scratch: `tmp/template-starter.pptx`
- Regenerate in external scratch: `tmp/starter-edit-map.json`

**Interfaces:**
- Consumes: 기존 30장 frame map, Simple Light Mode `reference.pptx`
- Produces: 6번 source slide 12, 7번 source slide 15, 기존 6~30번을 +2 이동한 32장 starter

- [ ] `expand-frame-map-to-32.mjs`에서 기존 `outputSlides`를 읽고 아래 두 엔트리를 5번 뒤에 삽입한다.

```js
const reasonSlide = {
  ...structuredClone(source12Entry),
  outputSlide: 6,
  narrativeRole: "why explain estimates matter",
};
const workflowSlide = {
  ...structuredClone(source15Entry),
  outputSlide: 7,
  narrativeRole: "explain to analyze verification workflow",
};
```

- [ ] 기존 6~30번 엔트리의 `outputSlide`를 `+2`하고 전체가 1~32 연속인지 검증한다.
- [ ] `validate_template_plan.mjs`의 결과가 `status: pass`, `issueCount: 0`인지 확인한다.
- [ ] `prepare_template_starter_deck.mjs`로 32장 starter와 edit map을 생성한다.

### Task 2: 두 슬라이드 카피와 발표자 노트 추가

**Files:**
- Modify in external scratch: `tmp/phase01-mysql-interview-guide.mjs`

**Interfaces:**
- Consumes: 32장 starter와 기존 `slides` 배열
- Produces: 기존 5번 뒤에 들어가는 두 개의 slide config

- [ ] 6번 4-card 슬라이드를 아래 내용으로 추가한다.

```js
{
  text: {
    title: "추정값이어도 실행 계획을 고르는 근거다",
    intro: "EXPLAIN은 시간을 재는 도구가 아니라 옵티마이저의 판단을 설명한다.",
    "card-1": "실행 전 위험 발견\ntype=ALL · rows=5,000,000이면 실제 실행 없이도 읽기 폭을 의심한다.",
    "card-2": "선택 이유 확인\npossible_keys=idx_status여도 key=NULL이면 전체 스캔 비용이 더 싸다고 본 것이다.",
    "card-3": "변경 전후 비교\nALL → ref, rows 1,000,000 → 30처럼 접근 경로가 바뀌었는지 확인한다.",
    "card-4": "추정 오류도 단서\nrows=100인데 actual rows=100,000이면 통계·편향·상관관계를 의심한다."
  }
}
```

- [ ] 7번 4-step 슬라이드를 아래 내용으로 추가한다.

```js
{
  text: {
    title: "EXPLAIN으로 가설을 세우고 ANALYZE로 검증한다",
    intro: "예상과 실제의 차이가 다음 개선 작업을 결정한다.",
    "label-1": "1 · EXPLAIN",
    "desc-1": "type · key · rows · filtered · Extra로 예상 경로를 읽는다.",
    "label-2": "2 · ANALYZE",
    "desc-2": "actual time · actual rows · loops로 실제 작업량을 확인한다.",
    "label-3": "3 · 차이 해석",
    "desc-3": "rows=100 → actual rows=100,000이면 선택도 추정이 1,000배 틀렸다.",
    "label-4": "4 · 개선",
    "desc-4": "통계 갱신 · 히스토그램 · 인덱스 · 쿼리를 바꾸고 다시 비교한다."
  }
}
```

- [ ] 6번 노트에 “옵티마이저는 추정값으로 비용을 비교하므로 추정 계획을 읽는 것이 중요하다”는 면접 답변을 넣는다.
- [ ] 7번 노트에 `EXPLAIN ANALYZE`가 실제 실행한다는 주의점과 예상/실제 오차 진단 순서를 넣는다.
- [ ] 빌더가 `presentation.slides.items.length === 32`를 검증한 뒤 최종 PPTX를 export하도록 한다.

### Task 3: 32장 렌더링과 최종 검증

**Files:**
- Regenerate: `outputs/phase01-mysql-interview-guide.pptx`
- Inspect in external scratch: `tmp/preview/final/slide-06.png`
- Inspect in external scratch: `tmp/preview/final/slide-07.png`
- Inspect in external scratch: `tmp/qa/independent-render/*`

**Interfaces:**
- Consumes: 32장 builder 결과
- Produces: 시각·구조·템플릿 검증이 끝난 PPTX

- [ ] 6·7번을 원본 크기로 확인해 제목이 한 줄이고 모든 본문이 16pt 이상인지 확인한다.
- [ ] 32장 montage에서 페이지 번호가 1~32 연속이고 기존 30장 내용 순서가 유지되는지 확인한다.
- [ ] `check_template_fidelity.mjs` 결과가 `status: pass`, `issueCount: 0`인지 확인한다.
- [ ] PPTX 내부 XML에서 slide 32개, notesSlide 32개, 텍스트가 있는 notesSlide 32개, 빈 slide placeholder 0개인지 확인한다.
- [ ] `render_slides.py`로 최종 PPTX를 독립 렌더링하고 32장 전체에 잘림·겹침·깨진 이미지가 없는지 확인한다.

