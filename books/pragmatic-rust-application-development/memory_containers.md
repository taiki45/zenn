---
title: "メモリコンテナ型"
---

この記事と図が便利です:

https://qiita.com/usagi/items/fc329895cebd3466910e

![Mmoery container cheat sheet](/images/pragmatic-rust/memory_container.png)

## 実世界のアプリケーションでよく使う型
どの型もそれなりにユースケースがあるんですが、実世界でよくあるアプリケーションコードに絞るとすれば、な解説です。

### Box
交換可能性のチャプターで紹介した動的ディスパッチを実現する時や、単にヒープに置きたい時に使います。

### Arc<T>
Webアプリケーションなど、複数のスレッド間でリードオンリーな値を共有したい時に使います。例えばなんらかのAPIクライアントになる構造体など。

`Arc` を使う・読む時の注意点は `Clone` とのメソッド名の競合でしょうか。`clone` を呼んでいるコードがぱっと見で `Clone::clone` なのか `Arc::clone` なのか分かりづらい問題で、Clippyの "clone_on_ref_ptr" lintが "unified function syntax" と呼ぶ記法で表記するとわかりやすいです。ちなみにこのlintはデフォルトではallowなので、個別に有効にする必要があります。

```rust
let some_value = Arc::new(SomeValue::new());
let s = Arc::clone(v);
other_function(s);
```

https://rust-lang.github.io/rust-clippy/master/#/clone_on_ref_ptr

https://doc.rust-lang.org/book/ch19-03-advanced-traits.html#fully-qualified-syntax-for-disambiguation-calling-methods-with-the-same-name

https://doc.rust-lang.org/reference/expressions/call-expr.html#disambiguating-function-calls

### Arc<Mutex<T>>
コネクションプールの実装を読む時とかに出てきます。Async Rustのチャプターで紹介したように、Tokioを使った非同期環境だとTokioが用意する同期プリミティブを使います。
