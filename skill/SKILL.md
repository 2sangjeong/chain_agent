---
name: xagent
description: Runs other model families (via opencode on private-network vLLM) as sub-agents through the `xagent` CLI, for cross-review and light delegated tasks. Use when the user asks for a cross-review, second opinion, or another model's review ("교차검토", "교차 검토", "크로스 리뷰", "세컨드 오피니언", "다른 모델한테 리뷰", "second opinion", "cross-review"), and before committing SQL, schema, permission, or infrastructure changes. Also use when the user explicitly asks to delegate a task to xagent / opencode / a cheaper model. Do not use it otherwise.
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

## 4. 위임 (task)

```bash
xagent <task 프리셋> "<구체적 지시: 대상 파일, 기대 결과, 검증 방법>"
```

- `worktree: true` 프리셋은 `../wt-xagent-<preset>-<ts>` worktree와 `xagent/<preset>-<ts>` 브랜치에서 돈다. 메인 트리는 건드리지 않는다.
- 결과를 가져오기 전에 **반드시 worktree의 diff를 직접 검토한다**: `git -C <worktree> diff`, 새 파일은 Read로 연다.
- 검토를 통과한 것만 가져온다. 방법은 커밋 후 cherry-pick이나 patch 적용이며, 사용자 확인을 받은 뒤에 한다. merge는 xagent가 하지 않는다.
- 끝나면 출력된 `정리:` 명령으로 worktree와 브랜치를 지운다(사용자가 남기길 원하지 않는 한).

## 주의

- xagent 안에서 xagent를 부르지 않는다(`XAGENT_DEPTH` 가드로 거부된다).
- 프리셋은 사설망 GPU 팜 vLLM만 쓴다(opencode.ai 경유 외부 무료 모델 제외). 외부 모델 프리셋을 추가하는 건 사용자 결정이다.
