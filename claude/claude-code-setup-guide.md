# Claude Code 환경 설정 가이드

> **작성일**: 2026-02-09
> **Claude Code 버전**: 2.1.37
> **플랫폼**: macOS Darwin 25.1.0

이 문서는 현재 사용 중인 Claude Code 환경을 동일하게 구축하기 위한 가이드입니다.

---

## 📁 디렉토리 구조

```
~/.claude/
├── CLAUDE.md                    # 메인 진입점 (SuperClaude 프레임워크)
├── COMMANDS.md                  # 커맨드 정의
├── FLAGS.md                     # 플래그 시스템
├── PRINCIPLES.md                # 개발 원칙
├── RULES.md                     # 운영 규칙
├── MCP.md                       # MCP 서버 통합
├── PERSONAS.md                  # 페르소나 시스템
├── ORCHESTRATOR.md              # 라우팅 시스템
├── MODES.md                     # 운영 모드
├── commands/
│   └── sc/                      # SuperClaude 스킬 커맨드 (17개)
├── plugins/                     # 플러그인 시스템
├── calab-marketplace/           # 커스텀 마켓플레이스
├── settings.json                # 메인 설정
├── settings.local.json          # 권한 설정
├── token-config.json            # 토큰 관리 설정
├── statusline-command.sh        # 상태바 커스텀 스크립트
└── config.json                  # API 키 설정
```

---

## 1️⃣ 기본 설정 파일

### settings.json
```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "statusLine": {
    "type": "command",
    "command": "bash $HOME/.claude/statusline-command.sh"
  },
  "enabledPlugins": {
    "calab-plugin@calab-marketplace": true,
    "gopls-lsp@claude-plugins-official": true
  }
}
```

### token-config.json
```json
{
  "plan": "max_5x",
  "session_token_budget": 300000,
  "reset_time_local": "19:00"
}
```

---

## 2️⃣ MCP 서버 설정

### 위치: `~/Library/Application Support/Claude/claude_desktop_config.json`

```json
{
  "mcpServers": {
    "notion": {
      "command": "npx",
      "args": ["-y", "@suekou/mcp-notion-server"],
      "env": {
        "NOTION_API_TOKEN": "<YOUR_NOTION_TOKEN>"
      }
    },
    "terminal": {
      "command": "npx",
      "args": ["-y", "@dillip285/mcp-terminal"]
    },
    "google-search": {
      "command": "npx",
      "args": ["-y", "g-search-mcp"]
    },
    "context7": {
      "command": "npx",
      "args": ["-y", "@upstash/context7-mcp@latest"]
    },
    "tavily-mcp": {
      "command": "npx",
      "args": ["-y", "tavily-mcp@latest"],
      "env": {
        "TAVILY_API_KEY": "<YOUR_TAVILY_API_KEY>"
      }
    }
  },
  "preferences": {
    "menuBarEnabled": false,
    "quickEntryShortcut": "off"
  }
}
```

### 활성화된 MCP 서버 목록

| 서버 | 용도 | 패키지 |
|------|------|--------|
| **Serena** | 코드 심볼릭 분석, LSP 통합 | 별도 설치 |
| **Tavily** | 검증된 웹 검색, 공식 문서 조회 | `tavily-mcp@latest` |
| **Context7** | 라이브러리 문서 조회 | `@upstash/context7-mcp@latest` |
| **Notion** | 노션 연동 | `@suekou/mcp-notion-server` |
| **Terminal** | 터미널 접근 | `@dillip285/mcp-terminal` |
| **Google Search** | 구글 검색 | `g-search-mcp` |

---

## 3️⃣ Serena MCP 서버 설정

### 위치: `~/.serena/serena_config.yml`

```yaml
# GUI/대시보드 설정
gui_log_window: false
web_dashboard: false
web_dashboard_open_on_launch: true
log_level: 20

# LSP 설정
trace_lsp_communication: false
tool_timeout: 240
language_backend: LSP

# 토큰 추정기
token_count_estimator: TIKTOKEN_GPT4O

# 프로젝트 등록
projects:
  - /Users/cho/cho/source/skuber-platform/apps/core
  - /Users/cho/cho/source/skuberplus-agent

# 모드 설정
default_modes:
  - interactive
  - editing

# 도구 설정
excluded_tools: []
included_optional_tools: []
default_max_tool_answer_chars: 150000
```

