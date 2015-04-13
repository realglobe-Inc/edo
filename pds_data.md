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
[TA 間連携プロトコル]により、アクセス主体およびアクセス元 TA が PDS に通知される。
加えて、アクセスするデータと操作の種類を指定することで、データアクセスに必要な情報を揃える。


## 2. アクセス制御

データアクセスは [PDS 権限変更プロトコル]等で設定されたアクセス権限に基づき[アクセス制御]される。


## 3. データ型

PDS に格納されるデータは型を持つ。
型によって対応する指定項目が異なる。


### 3.1. 型の種類

以下の型を含む。

* **`octet-stream`**
    * バイト列。
* **`directory`**
    * ディレクトリ。


### 3.2. 型の推定

書き込み・作成操作においてデータ型が指定されない場合、Content-Type ヘッダからデータ型を推定する。
推定不能な場合は `octet-stream` とみなす。


## 4. データ指定

URL または JSON により項目を指定することで、アクセスするデータを指定する。


### 4.1. 指定項目

以下の項目がある。

* アカウントタグ
    * 必須。
      アクセスするデータを所有するアカウントのアカウントタグ。
      [TA 間連携プロトコル]で付けたアカウントタグでなければならない。
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


#### 4.1.1. 項目例

|項目|例|
|:--|:--|
|アカウントタグ|self|
|TA の ID|https://writer.example.org|
|パス|/profile/career|
|データ型|`octet-stream`|


### 4.2. URL による指定

アカウントタグと TA の ID を[パーセントエンコード]した上で、パスと共に以下の形でデータアクセスエンドポイントに続けて指定する。

```
<アカウントタグ>/<TA の ID><パス>
```

データ型は以下のパラメータで指定する。

* **`dty`**
    * データ型。


#### 4.2.1. URL による指定例

```
self/https%3A%2F%2Fwriter.example.org/profile/career?dty=octet-stream
```


### 4.3. JSON による指定

以下の要素を含むオブジェクトにより指定する。

* **`owner_tag`**
    * アカウントタグ。
* **`ta`**
    * TA の ID。
* **`path`**
    * パス。
* **`dty`**
    * データ型。


#### 4.3.1. JSON による指定例

```json
{
    "owner_tag": "self",
    "ta": "https://writer.example.org",
    "path": "/profile/career",
    "dty": "octet-stream"
}
```


## 5. 操作の種類

|HTTP メソッド|操作|
|:--|:--|
|GET|読み取り|
|PUT|書き込み・作成|
|DELETE|削除|


<!--
## x. データ型
### x.1 読み取り操作
#### x.1.1. データ本体の読み取り
##### x.1.1.1. データ本体の読み取り例
#### x.1.2. メタデータの読み取り
##### x.1.2.1. メタデータの読み取り例
#### x.1.3. アクセス権限の読み取り
##### x.1.3.1. アクセス権限の読み取り例
#### x.1.4. 複合読み取り
##### x.1.4.1. 複合読み取り例
### x.2. 書き込み・作成操作
#### x.2.1. データの書き込み例
### x.3. 削除操作
#### x.3.1. データの削除例
-->

## 6. octet-stream データ型

任意のバイト列である `octet-stream` データ型に対する操作について。


### 6.1. 読み取り操作

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
              アクセス権限については[アクセス制御]を参照のこと。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|読み込みタイプ|`rty` に空白区切りで|`rty` に配列で|

データ本体以外の情報は基本 JSON で返す。
読み込みタイプが `content` を含む場合、データ本体はレスポンスボディに、その他の情報は [JWT] にして X-Pds-Datainfo ヘッダに入れて返す。

|ヘッダ名|値|
|:--|:--|
|X-Pds-Datainfo|メタデータやアクセス権限を含む [JWT]|


#### 6.1.1. データ本体の読み取り

データをそのままレスポンスボディに入れて返す。


##### 6.1.1.1. データ本体の読み取り例

リクエストは、

