---
name: xagent
description: Runs other model families (via opencode on private-network vLLM) as sub-agents through the `xagent` CLI, for cross-review and light delegated tasks. Use when the user asks for a cross-review, second opinion, or another model's review ("교차검토", "교차 검토", "크로스 리뷰", "세컨드 오피니언", "다른 모델한테 리뷰", "second opinion", "cross-review"), and before committing SQL, schema, permission, or infrastructure changes. Also use when the user asks to delegate work to xagent / opencode / a cheaper model ("opencode한테 시켜", "위임해"), and to PROPOSE delegation (ask first, one line) when a task is a clearly bounded mechanical job such as the same edit across many files, boilerplate, or test scaffolding. Do not use it for 1–2 line edits or work that needs this conversation's context.
---

# xagent — 외부 에이전트 교차검토·위임

계약·종료 코드·로그 형식은 `~/xagent/README.md`에 있다. 모호하면 그 파일을 읽는다.

## 1. 프리셋 확인 (하드코딩 금지)

매번 먼저 실행한다:

```bash
xagent list
```

프리셋 이름·모델·모드는 `~/.config/xagent/presets.yaml`이 진실이다. 이 파일에 적힌 이름을 쓰지 말고 `list` 결과에서 고른다.
- 교차검토: `mode=review` 프리셋 **전부**
- 위임: `mode=task` 프리셋

## 2. 교차검토

diff 소스를 상황에 맞게 고른다.
- 브랜치에 커밋된 변경: `xagent review <review 프리셋들>`. 기본은 `main...HEAD`, 다른 기준이면 `--base REF`.
- 아직 커밋 안 한 변경("이 변경", 방금 고친 것): `xagent review --uncommitted <review 프리셋들>`
- 무엇을 리뷰할지 모르면 `git status --short`와 `git log --oneline main..HEAD`로 먼저 확인한다.

Bash 도구 `timeout`은 프리셋 `timeout_sec` 최대값보다 길게 준다(예: 600000).

출력 해석:
- `### 공통 지적`: 여러 모델이 같은 file:line을 지적한 항목. 우선 확인 대상.
- 프리셋 status가 `ok`가 아니면(`timeout`, `runner-error`, `empty-output`, `no-contract-output`) 그 모델의 결과는 **없는 것**이다. "문제 없음"으로 보고하지 말고 실패로 보고한다.
- `dropped=N`이 크면 로그(`log=` 경로)에서 버려진 원문을 확인할 수 있다.
- 종료 코드: 0 전원 ok / 1 high 있음 / 2 사용 오류 / 3 일부 실패 / 4 전원 실패.

## 3. 리뷰 결과 반영 규칙

- **그대로 반영하지 않는다.** 각 항목마다 해당 file:line을 직접 읽고 실제 결함인지 확인한 뒤, 타당한 것만 수정한다.
- 사용자에게 항목별로 한 줄씩 보고한다: `반영|file:line|이유` 또는 `기각|file:line|이유`.
- 외부 모델은 오탐이 많다. 코드로 확인되지 않은 지적은 기각하고 그 근거를 적는다.

## 4. 위임 (task) — 제안 후 위임

### 언제 제안하나
아래 "위임" 조건을 대부분 만족하고 "직접" 조건에 하나도 걸리지 않을 때만, **실행 전에 한 줄로 묻는다**:
`이 작업은 xagent(<task 프리셋>)에 맡길 수 있습니다 — 맡길까요?`
사용자가 명시적으로 위임을 요청했으면 묻지 않고 바로 실행한다. **묻지 않고 자동 위임하지 않는다.**

| 위임 | 직접 처리 |
|---|---|
| 직접 하면 도구 턴 6회 이상이 들 작업(여러 파일 반복 수정, 보일러플레이트, 테스트 뼈대) | 수정 20줄 이하 또는 파일 2개 이하 |
| 새로 써야 할 코드가 약 100줄 이상(스크립트로 몇 턴에 끝나더라도 출력 토큰이 주 비용) | 스크립트 한 번이면 끝나는 일괄 치환(출력이 짧음) |
| 대화 맥락 없이 지시 10줄로 완결됨 | 대상 파일을 이미 읽었음 |
| 테스트나 grep으로 결과를 기계 검증할 수 있음 | 도메인 규칙 판단 필요(CLAUDE.md 절대 규칙, 스키마, 권한, 실거래 로직) |
| 예상 diff 300줄 이하 | 검증 기준이 없음 |

### 실행
```bash
xagent <task 프리셋> "<대상 파일, 기대 결과, 하지 말 것, 검증 기준>"
```
- Bash 도구에 `run_in_background: true`로 띄우고 다른 일을 계속한다. 독립 작업은 병렬로 여러 개 띄워도 된다.
- 포그라운드로 돌릴 때는 Bash `timeout: 600000`을 준다(기본 120초면 도중에 죽는다).
- worker는 **셸이 없다**(테스트를 못 돌림). 지시에 "테스트는 호출자가 돌린다"를 전제로 쓴다.
- 메인 트리의 미커밋 tracked 변경은 worktree base에 스냅샷으로 들어간다. 미추적 새 파일은 안 들어가니, 필요하면 먼저 알린다.

### 끝나면 (매번 이 순서)
1. 출력의 `id`와 `path`를 확인한다. 변경이 없으면 worktree는 이미 자동 삭제됐다.
2. **diff를 직접 검토**: `git -C <path> diff HEAD~1`(worker 변경은 커밋으로 고정됨), 새 파일은 Read.
3. **테스트를 직접 돌린다**: worktree 안에서(예: `cd <path> && python3 run_tests.py`). 여기서 생긴 파일은 apply에 들어가지 않는다.
4. 판정해서 기록한다. 수락률 측정에 쓰이므로 정직하게 고른다.
   - 그대로 좋음: `xagent apply <id>`
   - 가져온 뒤 내가 고칠 것: `xagent apply <id> --modified --note "<무엇을>"`
   - 못 씀: `xagent discard <id> --note "<이유>"`
5. 사용자에게 한 줄로 보고한다: 무엇을 위임했고, 판정이 무엇이며, 왜 그런지.

apply는 작업 트리에 패치를 적용할 뿐이다. 커밋은 평소 규칙대로 사용자 확인 후에 한다.

### 운영
- 비대화형(`claude -p`) 세션에서는 `run_in_background`를 쓰지 않는다(포그라운드 + `timeout: 600000`). 세션이 끝나면 xagent가 중단된다(`killed`로 기록됨).
- 쌓인 worktree: `xagent gc`(변경 없는 것 정리), `xagent gc --all`(전부 폐기, 사용자 확인 후).
- `xagent stats`에 "자동 위임 전환 기준 충족"이 뜨면 사용자에게 **전환 여부를 묻기만 한다.** 스스로 바꾸지 않는다.

## 주의

- xagent 안에서 xagent를 부르지 않는다(`XAGENT_DEPTH` 가드로 거부된다).
- 프리셋은 사설망 GPU 팜 vLLM만 쓴다(opencode.ai 경유 외부 무료 모델 제외). 외부 모델 프리셋을 추가하는 건 사용자 결정이다.
