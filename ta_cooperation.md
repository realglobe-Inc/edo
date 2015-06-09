<!--
Copyright 2015 realglobe, Inc.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
-->


# TA 間連携プロトコル

TA から別の TA に処理を要請する際のプロトコル。


## 1. 概要

処理に関連するアカウントの情報を、アカウントの属す IdP を間に挟み、TA 間で受け渡す。

1. 関連するアカウントが属す全ての IdP について次の 1, 2 を行う。
    1. 要請元 TA から IdP に、関連するアカウントの中でその IdP に属すアカウントおよび要請先 TA を伝える。
    2. IdP から要請元 TA に、仲介コードを発行する。
2. 要請元 TA から要請先 TA に、全ての仲介コードを渡す。
3. 全ての仲介コードについて次の 1, 2 を行う。
    1. 要請先 TA から仲介コードを発行した IdP に、仲介コードを提示する。
    2. IdP から要請先 TA に、仲介コードに紐付くアカウントと要請元 TA の情報を渡す。

```
+--------+                                                     +--------+
|        |                     +--------+                      |        |
|        |----(1-1) request--->|        |                      |        |
|        |                     |  IdP   |                      |        |
|        |<---(1-2) code-------|        |                      |        |
|        |                     +--------+                      |        |
|        |         ...            ...                          |        |
|        |                                                     |        |
|        |                                                     |        |
|   TA   |---------------------(2) codes---------------------->|   TA   |
|        |                                                     |        |
|        |                     +--------+                      |        |
|        |                     |        |<---(3-1) code--------|        |
|        |                     |  IdP   |                      |        |
|        |                     |        |----(3-2) userinfo--->|        |
|        |                     +--------+                      |        |
|        |                        ...             ...          |        |
|        |                                                     |        |
+--------+                                                     +--------+
```


## 2. アカウントの種類

関連するアカウントは、処理の主体とそれ以外に分ける。

* **処理の主体**
    * ユーザー認証を通り、アクセストークンを要請元 TA に渡し、処理の引き金になるアクセスを行ったユーザーのアカウント。
* **処理の主体でないアカウント**
    * 処理の対象となるアカウント等。

例えば、要請元 TA と要請先 TA が両方グループウェアであるとする。
要請元 TA で同じグループに入っているアカウントを要請先 TA のグループへ招待する場合、招待する側のアカウントは処理の主体、招待される側のアカウントは処理の主体でないアカウントとなる。


## 3. アカウントタグ

要請元 TA は、関連するアカウント全てに対して、その処理において一意のタグをつけなければならない。


## 4. 要請元仲介リクエスト

要請元 TA から IdP の要請元仲介エンドポイントにリクエストを送る。

リクエストの中身は、処理の主体が IdP に属すかどうかで変わる。
また、処理の主体が属さない IdP へのリクエストには、処理の主体が属す IdP へのリクエストの結果が必要になるため、最初に処理の主体が属す IdP へのリクエストを行う必要がある。


### 4.1. 要請元仲介エンドポイント

IdP は要請元仲介エンドポイントを TLS で提供しなければならない。


### 4.2. 処理の主体が属す IdP へのリクエストパラメータ

処理の主体が属す IdP への要請元仲介リクエストは、以下で指定するパラメータを最上位要素として含む JSON オブジェクトである。

* **`response_type`**
    * 必須。
      関連するアカウントが他の IdP に属さない場合は `code_token`、属す場合は `code_token referral` でなければならない。
* **`from_client`**
    * 必須。
      要請元 TA の ID。
* **`to_client`**
    * 必須。
      要請先 TA の ID。
* **`grant_type`**
    * 必須。
      `access_token` でなければならない。
* **`access_token`**
    * 必須。
      処理の主体に対する有効なアクセストークン。
* **`scope`**
    * 任意。
      要請先 TA に新しく発行されるアクセストークンに対して許可されるスコープの最大範囲。
      形式は [OAuth 2.0 Section 3.3] を参照のこと。
      `access_token` で指定したアクセストークンに対して許可されていないスコープを含んではならない。
* **`expires_in`**
    * 任意。
      要請先 TA に新しく発行されるアクセストークンの有効期間の上限。
      `access_token` で指定したアクセストークンの残り有効期間より長い場合は無視される。
* **`user_tag`**
    * 必須。
      処理の主体に付けたアカウントタグ。
* **`users`**
    * 処理の主体でないアカウントが IdP に属す場合は必須。
      そうでなければ無し。
      IdP に属す処理の主体でないアカウント全てについて、アカウントタグからアカウント ID へのマップ。
* **`related_users`**
    * `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      他の IdP に属す関連するアカウント全てについて、アカウントタグからアカウントのハッシュ値へのマップ。
* **`hash_alg`**
    * `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      `related_users` におけるアカウントのハッシュ値計算アルゴリズム。
