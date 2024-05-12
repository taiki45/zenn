---
title: "交換可能性"
---

実世界のアプリケーションコードを書いている時に、テストや異なるシナリオで動作を変えるために交換可能性が必要となることがあります。コンパイル時にどの関数が呼ばれるかわかるような静的な交換可能性には静的ディスパッチが、コンパイル時にどの関数呼ばれるかわからないような動的な交換可能性には動的ディスパッチがそれぞれ利用できます。

## 静的ディスパッチ
渡される型毎に実装を生成するので、実行時のオーバーヘッドがなく(もしくは少なく)、コンパイル時の最適化の余地もあり、パフォーマンス的に有利です。代わりに生成するコードが増えるので、コンパイル時間は(少し)増加します。

テスト用の実装置き換えやプログラムのエントリーポイントでシナリオが決まっているようなケースでは、静的にメソッドが解決可能なので静的ディスパッチで十分です。

例えば、GitHubのwebhookイベントを処理するようなハンドラー書くとします。さらに、処理の最中にGitHub APIとやりとりしたいのでGitHubクライアントを利用したいとします。この時に処理関数の中で毎回クライアントを生成するのではなく、ハンドラーを構造体にしてその構造体のフィールドにクライアントを持たせるとします。このクライアントをテスト時はモックに置き換えたい時の例です:

まずtraitを定義します。async関係は後のチャプターで紹介します。

```rust
#[allow(clippy::indexing_slicing)] // For automock.
#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait GithubClient: Send + Sync {
    async fn create_check_run(
        &self,
        owner: &str,
        repo: &str,
        input: &ChecksCreateRequest,
    ) -> Result<CheckRun>;

    async fn update_check_run(
        &self,
        owner: &str,
        repo: &str,
        check_run_id: i64,
        input: &ChecksUpdateRequest,
    ) -> Result<CheckRun>;
}
```

実際にGitHub APIと通信する具体的な型を定義します。余談ですが、octorust crateを新規に利用するのはおすすめしないです😇

```rust
pub struct OctorustClient {
    checks: Checks,
    repos: Repos,
    http: ClientWithMiddleware,
}

impl OctorustClient {
    // Traitに関係ないassociated functionとかメソッドも自由に定義して使えます。
    pub fn new(config: GithubApiConfig, app: GithubAppConfig) -> Result<Self> {
        // ...
    }
}

#[async_trait]
impl GithubClient for OctorustClient {
    async fn create_check_run(
        &self,
        owner: &str,
        repo: &str,
        input: &ChecksCreateRequest,
    ) -> Result<CheckRun> {
        // ...
    }

    async fn update_check_run(
        &self,
        owner: &str,
        repo: &str,
        check_run_id: i64,
        input: &ChecksUpdateRequest,
    ) -> Result<CheckRun> {
        // ...
    }
}
```

ハンドラーに持たせます。`GitHubClient` はtraitで `CL` はtype parameterです。

```rust
pub struct Handler<CL: GithubClient, CH: Checkout, F: TokenFetcher> {
    config: Config,
    runner_job_name: String,
    client: CL,
    checkout: CH,
    token_fetcher: F,
}
```

実際にGitHubのwebhookイベントを処理するサーバーなどのエントリーポイントで、ハンドラー構造体を初期化して処理をさせます。

```rust
let client = OctorustClient::new(args.github_config.clone(), args.github_app_config.clone())?;
// ...
let handler = Handler::new(args.handler_config, client, checkout, fetcher);

let service = service_fn(|event: LambdaEvent<EventBridgeEvent<CheckRequest>>| {
    let h = &handler;
    async move {
        let res = h.handle_event(&event.payload.detail).await;
        if let Err(e) = res.as_ref() {
            error!("handling event failed: {}", e);
        }
        res
    }
});
```

テスト時はGitHubクライアントの実装はモックに切り替えたいです。`GithubClient` traitを実装した具体的な型である、モック構造体を用意します。ここでは[mockall](https://docs.rs/mockall/latest/mockall/)というcrateを使って自動でモック構造体を生成しています。

```rust
// 再掲
#[allow(clippy::indexing_slicing)] // For automock.
#[cfg_attr(test, mockall::automock)]
#[async_trait]
pub trait GithubClient: Send + Sync {
    async fn create_check_run(
        &self,
        owner: &str,
        repo: &str,
        input: &ChecksCreateRequest,
    ) -> Result<CheckRun>;

    async fn update_check_run(
        &self,
        owner: &str,
        repo: &str,
        check_run_id: i64,
        input: &ChecksUpdateRequest,
    ) -> Result<CheckRun>;
}
```

テストコードのエントリーポイントでハンドラー構造体を初期化する時に、モック構造体をハンドラーに渡します。こうしてテスト時はモックに交換できます。

```rust
#[tokio::test]
async fn ok() {
    // ...

    let mut client = MockGithubClient::new();

    // ...

    let handler = Handler::new(config, client, checkout, fetcher);

    let mut req = build_checkrequest();
    // ...
    let res = handler.handle_event(&req).await;
    // ...
}
```

最初は慣れないかもしれないですが、この例で言う `OctorustClient` と `MockGithubClient` のそれぞれのバージョンに対応する関数や構造体をコンパイラが別々に生成しているんだな、というメンタルモデルでOKです。

## 動的ディスパッチ
例えば、「ユーザー入力を元に文字列を出力する先を決める」ような交換可能性を実現するためには動的ディスパッチを利用します。Rustではtraitをtrait objectsとして利用することで動的ディスパッチを実現できます。Trait objectsはポインタ型と `dyn` キーワードを使って書きます。例としてよく `Box` 型が使われますが、参照でも他のスマートポインタ型 (`Arc` など) でも可能です。

例の「ユーザー入力を元に文字列を出力する先を決める」ケースでは次のようになります。

```rust
let mut out: Box<dyn Write> = match args.output {
    Some(path) => Box::new(File::create(path)?),
    None => Box::new(stdout()),
};
write!(&mut out, "{}", toml::to_string(&config)?)?;
```

動的ディスパッチは他の言語でもよくあるので特に戸惑うことなく利用できると思います。

一点、Rustには `Deref` traitという仕組みがあり、`Box` のようなスマートポインタも通常の参照のようにdereference operator (`*`)を使わずに内部の型のメソッドにアクセスできるようになっています。`Box` や `Arc` 型なのに、内部の型を使っているように見えるのはこの仕組みのおかげです。ナイス抽象！

## 関連資料
- https://doc.rust-lang.org/book/ch17-02-trait-objects.html
- https://doc.rust-lang.org/book/ch19-04-advanced-types.html#dynamically-sized-types-and-the-sized-trait
- https://doc.rust-lang.org/book/ch15-02-deref.html#treating-smart-pointers-like-regular-references-with-the-deref-trait
