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

処理に関連するユーザーの情報を、ユーザーの属す IdP を間に挟み、TA 間で受け渡す。

1. 関連するユーザーが属す全ての IdP について次の 1, 2 を行う。
    1. 要請元 TA から IdP に、関連するユーザーの中でその IdP に属すユーザーおよび要請先 TA を伝える。
    2. IdP から要請元 TA に、仲介コードを発行する。
2. 要請元 TA から要請先 TA に、全ての仲介コードを渡す。
3. 全ての仲介コードについて次の 1, 2 を行う。
    1. 要請先 TA から仲介コードを発行した IdP に、仲介コードを提示する。
    2. IdP から要請先 TA に、仲介コードに紐付くユーザーと要請元 TA の情報を渡す。

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


## 2. ユーザーの種類

関連するユーザーは、処理の主体とそれ以外に分ける。

* **処理の主体**
    * ユーザー認証を通り、アクセストークンを要請元 TA に渡し、処理の引き金になるアクセスを行ったユーザー。
* **処理の主体でないユーザー**
    * 処理の対象となるユーザー等。

例えば、要請元 TA と要請先 TA が両方グループウェアであるとする。
要請元 TA で同じグループに入っているユーザーを要請先 TA のグループへ招待する場合、招待する側のユーザーは処理の主体、招待される側のユーザーは処理の主体でないユーザーとなる。


## 3. ユーザータグ

要請元 TA は、関連するユーザー全てに対して、その処理において一意のタグをつけなければならない。


## 4. 要請元仲介リクエスト

要請元 TA から IdP の要請元仲介エンドポイントにリクエストを送る。

リクエストの中身は、処理の主体が IdP に属すかどうかで変わる。
また、処理の主体が属さない IdP へのリクエストには、処理の主体が属す IdP へのリクエスト結果が必要になるため、最初に処理の主体が属す IdP へのリクエストを行う必要がある。


### 4.1. 要請元仲介エンドポイント

IdP は要請元仲介エンドポイントを TLS で提供しなければならない。


### 4.2. 処理の主体が属す IdP へのリクエストパラメータ

処理の主体が属す IdP への要請元仲介リクエストは、以下で指定するパラメータを最上位要素として含む JSON オブジェクトである。

* **`response_type`**
    * 必須。
      関連するユーザーが他の IdP に属さない場合は `code_token`、属す場合は `code_token referral` でなければならない。
* **`to_ta`**
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
      要請先 TA に新しく発行されるアクセストークンのスコープの最大範囲。
      `access_token` で指定したアクセストークンに許可されていないスコープを含んではならない。
* **`expires_in`**
    * 任意。
      要請先 TA に新しく発行されるアクセストークンの有効期間の上限。
      `access_token` で指定したアクセストークンの残り有効期間より長い場合は無視される。
* **`user_tag`**
    * 必須。
      処理の主体に付けたユーザータグ。
* **`users`**
    * 処理の主体でないユーザーが IdP に属す場合は必須。
      そうでなければ無し。
      IdP に属す処理の主体でないユーザー全てについて、ユーザータグからユーザー ID へのマップ。
* **`related_users`**
    * `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      他の IdP に属す関連するユーザー全てについて、ユーザータグからユーザー ID のハッシュ値へのマップ。
      ハッシュ値はハッシュ値計算アルゴリズムの出力バイト列の前半分を Base64URL エンコードしたものである。
* **`u_hash_alg`**
    * `related_users` におけるユーザー ID のハッシュ値計算アルゴリズムが IdP 側のデフォルトと異なる場合は必須。
      同じであれば任意。
      `related_users` におけるユーザー ID のハッシュ値計算アルゴリズム。
      以下のいずれかでなければならない。
        * `SHA256`
        * `SHA384`
        * `SHA512`
* **`related_idps`**
    * `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      関連するユーザーが属す他の全ての IdP の ID の配列。


リクエスト時の TA 認証は必須であり、[OpenID Connect Core 1.0 Section 9] で定義されているクライアント認証方式を利用する。
その際、application/x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


#### 4.2.1. 処理の主体が属す IdP への要請元仲介リクエスト例

