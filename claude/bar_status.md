# Claude Code Statusline 설정 가이드

## 필요 파일 3개

---

### 1. `~/.claude/settings.json` (statusLine 설정 추가)

기존 settings.json에 statusLine 블록만 추가하면 됩니다:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash $HOME/.claude/statusline-command.sh"
  }
}
```

---

### 2. `~/.claude/token-config.json` (새로 생성)

```json
{
  "plan": "max_5x",
  "session_token_budget": 300000,
  "reset_time_local": "19:00"
}
```

| 키 | 설명 |
|:---|:---|
| `session_token_budget` | Max 5x 기준 ~300K. /usage와 오차 있으면 조절 |
| `reset_time_local` | 토큰 리셋 시간 (KST 기준, /usage에서 확인) |

---

### 3. `~/.claude/statusline-command.sh` (새로 생성)

```bash
#!/bin/bash

# Read JSON input from stdin
input=$(cat)

# ANSI Color codes
RESET=$'\033[0m'
BOLD=$'\033[1m'
DIM=$'\033[2m'
RED=$'\033[31m'
GREEN=$'\033[32m'
YELLOW=$'\033[33m'
BLUE=$'\033[34m'
MAGENTA=$'\033[35m'
CYAN=$'\033[36m'
GRAY=$'\033[90m'
WHITE=$'\033[97m'

# Extract values from JSON
model_name=$(echo "$input" | jq -r '.model.display_name // "Claude"')
cwd=$(echo "$input" | jq -r '.workspace.current_dir // ""')
total_input=$(echo "$input" | jq -r '.context_window.total_input_tokens // 0')
total_output=$(echo "$input" | jq -r '.context_window.total_output_tokens // 0')
ctx_size=$(echo "$input" | jq -r '.context_window.context_window_size // 200000')
ctx_used_pct=$(echo "$input" | jq -r '.context_window.used_percentage // empty')

# Shorten model name
if echo "$model_name" | grep -qi "opus"; then
    model_short=$(echo "$model_name" | sed -E 's/.*([Oo]pus[^"]*)/\1/' | sed 's/^ *//')
elif echo "$model_name" | grep -qi "sonnet"; then
    model_short=$(echo "$model_name" | sed -E 's/.*([Ss]onnet[^"]*)/\1/' | sed 's/^ *//')
elif echo "$model_name" | grep -qi "haiku"; then
    model_short=$(echo "$model_name" | sed -E 's/.*([Hh]aiku[^"]*)/\1/' | sed 's/^ *//')
else
    model_short=$(echo "$model_name" | sed 's/Claude //')
fi

# Get git info
git_info=""
if [ -d "$cwd/.git" ] || git -C "$cwd" rev-parse --git-dir > /dev/null 2>&1; then
    git_branch=$(git -C "$cwd" --no-optional-locks branch --show-current 2>/dev/null || echo "")
    if [ -n "$git_branch" ]; then
        if git -C "$cwd" --no-optional-locks diff-index --quiet HEAD -- 2>/dev/null; then
            git_status=""
        else
            git_status="${RED}✗${RESET}"
        fi
        git_info=" ${GRAY}git:(${GREEN}${git_branch}${GRAY})${RESET}${git_status:+ $git_status}"
    fi
fi

# === Helpers ===

format_tokens() {
    local tokens=$1
    if [ "$tokens" -ge 1000000 ] 2>/dev/null; then
        printf "%.1fM" "$(echo "$tokens / 1000000" | bc -l)"
    elif [ "$tokens" -ge 1000 ] 2>/dev/null; then
        printf "%.1fK" "$(echo "$tokens / 1000" | bc -l)"
    else
        printf "%s" "$tokens"
    fi
}

build_bar() {
    local pct=$1 width=${2:-6}
    local pct_int=${pct%.*}
    [ -z "$pct_int" ] && pct_int=0
    local filled=$(( (pct_int * width + 50) / 100 ))
    [ "$filled" -gt "$width" ] && filled=$width
    [ "$filled" -lt 0 ] && filled=0
    local empty=$(( width - filled ))
    local bar=""
    for ((i=0; i<filled; i++)); do bar="${bar}█"; done
    for ((i=0; i<empty; i++)); do bar="${bar}░"; done
    echo "$bar"
}

pct_color() {
    local pct_int=${1%.*}
    [ -z "$pct_int" ] && pct_int=0
    if [ "$pct_int" -lt 50 ]; then echo "$GREEN"
    elif [ "$pct_int" -lt 75 ]; then echo "$YELLOW"
    elif [ "$pct_int" -lt 90 ]; then echo "$RED"
    else echo "${BOLD}${RED}"
    fi
}

# ============================================================
# 1) CONTEXT WINDOW BAR
# ============================================================

context_display=""
if [ -n "$ctx_used_pct" ] && [ "$ctx_used_pct" != "null" ]; then
    ctx_used="$ctx_used_pct"
else
    # Calculate from current_usage tokens when percentage not provided
    cur_input=$(echo "$input" | jq -r '.context_window.current_usage.input_tokens // 0')
    cur_output=$(echo "$input" | jq -r '.context_window.current_usage.output_tokens // 0')
    cur_cache_create=$(echo "$input" | jq -r '.context_window.current_usage.cache_creation_input_tokens // 0')
    cur_cache_read=$(echo "$input" | jq -r '.context_window.current_usage.cache_read_input_tokens // 0')
    cur_total=$((cur_input + cur_output + cur_cache_create + cur_cache_read))
    if [ "$ctx_size" -gt 0 ] 2>/dev/null && [ "$cur_total" -gt 0 ] 2>/dev/null; then
        ctx_used=$((cur_total * 100 / ctx_size))
        [ "$ctx_used" -gt 100 ] && ctx_used=100
    fi
