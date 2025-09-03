---
title: "GitHub ActionsのSHA pinning enforcementを有効にするまでの道のり"
emoji: "🔨"
type: "tech"
topics: ["github", "githubactions"]
published: false
publication_name: finatext
---

2025年8月15日にGitHubからGitHub Actions周りの新機能として ["SHA pinning enforcement"](https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/) 機能がリリースされました。今年の3月に[GitHub Actionsでのサプライチェーン攻撃](https://www.wiz.io/blog/new-github-action-supply-chain-attack-reviewdog-action-setup)があったことは記憶に新しいと思います。この新機能はこの種のサプライチェーン攻撃のリスクを低下させるためのものだと考えています。

まだリリース後間もないので不安定な部分もありますが、セキュリティ向上を目的にFinatextではこの機能を有効化しました。Finatextでのソフトウェア開発にはある程度の規模があるため、単に機能を有効にするだけではうまくいかないことは明白でした。この機能を有効化する前に、いくつかのプラットフォームとしての仕組みを整備することが効果的だったので、その過程を紹介します。

## 機能の概要

[SHA pinning enforcementに関するまとめ記事](https://zenn.dev/shunsuke_suzuki/articles/github-actions-enforce-sha-pinning)を参考にしましたが、検証した結果一部さらに詳細な挙動が分かりました。

- GitHub Actions Policyの一機能としてリリースされました。
- [GitHub Actions Policyのリポジトリ選択機能](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-github-actions-in-your-enterprise)は、プライベートリポジトリにおいてTeamプランでは利用できないですが、今回のSHA pinning enforcement機能についてはTeam/Freeプランでも使用可能です。
- SHA pinning違反がある場合、ワークフロー実行はエラーで終了します(実行時の検査)
- 特定のActionを除外する機能はないです。
- サードパーティActionを含めて、呼び出すreusable workflowやcomposite actionの中の依存先も辿って全ての呼び出しをチェックします。
- 同一organization内の別リポジトリのreusable workflowsの呼び出しは未固定(unpin)でもエラーになりません。
- 同一リポジトリ内のreusable workflowsの呼び出しは未固定でもエラーになりません。
- 同一organization内の別リポジトリのcomposite actionsの呼び出しは未固定だとエラーになります。
  - 後述しますが、おそらくGitHub側の実装バグ。
- 同一リポジトリ内のcomposite actionsの呼び出しは未固定でもエラーになりません。
  - [コミュニティの議論](https://github.com/orgs/community/discussions/170337)で報告されている挙動と異なります。ローカルなものは信頼できるはずなので修正したのだと思います。

直接的な呼び出しだけでなく、サードパーティActionを含む間接的な依存箇所の未固定(unpin)チェックを行いたいと考えていたため、この機能でそれが実現できるのは大きなメリットです。上述のバグが修正されれば、信頼できる同一organization内の呼び出しは対象外になるため、利便性も損なわずに済みそうです。

実装バグには気づいていなかったこともあり、下記で紹介するようにFinatextでは前準備がかなり整っていたので、セキュリティの向上を目的にSHA pinning enforcement機能を有効化することにしました。

## Finatextの取り組み

現在Finatextではアクティブなリポジトリが900以上あり、SHA pinningに向けた人手での修正作業は最小限に抑えたいです。

幸いにも、[全リポジトリを対象としたCI基盤](https://techblog.finatext.com/orgu-e3a3ad0219a8)上でGitHub Actionsのauto-fixジョブを運用しており、さらにGitHub ActionsのサードパーティActionも許可制としてリスト管理していました。この2つの仕組みのおかげで、最小限の労力でSHA pinning enforcementを有効化することができました。

### GitHub Actionsのauto-fix

3月のサプライチェーン攻撃の前から、セキュリティ向上と開発者体験の両立としてGitHub ActionsのSHA pinningを自動適用するCIジョブを整備していました。

SHA pinningを行うツールとして、[stacklok/frizbee](https://github.com/stacklok/frizbee)と[suzuki-shunsuke/pinact](https://github.com/suzuki-shunsuke/pinact)を検討し、当初はより機能セットが期待に近いpinactを利用していました。最終的には、1. 全リポジトリのコードを変更できる強めのジョブなのでコントローラブルにしたいこと、2. 強めのジョブに使うツールなのでpinactのコードを完全に理解してから導入していたこと、3. 期待する挙動が微妙に違っていたこと、以上の理由によりツールについては[gha-fix](https://github.com/Finatext/gha-fix)というシンプルなツールを作り利用するようにしました。

自動で修正するツールがあれば、あとはそれを自動コミットするだけです。[CI基盤](https://techblog.finatext.com/orgu-e3a3ad0219a8)ではGitHub Appのinstallation access tokenがセットされたリポジトリがCIジョブに渡されるので、ジョブの中でgit pushするだけでOKでした。

ジョブの実装ポイントとしては、1. コミットループ検知機構をジョブにいれること、2. コミッターとしてGitHub Appのアイコンを表示することで自動コミットの識別をしやすくする、の2点です。

(1)は単にGitの履歴を見ればOKです。

(2)について、Finatextでは複数の自動コミットCIジョブが動いているので、視覚的にわかりやすくしたいという狙いがあります。GitHubのドキュメントにはないのですが、[この記事](https://josh-ops.com/posts/github-apps-commit-email/)にあるように `<github-app-id>+<github-app-slug>[bot]@users.noreply.github.com` という形式で指定することで、GitHub Appのアイコンが表示されるようになって識別しやすくなります。

このCIジョブがあるので、coding agentなどが生成しがちなgit tagでのバージョン指定も、開発者は気にせずpushすれば全てのリポジトリで自動でいい感じになるようになりました。

以上の仕組みによって、Finatext内のリポジトリに限ればSHA pinningはすでに達成されていたので、問題はサードパーティActionです。

### サードパーティActionの管理
3月のサプライチェーン攻撃への対応として、直後にサードパーティAction管理の仕組みを整備しました。

GitHubは ["GitHub Actions policy"](https://docs.github.com/en/enterprise-cloud@latest/admin/enforcing-policies/enforcing-policies-for-your-enterprise/enforcing-policies-for-github-actions-in-your-enterprise) という機能名で、サードパーティActionのallow/deny list機能を用意しています。この機能を使うのもよいですが、allowlistの運用をスムーズにしたいので簡単な仕組みを内製しました。

GitHub周りの設定を管理するリポジトリでallowlistをホストします。文字列のリストなだけなのでフォーマットはなんでも良いと思いますが、YAMLにしました。

開発者が新規にサードパーティActionを使用する場合は、以下のようなワークフローにしています:

1. 開発者が新規Actionのセキュリティレビューを行ってその結果を使って上記管理リポジトリにPRを作る
    - 見るべき定性的な観点についてのドキュメントと、セキュリティコードレビュー用の生成AI向けプロンプトを整備しています
2. プラットフォームチームがPRのapproveとマージを行う
3. GitHub周りの設定を定期検知しているジョブが別途動いていて、そのジョブの中で上記allowlistを参照して未許可のActionを検知しています

(3)について。あるリポジトリが使用しているActionのリストアップにはGitHubの[SBOM API](https://docs.github.com/en/rest/dependency-graph/sboms?apiVersion=2022-11-28#export-a-software-bill-of-materials-sbom-for-a-repository)を利用しています。SBOM APIを使うためにGitHub organizationレベルで[dependency graph](https://docs.github.com/en/code-security/supply-chain-security/understanding-your-software-supply-chain/about-the-dependency-graph#supported-package-ecosystems)を有効にしました。Dependency graphをorganizationレベルで有効にするには、GitHub Advanced Securityの中に隠れている "security configuration" 機能を使いました。

この管理リポジトリのCIジョブも少し工夫していて、サードパーティAction名のtypoを検知するジョブやソートするジョブなどを入れました。

以上のワークフローにより新規アクションの導入をブロックすることなく、かつメンテナンス性にリスクのあるサードパーティActionの継続的な使用を防いでいます。

リスト管理できるようになっているので、今後は新規Action導入時点でのセキュリティコードレビューだけでなく、バージョン更新時も自動的にセキュリティコードレビューされるようにしていきたいです。

### サードパーティActionの修正
上記の仕組みによって、社内の全サードパーティActionのリストがあったので、各Actionを対象にループを回してcoding agentに要対応Actionをリストアップしてもらい、各ActionにPRを送りました。Coding agent生成のPRはレビュアーに負担をかけることが多いので、PRを作るところは人間が行いました。先に同じ目的でPRを出している人も複数いて、コミュニティ全体で前に進めている感があってほっこりしました。

## 有効化してみての結果
機能をオンにして1日運用してみて大きな問題はないかと思われたのですが、(おそらく)GitHubの実装バグに直面して機能をオフにしました。具体的には「同一organization内の他のリポジトリのcomposite action呼び出し」についてSHA未固定だとpinningエラーになります。同一organization内のcomposite actionsについては信頼できるのと、reusable workflowsについてはリポジトリをまたいでもエラーにならないので、実装バグだと思ってGitHubに詳細なバグレポートを送っておきました。経験的に2,3ヶ月後には修正されるはず(希望…)。

一応CIジョブの設定を変更して、一時的にorganization内のcomposite actions呼び出しもpinするようにして回避し、バグ修正後に、CIジョブに同一organizationのworkflowはunpinする機能を追加するなどして、一時的に失った利便性を戻すこともできますが、現状社内の依存Actionは全てpinされていることが確認できたのでバグ修正をしばらく待ってみようと思います。

## おわりに

かなり前準備が必要な機能ですがセキュリティ向上のメリットは大きいので、有効化に向けてこの記事が参考になるとうれしいです。