```HTTP
POST /cooperation/from HTTP/1.1
Host: idp1.example.org
Content-Type: application/json

{
    "response_type": "code_token referral",
    "to_ta": "https://to.example.org",
    "grant_type": "access_token",
    "access_token": "HkXuWaG_bYv7OS0qLA1Q28_2rSA3ENkIsDYmRr_Dad84mfRKCO",
    "user_tag": "inviter",
    "users": {
        "invitee": "86504857780817848362"
    },
    "related_users": {
        "observer": "mHmlXWLkpYbgcHIzJKbkvQ"
    },
    "related_idps": [
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
      IdP に属す関連するユーザー全てについて、ユーザータグからユーザー ID へのマップ。

リクエスト時の TA 認証は必須であり、[OpenID Connect Core 1.0 Section 9] で定義されているクライアント認証方式を利用する。
その際、application/x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


#### 4.3.1. 処理の主体が属さない IdP への要請元仲介リクエスト例

```HTTP
POST /cooperation/from HTTP/1.1
Host: idp2.example.org
Content-Type: application/json

{
    "response_type": "code_token",
    "grant_type": "referral",
    "referral": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOlsiaHR0cHM6Ly9pZHAyLmV4YW1wbGUub3
        JnIl0sImV4cCI6MTQyNTQ1MjgzNSwiaXNzIjoiaHR0cHM6Ly9pZHAxLmV4YW1wbGUub3JnIi
        wianRpIjoiUXNQUVNUMnMxalNTbUtKZkxaam1udyIsInJlbGF0ZWRfdXNlcnMiOnsib2JzZX
        J2ZXIiOiJtSG1sWFdMa3BZYmdjSEl6Sktia3ZRIn0sInN1YiI6Imh0dHBzOi8vZnJvbS5leG
        FtcGxlLm9yZyIsInRvX3RhIjoiaHR0cHM6Ly90by5leGFtcGxlLm9yZyIsInVoYXNoX2FsZy
        I6IlNIQTI1NiJ9.L2HnJ96s2DpzSzM_lFHd5W2D9KAFa5CwLgl1Me3JiH5B7pMcUuOh1jYtG
        boTxl2KxHL-L_rtlW4rWg3UETAOCQ",
    "users": {
        "observer": "40053950180034613776"
    },
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
* `to_ta` を含む場合、`to_ta` が指す TA が存在し、要請元 TA とは異なることを確認する。
    * --f--> `invalid_request`
* `users` を含む場合、`users` に含まれるユーザー ID が存在することを確認する。
    * --f--> `invalid_request`
* `related_idps` を含む場合、`related_idps` に含まれる IdP が存在することを確認する。
    * --f--> `invalid_request`
* 異なるユーザーに同じユーザータグが付けられていないことを確認する。
    * --f--> `invalid_request`
* `referral` を含む場合、`referral` の値を署名済み [JWT] として検証する。
    * --f--> `invalid_grant`
* `referral` を含む場合、`referral` の値の [JWT] が必要なクレームを含むことを確認する。
    * --f--> `invalid_grant`
* `referral` を含む場合、`referral` の値の [JWT] が含む `to_ta` クレームが指す TA が存在し、要請元 TA とは異なることを確認する。
    * --f--> `invalid_grant`
* `referral` を含む場合、`referral` の値の [JWT] が含む `related_users` クレームに、`users` に含まれるユーザータグが全て含まれており、そのハッシュ値が計算したハッシュ値と一致することを確認する。
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
        * **`user_tag`**
            * リクエストに `user_tag` が含まれていた場合は必須。
              そうでなければ無し。
              リクエストの `user_tag` の値そのまま。
        * **`user_tags`**
            * リクエストに `users` が含まれていた場合は必須。
              そうでなければ無し。
              リクエストの `users` に含まれていた全てのユーザータグの配列。
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
              リクエストの `related_idps` の内容そのまま。
        * **`exp`**
            * 必須。
              有効期限。
              [JWT Section 4.1.4] を参照のこと。
        * **`jti`**
            * 必須。
              トークン ID。
              [JWT Section 4.1.7] を参照のこと。
        * **`to_ta`**
            * 必須。
              リクエストの `to_ta` の値そのまま。
        * **`related_users`**
            * 必須。
              リクエストの `related_users` の内容そのまま。
        * **`u_hash_alg`**
            * 必須。
              `related_users` におけるユーザー ID のハッシュ値計算アルゴリズム。


#### 5.1.1. 処理の主体が属す IdP からの要請元仲介レスポンス例<a name="main-from-response-example" />

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code_token": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3Jn
        IiwiaXNzIjoiaHR0cHM6Ly9pZHAxLmV4YW1wbGUub3JnIiwic3ViIjoiT2FDLTA2eU93dGpw
        X1VVaWJoZEs2YlBsZHVGQ203cmpNTDVHV2ZuWSIsInVzZXJfdGFnIjoiaW52aXRlciIsInVz
        ZXJfdGFncyI6WyJpbnZpdGVlIl19.zHbWGWJr6PN_1pMgEeJ7layv0-IgY5IQo8QQ-9ZZexs
        87xwPFhKcZZvkphvXTOGZPnqF4EjbwtJYd6s3i5_KYA",
    "referral": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOlsiaHR0cHM6Ly9pZHAyLmV4YW1wbGUub3
        JnIl0sImV4cCI6MTQyNTQ1MjgzNSwiaXNzIjoiaHR0cHM6Ly9pZHAxLmV4YW1wbGUub3JnIi
        wianRpIjoiUXNQUVNUMnMxalNTbUtKZkxaam1udyIsInJlbGF0ZWRfdXNlcnMiOnsib2JzZX
        J2ZXIiOiJtSG1sWFdMa3BZYmdjSEl6Sktia3ZRIn0sInN1YiI6Imh0dHBzOi8vZnJvbS5leG
        FtcGxlLm9yZyIsInRvX3RhIjoiaHR0cHM6Ly90by5leGFtcGxlLm9yZyIsInVoYXNoX2FsZy
        I6IlNIQTI1NiJ9.L2HnJ96s2DpzSzM_lFHd5W2D9KAFa5CwLgl1Me3JiH5B7pMcUuOh1jYtG
        boTxl2KxHL-L_rtlW4rWg3UETAOCQ"
}
```

[JWT] の改行とインデントは表示の都合による。

`code_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp1.example.org",
    "sub": "OaC-06yOwtjp_UUibhdK6bPlduFCm7rjML5GWfnY",
    "aud": "https://to.example.org",
    "user_tag": "inviter",
    "user_tags": [
        "invitee"
    ]
}
```

`referral` のクレームセットの内容は、

```json
{
    "iss": "https://idp1.example.org",
    "sub": "https://from.example.org",
    "aud": [
        "https://idp2.example.org"
    ],
    "exp": 1425452835,
    "jti": "QsPQST2s1jSSmKJfLZjmnw",
    "to_ta": "https://to.example.org",
    "related_users": {
        "observer": "mHmlXWLkpYbgcHIzJKbkvQ"
    },
    "u_hash_alg": "SHA256"
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
        IiwiaXNzIjoiaHR0cHM6Ly9pZHAyLmV4YW1wbGUub3JnIiwic3ViIjoiRV84bkFkRDMyVklk
        dmRNOHl0X0ptdTlmN19HRnd4R0tIcU0tX2MxTSIsInVzZXJfdGFncyI6WyJvYnNlcnZlciJd
        fQ.ICsL5Uws9v6UvzhqZtfSojWJglpXgg-1I1oZ7_EV_7Sg-eap1V1Q3O7z8Iu-wm4JKwq_I
        7HBlePXVJrPXvgVLhOUafY_YFs-9urdqSYu8ybRy0xnhY8XXve1C7BSCS8-Z0wdDP5iTy8pv
        gc59bvoO3JGcCepRCdG-XWxtejKwJei7hvqKnHaE9-LDK7ZsJFvro-qvqQzMm2k42tJrUvWH
        XpmHVGSi6w3GDbkle_P98NXpedsqM-Uxm-HiIzUYbstox6G8RA4L4pAmDBJ7BQzFI1oHWo7V
        clKG_07bbC70xwjq4Rqs2PJsRoEu0O7_C7wuNYyc2k-Mw0S00F1-qOjoQ"
}
```

[JWT] の改行とインデントは表示の都合による。

`code_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp2.example.org",
    "sub": "E_8nAdD32VIdvdM8yt_Jmu9f7_GFwxGKHqM-_c1M",
    "aud": "https://to.example.org",
    "user_tags": [
        "observer"
    ]
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

要請元 TA は以下のように要請元仲介レスポンスを検証しなければならない。

* レスポンスが必要なパラメータを含むことを確認する。
* `code_token` が必要なクレームを含むことを確認する。
* `code_token` に含まれる `iss`, `aud`, `user_tag`, `user_tags` クレームの値がリクエストの内容と合致することを確認する。
* `referral` を含む場合、`referral` が必要なクレームを含むことを確認する。
* `referral` を含む場合、`referral` に含まれる `iss`, `sub`, `aud`, `to_ta`, `related_users` クレームの値がリクエストの内容と合致することを確認する。

また、JWT の署名を検証しても良い。


## 6. 処理要請リクエスト

要請元 TA から要請先 TA の処理エンドポイントに、仲介データを付加してリクエストを送る。


### 6.1. 仲介データ

仲介データは IdP から受け取った全ての `code_token` の値からなる JSON 配列である。


#### 6.1.1. 仲介データ例

```json
[
    "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnIiwiaXNzIjoiaH
    R0cHM6Ly9pZHAxLmV4YW1wbGUub3JnIiwic3ViIjoiT2FDLTA2eU93dGpwX1VVaWJoZEs2YlBsZH
    VGQ203cmpNTDVHV2ZuWSIsInVzZXJfdGFnIjoiaW52aXRlciIsInVzZXJfdGFncyI6WyJpbnZpdG
    VlIl19.zHbWGWJr6PN_1pMgEeJ7layv0-IgY5IQo8QQ-9ZZexs87xwPFhKcZZvkphvXTOGZPnqF4
    EjbwtJYd6s3i5_KYA",
    "eyJhbGciOiJSUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnIiwiaXNzIjoiaH
    R0cHM6Ly9pZHAyLmV4YW1wbGUub3JnIiwic3ViIjoiRV84bkFkRDMyVklkdmRNOHl0X0ptdTlmN1
    9HRnd4R0tIcU0tX2MxTSIsInVzZXJfdGFncyI6WyJvYnNlcnZlciJdfQ.ICsL5Uws9v6UvzhqZtf
    SojWJglpXgg-1I1oZ7_EV_7Sg-eap1V1Q3O7z8Iu-wm4JKwq_I7HBlePXVJrPXvgVLhOUafY_YFs
    -9urdqSYu8ybRy0xnhY8XXve1C7BSCS8-Z0wdDP5iTy8pvgc59bvoO3JGcCepRCdG-XWxtejKwJe
    i7hvqKnHaE9-LDK7ZsJFvro-qvqQzMm2k42tJrUvWHXpmHVGSi6w3GDbkle_P98NXpedsqM-Uxm-
    HiIzUYbstox6G8RA4L4pAmDBJ7BQzFI1oHWo7VclKG_07bbC70xwjq4Rqs2PJsRoEu0O7_C7wuNY
    yc2k-Mw0S00F1-qOjoQ"
]
```

[JWT] の改行とインデントは表示の都合による。


### 6.2. 付加方法

仲介データの付加は URL クエリ、HTTP ヘッダ、リクエストボディの JSON の最上位要素のいずれかで行う。


#### 6.2.1. URL クエリによる付加

`cooperation_codes` パラメータに入れる。
仲介データは適切にエスケープする。


#### 6.2.2. HTTP ヘッダによる付加

X-Edo-Cooperation-Codes ヘッダに入れる。
仲介データは適切にエスケープする。


#### 6.2.3. リクエストボディによる付加

リクエストの Content-Type が application/json の場合のみ、JSON の最上位要素 `cooperation_codes` としてリクエストボディに含めても良い。
ただし、URL クエリか HTTP ヘッダにて以下の方法で宣言を行わなければならない。

* URL クエリでの宣言
    * `cooperation_codes_in_body` パラメータを `true` とする。
* HTTP ヘッダでの宣言
    * X-Edo-Cooperation-Codes-In-Body ヘッダを `true` とする。


### 6.3. 処理要請リクエストの検証

要請先 TA は以下のように処理要請リクエストを検証しなければならない。

* 仲介データが付加されていることを確認する。
* 仲介データの各 [JWT] の署名を検証する。
* 仲介データの各 [JWT] が必要なクレームを含むことを確認する。
* 仲介データの各 [JWT] の `aud` クレームが自分の ID を含むことを確認する。
* `user_tag` クレームを含む [JWT] が仲介データの中にただ 1 つだけ存在することを確認する。
* 異なるユーザーに同じユーザータグが付けられていないことを確認する。

検証に失敗したときのエラーレスポンスの形式は要請先 TA の裁量であるが、`error` の値を `invalid_request` とした [OAuth 2.0 Section 5.2] 形式にすることを推奨する。


## 7. 要請先仲介リクエスト

要請先 TA から IdP の要請先仲介エンドポイントにリクエストを送る。


### 7.1. 要請先仲介エンドポイント

IdP は要請先仲介エンドポイントを TLS で提供しなければならない。


### 7.2. 要請先仲介リクエストパラメータ

要請先仲介リクエストは、以下の最上位要素を含む JSON オブジェクトである。

* **`grant_type`**
    * 必須。
      `code` でなければならない。
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
      ユーザータグから [OpenID Connect Core 1.0 Section 5.5] の `claims` パラメータの `id_token` 要素と同じものへのマップ。

リクエスト時の TA 認証は必須であり、[OpenID Connect Core 1.0 Section 9] で定義されているクライアント認証方式を利用する。
その際、application/x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。

要求した必須クレームに対する事前同意が無ければ拒否される。


#### 7.2.1. 処理の主体が属す IdP への要請先仲介リクエスト例

```HTTP
POST /cooperation/to HTTP/1.1
Host: idp1.example.org
Content-Type: application/json

{
    "grant_type": "code",
    "code": "OaC-06yOwtjp_UUibhdK6bPlduFCm7rjML5GWfnY",
    "claims": {
        "id_token": {
            "pds": { "essential": true }
        }
    },
    "user_claims": {
        "invitee": {
            "pds": { "essential": true }
        }
    }
}
```

TA 認証用データは省いている。


#### 7.2.2. 処理の主体が属さない IdP への要請先仲介リクエスト例

```HTTP
POST /cooperation/to HTTP/1.1
Host: idp2.example.org
Content-Type: application/json

{
    "grant_type": "code",
    "code": "E_8nAdD32VIdvdM8yt_Jmu9f7_GFwxGKHqM-_c1M",
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
* `user_claims` を含むなら、`user_claims` には `code` の指す仲介コードに紐付くユーザータグのみが含まれることを確認する。
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
              仲介コードに紐付く全てのユーザーについて、ユーザータグからユーザーの [ID トークン]に含まれるべき `iss`, `aud`, `exp`, `iat` 以外のクレームセットへのマップ。
              処理の主体のものもそれ以外のものも含む。


#### 8.1.1. 処理の主体が属す IdP からの要請先仲介レスポンス例

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "access_token": "lg4xjJC-tuaOmxaqEVYEBSp257Ru0N4AG1iE-8I4",
    "expires_in": 1037,
    "ids_token": "eyJhbGciOiJFUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnI
        iwiZXhwIjoxNDI1NDUzNzAyLCJpYXQiOjE0MjU0NTI2NjUsImlkcyI6eyJpbnZpdGVlIjp7I
        nBkcyI6eyJ0eXBlIjoic2luZ2xlIiwidXJpIjoiaHR0cHM6Ly9wZHMuZXhhbXBsZS5vcmcvI
        n0sInN1YiI6IjYwMDE5ODQ0ODQ1NTY5MzQ2MzA4In0sImludml0ZXIiOnsicGRzIjp7InR5c
        GUiOiJzaW5nbGUiLCJ1cmkiOiJodHRwczovL3Bkcy5leGFtcGxlLm9yZy8ifSwic3ViIjoiN
        jUwNDI0NTgzMDYyNzU0ODYwOTEifX0sImlzcyI6Imh0dHBzOi8vaWRwMS5leGFtcGxlLm9yZ
        yIsInN1YiI6Imh0dHBzOi8vZnJvbS5leGFtcGxlLm9yZyJ9.dCTzajmzTfPKn6X9mFSCzgrU
        -cWLuIyYbSjUoZdl4WulgG1iQgrV6C52oZ_DKuT4f1C-OgJnbZhagK01fzJ-yg"
}
```

[JWT] の改行とインデントは表示の都合による。

`ids_token` のクレームセットの内容は、

```json
{
    "iss": "https://idp1.example.org",
    "sub": "https://from.example.org",
    "aud": "https://to.example.org",
    "exp": 1425453702,
    "iat": 1425452665,
    "ids": {
        "inviter": {
            "sub": "65042458306275486091",
            "pds": {
                "type": "single",
                "uri": "https://pds.example.org/"
            }
        },
        "invitee": {
            "sub": "60019844845569346308",
            "pds": {
                "type": "single",
                "uri": "https://pds.example.org/"
            }
        }
    }
}
```

署名に使用した鍵は[処理の主体が属す IdP からの要請元仲介レスポンス例](#main-from-response-example)と同じ。


#### 8.1.2 処理の主体が属さない IdP からの要請先仲介レスポンス例

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "ids_token": "eyJhbGciOiJSUzI1NiJ9.eyJhdWQiOiJodHRwczovL3RvLmV4YW1wbGUub3JnI
        iwiZXhwIjoxNDI1NDUzNzA1LCJpYXQiOjE0MjU0NTI2NjgsImlkcyI6eyJvYnNlcnZlciI6e
        yJzdWIiOiI5NzQwNzMyMTI1Nzg5OTE4NDA5OCJ9fSwiaXNzIjoiaHR0cHM6Ly9pZHAyLmV4Y
        W1wbGUub3JnIiwic3ViIjoiaHR0cHM6Ly9mcm9tLmV4YW1wbGUub3JnIn0.olLLTDRKUyheh
        zxOOGD9HaJCJClZB4r_Qe3itA4zeFAQd423W7UX2yKBe1j3pG-qwPuhBol0ed-vpBBJckRV0
        i-hLNchYWKj02bAuxcTF6aEA-cpsQaeYxuwwp3z_nNkemFuehFve04t6NQv1tLi754JV0vRj
        Cnp4VC69TJHPSiQz8ohRXWT323VCDoyHmNOUYTeptWvKfU9lW1LA5shPeSJ1v8VfZjvLE_RG
        sANf_XEbudtYmd4Cg8qTeeIph4gWKo-gyuzleUNhnRW36bvpUPBR5XMquBwQ3UtWqUZ1TgcB
        j79NylLODgjxkHJsgVZiaDPrSuNJlIWbaNLccpIQw"
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
            "sub": "97407321257899184098"
        }
    }
}
```

署名に使用した鍵は[処理の主体が属さない IdP からの要請元仲介レスポンス例](#sub-from-response-example)と同じ。


### 8.2. 要請先仲介レスポンスの検証

要請先 TA は、以下のように要請先仲介レスポンスを検証しなければならない。

* 必要なパラメータを含むことを確認する。
* `ids_token` の値を署名済み [JWT] として検証する。


## 9. セッション

関連するユーザーが処理の主体のみ、かつ、要請先 TA に発行されたアクセストークンが有効である間は、上記プロトコルを省いても良い。
この場合、まず要請先 TA から要請元 TA へのレスポンスの際に、Set-Cookie ヘッダで以下のセッションを宣言する必要がある。

|Cookie 名|値|
|:--|:--|
|X-Edo-Cooperation-Session|セッション ID|

要請元 TA は、後の処理要請リクエストの際に、このセッションを Cookie ヘッダで宣言する。
ユーザータグは最初に用いたものがそのまま用いられる。


## 10. エラーレスポンス

IdP からのエラーは [OAuth 2.0 Section 5.2] の形式で返す。


<!-- 参照 -->
[ID トークン]: http://openid.net/specs/openid-connect-core-1_0.html#IDToken
[JWT Section 4.1.4]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32#section-4.1.4
[JWT Section 4.1.7]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32#section-4.1.7
[JWT]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32
[OAuth 2.0 Section 5.1]: http://tools.ietf.org/html/rfc6749#section-5.1
[OAuth 2.0 Section 5.2]: http://tools.ietf.org/html/rfc6749#section-5.2
[OpenID Connect Core 1.0 Section 5.5]: http://openid.net/specs/openid-connect-core-1_0.html#ClaimsParameter
[OpenID Connect Core 1.0 Section 9]: http://openid.net/specs/openid-connect-core-1_0.html#ClientAuthentication
