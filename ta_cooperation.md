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

## 2. ユーザーの種類

関連するユーザーは、処理の主体とそれ以外に分ける。

* **処理の主体**
    * ユーザー認証を通り、アクセストークンを要請元 TA に渡してアクセスしているユーザー。
* **処理の主体でないユーザー**
    * 操作の対象となるユーザー等。

例えば、要請元 TA と要請先 TA が両方グループウェアであるとする。
要請元 TA で同じグループに入っているユーザーを要請先 TA のグループへ招待する場合、招待する側は処理の主体、招待される側は処理の主体でないユーザーとなる。


## 3. ユーザータグ

要請元は、全ての関連するユーザーに対して、その処理において一意のタグをつけなければならない。


## 4. 要請元仲介リクエスト

要請元 TA から IdP の要請元仲介エンドポイントに、以下で指定するパラメータを最上位要素として含む JSON オブジェクトを送る。

* **`response_type`**
    * 必須。
      `code_token` でなければならない。
* **`to_ta`**
    * 必須。
      要請先 TA の ID。
* **`grant_type`**
    * 処理の主体が IdP に属す場合は必須。
      `access_token` でなければならない。
* **`access_token`**
    * `grant_type` が `access_token` の場合は必須。
      そうでなければ無し。
      処理の主体に対する有効なアクセストークン。
* **`scope`**
    * `access_token` を含める場合は任意。
      そうでなければ無し。
      要請先 TA に新しく発行されるアクセストークンのスコープの最大範囲。
      `access_token` で指定したアクセストークン発行時のスコープより広い場合は拒否される。
* **`expires_in`**
    * `access_token` を含める場合は任意。
      そうでなければ無し。
      要請先 TA に新しく発行されるアクセストークンの有効期間の上限。
      `access_token` で指定したアクセストークンの残り有効期間より長い場合は無意味。
* **`user_tag`**
    * 処理の主体が IdP に属す場合は必須。
      そうでなければ無し。
      処理の主体に付けたユーザータグ。
* **`users`**
    * 処理の主体でないユーザーが IdP に属す場合は必須。
      そうでなければ無し。
      IdP に属す全ての処理の主体でないユーザーについて、ユーザータグからユーザー ID へのマップ。


リクエスト時の TA 認証は必須であり、OpenID Connect 1.0 で定義されているクライアント認証方式を利用する。
その際、x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。


### 4.1. 要請元仲介リクエスト例

```HTTP
POST /cooperation/from HTTP/1.1
Host: idp.example.org
Content-Type: application/json

{
    "response_type": "code_token",
    "to_ta": "https://to.example.org",
    "grant_type": "access_token",
    "access_token": "HkXuWaG_bYv7OS0qLA1Q28_2rSA3ENkIsDYmRr_Dad84mfRKCO",
    "user_tag": "inviter",
    "users": {
        "invitee": "86504857780817848362"
    }
}
```


## 5. 要請元仲介レスポンス

リクエストに問題が無ければ、IdP から要請元 TA に、以下のクレームを含む JWT を返す。

* **`iss`**
    * 必須。IdP の ID。
* **`sub`**
    * 必須。仲介コード。
* **`user_tag`**
    * リクエストに `user_tag` が含まれていた場合は必須。
      そうでなければ無し。
      リクエストの `user_tag` の値そのまま。
* **`user_tags`**
    * リクエストに `users` が含まれていた場合は必須。
      そうでなければ無し。
      ユーザータグの配列。


### 5.1. 要請元仲介レスポンス例

クレームセットの内容は、

```json
{
    "iss": "https://idp.example.org",
    "sub": "OaC-06yOwtjp_UUibhdK6bPlduFCm7rjML5GWfnY",
    "user_tag": "inviter",
    "user_tags": [ "invitee" ]
}
```

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/jwt

eyJhbGciOiJFUzI1NiJ9.eyJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLm9
yZyIsInN1YiI6Ik9hQy0wNnlPd3RqcF9VVWliaGRLNmJQbGR1RkNtN3JqTUw
1R1dmblkiLCJ1c2VyX3RhZyI6Imludml0ZXIiLCJ1c2VyX3RhZ3MiOlsiaW5
2aXRlZSJdfQ.zavrHisbP0RZ3mwnmAxlv2RhOjG7AuEvBbwGqlb3d1VepK93
CJBjhU8cAlPVJWxkrx14C8EyH3PJQG0v8rIW6Q
```

ボディの改行は表示の都合による。
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
    "eyJhbGciOiJFUzI1NiJ9.eyJpc3MiOiJodHRwczovL2lkcC5le
     GFtcGxlLm9yZyIsInN1YiI6Ik9hQy0wNnlPd3RqcF9VVWliaGR
     LNmJQbGR1RkNtN3JqTUw1R1dmblkiLCJ0YWciOiJpbnZpdGVyI
     iwidXNlcl90YWdzIjpbImludml0ZWUiXX0.QlvkaGzb2g9XVW3
     CvL0W7BDSyiEInRkCw7aqxv7NVrtCFy_0jhmQcyCH_10p7h6JM
     P5J876k3NV4SYDley-gpQ"
]
```

