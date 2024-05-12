---
title: "補足: リリース"
---

いくつかのRustツールのプロジェクトを調査してみたのですが、Rustツールのバイナリのビルド・リリース・配布のツールのデファクトは今はないっぽいです。なので筆者は今は下記のようなGitHub Actions workflowを用意して使っています。

https://github.com/Finatext/gls/blob/main/.github/workflows/cicd.yml

`taiki-e/upload-rust-binary-action` とその周辺ツールがよさそうなので乗り換えを検討しています。

https://github.com/taiki-e/upload-rust-binary-action
