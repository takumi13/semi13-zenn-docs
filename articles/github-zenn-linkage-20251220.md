---
title: "KeycloakのFAPIポリシーに準拠したOIDCクライアントを作る(SpringBoot) 前編"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java", "SpringBoot", "Keycloak", "OIDC", "FAPI"]
published: false
publication_name: "oidfj"
---

## 1. はじめに

### 1.1. 自己紹介

はじめまして、@semi13 です。
OpenID Foundation Japan TG40WGのメンバとしてこれからいくつか記事を執筆予定です。よろしくお願いいたします。

Digital Identity 技術勉強会 #iddance Advent Calendar 2025
20 日目の記事です。

https://qiita.com/advent-calendar/2025/iddance

さて。私がこのような記事を執筆しようと思った理由はシンプルです。
OpenID Connect準拠のクライアントのバックエンド処理を自分で書いたことがないからです。
とはいえ一から書ける能力も時間も足りないため、バイブコーディングに頼ろうと思います。

そしてもう一つ。Keycloakを使ってみたかったからです。

Keycloak26.4.0のリリースでは、FAPI2 Finalをサポートしています。
https://www.keycloak.org/docs/latest/release_notes/index.html#fapi-2-final-supported

OSSのOpenID Provider（以下OP）であるKeycloakがサポートしてくれたのなら、難しいこと（OP目線の仕様や実装）を考えずに、クライアント目線の仕様理解と実装に集中できると思い、検証意欲が湧きました。

しかし、いきなりFAPI1.0をすっとばしてFAPI2.0の仕様を読むのも実装に走るのもつらいので、以下のような3編構成で記事化しようと思います。

| 編 | 記事内容 |
| ---- | ---- |
| 前編 | FAPI1.0 Part1: Baselineに準拠したOIDCクライアントの開発 |
| 中編 | FAPI1.0 Part2: Advancedに準拠したOIDCクライアントの開発 |
| 後編 | FAPI 2.0 Security Profileに準拠したOIDCクライアントの開発 |

ということで本記事は前編です。
中編/後編はアドカレ終了後に細々と更新予定です。

### 1.2. 記事のゴール

| 分類 | ゴール |
| ---- | ---- |
| 仕様理解 | FAPI 1.0 Part1: Baselineに関する仕様について順を追って説明する |
| 実装理解 | FAPI 1.0 Part1: Baselineの要求事項に準拠したOIDCクライアントを開発し、keycloakからAPI正常応答が返るようにする |

### 1.3. 想定読者層

以下に関する仕様理解は前提とし、説明は割愛します。

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
  - `Authorization Code Flow`に基づく`Authorization Request` , `Token Request`のシーケンスやリクエスト/レスポンスの各種パラメータ
  - `IDToken Validation`の際に検証対象とする各種パラメータ
