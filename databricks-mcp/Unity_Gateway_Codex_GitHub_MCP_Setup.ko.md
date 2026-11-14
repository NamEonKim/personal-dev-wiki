# Unity Gateway를 통해 Codex와 GitHub MCP 연결하기 (Windows)

초보자도 처음부터 따라 할 수 있도록 Windows PowerShell 기준으로 정리한
가이드입니다.

> **목표:** Databricks Unity Gateway에 로그인하고 `system.ai.github`
> MCP 서버를 Codex에 등록합니다.
>
> **개인정보 보호:** 아래 예시에는 실제 사용자 이메일, Windows 계정명,
> Databricks Workspace 식별자가 포함되지 않습니다. `<...>` 부분은 본인
> 환경의 값으로 바꾸세요.

---

## 1. 준비물

- Windows 10/11
- PowerShell
- Python 3.12 이상 및 `uv`
- Databricks Workspace 계정 및 해당 Workspace 접근 권한
- Workspace에서 `system.ai.github` MCP 서버를 사용할 권한
- Codex 설치 및 실행 환경

**중요:** Unity Gateway의 `ug mcp add`는 Codex용 MCP 설정을 등록하는
명령입니다. Codex Desktop을 사용한다면 MCP 등록 후 앱을 완전히 종료하고
다시 실행하세요.

## 2. `uv` 설치

PowerShell을 열고 다음을 실행합니다.

```powershell
winget install --id astral-sh.uv -e
```

설치가 끝나면 PowerShell을 완전히 종료한 뒤 새 창을 엽니다.

설치 확인:

```powershell
uv --version
```

정상 예시:

```text
uv 0.x.x
```

`uv`가 인식되지 않으면 새 PowerShell 창을 열었는지 확인하고, 그래도
인식되지 않으면 설치 및 PATH 설정을 점검하세요.

## 3. Unity Gateway 설치

다음 명령으로 Unity Gateway를 설치합니다.

```powershell
uv tool install git+https://github.com/databricks/unity-gateway
```

설치 확인:

```powershell
uv tool list
ug --help
```

`uv tool list`에 Unity Gateway 패키지가 표시되고 `ug --help`가 도움말을
출력하면 설치된 것입니다.

`ug` 명령을 찾을 수 없으면 PowerShell을 새로 열고 다시 확인하세요.

## 4. Databricks CLI 설치

Unity Gateway가 Databricks 인증을 시작할 때 `databricks` 명령을
호출하므로 Databricks CLI도 설치해야 합니다.

```powershell
winget install Databricks.DatabricksCLI
```

설치 확인:

```powershell
databricks -v
```

정상 예시:

```text
Databricks CLI v1.x.x
```

### CLI 설치가 필요한 이유

Databricks CLI가 없으면 `ug mcp add` 실행 중 다음과 같은 오류가 발생할
수 있습니다.

```text
FileNotFoundError: [WinError 2]
```

이 경우 Unity Gateway를 다시 설치하기 전에 Databricks CLI 설치 여부와
PATH를 먼저 확인하세요.

## 5. Databricks Workspace 로그인

아래 명령의 Workspace 주소를 본인의 Databricks Workspace 주소로
바꿉니다.

```powershell
databricks auth login --host "https://<DATABRICKS_WORKSPACE_HOST>"
```

예를 들어 Workspace 주소가
`https://<workspace-id>.cloud.databricks.com` 형식이라면 해당 주소를
그대로 사용합니다.

로그인 과정에서 브라우저가 열리면 Databricks 계정으로 로그인하고 권한을
승인합니다. 터미널에서 Profile 이름을 입력하라는 메시지가 나오면
기억하기 쉬운 이름을 입력하면 됩니다. 이메일 주소를 Profile 이름으로
사용할 필요는 없습니다.

인증 확인:

```powershell
databricks auth profiles
```

또는 등록한 프로필을 지정해 인증 상태를 확인할 수 있습니다.

