# noti — Discord/Slack Webhook 통합 Bash 알림

작업 결과를 **Discord/Slack Webhook**으로 보내는 단일 바이너리(스크립트).  
텍스트/임베드/파일 업로드/명령 실행 결과 요약을 전송하고, **프리셋(.noti.preset)** 으로 커맨드를 짧게 별칭처럼 쓸 수 있어.

> Webhook URL 에서 **provider 를 자동 감지**합니다 (`discord.com/api/webhooks/...` → Discord, `hooks.slack.com/services/...` → Slack). 동일한 CLI/플래그/preset 파일이 양쪽 모두에서 동작합니다.

---

## 목차

- [특징 (Features)](#특징-features)
- [Provider 지원 매트릭스](#provider-지원-매트릭스)
- [요구사항](#요구사항)
- [설치](#설치)
- [Claude Code skill](#claude-code-skill)
- [Webhook 발급받기](#webhook-발급받기)
  - [Discord — Server / Channel webhook](#discord--server--channel-webhook)
  - [Slack — Incoming Webhooks](#slack--incoming-webhooks)
  - [발급한 URL 확인](#발급한-url-확인)
- [빠른 시작 (Quick Start)](#빠른-시작-quick-start)
- [설정 (Configuration)](#설정-configuration)
- [환경변수 (Environment Variables)](#환경변수-environment-variables)
- [서브커맨드 (Commands)](#서브커맨드-commands)
  - [`send` — 텍스트 메시지](#1-send--텍스트-메시지)
  - [`embed` — 임베드 카드](#2-embed--임베드-카드)
  - [`file` — 파일 업로드 (Discord 전용)](#3-file--파일-업로드-discord-전용)
  - [`run` — 명령 실행 후 결과 전송](#4-run--명령-실행-후-결과-전송)
- [프리셋(별칭) 상세](#프리셋별칭-상세)
- [디버깅](#디버깅)
- [레이트 리밋/에러 처리](#레이트-리밋에러-처리)
- [보안 모범 사례](#보안-모범-사례)
- [CI/CD 예시](#cicd-예시)
- [문제 해결(Troubleshooting)](#문제-해결troubleshooting)
- [반환값 요약](#반환값-요약)
- [라이선스](#라이선스)

---

## 특징 (Features)

- **Discord & Slack 동시 지원**: webhook URL 로 자동 감지, 동일한 CLI/preset/플래그
- `send` 텍스트, `embed` 카드, `file` 업로드(Discord), `run` 실행/요약 전송 지원
- **프리셋(.noti.preset)**: `noti <이름> <args...>` 로 재사용 (git alias 느낌, `$1 $2 $@` 자리표시자)
- 성공/실패/소요시간/시작/종료 시각 자동 포함 (run)
- 로그 파일 자동 첨부 옵션(실패시에만/항상) — Slack 에서는 인라인 코드블록으로 fallback
- **Discord 멘션 차단 기본값**, `--allow-mentions`로 필요한 경우만 허용
- **429 rate limit** 자동 재시도(Discord: body `retry_after`, Slack: `Retry-After` 헤더), **5xx** 지수 백오프
- **디버그 모드**(마스킹된 URL, 페이로드 프리뷰)
- Linux/macOS/WSL에서 동작(표준 `bash`, `curl` 필요), `jq` 설치 시 임베드 품질↑

---

## Provider 지원 매트릭스

| 기능                       | Discord | Slack |
|----------------------------|:-------:|:-----:|
| `send` (텍스트)            |   ✅    |  ✅   |
| `embed` (제목/설명/색/필드)|   ✅    |  ✅   |
| `file` (파일 업로드)       |   ✅    |  ❌   |
| `run` (요약 임베드)        |   ✅    |  ✅   |
| `--attach-log[-on-fail]`   | ✅ 임베드 + 파일 | ✅ 임베드 + 인라인 코드블록 |
| `--allow-mentions`         |   ✅    | ⚠️ no-op |
| 자동 rate-limit 재시도     |   ✅    |  ✅   |

> Slack 의 파일 업로드/스레드 댓글은 Bot Token + `chat.postMessage`/`files.upload` 가 필요하므로 v1 범위 밖입니다. 코드에 `TODO(slack-bot-token)` 주석으로 확장 지점만 남겨두었습니다.

---

## 요구사항

- **bash 4+** (연관 배열 사용), `curl` 필수
- 선택: `jq` (JSON 생성/임베드/필드 처리 안정)

---

## 설치

```bash
# 1) 시스템 전역 설치 (권장)
sudo install -m 0755 noti /usr/local/bin/noti

# 또는 사용자 영역에 설치 (PATH 에 ~/.local/bin 포함되어 있어야 함)
install -m 0755 noti ~/.local/bin/noti

# 2) 개발용 symlink (편집이 즉시 PATH 에 반영됨)
ln -sf "$PWD/noti" ~/.local/bin/noti

# 실행 확인
noti help
```

---

## Claude Code skill

`skills/noti/SKILL.md` 가 함께 포함되어 있어, Claude Code 사용자라면 "이 빌드 결과 슬랙으로 알려" / "디스코드로 알림 보내" 같은 자연어 트리거로 LLM 이 `noti` CLI 를 자동 호출하게 만들 수 있습니다.

설치 (skill 디렉터리를 Claude Code 가 인식하는 경로로 symlink):

```bash
# user-level skill 디렉터리에 연결
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/noti" ~/.claude/skills/noti
```

`~/.claude/skills` 가 dotfiles repo 의 다른 경로로 symlink 되어 있다면 그 실제 경로에 링크하세요. 설치 후 Claude Code 세션이 자동으로 skill 을 picks up 합니다 (재시작 불필요).

전제 조건:
- `noti` 가 PATH 에 있을 것 (위 §설치 참고)
- `NOTI_WEBHOOK` 환경변수에 Discord/Slack incoming webhook URL 이 설정되어 있을 것

skill 본문은 Claude 가 따라가는 의사결정 가이드(언제 `send`/`embed` 를 쓸지, 색상 컨벤션, Slack 의 파일 첨부 우회 등)를 담고 있어 추가 설정 없이 그대로 동작합니다.

---

## Webhook 발급받기

`noti` 는 **incoming webhook URL** 만 있으면 동작합니다. URL 자체가 인증 토큰이므로 외부에 노출되지 않게 주의하세요(저장소 커밋 금지, CI 비밀변수 사용).

### Discord — Server / Channel webhook

1. Discord 서버에서 채널 우클릭 → **채널 편집(Edit Channel)** → **연동(Integrations)** → **웹후크(Webhooks)**.
   - 또는 서버 설정 → 연동 → 웹후크.
2. **새 웹후크(New Webhook)** 클릭.
3. 이름/아이콘/대상 채널을 지정한 뒤 **웹후크 URL 복사(Copy Webhook URL)**.
4. URL 형식: `https://discord.com/api/webhooks/<ID>/<TOKEN>`
5. 환경변수에 설정:
   ```bash
   export NOTI_WEBHOOK='https://discord.com/api/webhooks/.../...'
   ```

> 필요 권한: 서버에서 **웹후크 관리(Manage Webhooks)** 권한이 있어야 생성/조회 가능합니다.  
> 토큰 유출 시 같은 화면에서 **삭제** 또는 **새 URL 발급**(기존 URL 폐기) 하세요.

### Slack — Incoming Webhooks

가장 단순한 방식은 **Slack App** 의 *Incoming Webhooks* 기능을 사용하는 것입니다.

1. <https://api.slack.com/apps> 접속 → **Create New App** → **From scratch**.
2. 앱 이름과 워크스페이스 선택 → **Create App**.
3. 좌측 메뉴 **Features → Incoming Webhooks** → 토글을 **On** 으로.
4. 페이지 하단 **Add New Webhook to Workspace** → 게시할 채널 선택 → **Allow**.
5. 생성된 URL을 복사. 형식: `https://hooks.slack.com/services/<T...>/<B...>/<...>`
6. 환경변수에 설정:
   ```bash
   export NOTI_WEBHOOK='https://hooks.slack.com/services/.../.../...'
   ```

> 한 webhook URL 은 **하나의 채널에만 게시**됩니다. 채널을 바꾸려면 새 webhook 을 추가하세요.  
> 워크스페이스 관리자 승인이 필요할 수 있습니다(워크스페이스 설정에 따라 다름).  
> Slack incoming webhook 은 **파일 업로드/스레드 댓글**을 지원하지 않습니다. 해당 기능이 필요하면 Bot Token + `chat.postMessage`/`files.upload` API 를 사용해야 하며, 이는 v1 범위 밖입니다.

### 발급한 URL 확인

```bash
NOTI_DEBUG=1 noti send "hello"
# debug 로그에서 provider 와 마스킹된 URL 이 표시됨
# 예: [noti:debug] provider=discord
#     [noti:debug] post_json start provider=discord url=https://discord.com/api/webhooks/123/*** size=...
```

---

## 빠른 시작 (Quick Start)

```bash
# 1) Webhook 설정 (환경변수 or .noti.conf 중 하나). 둘 중 하나만 쓰면 됨.
export NOTI_WEBHOOK='https://discord.com/api/webhooks/.../...'   # Discord
# 또는
export NOTI_WEBHOOK='https://hooks.slack.com/services/.../.../...'  # Slack

# 2) 간단 메시지
noti send "hello from noti 👋"

# 3) 임베드(색/필드) — 색상은 10진수/hex/0x표기 모두 OK
noti embed --title "Deploy" --desc "✅ Success" --color 3066993   --field Service api --field Version v1.2.3
noti embed --title "Deploy" --desc "✅ Success" --color "#2ecc71" --field Service api --field Version v1.2.3

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
     webhook = https://discord.com/api/webhooks/xxx/yyy          # Discord
     # 또는
     webhook = https://hooks.slack.com/services/xxx/yyy/zzz       # Slack
     # NOTI_WEBHOOK = / webhook_url = 키도 동일하게 인식됨
     ```

> Provider 는 URL 패턴으로 자동 판별됩니다. 패턴에 매칭되지 않는 URL 은 exit code 2 와 함께 거부합니다.

> **보안 팁**: Webhook URL은 **토큰**. 저장소에 커밋 금지. 노출 시 Discord/Slack 에서 재발급/기존 삭제 권장.

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

- `NOTI_WEBHOOK` : Discord 또는 Slack incoming webhook URL (최우선)
- `NOTI_CONFIG`  : `.noti.conf` 경로 강제
- `NOTI_PRESET`  : `.noti.preset` 경로 강제
- `NOTI_DEBUG`   : 디버그 레벨
  - `0` 또는 unset: 끔
  - `1`: 기본 디버그(provider/시도/HTTP 코드/재시도/백오프, URL 마스킹)
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
- `--allow-mentions` : Discord 한정. 멘션 허용(기본 차단: `@everyone`, `@here` 안 울림).
  Slack incoming webhook 은 평문 `@channel`을 멘션으로 변환하지 않으므로 이 플래그는 Slack 에서 no-op.

**예시**
```bash
noti send "배포 시작"
noti send --allow-mentions "@here 배포 시작"          # Discord
noti send "<!channel> 배포 시작"                      # Slack 에서 채널 멘션을 원하면 본문에 <!channel>
```

---

### 2) `embed` — 임베드 카드

**형식**
```bash
noti embed --title <t> --desc <d> [--color <c>]   [--field <name> <value>]... [--allow-mentions]
```

**설명**
- `--color` 는 다음 형식 모두 허용:
  - 10진수: `3066993` (초록), `15158332` (빨강), `3447003` (파랑)
  - hex: `#2ecc71`, `#e74c3c`
  - 0x 표기: `0x2ecc71`
  - Discord 에서는 10진수, Slack 에서는 `#rrggbb` 로 변환되어 attachment 의 색상 사이드바로 표시됨.
- `--field` 는 여러 번 사용 가능 (Discord: `inline=true`, Slack: `short=true`)

**예시**
```bash
noti embed   --title "Build Result"   --desc "✅ Success"   --color 3066993   --field Branch main   --field Duration "12m 3s"
noti embed   --title "Build Result"   --desc "✅ Success"   --color "#2ecc71" --field Branch main   --field Duration "12m 3s"
```

> `jq` 없으면 간소 텍스트로 폴백됨.

> **Markdown 차이**: Discord 는 `**bold**`/`~~strike~~`, Slack 은 `*bold*`/`~strike~`. 본문은 자동 변환하지 않으니 대상에 맞춰 작성하세요.

---

### 3) `file` — 파일 업로드 (Discord 전용)

**형식**
```bash
noti file <path> [--message <m>] [--allow-mentions]
```

**설명**
- 멀티파트 업로드(`file1=@<path>`). Discord **업로드 제한 정책**에 따름.
- `--message` 로 캡션 텍스트 동봉
- **Slack 에서는 지원되지 않습니다.** Slack incoming webhook 은 파일 업로드를 받지 않으므로 호출 시 `exit 2` 와 함께 명확한 에러를 출력합니다. Slack 에서 로그를 보내고 싶다면 `noti run --attach-log` 의 인라인 fallback 을 사용하거나, 별도로 파일을 공유 채널/스토리지에 올린 뒤 `noti send` 로 링크를 게시하세요.

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
- **Slack 에서 `--attach-log[-on-fail]`**: 로그 파일 업로드 대신 **마지막 ~3500자를 인라인 코드블록**으로 후속 메시지에 첨부합니다 (Slack incoming webhook 은 `thread_ts` 를 받지 못해 진짜 스레드 댓글은 불가).

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

| 항목 | Discord | Slack |
|---|---|---|
| 성공 응답 | HTTP **204** (본문 없음) | HTTP **200** (본문 `ok`) |
| 429 대기 시간 출처 | 응답 body 의 `retry_after` 필드 (초; ms 단위 큰 값은 자동 보정) | 응답 헤더 `Retry-After` (초) |
| 5xx | 1~5초 지수 백오프 자동 재시도 | 동일 |
| 알 수 없는 URL | `ensure_webhook_or_die` 가 exit 2 | 동일 |
| 알 수 없는 HTTP 코드 | STDERR 로 코드/본문 출력 후 실패 반환 | 동일 |

재시도 한도는 `post_json` 의 기본 `max_retry=5`. 초과 시 stderr 에 `재시도 한도 초과` 출력 후 실패 반환.

---

## 보안 모범 사례

- Webhook URL은 절대 코드/레포에 커밋하지 말 것
- CI 비밀변수/환경변수에 저장
- 로컬 히스토리 회피: `HISTCONTROL=ignorespace` + 명령 앞에 공백
- 디버그 레벨 2에서도 URL은 마스킹됨

---

## CI/CD 예시

**GitHub Actions** (Discord)
```yaml
- name: Notify
  run: |
    echo "ok" > build.log
    NOTI_WEBHOOK="${{ secrets.DISCORD_WEBHOOK }}" noti file build.log --message "빌드 로그"
```

**GitHub Actions** (Slack — `noti file` 대신 `run --attach-log` 사용)
```yaml
- name: Notify
  env:
    NOTI_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
  run: |
    noti run --title "CI: build" --attach-log-on-fail -- make build
```

**GitLab CI**
```yaml
script:
  - export NOTI_WEBHOOK="$CI_NOTI_WEBHOOK"  # Discord 또는 Slack URL
  - noti run --title "CI: test" --attach-log-on-fail -- make test
```

**cron**
```bash
0 9 * * 1-5 NOTI_WEBHOOK=... /usr/local/bin/noti send "Good morning ☕"
```

**Makefile**
```makefile
notify:
	NOTI_WEBHOOK=$(NOTI_WEBHOOK) noti send "make notify"
```

---

## 문제 해결(Troubleshooting)

- `NOTI_WEBHOOK 이 설정되지 않았고 ...`  
  → `export NOTI_WEBHOOK=...` 또는 `.noti.conf`에 `webhook = ...` 추가
- `Unrecognized webhook URL provider.`  
  → URL 이 Discord/Slack 패턴(`discord.com/api/webhooks/...` 또는 `hooks.slack.com/services/...`) 에 맞지 않음. 오탈자/리다이렉트 도메인 확인.
- `Slack incoming webhook 은 파일 업로드를 지원하지 않습니다.`  
  → Slack URL 로는 `noti file` 사용 불가. `noti run --attach-log` 의 인라인 fallback 사용 또는 별도 스토리지에 파일 올려 링크 게시.
- `임베드가 텍스트로만 보임`  
  → `jq` 설치 필요
- **공백/따옴표 인자 깨짐**  
  → 프리셋 템플릿에서 반드시 `"${1}"` 식으로 따옴표 사용
- `알 수 없는 명령 또는 프리셋 없음`  
  → `.noti.preset` 경로/오탈자 확인. `NOTI_PRESET`로 위치 강제 가능.
- Discord 반응 없음  
  → 성공 시 204라 터미널 출력이 없음. 채널에서 도착 확인 + `NOTI_DEBUG=1`로 재시도.
- Slack `invalid_payload`  
  → 디버그 모드(`NOTI_DEBUG=2`)로 payload 프리뷰 확인. 빈 `text` + 빈 `attachments` 조합은 Slack 이 거부함.

---

## 반환값 요약

- `send|embed`: 성공 응답(Discord 204 / Slack 200)이면 **0**, 그 외 실패 시 **1**
- `file`: Discord 204 이면 **0**, 실패 시 **1**, Slack URL 에 대해 호출하면 **2** (미지원)
- `run`: **실행한 명령의 exit code** 그대로 반환

---

## 라이선스

[BSD 3-Clause License](./LICENSE). 자유롭게 수정/재배포 가능. 자세한 조건은 `LICENSE` 파일 참고.