* **`related_issuers`**
    * `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      関連するアカウントが属す他の全ての IdP の ID の配列。


リクエスト時の TA 認証は必須であり、[OpenID Connect Core 1.0 Section 9] で定義されているクライアント認証方式を利用する。
その際、application/x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


#### 4.2.1. アカウントのハッシュ値

`related_users` におけるアカウントのハッシュ値は以下のように計算する。

アカウントが属す IdP の ID とアカウント ID をヌル文字で連結したバイト列をつくる。
それをハッシュ値計算アルゴリズムの入力にする。
出力されたバイト列の前半分を Base64URL エンコードする。
できた文字列がアカウントのハッシュ値である。

```
Base64URLEncode(LeftHalf(Hash(<IdP の ID> || <ヌル文字> || <アカウント ID>)))
```


ハッシュ値計算アルゴリズムとしては以下を認める。

* `SHA256`
* `SHA384`
* `SHA512`


#### 4.2.2. 処理の主体が属す IdP への要請元仲介リクエスト例

```HTTP
POST /coop/from HTTP/1.1
Host: idp.example.org
Content-Type: application/json

{
    "response_type": "code_token referral",
    "from_client": "https://from.example.org",
    "to_client": "https://to.example.org",
    "grant_type": "access_token",
    "access_token": "cYcFjo0EF7FiN8Jx1NJ8Wn51gcYl84",
    "user_tag": "inviter",
    "users": {
        "invitee": "md04LUMHnwLYodm8e55hc8UbGISJc4ZCYJK3AWF5IBk"
    },
    "related_users": {
        "observer": "Gbr93kvtHPkuUX4YlRD4QA"
    },
    "hash_alg": "SHA256",
    "related_issuers": [
        "https://idp2.example.org"
    ]
}
```

TA 認証用データは省いている。


### 4.3. 処理の主体が属さない IdP へのリクエストパラメータ

処理の主体が属さない IdP へのリクエストは、以下で指定するパラメータを最上位要素として含む JSON オブジェクトである。

* **`response_type`**
    * 必須。
      `code_token` でなければならない。
* **`grant_type`**
    * 必須。
      `referral` でなければならない。
* **`referral`**
    * 必須。
      処理の主体が属す IdP が発行した `referral`。
* **`users`**
    * 必須。
      IdP に属す関連するアカウント全てについて、アカウントタグからアカウント ID へのマップ。

リクエスト時の TA 認証は必須であり、[OpenID Connect Core 1.0 Section 9] で定義されているクライアント認証方式を利用する。
その際、application/x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


#### 4.3.1. 処理の主体が属さない IdP への要請元仲介リクエスト例

```HTTP
POST /coop/from HTTP/1.1
Host: idp2.example.org
Content-Type: application/json

