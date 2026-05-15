---
name: noti
description: Use this skill when the user asks to send a notification to Discord or Slack via the local `noti` CLI. Trigger phrases include "슬랙으로/디스코드로 알려/보내/알림", "notify/ping the team/channel", "post build/deploy/test result", "이거 알림 보내/노티", "send a webhook message", or any request to alert someone about a finished task through a chat channel. Skip when the user wants email, Telegram, SMS, or other non-Discord/Slack channels, or when no NOTI_WEBHOOK is configured.
---

# noti — Discord/Slack 알림 보내기

The `noti` bash CLI (https://github.com/tot0rokr/noti-bash) sends messages to Discord or Slack incoming webhooks. Provider is auto-detected from the webhook URL; the same flags work for both.

## When to use this skill

Trigger this skill whenever the user wants a chat-channel notification dispatched from the current shell — typically after a build, deploy, migration, long task, or to ping a team. The skill is a thin guide around the `noti` binary; everything happens via the `Bash` tool.

Do **not** use this skill for:
- Channels other than Discord/Slack (email, Telegram, SMS, etc.).
- Code-review or content-creation tasks that just *mention* the word "노티" without intent to send a webhook.
- When `NOTI_WEBHOOK` is not configured — instead, tell the user to set it.

## Prerequisites

Before sending anything:

1. `command -v noti` — confirm the binary is on `PATH`.
2. `[[ -n "$NOTI_WEBHOOK" ]]` — confirm the webhook URL is set in the environment.

If either is missing, stop and tell the user concretely what to do. Example: *"NOTI_WEBHOOK 환경변수가 설정되어 있지 않습니다. `export NOTI_WEBHOOK=...` 로 Discord 또는 Slack incoming webhook URL을 지정해 주세요."*

Never paste the contents of `$NOTI_WEBHOOK` back to the user — the URL is a credential.

## Decision tree

Pick the smallest subcommand that does the job.

1. **Short one-line message?** → `noti send "..."`
2. **Rich card with title / description / color / fields?** → `noti embed --title ... --desc ... --color ... --field K V ...`
3. **Result of a build / test / deploy / migration?**
   - Run the command yourself with the `Bash` tool so the user sees stdout/stderr live.
   - Then call `noti send` or `noti embed` with the result.
   - Do **not** use `noti run` from this skill — it hides the live output that the `Bash` tool would otherwise show.
4. **File attachment?**
   - Discord: `noti file <path> --message "..."`
   - Slack: not supported. Inline the relevant snippet (last ~30 lines) inside a code block in `noti send` instead.

## Color convention (`--color`)

`noti` accepts decimal, `#hex`, or `0xhex` and normalizes per provider.

| Status        | Decimal     | Hex        |
|---------------|-------------|------------|
| ✅ Success    | `3066993`   | `#2ecc71`  |
| ❌ Failure    | `15158332`  | `#e74c3c`  |
| ℹ️ Info / WIP | `3447003`   | `#3498db`  |
| ⚠️ Warning    | `15844367`  | `#f1c40f`  |

## Common patterns

### Build / deploy outcome

```bash
if make build; then
  noti embed --title "Build" --desc "✅ Success" --color "#2ecc71" \
    --field Branch "$(git rev-parse --abbrev-ref HEAD)" \
    --field Commit "$(git rev-parse --short HEAD)"
else
  noti embed --title "Build" --desc "❌ Failed" --color "#e74c3c" \
    --field Branch "$(git rev-parse --abbrev-ref HEAD)" \
    --field Stage "make build"
fi
```

### Quick ping with timestamp

```bash
noti send "✅ 데이터 마이그레이션 완료 ($(date '+%H:%M'))"
```

### Channel mention

Mentions behave differently per provider — pick the right form:

```bash
# Discord (default mentions are blocked; opt in explicitly)
noti send --allow-mentions "@here 긴급: prod 5xx 폭증"

# Slack (use the literal <!channel>/<!here>/<@U...> form in the body;
# --allow-mentions is a no-op on Slack)
noti send "<!channel> 긴급: prod 5xx 폭증"
```

### Multi-line body

```bash
noti send "$(cat <<'EOF'
배포 완료
- 서비스: api
- 버전: v1.2.3
- 소요: 4분
EOF
)"
```

### Slack file-attachment fallback (inline snippet)

```bash
# Slack rejects `noti file`. Inline a tail of the log instead:
log_tail=$(tail -n 30 build.log)
noti send "$(printf 'Build log (tail):\n```\n%s\n```\n' "$log_tail")"
```

## Safety & etiquette

- **Never echo the raw webhook URL** to the user, into files, or into commits. `noti`'s own debug output already masks the token; reuse that, don't print `$NOTI_WEBHOOK` directly. If the user needs to confirm which webhook is set, show only the masked form: `printf '%s' "$NOTI_WEBHOOK" | sed -E 's#(/[A-Za-z0-9_-]{8})[A-Za-z0-9_-]+#\1***#g'`.
- Never write the webhook URL into project files (README, `.env.example`, scripts, CI configs in the repo). CI should pull it from a secret.
- Avoid sending more than ~5 messages in a short burst to the same channel. Slack rate-limits at roughly 1/sec; Discord also throttles. When you have many things to report, batch them into one `embed` with multiple `--field` entries.
- For sensitive content (tokens, PII, customer data) appearing in logs: redact before sending, or send a summary instead of the raw lines.

## Reading the result

- Success → exit `0`, no stdout.
- Failure → non-zero exit, human-readable Korean error on stderr (e.g. `HTTP 404`, `재시도 한도 초과`).
- Slack URL + `noti file` → exit `2` with a clear "incoming webhook 은 파일 업로드를 지원하지 않습니다" message. Switch to the inline-snippet pattern above.
- Always check the exit code before telling the user "전송 완료" — never assume success.

```bash
if noti send "..."; then
  echo "전송 완료"
else
  echo "전송 실패 (exit=$?). 위 stderr 메시지를 확인하세요."
fi
```

## Troubleshooting

- `Unrecognized webhook URL provider` → the `NOTI_WEBHOOK` value doesn't look like a Discord (`discord.com/api/webhooks/...`) or Slack (`hooks.slack.com/services/...`) URL. Re-check.
- `재시도 한도 초과` → transient 5xx or rate-limit. Suggest a 1–2 min wait, or batch into fewer messages.
- `jq: command not found` (warnings about embed degrading to plain text) → `sudo apt install jq` (Debian/Ubuntu) or `brew install jq` (macOS) for richer embeds.
- Message arrived but looks plain on Slack while expecting Discord-style bold (`**x**`) — Slack uses `*x*` for bold and `~x~` for strike. `noti` does not auto-convert; write provider-appropriate markdown when the channel matters.
