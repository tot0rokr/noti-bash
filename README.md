# noti — Discord Webhook 통합 Bash 알림

작업 결과를 **Discord Webhook**으로 보내는 단일 바이너리(스크립트).  
텍스트/임베드/파일 업로드/명령 실행 결과 요약을 전송하고, **프리셋(.noti.preset)** 으로 커맨드를 짧게 별칭처럼 쓸 수 있어.

---

## 특징 (Features)

- `send` 텍스트, `embed` 카드, `file` 업로드, `run` 실행/요약 전송 지원
- **프리셋(.noti.preset)**: `noti <이름> <args...>` 로 재사용 (git alias 느낌, `$1 $2 $@` 자리표시자)
- 성공/실패/소요시간/시작/종료 시각 자동 포함 (run)
- 로그 파일 자동 첨부 옵션(실패시에만/항상)
- **멘션 차단 기본값**, `--allow-mentions`로 필요한 경우만 허용
- **429 rate limit** 자동 재시도, **5xx** 지수 백오프
- **디버그 모드**(마스킹된 URL, 페이로드 프리뷰)
- Linux/macOS/WSL에서 동작(표준 `bash`, `curl` 필요), `jq` 설치 시 임베드 품질↑

---

## 요구사항

- **bash 4+** (연관 배열 사용), `curl` 필수
- 선택: `jq` (JSON 생성/임베드/필드 처리 안정)

---

## 설치

```bash
# noti 파일을 PATH에 둔다 (예: /usr/local/bin)
sudo install -m 0755 noti /usr/local/bin/noti

# 실행 확인
noti help
```

---

## 빠른 시작 (Quick Start)

```bash
# 1) Webhook 설정 (환경변수 or .noti.conf 중 하나)
export NOTI_WEBHOOK='https://discord.com/api/webhooks/.../...'

# 2) 간단 메시지
noti send "hello from noti 👋"

# 3) 임베드(색/필드)
noti embed --title "Deploy" --desc "✅ Success" --color 3066993   --field Service api --field Version v1.2.3

# 4) 명령 실행 후 결과 전송 (실패 시 로그 첨부)
noti run --title "CI: test" --attach-log-on-fail -- bash -c 'echo ok; exit 1'
```

---

## 설정 (Configuration)

### 1) Webhook 제공 방식 (우선순위)

1. 환경변수 **`NOTI_WEBHOOK`**
2. 설정 파일 **`.noti.conf`**
   - 탐색 순서: `$NOTI_CONFIG` → `./.noti.conf` → `~/.noti.conf`
   - 포맷(택1 키):  
     ```ini
     webhook = https://discord.com/api/webhooks/xxx/yyy
     # 또는
     NOTI_WEBHOOK = https://discord.com/api/webhooks/xxx/yyy
     # 또는
     webhook_url = https://discord.com/api/webhooks/xxx/yyy
     ```

> **보안 팁**: Webhook URL은 **토큰**. 저장소에 커밋 금지. 노출 시 Discord에서 재발급/기존 삭제 권장.

### 2) 프리셋 파일 `.noti.preset`

- 탐색 순서: `$NOTI_PRESET` → `./.noti.preset` → `~/.noti.preset`
- 형식: 한 줄에 **이름 = 템플릿** 또는 **이름: 템플릿**
- 템플릿 안에서 **`$1 $2 $@`** 사용해 인자 바인딩 (git alias와 동일한 감각)
- **예약어 금지**: `send`, `embed`, `file`, `run`, `help`, `-h`, `--help`
- 예시:
  ```ini
  # .noti.preset
  deploy = embed --title "Deploy" --desc "$1" --color 3066993 --field Service api --field Version "$2"
  ping = send "[PING] $1"
  test-fail = run --title "CI: $1" --attach-log-on-fail -- bash -lc "$2"
  tarball = run --title "Tar $1" -- bash -lc 'tar -czf "$2" ${@:3}'
  ```
- 사용:
  ```bash
  noti deploy "✅ 성공" v1.2.3
  noti ping "서버 체크"
  noti test-fail unit 'make test'
  noti tarball v1.2.3 build/out.tgz ./build
  ```

> **따옴표 요령**: 공백/특수문자가 들어갈 수 있는 자리는 템플릿에서 **반드시 따옴표**로 감싸.  
> 예: `--desc "$1"`, `-- bash -lc '$2'` 등.

---

## 환경변수 (Environment Variables)

- `NOTI_WEBHOOK` : Discord Webhook URL (최우선)
- `NOTI_CONFIG`  : `.noti.conf` 경로 강제
- `NOTI_PRESET`  : `.noti.preset` 경로 강제
- `NOTI_DEBUG`   : 디버그 레벨
  - `0` 또는 unset: 끔
  - `1`: 기본 디버그(시도/HTTP 코드/재시도/백오프, URL 마스킹)
  - `2`: + 페이로드 앞 200자 프리뷰

예:
```bash
export NOTI_DEBUG=1
noti send "debug on"
```

---

## 서브커맨드 (Commands)

### 1) `send` — 텍스트 메시지

