# Snowflake, GitHub, OpenAI Codex 및 GitHub Actions 연동 가이드

이 가이드는 Snowflake 개발 환경을 GitHub, Snowflake CLI, OpenAI Codex, GitHub Actions와 연동하는 방법을 설명합니다. 서로 다른 인증 경로, Snowflake Git Repository Clone, GitHub 변경 사항을 가져오는 최소 권한 워크플로를 다룹니다.

> **범위 및 보안:** 예시에는 자리표시자만 사용합니다. 개인 키, 암호문구, 토큰 및 기타 인증 정보를 커밋하지 마세요. 지원되는 경우 Snowflake GitHub App 또는 GitHub Actions OIDC를 사용하고, 필요한 경우에만 범위를 제한한 토큰이나 암호화된 키를 사용하세요. 일반 자동화에 `ACCOUNTADMIN`을 사용하지 마세요.

## 1. 구성 요소 이해하기

### OpenAI Codex와 Snowflake CoCo

- **OpenAI Codex**는 OpenAI 코딩 에이전트입니다. Codex CLI는 로컬 프로젝트 파일을 다룰 수 있고, 도구를 구성하고 실행을 승인하면 `snow` 같은 도구도 사용할 수 있습니다. 프로젝트의 `AGENTS.md` 파일로 저장소별 지침을 제공할 수 있지만, 이 파일이 Snowflake 접근 권한이나 인증 정보를 설정해 주지는 않습니다.
- **Snowflake CoCo**(Snowflake Cortex Code 문서에 설명된 제품)는 Snowflake의 AI 에이전트 환경입니다. Snowflake는 CoCo를 Snowsight, 데스크톱 IDE, CLI로 제공합니다. 구성된 도구와 사용자의 Snowflake 역할에 따라 Snowflake 연결 및 로컬 파일을 사용할 수 있습니다.
- 두 제품은 별개입니다. Codex를 Snowflake에 연결해도 Snowflake CoCo로 바뀌지 않으며, Snowflake CoCo가 Codex를 설정하지도 않습니다.

### 서로 독립된 두 가지 인증 경로

1. **GitHub Actions → Snowflake:** GitHub OIDC 워크로드 ID 연합(WIF)을 우선 사용합니다. GitHub Secrets에 장기 Snowflake 개인 키를 보관할 필요가 없습니다.
2. **Snowflake → GitHub:** Snowflake Git Repository Clone이 원격 Git 공급자에 인증합니다. GitHub.com에서는 적합한 경우 Snowflake GitHub App OAuth 흐름을 우선 사용합니다. 토큰 기반 Git Integration은 대안이며, 자격 증명은 저장소나 Workflow 파일이 아니라 Snowflake Secret에 보관합니다.

`ALTER GIT REPOSITORY ... FETCH`를 실행하는 Actions 작업은 원격 저장소에서 Snowflake Clone만 갱신합니다. 저장소의 SQL을 배포하거나 실행하지 않습니다. CI 파이프라인이 Actions Runner에서 파일을 체크아웃하고 배포하기만 한다면 Snowflake Git Clone은 선택 사항입니다.

## 2. Snowflake CLI 인증 방식 선택

Snowflake는 비밀번호 인증의 대안으로 키페어 인증을 지원합니다. MFA가 모든 상황에서 비밀번호 인증을 실패하게 만든다고 단정하지 마세요. MFA 요구사항은 계정 인증 정책과 클라이언트 흐름에 따라 달라집니다. 비대화형 워크로드에는 사용자 비밀번호 대신 WIF, OAuth, 키페어 인증처럼 지원되는 서비스 인증 방식을 사용하세요.

이 가이드의 GitHub Actions 예제는 WIF를 사용합니다. 로컬 Snowflake CLI 연결에는 지원되는 키페어 인증을 사용할 수 있습니다. Snowflake CLI는 암호화된 RSA 개인 키를 지원합니다. 가능한 경우 워크로드별로 별도의 키 또는 ID를 사용하고, 더 이상 필요하지 않은 키는 교체하거나 제거하세요.

### 로컬 암호화 RSA 키페어 만들기

아래 명령은 POSIX 호환 셸과 OpenSSL을 사용합니다(macOS, Linux, Git Bash 또는 WSL). Windows 기본 셸 사용자는 승인된 OpenSSL 설치를 이용하고 Windows ACL로 개인 키 파일을 보호하세요.

