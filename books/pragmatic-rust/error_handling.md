---
title: "実用: エラーハンドリング"
---

良いコード悪いコードの評価軸の1つにエラーハンドリングがあると思います。エラーハンドリングがどうだったら良いコードなのでしょうか？

1つ目は自動化です。なにかエラーが起きてもリトライとかで回復できるものなら自動で復旧できるようになります。

2つ目が可視化です。「なにが原因でエラーなのか？」「回避・復旧するためのアクションはなにか」をソフトウェアのユーザーに伝えるのが役割です。良いコードはエラーの内容を見ただけで回避・復旧のアクションができます。悪いコードはコードを読まないとエラーの原因や復旧手段がわからないです。

Rustは言語設計の点で良いエラーハンドリングが書きやすくなっているように感じます。さらなる改善のために既存の資産 = ライブラリを活かすと便利です。

## anyhowとthiserror
安定と信頼のdtolnay先生作のエラーハンドリングのためのライブラリとしてanyhowとthiserrorの2つのcrateがあります。

使い分けとしては、ライブラリコードではthiserrorを使う、アプリケーションコードではanyhowを使うでOKです。

- anyhow: https://docs.rs/anyhow/latest/anyhow/index.html
- thiserror: https://docs.rs/thiserror/latest/thiserror/

## anyhow
`anyhow::Result` と便利マクロがメインのライブラリです。アプリケーションを素早く書くのに欠かせない。

### `anyhow::Result`
`anyhow::Result` はだいたいどんなエラーも `anyhow::Error` に押し込んでくれる便利型です。`anyhow::Error` のメンタルモデル的には `Box<dyn std::error::Error>` が近いです。

```rust
use anyhow::Result;

fn get_cluster_info() -> Result<ClusterMap> {
    // std::io::Errorが返ってくる
    let config = std::fs::read_to_string("cluster.json")?;
    // serde_json::Errorが返ってくる
    let map: ClusterMap = serde_json::from_str(&config)?;
    Ok(map)
}
```

`anyhow::Result` を使っている時に型が合わなくてコンパイルエラーが出ている時は、エラーメッセージを一旦読んで考えた後にreturnしているexpressionを `anyhow::Ok(<expr>)` で囲んでみてエラーメッセージを再度見てみるといいかもしれないです。あとよくあるのは `anyhow::Result` をuseしてないがち。

### `anyhow::Context`
`Result` にお手軽にエラー内容を足せる便利拡張traitです。`core::result::Result` にimplmentしてくれてるのでだいたいどこでも使えます。拡張traitなので `use anyhow::Context as _` みたく名前をつけなくてOKです。

```rust
use anyhow::{Context, Result};

let content = std::fs::read(path)
        .with_context(|| format!("Failed to read instrs from {}", path))?;
```

`core::option::Option` にもimplしてくれてるのが最高に便利で、手軽にResultに変換できてかつエラーメッセージを付与できて良いエラーハンドリングに繋がります。

```rust
use anyhow::Context as _;

let commit = res
            .body
            .first() // Optionを返す
            .with_context(|| format!("no commits found: owner={owner}, repo={repo}"))?;
```

### 便利マクロ
`bail!`: `return Err(anyhow::Error)` を短く書ける。フォーマット処理もしてくれるので `format!` マクロも省ける。

```rust
let path = resolve_path(args.report_path, &root);
if path.extension().unwrap_or_default() != "json" {
    bail!("JSON file extension expected: {}", path.display())
};
```

### Downcasting
`anyhow::Result` を返す関数の戻り値に対して、特定のエラーの時だけ処理をしたい時は `downcast_ref` を使います。
こういうユースケースではアプリケーション内であってもthiserrorでエラー型を定義しておくと分岐に便利です。

```rust
let cloned = match self.checkout.create_dir_and_checkout(&checkout_input).await {
    Ok(v) => v,
    Err(e) => {
        match e.downcast_ref::<CheckoutError>() {
            Some(CheckoutError::Timeout(d)) => {
                info!(duration = %d, "checkout timed out");
                self.report_timeout(owner, repo, &check_run, input, d)
                    .await?;
                // This seems not orgu failure, so early return Ok.
                return Ok(());
            }
            _ => return Err(e),
        }
    }
};
```

### バックトレース
`RUST_BACKTRACE=1` をつけて実行するとバックトレースが出て便利。

## thiserror
エラー型定義用の便利ライブラリです。大抵のアプリケーションコードでは使い方が限られているので、だいたいdocsのexampleを見たら雰囲気で使えると思います。

```
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("unknown data store error")]
    Unknown,
}
```

## Resultとドメインの失敗
エラーハンドリングあるあるなのが、ドメインロジックの失敗もエラーに押し込もうとしてdowncastingが大量に必要になったりしておかしなことになります。エラーではない(ドメインの)失敗を含むようなドメインロジックは `XxxOutcome` のようなenumを定義して、正常系(`Ok` の方)で戻り値を返してあげてパターンマッチで処理するのが良いです。

```rust
enum XxxOutcome {
    Success,
    AnotherSituation,
    YyyFailure(String),
    ZzzFailure(String),
}

let out = domain_process()?;
match out {
  XxxOutcome::Success => <expr>,
  XxxOutcome::AnotherSituation => <expr>,
  XxxOutcome::YyyFailure => <expr>,
  XxxOutcome::ZzzFailure => <expr>,
};
```

axumのようなweb application frameworkなどでは `Err` の方で返した方が都合がいいこともあるのでケースバイケースですが。

Rustでは `Result` という単語をドメイン内の命名で使いづらいので `Outcome` という単語を使っていますが、もっといい単語あれば知りたいです。

## 余談
ちなみに同僚はアプリケーションコードであってもevolvingしないソフトウェアであれば、エラーの一覧性が高いので、thiserrorのみを使って `map_err` でなんとかしてるという話もあります。参考まで。
