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


# PDS 権限変更プロトコル

要請元 TA が PDS のデータアクセス権限の変更を要請するためのプロトコル。


## 1. 概要

1. 要請元 TA から PDS に、変更内容を通知する。
2. PDS から要請元 TA に、変更コードを発行する。
3. 要請元 TA から PDS に、変更コードと共にユーザーをリダイレクトさせる。
4. PDS とユーザー間で、合意を形成する。
5. PDS から要請元 TA に、結果と共にユーザーをリダイレクトさせる。

```
+--------+                                   +--------+
|        |                                   |        |
|        |------------(1) request----------->|        |
|        |                                   |        |
|        |<-----------(2) code---------------|        |
|        |                                   |        |
|   TA   |    +--------+                     |  PDS   |
|        |--->|        |----(3) code-------->|        |
|        |    |        |                     |        |
|        |    |  User  |<---(4) agreement--->|        |
|        |    |        |                     |        |
|        |<---|        |<---(5) result-------|        |
|        |    +--------+                     |        |
+--------+                                   +--------+
```


## 2. TA 間連携とユーザー認証

本プロトコルでは、途中、[TA 間連携プロトコル]と[ユーザー認証プロトコル]を利用する。

要請元 TA から PDS への変更内容の通知は TA 間連携で行われ、PDS とユーザー間で合意を形成する前にユーザー認証が行われる。


## 3. 変更対象タグ

要請元 TA は、全ての変更対象に対して、その処理において一意のタグをつけなければならない。


## 4. 変更要請リクエスト

要請元 TA から PDS の変更要請エンドポイントに、[TA 間連携プロトコル]の付加情報と共に、以下の最上位要素を含む JSON オブジェクトを送る。
ユーザータグは [TA 間連携プロトコル]で付けたものと同じでなければならない。

* **`chmod`**
    * 必須。
      変更対象タグから変更対象へのマップ。
      変更対象は以下の要素を含むオブジェクト。
        * `user_tag`
            * 必須。
              対象データの保持者のユーザータグ。
        * `ta`
            * 必須。
              対象データの領域を割り当てられた TA の ID。
        * `path`
            * 必須。
              データまたはディレクトリのパス。
              末尾が / ならディレクトリを指定しているとみなす。
        * `sub_tags`
            * 任意。
              アクセス権限を変更されるユーザーのユーザータグの配列。
              省略した場合は、処理の主体のみが指定されているとみなす。
              また、特殊な値として、`*` で全てのユーザーを指定できる。
        * `mod`
            * 必須。
              変更する権限。
              `[+-=][rw]+`。
              `+r` 等。
        * `recursive`
            * 対象がディレクトリなら任意。
              そうでなければ無し。
              ディレクトリに対して再帰的に適用するかどうか。
        * `essential`
            * 任意。
              true なら、変更が拒否された時に他の変更も破棄する。
              そうでなければ、他の変更は適用する。
* **`redirect_uri`**
    * 必須。
      PDS から要請元 TA にユーザーをリダイレクトさせる際の要請元 TA 配下のエンドポイント。
* **`state`**
    * 任意。
      PDS から要請元 TA にユーザーをリダイレクトさせる際にそのまま付加されるパラメータ。
* **`display`**
    * [OpenID Connect Core 1.0 Section 3.1.2.1] の `display` と同じ。
* **`ui_locales`**
    * [OpenID Connect Core 1.0 Section 3.1.2.1] の `ui_locales` と同じ。


### 4.1. 変更要請リクエストの例

```HTTP
POST /access-control/ta HTTP/1.1
Host: pds.example.org
Content-Type: application/json

{
    "chmod": {
        "profile": {
            "user_tag": "user",
            "ta": "https://writer.example.org",
            "path": "/profile",
            "mod": "+r",
            "recursive": true,
            "essential": true
        },
        "diary": {
            "user_tag": "user",
            "ta": "https://writer.example.org",
            "path": "/diary",
            "mod": "+r",
            "recursive": true
        }
    },
    "redirect_uri": "https://reader.example.org/return/chmod",
    "state": "SiuR29g1Iu"
}
```

[TA 間連携プロトコル]の付加情報は省いている。


## 5. 変更要請レスポンス

リクエストに問題が無ければ、PDS から要請元 TA に、以下の最上位要素を含む JSON オブジェクトを返す。

* **`code`**
     * 必須。
       変更コード。


