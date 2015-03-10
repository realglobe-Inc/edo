# PDS データアクセス API


## 1. 概要

TA 間連携プロトコルの利用を前提とする。
TA 間連携プロトコルにより、アクセスの主体となるユーザーおよびアクセス元 TA が PDS に通知される。
加えて、アクセスするデータと操作の種類を指定することで、データアクセスに必要な情報を揃える。


## 2. データ指定


### 2.1. 指定項目

以下の項目によりデータを指定する。

* ユーザータグ
    * 必須。
      アクセスするデータの保持者のユーザータグ。
      TA 間連携プロトコルで付けたものと同じでなければならい。
* TA の ID
    * 必須。
      アクセスするデータの領域を割り当てられた TA の ID。
* パス
    * 必須。
      データまたはディレクトリのパス。
      末尾が / ならディレクトリを指定しているとみなす。
      これはデータタイプを `directory` にすることと等しい。
* データタイプ
    * 任意。
      データの型。
      実際のデータの型と異なった場合は拒否される。
      ディレクトリなら `directory`。


### 2.1.1. 項目の例

|項目|例|
|:--|:--|
|ユーザータグ|user|
|TA の ID|https://writer.example.org|
|パス|/profile/career|
|データタイプ|text|


### 2.2. URL による指定

ユーザータグ、TA の ID をパーセントエンコードした上で、パスと共に以下のように指定する。

```
/<ユーザータグ>/<TA の ID><パス>
```

データタイプは以下のパラメータで指定する。

* **`cty`**
    * データタイプ。


### 2.2.1. URL による指定の例

```
/user/https%3A%2F%2Fwriter.example.org/profile/career?cty=text
```


### 2.3. JSON による指定

以下の要素を含むオブジェクトにより指定する。

* **`user_tag`**
    * ユーザータグ。
* **`ta`**
    * TA の ID。
* **`path`**
    * パス。
* **`cty`**
    * データタイプ。


### 2.3.1. JSON による指定の例

```json
{
    "user_tag": "user",
    "ta": "https://writer.example.org",
    "path": "/profile/career",
    "cty": "text"
}
```


## 3. 操作の種類

|HTTP メソッド|操作|
|:--|:--|
|GET|読み取り|
|PUT|書き込み・作成|
|DELETE|削除|


## 4. 読み取り操作

以下の指定項目が追加される。

* 読み込みタイプ
    * 任意。
      読み込む情報の種類の配列。
      以下が存在し、無指定なら `content` のみとみなす。
        * `content`
            * データ本体。
        * `metadata`
            * サイズ、更新日時等のメタデータ。
        * `permission`
            * アクセス権限。
* ディレクトリ内読み込みタイプ
    * 対象がディレクトリなら任意。
      そうでなければ無し。
      ディレクトリの中のデータおよび中のディレクトリに対して追加で読み込む情報の種類の配列。
      以下が存在し、無指定なら名前とディレクトリかどうかのみ読み込む。
        * `metadata`
            * サイズ、更新日時等のメタデータ。
        * `permission`
            * アクセス権限。
* 再帰フラグ
    * 対象がディレクトリなら任意。
      そうでなければ無し。
      ディレクトリに対して再帰的に操作を適用するかどうか。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|読み込みタイプ|`rty` に空白区切りで|`rty` に配列で|
|ディレクトリ内読み込みタイプ|`dir_rty` に空白区切りで|`dir_rty` に配列で|
|再帰フラグ|`recursive` に `true`/`false` で|`recursive` に真偽値で|

ディレクトリでないデータ本体を除き、情報は基本 JSON で返す。
読み込みタイプが content を含む場合、データ本体はレスポンスボディに、その他の情報は JWT で X-Pds-Datainfo ヘッダに入れて返す。

|ヘッダ名|値|
|:--|:--|
|X-Pds-Datainfo|メタデータやアクセス権限を含む JWT|


### 4.1. データの読み取り

データをそのままレスポンスボディに入れて返す。


#### 4.1.1. データの読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: plain/text

2012/03 博士号取得
        無職
2014/01 （株）リアルグローブ入社
        在職中
```


### 4.2. ディレクトリの読み取り

ディレクトリの中に入っているデータおよびディレクトリの情報を JSON 配列でレスポンスボディに入れて返す。


#### 4.2.1. ディレクトリの読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/?recursive=true HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

[
    {
        "name": "career"
    },
    {
        "name": "tmp",
        "is_dir": true,
        "children": [
            {
                "name": "test"
            }
        ]
    }
]
```


### 4.3. メタデータの読み取り

サイズや更新日時等を JSON オブジェクトでレスポンスボディに入れて返す。


