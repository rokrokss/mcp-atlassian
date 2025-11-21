# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

MCP Atlassian은 Atlassian 제품(Jira, Confluence)과 AI 모델을 연결하는 Model Context Protocol (MCP) 서버입니다. FastMCP 프레임워크를 기반으로 하며, Cloud 및 Server/Data Center 배포를 모두 지원합니다.

- **언어**: Python 3.10+
- **패키지 관리자**: uv (pip 사용 금지)
- **프레임워크**: FastMCP, atlassian-python-api
- **배포**: Docker 이미지
- **인증**: API Token, Personal Access Token (PAT), OAuth 2.0

## Core Architecture

### 디렉토리 구조

```
src/mcp_atlassian/
├── jira/           # Jira 클라이언트 및 믹스인 (issues, search, boards, sprints 등)
├── confluence/     # Confluence 클라이언트 및 믹스인 (pages, search, spaces 등)
├── models/         # Pydantic 데이터 모델 (jira/, confluence/)
├── servers/        # FastMCP 서버 구현 (main.py, context.py, dependencies.py)
├── preprocessing/  # 요청/응답 전처리 로직
└── utils/          # 공유 유틸리티 (auth, logging, SSL, OAuth)
```

### 설계 패턴

1. **Mixin 아키텍처**: 클라이언트 기능은 focused mixins으로 분리됨
   - 예: `JiraFetcher`는 `IssuesMixin`, `SearchMixin`, `BoardsMixin` 등을 상속
   - 각 믹스인은 관련된 작업 그룹만 처리

2. **도구 명명 규칙**: `{service}_{action}` 형식
   - 예: `jira_create_issue`, `confluence_search`, `jira_get_board_issues`

3. **모델 계층**: 모든 API 응답 모델은 `ApiModel` 베이스 클래스 확장
   - `models/jira/`: Issue, Project, Board, Sprint 등
   - `models/confluence/`: Page, Space, Comment, Label 등

4. **설정 관리**:
   - `JiraConfig`, `ConfluenceConfig`: 환경 변수에서 로드
   - 다중 인증 방법 자동 감지 (API Token > PAT > OAuth)

5. **전송 계층**: 3가지 전송 방식 지원
   - `stdio`: 기본, IDE 통합용
   - `sse`: Server-Sent Events (HTTP)
   - `streamable-http`: HTTP 스트리밍, 멀티 테넌트 지원

## Development Commands

### 의존성 관리
```bash
# 의존성 설치 (필수)
uv sync --frozen --all-extras --dev

# 가상환경 활성화
source .venv/bin/activate  # macOS/Linux
.venv\Scripts\activate.ps1  # Windows
```

### 로컬 실행 (개발 외 사용자용)
프로젝트를 clone하지 않고 PyPI에서 직접 실행:
```bash
# uvx 사용 (npx와 유사)
uvx rokrokss-mcp-atlassian

# 환경 변수와 함께 실행
JIRA_URL=https://company.atlassian.net \
JIRA_USERNAME=user@company.com \
JIRA_API_TOKEN=token \
uvx rokrokss-mcp-atlassian

# OAuth 설정
uvx rokrokss-mcp-atlassian --oauth-setup
```

### 테스트
```bash
# 전체 테스트 실행
uv run pytest

# 커버리지 포함
uv run pytest --cov=mcp_atlassian

# 특정 테스트 파일/함수 실행
uv run pytest tests/unit/jira/test_issues.py
uv run pytest tests/unit/jira/test_issues.py::test_create_issue

# 통합 테스트 (실제 API 필요)
uv run pytest tests/integration/
```

### 코드 품질
```bash
# Pre-commit hooks 설치 (최초 1회)
pre-commit install

# 모든 품질 검사 실행 (커밋 전 필수)
pre-commit run --all-files
# 포함: ruff (lint/format), pyright (type check), prettier (YAML/JSON)
```

### 서버 실행
```bash
# stdio 모드 (기본)
uv run mcp-atlassian

# 상세 로깅
uv run mcp-atlassian -v    # INFO
uv run mcp-atlassian -vv   # DEBUG

# OAuth 설정 마법사
uv run mcp-atlassian --oauth-setup

# HTTP 전송
uv run mcp-atlassian --transport sse --port 9000
uv run mcp-atlassian --transport streamable-http --port 9000

# 환경 파일 지정
uv run mcp-atlassian --env-file .env.production
```

### MCP Inspector로 테스트
```bash
# 로컬 개발 버전 테스트
npx @modelcontextprotocol/inspector uv --directory /path/to/mcp-atlassian run mcp-atlassian
```

## Code Style & Conventions

### 코드 스타일
- **Line length**: 88자 최대 (ruff enforced)
- **Imports**: 절대 경로, ruff가 자동 정렬
- **Naming**:
  - 함수/변수: `snake_case`
  - 클래스: `PascalCase`
  - 상수: `UPPER_SNAKE_CASE`
- **Type hints**: 모든 함수에 필수
  - `type[T]` for class types
  - Union: `str | None` (not `Optional[str]`)
  - Collections: `list[str]`, `dict[str, Any]`

### Docstrings
Google 스타일 필수 (public API):
```python
def create_issue(self, project_key: str, summary: str) -> JiraIssue:
    """Jira 이슈를 생성합니다.

    Args:
        project_key: 프로젝트 키 (예: "PROJ")
        summary: 이슈 요약

    Returns:
        생성된 이슈 객체

    Raises:
        AuthenticationError: 인증 실패 시
        ValidationError: 필수 필드 누락 시
    """
```

### 타입 체킹
- Pyright 사용 (mypy 아님)
- `pyproject.toml`의 `[tool.mypy]` 설정은 레거시
- 테스트 파일은 타입 체킹 완화 (`tests/**/*.py` 제외)