{
    "response_type": "code_token",
    "grant_type": "referral",
    "referral": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOlsiaHR0cHM6Ly9pZHAyLmV4YW1wbGUub3
        JnIl0sImV4cCI6MTQyNTQ1MjgzNSwiaGFzaF9hbGciOiJTSEEyNTYiLCJpc3MiOiJodHRwcz
        ovL2lkcC5leGFtcGxlLm9yZyIsImp0aSI6InlHLTh4Zm1Pb1Q3RDRERE0iLCJyZWxhdGVkX3
        VzZXJzIjp7Im9ic2VydmVyIjoiR2JyOTNrdnRIUGt1VVg0WWxSRDRRQSJ9LCJzdWIiOiJodH
        RwczovL2Zyb20uZXhhbXBsZS5vcmciLCJ0b19jbGllbnQiOiJodHRwczovL3RvLmV4YW1wbG
        Uub3JnIn0.YjH8_n8D00qjIDtbBQW1JpkxBoDFs78Eepo0Jn1WeI8PjTdDCiwZy8ZcvOfJTi
        soEFPunjWYplVE7wqUUV9mxw",
    "users": {
        "observer": "K30qzk6Ki1tBVOan-mte50JQrhmEfnf8hrW77zqL1jg"
    }
}
```

`referral` は[処理の主体が属す IdP からの要請元仲介レスポンス例](#main-from-response-example)のもの。
`referral` の改行とインデントは表示の都合による。
TA 認証用データは省いている。


### 4.4. 要請元仲介リクエストの検証

IdP は以下のように要請元仲介リクエストを検証しなければならない。
--f--> は失敗時のエラーレスポンスの `error` の値を示す。

* 要請元 TA を認証する。
    * --f--> `invalid_client`
* リクエストが必要なパラメータを含むことを確認する。
    * --f--> `invalid_request`
* `access_token` を含む場合、`access_token` の指すアクセストークンが有効であることを確認する。
    * --f--> `invalid_grant`
* `scope` を含む場合、`scope` の値が `access_token` の指すアクセストークンに対して許可されていないスコープを含まないことを確認する。
    * --f--> `invalid_scope`
* `to_client` を含む場合、`to_client` が指す TA が存在し、要請元 TA とは異なることを確認する。
    * --f--> `invalid_request`
* `users` を含む場合、`users` に含まれるアカウント ID が存在することを確認する。
    * --f--> `invalid_request`
* `related_issuers` を含む場合、`related_issuers` に含まれる IdP が存在することを確認する。
    * --f--> `invalid_request`
* 異なるアカウントに同じアカウントタグが付けられていないことを確認する。
    * --f--> `invalid_request`
* `referral` を含む場合、`referral` の値を署名済み [JWT] として検証する。
    * --f--> `invalid_grant`
* `referral` を含む場合、`referral` の値の [JWT] が必要なクレームを含むことを確認する。
    * --f--> `invalid_grant`
* `referral` を含む場合、`referral` の値の [JWT] が含む `to_client` クレームが指す TA が存在し、要請元 TA とは異なることを確認する。
    * --f--> `invalid_grant`
* `referral` を含む場合、`referral` の値の [JWT] が含む `related_users` クレームに、`users` に含まれるアカウントタグが全て含まれており、そのハッシュ値が計算したハッシュ値と一致することを確認する。
    * --f--> `invalid_grant`


## 5. 要請元仲介レスポンス

リクエストに問題が無ければ、IdP から要請元 TA にレスポンスを返す。


### 5.1. 要請元仲介レスポンスパラメータ

レスポンスは以下の最上位要素を含む JSON オブジェクトである。

* **`code_token`**
    * リクエストの `response_type` が `code_token` を含む場合は必須。
      そうでなければ無し。
      以下のクレームを含む署名済み [JWT]。
        * **`iss`**
            * 必須。
              IdP の ID。
        * **`sub`**
            * 必須。
              仲介コード。
        * **`aud`**
            * 必須。
              要請先 TA の ID。
        * **`from_client`**
            * リクエストに `from_client` が含まれていた場合は必須。
              そうでなければ無し。
              リクエストの `from_client` の値そのまま。
        * **`user_tag`**
            * リクエストに `user_tag` が含まれていた場合は必須。
              そうでなければ無し。
              リクエストの `user_tag` の値そのまま。
        * **`user_tags`**
            * リクエストに `users` が含まれていた場合は必須。
              そうでなければ無し。
              リクエストの `users` に含まれていた全てのアカウントタグの配列。
        * **`ref_hash`**
            * リクエストの `response_type` が `referral` を含む、または、リクエストの `grant_type` が `referral` の場合は必須。
              そうでなければ無し。
              リクエストの `response_type` が `referral` を含む場合は発行した `referral` のハッシュ値。
              リクエストの `grant_type` が `referral` の場合はリクエストの `referral` のハッシュ値。
              ハッシュ値はハッシュ値計算アルゴリズムの出力バイト列の前半分を Base64URL エンコードしたものである。
              ハッシュ値計算アルゴリズムは `referral` の値の [JWT] が含む `hash_alg` クレームと同じでなければならない。
* **`referral`**
    * リクエストの `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      以下のクレームを含む署名済み [JWT]。
        * **`iss`**
            * 必須。
              IdP の ID。
        * **`sub`**
            * 必須。
              要請元 TA の ID。
        * **`aud`**
            * 必須。
              リクエストの `related_issuers` の内容そのまま。
        * **`exp`**
            * 必須。
              有効期限。
              [JWT Section 4.1.4] を参照のこと。
        * **`jti`**
            * 必須。
              トークン ID。
              [JWT Section 4.1.7] を参照のこと。
        * **`to_client`**
            * 必須。
              リクエストの `to_client` の値そのまま。
        * **`related_users`**
            * 必須。
              リクエストの `related_users` の内容そのまま。
        * **`hash_alg`**
            * 必須。
              `related_users` におけるアカウントのハッシュ値計算アルゴリズム。