### Serena 설치 방법
```bash
# Serena 소스 클론
git clone <serena-repo> ~/cho/source/serena

# 설정 디렉토리 생성
mkdir -p ~/.serena

# 설정 파일 복사 및 편집
cp ~/cho/source/serena/src/serena/resources/serena_config.template.yml ~/.serena/serena_config.yml
```

---

## 4️⃣ 플러그인 시스템

### 설치된 플러그인

| 플러그인 | 마켓플레이스 | 버전 | 용도 |
|----------|--------------|------|------|
| `calab-plugin` | calab-marketplace | 1.0.0 | 개발 워크플로우 자동화 (16 스킬 + 23 에이전트 + 24 훅) |
| `gopls-lsp` | claude-plugins-official | 1.0.0 | Go 언어 LSP |

---

### calab-plugin 상세 구조

> **일관된 개발 품질을 보장하는** Claude Code 워크플로우 자동화 플러그인

```
16개 스킬 (10 active + 6 passive) + 23개 에이전트 + 24개 훅
```

#### 스킬 목록 (16개)

**Active 스킬 - Core (3개)**

| 스킬 | 명령어 | 역할 |
|------|--------|------|
| dev | `/dev --plan/--design/--tasks/--build` | 개발 워크플로우 |
| solve | `/solve --5whys/--rca/--hypothesis` | 문제 해결 |
| onboard | `/onboard` | 프로젝트 온보딩 |

**Active 스킬 - Utility (7개)**

| 스킬 | 명령어 | 역할 |
|------|--------|------|
| docs | `/docs --api/--component/--guide` | 문서 생성 |
| security | `/security --owasp/--secrets/--deps` | 보안 검사 |
| research | `/research --deep/--compare` | 웹 리서치 |
| jira | `/jira --sync/--create/--update` | JIRA 연동 |
| refactor | `/refactor --dead-code/--duplicates` | 리팩토링 |
| e2e | `/e2e --run/--debug/--record` | E2E 테스트 |
| guard | `/guard --rules/--context/--full` | 규칙 검증 |

**Passive 스킬 (6개)** - 자동 로드

| 스킬 | 트리거 |
|------|--------|
| best-practices | 기술 키워드 감지 시 |
| code-quality | 코드 생성/수정 시 |
| tdd-workflow | 테스트 키워드 감지 시 |
| project-rules | 모든 코드 작성 시 |
| work-tracker | 소스 파일 수정 시 |
| clarification-protocol | 서브에이전트 실행 시 |

#### 에이전트 목록 (23개)

**워크플로우 에이전트 (7개)**

| 에이전트 | 역할 |
|----------|------|
| dev-workflow | Plan → Design → Tasks → Build 오케스트레이션 |
| planner-phase | PHASE 기반 기획/분석 |
| planner-task | Task 분해 및 AC 정의 |
| design | 아키텍처 + ERD 설계 |
| dev-executor | 실제 코드 구현 |
| project-onboarder | 프로젝트 분석 |
| jira-connector | JIRA 이슈 동기화 |

**문제 해결 에이전트 (2개)**

| 에이전트 | 역할 |
|----------|------|
| root-cause-finder | 5 Whys, RCA 기반 근본 원인 분석 |
| bug-fixer | 버그 수정 및 테스트 작성 |

**리서치 에이전트 (2개)**

| 에이전트 | 역할 |
|----------|------|
| web-researcher | Tavily MCP 기반 실시간 웹 검색 |
| deep-researcher | 검색 결과 종합 분석 및 보고서 |

**품질 에이전트 (4개)**

| 에이전트 | 역할 |
|----------|------|
| code-reviewer | 코드 품질/스타일 검토 |
| security-reviewer | OWASP Top 10 검사 |
| project-guardian | 프로젝트 규칙 준수 검증 |
| build-error-resolver | 빌드/타입 오류 해결 + Circuit Breaker |

**검증/보강 에이전트 (3개)**

| 에이전트 | 역할 |
|----------|------|
| validator | AC/완전성/엣지케이스 검증 |
| task-validator | Task 단위 검증 |
| reinforcer | 검증 실패 항목 자동 수정 |

**자동화/문서화 에이전트 (5개)**

