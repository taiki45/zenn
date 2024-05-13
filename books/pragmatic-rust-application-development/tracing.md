---
title: "tracing"
---

Rustのロギングとトレーシングのデファクトスタンダード(だと思っている) tracing crateを手早く使うためのチャプターです。tracing はドキュメントも豊富なんですが、まずcrateのドキュメント読む前の予備知識があった方が理解が速いです。

tracingエコシステム(?)は3つのcrateから成り立っています。

- tracing: アプリケーションコードの一部やライブラリといったトレーシングやロギングを行うレイヤーで使うためのライブラリ。言い換えるとトレーシングデータやログデータを生み出すレイヤーで使うためのライブラリです。`info!`, `debug!` みたいなマクロが用意されています。
- tracing-subscriber: アプリケーションコードのエントリーポイント付近で `Subscriber` と `Registry` を初期化したりするために使います。`Subscriber` は、上記で生み出されたトレーシングデータやログデータを受け取って、OpenTelemetry Collectorとみたいな外部のコレクターに送信したりstdoutに出力したりする役割を担っています。`Resistory` はspanを保管するストレージやspan IDの生成といった役割を担っています。
  - アプリケーション開発では基本的に `Subscriber` を使うだけですが、例えば自前のコレクターに送信する `Subscriber` を実装する時にも使えます。
- tracing-core: 上記2つの共通部分となるコアを提供するcrateです。

`Subscriber` はレイヤー構造を取れるようになっているのとトレーシングデータをフィルターするための機構も備えています。

## 概要
tracingのドキュメントが豊富なのは非常にありがたいんですが、機能も豊富で初見では迷子になりやすいのでアプリケーション開発で利用する定番の機能に絞って軽く紹介します。

### 初期化
まずはグローバルな `Subscriber` の初期化と設定です。

`EnvFilter` は `RUST_LOG` 環境変数でログレベルを細かく制御できて便利です。

https://docs.rs/tracing-subscriber/latest/tracing_subscriber/filter/struct.EnvFilter.html