#### 5.1.1. 処理の主体が属す IdP からの要請元仲介レスポンス例<a name="main-from-response-example" />

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code_token": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3Jn
        IiwiZnJvbV9jbGllbnQiOiJodHRwczovL2Zyb20uZXhhbXBsZS5vcmciLCJpc3MiOiJodHRw
        czovL2lkcC5leGFtcGxlLm9yZyIsInJlZl9oYXNoIjoic1JJYWRBOExoTVNiN01VaXpiVEdY
        QSIsInN1YiI6InA5LUZweFh4WEJ0NVBRck05LTZULXQzbDllU3oxbiIsInVzZXJfdGFnIjoi
        aW52aXRlciIsInVzZXJfdGFncyI6WyJpbnZpdGVlIl19.jDVEXyRvEwx0CV26I5Hl2JxWgco
        6col4KobOiQSbEEcmGcyMCP8GX_tW8xEctpEPuTLzUBfMnjBaJGkZzH67YA",
    "referral": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOlsiaHR0cHM6Ly9pZHAyLmV4YW1wbGUub3
        JnIl0sImV4cCI6MTQyNTQ1MjgzNSwiaGFzaF9hbGciOiJTSEEyNTYiLCJpc3MiOiJodHRwcz
        ovL2lkcC5leGFtcGxlLm9yZyIsImp0aSI6InlHLTh4Zm1Pb1Q3RDRERE0iLCJyZWxhdGVkX3
        VzZXJzIjp7Im9ic2VydmVyIjoiR2JyOTNrdnRIUGt1VVg0WWxSRDRRQSJ9LCJzdWIiOiJodH
        RwczovL2Zyb20uZXhhbXBsZS5vcmciLCJ0b19jbGllbnQiOiJodHRwczovL3RvLmV4YW1wbG
        Uub3JnIn0.YjH8_n8D00qjIDtbBQW1JpkxBoDFs78Eepo0Jn1WeI8PjTdDCiwZy8ZcvOfJTi
        soEFPunjWYplVE7wqUUV9mxw"
}
```

[JWT] の改行とインデントは表示の都合による。

`code_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp.example.org",
    "sub": "p9-FpxXxXBt5PQrM9-6T-t3l9eSz1n",
    "aud": "https://to.example.org",
    "from_client": "https://from.example.org",
    "user_tag": "inviter",
    "user_tags": [
        "invitee"
    ],
    "ref_hash": "sRIadA8LhMSb7MUizbTGXA"
}
```

`referral` のクレームセットの内容は、

```json
{
    "iss": "https://idp.example.org",
    "sub": "https://from.example.org",
    "aud": [
        "https://idp2.example.org"
    ],
    "exp": 1425452835,
    "jti": "yG-8xfmOoT7D4DDM",
    "to_client": "https://to.example.org",
    "related_users": {
        "observer": "Gbr93kvtHPkuUX4YlRD4QA"
    },
    "hash_alg": "SHA256"
}
```

署名には以下の鍵を用いた。

```json
{
    "kty": "EC",
    "crv": "P-256",
    "x": "3tfF_QYgrjnyDzRPycEyx0yZUvX2xZS8JFQb74c91Og",
    "y": "uTrU4RGQ6ospbXLTKEQZnNSQavQ6IeWcm1EIqNWleOg",
    "d": "8VSjWeZq44mW3GXSd9pXHcERnrB2D6FEj5Lw59QgNBo"
}
```


#### 5.1.2. 処理の主体が属さない IdP からの要請元仲介レスポンス例<a name="sub-from-response-example" />

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code_token": "eyJhbGciOiJSUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3Jn
        IiwiaXNzIjoiaHR0cHM6Ly9pZHAyLmV4YW1wbGUub3JnIiwicmVmX2hhc2giOiJzUklhZEE4
        TGhNU2I3TVVpemJUR1hBIiwic3ViIjoieG9LcE9LQ2xEM1VRTktuOHBscXd0NEVGd3VuLVZy
        IiwidXNlcl90YWdzIjpbIm9ic2VydmVyIl19.s5Z_ipRg-lbbGrQnjlXq6HksZh01SpSKZeN
        rL7R2MSTZ3ZCtPlXMIMVscSaP5FabatDaBAVKBhA6RFvJXvLSWVwtFX2v25-aNWMHywgmtnR
        EKyrmcjNylZ4s52MEO0um35nFqXqr4BDdk0xS70jQt41Qg8tJiUm7kKjiy74t_Hp5s82Hpxy
        7zJ4BOUe3laexZkU42mBf383_jqEyDL4CMndtfaCkVKv14M1hjfP8cscRGuNT_ZDpMDodV54
        ubjtVc6rsNRnCTwd9DZU2UE5CkXSu930hOw8LnGlk56QN7xloI0EEWZdohZFkDLkxv84ozWm
        66VuAIlSuvE8WRd3fHQ"
}
```

[JWT] の改行とインデントは表示の都合による。

`code_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp2.example.org",
    "sub": "xoKpOKClD3UQNKn8plqwt4EFwun-Vr",
    "aud": "https://to.example.org",
    "user_tags": [
        "observer"
    ],
    "ref_hash": "sRIadA8LhMSb7MUizbTGXA"
}
```

署名には以下の鍵を用いた。