```bash
umask 077
mkdir -p ~/.ssh/snowflake

# 암호문구 입력을 요청하고 암호화된 PKCS#8 개인 키를 생성합니다.
openssl genrsa 2048 | openssl pkcs8 -topk8 -v2 aes-256-cbc \
  -inform PEM -out ~/.ssh/snowflake/rsa_key.p8

# 개인 키 암호문구를 요청하고 공개 키를 생성합니다.
openssl rsa -in ~/.ssh/snowflake/rsa_key.p8 \
  -pubout -out ~/.ssh/snowflake/rsa_key.pub

chmod 700 ~/.ssh/snowflake
chmod 600 ~/.ssh/snowflake/rsa_key.p8
```

개인 키와 암호문구는 서로 분리된 안전한 위치에 보관하세요. 로그에 개인 키를 출력하거나 두 파일을 커밋하지 마세요. 공개 키는 비밀 정보가 아닙니다.

### Snowflake 사용자에게 공개 키 등록

권한이 있는 관리 역할로 Snowflake 사용자에게 **이름이 지정된 키페어**를 등록하세요. Snowflake는 키 관리, 만료, 역할 제한을 지원하는 이름 지정 키페어를 권장합니다.

PEM 구분 줄을 제외한 공개 키 본문을 추출합니다.

```bash
grep -v "BEGIN PUBLIC KEY" ~/.ssh/snowflake/rsa_key.pub \
  | grep -v "END PUBLIC KEY" | tr -d '\n'
```

SQL 워크시트에서 자리표시자를 공개 키 값으로 바꾸어 등록하세요. 개인 키나 암호문구를 붙여 넣지 마세요.

```sql
ALTER USER <snowflake_user>
  ADD KEY PAIR <key_pair_name>
  PUBLIC_KEY = '<public_key_body>';
```

현재 Snowflake 구문에 따라 `SHOW USER KEY PAIRS`로 이름 지정 키페어를 확인하세요. 기존 `RSA_PUBLIC_KEY` 속성을 사용하는 경우 `DESC USER`에 지문이 표시됩니다. 불일치 문제를 해결할 때 지문을 비교하세요.

## 3. Snowflake와 GitHub 저장소 연결

Snowflake는 원격 HTTPS 저장소에서 가져올 수 있는 Git Repository Clone을 만듭니다. 이 Clone은 로컬 Git 체크아웃 및 GitHub Actions 인증과 별개입니다.

### 권장: Snowflake GitHub App OAuth

GitHub.com에는 Snowflake가 사전 구성한 GitHub App 흐름이 제공됩니다. 공식 예제는 `https://github.com` Prefix를 허용하며, GitHub App 인증에서 접근 가능한 저장소를 별도로 승인합니다.

```sql
CREATE API INTEGRATION <git_api_integration>
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com')
  API_USER_AUTHENTICATION = (TYPE = SNOWFLAKE_GITHUB_APP)
  ENABLED = TRUE;
```

관련 Snowflake 인터페이스에서 Snowflake GitHub App 인증을 완료한 뒤, 저장소의 HTTPS URL로 Clone을 만듭니다.

```sql
CREATE GIT REPOSITORY <database>.<schema>.<repository_object>
  API_INTEGRATION = <git_api_integration>
  ORIGIN = 'https://github.com/<github_owner>/<repository>.git';

ALTER GIT REPOSITORY <database>.<schema>.<repository_object> FETCH;
SHOW GIT BRANCHES IN <database>.<schema>.<repository_object>;
```

Snowflake GitHub App 흐름은 표준 GitHub Enterprise Cloud 조직을 포함한 GitHub.com용으로 문서화되어 있습니다. 데이터 레지던시를 사용하는 GitHub Enterprise Cloud 및 GitHub Enterprise Server에는 다른 OAuth 설정이 필요합니다. 구성 전에 현재 Snowflake 지원 여부를 확인하세요.

### 대안: 토큰 기반 Git Integration

GitHub App OAuth 흐름을 사용할 수 없는 경우에만 사용하세요. GitHub는 가능한 경우 Classic PAT보다 fine-grained PAT 사용을 권장합니다. 토큰을 단일 저장소로 제한하고, 소스 가져오기에 필요한 읽기 권한만 부여합니다(예: Contents: read). 만료일을 설정하고 조직의 승인 정책을 따르세요. 필요한 GitHub 통합이 fine-grained 토큰을 지원하지 않는다면 승인된 대안 중 범위가 가장 좁은 방법을 사용하고 교체 계획을 세우세요.

