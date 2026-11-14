# Postman API 테스트 및 OAuth 토큰 자동화

Postman에서 REST API 요청을 구성하고 응답을 검증하며, API별 액세스 토큰 흐름을 안전하게 자동화하는 방법을 정리합니다.

## 컬렉션과 환경 구성

Workspace는 개인 또는 팀의 작업 공간이고, Collection은 관련 요청을 묶습니다. 인증, 사용자, 주문 등 기능별로 폴더를 구성할 수 있습니다. 각 Request에는 HTTP 메서드, URL, 헤더, 인증 방식, 필요 시 요청 본문이 포함됩니다. 여러 요청이 같은 인증 방식을 쓰면 Collection 수준에서 설정하고 하위 요청은 상위 설정을 상속하도록 합니다. 별도 인증이 필요한 요청만 개별적으로 재정의합니다.

로컬, 테스트, 운영처럼 환경마다 달라지는 설정은 Environment에 둡니다. 예를 들어 `https://api.example.com`을 `base_url`로 등록하고 `{{base_url}}`로 참조합니다. 한 Collection에서만 공통으로 쓰며 환경에 따라 달라지지 않는 값은 Collection variable을 사용할 수 있습니다. Global variable은 범위가 넓으므로 임시 프로토타이핑에 적합합니다.

이름이 같은 변수가 여러 범위에 있으면 Postman은 가장 좁은 범위의 값을 우선 사용합니다(로컬, 데이터, 환경, 컬렉션, 전역 순). 예상치 못한 상위 범위 값이 적용되지 않도록 필요한 범위에만 변수를 두는 것이 좋습니다.

## REST 요청 전송과 검증

1. 메서드와 URL을 선택합니다. 예: `GET {{base_url}}/v1/users`
2. 쿼리 매개변수는 URL 문자열을 직접 이어 붙이기보다 **Params**에서 입력합니다.
3. JSON `POST`, `PUT`, `PATCH` 요청은 **Body > raw > JSON**을 선택하고 API 계약에 맞는 본문을 작성합니다.

   ```json
   {
     "username": "sample-user",
     "email": "user@example.com"
   }
   ```

4. 전송한 뒤 API 계약에 따라 상태 코드, 헤더, 본문, 응답 시간을 확인합니다. 예를 들어 `204 No Content`는 검사할 응답 본문이 없습니다.
5. 요청의 **Scripts > Post-response**에 검증을 작성하면 반복 실행할 수 있습니다.

   ```javascript
   pm.test("성공 응답을 반환한다", function () {
     pm.expect(pm.response.code).to.be.within(200, 299);
   });

   pm.test("JSON 본문을 반환한다", function () {
     pm.response.to.have.header("Content-Type");
   });
   ```

   검증 조건은 엔드포인트 계약에 맞게 조정합니다. 모든 성공 응답이 JSON이나 `200`을 반환하는 것은 아닙니다.

Collection Runner에서 Collection 또는 폴더를 순서대로 실행하고 각 요청의 테스트 결과를 확인할 수 있습니다. 반복 실행 데이터도 사용할 수 있습니다. CI/CD에서는 Postman CLI와 실행 환경의 비밀 값 주입 방식을 검토합니다.

## 인증과 변수의 안전한 사용

Bearer token을 사용하는 API라면 Collection의 **Authorization**에서 **Bearer Token**을 선택하고 토큰 변수로 설정합니다. Postman이 `Bearer` 접두사를 붙이므로 토큰 값만 입력합니다. 하위 요청은 **Inherit auth from parent**로 설정합니다. API가 다른 인증 방식을 요구하면 해당 API의 명세에 맞게 설정합니다.

OAuth 액세스 토큰은 리소스 서버의 API 호출을 인증합니다. 인증 서버가 발급하는 경우 리프레시 토큰은 토큰 엔드포인트에 보내 새 액세스 토큰을 요청하는 데 사용합니다. 토큰 만료, 응답 필드, 회전(rotation), 폐기, 재사용 정책은 API 제공자가 정합니다. 모든 API가 같은 만료 기간을 쓰거나 갱신 응답에서 새 리프레시 토큰을 반환한다고 가정하지 않습니다.