```json
{
    "kty": "RSA",
    "n": "2Tv_db3EjCJYgY4USmfVd2yjMZroIV4b9VR2SgW71Rq85likqd8hvcXI6BZV6rdKpqDEzQ
        1u-n3R5L2-9SCrece4HbngprXsvuYluLEtOkdrJy14ne55vawEstF9TRCcw7GT0aY-x91Uc-
        3Lc-D9FirkZua4Bz-mqQfFoe37XXZr_FFxk0yws9tyzbnj9Rl4lJs9fecVTP8qNjg8UBOTaV
        fb716Qud3fKepTkyhyeet3x5tAn_MTGJAbGhoFnzFvcyUFbY9eGv67wd50TxXD9RAo98IfBN
        LxXukxngnFeCSRom4u1Pvh4UNqedOBOF6hU1CmRdAzksvd6zqhRCpjQQ",
    "e": "AQAB",
    "d": "tdVj0uFiiM4i-Wa9Ez7wzsMPovEARxXoHyVw0isUe5-i6MjgJBTSwG-y6JnxlsOP6AQAV4
        dcEq6Ip4gwNC0Be7EyKyewpLr5DR3GC1-69OJdDfEg2kmIe7xmPmveD0jNf3y_t6gJOvDHxT
        YRq9By6EBE6OFMvzyHO0t3IUD7u9FdWSOIZu0OE_fpoEOzvmOnNOyjocOWxJRjTPFVFLJE4m
        REXxksb3HZ0g2NlvFquiGNRtWD4zdMJ49UyjEgO3eJQ-fWrsyPJPi08S91715jjgd1MoEzz2
        xrFuIbO7IPhiVmjgdVPER61LMg7TiKHCklhU7MlR8H7pSnHkYqVMcLAQ",
    "p": "_e_r1JP9FeqYyFeTbhZoDDUm_vaziFAuZwLDfwWttPnbsWzE_puDEeOY8PbsvJd7fmbX2t
        pnSoOfuYW6b3fh3SxpHNcvDHK7qbC22DGI_wkxY01aHV_LfS9GljCqXP7bBWdoXAqc8678Jm
        QQ2MJwznVkp5X1N6XvHeoJoCg8B5E",
    "q": "2v_AM0foiU3_3T-Hpz9igct-OnT3tkvUuZ5nbN6ejWVBRN2ogtBLN_SVfgxMgLNTcxG0vt
        dVpMZofsRobFnyA9tGH50pFGsr04Uf3gOvhbf1QB6WBxoLhoUh6xMJ1Y1Ab2XXDX6ZauAObX
        QXJ_N4hgckIYAv5ODaKoRr-W2SqLE",
    "dp": "ZwZY-sUT0Dl-xQFq6iYjDpjd-mFi03IccWSYpkdKg3s_m8tSXS4Azlg1q8WypI0c6FqXR
        s6HS579RYqw6hqMQ2yKNM5E41sFMkJk3G-0cixrois23WYJK__rNnIGHHa1q4qZt4YCyYb7_
        CNrBlZU6B6OuMNJWstyqQNT5muMd1E",
    "dq": "c71uiquaTdaXPwrwWoe5O_ecArEGqaVyC5eX-YW-LeQxln-_K0OCPVRaHX_KfspHdC0LZ
        UDQ1oC1gSm0Nm9i5H7ilJqut0fcpbFZojA4d2c9imGf0KkHJlT-FAq_y8kXIMil20_pLP61I
        UuVYVvfepYTllD0_vWG16mclvo95EE",
    "qi": "8tKplcbM5koJZEnZrjulDQNme1CQClHrEip-nPrvSQSd8yAC5MVFLl1GYq8MSNLT9eJ7W
        L2n9MOj1bBy2649qYiE0G2XEULgYPyVu5gciOConXDjSfhcZWxZcOMm3KzTT6451an6reHDY
        efcjdlgCQmU498Thy7XZnJhCwUFXgE"
}
```


### 5.2. 要請元仲介レスポンスの検証

要請元 TA は要請元仲介レスポンスのパラメータとその [JWT] に含まれるクレームを検証するべきである。
特に、`user_tag` と `user_tags` クレームにリクエストしたアカウントタグが含まれていることを確認すべきである。
また、[JWT] の署名を検証しても良い。


## 6. 処理要請リクエスト

要請元 TA から要請先 TA の処理エンドポイントに、仲介データを付加してリクエストを送る。


### 6.1. 処理要請エンドポイント

処理要請エンドポイントは TLS で提供しなければならない。


### 6.2. 仲介データ

仲介データは IdP から受け取った全ての `code_token` 値の配列である。


### 6.3. 付加方法

仲介データの付加は URL クエリ、HTTP ヘッダのどちらかで行う。


#### 6.3.1. URL クエリによる付加

空白区切りで `code_tokens` パラメータに入れる。


##### 6.3.1.1. URL クエリによる付加例

```http
GET /api/invite/invitee?
    code_tokens=eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnIiw
    iZnJvbV9jbGllbnQiOiJodHRwczovL2Zyb20uZXhhbXBsZS5vcmciLCJpc3MiOiJodHRwczovL2l
    kcC5leGFtcGxlLm9yZyIsInJlZl9oYXNoIjoic1JJYWRBOExoTVNiN01VaXpiVEdYQSIsInN1YiI
    6InA5LUZweFh4WEJ0NVBRck05LTZULXQzbDllU3oxbiIsInVzZXJfdGFnIjoiaW52aXRlciIsInV
    zZXJfdGFncyI6WyJpbnZpdGVlIl19.jDVEXyRvEwx0CV26I5Hl2JxWgco6col4KobOiQSbEEcmGc
    yMCP8GX_tW8xEctpEPuTLzUBfMnjBaJGkZzH67YA%20
    eyJhbGciOiJSUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnIiwiaXNzIjoiaHR
    0cHM6Ly9pZHAyLmV4YW1wbGUub3JnIiwicmVmX2hhc2giOiJzUklhZEE4TGhNU2I3TVVpemJUR1h
    BIiwic3ViIjoieG9LcE9LQ2xEM1VRTktuOHBscXd0NEVGd3VuLVZyIiwidXNlcl90YWdzIjpbIm9
    ic2VydmVyIl19.s5Z_ipRg-lbbGrQnjlXq6HksZh01SpSKZeNrL7R2MSTZ3ZCtPlXMIMVscSaP5F
    abatDaBAVKBhA6RFvJXvLSWVwtFX2v25-aNWMHywgmtnREKyrmcjNylZ4s52MEO0um35nFqXqr4B
    Ddk0xS70jQt41Qg8tJiUm7kKjiy74t_Hp5s82Hpxy7zJ4BOUe3laexZkU42mBf383_jqEyDL4CMn
    dtfaCkVKv14M1hjfP8cscRGuNT_ZDpMDodV54ubjtVc6rsNRnCTwd9DZU2UE5CkXSu930hOw8LnG
    lk56QN7xloI0EEWZdohZFkDLkxv84ozWm66VuAIlSuvE8WRd3fHQ HTTP/1.1
Host: to.example.org
```

