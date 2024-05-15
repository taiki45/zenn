---
title: "完全理解: AWS Lambda with Go and Rust"
type: "tech"
topics: ["aws", "lambda", "go", "rust"]
emoji: "🦀"
published: true
---

# 完全理解: AWS Lambda with Go and Rust

AWS Lambda functionを書いていて、「だいたいの仕組みはわかったけどなんか下層の解像度が低くてしっくりこないなぁ」と思ったことはないですか？ この記事では、AWS Lambdaのそんなちょっと下側の部分について「完全に理解した」となるような解説をしていきます。AWS Lambda functionを書くためのランタイムライブラリについてもGo版とRust版のそれぞれについて解説をします。

注: 「完全に理解した」とはインターネット上のミームの方で、真に完全に理解するような深堀り・網羅性はありません。

## AWS Lambdaの仕組み
ここではGoやRustでよく使われるコンテナイメージを使うデプロイメント方式に絞って解説していきます。

![Architecture diagram of the execution environment](/images/aws-lambda-with-go-and-rust/image.png)

From: https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html

[公式ドキュメント](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html)にあるように、AWS Lambdaは「Lambdaサービス」と「実際のLambda function」の間にRuntime APIと呼ばれるプロトコル(仕様?)が挟まっています。LambdaサービスはEventBridgeやSQSやAPI Gatewayのようなイベントソースとなるサービスからのイベントを受け取ってLambda functionとの仲介を果たす役割を担っています。このRuntime APIが「どのようにしてLambda functionを呼び出すか」を決定づけているので、このRuntime APIを詳しく見ていきます。

余談ですが、Runtime API周りのドキュメントを見ると、Runtime APIプロキシーなどについても言及されていて独自の拡張の可能性に気づけるので一読の価値があります。

![Runtime API overview](/images/aws-lambda-with-go-and-rust/image-1.png)

まずLambda functionが起動するとmain関数等を経てaws-lambda-goやaws-lambda-rust-runtimeのようなランタイムライブラリのイベントループが起動します。

このイベントループでは、まずLambdaサービスの `/runtime/invocation/next` に対してHTTP GETリクエストを送ります。ここで、イベントソースからイベントがまだ届いてなかったらそのままロングポーリングで待ちます。

イベントソースからイベントがすでに届いていたら、もしくはイベントソースからイベントが届いたら、Lambdaサービスは先ほどのHTTP GETリクエストに対してレスポンスを返します。ロングポーリングで待ち状態になっていたランタイムライブラリのイベントループで処理が始まって、Lambdaサービスから返ってきたレスポンスの処理を開始します。イベントループでは基本的にLambdaサービスから返ってきたレスポンスのでシリアライゼーションを行い、イベントソースのサービス毎に異なる構造の構造体にイベントソースからのデータを詰めて、ランタイムライブラリのユーザーが定義した関数に渡しつつその関数を実行します。

ランタイムライブラリのユーザーが定義した関数の処理が終わったら、ランタイムライブラリはその処理結果をLambdaサービスに戻します。先のLambdaサービスからのレスポンスには "AWD Request ID" なるユニークな値がレスポンスヘッダー経由で届いていたので、ランタイムライブラリはその値を使って、Lambdaサービスの `/runtime/invocation/<aws_request_id>/response` エンドポイントに対してHTTP POSTリクエストを送ります。

つまり、イベントソースからLambdaサービスにイベントが来る→ランタイムライブラリがイベントを取得する→ユーザー定義関数が呼ばれる、この繰り返しでLambda functionは実行されています。そしてその中ではRuntime APIというover HTTPなプロトコルが仲介しています。


### 余談: イベント処理の多重化
ここで1つ興味深いのが、ランタイムライブラリのイベントループでは多重化(multiplexing)をしない実装になっているライフタイムライブラリが標準的なことです。上記フローを見てすぐに思いつくのは、AWS Requset IDを使ってLambda functionの処理実行結果をLambdaサービスに渡すので、ランタイムライブラリはLambdaサービスからレスポンスを受け取ってユーザー定義の関数を実行しつつ、その実行終了を待たずに多重してまた次のイベントをLambdaサービスから受け取れば効率的でよさそうに見えます。これはGoであればgoroutine、Rustであればマルチスレッド非同期処理ランタイムを使えば容易に実現できそうです。

しかし、ライフタイムライブラリの実装を見ても、AWS Lambda with Goの公式ドキュメントに書かれている通り、多重化(or 並列)してイベントを処理する実装にはなっていません。

> A single instance of your Lambda function will never handle multiple events simultaneously.

https://docs.aws.amazon.com/lambda/latest/dg/golang-handler.html#golang-handler-state

ドキュメント等に(おそらく)書いてないので、ここからは完全に推測ですが以下のような理由がありそうな気がします。