```powershell
databricks auth describe --profile "<DATABRICKS_PROFILE_NAME>"
```

## 6. GitHub MCP 서버를 Codex에 등록

아래 명령에서 Workspace 주소를 본인의 주소로 바꿉니다.

```powershell
ug mcp add --agents codex --workspace "https://<DATABRICKS_WORKSPACE_HOST>" --names system.ai.github
```

각 옵션의 의미:

- `ug mcp add`: Unity Gateway MCP 서버를 에이전트 설정에 추가
- `--agents codex`: Codex에 등록
- `--workspace`: 사용할 Databricks Workspace 지정
- `--names system.ai.github`: GitHub MCP 서버 지정

처음 설정하는 경우 Databricks 로그인을 요청할 수 있습니다. 이미
인증되어 있다면 기존 인증을 재사용합니다.

성공하면 다음과 비슷한 메시지가 표시됩니다.

```text
Databricks auth already available
Add MCP Servers
Configuring for: Codex
Saved
```

## 7. MCP 등록 상태 확인

다음 명령을 실행합니다.

```powershell
ug mcp list --agents codex
```

정상 등록 예시:

```text
NAME              LOCATION          AGENTS  STATUS
system-ai-github  system.ai.github  codex   enabled
```

- `system-ai-github`: 설정 목록에 표시되는 서버 이름
- `system.ai.github`: Databricks MCP 서비스의 전체 이름
- `codex`: Codex용으로 등록됨
- `enabled`: 설정이 활성화됨

**주의:** `enabled`는 설정이 저장되고 활성화되었다는 뜻입니다. 실제
GitHub API 호출이나 저장소 접근까지 성공한다는 보증은 아니므로 실제
도구 호출을 테스트해야 합니다.

## 8. Codex 설정 파일 위치 확인

Codex는 MCP 서버 설정을 `config.toml`에 저장합니다.

Windows 기본 사용자 설정 경로:

```text
%USERPROFILE%\.codex\config.toml
```

PowerShell에서 메모장으로 열기:

```powershell
notepad "$env:USERPROFILE\.codex\config.toml"
```

파일이 아직 없으면 Codex 또는 설정 도구가 생성할 수 있습니다.
`ug mcp add --agents codex`는 Codex용 MCP 설정을 자동으로 등록하므로,
우선 직접 내용을 추가하거나 수정하지 마세요.

Codex MCP 설정은 보통 다음과 같은 TOML 테이블 형식을 사용합니다.

```toml
[mcp_servers.example]
command = "example-command"
args = ["--example"]
```

위 코드는 **형식을 설명하기 위한 예시**일 뿐이며 Unity Gateway가 생성한
실제 GitHub MCP 설정을 대체하지 않습니다. 자동 생성된 항목을 임의로
덮어쓰지 마세요.

프로젝트별 설정을 사용하는 경우 프로젝트 폴더의
`.codex/config.toml`을 사용할 수도 있습니다. 신뢰하는 프로젝트에서만
프로젝트 설정을 사용하세요.

## 9. Codex에서 실제 연결 테스트

1. Codex Desktop을 완전히 종료합니다.
2. Codex Desktop을 다시 실행합니다.
3. MCP 도구 목록 또는 Codex의 연결된 도구 화면에 GitHub 관련 도구가
   표시되는지 확인합니다.
4. 다음과 같이 실제 도구 호출을 요청합니다.

   ```text
   Use the GitHub MCP tools to list the GitHub repositories
   I am authorized to access. Do not make any changes.
   ```

도구 호출이 실행되고 접근 가능한 저장소 목록이 반환되면 연결 테스트가
성공한 것입니다.

**중요:** Codex가 자연어로 답변하는 것만으로는 MCP 연결을 확인할 수
없습니다. 실제로 GitHub MCP 도구가 호출되었는지 확인하세요.

## 10. 문제 해결

### A. `uv`를 인식하지 못함

오류 예시:

