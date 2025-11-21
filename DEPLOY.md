# PyPI 배포 가이드

이 문서는 `rokrokss-mcp-atlassian` 패키지를 PyPI에 배포하는 방법을 설명합니다.

## 사전 준비

### 1. PyPI 계정 생성
1. [PyPI](https://pypi.org/account/register/)에서 계정 생성
2. [TestPyPI](https://test.pypi.org/account/register/)에서도 계정 생성 (테스트용)
3. 이메일 인증 완료

### 2. PyPI API 토큰 생성

#### 방법 1: API Token 사용 (간단)

1. PyPI에 로그인 → Account Settings → API tokens
2. "Add API token" 클릭
3. Token name: `rokrokss-mcp-atlassian-deploy`
4. Scope: "Entire account" (처음 배포) 또는 "Project: rokrokss-mcp-atlassian" (이후)
5. 토큰 복사 (다시 볼 수 없으므로 안전하게 보관!)

#### 방법 2: Trusted Publishing (권장 - GitHub Actions용)

1. PyPI에 로그인
2. Publishing → Add a new pending publisher
3. 설정:
   - PyPI Project Name: `rokrokss-mcp-atlassian`
   - Owner: `rokrokss` (GitHub 사용자명)
   - Repository name: `mcp-atlassian` (또는 실제 저장소 이름)
   - Workflow name: `publish.yml`
   - Environment name: `pypi`

### 3. GitHub Secrets 설정 (API Token 방식)

GitHub 저장소 → Settings → Secrets and variables → Actions

**Secret 추가:**
- Name: `PYPI_API_TOKEN`
- Value: (위에서 생성한 PyPI API 토큰)

## 로컬에서 수동 배포

### 1. 빌드

```bash
# 의존성 설치
uv sync --frozen --all-extras --dev

# 패키지 빌드
uv build
```

빌드 완료 후 `dist/` 디렉토리 확인:
- `rokrokss_mcp_atlassian-X.Y.Z-py3-none-any.whl`
- `rokrokss_mcp_atlassian-X.Y.Z.tar.gz`

### 2. TestPyPI에 먼저 배포 (권장)

```bash
# TestPyPI에 업로드
uv publish --publish-url https://test.pypi.org/legacy/ --token <YOUR_TEST_PYPI_TOKEN>

# TestPyPI에서 설치 테스트
uvx --index-url https://test.pypi.org/simple/ --extra-index-url https://pypi.org/simple/ rokrokss-mcp-atlassian
```

### 3. PyPI에 배포

```bash
# PyPI에 업로드
uv publish --token <YOUR_PYPI_TOKEN>

# 또는 Trusted Publishing 사용 시
uv publish
```

### 4. 설치 테스트

```bash
# PyPI에서 설치
uvx rokrokss-mcp-atlassian --version

# 또는 pip
pip install rokrokss-mcp-atlassian
```

## GitHub Actions로 자동 배포

### Trusted Publishing 방식 (권장)

1. PyPI에서 Trusted Publishing 설정 (위 "방법 2" 참고)
2. `.github/workflows/publish.yml`에서 토큰 라인 제거:
   ```yaml
   - name: Publish package to PyPI
     run: uv publish dist/*  # --token 제거
   ```

### API Token 방식

1. GitHub Secrets에 `PYPI_API_TOKEN` 추가 (위 참고)
2. `.github/workflows/publish.yml`은 이미 설정됨

### 릴리스 생성하여 배포

```bash
# Git 태그 생성
git tag v0.1.0
git push origin v0.1.0

# GitHub에서 Release 생성
# 1. Releases → Draft a new release
# 2. Tag: v0.1.0 선택
# 3. Release title: v0.1.0
# 4. Description: 변경사항 작성
# 5. Publish release 클릭

# 자동으로 GitHub Actions가 PyPI에 배포합니다
```

## 버전 관리

이 프로젝트는 `uv-dynamic-versioning`을 사용하여 Git 태그에서 자동으로 버전을 생성합니다.

### 버전 넘버링 (Semantic Versioning)

- **MAJOR** (1.0.0): 호환되지 않는 API 변경
- **MINOR** (0.1.0): 호환되는 기능 추가
- **PATCH** (0.0.1): 호환되는 버그 수정

### 태그 생성 예시

```bash
# 첫 릴리스
git tag v0.1.0

# 버그 수정
git tag v0.1.1

# 새 기능 추가
git tag v0.2.0

# 큰 변경사항
git tag v1.0.0

# 태그 푸시
git push origin --tags
```

## 문제 해결

### "Package already exists" 오류

PyPI는 같은 버전을 다시 업로드할 수 없습니다. 새 Git 태그를 만들어야 합니다.

```bash
git tag v0.1.1  # 버전 증가
git push origin v0.1.1
```

### "Invalid credentials" 오류

1. API 토큰이 올바른지 확인
2. GitHub Secrets가 정확히 설정되었는지 확인
3. Trusted Publishing의 경우 설정이 정확한지 확인

### 빌드 실패

```bash
# 캐시 정리 후 재시도
rm -rf dist/ build/ *.egg-info
uv build
```

## 배포 후 확인

1. PyPI 페이지 확인: https://pypi.org/project/rokrokss-mcp-atlassian/
2. 설치 테스트:
   ```bash
   uvx rokrokss-mcp-atlassian --version
   ```
3. README가 PyPI에 잘 표시되는지 확인

## 참고 자료

- [PyPI Documentation](https://packaging.python.org/tutorials/packaging-projects/)
- [uv Documentation](https://docs.astral.sh/uv/)
- [Trusted Publishing Guide](https://docs.pypi.org/trusted-publishers/)