- ユーザー定義関数のうっかり実装の対策のため
  - Goであればrace conditionが起きるコードを簡単に書いてしまえる
  - Rustではスレッドをブロックしてしまうユーザー定義関数を実行する時に想定していたよりパフォーマンスが出ない(それで困る?)
- 課金単位としてfunction 1実行あたりの時間とメモリ使用量を計測しているので、同じプロセス内で複数の実行を重ねたくない
- 実行時間あたりで課金される都合上、限られた計算資源内で多重化するより、Lambdaサービスのイベントキューを詰まらせてLambdaサービスにオートスケールしてもらった方がユーザー的にはお得だからシリアルに実行するランタイムライブラリで十分

課金計測の都合が一番ありそうな気はしますが完全に余談です。

## Go: aws-lambda-go
Goのランタイムライブラリである[aws-lambda-go](https://github.com/aws/aws-lambda-go)は、次のようにユーザー定義関数を定義してランタイムライブラリに渡して実行することができます。これがどういう仕組みで実現されているのか見ていきます。

```go
package main

import (
	"context"
	"fmt"

	"github.com/aws/aws-lambda-go/events"
	"github.com/aws/aws-lambda-go/lambda"
)

func handleRequest(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
	fmt.Printf("Processing request data for request %s.\n", request.RequestContext.RequestID)
	fmt.Printf("Body size = %d.\n", len(request.Body))

	fmt.Println("Headers:")
	for key, value := range request.Headers {
		fmt.Printf("    %s: %s\n", key, value)
	}

	return events.APIGatewayProxyResponse{Body: request.Body, StatusCode: 200}, nil
}

func main() {
	lambda.Start(handleRequest)
}
```

まず、ランタイムライブラリのエントリーポイントとなる `lambda.Start` ですが、多様な引数の関数を実行することができます。[ドキュメント](https://pkg.go.dev/github.com/aws/aws-lambda-go/lambda#Start)によると、次のようなルールを満たす値なら渡せるようです:

- ハンドラとして渡される値は関数であること
- ハンドラは0〜2つの引数を取ること
- 2つの引数を取るなら、最初の引数は `context.Context` インターフェイスを満たすこと
- ハンドラは返り値として0~2の値を返すこと
- 2つの値を返すなら、2つ目の引数は `error` インターフェイスを満たすこと
- 1つだけ値を返すなら、`error` インターフェイスを満たすこと

かなり動的な性質を持っていて興味深いです。

まとめると、以下の引数の形を取る関数は渡すことできます。

```go
func ()
func (TIn)
func () error
func (TIn) error
func () (TOut, error)
func (TIn) (TOut, error)
func (context.Context)
func (context.Context) error
func (context.Context) (TOut, error)
func (context.Context, TIn)
func (context.Context, TIn) error
func (context.Context, TIn) (TOut, error)
```

余談ですが、ドキュメントに(たぶん)書いてない仕様としては、

```go
Invoke(context.Context, []byte) ([]byte, error)
```

というインターフェイスを持つ構造体も後方互換性のために渡せるようです。

次に `lambda.Start` がどのようにしてイベントデータをユーザー定義関数に渡しているのかを見ていきます。

`lambda.Start` の実体は `StartWithOptions` でその中で `newHandler` という関数でユーザー定義関数をラップしています。

```go
func StartWithOptions(handler interface{}, options ...Option) {
	start(newHandler(handler, options...))
}
```

`newHandler` の中で、最終的には `reflectHandler` という関数でユーザー定義の関数がラップされていて、その中で作る関数が処理の実体です。reflectパッケージを使いつつイベントデータをユーザーがほしい型にデシリアライズするのを試して、成功したらデシリアライズした構造体を渡してユーザー定義関数を呼んでいます。

```go
return func(ctx context.Context, payload []byte) (io.Reader, error) {
  out.Reset()
  in := bytes.NewBuffer(payload)
  decoder := json.NewDecoder(in)
  // ...

  // construct arguments
  var args []reflect.Value
  if takesContext {
    args = append(args, reflect.ValueOf(ctx))
  }
  if (handlerType.NumIn() == 1 && !takesContext) || handlerType.NumIn() == 2 {
    eventType := handlerType.In(handlerType.NumIn() - 1)
    event := reflect.New(eventType)
    if err := decoder.Decode(event.Interface()); err != nil {
      return nil, err
    }
    if nil != trace.RequestEvent {
      trace.RequestEvent(ctx, event.Elem().Interface())
    }
    args = append(args, event.Elem())
  }

  response := handler.Call(args)
```

ちなみに前出のハンドラとして渡す関数が満たすべきルールの大部分は `handlerTakesContext` と `validateReturns` 関数で検証されています。

Goの場合はreflectパッケージを使って動的なメタプログラミングができるのであまり驚きはなかったかもしれません。筆者の場合は、ハンドラとして渡せる関数の引数のバリエーションを知らなかったので、それが知れておもしろかったです。

### lambdaurlパッケージ
aws-lambda-goには[lambdaurl](https://github.com/aws/aws-lambda-go/tree/v1.47.0/lambdaurl)というパッケージがあって、それを使うとnet/httpパッケージの `http.Handler` をaws-lambda-goの期待するハンドラに変換(ラップ)することができます。このlambdaurlパッケージ自体はインターネット上で言及があまりないです。

[Echo](https://github.com/labstack/echo)などのwebフレームワークをライフタイムライブラリの期待するハンドラに変換する方法とかはインターネット上でも見るので、そのアプローチを使うとルーティングやミドルウェアレイヤーの機能を手に入れたり、手元での動作確認がしやすくできたりするのでよさそうです。

## Rust: aws-lambda-rust-runtime
![alt text](/images/aws-lambda-with-go-and-rust/image-4.png)

[aws-lambda-rust-runtime](https://github.com/awslabs/aws-lambda-rust-runtime)は[tower crate](https://docs.rs/tower/latest/tower/)の `Service` を使った抽象を行っていて、`Service` を実装してるものなら(その他のtrait boundsの範囲内で)なんでも受け取って実行するようになっています。towerの `service_fn` 関数を使って `Serivce` を作るのがよくあるコードだと思います。そして、以下のように `service_fn` 渡す関数・クロージャの引数で[aws_lambda_events crate](https://docs.rs/aws_lambda_events/latest/aws_lambda_events/)にあるイベント構造を定義している型を指定すると、イベントソースからのデータがデシリアライズされた状態で関数が実行されます。

```rust
use aws_lambda_events::apigw::{ApiGatewayProxyRequest, ApiGatewayProxyResponse};
use http::HeaderMap;
use lambda_runtime::{service_fn, Error, LambdaEvent};

async fn handler(
    event: LambdaEvent<ApiGatewayProxyRequest>,
) -> Result<ApiGatewayProxyResponse, Error> {
    let mut headers = HeaderMap::new();
    headers.insert("content-type", "text/html".parse().unwrap());
    let resp = ApiGatewayProxyResponse {
        status_code: 200,
        multi_value_headers: headers.clone(),
        is_base64_encoded: Some(false),
        body: Some("Hello AWS Lambda HTTP request".into()),
        headers,
    };
    Ok(resp)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    lambda_runtime::run(service_fn(handler)).await
}
```

HTTPイベント場合は[lambda_http crate](https://docs.rs/lambda_http/latest/lambda_http/)を使うと、[http crate](https://docs.rs/http/latest/http/)の `Request` がユーザー定義関数に渡されて実行されます。

```rust
use lambda_http::{service_fn, Error, IntoResponse, Request, RequestExt, Response};

async fn handler(event: Request) -> Result<impl IntoResponse, Error> {
    let resp = Response::builder()
        .status(200)
        .header("content-type", "text/html")
        .body("Hello AWS Lambda HTTP request")
        .map_err(Box::new)?;
    Ok(resp)
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    lambda_http::run(service_fn(handler)).await
}
```

後発なこともあり、よく考えられたインターフェイスになっています。一見するとマジカルなこの設計がどうやって実現されているのかを見ていきます。

ランタイムライブラリのイベントループは `Service` の `poll_ready` と `call` を呼んでいるのですが、そこに至るまでにユーザー定義関数(`tower::Service`)が何層かにラップされています。`CatchPanicService` なんて便利なラップもおもしろいですが、ここでは `RuntimeApiResponseService` に注目します。

![alt text](/images/aws-lambda-with-go-and-rust/image-2.png)

まず、ライブラリユーザーが使う `lambda_runtime::run` は最終的には `Runtime::run_with_incoming` が本体です。ここにランタイムライブラリのイベントループがあります。ここでの `service` はおおよそユーザー定義の関数(`tower::Service`)と同等です。ユーザー定義の関数をイベントデータとイベントコンテキストが入った `LambdaInvocation` を渡しつつ呼んでいます。

```rust
pub(crate) async fn run_with_incoming(
    mut service: S,
    config: Arc<Config>,
    incoming: impl Stream<Item = Result<http::Response<hyper::body::Incoming>, BoxError>> + Send,
) -> Result<(), BoxError> {
    tokio::pin!(incoming);
    while let Some(next_event_response) = incoming.next().await {
        trace!("New event arrived (run loop)");
        let event = next_event_response?;
        let (parts, incoming) = event.into_parts();

        // ...

        // Build the invocation such that it can be sent to the service right away
        // when it is ready
        let body = incoming.collect().await?.to_bytes();
        let context = Context::new(invoke_request_id(&parts.headers)?, config.clone(), &parts.headers)?;
        let invocation = LambdaInvocation { parts, body, context };

        // Setup Amazon's default tracing data
        amzn_trace_env(&invocation.context);

        // Wait for service to be ready
        let ready = service.ready().await?;

        // Once ready, call the service which will respond to the Lambda runtime API
        ready.call(invocation).await?;
    }
    Ok(())
}
```

そして肝心の `RuntimeApiResponseService::call` 内で、先ほどの `LambdaInvocation` 型を `EventPayload` 型にイベントデータをデシリアライズしています。`EventPayload` 型は `EventPayload: for<'de> Deserialize<'de>` な制約を持ったタイプパラメーターで、つまり「デシリアライズできるなにか」です。`service_fn` に渡す関数・クロージャの引数の具体的な型がここに渡ってきていて、イベントデータがデシリアライズされて、ユーザー定義の関数が実行されるようになっています。

```rust
fn call(&mut self, req: LambdaInvocation) -> Self::Future {
    // ...

    let request_id = req.context.request_id.clone();
    let lambda_event = match deserializer::deserialize::<EventPayload>(&req.body, req.context) {
        Ok(lambda_event) => lambda_event,
        Err(err) => match build_event_error_request(&request_id, err) {
            Ok(request) => return RuntimeApiResponseFuture::Ready(Some(Ok(request))),
            Err(err) => {
                error!(error = ?err, "failed to build error response for Lambda Runtime API");
                return RuntimeApiResponseFuture::Ready(Some(Err(err)));
            }
        },
    };

    // Once the handler input has been generated successfully, the
    let fut = self.inner.call(lambda_event);
    RuntimeApiResponseFuture::Future(fut, request_id, PhantomData)
}
```

もう一つのlambda_http版も見ていきます。

`Adapter` という型に変換を任せてあとは先述の `lambda_runtime::run` を呼ぶだけです。

![alt text](/images/aws-lambda-with-go-and-rust/image-3.png)

まずは `lambda_http::run` から。`Adapter` を作って `lambda_runtime::run` を呼ぶだけです。

```rust
pub async fn run<'a, R, S, E>(handler: S) -> Result<(), Error>
where
    S: Service<Request, Response = R, Error = E>,
    S::Future: Send + 'a,
    R: IntoResponse,
    E: std::fmt::Debug + std::fmt::Display,
{
    lambda_runtime::run(Adapter::from(handler)).await
}
```

`Adapter` も `tower::Service` を実装していて、その `call` が処理の実体です。`lambda_rntime::run` が想定する引数を指定して、イベントデータを `LambdaRequest` にデシリアライズしてもらい、`http::Requst` を作っています。`http::Request` には `Extensions` という仕組みがあり、そこにイベントデータのLambda独自となる部分を格納してユーザー定義関数に渡しています。

```rust
fn call(&mut self, req: LambdaEvent<LambdaRequest>) -> Self::Future {
    let request_origin = req.payload.request_origin();
    let event: Request = req.payload.into();
    let fut = Box::pin(self.service.call(event.with_lambda_context(req.context)));

    TransformResponse::Request(request_origin, fut)
}
```

https://docs.rs/http/latest/http/request/struct.Request.html#method.extensions

## おわりに
一見不思議に思えていたAWS Lambdaとランタイムライブラリの動作ですが、ドキュメントと実装を読み解いていくともはや自明に思えてきたのではないでしょうか。この記事が読者の方の今後の良いLambda生活の一助となればうれしいです。

最後に宣伝です:

- [「実用Rustアプリケーション開発」](https://zenn.dev/taiki45/books/pragmatic-rust-application-development)というタイトルでzenn本を書きました！実世界のRustアプリケーションを素早く良く開発するための実用的な知見集、という内容になってます。
- [Platform Engineering Kaigi 2024](https://www.cnia.io/pek2024/)というイベントで登壇する予定です。プラットフォームエンジニアリングの領域の技術的な取り組みについて喋る予定なので、興味があればぜひ聞きに来てください。GitHubのorganization-wide workflowを実現するOSSの話などおもしろいと思います！
- 引き続きおもしろいと思う情報発信をしていく予定なので、よければ筆者のXアカウントをフォローしてもらえるとうれしいです！
  - https://twitter.com/taiki45

そしてこの記事はFinatextという会社の仕事の中で執筆しました。会社のバックエンドはGoで、プラットフォームチームはGoと最近Rustを用いて開発しています。とてもおもしろい会社なのでぜひ採用情報を見てみてください！

https://speakerdeck.com/finatext/finatext-are-hiring-engineers