```HTTP
GET /data/self/https%3A%2F%2Fwriter.example.org/profile/career HTTP/1.1
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


#### 6.1.2. メタデータの読み取り

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
* **`create_user`**
    * データを作成したアカウントが [TA 間連携プロトコル]で通知されている場合のみ。
      データを作成したアカウントのアカウントタグ。
* **`updated_at`**
    * 更新日時。[RFC3339] 形式。
* **`update_user`**
    * データを更新したアカウントが [TA 間連携プロトコル]で通知されている場合のみ。
      データを更新したアカウントのアカウントタグ。


##### 6.1.2.1. メタデータの読み取り例

リクエストは、

```HTTP
GET /data/self/https%3A%2F%2Fwriter.example.org/profile/career?rty=metadata HTTP/1.1
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
    "create_user": "self",
    "updated_at": "2014-01-15T10:23:09+0900",
    "update_user": "self"
}
```


#### 6.1.3. アクセス権限の読み取り

アクセス権限の内、[TA 間連携プロトコル]で通知されたアカウントに関わるものだけを JSON オブジェクトでレスポンスボディに入れて返す。

アクセス権限は次の最上位要素に格納する。

* **`permission`**
    * [TA 間連携プロトコル]で付けられたアカウントタグからアクセス元 TA ごとのアクセス権限へのマップ。
      特殊な値として `*` で全てのアカウントや全ての TA を示す。

```json
{
    "permission": {
        <アクセス主体のアカウントタグ>: {
            <アクセス元 TA の ID>: <アクセス権限>,
            ...
            "*": <アクセス権限>
        },
        ...
        "*": {
            <アクセス元 TA の ID>: <アクセス権限>,
            ...
            "*": <アクセス権限>
        }
    }
}
```


##### 6.1.3.1. アクセス権限の読み取り例

リクエストは、

```HTTP
GET /data/self/https%3A%2F%2Fwriter.example.org/profile/career?rty=permission HTTP/1.1
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
            "*": "r"
        },
        "observer": {
            "https://reader.example.org": "r"
        },
        "*": {
            "https://recruit.example.org": "r"
        }
    }
}
```


#### 6.1.4. 複合読み取り

`rty` で複数情報を指定された場合、複合的な情報を返す。

`metadata` と `permission` が同時に指定された場合、ひとまとめの JSON オブジェクトにする。
また、`content` とその他の情報を同時に指定された場合、その他の情報は [JWT] にして X-Pds-Datainfo ヘッダに入れる。


##### 6.1.4.1. 複合読み取り例

リクエストは、

```HTTP
GET /data/self/https%3A%2F%2Fwriter.example.org/profile/career?
    rty=content%20metadata%20permission HTTP/1.1
Host: pds.example.org
```

改行とインデントは表示の都合による。
[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/octet-stream
X-Pds-Datainfo: eyJhbGciOiJub25lIn0.eyJieXRlcyI6MTAyLCJjcmVhdGVfdXNlciI6InNlbGYi
    LCJjcmVhdGVkX2F0IjoiMjAxMy0wMy0wOVQxODo0NDo0MCswOTAwIiwiY3R5Ijoib2N0ZXQtc3Ry
    ZWFtIiwibmFtZSI6ImNhcmVlciIsInBlcm1pc3Npb24iOnsiKiI6eyJodHRwczovL3JlY3J1aXQu
    ZXhhbXBsZS5vcmciOiJyIn0sIm9ic2VydmVyIjp7Imh0dHBzOi8vcmVhZGVyLmV4YW1wbGUub3Jn
    IjoiciJ9LCJzZWxmIjp7IioiOiJyIiwiaHR0cHM6Ly93cml0ZXIuZXhhbXBsZS5vcmciOiJydyJ9
    fSwidXBkYXRlX3VzZXIiOiJzZWxmIiwidXBkYXRlZF9hdCI6IjIwMTQtMDEtMTVUMTA6MjM6MDkr
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
    "dty": "octet-stream",
    "bytes": 102,
    "created_at": "2013-03-09T18:44:40+0900",
    "create_user": "self",
    "updated_at": "2014-01-15T10:23:09+0900",
    "update_user": "self",
    "permission": {
        "self": {
            "https://writer.example.org": "rw",
            "*": "r"
        },
        "observer": {
            "https://reader.example.org": "r"
        },
        "*": {
            "https://recruit.example.org": "r"
        }
    }
}
```


### 6.2. 書き込み・作成操作

リクエストボディをデータとして書き込む。
新規作成されるデータはディレクトリの権限を引き継ぐ。

次の指定項目が追加される。

* 親ディレクトリ作成フラグ
    * 対象のデータを置くディレクトリが存在しない場合に、必要なディレクトリを作成するかどうか。
      無指定の場合、作成しない。
* 作成フラグ
    * 作成であることを明示する。
      既にデータが存在する場合、操作が失敗する。
      無指定の場合、このフラグは立たない。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|親ディレクトリ作成フラグ|`parents` に `true`/`false` で|`parents` に真偽値で|