토큰, API 키, 비밀번호, 클라이언트 시크릿은 인증정보로 취급합니다.

- 실제 인증정보를 문서, 예제, 내보낸 Collection, 소스 관리, 스크린샷에 넣지 않습니다. 스크립트나 Postman Console에 토큰 값을 기록하지 않습니다.
- 민감한 값은 공유·동기화되는 Environment, Collection variable, Global variable에 넣지 않습니다. UI에서 값이 마스킹된다는 사실만으로 비공개 또는 미동기화 상태가 보장되는 것은 아닙니다.
- Postman Vault 또는 조직에서 승인한 시크릿 관리자를 우선합니다. Postman Local Vault 값은 Postman Cloud와 동기화되지 않습니다. Shared Vault는 Workspace 접근 제어가 적용되지만 팀 공유용입니다. 사용 중인 실행 방식의 지원 여부, 요금제, 접근 범위를 확인합니다.
- API별 워크플로에서 단기 토큰을 Environment variable에 저장해야 한다면 로컬·비공유 환경을 사용하고 민감한 값을 내보내거나 동기화하지 않습니다. 공유 템플릿에는 자리표시자만 둡니다.
- 필요한 최소 권한만 부여하고, 제공자 정책에 따라 인증정보를 폐기하거나 교체합니다.

변수 값은 스크립트에서 문자열로 사용할 수 있습니다. `pm.environment`, `pm.collectionVariables`, `pm.globals` 등 워크플로에 맞는 범위를 선택하고, 편의를 위해 시크릿을 전역 변수에 두지 않습니다.

## 로그인 응답에서 토큰 저장

로그인 응답에 토큰 필드가 있다면 **Post-response** 스크립트로 다음 요청에 사용할 값을 추출할 수 있습니다. 필드명과 응답 구조는 실제 API에 맞게 바꿉니다. 아래 예시는 선택된 Environment에 기록하므로 민감한 값에는 로컬·비공유 환경에서만 사용합니다.

```javascript
if (pm.response.code < 200 || pm.response.code >= 300) {
  throw new Error(`로그인 실패: HTTP ${pm.response.code}`);
}
const body = pm.response.json();
const accessToken = body.accessToken;
if (typeof accessToken !== "string" || accessToken.length === 0) {
  throw new Error("로그인 응답에 accessToken이 없습니다.");
}

pm.environment.set("access_token", accessToken);

// API가 새 리프레시 토큰을 반환한 경우에만 저장합니다.
if (typeof body.refreshToken === "string" && body.refreshToken.length > 0) {
  pm.environment.set("refresh_token", body.refreshToken);
}
```

Collection의 Authorization에서 Bearer token을 `{{access_token}}`으로 설정합니다. 응답 필드가 `data` 아래에 중첩되는 API라면 문서화된 경로를 사용합니다. API가 반환한 만료 정보를 사용하고, 임의로 만료 시각을 계산하지 않습니다.

## 토큰 갱신

가장 단순한 방법은 필요할 때 별도의 갱신 요청을 실행하는 것입니다.

1. API 명세에 명시된 토큰 갱신 엔드포인트로 `POST` 요청을 만듭니다.
2. 해당 엔드포인트가 요구하는 인증 방식을 설정합니다. 일부 API는 **No Auth**와 JSON 본문을 사용하지만, 다른 API는 클라이언트 인증이나 폼 매개변수를 요구할 수 있습니다.
3. API가 지정한 필드명과 Content-Type으로 리프레시 토큰을 전달합니다. 엔드포인트 URL, grant 매개변수, 응답 구조는 API별로 다릅니다.
4. Post-response 스크립트에서 응답을 검증하고 새 액세스 토큰을 저장합니다. 리프레시 토큰 회전을 사용하는 제공자라면 새 토큰이 반환된 경우에만 이전 값을 교체하고, 이전 토큰의 무효화 방식은 제공자 정책을 따릅니다.