## Key Implementation Patterns

### 클라이언트 초기화
서버는 `MainAppContext`를 통해 클라이언트를 관리합니다:
```python
# servers/context.py
app_context = MainAppContext(
    full_jira_config=jira_config,
    full_confluence_config=confluence_config,
    read_only=read_only,
    enabled_tools=enabled_tools,
)
```

HTTP 전송에서는 요청별 인증 헤더 지원:
- `Authorization: Bearer <oauth_token>` (Cloud OAuth)
- `Authorization: Token <pat>` (Server/DC)
- `X-Atlassian-Cloud-Id: <cloud_id>` (멀티 클라우드)

### 도구 필터링
`ENABLED_TOOLS` 환경 변수 또는 `--enabled-tools` 플래그로 제어:
```python
# utils/tools.py
if not should_include_tool(tool_name, enabled_tools, read_only):
    continue  # 도구 제외
```

Read-only 모드는 write 도구를 자동 비활성화합니다.

### 에러 처리
`exceptions.py`에서 커스텀 예외 사용:
- 일반적인 `Exception` 대신 구체적인 예외 던지기
- FastMCP가 자동으로 MCP 에러 형식으로 변환

## Testing Conventions

### 테스트 구조
```
tests/
├── unit/               # 단위 테스트 (mocking)
│   ├── jira/
│   ├── confluence/
│   ├── models/
│   ├── servers/
│   └── utils/
├── integration/        # 통합 테스트 (실제 API 또는 복잡한 시나리오)
└── fixtures/           # 공유 mock 데이터
```

### Fixtures
- `tests/conftest.py`: 전역 fixtures
- `tests/fixtures/`: Jira/Confluence mock 데이터
- `tests/unit/{service}/conftest.py`: 서비스별 fixtures

### 실제 API 테스트
통합 테스트는 환경 변수 필요:
```bash
# .env 파일 설정 후
uv run pytest tests/integration/test_real_api.py
```

## Git Workflow

### 브랜칭
```bash
# NEVER work on main
git checkout -b feature/new-jira-tool
git checkout -b fix/confluence-search-bug
```

### 커밋 메시지
```bash
# Clear, concise messages
git commit -m "feat: add jira_batch_get_changelogs tool"
git commit -m "fix: handle null values in Confluence page body"

# Attribution (선택 사항)
git commit --trailer "Reported-by: User Name <email>"
git commit --trailer "Github-Issue: #123"
```

### Pre-commit hooks
커밋 전 자동 실행:
- `ruff check --fix`: 자동 lint 수정
- `ruff format`: 코드 포맷팅
- `pyright`: 타입 체킹
- `prettier`: YAML/JSON 포맷팅
- Trailing whitespace, file endings, YAML/TOML 유효성 검사

## Environment Variables

주요 환경 변수 (`.env.example` 참조):

**필수**:
- `JIRA_URL`, `CONFLUENCE_URL`
- 인증: `{SERVICE}_USERNAME` + `{SERVICE}_API_TOKEN` 또는 `{SERVICE}_PERSONAL_TOKEN`

**선택 사항**:
- `READ_ONLY_MODE=true`: 쓰기 작업 비활성화
- `ENABLED_TOOLS=confluence_search,jira_get_issue`: 특정 도구만 활성화
- `CONFLUENCE_SPACES_FILTER=DEV,TEAM`: Space 필터링
- `JIRA_PROJECTS_FILTER=PROJ,DEV`: 프로젝트 필터링
- `MCP_VERBOSE=true`, `MCP_VERY_VERBOSE=true`: 로깅 레벨
- `MCP_LOGGING_STDOUT=true`: stdout으로 로그 출력
- `{SERVICE}_SSL_VERIFY=false`: SSL 검증 비활성화 (Server/DC 자체 서명 인증서)
- `{SERVICE}_CUSTOM_HEADERS=X-Custom=value`: 커스텀 HTTP 헤더

**OAuth (Cloud 전용)**:
- `ATLASSIAN_OAUTH_CLIENT_ID`, `ATLASSIAN_OAUTH_CLIENT_SECRET`
- `ATLASSIAN_OAUTH_REDIRECT_URI`, `ATLASSIAN_OAUTH_SCOPE`
- `ATLASSIAN_OAUTH_CLOUD_ID`
- `ATLASSIAN_OAUTH_ACCESS_TOKEN`: BYOT (Bring Your Own Token) 모드

**프록시**:
- `HTTP_PROXY`, `HTTPS_PROXY`, `SOCKS_PROXY`, `NO_PROXY`
- 서비스별: `JIRA_HTTPS_PROXY`, `CONFLUENCE_NO_PROXY`

## Common Gotchas

1. **패키지 관리**: 항상 `uv` 사용, `pip install` 절대 금지
2. **타입 체킹**: Pyright 사용 (mypy 설정은 무시)
3. **라인 길이**: 88자 엄격히 준수 (ruff가 자동 검사)
4. **OAuth 토큰 캐싱**: `~/.mcp-atlassian/` 디렉토리에 저장 (Docker 볼륨 마운트 필요)
5. **Server/DC vs Cloud**:
   - Server/DC: PAT 또는 username/password
   - Cloud: API Token (권장) 또는 OAuth
6. **읽기 전용 모드**: `READ_ONLY_MODE=true` 시 create/update/delete 도구 자동 비활성화
7. **테스트**: 항상 `uv run pytest` 사용 (직접 pytest 호출하면 의존성 문제 발생 가능)

## Release Process

Semantic versioning:
- **MAJOR**: 호환되지 않는 API 변경
- **MINOR**: 호환되는 기능 추가
- **PATCH**: 호환되는 버그 수정

버전은 `uv-dynamic-versioning`이 Git 태그에서 자동 생성합니다.