```text
uv : 'uv' 용어가 cmdlet ... 으로 인식되지 않습니다.
```

해결 방법:

1. `winget install --id astral-sh.uv -e`를 실행합니다.
2. PowerShell을 완전히 종료합니다.
3. 새 PowerShell을 실행합니다.
4. `uv --version`으로 확인합니다.

### B. `ug`를 인식하지 못함

확인:

```powershell
uv tool list
uv tool dir --bin
```

설치되지 않았다면 다시 설치합니다.

```powershell
uv tool install git+https://github.com/databricks/unity-gateway
```

새 PowerShell 창에서 `ug --help`를 다시 실행합니다.

### C. `FileNotFoundError: [WinError 2]`

Databricks CLI를 찾지 못하는 경우가 있습니다. 다음 명령으로 설치합니다.

```powershell
winget install Databricks.DatabricksCLI
```

PowerShell을 새로 열고 다음을 확인합니다.

```powershell
databricks -v
```

그런 다음 Databricks에 로그인합니다.

### D. Databricks 인증 오류

```powershell
databricks auth login --host "https://<DATABRICKS_WORKSPACE_HOST>"
```

브라우저 인증을 완료했는지, 올바른 Workspace에 로그인했는지, 해당
Workspace에 접근 권한이 있는지 확인합니다.

### E. `enabled`인데 Codex에서 도구가 보이지 않음

1. `ug mcp list --agents codex`로 등록 상태를 확인합니다.
2. `%USERPROFILE%\.codex\config.toml` 파일을 확인합니다.
3. Codex Desktop을 완전히 종료한 뒤 다시 실행합니다.
4. Codex의 MCP 도구 목록을 확인합니다.
5. Databricks Workspace에서 해당 MCP 서버를 사용할 권한이 있는지
   확인합니다.

## 11. 이번 설정에서 사용한 명령 모음

처음부터 설정할 때 다음 순서로 실행합니다.

```powershell
# 1. uv 설치
winget install --id astral-sh.uv -e

# 2. uv 확인 (새 PowerShell 창에서)
uv --version

# 3. Unity Gateway 설치
uv tool install git+https://github.com/databricks/unity-gateway

# 4. Unity Gateway 확인
ug --help

# 5. Databricks CLI 설치
winget install Databricks.DatabricksCLI

# 6. Databricks CLI 확인
databricks -v

# 7. Databricks 로그인
databricks auth login --host "https://<DATABRICKS_WORKSPACE_HOST>"

# 8. GitHub MCP를 Codex에 등록
ug mcp add --agents codex --workspace "https://<DATABRICKS_WORKSPACE_HOST>" --names system.ai.github

# 9. 등록 상태 확인
ug mcp list --agents codex
```

---

## 공식 문서

- Unity Gateway 프로젝트: https://github.com/databricks/unity-gateway
- Databricks — Connect Codex:
  https://docs.databricks.com/aws/en/ai-gateway/coding-agent-codex
- Databricks — Add MCP tools and skills:
  https://docs.databricks.com/aws/en/ai-gateway/coding-agent-mcp-skills
- OpenAI — Codex MCP 설정:
  https://learn.chatgpt.com/docs/extend/mcp?surface=cli

## 보안 및 개인정보 주의사항

- Workspace URL, 이메일, 사용자명, 토큰, OAuth 정보, 개인 액세스
  토큰(PAT)을 공개 저장소에 올리지 마세요.
- `config.toml`에 인증 토큰이나 민감한 값이 들어갈 수 있으므로 파일을
  공유하기 전에 내용을 확인하고 비밀값을 마스킹하세요.
- 화면 캡처나 오류 로그를 공유할 때 이메일 주소, Workspace 식별자,
  토큰 및 Profile 이름을 가리세요.
- GitHub MCP의 실제 읽기/쓰기 범위는 Databricks MCP 권한 및 GitHub
  연결·승인 범위에 따라 달라집니다.
