---
title: "serde"
---

serdeはドキュメントも利用例も豊富なのであまり紹介することはないですが、「これを先に知っておきたかった」という機能だけ紹介します。


## `rename_all`
例えばデシリアライズするJSONのフィールドが `camelCase` な時に一括でフィールド名の変換を行うやつです。他の言語とかが吐くJSONのフィールド命名規則はそれぞれなのでめちゃ便利です。

https://serde.rs/container-attrs.html#rename_all

## `deny_unknown_fields`
知らないフィールドがあったらエラーにします。より厳格にデシリアライズしたりするのに便利です。

https://serde.rs/container-attrs.html#deny_unknown_fields

## Enum系
https://serde.rs/enum-representations.html

## `flatten`
共通部分をまとめるのに便利です。

https://serde.rs/field-attrs.html#flatten

## `skip_serializing_if`
名前のまんまで便利です。

https://serde.rs/field-attrs.html#skip_serializing_if
