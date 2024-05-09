---
title: "tower-http"
---

tower-http crateは便利ミドルウェアが用意されていて便利です。モジュールレベルドキュメントを読めばだいたい使えると思います。

https://docs.rs/tower-http/latest/tower_http/index.html

## `tower_http::normalize_path` とaxum
Webアプリケーションフレームワークとしてaxum crateを使っていると、ミドルウェアスタックはルーティングが終わった後に通過するのでnormalize_pathのようなミドルウェアがワークしないです。他のフレームワークでもありそうなのでここで紹介します。

対処法としては `axum::Router` を作成した後にnomarlize_pathでラップします。

```rust
let router = Router::new()
    .route("/hc", get(health_check))
    .route("/github/events", post(webhook::<_, _, DefaultVerifier>))
    .with_state(shared_state);

let router = apply_middleware(router, &config);
NormalizePathLayer::trim_trailing_slash().layer(router)
```

https://docs.rs/axum/latest/axum/middleware/index.html#rewriting-request-uri-in-middleware

https://docs.rs/tower-http/latest/tower_http/normalize_path/index.html

あとaxum小ネタですが、ミドルウェアスタックの型がすごいことになるので、上記 `apply_middleware` のような `Router -> Router` なヘルパー関数を用意すると便利です。
