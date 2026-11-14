# Snowflake MCP 설정 가이드

이 문서는 Snowflake의 공식 **Snowflake-managed MCP server**를 구성하고 MCP 호환 클라이언트에 연결하는 방법을 설명합니다. 서버 URL, SQL 객체, 접근 제어, OAuth는 Snowflake 공식 문서를 기준으로 합니다. 예시에는 자리표시자만 사용하며 계정 식별자, 클라이언트 시크릿, 토큰, 개인 자격 증명을 문서나 소스 코드에 기록하지 마세요.

> **범위:** Managed MCP Server는 Snowflake 서비스입니다. Snowflake에서 MCP Server 객체를 만들며 로컬 Snowflake MCP 서버 프로세스를 설치하지 않습니다. 공식 문서는 관리형 엔드포인트와 OAuth를 통한 클라이언트 연결을 설명합니다. 타사 또는 커뮤니티 구현이 Snowflake에서 지원된다고 주장하지 않습니다.

## 1. 사전 준비

- Snowflake 계정 및 Snowflake-managed MCP Server를 사용할 수 있는 리전. 현재 리전별 제공 여부와 제한 사항은 공식 문서에서 확인합니다.
- MCP Server 객체를 만들 데이터베이스·스키마와 권한을 가진 역할.
- 공개할 기존 Snowflake 리소스. 예: Cortex Agent, Cortex Search 서비스, Cortex Analyst Semantic View, SQL 실행 기능, UDF 또는 Stored Procedure.
- 문서화된 원격 엔드포인트 및 Snowflake OAuth 또는 구성된 External OAuth를 지원하는 클라이언트.
- OAuth를 설정하고 필요한 권한만 부여할 관리자 또는 보안 담당자.

현재 Snowflake-managed MCP Server는 Tool 발견과 호출을 포함한 도구 기능을 지원합니다. MCP의 모든 기능을 구현한 것은 아니므로 Resources, Prompts, Roots 등 다른 프로토콜 기능에 의존하기 전에 공식 제한 사항을 확인하세요.

## 2. 최소 권한 MCP Server 만들기

거버넌스가 필요한 비즈니스 데이터 접근에는 Cortex Agent를 클라이언트에 노출되는 도구로 사용하는 방식을 Snowflake가 권장합니다. Agent가 승인된 Analyst, Search, 사용자 정의 도구를 조정하므로 클라이언트는 하나의 관리된 인터페이스에 연결할 수 있습니다.

아래는 Snowflake 공식 `CREATE MCP SERVER` 구문을 사용한 Agent 예시입니다.

```sql
CREATE MCP SERVER <database_name>.<schema_name>.<server_name>
  FROM SPECIFICATION $$
  tools:
    - title: "Governed business data agent"
      name: "business_data_agent"
      type: "CORTEX_AGENT_RUN"
      identifier: "<database_name>.<schema_name>.<agent_name>"
      description: "Answers governed business data questions."
  $$;
```

도구 이름과 설명은 목적을 분명하게 나타내도록 작성하세요. Snowflake는 Cortex Search, Cortex Analyst, SQL 실행, UDF/Stored Procedure를 위한 일반 도구 등 다른 도구 유형도 문서화합니다. 클라이언트에 필요한 도구만 구성하세요. Tool annotation은 클라이언트에 제공하는 힌트이지 보안 제어가 아닙니다.

## 3. 서버와 리소스 권한 부여

전용 역할을 만들고 필요한 권한만 부여한 뒤 승인된 사용자에게 해당 역할을 할당하세요. Agent 기반 서버에는 예를 들어 다음 권한이 필요할 수 있습니다.

```sql
CREATE ROLE <mcp_access_role>;

GRANT DATABASE ROLE SNOWFLAKE.CORTEX_AGENT_USER TO ROLE <mcp_access_role>;
GRANT USAGE ON WAREHOUSE <warehouse_name> TO ROLE <mcp_access_role>;
GRANT USAGE ON DATABASE <database_name> TO ROLE <mcp_access_role>;
GRANT USAGE ON SCHEMA <database_name>.<schema_name> TO ROLE <mcp_access_role>;
GRANT USAGE ON MCP SERVER <database_name>.<schema_name>.<server_name>
  TO ROLE <mcp_access_role>;
GRANT USAGE ON AGENT <database_name>.<schema_name>.<agent_name>
  TO ROLE <mcp_access_role>;
GRANT ROLE <mcp_access_role> TO USER <username>;
```

Agent 도구가 실제 사용하는 기반 객체 권한도 역할에 부여해야 합니다. 예를 들어 Search 도구는 Cortex Search 서비스에 `USAGE`가 필요합니다. Analyst 도구는 Semantic View 및 기반 데이터에 `SELECT`가 필요할 수 있고, 일반 도구는 UDF 또는 Stored Procedure의 문서화된 권한이 필요합니다. MCP Server 접근 권한만으로 모든 도구를 사용할 수 있는 것은 아닙니다. 클라이언트가 동작하도록 `PUBLIC`에 광범위하게 권한을 부여하거나 현재·미래의 모든 테이블에 접근을 열어서는 안 됩니다.

