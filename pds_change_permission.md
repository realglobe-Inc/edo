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

要請元 TA が PDS にあるデータのアクセス権限を変更するためのプロトコル。
[アクセス制御]も参照のこと。


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

要請元 TA から PDS の変更要請エンドポイントに、[TA 間連携プロトコル]の付加情報と共にリクエストを送る。


### 4.1. 変更要請エンドポイント

PDS は変更要請エンドポイントを TLS で提供しなければならない。


### 4.2. 変更要請リクエストパラメータ

変更要請リクエストは、以下の最上位要素を含む JSON オブジェクトである。
アカウントタグは [TA 間連携プロトコル]で付けたものと同じでなければならない。

* **`chmod`**
    * 必須。
      変更対象タグから変更対象へのマップ。
      変更対象は以下の要素を含むオブジェクト。
        * `owner_tag`
            * 必須。
              対象データを所有するアカウントのアカウントタグ。
        * `ta`
            * 必須。
              対象データの領域を割り当てられた TA の ID。
        * `path`
            * 必須。
              対象データのパス。
              ディレクトリを指す場合はサブツリー全体の権限を変更することを意味する。
        * `accessor`
            * 任意。
              アクセス元。
              アクセス主体のアカウントタグからアクセス元 TA の ID の配列へのマップ。
              無指定の場合、処理の主体と要請元 TA のみが指定されているとみなす。
              また、特殊な値として、`*` で全てのアカウント、全ての TA を指定できる。
        * `mod`
            * 必須。
              変更する権限。
              `[+-=]`&lt;アクセス権限&gt;。
              `+r` 等。
        * `essential`
            * 任意。
              変更が拒否された場合に他の変更を破棄するかどうか。
              無指定の場合、偽（他の変更は適用する）。
        * `check_exist`
            * 任意。
              対象のデータが存在しない場合に失敗するかどうか。
              無指定の場合、偽（失敗しない）。
* **`redirect_uri`**
    * 必須。
      PDS から要請元 TA にユーザーをリダイレクトさせる際の要請元 TA 配下のエンドポイント。
* **`state`**
    * 任意。
      PDS から要請元 TA にユーザーをリダイレクトさせる際にそのまま付加されるパラメータ。
* **`display`**
    * [OpenID Connect Core 1.0 Section 3.1.2.1] の `display` と同じもの。
* **`ui_locales`**
    * [OpenID Connect Core 1.0 Section 3.1.2.1] の `ui_locales` と同じもの。

以下は `accessor` の例。

```json
{
    "self": [
        "*"
    ],
    "observer": [
        "https://reader.example.org"
    ],
    "*": [
        "https://recruit.example.org"
    ]
}
```


#### 4.2.1. 変更要請リクエスト例

```HTTP
POST /access-control/ta HTTP/1.1
Host: pds.example.org
Content-Type: application/json

{
    "chmod": {
        "profile": {
            "owner_tag": "self",
            "ta": "https://writer.example.org",
            "path": "/profile",
            "mod": "+r",
            "essential": true
        },
        "diary": {
            "owner_tag": "self",
            "ta": "https://writer.example.org",
            "path": "/diary",
            "mod": "+r"
        }
    },
    "redirect_uri": "https://reader.example.org/return/chmod",
    "state": "SiuR29g1Iu"
}
```

[TA 間連携プロトコル]の付加情報は省いている。


### 4.3. 変更要請リクエストの検証

PDS は変更要請リクエストを以下のように検証しなければならない。
--f--> は失敗時のエラーレスポンスの `error` の値を示す。

* [TA 間連携プロトコル]の処理要請リクエストに則っていることを検証する。
    * --f--> `invalid_request`
* `redirect_uri` が要請元 TA の配下であることを確認する。
    * --f--> `invalid_request`
* `owner_tag` および `accesser` に現れるアカウントタグが [TA 間連携プロトコル]で定義されていることを確認する。
    * --f--> `invalid_request`
* `accesser` に現れる TA が存在することを確認する。
    * --f--> `invalid_request`
* `check_exist` が真の場合、対象データが存在することを確認する。
    * --f--> `not_exist`

また、全ての変更対象が非明示的に適用または転送で合意できる場合、`error` を `already_done` にしてエラーレスポンスを返しても良い。
その場合は、以下の最上位要素も返さなければならない。

* `applied`
    * 非明示的に適用で合意した変更対象があった場合は必須。
      そうでなければ任意。
      適用された変更対象全ての変更対象タグからなる配列。
* `forwarded`
    * 非明示的に転送で合意した変更対象があった場合は必須。
      そうでなければ任意。
      転送された変更対象全ての変更対象タグからなる配列。


## 5. 変更要請レスポンス

リクエストに問題が無ければ、PDS から要請元 TA にレスポンスを返す。


### 5.1. 変更要請レスポンスパラメータ

変更要請レスポンスは、以下の最上位要素を含む JSON オブジェクトである。

* **`code`**
     * 必須。
       変更コード。


#### 5.1.1. 変更要請レスポンス例

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "code": "SS2EVIbmR7X1IHiHzMInTINxGYi-OY"
}
```


## 6. PDS へのリダイレクト

要請元 TA から PDS の合意エンドポイントへ、パラメータを付加してユーザーをリダイレクトさせる。


### 6.1. 合意エンドポイント

PDS は合意エンドポイントを TLS で提供しなければならない。


### 6.2. リダイレクトパラメータ

リダイレクトに際し、以下のパラメータを付加する。

* **`code`**
    * 必須。
      変更コード。


#### 6.2.1. PDS にリダイレクトさせるためのレスポンス例

```HTTP
HTTP/1.1 302 Found
Location: https://pds.example.org/access-control/user?
    code=SS2EVIbmR7X1IHiHzMInTINxGYi-OY