fi

if [ -n "$ctx_used" ] && [ "$ctx_used" != "0" ]; then
    ctx_color=$(pct_color "$ctx_used")
    ctx_bar=$(build_bar "$ctx_used" 6)
    ctx_int=${ctx_used%.*}
    context_display="${GRAY}Ctx${RESET} ${ctx_color}${ctx_bar}${RESET} ${ctx_color}${ctx_int}%${RESET}"
fi

# ============================================================
# 2) TOKEN USAGE (from JSON input total_output_tokens)
# ============================================================

token_config="$HOME/.claude/token-config.json"
session_budget=300000
reset_time_local="19:00"
if [ -f "$token_config" ] && command -v jq >/dev/null 2>&1; then
    session_budget=$(jq -r '.session_token_budget // 300000' "$token_config" 2>/dev/null)
    reset_time_local=$(jq -r '.reset_time_local // "19:00"' "$token_config" 2>/dev/null)
fi

# Use total_output_tokens from the JSON input (Claude Code's cumulative tracking)
session_tokens=${total_output:-0}

token_display=""
if [ "$session_budget" -gt 0 ] 2>/dev/null && [ "$session_tokens" -gt 0 ] 2>/dev/null; then
    tok_pct=$((session_tokens * 100 / session_budget))
    [ "$tok_pct" -gt 100 ] && tok_pct=100
    tok_color=$(pct_color "$tok_pct")
    tok_bar=$(build_bar "$tok_pct" 6)
    fmt_used=$(format_tokens "$session_tokens")
    fmt_budget=$(format_tokens "$session_budget")
    token_display="${GRAY}Tok${RESET} ${tok_color}${tok_bar}${RESET} ${tok_color}${tok_pct}%${RESET} ${GRAY}${fmt_used}/${fmt_budget}${RESET}"
fi

# ============================================================
# 3) RESET TIMER
# ============================================================

reset_display=""
now_epoch=$(date +%s)
today=$(date +%Y-%m-%d)
reset_h=${reset_time_local%%:*}
reset_m=${reset_time_local##*:}
today_reset_epoch=$(date -j -f "%Y-%m-%d %H:%M:%S" "${today} ${reset_h}:${reset_m}:00" +%s 2>/dev/null)

if [ -n "$today_reset_epoch" ]; then
    if [ "$now_epoch" -ge "$today_reset_epoch" ]; then
        if [ "$(uname)" = "Darwin" ]; then
            tomorrow=$(date -v+1d +%Y-%m-%d)
        else
            tomorrow=$(date -d "+1 day" +%Y-%m-%d)
        fi
        today_reset_epoch=$(date -j -f "%Y-%m-%d %H:%M:%S" "${tomorrow} ${reset_h}:${reset_m}:00" +%s 2>/dev/null)
    fi

    secs_left=$((today_reset_epoch - now_epoch))
    h_left=$((secs_left / 3600))
    m_left=$(( (secs_left % 3600) / 60 ))

    reset_display="${GRAY}Reset${RESET} ${DIM}${h_left}h${m_left}m${RESET} ${GRAY}(${reset_time_local})${RESET}"
fi

# ============================================================
# ASSEMBLE
# ============================================================

dir_name=$(basename "$cwd")

parts="${BOLD}${MAGENTA}[${model_short}]${RESET}"
[ -n "$context_display" ] && parts="${parts} ${context_display}"
[ -n "$token_display" ] && parts="${parts} ${GRAY}|${RESET} ${token_display}"
parts="${parts} ${GRAY}|${RESET} ${CYAN}➜${RESET} ${BOLD}${BLUE}${dir_name}${RESET}${git_info}"
[ -n "$reset_display" ] && parts="${parts} ${GRAY}|${RESET} ${reset_display}"

printf "%s" "$parts"
```

---

## 셋업 원라이너

다른 컴퓨터에서 아래 한 줄로 세팅 가능:

```bash
mkdir -p ~/.claude && curl -sL <gist-url>/statusline-command.sh -o ~/.claude/statusline-command.sh && echo '{"plan":"max_5x","session_token_budget":300000,"reset_time_local":"19:00"}' > ~/.claude/token-config.json && jq '.statusLine = {"type":"command","command":"bash $HOME/.claude/statusline-command.sh"}' ~/.claude/settings.json > /tmp/s.json && mv /tmp/s.json ~/.claude/settings.json
```

> (gist에 올린 후 URL 교체, 또는 수동으로 파일 3개 복사)

---

## 의존성

- **jq** - JSON 파서 (`brew install jq`)
- **git** - git 브랜치/상태 표시용 (없으면 해당 부분 생략됨)
- **bc** - 토큰 K/M 포맷 (macOS 기본 포함)

---

## 출력 예시

```
[Opus 4.5] Ctx ███░░░ 52% | Tok █░░░░░ 24% 74.0K/300.0K | ➜ myproject git:(main) | Reset 1h40m (19:00)
```

| 항목 | 설명 |
|:---|:---|
| **Ctx** | 컨텍스트 윈도우 사용률 (초록→노랑→빨강) |
| **Tok** | 세션 출력 토큰 사용률 (/usage와 근사) |
| **Reset** | 다음 리셋까지 남은 시간 |
