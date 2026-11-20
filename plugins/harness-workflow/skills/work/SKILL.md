---
name: work
description: "workspace/plan.md 의 다음 미완료 Task 1개를 TDD 사이클(test-coder → coder → code-review → verifier → simplify)로 자동 진행하고 plan.md 커밋 계획에 따라 정해진 커밋에 넣어 마무리하는 스킬. '/work', 'work', '다음 task', '다음 task 진행', '다음 task 진행해줘', '다음 작업', '다음 작업 시작', '다음 작업 진행', '다음 거 작업', '다음 거 진행', 'tdd 사이클', 'tdd 진행', 'tdd 한 번 돌려', 'red green refactor', '한 사이클 돌려', '다음 단계 진행', 'plan 다음 task', 'workspace 다음 task', 'task 이어서 진행', 'phase x task y 진행', 'task 1.x 진행' 등 workspace/plan.md 의 Task 를 이어서 진행하려는 모든 표현에 반드시 사용한다."
---

# /work — Task 1개 TDD 사이클 실행

`workspace/plan.md` 의 **다음 미완료 Task 1개** 를 자동 선택하고
test-coder → coder → code-review → verifier → simplify 순으로 진행한 뒤
변경분을 plan.md `# Commits` 에서 이 Task 가 속한 커밋에 넣어 마무리한다 (커밋 규칙은 아래 "커밋 규칙" 절).

## 전제 조건

- 현재 작업 디렉토리가 대상 프로젝트 루트
- `workspace/plan.md` 가 존재 (없으면 `/plan` 부터 실행 안내)
- 프로젝트 루트 `CLAUDE.md` 에 빌드 명령 라인 존재 (예: `` 본 프로젝트 빌드 명령: `./gradlew test` ``)
- `git` 작업 트리가 깨끗하거나, 현재 진행 중인 Task 의 변경분만 남아있어야 함
- 작업은 `main`/`master` 가 아닌 별도 작업 브랜치에서 진행 (Step 0 에서 보장)

## 워크플로우

### Step 0: 브랜치 점검 (main 보호)

작업은 **절대 `main` 브랜치에서 직접 진행하지 않는다.**

1. 현재 브랜치 확인:
   ```bash
   git rev-parse --abbrev-ref HEAD
   ```

2. 분기 처리:
   - **현재 브랜치가 `main` (또는 `master`)인 경우**:
     - 사용자에게 새 작업 브랜치명을 물어본다:
       ```
       현재 main 브랜치입니다. 작업용 브랜치를 새로 만들겠습니다.
       브랜치명을 입력하세요 (예: BA-2291 — 티켓 번호):
       ```
     - 입력받은 이름으로 새 브랜치를 만들고 checkout:
       ```bash
       git checkout -b {입력받은 브랜치명}
       ```
     - checkout 성공 후 다음 단계로 진행.
   - **현재 브랜치가 `main`/`master`가 아닌 다른 브랜치인 경우**:
     - 사용자에게 1회 확인:
       ```
       현재 브랜치: {브랜치명}
       이 브랜치에서 작업을 진행할까요? (y/N)
       ```
     - `y` → 그대로 다음 단계 진행.
     - `N` → 기존 로컬 브랜치 목록을 보여주고, 맨 마지막에 직접 입력 옵션을 함께 제시:
       ```bash
       git branch --format='%(refname:short)'
       ```
       위 목록을 번호와 함께 출력하고, 마지막 항목으로 `직접 입력 (새 브랜치 생성)` 을 추가:
       ```
       작업할 브랜치를 선택하세요:
         1) {기존 브랜치 A}
         2) {기존 브랜치 B}
         ...
         N) 직접 입력 (새 브랜치 생성)
       ```
       - 기존 브랜치 선택 → `git checkout {선택 브랜치}` 후 진행.
       - `직접 입력` 선택 → 새 브랜치명을 물어보고 `git checkout -b {입력값}` 후 진행.