|作成フラグ|`create` に `true`/`false` で|`create` に真偽値で|


#### 6.2.1. データの書き込み例

リクエストは、

```HTTP
PUT /data/self/https%3A%2F%2Fwriter.example.org/profile/hobby HTTP/1.1
Host: pds.example.org

食っちゃ寝。
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


### 6.3. 削除操作

指定したデータを削除する。


#### 6.3.1. データの削除例

リクエストは、

```HTTP
DELETE /data/self/https%3A%2F%2Fwriter.example.org/profile/hobby HTTP/1.1
Host: pds.example.org
```

[TA 間連携プロトコル]の付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


## 7. directory データ型

`directory` データ型に対する操作について。


### 7.1. 読み取り操作

`octet-stream` 型で追加した読み込みタイプに加えて、以下の指定項目が追加される。

* ディレクトリ内読み込みタイプ
    * 読み込みタイプが `content` を含む場合は任意。
      そうでなければ無し。
      ディレクトリの中のデータに対して追加で読み込む情報の種類の集合。
      以下が存在し、無指定なら名前とデータ型のみ読み込む。
        * `metadata`
            * サイズ、更新日時等のメタデータ。
        * `permission`
            * アクセス権限。
* 再帰フラグ
    * 読み込みタイプが `content` を含む場合は任意。
      そうでなければ無し。
      中のディレクトリに対して再帰的に操作を適用するかどうか。
      無指定の場合、適用しない。

指定方法は以下の通り。

|項目|URL パラメータ|JSON|
|:--|:--|:--|
|ディレクトリ内読み込みタイプ|`dir_rty` に空白区切りで|`dir_rty` に配列で|
|再帰フラグ|`recursive` に `true`/`false` で|`recursive` に真偽値で|

メタデータやアクセス権限の読み取りは `octet-stream` 型と変わらない。


#### 7.1.1. ディレクトリ本体の読み取り

ディレクトリの中のデータの情報を JSON 配列でレスポンスボディに入れて返す。
データの情報は以下の最上位要素を含む。

* **`name`**
    * データの名前。
* **`dty`**
    * データ型。
* **`children`**
    * 再帰フラグが立っていて、かつ、データがディレクトリの場合のみ。
      ディレクトリ内のデータの情報。

ディレクトリ内読み込みタイプに `metadata` や `permission` を指定した場合に追加される情報は、メタデータやアクセス権限の読み取りで得られるものと同じである。


##### 7.1.1.1. ディレクトリの読み取り例

リクエストは、

```HTTP
GET /data/self/https%3A%2F%2Fwriter.example.org/profile/?recursive=true HTTP/1.1
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


### 7.2. 書き込み・作成操作

リクエストボディは使わない。
データ型の指定は必須である。
それ以外は `octet-stream` 型と変わらない。


### 7.3. 削除操作

指定項目として読み取り操作と同じ再帰フラグが追加される。
空でないディレクトリを削除する場合、再帰フラグは必須である。
それ以外は `octet-stream` 型と変わらない。


## 8. エラーレスポンス

エラーは [OAuth 2.0 Section 5.2] の形式で返す。

`error` の値として以下を追加する。

* **`invalid_dty`**
    * データ型が異なる。
* **`not_empty`**
    * ディレクトリが空でない。
* **`not_exist`**
    * 対象のデータが存在しない。

主な `error` の値は以下の通り。

* アカウントタグが [TA 間連携プロトコル]で付けられたものでなかった場合、`invalid_request`。
* データ型が実際のデータ型と異なる場合、`invalid_dty`。
* 対象のデータが存在しない場合、`not_exist`。
* アクセス権限が無い場合、`access_denied`。
* 親ディレクトリ作成フラグ無しで存在しないディレクトリの中にデータを作成しようとした場合、`not_exist`。
* 作成しようとした親ディレクトリが別のデータ型として存在する場合、`invalid_dty`。
* 再帰フラグ無しで空でないディレクトリを削除しようとした場合、`not_empty`。


<!-- 参照 -->
[JWT]: https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32
[OAuth 2.0 Section 5.2]: http://tools.ietf.org/html/rfc6749#section-5.2
[PDS 権限変更プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/pds_change_permission.md
[RFC3339]: http://tools.ietf.org/html/rfc3339
[TA 間連携プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/ta_cooperation.md
[パーセントエンコード]: http://tools.ietf.org/html/rfc3986#section-2.1
[アクセス制御]: https://github.com/realglobe-Inc/edo/blob/master/access_control.md