만료 시각을 알고 있다면 Collection의 **Pre-request** 스크립트로 요청 직전에 갱신할 수 있습니다. 아래 코드는 흐름을 보여주는 예시이지 범용 OAuth 구현이 아닙니다. URL, 본문, 응답 속성은 API 문서에 맞게 변경해야 합니다. 로컬 Environment를 사용하고, 회전되는 리프레시 토큰을 동시에 여러 요청이 갱신하면서 경합하지 않도록 합니다.

```javascript
const refreshToken = pm.environment.get("refresh_token");
const expiresAt = Number(pm.environment.get("access_token_expires_at"));
const accessToken = pm.environment.get("access_token");
const refreshEarlyByMs = 30_000;
const needsRefresh =
  !accessToken ||
  (Number.isFinite(expiresAt) && Date.now() >= expiresAt - refreshEarlyByMs);

if (needsRefresh) {
  if (!refreshToken) {
    throw new Error("토큰 갱신 불가: refresh_token이 설정되지 않았습니다.");
  }

  const response = await pm.sendRequest({
    url: `${pm.environment.get("base_url")}/v1/auth/refresh`,
    method: "POST",
    header: { "Content-Type": "application/json" },
    body: {
      mode: "raw",
      raw: JSON.stringify({ refreshToken })
    }
  });

  if (response.code < 200 || response.code >= 300) {
    throw new Error(`토큰 갱신 실패: HTTP ${response.code}`);
  }

  const body = response.json();
  if (typeof body.accessToken !== "string" || body.accessToken.length === 0) {
    throw new Error("갱신 응답에 accessToken이 없습니다.");
  }
  pm.environment.set("access_token", body.accessToken);

  if (typeof body.refreshToken === "string" && body.refreshToken.length > 0) {
    pm.environment.set("refresh_token", body.refreshToken);
  }
  const expiresIn = Number(body.expiresIn);
  if (
    body.expiresIn !== undefined &&
    body.expiresIn !== null &&
    Number.isFinite(expiresIn) &&
    expiresIn > 0
  ) {
    pm.environment.set(
      "access_token_expires_at",
      String(Date.now() + expiresIn * 1000)
    );
  }
}
```

이 예시는 `expiresIn`이 초 단위이고 응답이 JSON인 API를 가정합니다. API가 다른 만료 필드를 제공한다면 그 형식에 맞춥니다. 만료 정보를 알 수 없다면 API가 명시적으로 보장하지 않는 한 토큰만으로 만료 시각을 추정할 수 없습니다. 갱신 실패 시 응답을 확인하고 필요한 경우 다시 인증해야 합니다. 오래된 토큰으로 조용히 계속 진행하지 않습니다.

Postman의 OAuth 2.0 Authorization은 제공자가 지원하는 경우 대화형으로 토큰을 요청·갱신할 수 있습니다. 토큰 동기화 전에 공유 설정을 확인합니다. 현재 Postman 문서에 따르면 예약 실행, 모니터, Postman CLI, Newman에서는 OAuth 2.0 인증을 대화형 요청과 같은 방식으로 지원하지 않으며, 동기화된 토큰이 필요할 수 있고 자동 토큰 갱신은 지원되지 않습니다. 자동화에 의존하기 전에 최신 제품 및 요금제 동작을 확인합니다.

## 참고 자료

- [Postman: 변수 저장 및 재사용](https://learning.postman.com/docs/use/send-requests/variables/variables/)
- [Postman: Postman Vault에 시크릿 저장](https://learning.postman.com/docs/use/postman-vault/postman-vault-secrets/)
- [Postman: API 인증 및 권한 부여](https://learning.postman.com/docs/use/send-requests/authorization/authorization/)
- [Postman: OAuth 2.0 인증](https://learning.postman.com/docs/use/send-requests/authorization/oauth-20/)
- [Postman: 스크립트에서 요청 전송](https://learning.postman.com/docs/tests-and-scripts/write-scripts/postman-sandbox-reference/pm-send-request/)
- [Postman: Collection Runner로 API 테스트](https://learning.postman.com/docs/tests-and-scripts/running-collections/intro-to-collection-runs/)
