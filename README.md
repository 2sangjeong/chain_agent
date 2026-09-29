# xagent

Claude Code에서 외부 코딩 에이전트를 서브에이전트로 호출하는 래퍼. 용도는 (1) 다른 모델 계열의 교차검토, (2) 가벼운 위임 작업.
현재 runner는 **opencode만** 지원한다(codex: API 키 401, kiro-cli: 미설치 — 2026-09-29 실측).

| 경로 | 실체 |
|---|---|
| `~/.config/xagent/presets.yaml` | → `~/xagent/presets.yaml` (프리셋·agent 정의 단일 소스) |
| `~/bin/xagent` | → `~/xagent/bin/xagent` |
| `~/.claude/skills/xagent/` | → `~/xagent/skill/` |
| `~/.xagent/logs/` | 실행 로그 (repo 밖) |
| `~/.xagent/tasks.jsonl` | task 실행·결과 기록 (`xagent stats`의 원천) |

## CLI

```
xagent list                                        # 프리셋 목록 (YAML에서 읽음)
xagent <preset> ["<prompt>"]                       # 단일 실행
xagent review [--base REF | --uncommitted | --diff-file PATH|-] <preset> [<preset>...]
xagent apply <id|worktree경로> [--modified] [--note TEXT]   # worktree 변경을 메인 트리에 적용 + 기록 + 정리
xagent discard <id|worktree경로> [--note TEXT]              # 버림 + 기록 + 정리
xagent gc [--all]                                  # 현재 repo의 xagent worktree 정리
xagent stats                                       # task 수락률·토큰·소요시간 집계
```

`<id>`는 `<preset>-<YYYYmmdd-HHMMSS>-<rand4>`. worktree는 `../wt-xagent-<id>`, 브랜치는 `xagent/<id>`.
`apply|discard|gc`는 대상 repo(메인 작업 트리) 안에서 실행한다.

### 입력

- **단일 실행**: 프롬프트는 인자. stdin은 **FIFO(파이프)나 일반 파일(리다이렉트)일 때만** 읽고, 인자 뒤에 `<stdin>` 블록으로 붙인다.
  인자가 없으면 stdin만 프롬프트가 된다. 둘 다 없으면 사용 오류.
  TTY·소켓·`/dev/null` stdin은 읽지 않는다 — Claude Code Bash 도구의 stdin은 닫히지 않는 소켓이라 무조건 읽으면 영원히 멈춘다(실측).
- **review**: diff 소스는 셋 중 하나.
  - 기본 `git diff <base>...HEAD`, `--base` 기본값 `main`.
  - `--uncommitted`: `git diff HEAD`(스테이지+미스테이지) + 미추적 파일(`--exclude-standard`)을 새 파일 diff로 추가. 인덱스는 건드리지 않는다.
  - `--diff-file PATH`: 파일에서 읽는다. `-`는 stdin.

  어느 소스든 `secret_patterns`에 걸리는 파일 구간은 diff에서 빼고 stderr에 알린다. 미추적 `.env`가 `--uncommitted` diff로 프롬프트에 들어간 적이 있다(실측).

  diff가 비면 모델을 부르지 않고 사용 오류로 끝낸다.
- 러너에게는 항상 프롬프트를 **stdin으로** 넘긴다(argv 128KB 한도 회피, 200KB 실측). 입력이 없으면 `/dev/null`을 준다.

### review 모드

- 대상은 `mode: review` 프리셋만. 여러 개면 **병렬** 실행.
- 읽기 전용 강제: `permissions: read-only` 프리셋은 모드와 무관하게 실행 시 `edit/bash/webfetch/task/external_directory: deny`를 덮어써 주입한다. review 모드에서 `permissions`가 `read-only`가 아니면 설정 오류.
- 작업 디렉터리는 git toplevel. reviewer는 read/grep/glob으로 주변 코드를 읽을 수 있다(비밀 파일 제외, 아래 "격리").
- 리뷰 출력 계약(아래)을 프롬프트에 자동 삽입한다. diff 줄 앞에는 **새 파일(HEAD 쪽) 줄 번호**를 붙여 넘긴다.
- 계약 형식이 아닌 줄은 걸러낸다. 원문은 로그에만 남긴다. 걸러진 줄 수는 결과 헤더에 `dropped=N`으로 표시한다.
- 출력 순서:
  1. `### 공통 지적`: 2개 이상 프리셋이 **같은 file:line**을 지적한 항목. severity는 가장 높은 것, 뒤에 `[프리셋들]`.
  2. 프리셋별 섹션: 헤더 `### <preset>  <status>  <초>s  kept=N dropped=N  log=<경로>` 다음에 계약 줄들.

### task 모드