3. 브랜치 확정 전에는 어떤 agent 호출도, 커밋도 하지 않는다.

### Step 1: 사전 점검

1. `workspace/plan.md` 존재 확인. 없으면:
   ```
   ❌ workspace/plan.md 가 없습니다.
   먼저 /plan 으로 작업 계획을 작성하세요.
   ```
   라고 보고하고 종료.

2. `git status --short` 으로 현재 작업 트리 상태 확인.
   - 깨끗하면 OK.
   - 변경분이 있으면 사용자에게 표시하고 `이 변경분을 그대로 안고 다음 Task 를 진행할까요?` 1회 확인.

3. 빌드 명령 추출. 프로젝트 루트 `CLAUDE.md` 에서 다음 패턴을 grep:
   ```
   본 프로젝트 빌드 명령:\s*`([^`]+)`
   ```
   - 매칭되면 그 명령을 `$BUILD_CMD` 로 사용.
   - 매칭 안되면 사용자에게 1회 확인:
     `CLAUDE.md 에 빌드 명령 라인이 없습니다. 어떤 명령을 사용할까요? (기본: ./gradlew test)`

4. `workspace/plan.md` 에서 `ticket` 과 `# Commits` 목록(C1, C2, ...) 추출.
   없으면(예전 plan.md) 사용자에게 커밋 목록을 제안하고 확인받아 plan.md 에 추가한다 (추측 금지).

### Step 2: 다음 Task 선정

`workspace/plan.md` 를 읽고:

- 가장 위쪽의 **`- [ ] Task X.Y`** 라인을 찾는다.
- 해당 Task 가 속한 Phase 번호와 Task 설명을 추출한다.
- 모든 Task 가 완료(`- [x]`)면:
  ```
  ✅ 모든 Task 가 완료되었습니다.
  /close 로 archive 진행을 권장합니다.
  ```
  종료.

확인 출력:
```
다음 Task: Phase {N} / Task {N.M}
설명: {Task 설명}

이 Task 를 진행할까요? (y/N)
```
사용자 승인 후 진행.

### Step 3: Red 단계 (test-coder)

1. `test-coder` agent 를 호출. 프롬프트에 다음을 포함:
   - 대상 Task 의 Phase / Task 번호 + 설명
   - `workspace/plan.md` 경로
   - 프로젝트 `CLAUDE.md` 경로
   - "build/test 명령은 직접 실행하지 말 것" 명시

2. agent 완료 후 자동 빌드 (redirect+tail):
   ```bash
   $BUILD_CMD > workspace/.last_build.log 2>&1; tail -25 workspace/.last_build.log
   ```
   - `workspace/.last_build.log` 에 전체 출력 저장
   - `tail -25` 결과만 context 에 적재

3. 결과 판정:
   - **BUILD FAILED + 테스트 실패**: Red 정상 → 다음 단계 진행
   - **BUILD SUCCESSFUL**: 테스트가 실패하지 않음 → 테스트가 의미 있는지 사용자 확인 요청 후 일시 중단
   - **컴파일 실패**: 스켈레톤 누락 가능성 → test-coder 에게 보강 요청 1회 재호출, 그래도 실패 시 중단

### Step 4: (커밋 없음)

Red 단계의 테스트·스켈레톤 변경분은 커밋하지 않고 워킹트리에 둔다. 커밋은 Task 끝(Step 8.5)에 한다.

### Step 5: Green 단계 (coder)

1. `coder` agent 호출. 프롬프트에 동일 컨텍스트 + Step 3 의 테스트 코드 위치 전달.

2. agent 완료 후 자동 빌드 (Step 3 와 동일 패턴).

3. 결과 판정:
   - **BUILD SUCCESSFUL**: Green → 다음 단계 진행
   - **BUILD FAILED**:
     ```
     ❌ Green 실패.
     로그: workspace/.last_build.log
     원인 파악 후 다시 /work 호출하거나, coder 에게 직접 수정 요청하세요.
     ```
     중단.

### Step 5.5: code-review (커밋 전 버그 게이트)

Green 빌드 성공 직후, **커밋하기 전**(워킹트리에 구현 diff 가 남아있는 상태)에서 내장 `code-review` skill 을 호출한다.

- 옵션: `--fix` / `--comment` **둘 다 사용하지 않는다** (read-only 리뷰만). effort 는 **low 또는 medium** (high 는 불확실 finding 이 많아 게이트가 시끄러워짐).
- 목적: 방금 짠 구현 diff 의 correctness 버그 탐지. 정리(cleanup) finding 은 Step 8 `simplify` 와 겹치므로 **버그성 finding 위주로** 판단한다.
- 결과 판정:
  - **심각 버그 없음**: 다음 단계(Step 7)로 진행.
  - **심각 버그 발견**:
    ```
    ⚠️ code-review 가 버그를 지적했습니다.
    지적 사항: {요약}
    → coder 재호출로 수정 / 수동 수정 / 무시하고 진행 중 선택하세요.
    ```
    커밋하지 않고 **중단**, 사용자 결정을 받는다. 수정 후 빌드가 다시 Green 이면 이 단계를 재실행한다.
- `code-review` 가 가용하지 않으면 한 줄 보고 후 스킵하고 Step 7 로 진행한다.

### Step 7: Verifier

`verifier` agent 호출. 결과:
- **종합 판정 PASS**: 다음 단계 진행
- **종합 판정 FAIL**:
  ```
  ❌ Verifier FAIL.
  지적 사항을 정리해 사용자에게 의사결정 요청.
  ```
  중단. 사용자가 coder 재호출 / 수동 수정 / 그대로 진행 결정.

### Step 8: Refactor 단계 (선택)

1. Claude Code 내장 `simplify` skill 을 호출한다.
   - `simplify` 는 변경된 코드의 중복/단순화/효율/altitude 정리만 수행하고 **버그 헌팅은 하지 않는다** (정확성 검증은 Step 7 Verifier 가 이미 담당).
   - 동작을 바꾸지 않는 구조 개선이 목적이므로 TDD Refactor 단계와 정확히 일치한다.
   - 내장 skill 이라 별도 설치가 필요 없다. 혹시 가용하지 않으면:
     ```
     ⏭️ simplify 미가용 — refactor 단계 스킵.
     ```
     Step 8.5 로.

2. 호출 후 변경사항이 있으면 자동 빌드 (Step 3 와 동일 패턴).
   - **BUILD SUCCESSFUL**: Step 8.5 로 (refactor 변경분도 같은 커밋에 들어간다)
   - **BUILD FAILED**:
     ```
     ❌ Refactor 후 빌드 실패. 로그: workspace/.last_build.log
     ```
     중단. 사용자가 결정.
   - **변경사항 없음**: Step 8.5 로.

### Step 8.5: 커밋 (plan 의 커밋 계획대로)

이번 Task 의 변경분(테스트·구현·refactor)을 plan.md 에서 이 Task 에 적힌 커밋 `C{n}` 에 넣는다.

1. 이 브랜치에 `C{n}` 메시지와 같은 커밋이 이미 있는지 확인:
   ```bash
   git log --format='%H %s' {base}..HEAD
   ```
   `{base}` 는 분기한 브랜치(보통 `origin/main`). 모르면 사용자에게 확인.
2. 분기:
   - **`C{n}` 커밋이 없다** → 새 커밋:
     ```bash
     git add {이번 Task 변경 파일들}
     git commit -m "{C{n} 메시지}"
     ```
   - **`C{n}` 커밋이 HEAD 이고 아직 push 안 됐다** (`git status -sb` 에 upstream 이 없거나 ahead 상태) → 그 커밋에 합친다:
     ```bash
     git add {이번 Task 변경 파일들}
     git commit --amend --no-edit
     ```
   - **`C{n}` 커밋이 HEAD 가 아니거나 이미 push 됐다** → amend·rebase·force push 하지 않는다. 같은 메시지로 새 커밋을 만들고, 완료 보고에 "MR 전에 커밋 계획대로 정리 필요"를 적는다. 정리(squash·force push)는 사용자가 지시할 때만 한다.
3. 진행 중에 커밋 계획을 바꿔야 할 것 같으면(성격이 다른 변경이 새로 생김 등) 커밋하기 전에 사용자에게 묻고 plan.md `# Commits` 를 고친다. 혼자 커밋을 늘리지 않는다.

### Step 9: plan.md Task 체크 + handoff.md 갱신

1. `workspace/plan.md` 에서 해당 Task 라인을 `- [ ]` → `- [x]` 로 변경.

2. `workspace/handoff.md` 갱신:
   - `마지막 수정일` 현재 시각으로 변경
   - `완료 Phase` / `진행 Phase` / `다음 Phase` 업데이트
   - `commit log` 섹션 갱신 (amend 하면 hash 가 바뀌므로 최신 것으로 덮어쓴다):
     ```
     - C{n} {hash} {메시지} — Task {N.M}, ... 포함
     ```

3. 변경된 plan.md / handoff.md 는 별도 커밋하지 않는다 (다음 `/work` 또는 `/close` 호출 시 함께 처리).
   단, plan.md / handoff.md 만 단독 변경된 상태가 누적되지 않도록 사용자에게 짧게 보고.

### Step 10: 완료 보고

```
✅ Phase {N} / Task {N.M} 완료

커밋: C{n} {hash} {메시지}  (새 커밋 / amend / 정리 필요 중 하나)

plan.md: Task {N.M} 체크 완료
handoff.md: commit log 갱신

남은 Task: {M개}

다음 단계:
  - 이어서 진행: /work 다시 호출
  - 모두 끝났으면: /close
```

## 주의사항

- **빌드 결과 판단은 tail -25 만 본다**: 더 자세한 분석이 필요하면 사용자에게 `workspace/.last_build.log` 직접 확인 요청.
- **agent 안에서 빌드/테스트 명령을 직접 실행하지 못하도록** test-coder/coder 호출 프롬프트에 명시 (이미 agent 정의에 적혀 있지만 한 번 더 강조).
- **커밋 규칙**
  - 형식: `{type}: {한국어 설명} [{ticket}]` 한 줄. 필요하면 `fix(otc): ...` 처럼 scope 를 붙이고, 본문은 필요할 때만 불릿으로.
  - 커밋은 최대한 적게, 그래도 무슨 작업인지 구분되게. 기본 티켓당 1개, 성격이 확실히 다른 변경이 섞일 때만 2~3개. 단위는 plan.md `# Commits` 가 정한다.
  - `[phase-N]` `[task-N.M]` 태그, commit-name prefix 를 쓰지 않는다.
  - `Co-Authored-By` 등 attribution trailer 를 넣지 않는다.
  - Task·Phase 별로 커밋을 나누지 않는다.
  - 메시지는 plan.md `# Commits` 의 것을 그대로 쓴다. 추측 금지.
- **`workspace/.last_build.log` 는 `.gitignore` 에 포함된 `workspace/` 하위라서 별도 ignore 불필요**.
- **PreToolUse ask hook 이 git commit 마다 뜸 (amend 포함)**: 사용자가 거부하면 즉시 중단한다. 거부된 커밋을 다른 방식으로 우회하지 않고, push 된 커밋을 amend·force push 하지 않는다.
- **Task 1개 단위로 종료한다**: Phase 전체 자동 진행 금지. 매 호출마다 다음 Task 1개씩.
- **Verifier FAIL 시 자동 수정 금지**: 반드시 사용자 결정.
- **`main`/`master` 직접 작업 금지**: Step 0 에서 브랜치를 확정하기 전에는 agent 호출/커밋을 시작하지 않는다. main 이면 새 브랜치명을 물어 `checkout -b`, 다른 브랜치면 진행 여부를 1회 확인한다.