승인된 보안 관리 절차를 통해 Snowflake Secret을 만듭니다. 실제 토큰을 체크인되는 SQL 파일, Workflow, 터미널 기록, 채팅에 넣지 마세요. 아래는 구문 형태만 보여주는 예시입니다.

```sql
CREATE SECRET <database>.<schema>.<github_secret>
  TYPE = password
  USERNAME = '<github_username>'
  PASSWORD = '<승인된 보안 절차를 통해 제공하는 토큰>';
```

API Integration의 허용 URL Prefix를 필요한 GitHub 소유자로 제한하고 해당 Secret만 허용합니다.

```sql
CREATE API INTEGRATION <git_api_integration>
  API_PROVIDER = git_https_api
  API_ALLOWED_PREFIXES = ('https://github.com/<github_owner>')
  ALLOWED_AUTHENTICATION_SECRETS = (<database>.<schema>.<github_secret>)
  ENABLED = TRUE;
```

`GIT_CREDENTIALS = <database>.<schema>.<github_secret>`을 지정하여 Git Repository Clone을 만듭니다. Secret은 Integration의 허용 인증 Secret 목록에 있어야 합니다. 최초 생성에는 `CREATE OR REPLACE` 대신 `CREATE`를 사용하세요. 기존 객체를 바꾸면 동작 중인 Integration에 영향을 줄 수 있습니다.

### 최소 권한 부여

적절한 Snowflake 관리자에게 API Integration을 만들고 Clone 생성 또는 Fetch에 필요한 권한만 부여하도록 요청하세요. 설정을 쉽게 하려고 광범위한 데이터베이스·스키마·계정 역할을 부여하지 마세요.

```sql
GRANT USAGE ON DATABASE <database> TO ROLE <git_setup_role>;
GRANT USAGE ON SCHEMA <database>.<schema> TO ROLE <git_setup_role>;
GRANT CREATE GIT REPOSITORY ON SCHEMA <database>.<schema>
  TO ROLE <git_setup_role>;
GRANT USAGE ON INTEGRATION <git_api_integration>
  TO ROLE <git_setup_role>;
```

토큰 인증에서는 Git Repository Clone을 만드는 역할에 Secret `USAGE`도 부여하세요. 별도의 Actions 역할이 `FETCH`를 실행하도록 하려면 Git Repository에 `WRITE`를 부여합니다. 꼭 필요하고 승인된 경우가 아니라면 소유권을 넘기지 마세요.

```sql
GRANT USAGE ON DATABASE <database> TO ROLE <git_sync_role>;
GRANT USAGE ON SCHEMA <database>.<schema> TO ROLE <git_sync_role>;
GRANT USAGE ON WAREHOUSE <warehouse> TO ROLE <git_sync_role>;
GRANT WRITE ON GIT REPOSITORY <database>.<schema>.<repository_object>
  TO ROLE <git_sync_role>;
```

## 4. 로컬 Snowflake CLI 설치 및 구성

Snowflake의 독립 설치 프로그램 또는 지원되는 패키지 관리자를 사용해 Snowflake CLI를 설치하세요. 예를 들어 `uv`를 사용하면 다음과 같습니다.

```bash
uv tool install snowflake-cli
snow --version
```

대화형으로 로컬 연결을 추가합니다.

```bash
snow connection add
```

Snowflake 계정 식별자, 사용자 이름, 최소 권한 역할, Warehouse를 입력하세요. 키페어 인증에는 `SNOWFLAKE_JWT`를 선택하고 개인 키 파일 경로를 지정합니다. 개인 키 암호문구를 공유 설정 파일에 저장하지 말고 보호된 로컬 시크릿 저장소 또는 CLI가 지원하는 암호문구 환경 변수를 사용하세요. 로컬 Snowflake 연결 파일을 소스 제어에 포함하지 마세요.

연결 이름을 지정해 테스트합니다.

```bash
snow connection test -c <connection_name>
```

권한이 있고 선택한 환경에서 실행이 승인된 경우에만 검토가 끝난 SQL 파일을 실행하세요.

```bash
snow sql -c <connection_name> -f <sql_file>
```

## 5. GitHub Actions에서 Snowflake Git Clone 가져오기

### GitHub OIDC 및 Snowflake 서비스 사용자 구성

Snowflake 서비스 사용자의 워크로드 ID에 발급자를 `https://token.actions.githubusercontent.com`으로 설정하고, 사용할 GitHub Actions Workflow ID와 정확히 일치하는 Subject를 지정하세요. Subject 범위를 저장소와 보호된 브랜치 또는 실행 환경으로 제한합니다. 이 사용자에게는 `<git_sync_role>` 역할만 할당하세요.

