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


# PDS の機能

PDS が持たなければならない、または、持つべき機能集。


## 1. 権限変更プロトコル

PDS は、[権限変更プロトコル]に対応しなければならない。


## 2. データアクセス API

PDS は、[データアクセス API]を提供しなければならない。


## 3. 権限変更要求を処理する UI

PDS は、権限変更権限保持者が[権限変更プロトコル]にて受信した権限変更要求を処理するための UI を持たなければならない。


## 4. 権限変更 UI

PDS は、権限変更権限保持者が権限を確認・変更するための UI を持つべきである。


## 5. インポート / エクスポート

PDS は、異なる実装および運用の間での移行を可能にするために、インポートとエクスポートの機能を持たなければならない。


<!-- 参照 -->
[権限変更プロトコル]: https://github.com/realglobe-Inc/edo/blob/master/pds_change_permission.md
[データアクセス API]: https://github.com/realglobe-Inc/edo/blob/master/pds_data.md
