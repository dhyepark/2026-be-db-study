# Phase01 MySQL 면접 대비 발표 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Simple Light Mode 템플릿으로 30장 내외·45~55분 분량의 Phase01 MySQL 면접 대비 발표를 만든다.

**Architecture:** retained reference PPTX를 검사하고 모든 출력 슬라이드를 원본 프레임에 매핑한다. artifact-tool로 starter PPTX를 가져와 상속된 텍스트·이미지 슬롯만 수정하고, 슬라이드별 PNG와 layout JSON을 생성해 반복 검증한다.

**Tech Stack:** PowerPoint, `@oai/artifact-tool`, Node.js ES modules, Presentations template-following scripts, Poppler/LibreOffice renderer

## Global Constraints

- 최종 파일: `outputs/phase01-mysql-interview-guide.pptx`
- 총 30장 내외, 45~55분 발표
- Simple Light Mode 원본의 레이아웃·타이포그래피·여백·페이지 표시를 유지
- visible copy는 청중용 문장만 사용하고 제작 메모는 speaker notes로 분리
- phase01 실제 캡처의 값을 확대하여 읽을 수 있게 함
- `Using index`, `Using index condition`, `Using index for group-by`를 분리 설명
- 빈 placeholder, 텍스트 overflow, 의도하지 않은 overlap을 허용하지 않음

---

### Task 1: 템플릿 검사와 30장 프레임 매핑

**Files:**
- Read: Simple Light Mode `assets/reference.pptx`
- Create in external scratch: `tmp/template-audit.txt`
- Create in external scratch: `tmp/template-frame-map.json`
- Create in external scratch: `tmp/deviation-log.txt`
- Create in external scratch: `tmp/template-starter.pptx`

**Interfaces:**
- Consumes: retained reference와 template-following scripts
- Produces: 30개 output slide의 source slide·edit target ID·narrative role

- [ ] artifact-tool scratch workspace를 초기화한다.
- [ ] reference 26장을 모두 렌더링·inspect하고 레이아웃 역할과 placeholder를 기록한다.
- [ ] 각 출력 슬라이드를 title, half-image, comparison, grid, checklist, table 레이아웃에 매핑한다.
- [ ] `validate_template_plan.mjs`로 모든 edit target이 실제 상속 요소인지 확인한다.
- [ ] `prepare_template_starter_deck.mjs`로 30장 starter를 생성하고 montage를 검사한다.

### Task 2: 슬라이드 카피와 출처 원장 확정

**Files:**
- Create in external scratch: `tmp/slide-copy.txt`
- Create in external scratch: `tmp/source-notes.txt`
- Create in external scratch: `tmp/assets/*`

**Interfaces:**
- Consumes: 디자인 문서, phase01 Markdown 7개, phase01 이미지, MySQL 공식 문서
- Produces: 슬라이드별 제목·본문·면접 한 문장·speaker notes·이미지 crop 정보

- [ ] 30개 슬라이드마다 하나의 takeaway 제목을 작성한다.
- [ ] 본문은 슬라이드당 3~5개 항목 또는 읽을 수 있는 SQL/EXPLAIN 하나로 제한한다.
- [ ] `type`과 Extra의 발생 조건·오해 방지 문장을 공식 문서와 대조한다.
- [ ] 실제 이미지 파일, crop, 설명할 컬럼을 `source-notes.txt`에 기록한다.
- [ ] 슬라이드 6~13, 22~25, 27~30에 면접 답변 한 문장이나 질문을 넣는다.

### Task 3: artifact-tool로 30장 편집

**Files:**
- Create in external scratch: `tmp/phase01-mysql-interview-guide.mjs`
- Read: `tmp/template-starter.pptx`
- Read: `tmp/template-frame-map.json`
- Create: `outputs/phase01-mysql-interview-guide.pptx`

**Interfaces:**
- Consumes: starter deck, mapped inherited IDs, slide copy, byte-backed PNG assets
- Produces: editable PPTX, slide PNGs, layout JSON, montage

- [ ] `PresentationFile.importPptx`로 starter를 가져오고 슬라이드 수를 검증한다.
- [ ] 모든 텍스트는 mapped inherited text target에 적용한다.
- [ ] 모든 이미지는 mapped inherited image target을 실제 phase01 PNG로 교체한다.
- [ ] screenshot 슬라이드는 동일 배율 비교와 `type/key/rows/Extra` 판독 순서를 유지한다.
- [ ] 핵심 슬라이드에 speaker notes를 추가한다.
- [ ] 30개 PNG, 30개 layout JSON, montage, PPTX를 export한다.

### Task 4: 슬라이드별 시각 QA와 수정

**Files:**
- Inspect: `tmp/preview/final/slide-01.png` ... `slide-30.png`
- Inspect: `tmp/layout/final/*.layout.json`
- Create: `tmp/qa/slide-review.txt`
- Modify: builder와 slide copy

**Interfaces:**
- Consumes: 1차 렌더링
- Produces: 제목 줄바꿈·본문 clipping·이미지 판독 문제를 수정한 deck

- [ ] montage에서 주차별 진행과 레이아웃 반복을 확인한다.
- [ ] 30장을 full-size로 보고 title, body, code, image crop, page marker를 PASS/repair로 기록한다.
- [ ] EXPLAIN 핵심 슬라이드에서 `type`, `key`, `rows`, `Extra`가 확대 없이 읽히는지 확인한다.
- [ ] 문제는 카피 축약 → 다른 원본 레이아웃 매핑 → 이미지 crop 조정 순서로 고친다.
- [ ] 모든 항목이 PASS가 될 때까지 재생성한다.

### Task 5: 구조·템플릿·최종 파일 검증

**Files:**
- Test: `outputs/phase01-mysql-interview-guide.pptx`
- Create: `tmp/qa/final-verification.txt`

**Interfaces:**
- Consumes: 시각 QA를 통과한 PPTX
- Produces: 최종 검증 기록

- [ ] `slides_test.py`로 캔버스 overflow를 검사한다.
- [ ] `check_template_fidelity.mjs`로 mapped edit 외 템플릿 drift를 검사한다.
- [ ] 최종 PPTX XML에서 빈 structural placeholder와 sample text를 검사한다.
- [ ] `render_slides.py`로 최종 PPTX를 독립 렌더링하고 30장을 다시 확인한다.
- [ ] `git status --short`로 `.obsidian/`가 손대지 않은 사용자 파일인지 확인한다.