改行とインデントは表示の都合による。


#### 6.3.2. HTTP ヘッダによる付加

カンマ区切りで X-Edo-Code-Tokens ヘッダに入れる。


##### 6.3.2.1. HTTP ヘッダによる付加例

```http
GET /api/invite/invitee HTTP/1.1
Host: to.example.org
X-Edo-Code-Tokens: eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3Jn
    IiwiZnJvbV9jbGllbnQiOiJodHRwczovL2Zyb20uZXhhbXBsZS5vcmciLCJpc3MiOiJodHRwczov
    L2lkcC5leGFtcGxlLm9yZyIsInJlZl9oYXNoIjoic1JJYWRBOExoTVNiN01VaXpiVEdYQSIsInN1
    YiI6InA5LUZweFh4WEJ0NVBRck05LTZULXQzbDllU3oxbiIsInVzZXJfdGFnIjoiaW52aXRlciIs
    InVzZXJfdGFncyI6WyJpbnZpdGVlIl19.jDVEXyRvEwx0CV26I5Hl2JxWgco6col4KobOiQSbEEc
    mGcyMCP8GX_tW8xEctpEPuTLzUBfMnjBaJGkZzH67YA,
    eyJhbGciOiJSUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnIiwiaXNzIjoiaHR
    0cHM6Ly9pZHAyLmV4YW1wbGUub3JnIiwicmVmX2hhc2giOiJzUklhZEE4TGhNU2I3TVVpemJUR1h
    BIiwic3ViIjoieG9LcE9LQ2xEM1VRTktuOHBscXd0NEVGd3VuLVZyIiwidXNlcl90YWdzIjpbIm9
    ic2VydmVyIl19.s5Z_ipRg-lbbGrQnjlXq6HksZh01SpSKZeNrL7R2MSTZ3ZCtPlXMIMVscSaP5F
    abatDaBAVKBhA6RFvJXvLSWVwtFX2v25-aNWMHywgmtnREKyrmcjNylZ4s52MEO0um35nFqXqr4B
    Ddk0xS70jQt41Qg8tJiUm7kKjiy74t_Hp5s82Hpxy7zJ4BOUe3laexZkU42mBf383_jqEyDL4CMn
    dtfaCkVKv14M1hjfP8cscRGuNT_ZDpMDodV54ubjtVc6rsNRnCTwd9DZU2UE5CkXSu930hOw8LnG
    lk56QN7xloI0EEWZdohZFkDLkxv84ozWm66VuAIlSuvE8WRd3fHQ
```

改行とインデントは表示の都合による。


### 6.4. 処理要請リクエストの検証

要請先 TA は以下のように処理要請リクエストを検証しなければならない。

* 仲介データが付加されていることを確認する。
* 仲介データの各 [JWT] を署名済み [JWT] として検証する。
* 仲介データの各 [JWT] が必要なクレームを含むことを確認する。
* 仲介データに複数の [JWT] が含まれる場合、`ref_hash` クレームが含まれ、値が等しいことを確認する。
* 仲介データの中に `user_tag` クレームを含む [JWT] がただ 1 つだけ存在することを確認する。
* 異なるアカウントに同じアカウントタグが付けられていないことを確認する。

検証に失敗したときのエラーレスポンスの形式は要請先 TA の裁量である。
`error` の値を `invalid_request` とした [OAuth 2.0 Section 5.2] 形式にすることを推奨する。


## 7. 要請先仲介リクエスト

要請先 TA から IdP の要請先仲介エンドポイントにリクエストを送る。


### 7.1. 要請先仲介エンドポイント

IdP は要請先仲介エンドポイントを TLS で提供しなければならない。


### 7.2. 要請先仲介リクエストパラメータ

要請先仲介リクエストは、以下の最上位要素を含む JSON オブジェクトである。

* **`grant_type`**
    * 必須。
      `cooperation_code` でなければならない。
* **`code`**
    * 必須。
      仲介コード。
* **`claims`**
    * 仲介コードに `user_tag` が付いていた場合は任意。
      そうでなければ無し。
      処理の主体に対する [OpenID Connect Core 1.0 Section 5.5] の `claims` パラメータと同じもの。
* **`user_claims`**
    * 仲介コードに `user_tags` が付いていた場合は任意。
      そうでなければ無し。
      アカウントタグから [OpenID Connect Core 1.0 Section 5.5] の `claims` パラメータの `id_token` 要素と同じものへのマップ。

リクエスト時の TA 認証は必須であり、[OpenID Connect Core 1.0 Section 9] で定義されているクライアント認証方式を利用する。
その際、application/x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。

要求した必須クレームに対する事前同意が無ければ拒否される。


#### 7.2.1. 処理の主体が属す IdP への要請先仲介リクエスト例