- `worktree: true`면 git toplevel에서 `git worktree add ../wt-xagent-<id> -b xagent/<id>`(HEAD 기준)를 만들고, 그 안에서 실행한다.
  - `<id>`에 4자리 난수를 붙여 같은 초에 병렬로 띄워도 충돌하지 않는다.
  - **base 스냅샷**: 메인 트리에 커밋 안 된 **tracked** 변경(`git diff HEAD`)이 있으면 worktree에 적용해 `xagent: base snapshot` 커밋으로 만든다. worker는 현재 상태를 보고 작업하고, 이후 diff에는 worker 변경만 남는다. 미추적 파일은 넘어가지 않는다.
  - 종료 후 worktree 안에서 `git add -N`(worktree 인덱스만)을 한 뒤 `git diff --stat`(최대 40줄), id, worktree 경로를 출력한다.
  - **변경이 없으면 worktree와 브랜치를 자동 삭제**하고 결과를 `empty`로 기록한다. 변경이 있으면 남겨 둔다.
  - **merge는 하지 않는다.** 가져오기는 `xagent apply`, 버리기는 `xagent discard`.
- worker는 셸이 없다. 테스트 실행은 호출자가 worktree에서 한다. xagent는 이 사실을 task 프롬프트 앞에 붙인다.
- `worktree: false`면 현재 디렉터리에서 실행한다(메인 트리가 바뀔 수 있음).
- stdout에는 에이전트의 마지막 텍스트 응답을 출력하되 **뒤쪽 4000자까지만**. 넘으면 생략 표시를 하고 전문은 로그에 둔다.

### apply / discard / gc / stats

- `apply`:
  1. worktree 변경(`git diff --binary`, base 스냅샷 이후분)을 메인 toplevel에 `git apply`한다(`--check` 선행).
  2. 성공하면 결과를 `accepted`(`--modified`면 `modified`)로 기록하고 worktree와 브랜치를 지운다.
  3. 적용이 실패하면 아무것도 지우지 않고 종료 코드 2.
- `discard`: `discarded`로 기록하고 지운다.
- `gc`:
  - 변경 없는 xagent worktree를 지운다.
  - `--all`이면 변경 있는 것도 지우고, 결과가 없던 건 `discarded`(note=gc)로 기록한다.
  - worktree 없는 `xagent/*` 브랜치도 지운다.
  - 6시간 넘은 `$TMPDIR/xagent-db-*` 임시 DB(SIGKILL 잔여물)도 지운다.
- `stats`:
  - 프리셋별 실행 수, status, 결과 분포, 토큰 합, 소요 중앙값을 보여준다.
  - **수락률** = (accepted+modified) / (accepted+modified+discarded+empty).

`tasks.jsonl` 형식(한 줄 한 이벤트):

```
{"event":"run", "id", "at", "preset", "model", "repo", "worktree", "branch", "base", "status", "elapsed", "tokens":{input,output,reasoning,cache_read,cache_write}, "files_changed"}
{"event":"outcome", "id", "at", "outcome": "accepted|modified|discarded|empty", "note"}
```

### 격리

- **비밀 파일**: `presets.yaml`의 `secret_patterns`를 모든 agent의 `read` 권한에 deny로 강제한다(`secret_allow`는 예외 허용).
  - grep 도구는 read 규칙을 따르지 않는다. `.gitignore`는 존중하지만, 모델이 `include` 글롭을 주면 ripgrep `--glob`가 ignore 규칙을 덮어써 gitignore된 `.env`도 읽힌다(실측).
  - 그래서 실행 전 작업 트리(`.git` 제외)를 훑어 비밀 패턴 파일이 **하나라도 있으면**(ignore 여부 무관) 그 실행에서 grep을 deny하고 stderr와 로그에 알린다.
  - 이 경우 모델은 glob·read로만 탐색한다. crypto-bot처럼 `.env`·`*.db`가 있는 repo에서는 grep이 항상 꺼진다.
- **하위 에이전트 금지**: 모든 agent에 `task: deny`, `webfetch: deny`, `external_directory: deny`를 강제한다. `task`가 열리면 모델이 제한을 물려받지 않는 하위 에이전트를 띄워 `.env`·DB를 읽었다(실측).
- **worker 셸 금지**: `bash: deny`.
- **환경변수 허용 목록**: 러너에는 다음만 넘긴다.
  - `PATH HOME USER LOGNAME LANG LC_* TMPDIR XDG_*_HOME XDG_RUNTIME_DIR` 및 프록시 변수
  - xagent 고정값
  - 나머지(API 키 등)는 넘기지 않는다.

### 러너 호출 (opencode, 실측 근거)

```
opencode run --pure --format json --dir <cwd> --agent <agent> -m <model>   < prompt
env(허용 목록 + 아래 고정값만): PWD=<cwd>  OPENCODE_DB=<실행별 임시 DB>  OPENCODE_CONFIG_CONTENT=<agent 정의 JSON>  OPENCODE_DISABLE_PROJECT_CONFIG=1
     OPENCODE_DISABLE_CLAUDE_CODE_SKILLS=1  OPENCODE_DISABLE_AUTOUPDATE=1  XAGENT_DEPTH=1
```

