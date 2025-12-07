---
title: "KeycloakのFAPIポリシに準拠したOIDCクライアントを作る(SpringBoot) 中編"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Java", "SpringBoot", "Keycloak", "OIDC", "FAPI"]
published: false
---

## 1. はじめに

### 1.1. 自己紹介

（ToDo:自己紹介）

今回は中編です。前編の記事はこちらをご参照ください。

| 編 | 記事内容 |
| ---- | ---- |
| 前編 | FAPI1.0 Part1: Baselineに準拠したOIDCクライアントの開発 |
| 中編 | FAPI1.0 Part2: Advancedに準拠したOIDCクライアントの開発 |
| 後編 | FAPI 2.0 Security Profileに準拠したOIDCクライアントの開発 |

ということで本記事は前編です。
（中編/後編も間に合えばアドカレに載せたいが、、感触厳しそう）

### 1.2. 記事のゴール

| 分類 | ゴール |
| ---- | ---- |
| 仕様理解 | FAPI 1.0 Part1: Advancedに関する仕様を順を追って説明する |
| 実装理解 | FAPI 1.0 Part1: Advancedの要求事項に準拠したOIDCクライアントを開発し、keycloakからAPI正常応答が返るようにする |

### 1.3. 想定読者層

基本的には前編記事と同様です。
FAPI1 Part2: Advancedに準拠するために実装する必要がある以下については、本記事でも概要を説明します。

- [JWT-Secured Authorization Request (JAR)](https://www.rfc-editor.org/rfc/rfc9101.html)
- [Pushed Authorization Requests(PAR)](https://datatracker.ietf.org/doc/html/rfc9126)
- [JWT Secured Authorization Response Mode(JARM)](https://openid.net/specs/oauth-v2-jarm.html)

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
| tools | openssl | OpenSSL 1.1.1f  31 Mar 2020 |

### 1.5. ソースコード

本記事に掲載するソースコード(※)は以下の`feature/ph03/fapi1-part2-advanced`ブランチのものを一部記事用に修正して掲載しています。
https://github.com/takumi13/keycloak-idp-rp/tree/feature/ph03/fapi1-part2-advanced

### 1.6. 免責事項

ソースコードやシーケンスの大部分は`GPT-5.1-Codex (Preview)`を利用して作成していますが、OIDCに関する各種specやRFCと照らし合わせた処理の正当性確認は筆者自身が行っています。
また、セキュアコーディングに関しても（私の拙い知見を基に）可能な限り注意してレビューしています。

上記の前提で、コメント、Twitter(現X)等にていただいたご指摘は謹んで受け取り、コード改修/記事反映に努めさせていただきます。

## 2. FAPI準拠クライアントへの成長サクセスストーリー

:::message
OIDCクライアントのMTI
- MTIはMandatory To Implementの略で「実装必須」と訳されます
- シーケンス中のOIDCクライアントの各処理を「OIDCクライアントのMTI」として本メッセージ形式で記載します
:::

### 2.1. FAPI 1.0 Part2: Advanced対応

FAPI 1.0 Part2: Advancedは、以下の対応を講じることで要求仕様を満たすことができます。
本編では `mTLS` + `PAR` + `JARM` の組み合わせ（★採用）を利用します。

前編のFAPI1.0 Part1: Baseline時点でOIDCクライアントは`mTLS`に対応済なので、本編では`PAR`, `JARM`に対応します。

1. **クライアント認証の強化: 以下いずれかのクライアント認証方式を採用**
    - `tls_client_auth`
    - `self_signed_tls_client_auth（mTLS）` ★採用
    - `private_key_jwt`
2. **署名付きJWTリクエストオブジェクト (Signed JWT Request Object) の利用: `Authorization Request`として以下いずれかの拡張仕様に対応したリクエストを実行する**
      - [JWT-Secured Authorization Request (JAR)](https://www.rfc-editor.org/rfc/rfc9101.html)
      - [Pushed Authorization Requests(PAR)](https://datatracker.ietf.org/doc/html/rfc9126) ★採用
3. **認可レスポンスの保護: 以下いずれかの方式を採用**
    - OPが以下の仕様に従って返却する署名付きJWT型の認可応答を検証する ★採用
      - [JWT Secured Authorization Response Mode(JARM)](https://openid.net/specs/oauth-v2-jarm.html)
    - OpenID ConnectのHybrid Flow(response_type=code+id_token)を活用することでIDトークンを分離署名として使用
      - IDトークンには、認可レスポンスのパラメータ(code, state)のハッシュ値: `s_hash (State hash value) `が含まれ、発行者(OP)による分離署名として機能する。
      - OIDCクライアントは`state`を検証することで認可レスポンスが改ざんされていないことを確認

#### ○ PAR対応


#### ○ JARM対応