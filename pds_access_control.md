# PDS 権限変更プロトコル

要請元 TA が PDS のデータアクセス権限の変更を要請するためのプロトコル。


## 1. 概要

1. 要請元 TA から PDS に、変更内容を通知する。
2. PDS から要請元 TA に、変更チケットを発行する。
3. 要請元 TA から PDS に、変更チケットと共にユーザーをリダイレクトさせる。
4. PDS とユーザー間で、合意を形成する。
5. PDS から要請元 TA に、結果と共にユーザーをリダイレクトさせる。


## 2. TA 間連携とユーザー認証

本プロトコルでは、途中、TA 間連携プロトコルとユーザー認証プロトコルを利用する。

要請元 TA から PDS への変更内容の通知は TA 間連携により行われ、PDS とユーザー間で合意を形成する前にはユーザー認証が必要になる。


## 3. 操作タグ

要請元 TA は、全ての変更対象に対して、その処理において一意のタグをつけなければならない。


## 4. 変更要請リクエスト

要請元 TA から PDS の変更要請エンドポイントに、TA 間連携プロトコルの付加情報と共に、以下の最上位要素を含む JSON オブジェクトを送る。
ユーザータグは TA 間連携プロトコルで付けたものと同じでなければならない。

* **`chmod`**
    * 必須。
      操作タグから変更対象へのマップ。
      変更対象は以下の要素を含むオブジェクト。
        * `user_tag`
            * 必須。
              データ保持者のユーザータグ。
        * `ta`
            * 必須。
              データ領域を割り当てられた TA の ID。
        * `path`
            * 必須。
              データまたはデータディレクトリのパス。
        * `mod`
            * 必須。
              変更する権限。
              `+r` 等。
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
    * OpenID Connect 1.0 の Authentication Request の `display` と同じ。
* **`ui_locales`**
    * OpenID Connect 1.0 の Authentication Request の `ui_locales` と同じ。


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
            "essential": true
        },
        "diary": {
            "user_tag": "user",
            "ta": "https://writer.example.org",
            "path": "/diary",
            "mod": "+r"
        }
    },
    "redirect_uri": "https://reader.example.org/return/chmod",
    "state": "SiuR29g1Iu"
}
```

TA 間連携プロトコルの付加情報は省いている。


## 5. 変更要請レスポンス

リクエストに問題が無ければ、PDS から要請元 TA に、以下の最上位要素を含む JSON オブジェクトを返す。

* **`code`**
     * 必須。
       変更チケット。


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
      変更チケット。


### 6.1. PDS にリダイレクトさせるためのレスポンス例

```HTTP
HTTP/1.1 302 Found
Location: https://pds.example.org/access-control/user?
    code=6YYIWD6mH9B6ER5eDw3AkAnnCi-aPNutiqI6IwHN
```

Location 値の改行は表示の都合による。


## 7. ユーザーと PDS 間での合意形成


### 7.1. 合意内容

全ての変更対象に対して、ユーザーから以下のいずれかの合意を得なければならない。

* **適用**
    * 変更を行い、要請元 TA が求める状態にする。
      元から適用された状態だった場合は、非明示的に**適用**で合意したとみなしても良い。
      ユーザーに変更権限が無い場合 (ユーザーとデータ保持者が異なる場合の多く) は**適用**で合意してはならない。
* **転送**
    * データ保持者のキューに、ユーザーと要請元 TA からの要求として変更対象を入れる。
      既に同じ内容の要求がキューに入っている場合は、非明示的に**転送**で合意したとみなしても良い。
      ユーザーに変更権限がある場合は**転送**で合意してはならない。
* **拒否**
    * 適用も転送もしない。


### 7.2. 合意形成の方法

ユーザーと PDS 間での合意形成の具体的な方法は本仕様では規定しない。


## 8. 要請元 TA へのリダイレクト

ユーザーから必要な同意が得られたら、PDS は以下のパラメータを付加してユーザーを変更要請リクエストの `redirect_uri` へリダイレクトさせる。

* **`applied`**
    * 適用された変更対象があった場合は必須。
      そうでなければ任意。
      全ての適用された変更対象の操作タグの JSON 配列。
* **`queued`**
    * 転送された変更対象があった場合は必須。
      そうでなければ任意。
      全ての転送された変更対象の操作タグの JSON 配列。
* **`denied`**
    * 拒否された変更があった場合は必須。
      そうでなければ任意。
      全ての拒否された変更対象の操作タグの JSON 配列。


### 8.1. 要請元 TA にリダイレクトさせるためのレスポンス例

```HTTP
HTTP/1.1 302 Found
Location: https://reader.example.org/return/chmod?
    applied=%5B%22profile%22%5D
    &denied=%5B%22diary%22%5D
```

Location 値の改行は表示の都合による。