改行とインデントは表示の都合による。
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
    * 仲介コードに `user_tag` が付いている場合は任意。
      そうでなければ無し。
      処理の主体に対する OpenID Connect 1.0 の `claims` パラメータと同じ。
* **`user_claims`**
    * 仲介コードに `user_tags` が付いている場合は任意。
      そうでなければ無し。
      ユーザータグから OpenID Connect 1.0 の `claims` パラメータの `id_token` オブジェクトと同じものへのマップ。

リクエスト時の TA 認証は必須であり、OpenID Connect 1.0 で定義されているクライアント認証方式を利用する。
その際、x-www-form-urlencoded フォームパラメータとして含めるはずのものは、代わりに JSON の最上位要素として含める。

要求したクレームに対する事前同意が無ければ拒否される。


### 7.1. 要請先仲介リクエスト例

```HTTP
POST /cooperation/from HTTP/1.1
Host: idp.example.org
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


## 8. 要請先仲介レスポンス

リクエストに問題が無ければ、IdP から要請先 TA に、以下のクレームを含む JWT を返す。

* **`iss`**
    * 必須。
      IdP の ID。
* **`sub`**
    * 必須。
      要請元 TA の ID。
* **`aud`**
    * ID トークンの `aud` と同じもの。
* **`exp`**
    * ID トークンの `exp` と同じもの。
* **`iat`**
    * ID トークンの `iat` と同じもの。
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
* **`user_ids`**
    * 必須。
      ユーザータグからユーザーの ID トークンに含まれるべき `iss`, `aud`, `exp`, `iat` 以外のクレームセットへのマップ。
      処理の主体もそれ以外も一緒。


### 8.1. 要請先仲介レスポンス例

クレームセットの内容は、

```json
{
    "iss": "https://idp.example.org",
    "sub": "https://from.example.org",
    "aud": "https://to.example.org",
    "exp": 1425453702,
    "iat": 1425452665,
    "access_token": "lg4xjJC-tuaOmxaqEVYEBSp257Ru0N4AG1iE-8I4",
    "expires_in": 1037,
    "user_ids": {
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

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/jwt

eyJhbGciOiJFUzI1NiJ9.eyJhY2Nlc3NfdG9rZW4iOiJsZzR4akpDLXR1YU9
teGFxRVZZRUJTcDI1N1J1ME40QUcxaUUtOEk0IiwiYXVkIjoiaHR0cHM6Ly9
0by5leGFtcGxlLm9yZyIsImV4cCI6MTQyNTQ1MzcwMiwiZXhwaXJlc19pbiI
6MTAzNywiaWF0IjoxNDI1NDUyNjY1LCJpc3MiOiJodHRwczovL2lkcC5leGF
tcGxlLm9yZyIsInN1YiI6Imh0dHBzOi8vZnJvbS5leGFtcGxlLm9yZyIsInV
zZXJfaWRzIjp7Imludml0ZWUiOnsicGRzIjp7InR5cGUiOiJzaW5nbGUiLCJ
1cmkiOiJodHRwczovL3Bkcy5leGFtcGxlLm9yZy8ifSwic3ViIjoiNjAwMTk
4NDQ4NDU1NjkzNDYzMDgifSwiaW52aXRlciI6eyJwZHMiOnsidHlwZSI6InN
pbmdsZSIsInVyaSI6Imh0dHBzOi8vcGRzLmV4YW1wbGUub3JnLyJ9LCJzdWI
iOiI2NTA0MjQ1ODMwNjI3NTQ4NjA5MSJ9fX0.kMATHQZZN3xadA5xAFPox2n
CYKv2q370o4Z__MTWWPHCEicoB9QyG54CWqVyudBcN8gJ2ORSkeZ6c7TAlCt
M4w
```

ボディの改行は表示の都合による。
署名に使用した鍵は要請元仲介レスポンス例と同じ。