```HTTP
POST /coop/to HTTP/1.1
Host: idp.example.org
Content-Type: application/json

{
    "grant_type": "cooperation_code",
    "code": "p9-FpxXxXBt5PQrM9-6T-t3l9eSz1n",
    "claims": {
        "id_token": {
            "pds": {
                "essential": true
            }
        }
    },
    "user_claims": {
        "invitee": {
            "pds": {
                "essential": true
            }
        }
    }
}
```

TA 認証用データは省いている。


#### 7.2.2. 処理の主体が属さない IdP への要請先仲介リクエスト例

```HTTP
POST /coop/to HTTP/1.1
Host: idp2.example.org
Content-Type: application/json

{
    "grant_type": "cooperation_code",
    "code": "xoKpOKClD3UQNKn8plqwt4EFwun-Vr"
}
```

TA 認証用データは省いている。


### 7.3. 要請先仲介リクエストの検証

IdP は以下のように要請先仲介リクエストを検証しなければならない。
--f--> は失敗時のエラーレスポンスの `error` の値を示す。

* 要請元 TA を認証する。
    * --f--> `invalid_client`
* リクエストが必要なパラメータを含むことを確認する。
    * --f--> `invalid_request`
* `code` の指す仲介コードが有効であることを確認する。
    * --f--> `invalid_grant`
* `code` の指す仲介コードが要請先 TA 用に発行されたものであることを確認する。
    * --f--> `invalid_grant`
* `claims` を含むなら、`code` の指す仲介コードがアクセストークンに紐付くことを確認する。
    * --f--> `invalid_grant`
* `user_claims` を含むなら、`user_claims` には `code` の指す仲介コードに紐付くアカウントタグのみが含まれることを確認する。
    * --f--> `invalid_grant`
* 要求された必須クレームに対する事前同意があることを確認する。
    * --f--> `access_denied`


## 8. 要請先仲介レスポンス

リクエストに問題が無ければ、IdP から要請先 TA にレスポンスを返す。


### 8.1. 要請先仲介レスポンスパラメータ

要請先仲介レスポンスは以下の最上位要素を含む JSON オブジェクトである。

* **`access_token`**
    * 仲介コードがアクセストークンに紐付いていた場合は必須。
      そうでなければ無し。
      新しく発行した処理の主体に対するアクセストークン。
* **`token_type`**
    * `access_token` を含む場合のみ。
      [OAuth 2.0 Section 5.1] の `token_type` と同じもの。
* **`expires_in`**
    * `access_token` を含む場合のみ。
      [OAuth 2.0 Section 5.1] の `expires_in` と同じもの。
* **`scope`**
    * `access_token` を含む場合のみ。
      [OAuth 2.0 Section 5.1] の `scope` と同じもの。
* **`ids_token`**
    * 以下のクレームを含む署名済み [JWT]。
        * **`iss`**
            * 必須。
              IdP の ID。
        * **`sub`**
            * 必須。
              要請元 TA の ID。
        * **`aud`**
            * 必須。
              要請先 TA の ID。
        * **`exp`**
            * 必須。
              有効期限。
              [ID トークン]を参照のこと。
        * **`iat`**
            * 必須。
              発行日時。
              [ID トークン]を参照のこと。
        * **`ids`**
            * 必須。
              仲介コードに紐付く全てのアカウントについて、アカウントタグからアカウントの [ID トークン]に含まれるべき `iss` 以外の属性情報を表すクレームセットへのマップ。
              処理の主体のものもそれ以外のものも含む。


