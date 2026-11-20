---
name: close
description: "workspace/ 의 작업이 완료되었을 때 plan.md + handoff.md 를 archive/ 로 이동하고 brain(개인 지식베이스)에 작업 기록을 남기는 스킬.
  '작업 완료', '클로즈', 'close', '아카이브해줘', '다 끝났어' 등을 요청할 때 사용한다."
---

# /close — 작업 완료 + archive 이동

`workspace/plan.md` + `workspace/handoff.md` 를 `workspace/archive/{N}_{slug}/` 로 이동하고
brain 이 있으면 brain 규칙대로 작업 기록을 남긴다.

## 전제 조건

- 현재 작업 디렉토리가 대상 프로젝트 루트
- `workspace/plan.md` 가 존재

## 워크플로우

### Step 1: 완료 여부 확인

`workspace/plan.md` 의 모든 Task 체크박스(`- [x]`)를 확인한다.
미완료 Task(`- [ ]`) 가 있으면:
→ "아직 완료되지 않은 Task {N}건이 있습니다. 그래도 close 할까요?" 확인

### Step 2: archive 번호 결정

`workspace/archive/` 의 기존 폴더 수를 확인해 다음 번호를 결정한다.
- 비어있으면 `1_`
- `1_*` 있으면 `2_`
- N개 있으면 `{N+1}_`

### Step 3: slug 확인

`workspace/plan.md` 의 `project name` 필드에서 slug를 가져온다.
없으면 사용자에게 확인.

### Step 4: handoff 최종 업데이트

**이동하기 전에** `handoff.md` 의 `마지막 수정일` 과 `완료 Phase` 를 먼저 업데이트한다.
(이동 후에는 archive 안의 파일을 건드리지 않는 불변 원칙 때문에, 갱신은 반드시 이동 전에 끝낸다.)

### Step 5: archive 이동

`workspace/archive/{N}_{slug}/` 폴더 생성 후
`workspace/plan.md`, `workspace/handoff.md` 를 해당 폴더로 이동.

### Step 6: brain 기록

brain = 사용자의 개인 지식베이스(LLM wiki). 위치는 아래 순서로 찾는다.
1. 로드된 CLAUDE.md 에 brain 위치가 적혀 있으면 그 경로
2. 없으면 프로젝트 루트와 그 상위 폴더에서 `brain/` 디렉토리 탐색
3. 둘 다 없으면 이 Step 을 건너뛰고 Step 7 에서 "brain 없음" 으로 보고

brain 을 찾았으면 안내만 하지 말고 **실제로 기록한다**.
1. brain 의 `CLAUDE.md` 와 위키 폴더의 `CLAUDE.md` 를 읽는다. 폴더 구조·frontmatter·index/log 갱신·작성 방식은 모두 이 규칙을 따른다 (이 스킬에 경로나 형식을 따로 정하지 않는다)
2. 소스는 방금 이동한 `workspace/archive/{N}_{slug}/plan.md`·`handoff.md` 와 이번 작업의 커밋·MR 내용이다
3. 같은 프로젝트 페이지가 이미 있으면 새로 만들지 않고 그 페이지에 이번 작업을 추가한다
4. 규칙이 요구하는 index·log 갱신까지 끝낸다

### Step 7: 결과 보고

```
✅ close 완료

archive: workspace/archive/{N}_{slug}/
brain 기록: {갱신·생성한 페이지 경로 목록 | brain 없음 → 건너뜀}

다음 작업 시 /plan 으로 새로 시작하세요.
```

## 주의사항

- archive 이동 전 반드시 사용자 최종 확인
- `workspace/archive/` 안의 파일은 수정하지 않는다 (불변 원칙)
