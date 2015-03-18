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


# PDS データアクセス API

データを活用するための API。


## 1. 概要

[TA 間連携プロトコル]の利用を前提とする。
[TA 間連携プロトコル]により、アクセスの主体となるユーザーおよびアクセス元 TA が PDS に通知される。
加えて、アクセスするデータと操作の種類を指定することで、データアクセスに必要な情報を揃える。


## 2. データ型

PDS に格納されるデータは型を持つ。
型によって対応する指定項目が異なる。


### 2.1. 型の種類

以下の型を含む。

* **`octet-stream`**
    * バイト列。
* **`directory`**
    * ディレクトリ


## 3. データ指定

URL または JSON により項目を指定することで、アクセスするデータを指定する。


### 3.1. 指定項目

以下の項目がある。

* ユーザータグ
    * 必須。
      アクセスするデータの保持者のユーザータグ。
      [TA 間連携プロトコル]で付けたユーザータグでなければならない。
* TA の ID
    * 必須。
      アクセスするデータの領域を割り当てられた TA の ID。
* パス
    * 必須。
      データのパス。
      末尾が / ならディレクトリを指定しているとみなす。
      これはデータ型を `directory` にすることと等しい。
* データ型
    * 任意。
      実際のデータ型と異なる場合は拒否される。


#### 3.1.1. 項目例

|項目|例|
|:--|:--|
|ユーザータグ|user|
|TA の ID|https://writer.example.org|
|パス|/profile/career|
|データ型|`octet-stream`|


### 3.2. URL による指定

ユーザータグと TA の ID を [URL エンコード]した上で、パスと共に以下の形でデータアクセスエンドポイントに続けて指定する。

```
<ユーザータグ>/<TA の ID><パス>
```

データ型は以下のパラメータで指定する。

* **`dty`**
    * データ型。


#### 3.2.1. URL による指定例

```
user/https%3A%2F%2Fwriter.example.org/profile/career?dty=octet-stream
```


### 3.3. JSON による指定

以下の要素を含むオブジェクトにより指定する。

* **`user_tag`**
    * ユーザータグ。
* **`ta`**
    * TA の ID。
* **`path`**
    * パス。
* **`dty`**
    * データ型。


#### 3.3.1. JSON による指定例

```json
{
    "user_tag": "user",
    "ta": "https://writer.example.org",
    "path": "/profile/career",
    "dty": "octet-stream"
}
```


## 4. 操作の種類

|HTTP メソッド|操作|
|:--|:--|
|GET|読み取り|
|PUT|書き込み・作成|
|DELETE|削除|


## 5. 汎用読み取り操作

以下の指定項目が追加される。

* 読み込みタイプ
    * 任意。
      読み込む情報の種類の集合。
      以下が存在し、無指定なら `content` のみとみなす。
        * `content`
            * データ本体。
        * `metadata`
            * サイズ、更新日時等のメタデータ。
        * `permission`
            * アクセス権限。
              アクセス権限については [PDS 権限変更プロトコル] を参照のこと。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|読み込みタイプ|`rty` に空白区切りで|`rty` に配列で|

データ本体以外の情報は基本 JSON で返す。
読み込みタイプが `content` を含む場合、データ本体はレスポンスボディに、その他の情報は [JWT] にして X-Pds-Datainfo ヘッダに入れて返す。

|ヘッダ名|値|
|:--|:--|
|X-Pds-Datainfo|メタデータやアクセス権限を含む [JWT]|


### 5.1. データ本体の読み取り

データをそのままレスポンスボディに入れて返す。


#### 5.1.1. データ本体の読み取り例

リクエストは、

```HTTP
GET /data/user/https%3A%2F%2Fwriter.example.org/profile/career HTTP/1.1
Host: pds.example.org
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/octet-stream

2012/03 博士号取得
        無職
2014/01 （株）リアルグローブ入社
        在職中
```


### 5.2. メタデータの読み取り

サイズや更新日時等を JSON オブジェクトでレスポンスボディに入れて返す。
メタデータは以下の最上位要素を含む。

* **`name`**
    * データの名前。
* **`dty`**
    * データ型。
* **`bytes`**
    * データサイズ。
* **`created_at`**
    * 作成日時。[RFC3339] 形式。
* **`updated_at`**
    * 更新日時。[RFC3339] 形式。


