---
title: "はじめに"
---

この本は筆者がRustのアプリケーションを開発して得た知見を伝播すべくまとめた本です。

中で出てくるサンプルコードの大部分はOSSとして公開予定なアプリケーションの一部を適宜改変しながら引用しました。全体像も見たい時はリポジトリを参照してください。

- gls: https://github.com/Finatext/gls
  - リポジトリはパブリックですが、まだ内容は公にはしてないのでひっそりと公開しています。テックブログ記事を書く予定なのでご期待ください！
- orgu: (TODO)
  - リポジトリをパブリックにする予定ですが、まだです。GitHubのorganization-wide workflowを、AWS Lambdaやその他のプラットフォームで実現するOSSになっていて、開発したFinatextだけでなく他の組織でも役に立つソフトウェアだと思っています。こちらもテックブログ記事をご期待ください！(背水の陣)

がんばって本を書いたので、内容がおもしろい・役に立ちそうと思ったらぜひzenn上でのライクやXなどで共有してもらえるととてもうれしいです！引き続きおもしろいと思う情報発信をしていく予定なので、よければ筆者のXアカウントをフォローしてもらえるとうれしいです。

https://twitter.com/taiki45

執筆当時、筆者はFinatextという会社のプラットフォームチームで働いています。プラットフォームチームではツールの開発に主にGoと(最近)Rustを用いていておもしろい環境だと思います。会社自体もとてもいい会社なので、気になった方はぜひテックブログや採用情報を覗いてみてください！

https://techblog.finatext.com/

https://speakerdeck.com/finatext/finatext-are-hiring-engineers

## コーディング環境
rust-analyzerのサポートがあるとないとだと雲泥の差だと個人的には思ってるので、エディタのセットアップをおすすめします。

特に変数やメソッドチェーンの途中の式に対して型を表示してくれる[inlay-hints](https://rust-analyzer.github.io/manual.html#inlay-hints)という機能が学習途中には役に立ちすぎる(が好みはわかれる)のでおすすめです。

## Lint設定
個別に有効にしたいlintは `Cargo.toml` に設定を書けるようになったのでそこがおすすめです。

```toml
[lints.clippy]
absolute_paths = "warn"
# ...

# Enable all pedantic lints.
nursery = { level = "warn", priority = -1 }
```

https://github.com/rust-lang/cargo/blob/master/CHANGELOG.md#cargo-174-2023-11-16

Clippyのlint毎の設定は `clippy.toml` とかに書けます。

https://doc.rust-lang.org/clippy/configuration.html