また[clap_verbosity_flag](https://docs.rs/clap-verbosity-flag/latest/clap_verbosity_flag/index.html) crateもclapで `-v` みたいなフラグを受け取るCLIツールを実装するのに便利です。CLIツールでデバッグレベルをフラグで渡された時に、tracingの全ターゲットをデバッグレベルにしてしまうと外部のcrateのターゲットがうるさいので、自分のターゲットだけデバッグレベルにする小技を使っています(便利)。

`fmt::Subscriber` にはJSON以外に便利なフォーマットがあるので見てみてください。個人的には手元での確認用に `Pretty` が便利です。

https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/index.html#formatters

```rust
use clap_verbosity_flag::{LogLevel, Verbosity};
use tracing::{level_filters::LevelFilter, Level};
use tracing_log::AsTrace as _;
use tracing_subscriber::{
    fmt::{
        format::{DefaultFields, Format, Full},
        time::ChronoLocal,
        SubscriberBuilder,
    },
    util::SubscriberInitExt,
    EnvFilter,
};

// エントリーポイントでこれを呼び出す。
pub fn init_fmt_with_json<L: LogLevel>(v: &Verbosity<L>) {
    init_subscriber(v, |b| b.json());
}

pub fn init_fmt_with_pretty<L: LogLevel>(v: &Verbosity<L>) {
    init_subscriber(v, |b| b.pretty());
}

pub fn init_fmt_with_full<L: LogLevel>(v: &Verbosity<L>) {
    init_subscriber(v, |b| b.with_ansi(false));
}

type DefaultSubscriberBuilder =
    SubscriberBuilder<DefaultFields, Format<Full, ChronoLocal>, EnvFilter>;

fn init_subscriber<L, F, B>(v: &Verbosity<L>, f: F)
where
    L: LogLevel,
    F: FnOnce(DefaultSubscriberBuilder) -> B,
    B: SubscriberInitExt,
{
    // Don't set subscriber if user wants to silence output.
    match v.log_level_filter().as_trace() {
        LevelFilter::OFF => (),
        filter => {
            let env_filter = into_env_filter(filter);
            let builder = SubscriberBuilder::default()
                .with_timer(ChronoLocal::rfc_3339())
                .with_env_filter(env_filter);
            f(builder).init();
        }
    }
}

fn into_env_filter(filter: LevelFilter) -> EnvFilter {
    // If log level is lower than debug, only apply it to orgu targets.
    let default = if filter >= Level::DEBUG {
        format!("info,orgu={filter}")
    } else {
        filter.to_string()
    };
    EnvFilter::try_from_default_env().unwrap_or_else(|_| default.into())
}
```

### Spanの作成
基本的には `#[instrument]` マクロを使うと思います。

https://docs.rs/tracing-attributes/latest/tracing_attributes/attr.instrument.html

以前のチャプターであった「GitHubのwebhookイベントを処理するようなハンドラー」での例です:

```rust
#[instrument(
    skip(self, req),
    fields(
        request_id = req.request_id,
        delivery_id = req.delivery_id,
        owner = req.repository.owner.login, repo = req.repository.name,
        head_sha = req.head_sha, pull_request_number = req.pull_request_number.unwrap_or_default(),
    ),
)]
pub async fn handle_event(&self, req: &CheckRequest) -> Result<()> {
    // ...
}
```

Spanのフィールドを後から埋めたい場合は `Empty` を指定しておくと実現できます。Webアプリケーションのリクエストハンドラーなどで便利です。

```rust
#[instrument(
    skip_all,
    fields(
        delivery_id = Empty,
        event_name = Empty,
        action = Empty,
        owner = Empty,
        repo = Empty
    )
)]
pub async fn webhook<EB, GH, V>(
    headers: HeaderMap,
    State(state): State<Arc<AppState<EB, GH>>>,
    body: String,
) -> Result<impl IntoResponse, AppError>
where
    EB: EventQueueClient,
    GH: GithubClient,
    V: GithubRequestVerifier,
{
    // ...
    let delivery_id = get_header_str(&headers, "x-github-delivery")?;
    Span::current().record("delivery_id", delivery_id);
    let event_name = get_header_str(&headers, "x-github-event")?;
    Span::current().record("event_name", event_name);
    // ...
}
```

### Eventの作成
ロギングライブラリでの `info!` みたいなログレコードの作成は、tracingではeventの作成と呼ばれています。ログレコードと違ってeventはspanの中で発生するからみたいです。Eventの作成についてはすぐに慣れると思うので軽めに紹介します。

`?` 記法は `fmt::Debug` を使う、`%` 記法は `fmt::Display` を使う、だけ抑えておけば他の記法はすぐに手に馴染むと思います。

```rust
info!(duration = %d, "checkout timed out");
```

https://docs.rs/tracing/latest/tracing/#recording-fields

## Async Rustでの注意
Spanの作成を手でやる時にdrop guard (例 `let _enter = span.enter();`)が `await` (yieldされるポイント)をまたぐ時に注意が必要です。Async Rustのチャプターで見たように、タスクの実行は `await` を起点にいろんなタスクに行ったり来たりするからです。

https://docs.rs/tracing/latest/tracing/struct.Span.html#in-asynchronous-code

`#[instrument]` マクロを使う以外だと、`Future` を拡張する `Instrument` traitが便利です。

https://docs.rs/tracing/latest/tracing/trait.Instrument.html

```rust
let mut cmd = self.build_command(&cloned.path, req, &token)?;
let span =
    info_span!("run command", command = fmt_cmd(&cmd), path = %cloned.path.display());
// To instrument the command execution, wrap with async block.
async move {
    info!("running command");
    let out = cmd
        .output()
        .await
        .with_context(|| format!("failed to run command: {}", fmt_cmd(&cmd)))?;
    let stdout = String::from_utf8_lossy(&out.stdout);
    let stderr = String::from_utf8_lossy(&out.stderr);

    if out.status.success() {
        self.report_success(owner, repo, &check_run, input, &cmd, &stdout, &stderr)
            .await?;
    } else {
        self.report_failure(owner, repo, &check_run, input, &cmd, &stdout, &stderr, &out)
            .await?;
    }

    anyhow::Ok::<()>(())
}
.instrument(span)
.await?;
```

## 他の `Subscriber`
ここにリストがあります。`Subscriber` 以外にも他のcrateとのインテグレーションを提供するようなcrateもあります。

https://docs.rs/tracing/latest/tracing/index.html#related-crates

## 参考資料
https://tokio.rs/tokio/topics/tracing

https://docs.rs/tracing/latest/tracing/

https://docs.rs/tracing-subscriber/latest/tracing_subscriber/index.html
