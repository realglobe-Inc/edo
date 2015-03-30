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


# アクセス制御

PDS のデータ等、リソースに対するアクセス権限に基づくアクセス制御について。


## 1. アクセス権限

アクセス権限は、以下の権限を含む。

1. **`r`**
    * 読み取り権限。
2. **`w`**
    * 書き込み権限。

アクセス権限は、権限を組み合わせた文字列で表す。
組み合わせる順番は上記の通りでなければならない。
`rw` は正しいアクセス権限の表記であるが、`wr` はそうではない。


## 2. アクセス制御ルール

アクセス制御ルールは以下の属性に紐付く。

* **`U_access`**
    * アクセスの主体のアカウント。
      特別に全てのアカウントを指定することもできる。
* **`TA_from`**
    * アクセス元 TA。
      特別に全ての TA を指定することもできる。
* 許可するアクセス権限

`U_access` による `TA_from` からのアクセスについて、アクセス内容が許可するアクセス権限の範疇ならば許可し、範疇でなければ拒否することを示す。


## 3. リソース

リソースは以下を含む識別子で指定する。

* **`U_holder`**
    * リソースの保持者のアカウント。
      無しを許しても良い。
* **`TA_master`**
    * リソースを割り当てられている TA。
      無しを許しても良い。
* パス

この識別子に対して以下の属性が紐付く。

* アクセス制御ルール集
    * 識別子で指定されるリソースに対するアクセス制御ルールの集合。


### 3.1. アクセス制御ルール集の制限

アクセス制御ルール集は以下の条件を満たさなければならない。

* 含まれるアクセス制御ルールの `U_access`, `TA_from` の組は重複しない。
* `TA_master` が指定されている場合、`TA_master` と異なる `TA_from` に対して書き込み権限を許可するアクセス制御ルールは含まない。


## 4. アクセス制御

アクセスリクエストからアクセスの主体、アクセス元 TA とデータ識別子を特定し、データ識別子と紐付くアクセス制御ルール集を適用し、リクエストを受け入れるかどうかを決める。


### 4.1. アクセスの主体とアクセス元 TA が特定できる場合

アクセス制御ルール集が以下に該当するアクセス制御ルールを含むなら、以下の順番で最初のものを適用しなければならない。

1. `U_access` がアクセスの主体と一致し、`TA_from` がアクセス元 TA と一致するアクセス制御ルール。
2. `U_access` がアクセスの主体と一致し、`TA_from` が全ての TA であるアクセス制御ルール。
3. `U_access` が全てのアカウントであり、`TA_from` がアクセス元 TA と一致するアクセス制御ルール。
4. `U_access` が全てのアカウントであり、`TA_from` が全ての TA であるアクセス制御ルール。

該当するアクセス制御ルールを含まないなら、拒否しなければならない。


### 5.2. アクセスの主体やアクセス元 TA が特定できない場合

アクセスの主体とアクセス元 TA が特定できる場合と同じように処理しても良い。
この場合、`U_access` や `TA_from` が特定できなかった値と一致することはない。
同じように処理しない場合は拒否しなければならない。
同じように処理するかどうかは、アクセスの主体とアクセス元 TA の両方が特定できなかった場合、アクセスの主体だけが特定できなかった場合、アクセス元 TA だけが特定できなかった場合のそれぞれで別個に決めて良い。
