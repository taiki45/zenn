---
title: "Async Rust"
---

[Async book](https://rust-lang.github.io/async-book/)を読めばOK！

なんですが、async bookをより速く理解するための予備知識をここでは紹介します。

## Async bookの前に
![Futureと非同期処理ランタイムの関係](/images/pragmatic-rust/async.png)

他の言語から来た人にとっての最初の注意点は、Rustの `Future` は「作成した時点で計算が行われるもの」ではないことです。`Future` は主に `poll` とpending/readyの状態を提供するだけの単なるtraitで、tokioなどの非同期処理ランタイムが `Future` などの値を使って非同期処理を実現しています。典型的なユースケースでは、`main` 関数内で非同期処理ランタイムを作成してランタイムのイベントループを起動します。`tokio::main` はそのためのマクロです。

例えば次のコードを他の言語から来た人が見たら、出力される順番としてはbかc(順不同)で、5秒後にa、と思うかもしれません。

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let future_a = async {
        // 標準ライブラリの `sleep` を使っていることに注意
        std::thread::sleep(Duration::from_secs(5));
        println!("a")
    };
    let future_b = async { println!("b") };

    println!("c");

    future_a.await;
    future_b.await;
}
```

実際はc、5秒待ってa, bの順です。

また、Tokioのドキュメントにマクロ無しの `main` 関数を定義するコードの例があって、ランタイムと `Future` の関係の理解に役立つので紹介します:

```rust
use tokio::net::TcpListener;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::runtime::Runtime;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Create the runtime
    let rt  = Runtime::new()?;

    // Spawn the root task
    rt.block_on(async {
        let listener = TcpListener::bind("127.0.0.1:8080").await?;

        loop {
            let (mut socket, _) = listener.accept().await?;

            tokio::spawn(async move {
                let mut buf = [0; 1024];

                // In a loop, read data from the socket and write the data back.
                loop {
                    let n = match socket.read(&mut buf).await {
                        // socket closed
                        Ok(n) if n == 0 => return,
                        Ok(n) => n,
                        Err(e) => {
                            println!("failed to read from socket; err = {:?}", e);
                            return;
                        }
                    };

                    // Write the data back
                    if let Err(e) = socket.write_all(&buf[0..n]).await {
                        println!("failed to write to socket; err = {:?}", e);
                        return;
                    }
                }
            });
        }
    })
}
```

このようにRustの非同期処理の要点は非常にシンプルです。このシンプルなものを、上記のような問題を避けるために、`Waker` を用意したりIOイベントを使ったり専用のタイマーや専用の同期プリミティブ(`Mutex` など)を用意したりマクロを用意することで、洗練された非同期処理の仕組みにしています。この点を抑えるとasync bookをより速く理解できると思います。

### Pinning
Async bookはとても良いのですが、pinningのチャプターだけは頭に入りにくい気がするので以下の記事を読むことをおすすめします。

https://blog.cloudflare.com/pin-and-unpin-in-rust

https://fasterthanli.me/articles/pin-and-suffering

## Tokio
Tokioもドキュメントとチュートリアルが充実しているので、紹介することは少ないです。

https://tokio.rs/tokio/tutorial

https://docs.rs/tokio/latest/tokio/

抑えておく要点としては、上記で紹介したように非同期処理中のブロッキングな処理は避ける必要があることです。

- 標準ライブラリのIO系のモジュールを使うのではなく[Tokioが用意したIO系のモジュール](https://docs.rs/tokio/1.37.0/tokio/index.html#asynchronous-io)を使う
- 標準ライブラリの `Mutex` などの同期プリミティブを使うのではなく[tokio::sync](https://docs.rs/tokio/latest/tokio/sync/index.html)の同期プリミティブを使う
- 標準ライブラリの `sleep` などではなく[Tokioが用意したもの](https://docs.rs/tokio/latest/tokio/time/index.html)を使う
- その他でブロッキングな処理が必要な時は[Tokioが用意している仕組み](https://docs.rs/tokio/latest/tokio/task/index.html#blocking-and-yielding)を使う

こうすることで、今まではブロックしていた箇所で`await` (yield)して他のタスクを進めたり、そもそも別のスレッドで処理を進めたりできるようになります。

また、紹介しておくべきTokioランタイムの特性としてタスクスケジューリングがあります。Tokioがタスク(おおよそ `Future` に近い概念)をどのようなポリシーでスケジューリングするかがドキュメントで解説されています。必ずしも読む必要があるものではないですが、Rustの非同期処理とTokioランタイムの理解に役立つので読むのをおすすめします。

https://docs.rs/tokio/latest/tokio/runtime/index.html#detailed-runtime-behavior

## async-trait crate
動的ディスパッチをしたい場合、`async fn` のサポートがまだRustに来てないのでasync-traitがやるようなワークアウランドが必要です。詳しくはRust 1.75のリリース関連のブログ記事にあります。trait-variant crateでなんらかのサポートが入るっぽいです。

>We plan to provide utilities that enable dynamic dispatch in an upcoming version of the `trait-variant` crate.

https://blog.rust-lang.org/2023/12/21/async-fn-rpit-in-traits.html

https://docs.rs/async-trait/latest/async_trait/
