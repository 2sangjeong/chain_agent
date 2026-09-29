---
name: xagent
description: Cross-review and delegation to other model families through the `xagent` CLI (opencode on private-network vLLM — GLM, Qwen). Triggers — the user mentions opencode/OpenCode/오픈코드; asks for a cross-review or second opinion ("교차검토", "크로스 리뷰", "세컨드 오피니언", "다른 모델한테 리뷰", "second opinion"); or SQL/schema/permission/infra changes are about to be committed. Cross-review triggers always invoke. For an opencode TASK request, first apply this gate using only the request text (no tool calls) — do NOT invoke, do it yourself and say in one line why ("직접 처리가 더 싸서 opencode 없이 했습니다 — '무조건 opencode로'라고 하면 위임합니다"), when ANY holds — the exact change is already known or spelled out (instruction ≈ result); it is small (about ≤20 lines or ≤2 files); it needs this conversation's context or domain judgment (live-trading logic, CLAUDE.md rules); or checking the result means re-reading all of it. Invoke when it is read-heavy with a short answer (codebase survey, find-all-usages, big log/file scans) or short-instruction/long-output (many tests, boilerplate). If unsure, or the user insists ("무조건", "반드시", "그래도"), invoke. Questions about opencode itself (install, config, models) — answer without invoking.
---

# xagent

게이트(부를지 말지)는 description에서 끝났다. 여기 왔으면 실행한다. 계약·종료 코드는 `~/xagent/README.md`.
Bash는 포그라운드면 `timeout: 600000`. 비대화형(`claude -p`)에서는 `run_in_background`를 쓰지 않는다(세션 종료 시 `killed`).

## 교차검토
- 커밋된 변경: `xagent review --all` (기본 `main...HEAD`, 다른 기준은 `--base REF`)
- 미커밋 변경("이 변경"): `xagent review --all --uncommitted`
- `### 공통 지적`부터 본다. status가 `ok`가 아닌 모델의 결과는 **없는 것**이다. "문제 없음"으로 보고하지 않는다.
- **그대로 반영하지 않는다.** 항목마다 file:line을 직접 읽어 확인하고, `반영|file:line|이유` 또는 `기각|file:line|이유`로 한 줄씩 보고한다.

## 위임
```bash
xagent task "<대상 파일, 기대 결과, 하지 말 것, 검증 기준>"                # 기본 모델(GLM)
xagent task --model qwen "<...>"                                            # 모델 지정
```
- 대화형이면 `run_in_background: true`로 띄우고 다른 일을 계속한다.
- **같이 쓰기**(중요하거나 답이 갈릴 작업만): 위 두 줄을 한 메시지에서 병렬로 띄우고 결과를 비교한다. 나은 쪽만 apply, 다른 쪽은 `discard --note "compared: <채택 id>에 밀림 — <이유>"`. Claude가 diff를 두 번 읽어 비용이 두 배다.
- worker는 셸이 없다(테스트 못 돌림). 메인 트리의 미커밋 tracked 변경은 base로 넘어가지만, 미추적 새 파일은 넘어가지 않는다.

끝나면 이 순서로 한다.
1. 출력에서 `id`와 `path`를 확인한다. 변경이 없었으면 worktree는 이미 삭제됐다.
2. `git -C <path> diff HEAD~1`로 검토하고, 필요하면 worktree 안에서 테스트를 돌린다. 여기서 생긴 파일은 apply에 들어가지 않는다.
3. 판정해서 기록한다(수락률 측정용이니 정직하게).
   - 그대로 좋음: `xagent apply <id>`
   - 가져온 뒤 고칠 것: `xagent apply <id> --modified --note "<무엇>"`
   - 못 씀: `xagent discard <id> --note "<이유>"`
4. 무엇을 맡겼고, 판정이 무엇이며, 왜 그런지 한 줄로 보고한다. 커밋은 평소 규칙대로 사용자 확인 후에 한다.

## 운영
- 남은 worktree 정리: `xagent gc`. `xagent gc --all`은 사용자 확인 후에만.
- `xagent stats`의 프리셋별 수락률이 쌓이면, 기본 모델 변경은 사용자에게 제안만 한다.
- xagent 안에서 xagent를 부르지 않는다. 외부 모델 프리셋 추가는 사용자 결정이다.