```

ヘッダ値の改行とインデントは表示の都合による。


### 6.3. リダイレクトパラメータの検証

PDS は以下のようにリダイレクトパラメータを検証しなければならない。
--f--> は失敗時のエラーレスポンスの `error` の値を示す。

* `code` を含むことを確認する。
    * 失敗した場合、ユーザーにエラーを表示する。
* `code` の指す変更コードが有効であることを確認する。
    * --f--> `invalid_grant`


## 7. ユーザーと PDS 間の合意

変更対象に対する合意をユーザーから取得する。


### 7.1. 権限変更権限

本仕様では、データアクセス権限を変更する権限の保持者について規定しない。

データの所有者が変更権限の保持者になるのが自然だが、それ以外のユーザーにも管理者として変更権限を与えるかもしれない。
現実の親子関係を持つ保護者ユーザーが子供ユーザーの変更権限を全面的に引き受ける形態も考えられる。


### 7.2. 権限変更要求の保存

変更処理の主体が変更権限を持たない場合に備え、PDS は権限変更要求を保存し、後で変更権限の保持者から合意を得る仕組みを持たなければならない。
ただし、本仕様では、その詳細は規定しない。


### 7.3. 合意の種類

全ての変更対象に対して、ユーザーから以下のいずれかの合意を得なければならない。

* **適用**
    * 要求された変更を行う。
      元から変更された状態だった場合、非明示的に適用で合意したとみなしても良い。
      そうでなく、かつ、変更処理の主体に変更権限が無い場合、適用で合意してはならない。
* **転送**
    * 変更処理の主体と要請元 TA からの要求として変更対象を保存する。
      既に同じ内容が保存されている場合、非明示的に転送で合意したとみなしても良い。
      変更処理の主体に変更権限がある場合、転送で合意してはならない。
* **拒否**
    * 適用も転送もしない。


### 7.4. 合意形成の方法

PDS はユーザーに対して、非明示的には合意できない変更対象全てと可能な合意の種類を明示しなければならない。
本仕様では、これ以上の具体的な合意の方法は規定しない。


### 7.5. 合意の履行

適用で合意した場合、速やかに権限を変更しなければならない。
変更対象の範囲が重複する場合、範囲が広いものから順に変更した状態にしなければならない。
例えば、/profile サブツリーの変更を行った後に /profile/career の変更を行う。
また、変更要請リクエストで指定されなかったアクセス元のアクセス権限に変更を加えてはならない。
`*` は `*` であり、他のアクセス元のアクセス権限を変えてはならない。

転送で合意した場合、実際に権限変更要求を保存するかどうかは PDS の裁量とする。
適切なスパム防止策を施すことを推奨する。

拒否で合意した場合、適用や転送を行ってはならない。


## 8. 要請元 TA へのリダイレクト

ユーザーから必要な同意が得られたら、PDS から要請元 TA へ、パラメータを付加してユーザーを変更要請リクエストの `redirect_uri` へリダイレクトさせる。


### 8.1. リダイレクトパラメータ

リダイレクトに際し、以下のパラメータを付加する。

* **`applied`**
    * 適用で合意した変更対象があった場合は必須。
      そうでなければ任意。
      適用された変更対象全ての変更対象タグからなる JSON 配列。
* **`forwarded`**
    * 転送で合意した変更対象があった場合は必須。
      そうでなければ任意。
      転送された変更対象全ての変更対象タグからなる JSON 配列。
* **`denied`**
    * 拒否で合意した変更対象があった場合は必須。
      そうでなければ任意。
      拒否された変更対象全ての変更対象タグからなる JSON 配列。
* **`state`**
    * 変更要請リクエストに `state` が含まれていた場合は必須。
      そうでなければ無し。
      変更要請リクエストの `state` の値そのまま。


#### 8.1.1. 要請元 TA にリダイレクトさせるためのレスポンス例

```HTTP
HTTP/1.1 302 Found
Location: https://reader.example.org/return/chmod?applied=%5B%22profile%22%5D
    &denied=%5B%22diary%22%5D&state=SiuR29g1Iu
```

ヘッダ値の改行とインデントは表示の都合による。


## 9. エラーレスポンス

変更要請リクエストに対するエラーは [OAuth 2.0 Section 5.2] の形式で返す。
合意中のエラーは [OAuth 2.0 Section 4.1.2.1] の形式で、主に要請元 TA にリダイレクトして返す。

`error` の値として以下を追加する。

* **`already_done`**
    * 既に変更が適用または転送されている。
* **`not_exist`**
    * 対象のデータが存在しない。


<!-- 参照 -->
[OAuth 2.0 Section 4.1.2.1]: http://tools.ietf.org/html/rfc6749#section-4.1.2.1
[OAuth 2.0 Section 5.2]: http://tools.ietf.org/html/rfc6749#section-5.2
[OpenID Connect Core 1.0 Section 3.1.2.1]: http://openid.net/specs/openid-connect-core-1_0.html#AuthRequest
[TA 間連携プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/ta_cooperation.md
[アクセス制御]: https://github.com/realglobe-Inc/edo/blob/master/access_control.md
[ユーザー認証プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/user_authentication.md