#### 4.3.1. メタデータの読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career?rty=metadata HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "name": "career",
    "bytes": 102,
    "created_at": "2013-03-09T18:44:40+0900",
    "updated_at": "2014-01-15T10:23:09+0900"
}
```


### 4.4. アクセス権限の読み取り

アクセス権限の内、TA 間連携プロトコルで通知されたユーザーに関わるものだけを JSON オブジェクトでレスポンスボディに入れて返す。

JSON オブジェクトは次の最上位要素からなる。

* **`permission`**
    * TA 間連携プロトコルで付けたユーザータグからアクセス元 TA ごとのアクセス権限へのマップ。
      特殊なユーザータグとして `*` は全てのユーザーを表す。


#### 4.4.1. アクセス権限の読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career?rty=permission HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "permission": {
        "self": {
            "https://writer.example.org": {
                "read": true,
                "write": true
            },
            "https://reader.example.org": {
                "read": true
            }
        },
        "observer": {
            "https://reader.example.org": {
                "read": true
            }
        }
    }
}
```


### 4.5. 複合読み取り

`rty` で複数情報を指定すれば、複合的な情報を取得できる。

`metadata` と `permission` が同時に指定された場合、ひとまとめの JSON オブジェクトにして返す。
また、`content` とその他の情報を同時に指定した場合、その他の情報は JWT で X-Pds-Datainfo ヘッダに格納して返す。


#### 4.5.1. 複合読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career?
    rty=content+metadata+permission HTTP/1.1
Host: pds.example.org
```

改行とインデントは表示の都合による。
TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: plain/text
X-Pds-Datainfo: eyJhbGciOiJub25lIn0.eyJieXRlcyI6MTAyLCJjcmVhdGVkX2F0IjoiMjAxMy0w
    My0wOVQxODo0NDo0MCswOTAwIiwibmFtZSI6ImNhcmVlciIsInBlcm1pc3Npb24iOnsib2JzZXJ2
    ZXIiOnsiaHR0cHM6Ly9yZWFkZXIuZXhhbXBsZS5vcmciOnsicmVhZCI6dHJ1ZX19LCJzZWxmIjp7
    Imh0dHBzOi8vcmVhZGVyLmV4YW1wbGUub3JnIjp7InJlYWQiOnRydWV9LCJodHRwczovL3dyaXRl
    ci5leGFtcGxlLm9yZyI6eyJyZWFkIjp0cnVlLCJ3cml0ZSI6dHJ1ZX19fSwidXBkYXRlZF9hdCI6
    IjIwMTQtMDEtMTVUMTA6MjM6MDkrMDkwMCJ9.

2012/03 博士号取得
        無職
2014/01 （株）リアルグローブ入社
        在職中
```

ヘッダの改行とインデントは表示の都合による。
JWT のクレームセットの内容は、

```json
{
    "name": "career",
    "bytes": 102,
    "created_at": "2013-03-09T18:44:40+0900",
    "updated_at": "2014-01-15T10:23:09+0900",
    "permission": {
        "self": {
            "https://writer.example.org": {
                "read": true,
                "write": true
            },
            "https://reader.example.org": {
                "read": true
            }
        },
        "observer": {
            "https://reader.example.org": {
                "read": true
            }
        }
    }
}
```


## 5. 書き込み・作成操作

次の指定項目が追加される。

* 親ディレクトリ作成フラグ
    * 必要なら親ディレクトリまでを作成するかどうか。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|親ディレクトリ作成フラグ|`parents` に `true`/`false` で|`parents` に真偽値で|


### 5.1. データの書き込み

リクエストボディをそのまま書き込む。


#### 5.1.1. データの書き込み例

リクエストは、

```HTTP
PUT /user/https%3A%2F%2Fwriter.example.org/profile/hobby HTTP/1.1
Host: pds.example.org

食っちゃ寝。
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


### 5.2. ディレクトリの作成

指定したディレクトリを作成する。


#### 5.2.1. ディレクトリの作成例

リクエストは、

```HTTP
PUT /user/https%3A%2F%2Fwriter.example.org/secret/himitsu/maruhi/gokuhi/?parents=true HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


## 6. 削除操作

指定項目として読み取り操作と同じ再帰フラグが追加される。
空でないディレクトリを削除する場合、再帰フラグが必須になる。


### 6.1. データの削除

指定したデータを削除する。


#### 6.1.1. データの削除例

リクエストは、

```HTTP
DELETE /user/https%3A%2F%2Fwriter.example.org/profile/hoby HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


### 6.2. ディレクトリの削除

指定したディレクトリを削除する。


#### 6.2.1. ディレクトリの削除例

リクエストは、

```HTTP
DELETE /user/https%3A%2F%2Fwriter.example.org/secret/himitsu/?recursive=true HTTP/1.1
Host: pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


## 7. エラーレスポンス

エラーは OAuth 2.0 Section 5.2 の形式で返す。