| 에이전트 | 역할 |
|----------|------|
| refactor-cleaner | 데드 코드/미사용 import 정리 |
| e2e-runner | Playwright/Puppeteer 테스트 |
| doc-updater | 코드 변경 감지 문서 업데이트 |
| docs-generator | 문서 자동 생성 |
| qa | 테스트 계획/실행/보고 |

#### 워크플로우

```
/onboard → /dev (plan→design→tasks→build) → QA(자동) → 완료
                                              ↓ 실패
                                           /solve
                                              ↓
                     ┌─────────────────────────┴─────────────────────────┐
                     │ 단순 버그: bug-fixer    │ 복잡한 문제: /dev 재설계  │
                     └─────────────────────────┬─────────────────────────┘
                                              ↓
                                      validator → reinforcer
                                         (최대 2회 재시도)
```

#### 프로젝트 구조

```
calab-claude-plugin/
├── plugins/calab-plugin/
│   ├── .claude-plugin/plugin.json   # 플러그인 메타데이터
│   ├── CLAUDE.md                    # Claude 지침
│   ├── skills/                      # 16개 스킬
│   │   ├── dev/                     # 개발 워크플로우
│   │   ├── solve/                   # 문제 해결
│   │   ├── onboard/                 # 프로젝트 온보딩
│   │   ├── docs/                    # 문서 생성
│   │   ├── security/                # 보안 검사
│   │   ├── research/                # 웹 리서치
│   │   ├── jira/                    # JIRA 연동
│   │   ├── refactor/                # 리팩토링
│   │   ├── e2e/                     # E2E 테스트
│   │   ├── guard/                   # 규칙 검증
│   │   └── (passive skills...)      # 패시브 스킬 6개
│   ├── agents/                      # 23개 에이전트 (.md 파일)
│   └── hooks/                       # 24개 훅 스크립트
```

---

### 마켓플레이스 설정

#### 공식 마켓플레이스
```bash
# 자동으로 등록됨
# 위치: ~/.claude/plugins/marketplaces/claude-plugins-official
# 소스: github:anthropics/claude-plugins-official
```

#### 커스텀 마켓플레이스 (calab-marketplace)
```bash
# 디렉토리 생성
mkdir -p ~/.claude/calab-marketplace/plugins

# 플러그인 심볼릭 링크 (개발용)
ln -s ~/cho/source/calab-claude-plugin ~/.claude/calab-marketplace/plugins/calab-plugin

# 또는 마켓플레이스 추가 (GitHub에서)
/plugin marketplace add Wondermove-Inc/calab-claude-plugin
```

### 플러그인 활성화
```bash
# Claude Code에서 실행
/plugins install calab-plugin@calab-marketplace
/plugins install gopls-lsp@claude-plugins-official
```

### 스킬 자동완성 설정 (선택)
```bash
# 플러그인 디렉토리에서 실행
cd ~/cho/source/calab-claude-plugin
./link-skills.sh

# 결과: ~/.claude/skills/calab-* 심볼릭 링크 생성
# /calab-dev, /calab-solve, /calab-docs 등 자동완성 가능
```

---

## 5️⃣ SuperClaude 프레임워크

### 설치 방법
SuperClaude 프레임워크 파일들을 `~/.claude/` 디렉토리에 복사:

```
CLAUDE.md          # 메인 진입점
COMMANDS.md        # 17개 커맨드 정의
FLAGS.md           # 플래그 시스템
PRINCIPLES.md      # 개발 원칙
RULES.md           # 운영 규칙
MCP.md             # MCP 통합
PERSONAS.md        # 11개 페르소나
ORCHESTRATOR.md    # 라우팅
MODES.md           # 운영 모드
```

### 사용 가능한 스킬 커맨드 (17개)

| 커맨드 | 용도 |
|--------|------|
| `/sc:analyze` | 다차원 코드/시스템 분석 |
| `/sc:build` | 프로젝트 빌드 |
| `/sc:cleanup` | 프로젝트 정리, 기술 부채 감소 |
| `/sc:design` | 시스템 설계 |
| `/sc:document` | 문서화 |
| `/sc:estimate` | 개발 기간 추정 |
| `/sc:explain` | 코드/개념 설명 |
| `/sc:git` | Git 워크플로우 |
| `/sc:implement` | 기능 구현 |
| `/sc:improve` | 코드 개선 |
| `/sc:index` | 프로젝트 인덱싱 |
| `/sc:load` | 프로젝트 컨텍스트 로드 |
| `/sc:spawn` | 작업 분산 |
| `/sc:task` | 장기 프로젝트 관리 |
| `/sc:test` | 테스트 워크플로우 |
| `/sc:troubleshoot` | 문제 진단 |
| `/sc:workflow` | 워크플로우 생성 |