**형식**
```bash
noti send [--allow-mentions] <message...>
``)

**옵션**
- `--allow-mentions` : 멘션 허용(기본 차단: `@everyone`, `@here` 안 울림)

**예시**
```bash
noti send "배포 시작"
noti send --allow-mentions "@here 배포 시작"
```

---

### 2) `embed` — 임베드 카드

**형식**
```bash
noti embed --title <t> --desc <d> [--color <int>]   [--field <name> <value>]... [--allow-mentions]
```

**설명**
- `--color` 는 10진수(예: 초록 `3066993`, 빨강 `15158332`, 파랑 `3447003`)
- `--field` 는 여러 번 사용 가능 (inline=true로 표시)

**예시**
```bash
noti embed   --title "Build Result"   --desc "✅ Success"   --color 3066993   --field Branch main   --field Duration "12m 3s"
```

> `jq` 없으면 간소 텍스트로 폴백됨.

---

### 3) `file` — 파일 업로드

**형식**
```bash
noti file <path> [--message <m>] [--allow-mentions]
```

**설명**
- 멀티파트 업로드(`file1=@<path>`). Discord **업로드 제한 정책**에 따름.
- `--message` 로 캡션 텍스트 동봉

**예시**
```bash
printf "line 1
line 2
" > /tmp/demo.log
noti file /tmp/demo.log --message "빌드 로그"
```

---

### 4) `run` — 명령 실행 후 결과 전송

**형식**
```bash
noti run [--title <t>] [--attach-log] [--attach-log-on-fail]   [--allow-mentions] -- <cmd> [args...]
```

**설명**
- 실행 시간/시작/종료/exit code 포함
- `--attach-log`           : 항상 실행 로그 첨부
- `--attach-log-on-fail`   : 실패시에만 로그 첨부
- `jq` 있으면 임베드 카드로 요약, 없으면 텍스트

**예시**
```bash
# 성공 케이스
noti run --title "Quick success" -- echo "ok"

# 실패 + 실패시에만 로그 첨부
noti run --title "Fail demo" --attach-log-on-fail --   bash -c 'echo "compile..."; echo "ERR" >&2; exit 1'

# 항상 로그 첨부
noti run --title "With log always" --attach-log --   bash -c 'echo "doing work"; sleep 2; echo "done"'
```

**반환값**
- 내부적으로 실행한 명령의 **exit code** 를 그대로 반환.

---

## 프리셋(별칭) 상세

- 프리셋은 **서브커맨드 앞단 프리 디스패치**로 처리돼서, `noti <이름> ...` 형태로 바로 실행 가능.
- 템플릿은 이후 **재호출**되어 실제 `send|embed|file|run` 으로 전달됨.
- 인자 바인딩 규칙:
  - `$1`, `$2`, … : 위치 인자
  - `${@:3}`      : 3번째 이후 전부
  - 공백/특수문자 포함 가능(템플릿에서 꼭 따옴표로 감싸)
- 예약어 이름 금지: `send`, `embed`, `file`, `run`, `help`, `-h`, `--help`

**고급 예시**
```ini
# .noti.preset
# 멘션 허용 ping
ping-here = send --allow-mentions "@here [PING] $1"

# 커스텀 색으로 요약
summary = embed --title "$1" --desc "$2" --color 3447003   --field Tag "$3" --field Host "$4"
```

```bash
noti ping-here "서버 체크"
noti summary "Backup" "✅ OK" nightly db01
```

---

## 디버깅

```bash
export NOTI_DEBUG=1   # 기본 디버그
export NOTI_DEBUG=2   # + 페이로드 프리뷰(앞 200자)
```

**나오는 로그**
- 전송 URL(토큰 마스킹)
- 시도 횟수/HTTP 코드
- 429의 `retry_after` 대기 / 5xx 백오프
- 프리셋 로드 경로/개수
- (레벨2) 페이로드 일부 프리뷰

---

## 레이트 리밋/에러 처리

- **204**: 성공(Discord Webhook은 본문 없이 204 반환)
- **429**: `retry_after` 만큼 자동 대기 후 재시도
- **5xx**: 지수 백오프 재시도
- 그 외: STDERR로 코드/본문 출력 후 실패 반환

---

## 보안 모범 사례

- Webhook URL은 절대 코드/레포에 커밋하지 말 것
- CI 비밀변수/환경변수에 저장
- 로컬 히스토리 회피: `HISTCONTROL=ignorespace` + 명령 앞에 공백
- 디버그 레벨 2에서도 URL은 마스킹됨

---

## CI/CD 예시

**GitHub Actions**
```yaml
- name: Notify
  run: |
    echo "ok" > build.log
    NOTI_WEBHOOK="${{ secrets.DISCORD_WEBHOOK }}" noti file build.log --message "빌드 로그"
```

**GitLab CI**
```yaml
script:
  - export NOTI_WEBHOOK="$DISCORD_WEBHOOK"
  - noti run --title "CI: test" --attach-log-on-fail -- make test
```

**cron**
```bash
0 9 * * 1-5 NOTI_WEBHOOK=... /usr/local/bin/noti send "Good morning ☕"
```

**Makefile**
```makefile
notify:
	NOTI_WEBHOOK=$(DISCORD_WEBHOOK) noti send "make notify"
```

---

## 문제 해결(Trbleshooting)

- `NOTI_WEBHOOK 가 비어있음`  
  → `export NOTI_WEBHOOK=...` 또는 `.noti.conf`에 `webhook = ...` 추가
- `임베드가 텍스트로만 보임`  
  → `jq` 설치 필요
- **공백/따옴표 인자 깨짐**  
  → 프리셋 템플릿에서 반드시 `"${1}"` 식으로 따옴표 사용
- `알 수 없는 명령 또는 프리셋 없음`  
  → `.noti.preset` 경로/오탈자 확인. `NOTI_PRESET`로 위치 강제 가능.
- 반응 없음  
  → 성공 시 204라 터미널 출력이 없음. 채널에서 도착 확인 + `NOTI_DEBUG=1`로 재시도.

---

## 반환값 요약

- `send|embed|file`: HTTP 204이면 **0**, 그 외 실패 시 **1**
- `run`: **실행한 명령의 exit code** 그대로 반환

---

## 라이선스 / 크레딧

- 사내/개인 자동화용 스크립트 성격. 별도 라이선스가 필요하면 README 하단에 추가하면 돼.