## 4. OAuth 구성

Snowflake-managed MCP Server는 OAuth 2.0을 사용합니다. 기본 인증은 Snowflake OAuth이며, 조직의 ID 공급자를 사용하도록 External OAuth를 구성할 수 있습니다. MCP Server는 동적 클라이언트 등록을 지원하지 않습니다.

클라이언트 요구사항에 맞는 클라이언트 유형과 정확한 Redirect URI를 사용해 Snowflake OAuth Security Integration을 만듭니다. Snowflake는 클라이언트 시크릿을 안전하게 저장할 수 있는 경우 `CONFIDENTIAL`, 시크릿 없이 PKCE를 사용하는 경우 `PUBLIC`을 문서화합니다. Redirect URI를 추측하지 말고 해당 클라이언트의 최신 설정 화면에 표시된 콜백 URI를 사용하세요.

Confidential Client의 공식 SQL 형태:

```sql
CREATE SECURITY INTEGRATION <integration_name>
  TYPE = OAUTH
  OAUTH_CLIENT = CUSTOM
  ENABLED = TRUE
  OAUTH_CLIENT_TYPE = 'CONFIDENTIAL'
  OAUTH_REDIRECT_URI = '<exact_client_redirect_uri>'
  OAUTH_USE_SECONDARY_ROLES = NONE
  ALLOWED_ROLES_LIST = ('<mcp_access_role>');
```

PKCE를 사용하는 Public Client라면 Snowflake 공식 문서의 `OAUTH_CLIENT_TYPE = 'PUBLIC'` 및 해당 클라이언트의 정확한 Redirect URI 설정을 따르세요. `ALLOWED_ROLES_LIST`는 필요한 접근 역할로 제한하세요. 더 넓은 권한이 의도적으로 필요한 경우가 아니라면 Snowflake는 최소 권한 MCP 구성을 위해 보조 역할을 비활성화할 것을 권장합니다.

통합 객체 생성 후, 적절한 권한을 가진 관리자는 다음 공식 시스템 함수로 클라이언트 자격 증명을 조회할 수 있습니다.

```sql
SELECT SYSTEM$SHOW_OAUTH_CLIENT_SECRETS('<INTEGRATION_NAME>');
```

함수의 통합 이름은 대소문자를 구분하며 대문자여야 합니다. 결과는 시크릿으로 취급하세요. 가이드, 소스 제어, 티켓, 터미널 기록, 채팅에 붙여 넣지 마세요. MCP 클라이언트의 승인된 보안 저장소에 보관하세요.

External OAuth를 사용하는 경우 IdP용 External OAuth Security Integration을 구성하고 `OAUTH_AUTHORIZATION_SERVER`를 사용해 계정·데이터베이스·스키마 범위에 연결합니다. Snowflake의 발급자, 대상(Audience), Claim 매핑, 역할 Scope 요구사항을 확인하세요. 예시 IdP 값을 실제 설정으로 그대로 복사하지 마세요.

## 5. MCP 엔드포인트 확인

Snowflake가 문서화한 엔드포인트 형식:

```text
https://<account_url>/api/v2/databases/<database>/schemas/<schema>/mcp-servers/<name>
```

계정에 맞는 Snowflake URL과 데이터베이스·스키마·서버의 전체 이름을 사용하세요. 계정 식별자 형식은 Snowflake 문서를 참고하세요. 일부 클라이언트는 밑줄이 포함된 호스트 이름에서 연결 문제가 발생할 수 있으며, Snowflake는 MCP 연결 호스트 이름에 하이픈 사용을 권장합니다. 리전 URL을 추측하거나 다른 계정의 URL을 사용하지 마세요.

## 6. 클라이언트 연결

Snowflake는 MCP 호환 클라이언트 연결 방법을 문서화합니다. 클라이언트별 UI 및 OAuth 콜백은 다르며 변경될 수 있습니다.

### Claude 사용자 지정 Connector

Snowflake의 현재 공통 클라이언트 안내에는 Claude(Claude.ai 및 Claude Desktop)가 포함됩니다.

1. Claude에서 **Settings → Connectors**를 엽니다.
2. **Add custom connector**를 선택합니다. (또는 사용 가능한 경우 Snowflake를 찾아봅니다.)
3. Connector 이름과 앞 절의 MCP 엔드포인트를 입력합니다.
4. Snowflake Integration의 OAuth Client ID와 Secret을 클라이언트의 보안 설정 절차를 통해서만 제공합니다.
5. Snowflake의 OAuth Redirect URI를 Claude가 현재 표시한 정확한 URI로 설정합니다. Claude.ai와 Claude Desktop의 콜백 URI는 다릅니다.
6. 브라우저에서 Snowflake 로그인과 동의를 완료한 뒤 예상한 도구가 표시되는지 확인합니다.

