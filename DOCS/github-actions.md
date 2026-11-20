# GitHub 이전 및 GitHub Actions CI

2026-11-20, 저장소를 GitLab(`lab.ssafy.com/s15-final/S15P31D107`)에서 GitHub(`github.com/cmy397264/NEXA`)로 옮겼다.
목적은 **기록 보관과 이후 유지보수**이다. 그래서 `lab.ssafy.com` 에 접근할 수 없어도 GitHub 저장소만으로 빌드·테스트가 되게 했다.

## 저장소 이전

- GitLab 서버의 브랜치 4개(`develop`, `master`, `feature/frontend-p1`, `feature/gpu-server`)와 태그 2개(`nexa-security-v1.0.0`, `v1.1.0`)를 히스토리째 옮겼다.
- GitLab 전용 참조(`refs/merge-requests/*`, `refs/environments/*`)는 옮기지 않았다.
- MR·이슈·위키는 옮기지 않았다.
- 기존 clone 은 remote 만 바꾸면 된다.
  ```bash
  git remote set-url origin https://github.com/cmy397264/NEXA.git
  git fetch origin --prune
  ```

## 바뀐 것

### 1. nexa-security: 레지스트리 → 소스 직접 참조

이전에는 approval·attendance·leave·migration·payroll 서비스가 `nexa-security` 를 GitLab Package Registry 에서 받았다.
GitLab 접근이 끊기면 빌드가 깨지므로, 저장소 안의 소스를 Gradle composite build(`includeBuild`)로 직접 빌드하게 바꿨다.
GitHub Packages 는 public 이어도 Maven 패키지를 받을 때 토큰이 필요해서 쓰지 않았다.

| 파일 | 변경 |
|---|---|
| `Backend/<5개 서비스>/settings.gradle` | `includeBuild '../nexa-security'` 추가 |
| `Backend/<5개 서비스>/build.gradle` | GitLab 레지스트리 `maven { url ... }` 제거. 의존성 선언 `com.ssafy.nexa:nexa-security:1.1.0` 은 그대로(자동 대체됨) |
| `Backend/<5개 서비스>/Dockerfile` | 빌드 단계에 `COPY --from=nexa-security . /nexa-security` 추가 |
| `infra/compose/backend.yml` | 5개 서비스 `build` 에 `additional_contexts: nexa-security: ./Backend/nexa-security` 추가 |
| `Backend/nexa-security/.dockerignore` | 추가 (`build`, `.gradle` 등을 Docker 빌드에 보내지 않음) |

- 서비스별 Docker build context 는 그대로(`./Backend/<service>`)이고, `additional_contexts` 로 nexa-security 폴더만 추가로 넘긴다. Docker Compose 2.17 이상(BuildKit)이 필요하다.
- 이제 nexa-security 를 고치면 다섯 서비스에 바로 반영된다. 버전 올리기·태그 publish 절차는 없어졌다 ([nexa-security.md](nexa-security.md)).
- dev 서버 배포용 `infra/docker/Dockerfile.ci` 는 CI 가 만든 JAR 만 복사하므로 영향 없다.

### 2. CI: GitLab CI → GitHub Actions

`.github/workflows/ci.yml` 하나에 GitLab CI 의 빌드·테스트 잡을 옮겼다.

| 트리거 | 동작 |
|---|---|
| Pull Request | 바뀐 경로의 잡만 실행 |
| `develop` push | 바뀐 경로의 잡만 실행 |
| 수동 실행(Actions 탭 → CI → Run workflow) | 전부 실행 |

| 잡 | 내용 | GitLab 대응 |
|---|---|---|
| `changes` | `dorny/paths-filter` 로 바뀐 영역 판별 | 각 잡의 `rules: changes:` |
| `backend (<서비스>)` | 서비스별 `gradlew test bootJar` (matrix) | `build-<서비스>`, `test-nexa-security` |
| `fastapi` | postgres 17 서비스 + 의존성 설치 + `/health` 확인 | `verify-fastapi` |
| `frontend` | `npm ci && npm run build` | `build-frontend` |

- 서비스별 테스트 환경변수(`EUREKA_CLIENT_ENABLED`, `CONFIG_SERVER_PASSWORD`, `SPRING_CONFIG_LOCATION` 등)는 GitLab 잡과 같다.
- `leave-service` 는 GitLab 과 같이 테스트를 생략하고 `bootJar` 만 한다.
- `Backend/nexa-security/**` 가 바뀌면 nexa-security 와 사용하는 서비스 5개가 함께 돈다.
- Testcontainers 는 GitHub 러너의 Docker 를 그대로 쓴다(GitLab 의 dind 서비스 불필요).
- GitLab 은 `develop` push 때 테스트를 생략했지만(배포 속도용), 배포 잡이 없으므로 GitHub 에서는 항상 테스트한다.
- 테스트가 실패하면 리포트가 `test-reports-<서비스>` 아티팩트로 7일 보관된다.
- MR 템플릿은 `.github/pull_request_template.md` 로 복사했다.

## 옮기지 않은 것

| 항목 | 이유 / 다시 하려면 |
|---|---|
| dev 서버 배포 (`deploy-*-dev`, `.gitlab/ci/deploy.yml`) | dev 서버 안의 GitLab Runner(`nexa-dev-deploy`)가 `/opt/nexa-dev/.env` 로 `docker compose up` 하는 구조. 다시 하려면 서버에 GitHub self-hosted runner 를 설치하고 `deploy.yml` 스크립트를 그대로 옮기거나, SSH 키를 Secrets 에 넣고 SSH 로 배포 |
| `publish-nexa-security` | 레지스트리를 쓰지 않게 되어 불필요 |
| `check-ci-environment` | GitLab Runner 확인용 수동 잡 |

`.gitlab-ci.yml` 과 `.gitlab/` 은 기록용으로 남겨뒀다. GitHub 에서는 실행되지 않는다.
단, `.gitlab/ci/*.yml` 의 백엔드 잡은 Docker 빌드가 아니라 gradle 로 빌드하므로 includeBuild 변경 후에도 동작한다.
