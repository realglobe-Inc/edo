EDO
========

EDOは、HTTPベースで構成されたプラットフォームフレームワークです。様々な主体によって運営されたアプリケーションが混在するプラットフォーム上で、安全性を担保しつつ、ユーザーの様々なデータを相互に活用することできるような枠組みを規定しています。

株式会社リアルグローブが開発して、その参照実装群である「edo-toolkit」と共に公開している、日本発のオープンソースプロジェクトです。

あらゆるパーソナルレコード（PxR；Personal x Record）を安全に相互活用することができるコンパクトなプラットフォームを実現するためのスケーラブルなフレームワークを提供することがEDOの目的です。


EDOは、以下のことを目標としています。
+ パーソナルデータを安全に相互活用する際に必要な様々な規定を提供する。
+ HTTPベースのAPIを持つコンポーネントとして全てが部品化される。
+ シボレスやオープンIDコネクトなどの様々な従来技術と柔軟に組み合わせて拡張できる。


EDOは、以下のようなデータを安全に相互活用するプラットフォームの構築に適しています。
+ 学習記録データ（PLR;Personal Learning Record）
+ 健康/医療データ（PHR;Personal Healthcare Record）
+ 飲食記録データ（PDR;Personal Diet Record）
+ 購買記録データ（PSR;Personal Shopping Record）
+ 就業履歴データ（PJR;Personal Work Record）


EDOは、以下のようなコンポーネントから構成されています。
+ IdP（ID Providor）
+ SP/TA（Service Providor / Trusted Agent）
+ PDS（Personal Data Store）
+ DS（Discovery Service）