#### 8.1.1. 処理の主体が属す IdP からの要請先仲介レスポンス例

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "access_token": "EM0VI_NbAd-ClytRWFQU3WNONsZRYr",
    "expires_in": 1037,
    "ids_token": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnI
        iwiZXhwIjoxNDI1NDUzNzAyLCJpYXQiOjE0MjU0NTI2NjUsImlkcyI6eyJpbnZpdGVlIjp7I
        nBkcyI6eyJ0eXBlIjoic2luZ2xlIiwidXJpIjoiaHR0cHM6Ly9wZHMuZXhhbXBsZS5vcmcvI
        n0sInN1YiI6ImI3d1BOS0dEbUdTRTRjRmE2Z3Q3THlNdERwNVYzcmJJUnVOQmdfLWdlLVEif
        SwiaW52aXRlciI6eyJwZHMiOnsidHlwZSI6InNpbmdsZSIsInVyaSI6Imh0dHBzOi8vcGRzL
        mV4YW1wbGUub3JnLyJ9LCJzdWIiOiJESUUwUER6N1JyY29Cdmo2M2t4Z2hUMkVaT2xiUHBuQ
        3BFbUlXRUNzazdVIn19LCJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLm9yZyIsInN1YiI6I
        mh0dHBzOi8vZnJvbS5leGFtcGxlLm9yZyJ9.gRkwnH2utpBNykR8yeRmSJH-Y0qro-U85wm9
        v8aX_4ZzGWU-AFCrvF4BhiCDWi9pgS0zSPQ_YtlFbCHT8VY7Tg"
}
```

[JWT] の改行とインデントは表示の都合による。

`ids_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp.example.org",
    "sub": "https://from.example.org",
    "aud": "https://to.example.org",
    "exp": 1425453702,
    "iat": 1425452665,
    "ids": {
        "inviter": {
            "sub": "DIE0PDz7RrcoBvj63kxghT2EZOlbPpnCpEmIWECsk7U",
            "pds": {
                "type": "single",
                "uri": "https://pds.example.org/"
            }
        },
        "invitee": {
            "sub": "b7wPNKGDmGSE4cFa6gt7LyMtDp5V3rbIRuNBg_-ge-Q",
            "pds": {
                "type": "single",
                "uri": "https://pds.example.org/"
            }
        }
    }
}
```

署名に使用した鍵は[処理の主体が属す IdP からの要請元仲介レスポンス例](#main-from-response-example)と同じ。


#### 8.1.2. 処理の主体が属さない IdP からの要請先仲介レスポンス例

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "ids_token": "eyJhbGciOiJSUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnI
        iwiZXhwIjoxNDI1NDUzNzA1LCJpYXQiOjE0MjU0NTI2NjgsImlkcyI6eyJvYnNlcnZlciI6e
        yJzdWIiOiItdkE1Y2FTVkFFQ1daT19QTW91eVl6dlFFZUZWSXVOTXo5eEdFRUxXZGRvIn19L
        CJpc3MiOiJodHRwczovL2lkcDIuZXhhbXBsZS5vcmciLCJzdWIiOiJodHRwczovL2Zyb20uZ
        XhhbXBsZS5vcmcifQ.hQFSF7kZBHQI-cF9kP4WYcWoWve_YICgaUTu8FnzkYDXd7v2FOctff
        3akGhQUBFjM6ZtX3BInBxVGfKxkD7aZP5E44KM8VMXYineyrEpcGeTehuIFnqjkOnzHz-yxe
        SLAtpTN-SPq0cupWeOwMFo2ehp6pHViJ34BpVhnkS8eTRXeGvlidA_Zzg5u8X5zpVcz9klVq
        g-wSBdfbojTgV5xdxaIS9X7pa7-jm2-2o-Y54WO1umh_EgXWYOTdj0H9wLYq7YFqN7ubQv-T
        j15ddA0xFkvxRt33D04521kjnhT5qn4Erx9iQcTSm6DqcEW2LhGEhgzu01EhNZHPYQYpMzWg"
}
```

[JWT] の改行とインデントは表示の都合による。

`ids_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp2.example.org",
    "sub": "https://from.example.org",
    "aud": "https://to.example.org",
    "exp": 1425453705,
    "iat": 1425452668,
    "ids": {
        "observer": {
            "sub": "-vA5caSVAECWZO_PMouyYzvQEeFVIuNMz9xGEELWddo"
        }
    }
}
```

署名に使用した鍵は[処理の主体が属さない IdP からの要請元仲介レスポンス例](#sub-from-response-example)と同じ。


### 8.2. 要請先仲介レスポンスの検証

要請先 TA は、以下のように要請先仲介レスポンスを検証しなければならない。

* 必要なパラメータを含むことを確認する。
* `ids_token` の値を署名済み [JWT] として検証する。
* `ids_token` の値の [JWT] に含まれる `ids` クレームに、リクエストに対応するアカウント情報が含まれることを確認する。


## 9. セッション

関連するアカウントが処理の主体のみ、かつ、要請先 TA に発行されたアクセストークンが有効である間は、上記プロトコルを省いても良い。
この場合、まず要請先 TA から要請元 TA へのレスポンスの際に、Set-Cookie ヘッダでセッションを宣言する必要がある。

|Cookie 名|値|
|:--|:--|
|Edo-Cooperation|セッション ID|

要請元 TA は、後の処理要請リクエストの際に、このセッションを Cookie ヘッダで宣言する。
アカウントタグは最初に用いたものがそのまま用いられる。


## 10. エラーレスポンス

IdP からのエラーは [OAuth 2.0 Section 5.2] の形式で返す。

要請先 TA において、本プロトコルでのエラーが発生した場合、要請元 TA へのレスポンスに以下のヘッダを含めなければならない。

|HTTP ヘッダ|値|
|:--|:--|
|X-Edo-Cooperation-Error|適切なメッセージ|

その他の形式は要請先 TA の裁量である。
`error` の値を `invalid_request` とした [OAuth 2.0 Section 5.2] 形式にすることを推奨する。


<!-- 参照 -->
[ID トークン]: http://openid.net/specs/openid-connect-core-1_0.html#IDToken
[JWT Section 4.1.4]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32#section-4.1.4
[JWT Section 4.1.7]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32#section-4.1.7
[JWT]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32
[OAuth 2.0 Section 3.3]: http://tools.ietf.org/html/rfc6749#section-3.3
[OAuth 2.0 Section 5.1]: http://tools.ietf.org/html/rfc6749#section-5.1
[OAuth 2.0 Section 5.2]: http://tools.ietf.org/html/rfc6749#section-5.2
[OpenID Connect Core 1.0 Section 5.5]: http://openid.net/specs/openid-connect-core-1_0.html#ClaimsParameter
[OpenID Connect Core 1.0 Section 9]: http://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication
