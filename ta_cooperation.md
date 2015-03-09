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
    2. IdP から要請先 TA に、要請元 TA と仲介コードに紐付くユーザーの情報を渡す。

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
    * ユーザー認証を通り、アクセストークンを要請元 TA に渡してアクセスしているユーザー。
* **処理の主体でないユーザー**
    * 操作の対象となるユーザー等。

例えば、要請元 TA と要請先 TA が両方グループウェアであるとする。
要請元 TA で同じグループに入っているユーザーを要請先 TA のグループへ招待する場合、招待する側のユーザーは処理の主体、招待される側のユーザーは処理の主体でないユーザーとなる。


## 3. ユーザータグ

要請元 TA は、関連するユーザー全てに対して、その処理において一意のタグをつけなければならない。


## 4. 要請元仲介リクエスト

要請元 TA から IdP への仲介リクエストは、処理の主体が IdP に属すかどうかで中身が異なる。
また、処理の主体が属さない IdP へのリクエストには、処理の主体が属す IdP へのリクエスト結果が必要になるため、最初に処理の主体が属す IdP へのリクエストを行う必要がある。


### 4.1. 処理の主体が属す IdP への要請元仲介リクエスト

要請元 TA から IdP の要請元仲介エンドポイントに、以下で指定するパラメータを最上位要素として含む JSON オブジェクトを送る。

* **`response_type`**
    * 必須。
      他の IdP に属す関連するユーザーがいない場合は `code_token`、いる場合は `code_token referral` でなければならない。
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
      `access_token` で指定したアクセストークン発行時のスコープより広い場合は拒否される。
* **`expires_in`**
    * 任意。
      要請先 TA に新しく発行されるアクセストークンの有効期間の上限。
      `access_token` で指定したアクセストークンの残り有効期間より長い場合は無意味。
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
* **`uhash_alg`**
    * `related_users` を含める場合は任意。
      そうでなければ無し。
      `related_users` におけるユーザー ID のハッシュ値計算アルゴリズム。
* **`related_idps`**
    * `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      関連するユーザーが属す他の全ての IdP の ID の配列。


リクエスト時の TA 認証は必須であり、OpenID Connect 1.0 で定義されているクライアント認証方式を利用する。
その際、x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


### 4.1.1. 処理の主体が属す IdP への要請元仲介リクエスト例

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
        "observer": "mHmlXWLkpYbgcHIzJKbkvd8tENXcI66L0qo+4nb4YEE"
    },
    "related_idps": [
        "https://idp2.example.org"
    ]
}
```


### 4.2. 処理の主体が属さない IdP への要請元仲介リクエスト

要請元 TA から IdP の要請元仲介エンドポイントに、以下で指定するパラメータを最上位要素として含む JSON オブジェクトを送る。

* **`response_type`**
    * 必須。
      `code_token` でなければならない。
* **`grant_type`**
    * 必須。
      `referral` でなければならない。
* **`referral`**
    * 必須。
      処理の主体の IdP が発行した `referral`。
* **`users`**
    * 必須。
      IdP に属す関連するユーザー全てについて、ユーザータグからユーザー ID へのマップ。

リクエスト時の TA 認証は必須であり、OpenID Connect 1.0 で定義されているクライアント認証方式を利用する。
その際、x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


### 4.2.1. 処理の主体が属さない IdP への要請元仲介リクエスト例

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
        J2ZXIiOiJtSG1sWFdMa3BZYmdjSEl6Sktia3ZkOHRFTlhjSTY2TDBxbys0bmI0WUVFIn0sIn
        N1YiI6Imh0dHBzOi8vZnJvbS5leGFtcGxlLm9yZyIsInRvX3RhIjoiaHR0cHM6Ly90by5leG
        FtcGxlLm9yZyIsInVoYXNoX2FsZyI6IlNIQTI1NiJ9.IdjEqxS-_qKa4oWeCE9vl3RI3Sn88
        5qQqZaC9V2V0IJcgHPnMGx3KY9JjnC_y6Ud7fD7kbg5A7dzj-u-gB7wVA",
    "users": {
        "observer": "40053950180034613776"
    },
}
```

`referral` は処理の主体が属す IdP からの要請元仲介レスポンス例のもの。


## 5. 要請元仲介レスポンス

リクエストに問題が無ければ、IdP から要請元 TA に、以下の最上位要素を含む JSON オブジェクトを返す。

* **`code_token`**
    * リクエストの `response_type` が `code_token` を含む場合は必須。
      そうでなければ無し。
      以下のクレームを含む署名済み JWT。
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
              リクエストの `user_tags` に含まれていた全てのユーザータグの配列。
* **`referral`**
    * リクエストの `response_type` が `referral` を含む場合は必須。
      そうでなければ無し。
      以下のクレームを含む署名済み JWT。
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
              JWT の `exp`。
        * **`jti`**
            * 必須。
              JWT の `jti`。
        * **`to_ta`**
            * 必須。
              リクエストの `to_ta` の値そのまま。
        * **`related_users`**
            * 必須。
              リクエストの `related_users` の内容そのまま。
        * **`uhash_alg`**
            * 必須。
              `related_users` におけるユーザー ID のハッシュ値計算アルゴリズム。


### 5.1. 処理の主体が属す IdP からの要請元仲介レスポンス例

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
        J2ZXIiOiJtSG1sWFdMa3BZYmdjSEl6Sktia3ZkOHRFTlhjSTY2TDBxbys0bmI0WUVFIn0sIn
        N1YiI6Imh0dHBzOi8vZnJvbS5leGFtcGxlLm9yZyIsInRvX3RhIjoiaHR0cHM6Ly90by5leG
        FtcGxlLm9yZyIsInVoYXNoX2FsZyI6IlNIQTI1NiJ9.IdjEqxS-_qKa4oWeCE9vl3RI3Sn88
        5qQqZaC9V2V0IJcgHPnMGx3KY9JjnC_y6Ud7fD7kbg5A7dzj-u-gB7wVA"
}
```

JWT の改行とインデントは表示の都合による。

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
        "observer": "mHmlXWLkpYbgcHIzJKbkvd8tENXcI66L0qo+4nb4YEE"
    },
    "uhash_alg": "SHA256"
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

### 5.2. 処理の主体が属さない IdP からの要請元仲介レスポンス例

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

JWT の改行とインデントは表示の都合による。

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


## 6. 処理要請リクエスト

要請元 TA から要請先 TA の処理エンドポイントに、IdP から受け取った全ての JWT を JSON 配列の形で付加してリクエストを送る。

付加は URL クエリか HTTP ヘッダにて行う。


### 6.1. URL クエリによる付加

**`cooperation_codes`** パラメータに入れる。


### 6.2. ヘッダによる付加

**`X-Edo-Cooperation-Codes`** ヘッダに入れる。


### 6.3. 付加データの例

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

JWT の改行とインデントは表示の都合による。
付加の際は適切にエスケープする。


## 7. 要請先仲介リクエスト

要請先 TA から IdP の要請先仲介エンドポイントに、以下の最上位要素を含む JSON オブジェクトを送る。

* **`grant_type`**
    * 必須。
      `code` でなければならない。
* **`code`**
    * 必須。
      仲介コード。
* **`claims`**
    * 仲介コードに `user_tag` が付いていた場合は任意。
      そうでなければ無し。
      処理の主体に対する OpenID Connect 1.0 の `claims` パラメータと同じ。
* **`user_claims`**
    * 仲介コードに `user_tags` が付いていた場合は任意。
      そうでなければ無し。
      ユーザータグから OpenID Connect 1.0 の `claims` パラメータの `id_token` オブジェクトと同じものへのマップ。

リクエスト時の TA 認証は必須であり、OpenID Connect 1.0 で定義されているクライアント認証方式を利用する。
その際、x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。

要求したクレームに対する事前同意が無ければ拒否される。


### 7.1. 処理の主体が属す IdP への要請先仲介リクエスト例

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


### 7.2. 処理の主体が属さない IdP への要請先仲介リクエスト例

```HTTP
POST /cooperation/to HTTP/1.1
Host: idp2.example.org
Content-Type: application/json

{
    "grant_type": "code",
    "code": "E_8nAdD32VIdvdM8yt_Jmu9f7_GFwxGKHqM-_c1M",
}
```


## 8. 要請先仲介レスポンス

リクエストに問題が無ければ、IdP から要請先 TA に、以下の最上位要素を含む JSON オブジェクトを返す。

* **`access_token`**
    * 仲介コードが処理の主体に紐付いていた場合は必須。
      そうでなければ無し。
      新しく発行した処理の主体に対するアクセストークン。
* **`token_type`**
    * `access_token` が含まれる場合のみ。
      OAuth 2.0 の `token_type` と同じもの。
* **`expires_in`**
    * `access_token` が含まれる場合のみ。
      OAuth 2.0 の `expires_in` と同じもの。
* **`scope`**
    * `access_token` が含まれる場合のみ。
      OAuth 2.0 の `scope` と同じもの。
* **`ids_token`**
    * 以下のクレームを含む署名済み JWT。
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
            * ID トークンの `exp` と同じもの。
        * **`iat`**
            * ID トークンの `iat` と同じもの。
        * **`ids`**
            * 必須。
              ユーザータグからユーザーの ID トークンに含まれるべき `iss`, `aud`, `exp`, `iat` 以外のクレームセットへのマップ。
              処理の主体もそれ以外も一緒。


### 8.1. 処理の主体が属す IdP からの要請先仲介レスポンス例

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

JWT の改行とインデントは表示の都合による。

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

署名に使用した鍵は処理の主体が属す IdP からの要請元仲介レスポンス例と同じ。


### 8.1. 処理の主体が属さない IdP からの要請先仲介レスポンス例

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

JWT の改行とインデントは表示の都合による。

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

署名に使用した鍵は処理の主体が属さない IdP からの要請元仲介レスポンス例と同じ。
