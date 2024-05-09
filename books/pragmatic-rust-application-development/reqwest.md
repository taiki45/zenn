---
title: "reqwest"
---

## Clientの注意点
内部で `Arc` でクライアント実装をラップしているので、webアプリケーションなどでクライアントを再利用したい時にユーザー側で `Arc` でラップしなくてよくなってたりします。

https://docs.rs/reqwest/latest/reqwest/struct.Client.html

## タイムアウト・リトライ
reqwest-middleware crateがあって `reqwest::Client` を拡張してくれます。リトライはreqwest-retry crateを使うのがおすすめです。`read_timeout` は比較的新しいメソッドなので使用しているreqwestのバージョンに注意。

タイムアウトとリトライを設定する例です:

```rust
pub fn reqwest_client(config: GithubApiConfig) -> Result<ClientWithMiddleware> {
    let http = reqwest::Client::builder()
        .connect_timeout(config.github_connect_timeout.into())
        .read_timeout(config.github_read_timeout.into())
        .build()?;
    let retry_policy = ExponentialBackoff::builder()
        .jitter(config.github_retry_jitter.into())
        .base(config.github_retry_base)
        .retry_bounds(
            config.github_min_retry_interval.into(),
            config.github_max_retry_interval.into(),
        )
        .build_with_max_retries(config.github_max_retry);

    Ok(ClientBuilder::new(http)
        .with(RetryTransientMiddleware::new_with_policy(retry_policy))
        .build())
}
```

https://docs.rs/reqwest-middleware/latest/reqwest_middleware/

https://docs.rs/reqwest-retry/latest/reqwest_retry/

## 小ネタ: `bearer_auth`, `basic_auth`
APIクライアントを書いている時にbearerトークンをセットする時に生のヘッダーを使わなくてもこのメソッドがあります。他にも便利メソッドがあったりするのでドキュメントを一読あれ。

https://docs.rs/reqwest/latest/reqwest/struct.RequestBuilder.html