### 페르소나 시스템 (11개)

```bash
--persona-architect     # 시스템 아키텍처
--persona-frontend      # UI/UX 전문가
--persona-backend       # 서버/API 전문가
--persona-analyzer      # 근본 원인 분석
--persona-security      # 보안 전문가
--persona-mentor        # 지식 전달
--persona-refactorer    # 코드 품질
--persona-performance   # 성능 최적화
--persona-qa            # 품질 보증
--persona-devops        # 인프라/배포
--persona-scribe=lang   # 문서화 (en, ko 등)
```

### 주요 플래그

```bash
# 사고 깊이
--think              # 4K 토큰 분석
--think-hard         # 10K 토큰 분석
--ultrathink         # 32K 토큰 분석

# MCP 서버
--c7 / --context7    # Context7 활성화
--seq / --sequential # Sequential 활성화
--tavily / --tv      # Tavily 활성화
--all-mcp            # 모든 MCP 활성화
--no-mcp             # MCP 비활성화

# 효율성
--uc                 # 압축 모드 (30-50% 토큰 감소)
--validate           # 검증 모드
--safe-mode          # 안전 모드
```

---

## 6️⃣ 상태바 커스텀 스크립트

### 위치: `~/.claude/statusline-command.sh`

기능:
- 모델명 표시 (Opus/Sonnet/Haiku)
- 컨텍스트 윈도우 사용량 바
- 토큰 사용량 바 및 예산
- Git 브랜치 및 상태
- 리셋 타이머

### 의존성
```bash
# jq 설치 필요
brew install jq
```

---

## 7️⃣ 권한 설정

### 위치: `~/.claude/settings.local.json`

```json
{
  "permissions": {
    "allow": [
      "Bash(kubectl:*)",
      "Bash(aws:*)",
      "Bash(curl:*)",
      "Bash(python3:*)",
      "Bash(go test:*)",
      "mcp__serena__list_dir",
      "mcp__serena__find_file",
      "mcp__serena__find_symbol",
      "mcp__serena__get_symbols_overview",
      "mcp__serena__search_for_pattern",
      "mcp__tavily__tavily_search",
      "WebSearch"
    ],
    "deny": []
  }
}
```

---

## 8️⃣ 빠른 설치 체크리스트

```bash
# 1. Claude Code 설치
curl -fsSL https://claude.ai/install.sh | sh

# 2. 필수 의존성
brew install jq

# 3. 디렉토리 구조 생성
mkdir -p ~/.claude/{commands/sc,plugins,calab-marketplace/plugins}
mkdir -p ~/.serena

# 4. SuperClaude 프레임워크 파일 복사
# (CLAUDE.md, COMMANDS.md, FLAGS.md 등)

# 5. MCP 설정 파일 생성
# ~/Library/Application Support/Claude/claude_desktop_config.json

# 6. Serena 설정
# ~/.serena/serena_config.yml

# 7. 상태바 스크립트 복사 및 권한 설정
chmod +x ~/.claude/statusline-command.sh

# 8. 플러그인 설치
# Claude Code 내에서 /plugins 명령 사용

# 9. 환경 변수 설정 (필요시)
# NOTION_API_TOKEN, TAVILY_API_KEY 등
```

---

## 📌 참고 사항

1. **API 키 보안**: 실제 환경에서는 API 키를 환경 변수로 관리
2. **Serena 프로젝트**: 사용할 프로젝트를 `~/.serena/serena_config.yml`에 등록 필요
3. **SuperClaude**: 프레임워크 파일은 별도 저장소에서 관리 권장
4. **플러그인**: 커스텀 플러그인은 심볼릭 링크로 개발 편의성 유지

---

## 🔗 관련 리소스

- Claude Code 공식 문서: https://docs.anthropic.com/claude-code
- MCP 프로토콜: https://modelcontextprotocol.io
- Serena: https://github.com/oraios/serena
- Tavily: https://tavily.com
- Context7: https://context7.com