#### 5.2.1. メタデータの読み取り例

リクエストは、

```HTTP
GET /data/user/https%3A%2F%2Fwriter.example.org/profile/career?rty=metadata HTTP/1.1
Host: pds.example.org
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "name": "career",
    "dty": "octet-stream",
    "bytes": 102,
    "created_at": "2013-03-09T18:44:40+0900",
    "updated_at": "2014-01-15T10:23:09+0900"
}
```


### 5.3. アクセス権限の読み取り

アクセス権限の内、[TA 間連携プロトコル]で通知されたユーザーに関わるものだけを JSON オブジェクトでレスポンスボディに入れて返す。

アクセス権限は次の最上位要素に格納する。

* **`permission`**
    * [TA 間連携プロトコル]で付けられたユーザータグからアクセス元 TA ごとのアクセス権限へのマップ。
      特殊な値として `*` で全てのユーザーの権限を示す。

```json
{
    "permission": {
        <ユーザータグ>: {
            <アクセス元 TA の ID>: <アクセス権限>,
            ...
        },
        ...
        "*": {
            <アクセス元 TA の ID>: <アクセス権限>,
            ...
        }
    }
}
```


#### 5.3.1. アクセス権限の読み取り例

リクエストは、

```HTTP
GET /data/user/https%3A%2F%2Fwriter.example.org/profile/career?rty=permission HTTP/1.1
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
            "https://writer.example.org": "rw",
            "https://reader.example.org": "r"
        },
        "observer": {
            "https://reader.example.org": "r"
        }
    }
}
```


### 5.4. 複合読み取り

`rty` で複数情報を指定された場合、複合的な情報を返す。

`metadata` と `permission` が同時に指定された場合、ひとまとめの JSON オブジェクトにする。
また、`content` とその他の情報を同時に指定された場合、その他の情報は [JWT] にして X-Pds-Datainfo ヘッダに入れる。


#### 5.4.1. 複合読み取り例

リクエストは、

```HTTP
GET /data/user/https%3A%2F%2Fwriter.example.org/profile/career?
    rty=content+metadata+permission HTTP/1.1
Host: pds.example.org
```