다른 클라이언트는 원격 MCP 엔드포인트 및 OAuth에 대한 현재 공식 설정 문서를 따르세요. JSON/TOML 클라이언트 설정 스키마, OAuth 지원 방식, 콜백 요구사항이 클라이언트마다 다르므로 확인되지 않은 구성 예시는 이 가이드에 싣지 않습니다.

## 7. 검증 및 문제 해결

| 증상 | 확인할 항목 |
|---|---|
| OAuth 동의 또는 연결 실패 | Redirect URI가 클라이언트의 최신 콜백 URI와 정확히 일치하는지, Integration이 활성화되었는지 확인합니다. |
| 인증은 성공하지만 서버 연결 실패 | 전체 엔드포인트, 계정 URL, 데이터베이스, 스키마, 서버 이름을 확인합니다. |
| 호스트 이름 연결 오류 | 계정 호스트 이름에 밑줄이 있는지 확인하고 해당하는 경우 Snowflake가 안내한 하이픈 형식을 사용합니다. |
| 세션 초기화 실패 | 사용자에게 기본 Warehouse가 있는지, 접근 역할에 Warehouse `USAGE`가 있는지 확인합니다. |
| 잘못된 역할/데이터 범위 | 사용자 기본 역할 및 OAuth Integration의 `ALLOWED_ROLES_LIST`를 확인합니다. 보조 역할이 의도하지 않은 접근을 허용하지 않는지 확인합니다. |
| 도구 목록이 비어 있음 | MCP Server의 `USAGE`와 Agent 도구의 대상 Agent `USAGE`를 부여했는지 확인하고 사용자의 유효 역할을 확인합니다. |
| 도구는 보이지만 호출 실패 | 해당 Agent, Search 서비스, Semantic View, 함수, 프로시저에 필요한 최소 권한을 부여합니다. |
| PrivateLink SaaS 클라이언트 연결 실패 | Snowflake는 SaaS MCP 클라이언트에 공개 MCP 엔드포인트를 사용하고 문서화된 대로 권한 부여 엔드포인트에 PrivateLink를 설정하도록 안내합니다. |
| 원격 SaaS 클라이언트가 네트워크 정책에 의해 차단 | Snowflake의 네트워크 정책 지침에 따라 클라이언트 사업자가 공개한 아웃바운드 IP를 허용합니다. IP 범위를 추측하지 마세요. |

`SHOW MCP SERVERS` 및 `DESCRIBE MCP SERVER`로 서버 객체를 확인할 수 있습니다. 소유자가 변경을 승인한 경우에만 객체를 제거하세요. 더 많은 사례는 Snowflake의 최신 문제 해결 표를 참고하세요.

## 보안 점검 목록

- 하드코딩 토큰보다 OAuth를 우선 사용합니다. OAuth Client Secret은 승인된 시크릿 관리자 또는 클라이언트의 보안 자격 증명 저장소에 보관합니다.
- 전용 최소 권한 역할과 전용 Warehouse를 사용하고 일상적인 MCP 접근에 관리자 역할을 사용하지 않습니다.
- MCP Server 및 각 기반 도구의 권한을 별도로 부여하고, 실제 사용자의 권한으로 표시·호출 가능한 도구를 검증합니다.
- 타사 MCP Server와 도구 설명을 활성화 전에 검증합니다. 도구 설명은 실제 동작을 오도할 수 있습니다.
- Tool annotation을 권한 제어로 간주하지 않습니다. Snowflake 역할과 객체 권한으로 접근을 제어합니다.
- 클라이언트 아웃바운드 IP 허용 목록 및 사설 연결 요구사항을 보안팀과 확인합니다.
- 무한하거나 과도한 작업을 일으킬 수 있는 순환 Agent/MCP 호출 경로를 피합니다.
- 운영 배포 전에 Snowflake 공식 문서에서 제공 리전, 지원 클라이언트, OAuth 동작, 제한 사항을 다시 확인합니다.

## 공식 문서

- [Snowflake-managed MCP server](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp)
- [Snowflake 핵심 개념 및 아키텍처](https://docs.snowflake.com/en/user-guide/intro-key-concepts)
- [Snowflake OAuth 개요](https://docs.snowflake.com/en/user-guide/oauth-snowflake-overview)
- [Snowflake 계정 식별자](https://docs.snowflake.com/en/user-guide/admin-account-identifier)
- [CREATE SECURITY INTEGRATION (Snowflake OAuth)](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oauth-snowflake)
- [CREATE SECURITY INTEGRATION (External OAuth)](https://docs.snowflake.com/en/sql-reference/sql/create-security-integration-oauth-external)
- [Snowflake 네트워크 정책](https://docs.snowflake.com/en/user-guide/network-policies)

*문서 확인일: 2026-11-14. Snowflake 및 클라이언트 기능은 바뀔 수 있으므로 배포 전에 연결된 벤더 문서를 확인하세요.*
