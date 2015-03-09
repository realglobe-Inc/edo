# PDS データアクセス API


## 概要

TA 間連携プロトコルの利用を前提とする。
TA 間連携プロトコルにより、アクセスの主体となるユーザーおよびアクセス元 TA が PDS に通知される。
加えて、アクセスするデータと操作の種類を指定することで、データアクセスに必要な情報が揃う。


## 1. データ指定

### 1.1. 指定項目

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


### 1.1.1 項目の例

|項目|例|
|:--|:--|
|ユーザータグ|user|
|TA の ID|https://writer.example.org|
|パス|/profile/career|
|データタイプ|text|


### 1.2. URL による指定

ユーザータグ、TA の ID はパーセントエンコードした上で、パスと共に以下のように指定する。

```
/<ユーザータグ>/<TA の ID><パス>
```

データタイプは以下のパラメータで指定する。

* **`cty`**
    * データタイプ。


### 1.2.1 URL による指定の例

```
/user/https%3A%2F%2Fwriter.example.org/profile/career?cty=text
```


### 1.3. JSON による指定

以下の要素を含むオブジェクトにより指定する。

* **`user_tag`**
    * ユーザータグ。
* **`ta`**
    * TA の ID。
* **`path`**
    * パス。
* **`cty`**
    * データタイプ。


### 1.3.1. JSON による指定の例

```JSON
{
    "user_tag": "user",
    "ta": "https://writer.example.org",
    "path": "/profile/career",
    "cty": "text"
}
```


## 2. 操作の種類

|メソッド|操作|
|:--|:--|
|GET|読み取り|
|PUT|書き込み・作成|
|DELETE|削除|


## 3. 読み取り操作

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
      ディレクトリの中のデータおよびディレクトリに対して追加で読み込む情報の種類の配列。
      以下が存在し、無指定なら名前とディレクトリのみ。
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
|再帰フラグ|`recursive` に `true` または `false` で|`recursive` に真偽値で|

ディレクトリでないデータ本体を除き、情報は全て JSON で返す。
読み込みタイプが content を含む場合、データ本体はレスポンスボディに、その他の情報は X-Pds-Datainfo ヘッダに入れて返される。

|ヘッダ名|値|
|:--|:--|
|X-Pds-Datainfo|メタデータやアクセス権限を表す JSON|


### 3.1. データの読み取り

データをそのまま返す。

#### 3.1.1. データの読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career HTTP/1.1
Host pds.example.org
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


### 3.2. ディレクトリの読み取り

ディレクトリの中に入っているデータおよびディレクトリの情報を配列で返す。


#### 3.2.1. ディレクトリの読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/ HTTP/1.1
Host pds.example.org
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
        "is_dir": true
    }
]
```

### 3.3. メタデータの読み取り

サイズや更新日時等を返す。


#### 3.3.1. メタデータの読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career?rty=metadata HTTP/1.1
Host pds.example.org
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


### 3.4. アクセス権限の読み取り

アクセス権限を返す。


#### 3.4.1. アクセス権限の読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career?rty=permission HTTP/1.1
Host pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: application/json

{
    "read": true
}
```


### 3.5. 複合読み取り

`rty` で複数情報を指定すれば、複合的な情報を取得できる。


#### 3.5.1. 複合読み取り例

リクエストは、

```HTTP
GET /user/https%3A%2F%2Fwriter.example.org/profile/career?
    rty=metadata+permission HTTP/1.1
Host pds.example.org
```

改行とインデントは表示の都合による。
TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 200 OK
Content-Type: plain/text
X-PDS-Datainfo: {"name":"career","bytes":102,"created_at":"2013-03-09T18:44:40+0900",
    "updated_at":"2014-01-15T10:23:09+0900","read":true}

2012/03 博士号取得
        無職
2014/01 （株）リアルグローブ入社
        在職中
```

ヘッダの改行とインデントは表示の都合による。


## 4. 書き込み・作成操作

指定項目として読み取り操作と同じ再帰フラグが追加される。


### 4.1. データの書き込み

そのまま書き込む。


#### 4.1.1. データの書き込み例

```HTTP
PUT /user/https%3A%2F%2Fwriter.example.org/profile/hobby&cty=text HTTP/1.1
Host pds.example.org

食っちゃ寝。
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```

### 4.2. ディレクトリの作成

ディレクトリを作成する。


#### 4.2.1. ディレクトリの作成例


```HTTP
PUT /user/https%3A%2F%2Fwriter.example.org/secret/himitsu/maruhi/gokuhi/?recursive=true HTTP/1.1
Host pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


## 5. 削除操作

指定項目として読み取り操作と同じ再帰フラグが追加される。


### 5.1. データの削除

データを削除する。


#### 5.1.1. データの削除例

```HTTP
DELETE /user/https%3A%2F%2Fwriter.example.org/profile/hoby HTTP/1.1
Host pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```


### 5.2. ディレクトリの削除

ディレクトリを削除する。


#### 5.2.1. ディレクトリの削除例

```HTTP
DELETE /user/https%3A%2F%2Fwriter.example.org/secret/himitsu/?recursive=true HTTP/1.1
Host pds.example.org
```

TA 間連携プロトコルの付加情報は省いている。

レスポンスは、

```HTTP
HTTP/1.1 204 No Content
```