改行とインデントは表示の都合による。
[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/octet-stream
X-Pds-Datainfo: eyJhbGciOiJub25lIn0.eyJieXRlcyI6MTAyLCJjcmVhdGVkX2F0IjoiMjAxMy0w
    My0wOVQxODo0NDo0MCswOTAwIiwiY3R5Ijoib2N0ZXQtc3RyZWFtIiwibmFtZSI6ImNhcmVlciIs
    InBlcm1pc3Npb24iOnsib2JzZXJ2ZXIiOnsiaHR0cHM6Ly9yZWFkZXIuZXhhbXBsZS5vcmciOiJy
    In0sInNlbGYiOnsiaHR0cHM6Ly9yZWFkZXIuZXhhbXBsZS5vcmciOiJyIiwiaHR0cHM6Ly93cml0
    ZXIuZXhhbXBsZS5vcmciOiJydyJ9fSwidXBkYXRlZF9hdCI6IjIwMTQtMDEtMTVUMTA6MjM6MDkr
    MDkwMCJ9.

2012/03 博士号取得
        無職
2014/01 （株）リアルグローブ入社
        在職中
```

ヘッダの改行とインデントは表示の都合による。
[JWT] のクレームセットの内容は、

```json
{
    "name": "career",
    "bytes": 102,
    "dty": "octet-stream",
    "created_at": "2013-03-09T18:44:40+0900",
    "updated_at": "2014-01-15T10:23:09+0900",
    "permission": {
        "self": {
            "https://writer.example.org": "rw",
            "https://reader.example.org": "r"
        },
        "observer": {
            "https://reader.example.org": "r"
        }
    }
}
```


## 6. 汎用書き込み・作成操作

主にリクエストボディをデータとして書き込む。
データ型を指定しない場合、Content-Type ヘッダからデータ型を推定する。
推定不能な場合は `octet-stream` とみなす。

次の指定項目が追加される。

* 親ディレクトリ作成フラグ
    * 対象のデータを置くディレクトリが存在しない場合に、必要なディレクトリを作成するかどうか。
      無指定の場合、作成しない。
* 作成フラグ
    * データ作成であることを明示する。
      既にデータが存在する場合、操作が失敗する。
      無指定の場合、このフラグは立たない。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|親ディレクトリ作成フラグ|`parents` に `true`/`false` で|`parents` に真偽値で|
|作成フラグ|`create` に `true`/`false` で|`create` に真偽値で|


### 6.1. データの書き込み例

リクエストは、

```HTTP
PUT /data/user/https%3A%2F%2Fwriter.example.org/profile/hobby HTTP/1.1
Host: pds.example.org

食っちゃ寝。
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


## 7. 汎用削除操作

指定したデータを削除する。


### 7.1. データの削除例

リクエストは、

```HTTP
DELETE /data/user/https%3A%2F%2Fwriter.example.org/profile/hobby HTTP/1.1
Host: pds.example.org
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


## 8. octet-stream データ型

`octet-stream` データ型には汎用操作と異なる点は無い。


## 9. directory データ型

`directory` データ型における汎用操作と異なる点を挙げる。


### 9.1. 読み取り操作

以下の指定項目が追加される。

* ディレクトリ内読み込みタイプ
    * 任意。
      ディレクトリの中のデータに対して追加で読み込む情報の種類の集合。
      以下が存在し、無指定なら名前とデータ型のみ読み込む。
        * `metadata`
            * サイズ、更新日時等のメタデータ。
        * `permission`
            * アクセス権限。
* 再帰フラグ
    * 任意。
      中のディレクトリに対して再帰的に操作を適用するかどうか。
      無指定の場合、適用しない。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|ディレクトリ内読み込みタイプ|`dir_rty` に空白区切りで|`dir_rty` に配列で|
|再帰フラグ|`recursive` に `true`/`false` で|`recursive` に真偽値で|


#### 9.1.1. ディレクトリの読み取り

ディレクトリの中のデータの情報を JSON 配列でレスポンスボディに入れて返す。
データの情報は以下の最上位要素を含む。

* **`name`**
    * データの名前。
* **`dty`**
    * データ型。
* **`children`**
    * 再帰フラグが立っていて、かつ、データがディレクトリの場合のみ。
      ディレクトリ内のデータの情報。

ディレクトリ内読み込みタイプに `metadata` や `permission` を指定した場合に追加される情報は汎用読み取り操作のものと同じ。


##### 9.1.1.1 ディレクトリの読み取り例

リクエストは、

```HTTP
GET /data/user/https%3A%2F%2Fwriter.example.org/profile/?recursive=true HTTP/1.1
Host: pds.example.org
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

[
    {
        "name": "career",
        "dty": "octet-stream"
    },
    {
        "name": "draft",
        "dty": "directory",
        "children": [
            {
                "name": "family",
                "dty": "octet-stream"
            }
        ]
    }
]
```


### 9.2. 削除操作

指定項目として読み取り操作と同じ再帰フラグが追加される。

空でないディレクトリを削除する場合、再帰フラグは必須とする。


## 10. エラーレスポンス

エラーは [OAuth 2.0 Section 5.2] の形式で返す。

`error` の値として以下を追加する。

* **`directory_not_exist`**
    * 親ディレクトリが存在しない。
* **`invalid_dty`**
    * データ型が異なる。
* **`not_empty`**
    * ディレクトリが空でない。
* **`not_exist`**
    * 対象のデータが存在しない。

主な `error` の値は以下の通り。

* ユーザータグが [TA 間連携プロトコル]で付けられたものでなかった場合、`invalid_request`。
* データ型が実際のデータ型と異なる場合、`invalid_dty`。
* 対象のデータが存在しない場合、`not_exist`。
* アクセス権限が無い場合、`access_denied`。
* 親ディレクトリ作成フラグ無しで親ディレクトリの無いデータを作成しようとした場合、`directory_not_exist`。
* 作成しようとした親ディレクトリが別のデータ型として存在する場合、`invalid_dty`。
* 再帰フラグ無しで空でないディレクトリを削除しようとした場合、`not_empty`。


<!-- 参照 -->
[JWT]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32
[OAuth 2.0 Section 5.2]: http://tools.ietf.org/html/rfc6749#section-5.2
[PDS 権限変更プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/pds_access_control.md
[RFC3339]: http://tools.ietf.org/html/rfc3339
[TA 間連携プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/ta_cooperation.md
[URL エンコード]: http://tools.ietf.org/html/rfc1866#section-8.2.1