- [PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
  - 各種シーケンス
  - `code_verifier`, `code_challenge`の生成ロジック
- [OAuth 2.0 Threat Model and Security Considerations](https://openid-foundation-japan.github.io/rfc6819.ja.html)に記載の各Security Considerations

以下については本記事でも概要を説明します。

- [Auth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens(mTLS)](https://datatracker.ietf.org/doc/html/rfc8705)

### 1.4. 技術スタック

| 分類 | 技術 | バージョン |
| --- | --- | ---|
| RP | 開発環境 | wsl 2.4.12|
| RP | 開発言語（クライアント） | Java21 |
| RP | Webアプリケーションフレームワーク | SpringBoot4.0.0 |
| OP | Keycloak | version 26.4.1 |
| OP | DB (keycloak利用) | MySQL8.0.43 |
| tools | Docker | 28.1.1 | 
| tools | Docker Compose | v2.35.1 |
| tools | keytool | 21.0.7 |
| tools | openssl | OpenSSL 1.1.1f 31 Mar 2020 |

### 1.5. ソースコード

本記事に掲載するソースコードは以下の`feature/ph02.3/security_enhanced`ブランチのものを一部記事用に修正して掲載しています。
https://github.com/takumi13/keycloak-idp-rp/tree/feature/ph02.3/security_enhanced

### 1.6. 本記事へFB

ソースコードの大部分は`GPT-5.1-Codex (Preview)`を利用して作成していますが、OIDCに関する各種specやRFCと照らし合わせた処理の正当性確認は筆者自身がおこなっています。誤りがあればコメント、Twitter(現X)等にてご指摘いただけますと幸いです。

## 2. FAPI準拠クライアントへの成長サクセスストーリー

:::message
OIDCクライアントのMTI
- MTIはMandatory To Implementの略で「実装必須」と訳されます
- シーケンス中のOIDCクライアントの各処理を「OIDCクライアントのMTI」として本メッセージ形式で記載します
:::

### 2.1. 基本的なOIDCクライアント

ここでは[OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)に定義される`Authorization Code Flow`に準拠した最低限のOIDCクライアントの実装を行います。

クライアントは以下の3つの処理を行います。

1. `Authorization Request`を実行
    - state, nonce, [pkce](https://datatracker.ietf.org/doc/html/rfc7636)パラメータをそれぞれ生成
2. 受け取った認可コードを使って`Token Request`を実行
3. 受け取ったJWT型の`access_token`を復号して`id_token`を取り出し、IDトークンを検証

以下のシーケンスに基づくOIDCクライアントのバックエンドを実装します。

なお、フロントエンドの実装はOIDCクライアントのデモンストレーションの都合、各種リクエストパラメータ/レスポンスを表示することを中心とした実装としています。
![](/images/2.1.AuthorizationCodeFlow.png)
*[2.1.AuthorizationCodeFlow.pu](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/docs/2.%E3%82%B7%E3%83%BC%E3%82%B1%E3%83%B3%E3%82%B9/plantuml/2.1.AuthorizationCodeFlow.pu)*

#### ○ Authorization Code Flow対応

##### Keycloakの設定

基本路線としては、公式が配布するDockerイメージを使ってビルドしつつ、流れに沿って各種クライアントを作成します。
https://www.keycloak.org/getting-started/getting-started-docker

今回は以下のような`Realm`, `user`, `Client`を作成しました。
|項目|値|
|-|-|
|`Realm`|`myrealm`|
|`user`|`semi`|
|`Client`|`semi_client`|

また、設定を`MySQL`に保存するために`keycloak`のイメージと一緒に`mysql`イメージもビルドしています。これにより、`docker compose up`と`docker compose down`によるコンテナ起動管理を通じた、keycloak設定のDBへの永続化が可能となります。
ついでに、デモンストレーション目的のアプリケーションであるため、ログレベルも`DEBUG`に設定します。

```yml:docker-compose.yml
volumes:
  mysql_data:

services:
  mysql:
    image: mysql:8.0.43
    container_name: mysql
    volumes:
      - mysql_data:/var/lib/mysql
    environment:
      # ------------------------------------------------------------------
      # [mysql] MySQL用資格情報
      # ------------------------------------------------------------------
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: keycloak
      MYSQL_USER: keycloak
      MYSQL_PASSWORD: password
    ports:
      - "3306:3306"
    healthcheck:
      test: ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -ppassword --silent"]
      interval: 10s
      timeout: 5s
      retries: 10
    restart: unless-stopped

  keycloak:
    image: quay.io/keycloak/keycloak:26.4.1
    container_name: keycloak
    environment:
      # ------------------------------------------------------------------
      # [keycloak admin] 管理者サイトログイン用資格情報
      # ------------------------------------------------------------------
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      # ------------------------------------------------------------------
      # [database] MySQL接続設定
      # ------------------------------------------------------------------
      KC_DB: mysql
      KC_DB_URL: jdbc:mysql://mysql:3306/keycloak
      KC_DB_URL_DATABASE: keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: password
      # ------------------------------------------------------------------
      # [logging] ローカル検証向けに DEBUG ログとアクセスログを出力
      # ------------------------------------------------------------------
      KC_LOG: "console,file"
      KC_LOG-FILE: "data/server.log"
      KC_LOG-LEVEL: "DEBUG"
      QUARKUS_HTTP_ACCESS_LOG_LOG_DIRECTORY: "logs"
      QUARKUS_HTTP_ACCESS_LOG_BASE_FILE_NAME: "access"
      QUARKUS_HTTP_RECORD_REQUEST_START_TIME: true
      QUARKUS_HTTP_ACCESS_LOG_PATTERN: "%h %l %u %t \"%r\" %s %b \"%{i,Referer}\" \"%{i,User-Agent}\" %D"
    ports:
      - "8080:8080"
    depends_on:
      mysql:
        condition: service_healthy
    command: ["start-dev"]
    restart: unless-stopped
```

##### OIDCクライアントの実装

###### 2.1.1. Authorization Requestの実行

:::message
OIDCクライアントのMTI
- `state`, `nonce`をUUIDなどの乱数文字列として生成
- `code_verifier`, `code_challenge`をPKCE仕様に従って生成
- 上記パラメータをクライアント側のメモリ上に保存（今回はHTTPセッションへ格納）
- 各種リクエストパラメータをセットしてKeycloakの`Authorization Request Endpoint` (`/auth`) へHTTP GETリクエスト
:::

`PkceService.java`は、PKCE仕様に従って`code_verifier`と`code_challenge`を生成します。
参照: [PkceService.java](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/PkceService.java)

`SessionStateService.java`は、HTTPセッションに`state`, `nonce`, `code_verifier`, `code_challenge`を格納し、そのパラメータを管理します。
参照: [SessionStateService.java](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/SessionStateService.java)

`OidcClientService.java`は、`Authorization Code Flow`のリクエストを生成するために各種リクエストパラメータの組み立て、クライアントサイドに格納したパラメータ（`client_secret`等）の取り出しを行います。
参照: [OidcClientService.java](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/OidcClientService.java)

`AuthorizationFlowController.java`は、以下のように（デモンストレーション用に）フロントエンドから受け取った各種リクエストパラメータや、バックエンドで事前に生成したパラメータを利用して`Authorization Request`を発行します。

```java:AuthorizationFlowController.java
/* ------------------------------
 * GET /authorization-flow
 * ------------------------------
*/
@GetMapping("${application.path.authorization-flow:/authorization-flow}")
public String showForm(HttpSession session, Model model) {
    // state, nonceを生成
    String state = UUID.randomUUID().toString();
    String nonce = UUID.randomUUID().toString();

    // PKCEパラメータを生成
    String codeVerifier = pkceService.generateVerifier();
    String codeChallenge = pkceService.generateChallenge(codeVerifier);
    String codeChallengeMethod = "S256";

    // セッションに保存（state に紐付け）
    sessionStateService.storePkceBundle(session, state, nonce, codeVerifier, codeChallengeMethod);

    // Thymeleaf に渡す（デモンストレーション用）
    model.addAttribute("code_verifier", codeVerifier);
    model.addAttribute("state", state);
    model.addAttribute("nonce", nonce);
    model.addAttribute("code_challenge", codeChallenge);
    model.addAttribute("code_challenge_method", codeChallengeMethod);

    return "authorization_flow";
}

/* ------------------------------
 * POST /authorize
 * ------------------------------
*/
@PostMapping("${application.path.authorize:/authorize}")
// デモンストレーション用にフロントエンドから受け取ったパラメータを利用する
public RedirectView authorize(
        @RequestParam(name = "authorization_endpoint", required = false) String authorizationEndpoint,
        @RequestParam(name = "response_type", required = false) String responseType,
        @RequestParam(name = "client_id", required = false) String clientId,
        @RequestParam(name = "redirect_uri", required = false) String redirectUri,
        @RequestParam(name = "scope", required = false) String scope,
        @RequestParam(name = "state", required = false) String state,
        @RequestParam(name = "nonce", required = false) String nonce,
        @RequestParam(name = "code_challenge", required = false) String codeChallenge,
        @RequestParam(name = "code_challenge_method", required = false) String codeChallengeMethod,
        HttpSession session
) {
    String endpoint = oidcClientService.resolveAuthorizationEndpoint(authorizationEndpoint);

    // 値チェック
    Map<String, String> params = new LinkedHashMap<>();
    if (responseType != null && !responseType.isBlank()) params.put("response_type", responseType);
    if (clientId != null && !clientId.isBlank()) params.put("client_id", clientId);
    if (redirectUri != null && !redirectUri.isBlank()) params.put("redirect_uri", redirectUri);
    String resolvedScope = (scope != null && !scope.isBlank()) ? scope : "openid";
    params.put("scope", resolvedScope);
    if (state != null && !state.isBlank()) params.put("state", state);
    if (nonce != null && !nonce.isBlank()) params.put("nonce", nonce);

    // code_challenge_methodが指定されていない場合はPKCEを利用しない
    // keycloak側でクライアントのPKCE利用を強制している場合はこの時点でエラーを返したほうが良い
    String method = (codeChallengeMethod != null && !codeChallengeMethod.isBlank()) ? codeChallengeMethod : "";
    if (method.isEmpty()) {
        codeChallenge = null;
    }
    if (codeChallenge != null && !codeChallenge.isBlank()) {
        params.put("code_challenge", codeChallenge);
        params.put("code_challenge_method", method);
        sessionStateService.rememberCodeChallengeMethod(session, state, method);
    }

    // Keycloak側のAuthorization Request URL（/authz）を作成
    String authUrl = oidcClientService.buildAuthorizationRequestUri(endpoint, params);
    logger.debug("Redirecting to Authorization Endpoint: " + authUrl);
    return new RedirectView(authUrl);
}
```

###### 2.1.2. Authorization Responseの受け取り

:::message
OIDCクライアントのMTI
- `state`の値を検証（`Authorization Requets`時に指定した値との一致を確認）
- 認可コード（`code`）を取得
:::

`CallbackController.java`は、リダイレクトによって`Authorization Response`から`state`を受け取り、値を検証します。

※認可コード（`code`）を受け取り、そのまま`Token Request`を実行することが本来の処理ですが、今回はデモンストレーション用に返却値を確認するために一度フロントエンド側で値を表示させます。

```java:CallbackController.java
@GetMapping("${application.path.callback:/callback}")
public String callback(@RequestParam MultiValueMap<String, String> requestParams,
                        HttpSession session,
                        Model model) {
    
    // ...(中略)...

    // stateを検証
    String state = firstValue(requestParams, "state");
    PkceContext pkceContext = sessionStateService.loadPkceContext(session, state);
    if (!isValidState(state, pkceContext)) {
        model.addAttribute("error_message", "State mismatch detected");
        model.addAttribute("error_detail", state == null || state.isBlank()
                ? "Authorization response did not include a state parameter."
                : "No PKCE context exists for state=" + state);
        return "error";
    }

    // (※)記事参照
    CallbackViewModel viewModel = new CallbackViewModel(
            firstValue(requestParams, "code"),
            state,
            firstValue(requestParams, "redirect_uri"),
            firstValue(requestParams, "error"),
            firstValue(requestParams, "error_description"),
            orderedParams,
            pkceContext
    );
    model.addAttribute("callback", viewModel);
    return "callback";
}
private boolean isValidState(String state, PkceContext context) {
    if (state == null || state.isBlank()) {
        return false;
    }
    return context != null && !context.isEmpty() && state.equals(context.state());
}

// ...(中略)...

private String firstValue(MultiValueMap<String, String> params, String key) {
    List<String> values = params.get(key);
    if (values == null || values.isEmpty()) {
        return null;
    }
    return values.get(0);
}
```

###### 2.1.3. Token Requestの実行

:::message
OIDCクライアントのMTI
- HTTPセッションから`code_verifier`を取り出し`Token Request`のリクエストパラメータとして利用
- リクエストパラメータをセットしてKeycloakの`Token Request Endpoint` (`/token`) へHTTP POSTリクエスト
:::

```java:TokenController.java
@PostMapping("${application.path.token-request:/token-request}")
public String requestToken(
// デモンストレーション用にフロントエンドから受け取ったパラメータを利用する
        @RequestParam(name = "token_endpoint", required = false) String tokenEndpoint,
        @RequestParam(name = "code", required = false) String code,
        @RequestParam(name = "redirect_uri", required = false) String redirectUri,
        @RequestParam(name = "client_id", required = false) String clientId,
        @RequestParam(name = "grant_type", required = false, defaultValue = "authorization_code") String grantType,
        @RequestParam(name = "state") String state,
        HttpSession session
) throws Exception {

    // PKCEパラメータ（code_verifier）をHTTPセッションから取り出し
    PkceContext pkceContext = sessionStateService.loadPkceContext(session, state);
    if (pkceContext.isEmpty()) {
        throw new IllegalStateException("PKCE context not found for state=" + state);
    }
    boolean pkceActive = pkceContext.codeChallengeMethod() != null && !pkceContext.codeChallengeMethod().isBlank();

    String codeVerifier = pkceActive ? sessionStateService.consumeCodeVerifier(session, state) : null;
    if (pkceActive && (codeVerifier == null || codeVerifier.isBlank())) {
        throw new IllegalStateException("Missing code_verifier for PKCE-enabled authorization state=" + state);
    }

    // Token Requestパラメータをセット
    Map<String, String> form = new LinkedHashMap<>();
    if (grantType != null && !grantType.isBlank()) form.put("grant_type", grantType);
    if (code != null && !code.isBlank()) form.put("code", code);
    if (redirectUri != null && !redirectUri.isBlank()) form.put("redirect_uri", redirectUri);
    String resolvedClientId = (clientId != null && !clientId.isBlank()) ? clientId : oidcClientProperties.getClientId();
    if (resolvedClientId != null && !resolvedClientId.isBlank()) {
        form.put("client_id", resolvedClientId);
    }
    String resolvedClientSecret = oidcClientProperties.getClientSecret();
    if (resolvedClientSecret != null && !resolvedClientSecret.isBlank()) {
        form.put("client_secret", resolvedClientSecret);
    }
    if (codeVerifier != null && !codeVerifier.isBlank()) form.put("code_verifier", codeVerifier);

    // KeycloakのToken Request Endpoint（/token）へHTTP POST
    try {
        String response = oidcClientService.requestToken(tokenEndpoint, form);
        tokenResponseValidator.validate(response, pkceContext.nonce());
        return response;
    } finally {
        sessionStateService.clearPkceContext(session, state);
    }
}
```

###### 2.1.4. Token Responseの受け取り

:::message
OIDCクライアントのMTI
- （Keycloak特有）JWT型の`access_token`をデコード&検証
- `id_token`をデコード&検証
- 不要となった各種`Authorization Code Flow`用のパラメータをHTTPセッションから削除
:::

```java:TokenController.java(抜粋再掲)
    // KeycloakのToken Request Endpoint（/token）へHTTP POST
    try {
        String response = oidcClientService.requestToken(tokenEndpoint, form);

        // ここでAccess Token（JWT）とID Tokenを検証
        tokenResponseValidator.validate(response, pkceContext.nonce());

        return response;
    } finally {
        // Token Request完了後はHTTPセッションから不要となった`state`, `nonce`, `PKCEパラメータ`を削除
        sessionStateService.clearPkceContext(session, state);
    }
```

`TokenResponseValidator.java`は、`Authorization Response`から`access_token`と`id_token`を受け取って、それぞれを検証ロジックにかけます。
参照: [TokenResponseValidator.java](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/TokenResponseValidator.java)

```java:TokenResponseValidator.java
public void validate(String rawResponse, String expectedNonce) throws Exception {

    // ...(前略)...

    if (hasIdToken) {
        idTokenValidator.validate(idToken, expectedNonce, accessToken, ValidatedTokenType.ID_TOKEN);
    }
    if (hasAccessToken) {
        idTokenValidator.validate(accessToken, null, null, ValidatedTokenType.ACCESS_TOKEN);
    }
}
```

`IdTokenValidator.java`は、受け取ったToken（`access_token` or `id_token`）をデコードし、得られたJsonの各種`key-value`を検証します。

`OidcJwksClient.java`は、KeycloakのJWKSエンドポイントから署名鍵セットを取得し、TTL付きでキャッシュしておくサービスです。ID/アクセストークンの署名検証時にkidからRSA鍵を取得します。
参照: [OidcJwksClient.java](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/OidcJwksClient.java)

KeycloakのJWKSエンドポイントは`/realms/{Realm名}/protocol/openid-connect/certs`から取得できます。以下は具体的な取得例（`{Realm名}=myrealm`）です。

```json:/realms/myrealm/protocol/openid-connect/certs.json
{
  "keys": [
    {
      "kid": "mq6ErunPHEGTM8EtjO9ROL5-hRNRPdJ8IJj68aqTan8",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "x5c": [
        "MIICnTCCAYUCBgGaAfEQeTANBgkqhkiG9w0BAQsFADASMRAwDgYDVQQDDAdteXJlYWxtMB4XDTI1MTAyMDE0MDQyNloXDTM1MTAyMDE0MDYwNlowEjEQMA4GA1UEAwwHbXlyZWFsbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAK9/ti27d7R2L62ZTGkqjUTRc3xrghyCuyCFcaWzWX64AUmjjHmhSw72ifkd4xsp/12Mimzwe4U+FJ1bZztwETYfzj6JypU0K36q3w+K4jsYw0Le451+HYuehl+BmK/6SPjFAxiaZmG0MTmrnDUsvo+rTbVQ9OT1G0q+2Uxrx2bdH8blHWmu5SsyrrEdQFc/WFdXl/ev5YmJPgzttA0n1loeB0566VEbXWVLzm//tAGD6bSQwcZmq3MGJgT63FpEDbQOzCYUBXw80knydXyboX3Thzi/+YjSWFdlhkI2wuoMQwr+QG691rA8pjySA/stuEayiUvRC43SP5bw7iFYXRkCAwEAATANBgkqhkiG9w0BAQsFAAOCAQEAOxoVgi29FqDBxd60O7UgyXLSY5ugRBDtEz5XBZ7QxQxj4RPYBpJJCLlyOuKybXf2lHJiq/4AmBQYW04jIwcu6siFxCTPfQbyILj/JC6v5yR3DC+3lwRzXQA4Y88JGQnnGEDNOzygM6ASfvf31BEu+X9i1cd/ASrncyNxAqvW98f3afo70BjlALQthdmNJXbJqFN0edoURr9ZTF1NpsoaeFELCEfP3aSymjsYPUeAZvoC3p527KFGUYxQberWMW0RFNqIZxZb3vEOO5waQTt64nFF5vpuaNRRHU61jwG1zmR7hT1q3FvDKKOVc8UtCypujUVQG/Y8I1El2VSQ0Suz9Q=="
      ],
      "x5t": "mAMNTgins7mBF1EdvUKfsYugAcs",
      "x5t#S256": "XBgUdEBxxK74C5SI9D4_LACoJQBhnWbjK90kQOOhR9c",
      "n": "r3-2Lbt3tHYvrZlMaSqNRNFzfGuCHIK7IIVxpbNZfrgBSaOMeaFLDvaJ-R3jGyn_XYyKbPB7hT4UnVtnO3ARNh_OPonKlTQrfqrfD4riOxjDQt7jnX4di56GX4GYr_pI-MUDGJpmYbQxOaucNSy-j6tNtVD05PUbSr7ZTGvHZt0fxuUdaa7lKzKusR1AVz9YV1eX96_liYk-DO20DSfWWh4HTnrpURtdZUvOb_-0AYPptJDBxmarcwYmBPrcWkQNtA7MJhQFfDzSSfJ1fJuhfdOHOL_5iNJYV2WGQjbC6gxDCv5Abr3WsDymPJID-y24RrKJS9ELjdI_lvDuIVhdGQ",
      "e": "AQAB"
    },
    {
      "kid": "Ytcduim_K29Y1l710vfPnvZVslpvuF5zFUQE_64LFyc",
      "kty": "RSA",
      "alg": "RSA-OAEP",
      "use": "enc",
      "x5c": [
        "MIICnTCCAYUCBgGaAfESLDANBgkqhkiG9w0BAQsFADASMRAwDgYDVQQDDAdteXJlYWxtMB4XDTI1MTAyMDE0MDQyN1oXDTM1MTAyMDE0MDYwN1owEjEQMA4GA1UEAwwHbXlyZWFsbTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBAKY6tHOkVJ615f0exovql9IsPgTIPZ3nh58mI/Nmq6DSo4vQXAA6j9Bl+GRlgjLSQnxc76wFLDHU7sv5scPBrGWzYLYTI8PTYu+ooKOUtVJeOLaQAe2QHVl9aGLLxb8BSLbSi4QuAuellBOpuoh5FJXbG2GSLo1bWI/jzDesRyaCjaQ/Bg/SxDRmkQo0MskqSndccd6FevRIlDubL8juWnDinAf/MDvImiTjvxvQxtRq0PUQRAV2HTkkq37hpByYBMGPPesMQq98mwLcSxwXWoN6HMTOTgT/N5cUewQIUVItD9jIY8avQwzwUux63DOxWw3NAPBwbB8nTpBeSYRKjRcCAwEAATANBgkqhkiG9w0BAQsFAAOCAQEALPgV57UIlYod1OM6l7fAlx2umsGnLUAOQ6cGKW4NpSvfoqtdHs+7VjYAL6a5f+BQO84eIrBhtFHtBDAcwbNRdTYpgHPm1m2eMxcLv+Q9q1ukRZ5oXtaR4EKGvhJVcynIbK9vqdA94jfM0RehAS71tIj6BDXAIRhvOH8EdXnecC6m73TtJIU4ASqkm/m3crwk9wktjqeEieYvoiBaWjnlWzI3pa+nvaueQUSDVzKTLs5SW+r/Rth6/d39i4NKwDfm1g9NEzk7y5OFhdSDqs4zbekdioJGFQnaOkuhEsegXYriXqTae9XaUD3FjPAty2a+SHTsY4FShIDAY32ZiuST6g=="
      ],
      "x5t": "fuAOnnuFWWve68FYIhmnqKAr7Ng",
      "x5t#S256": "fvPhCNS5U5yoIgDKOc0mlsiP5lUxoSDcnHXgVrUrDxY",
      "n": "pjq0c6RUnrXl_R7Gi-qX0iw-BMg9neeHnyYj82aroNKji9BcADqP0GX4ZGWCMtJCfFzvrAUsMdTuy_mxw8GsZbNgthMjw9Ni76igo5S1Ul44tpAB7ZAdWX1oYsvFvwFIttKLhC4C56WUE6m6iHkUldsbYZIujVtYj-PMN6xHJoKNpD8GD9LENGaRCjQyySpKd1xx3oV69EiUO5svyO5acOKcB_8wO8iaJOO_G9DG1GrQ9RBEBXYdOSSrfuGkHJgEwY896wxCr3ybAtxLHBdag3ocxM5OBP83lxR7BAhRUi0P2Mhjxq9DDPBS7HrcM7FbDc0A8HBsHydOkF5JhEqNFw",
      "e": "AQAB"
    }
  ]
}
```

ここから`id_token`の具体的な検証ロジックを確認します。

**1. 署名付きJWT（JWS）の署名検証**
`jwksClient`（`OidcJwksClient`クラスのインスタンス）がJWKSエンドポイントから取得した`kid`をもとに`RSASSAVerifier`インスタンスを作成し、JWSを検証します。

https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/IdTokenValidator.java#L46-L60

なお、`logValidation(...)`は、デバッグ用に検証結果をログ出力するメソッドです。

https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/IdTokenValidator.java#L196-L206

**2. `id_token`の各種`key-value`の検証**
以下の8個の`key-value`を真面目に検証します。検証ロジックは [OpenID Connect Core 1.0の3.1.3.7.  ID Token Validation](https://openid.net/specs/openid-connect-core-1_0.html#IDTokenValidation) を参照します。

|パラメータ名|用途|検証方法|
|-|-|-|
|`iss` (issuer)|トークン発行者の識別|設定値のIssuerと一致するか比較し、不一致なら拒否|
|`aud` (audience)|トークン受信者(クライアント)の判別|クライアントIDがaud配列に含まれるか確認し、欠落時はエラー|ク
|`azp` (authorized party)|複数aud時に実際に権限付与されたクライアント識別|audが複数ならazpがclient_idと一致するか、アクセス用トークンでは常に一致必須|
|`sub` (subject)|エンドユーザーの一意識別子|subが空でないことを確認し、欠如なら拒否|
|`exp` (expiration)|トークン有効期限|expが存在し、現在時刻が有効期限内であることを確認|
|`iat` (issued-at)|トークン発行時刻|iatが存在し、未来日時でないことを確認|
|`nonce`|リプレイ攻撃防止|nonce要求済みなら、保存したnonceと一致するか確認|
|`at_hash`|access_token完全性検証|access_tokenがある場合、署名アルゴリズムに基づき計算した値と一致するか確認|

https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/IdTokenValidator.java#L62-L72

具体的な検証ロジックは以下を参照してください。
参照: [IdTokenValidator.java](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/spring-boot-oidc-client/src/main/java/com/example/oidcclient/service/IdTokenValidator.java)

### 2.2. FAPI 1.0 Part1: Baseline対応

ここではOIDCクライアントに以下の処理を追加することで、FAPI 1.0 Part1: Baselineに対応します。

1. keycloak, OIDCクライアントのTLS対応
2. mTLS対応（`Token Request`実行時にクライアント証明書を付与）

#### ○ keycloak, OIDCクライアントのTLS対応

FAPI 1.0 Part1: Baselineでは、通信経路のTLS保護が必須とされています（SHALL）。

https://openid.net/specs/openid-financial-api-part-1-1_0.html#tls-and-dnssec-considerations

> *7.1.  TLS and DNSSEC considerations*
> *As confidential information is being exchanged, all interactions shall be encrypted with TLS (HTTPS).*

本章では、ローカル開発環境（`localhost`）であっても、HTTPSでの通信を実現するために、独自のCA（認証局）と自己署名証明書を作成し、KeycloakおよびSpring Bootに適用します。

##### 1. CAとKeycloak用サーバ証明書の作成
まず、OpenSSLを使用して独自CA（Certificate Authority）を作成し、それを使ってKeycloakのHTTPSサーバ証明書を発行します。

```bash
# ディレクトリ作成
mkdir -p certs ssl

# 1.1. 開発用 CA 作成
openssl genrsa -out certs/ca.key 2048

openssl req -x509 -new -key certs/ca.key \
  -sha256 -days 365 \
  -out certs/ca.crt \
  -subj "/CN=test-ca"

# 1.2. Keycloak HTTPS サーバ用秘密鍵
openssl genrsa -out certs/keycloak-https-server.key 2048

# 1.3. Keycloak HTTPS サーバ用 CSR
openssl req -new \
  -key certs/keycloak-https-server.key \
  -out certs/keycloak-server.csr \
  -subj "/CN=localhost"

# 1.4. CA でサーバ証明書を発行
openssl x509 -req \
  -in certs/keycloak-server.csr \
  -CA certs/ca.crt \
  -CAkey certs/ca.key \
  -CAcreateserial \
  -out certs/keycloak-https-server.crt \
  -days 365 \
  -sha256

# 1.5. サーバ証明書を表示
openssl x509 -in certs/keycloak-https-server.crt -text -noout
```

##### 2. OIDCクライアント用サーバ証明書（KeyStore）の作成
次に、OIDCクライアント自身もHTTPSで待機させるため、PKCS12形式のKeystoreを作成します。

```bash
# 2.1. Keystoreを作成
mkdir -p spring-boot-oidc-client/src/main/resources/ssl

keytool -genkeypair \
  -alias oidc-client \
  -keyalg RSA \
  -keysize 2048 \
  -dname "CN=localhost, OU=Dev, O=YourOrg, L=Tokyo, ST=Tokyo, C=JP" \
  -validity 3650 \
  -storetype PKCS12 \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/oidc-https-keystore.p12 \
  -storepass changeit \
  -keypass changeit \
  -ext "SAN=dns:localhost,ip:127.0.0.1,ip:::1"

# 2.2. OIDCクライアントのKeystoreを表示
keytool -list -v \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/oidc-https-keystore.p12 \
  -storetype PKCS12 \
  -storepass changeit
```

##### 3. OIDCクライアント用TrustStoreの作成
OIDCクライアントからKeycloakへHTTPSアクセスする際、Keycloakの証明書は独自CAで署名されているため、Java標準のTrustStoreでは信頼できません。そのため、Keycloakのサーバ証明書をインポートしたTrustStoreを作成し、OIDCクライアントに認識させます。

```bash
# 3.1. Keycloak HTTPS サーバ証明書を truststore に登録
keytool -importcert \
  -alias keycloak-https-server \
  -file certs/keycloak-https-server.crt \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/keycloak-server-truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit \
  -noprompt

# 3.2. Keycloakのサーバ証明書をインポートしたTruststoreを表示
keytool -list -v \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/keycloak-server-truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit
```

##### 4. 設定ファイルの反映
作成した証明書をコンテナおよびアプリケーションに適用します。

**Keycloak側 (`docker-compose.yml`)**
Keycloakコンテナに証明書と秘密鍵をマウントし、HTTPS関連の環境変数を設定します。

```yaml:docker-compose.yml
  keycloak:
    # ...
    environment:
      # HTTPS設定
      KC_HTTPS_CERTIFICATE_FILE: "/opt/keycloak/conf/tls.crt"
      KC_HTTPS_CERTIFICATE_KEY_FILE: "/opt/keycloak/conf/tls.key"
      # ...
    volumes:
      - ./certs/keycloak-https-server.crt:/opt/keycloak/conf/tls.crt:ro
      - ./certs/keycloak-https-server.key:/opt/keycloak/conf/tls.key:ro
      # ...
```

**Spring Boot側 (`application.properties`)**
自身のサーバ証明書と、Keycloakへの接続時に使用するTrustStoreを指定します。

```properties:application.properties
# Spring Boot Server SSL
server.ssl.enabled=true
server.ssl.key-store=classpath:ssl/oidc-https-keystore.p12
server.ssl.key-store-password=changeit
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=oidc-client

# Keycloak接続用 TrustStore (TokenClientServiceなどで使用)
application.keycloak.mtls.trust-store=ssl/keycloak-server-truststore.p12
application.keycloak.mtls.trust-store-password=changeit
application.keycloak.mtls.trust-store-type=PKCS12
```

#### ○ mTLS対応

FAPI 1.0 Part1: Baselineでは、`Token Request`等へのアクセスにおいて以下いずれかのクライアント認証が求められます。

https://openid.net/specs/openid-financial-api-part-1-1_0.html#confidential-client

> 5.2.4.  Confidential client
> In addition to the provisions for a public client, a confidential client
> 
> shall support the following methods to authenticate against the token endpoint:
>    1. Mutual TLS for OAuth Client Authentication as specified in Section 2 of MTLS, and
>    2. client_secret_jwt or private_key_jwt as specified in Section 9 of OIDC;

ここでは **mTLS (Mutual TLS)** を採用し、クライアント証明書を用いた相互認証を実装します。

mTLSは、OAuth 2.0 Mutual-TLS Client Authentication and Certificate-Bound Access Tokens（[RFC8705](https://datatracker.ietf.org/doc/html/rfc8705)）にて定義されています。

サーバ証明書（X.509証明書）を利用したTLS通信は、一般的な文脈としては以下の2つを同時に満たす仕組みのことを指します。
  1. クライアントサーバ（例：OIDCクライアント） → リモートサーバ（例：Keycloak）への接続をHTTPSプロトコルにより暗号化
  2. リモートサーバが提示するサーバ証明書をクライアントサーバが検証することで、接続先サーバの真正性を保証

ホストサーバをサーバ証明書によって信頼できる（真正性が保たれた）状態にすることで、クライアントサーバは安心してリモートサーバへアクセス可能となります。

このときリモートサーバ目線では、クライアントサーバが信頼できる接続元であるかどうかをTLS層で判断することができません。代替方式として、OpenID Connect Core1.0では例えばクライアント認証方式として`client_secret_post`を利用することで、アプリケーション層で`client_id`と`client_secret`の一致確認をし、接続元のクライアントを信頼します。

上記の例に当てはめると、mTLSは、「**クライアントサーバが提示したクライアント証明書をリモートサーバが検証することで、TLS層においてクライアントサーバの真正性を保証し、そのクライアントを認証する仕組み**」といえます。

まずは、mTLSによってKeycloak → OIDCクライアントの向きでの信頼関係を構築する準備を行います。

##### 1. クライアント証明書の作成
Spring BootがKeycloakへ提示するためのクライアント証明書（KeyStore）を作成し、Keycloak側へ登録するためにPEM形式でエクスポートします。

```bash
# 1.1. クライアント証明書用Keystore（p12）の作成
keytool -genkeypair \
  -alias oidc-client-user \
  -keyalg RSA \
  -keysize 2048 \
  -dname "CN=semi, OU=Dev, O=YourOrg, L=Tokyo, ST=Tokyo, C=JP" \
  -validity 3650 \
  -storetype PKCS12 \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/oidc-mtls-client-keystore.p12 \
  -storepass changeit \
  -keypass changeit

# 1.2. PEM形式へのエクスポート（KeycloakのTrustStore登録用）
keytool -exportcert \
  -alias oidc-client-user \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/oidc-mtls-client-keystore.p12 \
  -storetype PKCS12 \
  -storepass changeit \
  -rfc \
  -file spring-boot-oidc-client/src/main/resources/ssl/client-cert.pem

# 1.3. クライアント証明書用Keystoreを表示
keytool -list -v \
  -keystore spring-boot-oidc-client/src/main/resources/ssl/oidc-mtls-client-keystore.p12 \
  -storetype PKCS12 \
  -storepass changeit
```

##### 2. Keycloak用TrustStoreの作成
Keycloakがクライアント証明書を検証するためのTrustStoreを作成し、エクスポートしたPEM証明書をインポートします。このTrustStoreはKeycloakコンテナにマウントされます。

```bash
# 2.1. Keycloakがクライアント証明書を検証するためのTrustStoreを作成
keytool -importcert \
  -alias oidc-client-user \
  -file spring-boot-oidc-client/src/main/resources/ssl/client-cert.pem \
  -keystore ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit \
  -noprompt

# 2.2. TrustStoreを表示
keytool -list -v \
  -keystore ssl/truststore.p12 \
  -storetype PKCS12 \
  -storepass changeit
```

##### 3. KeycloakでのmTLS有効化
Keycloak側でクライアント認証を受け付けるように設定します。`KC_HTTPS_CLIENT_AUTH` に `request` または `required` を設定し、作成したTrustStoreをマウントします。

```yaml:docker-compose.yml
  keycloak:
    # ...
    environment:
      # mTLS設定
      KC_HTTPS_TRUST_STORE_FILE: "/opt/keycloak/conf/truststore.p12"
      KC_HTTPS_TRUST_STORE_PASSWORD: "changeit"
      KC_HTTPS_TRUST_STORE_TYPE: "PKCS12"
      KC_HTTPS_CLIENT_AUTH: "request" # クライアント証明書を要求する
    volumes:
      # ...
      - ./ssl/truststore.p12:/opt/keycloak/conf/truststore.p12:ro
```

##### 4. Spring Bootでのクライアント証明書利用設定
最後に、Spring BootがKeycloakへのリクエスト時にクライアント証明書を提示できるよう、設定ファイルにパスを記述します。Java側ではこのKeystoreをロードして `SSLContext` を構築することになります。

```properties:application.properties
# KeycloakへのmTLS接続用 KeyStore
application.keycloak.mtls.key-store=ssl/oidc-mtls-client-keystore.p12
application.keycloak.mtls.key-store-password=changeit
application.keycloak.mtls.key-store-type=PKCS12
```

##### 5. OIDCクライアント実装の修正

:::message
OIDCクライアントのMTI
- mTLS経由で`Token Request`を実行する
:::

以下のシーケンスを実現します。
![](/images/2.2.1.mTLS_TokenRequest.png)
*[2.2.1.mTLS_TokenRequest.pu](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/docs/2.%E3%82%B7%E3%83%BC%E3%82%B1%E3%83%B3%E3%82%B9/plantuml/2.2.1.mTLS_TokenRequest.pu)*

###### Token Request with mTLS

トークン取得処理のエントリポイントは`TokenClientService.requestToken()`です。トークンエンドポイントに対して`application/x-www-form-urlencoded`で`HTTP POST`を投げ、そのレスポンスボディを返却します。

```java:TokenClientService.java
@Service
public class TokenClientService {

    private static final Logger logger = LoggerFactory.getLogger(TokenClientService.class);
    private final KeycloakHttpClientFactory keycloakHttpClientFactory;

    public TokenClientService(KeycloakHttpClientFactory keycloakHttpClientFactory) {
        this.keycloakHttpClientFactory = keycloakHttpClientFactory;
    }

    public String requestToken(String tokenEndpoint, Map<String, String> formParams) throws Exception {
        logger.debug("start method: {}", TokenClientService.class.getName() + ".requestToken");
        logger.debug("Requesting token from: {}", tokenEndpoint);
        formParams.forEach((k, v) -> logger.debug("Param: {} = {}", k, v));

        // ★ mTLS 対応 HttpClient をここで取得
        HttpClient httpClient = keycloakHttpClientFactory.getHttpClient();

        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(tokenEndpoint))
                .header("Content-Type", "application/x-www-form-urlencoded")
                .POST(buildForm(formParams))
                .build();

        HttpResponse<String> response = httpClient.send(request, HttpResponse.BodyHandlers.ofString());
        logger.debug("Token response status: {}", response.statusCode());
        logger.debug("Token response body: {}", response.body());

        return response.body();
    }

    private static HttpRequest.BodyPublisher buildForm(Map<String, String> formParams) {
        String body = formParams.entrySet().stream()
                .map(e -> URLEncoder.encode(e.getKey(), StandardCharsets.UTF_8)
                        + "="
                        + URLEncoder.encode(e.getValue(), StandardCharsets.UTF_8))
                .collect(Collectors.joining("&"));
        return HttpRequest.BodyPublishers.ofString(body);
    }
}
```

###### KeycloakHttpClientFactoryが担うmTLS通信の準備

`KeycloakHttpClientFactory`は、クライアント証明書・秘密鍵・信頼済みCA を読み込み、これらを使ってmTLS対応の`SSLContext`と`HttpClient`を構築します。

```java:KeycloakHttpClientFactory.java
package com.example.oidcclient.http;

import com.example.oidcclient.config.properties.MtlsProperties;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import javax.net.ssl.*;
import java.io.InputStream;
import java.net.http.HttpClient;
import java.security.KeyStore;
import java.security.SecureRandom;
import java.time.Duration;

@Component
public class KeycloakHttpClientFactory {

    private static final Logger logger = LoggerFactory.getLogger(KeycloakHttpClientFactory.class);

    private final MtlsProperties mtlsProperties;
    private final SSLContext sslContext;
    private final HttpClient httpClient;

    public KeycloakHttpClientFactory(MtlsProperties mtlsProperties) throws Exception {
        this.mtlsProperties = mtlsProperties;
        logger.debug("start method: {}.constructor", KeycloakHttpClientFactory.class.getName());

        // ★ mTLS 用 SSLContext を構築
        this.sslContext = buildSslContext();

        // ★ この SSLContext を使った HttpClient をビルド（再利用前提）
        this.httpClient = HttpClient.newBuilder()
                .sslContext(sslContext)
                .connectTimeout(Duration.ofSeconds(10))
                .build();
    }

    public HttpClient getHttpClient() {
        return httpClient;
    }

    private SSLContext buildSslContext() throws Exception {
        logger.debug("start method: {}.buildSslContext", KeycloakHttpClientFactory.class.getName());

        // 1) クライアント証明書キーストアのロード
        KeyStore keyStore = KeyStore.getInstance(mtlsProperties.getKeyStoreType());
        try (InputStream ksStream = getResource(mtlsProperties.getKeyStore())) {
            keyStore.load(ksStream, mtlsProperties.getKeyStorePassword().toCharArray());
        }

        KeyManagerFactory kmf = KeyManagerFactory.getInstance(KeyManagerFactory.getDefaultAlgorithm());
        kmf.init(keyStore, mtlsProperties.getKeyStorePassword().toCharArray());

        // 2) 信頼済み CA を保持するトラストストアのロード
        KeyStore trustStore = KeyStore.getInstance(mtlsProperties.getTrustStoreType());
        try (InputStream tsStream = getResource(mtlsProperties.getTrustStore())) {
            trustStore.load(tsStream, mtlsProperties.getTrustStorePassword().toCharArray());
        }

        TrustManagerFactory tmf = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
        tmf.init(trustStore);

        // 3) キーマネージャとトラストマネージャを使って SSLContext を初期化
        SSLContext context = SSLContext.getInstance("TLS");
        context.init(kmf.getKeyManagers(), tmf.getTrustManagers(), new SecureRandom());

        return context;
    }

    private InputStream getResource(String path) {
        InputStream is = Thread.currentThread().getContextClassLoader().getResourceAsStream(path);
        if (is == null) {
            throw new IllegalStateException("Resource not found: " + path);
        }
        return is;
    }
}
```

###### キーストアとトラストストアの役割

* **キーストア (KeyStore)**
クライアント証明書とその秘密鍵を保持します。mTLSでは、クライアント側もサーバに対して証明書を提示する必要があるため、このキーストアから秘密鍵付きの証明書チェーンを取り出し、KeyManagerとしてSSLContext に登録します。
* **トラストストア (TrustStore)**
「どの証明書を信頼するか」を定義するストアです。ここにはKeycloakサーバ証明書を発行したCA証明書、あるいは自己署名証明書そのものが格納されます。これをTrustManagerとしてSSLContextに登録することで、Keycloak側サーバ証明書の検証ができるようになります。

MtlsProperties からはこれらのパスやパスワード、タイプが注入されます（例: PKCS12 / JKS など）。

```java:MtlsPropertes.java
@ConfigurationProperties(prefix = "application.keycloak.mtls")
public class MtlsProperties {

    private String keyStore;
    private String keyStorePassword;
    private String keyStoreType;
    private String trustStore;
    private String trustStorePassword;
    private String trustStoreType;

    // getter/setter 省略
}
```

mTLS通信のための準備はすべて`KeycloakHttpClientFactory`に委譲しており、前述の`TokenClientService.requestToken()`は「mTLSが有効なHttpClientでトークンエンドポイントにHTTP POSTすること」のみに責務を絞っています。

具体的には、`TokenClientService.requestToken()`が`KeycloakHttpClientFactory.getHttpClient()`を呼び出すことで、
* クライアント証明書＋秘密鍵を含む`KeyManager`
* 信頼済みCA証明書を含む`TrustManager`を持つ`SSLContext`

によって初期化された`HttpClient`が返却されます。

この`HttpClient`でKeycloakのトークンEPに`HTTPS POST`することで、TLSハンドシェイクの中で
* サーバ証明書をトラストストアで検証
* クライアント証明書をサーバに提示し、Keycloak側のmTLS要求を満たす
* ハンドシェイクが成功すると、通常の HTTP/1.1 通信としてトークン応答を受け取る

という一連の処理を実行できます。

最終的には以下のようなシーケンスに沿ってmTLS通信によるトークン要求を実行する流れとなります。
![](/images/2.2.2.mTLS_TokenRequestDetails.png)
*[2.2.2.mTLS_TokenRequestDetails.png](https://github.com/takumi13/keycloak-idp-rp/blob/feature/ph02.3/security_enhanced/docs/2.%E3%82%B7%E3%83%BC%E3%82%B1%E3%83%B3%E3%82%B9/plantuml/2.2.2.mTLS_TokenRequestDetails.pu)*

### 3. デモンストレーション

ここでは実際に作成したOIDCクライアントの挙動を確認します。

#### 3.1. Authorization Request

OIDCクライアントから`Authorization Request`を開始するエンドポイントへアクセスします。
```
https://localhost:8081/authorization-flow
```
![](/images/3.1.AuthorizationRequest.png)

`Authorize`ボタンを押下すると、画面に記載のパラメータを使ってKeycloakへ`Authorization Request`を行います。
```
https://localhost:8443/realms/myrealm/protocol/openid-connect/auth
?response_type=code
&client_id=semi_client
&redirect_uri=https%3A%2F%2Flocalhost%3A8081%2Fcallback
&scope=openid
&state=ba757414-00a1-4202-a225-9a2e85ae9e71
&nonce=96a67adb-1cb3-40b1-8868-c0dabc2a57a4
&code_challenge=XrU8p7N-IR5D5NGiOddU1Lb7OKwj9pDP1UWy_dXM1Xk
&code_challenge_method=S256
```

その後KeycloakのUser Signinページが表示されるため、作成したUserのID/PWを入力して`Sign in`ボタンを押下します。
![](/images/3.1.Signin.png)

#### 3.2. Authorization Response

ID/PW認証に成功すると、（未実施であれば指定scopeに基づく属性提供同意画面が表示され、そこで同意すると）Keycloakは指定された`redirect_uri`に`Authorization Response`をHTTPリダイレクトします。
```
https://localhost:8081/callback
?state=ba757414-00a1-4202-a225-9a2e85ae9e71
&session_state=5a55d42a-96cf-8eb1-1dac-c5b2d6742b14
&iss=https%3A%2F%2Flocalhost%3A8443%2Frealms%2Fmyrealm
&code=b29387d8-9e8d-ecf4-2376-f4bb79b59532.5a55d42a-96cf-8eb1-1dac-c5b2d6742b14.7dd8b29d-c8da-459f-9f3c-545b753026a6
```
![](/images/3.2.AuthorizationResponse.png)

#### 3.3. Token Request

そのまま`トークン要求送信`ボタンを押下すると、以下の伝聞例に従って`Token Request`を実行します。
```
POST /token HTTP/1.1
Host: localhost:8080
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=b29387d8-9e8d-ecf4-2376-f4bb79b59532.5a55d42a-96cf-8eb1-1dac-c5b2d6742b14.7dd8b29d-c8da-459f-9f3c-545b753026a6
&redirect_uri=https%3A%2F%2Flocalhost%3A8081%2Fcallback
&client_id=semi_client
&client_secret=Gx6Yh6Q9mVWyPeg9GDIKgKzTiN8WU5Cf
&state=ba757414-00a1-4202-a225-9a2e85ae9e71
&code_verifier=VxRYK91fgxA48SL_IYt7-K9JPHl5.NF_TOYCuKfJvHtHP0dB..3YBBJYoBEFLU09GQ
```
![](/images/3.3.TokenRequest.png)

#### 3.4. Token Response

応答から`access_token`と`id_token`を受け取ることができます。画面表示前に、バックエンドでこれらのパラメータの`key-value`を2.1.4の方法に従って検証しています。
![](/images/3.4.TokenResponse.png)

## 4. おわりに

今回は前編として、OIDCクライアントをKeycloakのFAPI 1.0 Part1: Baselineポリシーに準拠させました。
次回記事ではFAPI 1.0 Part2: Advancedに準拠したOIDCクライアントを開発してみます。

おそらく、Qiitaアドカレ2025には間に合いません。。また年明けに記事を書こうと思います。

## 5. 参考文献

- Keycloak で クライアントポリシー + FAPI を試す with PKCE
  - https://www.creationline.com/tech-blog/cloudnative/keycloak/46316
- QuarkusベースのKeycloakが出力するログについて
  - https://qiita.com/tamura__246/items/cd96437725be9feea712
- Docker-Composeを使用した際のKeyCloakのデータを外部DBに依存させる方法
  - https://zenn.dev/emp_tech_blog/articles/996920de0b93b7