- `PWD`와 `--dir`를 실행 디렉터리로 맞춘다: opencode는 실제 cwd가 아니라 `$PWD`로 프로젝트 디렉터리를 정한다. 안 맞추면 worktree task가 호출한 셸의 메인 트리를 수정한다(2026-09-29 Phase 3 실측 사고).
- `OPENCODE_DB`는 실행마다 임시 파일로 주고 끝나면 지운다. 공유 `opencode.db`는 병렬 기동 시 `database is locked`로 간헐 실패했다(Phase 3 #6 실측). 세션은 opencode UI에 남지 않으며, 원문은 xagent 로그에 있다.
- `--agent` 항상 명시: 생략 시 플러그인 기본 agent(Sisyphus, `*: allow`)로 떨어진다.
- `--pure`: 없으면 oh-my-openagent 플러그인이 cwd에 `.omo/`를 쓴다(reviewer도).
- `--format json`: 최종 답은 `type=="text"` 이벤트의 `part.text`. 기본 포맷은 도구 에러로 끝난 턴에서 빈 출력이 나온 적이 있다.
- 프로젝트 `opencode.json`·Claude 스킬 로딩을 끈다(재현성, 재귀 방지). `XAGENT_DEPTH`가 이미 설정된 환경에서 xagent는 실행을 거부한다.

### 상태·종료 코드

프리셋별 status:

| status | 조건 |
|---|---|
| `ok` | 정상 |
| `timeout` | `timeout_sec` 초과 → 프로세스 그룹 SIGTERM, 5초 후 SIGKILL |
| `runner-error` | rc≠0 또는 `type=="error"` 이벤트 (없는 모델 rc=1/1s, 죽은 서버 rc=1/약 64s 재시도 후 — 실측) |
| `empty-output` | 텍스트 응답 0 |
| `no-contract-output` | review에서 계약 줄 0 — **LGTM으로 간주하지 않는다** |
| `killed` | xagent가 SIGTERM/SIGHUP/SIGINT를 받음 (task 기록에만 나타남) |

| 종료 코드 | 의미 |
|---|---|
| 0 | 모든 프리셋 ok, `high` 없음 |
| 1 | 모든 프리셋 ok, `high` 1건 이상 (review) |
| 2 | 사용·설정 오류 (아무것도 실행 안 함: 없는 프리셋, YAML 오류, 빈 diff, git repo 아님, 재귀) |
| 3 | 일부 프리셋 실패 |
| 4 | 모든 프리셋 실패 |

우선순위: 2 > 4 > 3 > 1 > 0. `apply|discard|gc|stats`는 0 성공 / 2 오류.

**시그널로 중단되면** 실행 중인 러너 프로세스 그룹을 종료하고 임시 DB를 지운다. task는 `status=killed`로 기록하고, 변경이 없는 worktree는 지운다(`empty`). 이후 종료 코드는 143이다. `claude -p`처럼 세션이 끝나면 백그라운드 xagent가 SIGTERM으로 죽는다(실측). SIGKILL이면 이 처리가 돌지 못하므로 `gc`가 뒷정리한다.
 타임아웃은 해당 프리셋만 실패시키고 나머지는 끝까지 기다린다. `timeout_sec`는 러너 프로세스에만 적용된다(worktree 생성 제외).

### 로그

`~/.xagent/logs/<YYYYmmdd-HHMMSS>-<preset>.log`. 이름이 겹치면 `.N`을 붙인다.

기록 항목:
- argv, cwd
- 주입 설정
- 전체 프롬프트
- 러너 stdout(JSON 이벤트) 원문과 stderr
- status, 소요 시간
- 토큰 사용량(`step_finish` 이벤트 합)
- grep 차단 여부와 사유
- 계약 필터 결과(kept/dropped)

결과 헤더에도 `tok=<input>/<output>`을 표시한다.

## 리뷰 출력 계약

```
severity|file:line|이유
```

- 한 줄에 한 건. `severity` ∈ `high|med|low`.
- `file`은 repo 루트 기준 경로, `line`은 새 파일 기준 정수. 범위 `N-M`도 받지만 합의 판정 키는 시작 줄 `N`이다.
- 문제가 없으면 정확히 `LGTM` 한 줄.
- 파서는 앞뒤 공백, 줄머리 `- `/`* `/번호 목록, 감싼 백틱만 벗긴 뒤 엄격히 매칭한다. `medium`·`critical` 같은 다른 severity 이름은 버린다.
- 계약 줄과 `LGTM`이 섞이면 `LGTM`을 버린다.
- 파일 경로의 `./`, `a/`, `b/` 접두사는 정규화한다.