아래는 Snowflake 서비스 사용자 정의의 형태입니다. Subject는 실제 GitHub OIDC 설정에서 확인한 정확한 값으로 바꿔야 합니다.

```sql
CREATE USER <github_actions_service_user>
  TYPE = SERVICE
  WORKLOAD_IDENTITY = (
    TYPE = OIDC
    ISSUER = 'https://token.actions.githubusercontent.com'
    SUBJECT = '<exact_github_actions_subject>'
  )
  DEFAULT_ROLE = <git_sync_role>;

GRANT ROLE <git_sync_role> TO USER <github_actions_service_user>;
```

Actions Subject는 이벤트 종류에 따라 달라지며 사용자 지정도 가능합니다. 실제 Workflow Claim을 확인하지 않고 예시 Subject를 운영 환경에 복사하지 마세요. Snowflake의 `snowflake-actions`는 Snowflake CLI 3.11 이상에서 OIDC를 지원합니다.

비밀이 아닌 계정 및 사용자 식별자는 필요하면 GitHub 저장소 변수로 설정할 수 있습니다. OIDC를 사용하면 GitHub Secrets에 Snowflake 비밀번호나 개인 키를 저장할 필요가 없습니다. 기본 브랜치를 보호하고 Workflow를 수정할 수 있는 사용자를 제한하세요.

### Workflow 추가

아래 Workflow는 `main`에 Push되거나 수동 실행될 때 Snowflake Clone을 가져옵니다. SQL 파일을 실행하지 않습니다. Repository Object 이름, 계정, 서비스 사용자는 자격 증명이 아닌 설정 값입니다.

```yaml
name: Fetch Snowflake Git repository

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

concurrency:
  group: snowflake-git-fetch
  cancel-in-progress: false

jobs:
  fetch:
    runs-on: ubuntu-latest
    steps:
      - name: Set up Snowflake CLI with OIDC
        uses: snowflakedb/snowflake-actions@v3
        with:
          cli-version: "3.20.0"
          use-oidc: true

      - name: Fetch remote changes into Snowflake
        env:
          SNOWFLAKE_ACCOUNT: ${{ vars.SNOWFLAKE_ACCOUNT }}
          SNOWFLAKE_USER: ${{ vars.SNOWFLAKE_USER }}
          SNOWFLAKE_ROLE: ${{ vars.SNOWFLAKE_ROLE }}
          SNOWFLAKE_WAREHOUSE: ${{ vars.SNOWFLAKE_WAREHOUSE }}
        run: |
          snow connection test -x
          snow sql -x -q "ALTER GIT REPOSITORY <DATABASE>.<SCHEMA>.<REPOSITORY_OBJECT> FETCH;"
```

해당 GitHub 변수에 비밀이 아닌 값을 설정하고 역할에 데이터베이스, 스키마, Warehouse 및 Git Repository에 필요한 권한을 부여하세요. 업그레이드하거나 버전을 고정하기 전에 Snowflake CLI 및 Action 버전을 최신 릴리스 문서와 대조하세요. 공급망 통제를 강화하려면 검토한 전체 커밋 SHA로 GitHub Actions를 고정하세요. 신뢰할 수 없는 Pull Request 코드에 OIDC 권한을 노출하지 말고 이 Workflow에 `pull_request_target`을 추가하지 마세요.

WIF를 사용할 수 없다면 전용 서비스 사용자의 암호화된 키페어를 구성하고 개인 키와 암호문구를 별도의 보호된 GitHub Environment Secret으로 저장하세요. Workflow를 보호된 브랜치/환경으로 제한하고, 스크립트 본문에 시크릿을 직접 보간하지 않는 방법으로 키 파일을 작성하고 소유자만 읽을 수 있게 권한을 설정한 뒤 작업 종료 시 임시 파일을 삭제하세요. 가능하면 WIF를 우선 사용하세요.

## 6. OpenAI Codex에 안전한 프로젝트 지침 제공

Codex는 `AGENTS.md` 파일을 프로젝트 지침으로 읽습니다. 내용을 간결하고 저장소에 맞게 유지하세요. 자격 증명과 접근 권한은 별도로 설정해야 합니다. 예를 들면 다음과 같습니다.

