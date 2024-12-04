---
title: "モノリポでGitHub Actionsで効率よくCI/CDや自動化を実装するTips 3選"
emoji: "🔨"
type: "tech"
topics: ["github", "githubactions"]
published: true
published_at: 2024-12-09 09:00
publication_name: finatext
---

この記事は[Finatextグループ Advent Calendar 2024](https://qiita.com/advent-calendar/2024/finatextgroup)の12/9の記事です。

---

モノリポでGitHub Actionsを使ってCI/CDを構築する際に直面する問題について、インターネットであまり見つからなかったtips 3選を紹介します。

- Matrix strategyを使ったパイプライン並列化
- Required status checks
- Dependabotの完全自動マージ

## Matrix Strategyを使ったパイプライン並列化
### Matrix Strategyを動的生成を利用する
Finatextのプラットフォームチームでは社内プラットフォームを担うアプリケーション群を領域や性質に合わせて複数のAWSアカウントに区切り運用しています。それぞれのAWSアカウントの中でモニタリングや自動化などの各種タスクのために複数のAWS Lambdaのfunctionを実行しています。それぞれのLambda functionのソースコードはあるリポジトリの中でモノリポとして管理しています。

GitHub ActionsでLambda functionをデプロイするのざっくり以下のような処理が必要でした。

- 変更されたアプリケーションの検知
- 変更されたアプリケーション毎に
  - 対応するAWSアカウントのクレデンシャル作成 ([aws-actions/amazon-ecr-login](https://github.com/aws-actions/amazon-ecr-login)を使う)
  - コンテナイメージのビルド・プッシュ ([docker/build-push-action](https://github.com/docker/build-push-action)を使う)
  - Lambda functionのデプロイ (in-houseのreusable workflowを使う)

こういうactionのinputsに異なる値を渡すような繰り返し処理を実装するのに、GitHub Actionsのワークフローファイルを異なる値毎に生成するのはアプローチの1つとしてあります。今回のケースで言うとAWSアカウント・イメージのビルド対象・Lambda function辺りが変数でした。

実際当初はファイルの生成アプローチと外部actionを諦めて手で実装する方向で実装されていたのですが、これはGitHub Actionsのmatrix strategyを使うと効率よく実装・実行できたので紹介します。Matrix strategyのよくある使い方としては静的な値をワークフローファイルに書いておいて、その値をもとにジョブをファンアウトさせると思います。静的な値の他に、matrix strategyは手前のジョブのoutputsを参照してファンアウトもできるのでそれを利用します。

例えば、差分を検知するようなジョブの中で次のようなJSONを生成してジョブのoutputsとして出力します。

```json
[
  {
    "account_name": "A",
    "account_id": "1"
  },
  {
    "account_name": "B",
    "account_id": "2"
  }
]
```

そしてジョブのmatrix strategy定義でさっきのoutputsを使います。

```yaml
    strategy:
      matrix:
        target: ${{ fromJson(needs.collect.outputs.targets) }}
```

こうすることで、リストの要素それぞれに対応するジョブが実行され、そのジョブの中で `${{ matrix.target.account_id }}` などで値を利用できるので、以下のように外部のactionに渡すinputsを変えることができます。

```yaml
       uses: aws-actions/configure-aws-credentials@e3dd6a429d7300a6a4c196c26e071d42e0343502 # v4.0.2
        with:
          role-to-assume: arn:aws:iam::${{ matrix.target.account_id }}:role/xxx
          aws-region: ap-northeast-1
```

以前はループ処理でビルド・デプロイを行っていたのですが、この実装方法に変えることで自然とパイプライン並列化するので多くのアプリケーションをデプロイするのが少し速くなったのもうれしいポイントです。

### Matrix strategyの生成
Matrix strategyを使って実装を効率化できることはわかったので次はどうやってこのstrategyを生成するかです。単純なstrategyで済めばよかったのですが、このモノレポでは以下のような特性がありました。

- 過去使用していたツールの歴史的経緯でコードのディレクトリ構造が違うアプリケーションがある
- コンテナイメージだけ更新するアプリケーション、自動デプロイしたくないアプリケーション、パブリックな別のリポジトリでコンテナイメージをビルドしているアプリケーションが存在する

このような複数のメタデータ的なものを扱うならシェルスクリプトでがんばるより、いっそ簡単なCLIアプリケーションを作って任せた方がよさげでは？と考えて小さなCLIアプリケーションを実装しました(社内向けコンテキストになるのですが、container/kagoと呼んでいます)。最初の責務としては、次の通り。

- メタデータの管理
- 異なるGitHub Actionsのeventに対応したmatrix strategyの生成

メタデータの管理についてはアプリケーションディレクトリにYAMLファイルで管理する方法を採用しました。そこで「どのデプロイタイプか」「デプロイタイプ毎のメタデータ(GitHub Container Registryのpackage情報など)」などを管理します。

Matrix strategyの生成については、`on: push` の場合は差分を計算してデプロイすべきアプリケーションを決定してからメタデータを収集してそれらの情報からmatrix strategyを生成します。手動デプロイ用に `on: workflow_dispatch` もあるのでそのケースではinputsから生成します。

デプロイ種別毎の分岐については、今はmatrix strategyにデプロイ種別を値として含んでいてworkflowのstepの中で条件分岐しています。複雑になってきたらcomposit actionに切り出すと見通しがよくなりそうと考えています。

### CLIアプリケーションのデプロイ
この小さいCLIアプリケーションを労力なくGitHub Actions上で動かす方法について少し悩んだのですが、ジョブの中でCLIアプリケーションをビルドしておいてGitHub Actionsのcacheにバイナリを保管するのが手頃で便利でした。

### プレビュージョブ
Matrix strategyの生成やデプロイ用のメタデータの管理をCLIアプリケーションとして実装したので、pull requestに対してデプロイ予定のアプリケーション一覧を表示するジョブとかもちょっとした労力で実装できてけっこう便利だったのでおすすめです。

## Required Status Checks
モノリポでGitHub ActionsでCI/CDする時の悩みポイントとしてありがちなrequired status checksをいい感じにするのも工夫ポイントがあったので紹介します。

### 課題
モノリポで「そのPRで関心のあるアプリケーションのみCIの成否を見てマージ可否を判定したい、かつ関心のあるアプリケーションのみCIを実行したい」という問題を回避するために、同一名のジョブを用意してrequired status checksにそのジョブ文字列を指定するという回避方法はインターネット上でもよく見つけられます。この問題については例えば次のような記事がわかりやすいと思います。簡単にまとめると、CIジョブの実行を効率化するためにGitHub Actionsの `paths` などのフィルターを使ってジョブの起動を制御した場合に、どのジョブをrequired status checksとして採用するかが難しいという問題があるが、それは「同一名のジョブを複数用意してrequired status checksにそのジョブ名を設定し、pull requiestなどでは同一名のジョブのいずれかが起動するようにフィルターを設定する」ことで回避可能という話です。

https://zenn.dev/bigwheel/articles/05accc6323de18

今回のCI/CD整備ではmatrix strategyを利用しているので、CIジョブ名はstrategyの値を使って自動生成される文字列になるので上記のアプローチは使えません。

### 解決策
解決策として、各アプリケーションのCIジョブの終了を待ち受けるジョブを後続に実行して、その中で1つでも失敗していたら全体を失敗にする方法で解決しました。

![CI job workflow](/images/monorepo-github-actions/ci_job.png)

- `result` の乗算的な振る舞い
- スキップとキャンセルを考慮する

`needs` を使って各アプリケーションのCIジョブを待ち受けて、ジョブの結果はneeds contextのresult (`needs.<job_id>.result`) を使って参照するのですが、上述のようにmatrix strategyを使っている時にはresultは乗算的な振る舞いをする値が入ることです。待ち受けるジョブ(dependent jobs)が1つでも失敗したら `failure`、1つでもキャンセルされたら `cancelled` などです。

スキップについては、GitHub Actionsのneedsを使ったジョブチェインは待ち受けるジョブが1つでもスキップしたらそのジョブ自体もスキップする仕様になっています。なので、結果判定のジョブは待ち受けるジョブがスキップやキャンセルしても実行されるようにして、そのようなケースではそのジョブを失敗させる必要があります。このような用途にstatus check functionsの1つである `always()` が使えます。

```yaml
    if: ${{ always() }}
```

関連ドキュメント:

https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/using-jobs-in-a-workflow#defining-prerequisite-jobs

https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/evaluate-expressions-in-workflows-and-actions#status-check-functions

### 空コミットPR問題
また、今回のリポジトリではLambda functionのソースコード以外にも、(Lambda functionとは関係ない)AWS全体の権限周りなどの管理をしているTerraformのconfiguration file (`.tf` ファイル)も含んだリポジトリで、プラットフォームチーム以外の開発者によるTerraform関係のファイルの変更の方が割合が多いという状況でした。Terraform関係のpull requestのケースのために、先述の同一文字列ジョブ名を付けたダミージョブを用意することでrequired status checksを回避する策も利用しています。

ここで問題になったのが、Terraform + Atlantisリポジトリであるあるなapply忘れ・漏れのapplyのための空コミットpull requestでrequired status checksをどう解決するかでした。GitHub Actionsのpathsフィルターを利用すると空コミットではなにもジョブは起動しないので、常に成功するダミージョブも成功しません。

手頃かつ課金的にもローコストな解決策として、 `if: ${{ github.event.pull_request.changed_files == 0 }}` でなにも変更がない時にのみ起動してpull requestにガイドコメントをするジョブを用意することで回避しました。ガイドコメントでは「なにか無関係な変更をpull requestに入れてapply後にpull requestをマージではなくクローズする」と案内しています。

```yaml
# Show guide message on empty changed files PRs.
name: Container CI Skip Guide
on:
  pull_request

jobs:
  guide:
    runs-on: ubuntu-latest
    if: ${{ github.event.pull_request.changed_files == 0 }}
    permissions:
      contents: read
      pull-requests: write
    steps:
      - env:
          GH_TOKEN: ${{ github.token }}
          PR_URL: ${{ github.event.pull_request.html_url }}
        run: |
          # Comment guide message on PR.
          gh pr comment "${PR_URL}" --body 'Empty PR detected. "CI Result" job will not run due to GitHub limitation. Please add any changes to the PR. After applying with Atlantis, just close this PR instead of merging.'
```

## Dependabotの完全自動マージ
アプリケーションの依存パッケージの更新にDependabotを利用していて、今回2つ工夫した点があるので紹介します。

- Required review count設定下で完全な自動マージの実現
- 一斉更新を避けるためのスケジューリングの実装

### Required Reviews
Finatextではtwo-person integrityの実現のためにGitHubのrequired reviews countを活用していて、このリポジトリもその対象です。この設定がない場合はGitHubのドキュメントにあるように簡単にDependabotが作成するpull requestの自動マージが実現できます。

https://docs.github.com/en/code-security/dependabot/working-with-dependabot/automating-dependabot-with-github-actions

今回のリポジトリでは自動でデプロイしたくないような厳密な管理をしたいアプリケーションは自動デプロイの対象から外していることもあり、Dependabotのpull requestは基本的に人の手を介さずにマージからデプロイまで自動化したいです。

Required reviews設定がある中での実現方法を悩みましたが、approve専用のGitHub Appを作成してCIジョブにapprove処理を組み込みました。GitHub AppのクレデンシャルはDependabot secret storeに保管することで、Dependabot以外がrequired reviews設定をバイパスできないようにしました。"Dependabot secret store" という単語はGitHubにはなくドキュメントもあまりないですが、organization設定の "Secrets and variables" からアクセスできる "Dependabot secrets" を指しています。

### 更新を分散させるスケジューリング
プラットフォームチームではチームの性質上チームメンバー1人当たりの運用アプリケーション数が多くなっていて、1つあたりの運用コストはなるべく減らしたいです。また、中には責務の大きいクリティカルなアプリケーションもいくつかあり、手間を少なくかつ堅牢に運用したいという相反する力学があります。そういう背景から、Dependabotによる依存パッケージのアップデートについて、一斉に対象となるアプリケーションを更新するのではなく、アップデートを分散させて自動テストでも見つけられない問題があった時の影響を小さくすることにしました。これを実現する直接の機能はDependabot, GitHub両方からも提供されてないため簡単な仕組みを自作しました。

分散方法について、約1ヶ月の間でアプリケーションの更新が分散すれば良しとし、アプリケーション毎に「月の何曜日に更新されるか」を割り振ることにしました。例えば「アプリケーションAは第一週の月曜日、アプリケーションBは第一週の火曜日」のようにざっくり1日ずつずらしていく方式です。具体的な実装としては、Dependabotの設定ファイルで曜日指定をしています。また、ラベルを自動付与する設定があるので、そこで「第何週にマージすべきか」という情報を付与しています。

以下のようなイメージです:

```yaml
  - package-ecosystem: cargo
    directory: container/accounts/xxx/orgu_runner_secrets_scan_dev
    schedule:
      interval: weekly
      time: "12:00"
      timezone: Asia/Tokyo
      day: monday
    labels:
    - week1
    groups:
      cargo-all:
        patterns:
        - '*'
  - package-ecosystem: docker
    directory: container/accounts/xxx/orgu_runner_secrets_scan_dev
    schedule:
      interval: weekly
      time: "12:00"
      timezone: Asia/Tokyo
      day: monday
    labels:
    - week1
    groups:
      docker-all:
        patterns:
        - '*'
  - package-ecosystem: gomod
    directory: container/accounts/yyy/activity-collector-reporter
    schedule:
      interval: weekly
      time: "12:00"
      timezone: Asia/Tokyo
      day: tuesday
    labels:
    - week1
    groups:
      gomod-all:
        patterns:
        - '*'
  - package-ecosystem: docker
    directory: container/accounts/yyy/activity-collector-reporter
    schedule:
      interval: weekly
      time: "12:00"
      timezone: Asia/Tokyo
      day: tuesday
    labels:
    - week1
    groups:
      docker-all:
        patterns:
        - '*'
```

この方式ではマージすべき週ではない週でも曜日さえ合致していたらDependabotによるpull request自体は作成されるので、`on: pull request` のworkflowでpull requestのラベルを参照してマージすべき週でないアプリケーションのpull requestは自動でクローズするようにしました。

アプリケーション数が多いという都合上、Dependabotの設定ファイルを手書き管理したくなかったので、上述のcontainer/kagoにDependabotの設定ファイルを一部生成する機能を追加して、アプリケーションの追加のタイミングでDependabotの設定ファイルを生成し直しています。

## 終わりに
ドキュメントやインターネットにあまりないtipsについて紹介しました。この記事がGitHub Actionsを使ってCI/CDを構築する人の助けになるとうれしいです！また、より良いアプローチや機能の組み合わせで実現している方はぜひ教えてください！