### 5.1. 変更要請レスポンス例

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code": "6YYIWD6mH9B6ER5eDw3AkAnnCi-aPNutiqI6IwHN"
}
```


## 6. PDS へのリダイレクト

要請元 TA は以下のパラメータと共にユーザーを PDS の変更エンドポイントへリダイレクトさせる。

* **`code`**
    * 必須。
      変更コード。


### 6.1. PDS にリダイレクトさせるためのレスポンス例

```HTTP
HTTP/1.1 302 Found
Location: https://pds.example.org/access-control/user?
    code=6YYIWD6mH9B6ER5eDw3AkAnnCi-aPNutiqI6IwHN
```

ヘッダ値の改行とインデントは表示の都合による。


## 7. ユーザーと PDS 間での合意


### 7.1. 権限変更権限

本仕様では、データアクセス権限を変更する権限の保持者について規定しない。

データの保持者が変更権限の保持者になるのが自然だが、それ以外のユーザーにも管理者として変更権限を持たせるかもしれない。
現実の親子関係を持つ保護者ユーザーが子供ユーザーのアクセス権限を全面的に引き受ける等の形態も考えられる。


### 7.2. 権限変更要求受信箱

変更処理の主体となるユーザーが変更権限を持たない場合に備え、PDS は権限変更要求の受信箱をユーザーごとに用意しなければならない。
ただし、本仕様では、権限変更要求受信箱の詳細は規定しない。


### 7.3. 合意の種類

全ての変更対象に対して、ユーザーから以下のいずれかの合意を得なければならない。

* **適用**
    * 変更を行い、要請元 TA が求める状態にする。
      元から適用された状態だった場合、非明示的に適用で合意したとみなしても良い。
      そうでなく、かつ、ユーザーに変更権限が無い場合、適用で合意してはならない。
* **転送**
    * 変更権限保持者の受信箱に、ユーザーと要請元 TA からの要求として変更対象を追加する。
      既に同じ内容の要求が受信箱に入っている場合、非明示的に転送で合意したとみなしても良い。
      ユーザーに変更権限がある場合、転送で合意してはならない。
* **拒否**
    * 適用も転送もしない。


### 7.4. 合意形成の方法

本仕様では、ユーザーと PDS 間での合意形成の具体的な方法は規定しない。

変更対象を明示し、合意ボタンを押させるような UI を通す形が一般的と思われる。


### 7.5. 合意の履行

適用で合意した場合、速やかに権限を変更しなければならない。

転送で合意した場合、実際に変更権限保持者の受信箱に要求を追加するかどうかは PDS の裁量とする。
適切なスパム防止策を施すことを推奨する。

適用で合意した場合、適用や転送を行ってはならない。


## 8. 要請元 TA へのリダイレクト

ユーザーから必要な同意が得られたら、PDS は以下のパラメータを付加してユーザーを変更要請リクエストの `redirect_uri` へリダイレクトさせる。

* **`applied`**
    * 適用で合意した変更対象があった場合は必須。
      そうでなければ任意。
      全ての適用された変更対象の変更対象タグの JSON 配列。
* **`forwarded`**
    * 転送で合意した変更対象があった場合は必須。
      そうでなければ任意。
      全ての転送された変更対象の変更対象タグの JSON 配列。
* **`denied`**
    * 拒否で合意した変更対象があった場合は必須。
      そうでなければ任意。
      全ての拒否された変更対象の変更対象タグの JSON 配列。


### 8.1. 要請元 TA にリダイレクトさせるためのレスポンス例

```HTTP
HTTP/1.1 302 Found
Location: https://reader.example.org/return/chmod?
    applied=%5B%22profile%22%5D
    &denied=%5B%22diary%22%5D
    &state=SiuR29g1Iu
```

ヘッダ値の改行とインデントは表示の都合による。


## 9. エラーレスポンス

変更要請リクエストに対するエラーは [OAuth 2.0 Section 5.2] の形式で返す。
合意中のエラーは [OAuth 2.0 Section 4.1.2.1] の形式で、主に要請元 TA にリダイレクトして返す。


<!-- 参照 -->
[OAuth 2.0 Section 4.1.2.1]: http://tools.ietf.org/html/rfc6749#section-4.1.2.1
[OAuth 2.0 Section 5.2]: http://tools.ietf.org/html/rfc6749#section-5.2
[OpenID Connect Core 1.0 Section 3.1.2.1]: http://openid.net/specs/openid-connect-core-1_0.html#AuthRequest
[TA 間連携プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/ta_cooperation.md
[ユーザー認証プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/user_authentication.md
