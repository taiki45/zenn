---
title: "clap"
---

clapもドキュメントが充実しててCLIツールを見れば良くて利用例も見やすいのであまり紹介するものはないです。deriveのチュートリアルを読み進めつつstructドキュメントを必要に応じて読んでいくとだいたい使えると思います。

https://docs.rs/clap/latest/clap/_derive/_tutorial/chapter_0/index.html

「これだけは先に知っておきたい」ものを軽めに紹介します。

## `rename_all`
`ValueEnum` を別のケース記法で扱いたい時に便利です。

```rust
#[derive(Debug, Clone, ValueEnum, Display)]
#[clap(rename_all = "snake_case")]
#[strum(serialize_all = "snake_case")]
enum EventType {
    PullRequest,
    CheckSuite,
}
```

## `hide_env`
秘匿値が `--help` などで出力されないようになります。

```rust
#[derive(Debug, Args, Clone)]
pub struct GithubAppConfig {
    // ...

    /// GitHub App private key.
    #[arg(env = "GITHUB_PRIVATE_KEY", hide_env_values = true, long)]
    pub private_key: String,
}
```

## `flatten`
いくつかのコマンドで共通のCLI引数を共通化できて、その構造体をconfigとして別の構造体に渡せて便利です。

例えばGitHubクライアントのタイムアウトなどの設定という共通引数がある時に、それを定義した構造体を `flatten` で指定しておいて、エントリーポイントでその構造体をGitHubクライアントの初期化関数に渡す、みたいなユースケースです。

```rust
#[derive(Debug, Args, Clone)]
pub struct GithubApiConfig {
    /// Connect timeout for GitHub API requests.
    #[arg(env, long, default_value = "1s")]
    pub github_connect_timeout: humantime::Duration,
    // ...
}

#[derive(Debug, Clone, Args)]
pub struct ServerArgs {
    #[command(flatten)]
    github_config: GithubApiConfig,
    // ...
}

pub async fn server(args: ServerArgs) -> Result<()> {
    let client = OctorustClient::new(args.github_config.clone(), args.github_app_config.clone())?;
    // ...
}
```

## テスト
clapを使っていて、リファクタリングとかで気づかない内にpanicするコードを書いてしまうことがあるので、下のようなテストを1つ埋めておくと安心です(おそらく)。

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn verify_cli() {
        use clap::CommandFactory;
        Cli::command().debug_assert()
    }
}
```

https://docs.rs/clap/latest/clap/struct.Command.html#method.debug_assert