```markdown
# AGENTS.md

## 프로젝트 개요
- 이 저장소는 Snowflake SQL 및 데이터 파이프라인 코드를 포함합니다.
- 구성된 최소 권한 연결에서만 `snow` CLI를 사용합니다.
- 별도 요청이 없으면 개발 데이터베이스와 스키마를 기본으로 사용합니다.

## 안전한 작업 흐름
- 파일을 수정하거나 SQL을 실행하기 전에 작업 트리와 대상 환경을 확인합니다.
- 생성된 SQL 및 diff를 검토한 후 실행합니다.
- 자격 증명, 개인 키, 토큰, 암호문구를 출력하거나 커밋하거나 복사하지 않습니다.
- 명시적 승인 없이 운영 환경을 변경하거나 파괴적 SQL을 실행하지 않습니다.
- 변경 사항을 자동으로 커밋하거나 Push하지 않고 검토할 수 있도록 준비합니다.
```

Codex 지침은 Snowflake 권한을 부여하지 않습니다. 명령이 접근할 수 있는 범위는 Snowflake 역할, 계정 정책, CLI 연결로 결정됩니다. 에이전트가 검토 없이 `main`에 Push하거나 파괴적 SQL을 실행하도록 지시하지 마세요.

## 7. 자주 발생하는 문제 해결

| 증상 | 확인할 항목 |
|---|---|
| 키페어 로그인 실패 | 등록된 공개 키와 개인 키가 일치하는지, CLI 연결에 `SNOWFLAKE_JWT`가 설정됐는지, 개인 키 경로가 올바른지, 보호된 경로에서 암호문구를 사용할 수 있는지 확인합니다. |
| 자동화에서 비밀번호 또는 MFA 오류 | Workflow가 지정된 서비스 ID와 지원되는 비대화형 인증(WIF/OIDC 권장)을 사용하는지 확인합니다. 정책을 우회하려고 MFA를 끄거나 사람의 비밀번호를 재사용하지 마세요. |
| Git API Integration이 저장소 접근을 거부 | HTTPS Origin과 Integration 허용 Prefix를 비교하고 Integration 활성화 여부와 선택한 GitHub 인증 방식을 확인합니다. |
| `Object does not exist` 또는 `not authorized` 오류 | 활성 역할에 데이터베이스/스키마 `USAGE` 및 Git Repository `WRITE` 또는 `OWNERSHIP`이 있는지 확인합니다. 토큰 인증은 Secret이 API Integration에서 허용되며 필요한 역할이 접근할 수 있는지 확인합니다. |
| OIDC 인증 실패 | `id-token: write`, Snowflake CLI 버전, 서비스 사용자 유형, 발급자, 정확한 Subject Claim, 계정 식별자, 할당된 역할을 확인합니다. 모든 브랜치로 Subject를 넓히지 말고 정확히 제한합니다. |
| Workflow는 성공했지만 SQL 변경이 적용되지 않음 | `FETCH`는 Snowflake Git Clone만 갱신합니다. SQL을 실행하거나 배포하려면 별도의 검토된 배포 단계를 추가해야 합니다. |

## 공식 문서

- [Snowflake 키페어 인증 및 키 교체](https://docs.snowflake.com/en/user-guide/key-pair-auth)
- [Snowflake 워크로드 ID 연합](https://docs.snowflake.com/en/user-guide/workload-identity-federation)
- [Snowflake Git 설정 및 인증](https://docs.snowflake.com/en/developer-guide/git/git-setting-up-public)
- [Snowflake Git Repository 개요](https://docs.snowflake.com/en/developer-guide/git/git-overview)
- [CREATE GIT REPOSITORY](https://docs.snowflake.com/en/sql-reference/sql/create-git-repository)
- [ALTER GIT REPOSITORY 및 접근 권한](https://docs.snowflake.com/en/sql-reference/sql/alter-git-repository)
- [Snowflake CLI 설치](https://docs.snowflake.com/en/developer-guide/snowflake-cli/installation)
- [Snowflake CLI 연결](https://docs.snowflake.com/en/developer-guide/snowflake-cli/connecting/configure-connections)
- [GitHub Actions용 Snowflake Actions](https://github.com/snowflakedb/snowflake-actions)
- [Snowflake CoCo (Cortex Code) 개요](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code)
- [OpenAI Codex `AGENTS.md` 지침](https://developers.openai.com/codex/guides/agents-md)
- [GitHub Personal Access Token 지침](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)
- [GitHub Actions의 시크릿 사용](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions)
- [GitHub Actions OIDC 참조](https://docs.github.com/en/actions/reference/security/oidc)

*문서 확인일: 2026-11-14. Snowflake, GitHub 및 클라이언트 기능은 변경될 수 있으므로 배포 전에 연결된 벤더 문서와 계정 정책을 확인하세요.*
