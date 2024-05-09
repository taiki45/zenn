---
title: "イテレーター"
---

Rustではイテレーターによる抽象がうまくデザインされていて、コレクションに対する操作が書きやすいです。

網羅的な情報と解説はすでにあるので譲ります。公式ドキュメントも例が豊富で良いです。

Qiitaの記事:

https://qiita.com/lo48576/items/34887794c146042aebf1

公式ドキュメント:

https://doc.rust-lang.org/std/iter/trait.Iterator.html

いいコードが思い浮かばない時は、for loopやすぐに思いつくメソッドの組み合わせで一度書いてみて、構造を見直してリファクタリングしたりLLMに聞いてみたりすると綺麗に書けることが多いです。

## 便利メソッド
### `try_fold`
まずは `fold` から。イテレーターに対して計算をして折りたたみます。

```rust
let a = [1, 2, 3];
let sum = a.iter().fold(0, |acc, x| acc + x);
assert_eq!(sum, 6);
```

`try_fold` はその計算が失敗するかもしれない時に、失敗したらすぐに `Err` (相当)を返してくれます。早期リターンな挙動が実現できるのが、`map` と `fold` や `collect` の組み合わせとは違うところです。

例えばファイルパスのリストがある時に、ファイルの内容を読み込んで、その内容がJSONフォーマットなのでデシリアライズして、最後にコレクション型に詰め込む、みたいな途中に失敗する処理を挟みつつ `map` 相当のことを行う時によく使います。

```rust
fn collect_dir<B, F>(path: &path::Path, f: F) -> Result<Vec<B>>
where
    F: Fn(Vec<B>, path::PathBuf) -> Result<Vec<B>>,
{
    read_dir(path)
        .with_context(|| format!("Failed to read path: {path:?}"))?
        .try_fold(Vec::new(), |acc, entry| {
            let entry =
                entry.with_context(|| format!("Failed to read dir entry in {}", path.display()))?;
            f(acc, entry.path())
        })
}

collect_dir(path, |mut acc, path| {
    let mut allowlists = read_allowlists(&path)?;
    acc.append(&mut allowlists);
    Ok(acc)
})? // -> Result<Vec<Allowlist>>
```

もう1つの例は、「あるディレクトリからディレクトリ名のリストを取ってそれを `Vec<String>` としてほしい」処理です:

```rust
let mut dirs: Vec<_> = read_dir(&repos_path)
    .with_context(|| format!("Failed to read dir: {repos_path:?}"))?
    .try_fold(Vec::new(), |mut acc, entry| {
        let path = entry
            .with_context(|| format!("Failed to read dir entry in {repos_path:?}"))?
            .path();
        let directory_name = path
            .file_name()
            .and_then(|n| n.to_str())
            .with_context(|| format!("Failed to get name: {path:?}"))?;
        acc.push(directory_name.to_owned());
        anyhow::Ok(acc)
    })?;
```

Nightlyだと `try_*` 系のメソッドが増えていて、`iterator_try_reduce` とかはfinal comment periodに入っているっぽいので、そのうち便利メソッドがもっと増えそうです。

https://doc.rust-lang.org/nightly/std/iter/trait.Iterator.html

## その他頻出メソッド
- `find`, `map`, `filter`, `filter_map`, `flatten`: 他の言語でもある定番
- `take`, `take_while`: 無限に続く繰り返しなどで便利。あと文字列処理でよく使います。
- `any`: `find` でしていた処理を書き換えれることが多い。双対のような `all` もよく使います。
- `chain`: イテレーター同士を順番に繋げる。ファイルやネットワークなどから読んだものを順に繋げる時に使います。
- `zip`: イテレーター同士を並列に繋げる。なので中身はタプルになる。
- `collect`: だいたいはこれで `Vec` とかのコレクション型に詰める。ターボフィッシュ記法に早めに慣れると良いです: `collect::<Vec<Nanika>>()`

Tips: 関数ポインタをそのまま渡すことができてクロージャを書かなくてよいです: `iterA.map(Into::into)`、`iterA.map(convert_nanika)`

## `Option`
`Option` は最大1つの値を持つコンテナ型なので、イテレーターとして扱えます。

https://doc.rust-lang.org/std/option/enum.Option.html#impl-IntoIterator-for-Option%3CT%3E

 `Option<Vec<T>>` と `Option<T>` を繋げたい時にパターンマッチで取り出すのではなく、`flatten`, `chain`, `extend` とかで操作できて見通しがよくなります。抽象のパワー！

## 余談: `Option::transpose`
`Option` な値をmapする時にmapに渡す関数が `Resut` を返す時に `Option<Result<T, E>>` ができてしまうと思います。通常なら `anyhow::Context` などで `None` をエラーとして扱えばいいのですが、他にも `Option` を返す計算と組み合わせてなにかをする時は `Option` 同士だと毎回エラーハンドリングしなくても上記の `Option` のイテレーターメソッドを使うと便利です。

そのような時に、先に中の `Result` を剥がして中身の `Err` を早期リターンしつつ `Option<T>` に直す時に便利です。`transpose` 自体は `Result<Option<T>, E>` に直すメソッドです。だいたいのアプリケーションコードでは `Result` が返り値の型になってると思うので、`Result` にさえ直せばquestion mark operator (`?`)で楽にエラーを早期リターンできます。

```rust
let global_allowlist = config
    .allowlist
    .map(|a| Allowlist::from_gitleaks(a, GLOBAL_ALLOWLIST_ID.to_owned(), Vec::new()))
    .transpose()?;

let rule_local_allowlists = config
    .rules
    .map(|rules| {
        rules.into_iter().try_fold(Vec::new(), |mut acc, rule| {
            if let Some(allowlist) = rule.allowlist {
                let id = format!("gitleaks-{}", rule.id);
                let a = Allowlist::from_gitleaks(allowlist, id, vec![rule.id.clone()])?;
                acc.push(a);
            };
            anyhow::Ok(acc)
        })
    })
    .transpose()?;

let allowlists = global_allowlist
    .into_iter()
    .chain(rule_local_allowlists.into_iter().flatten())
    .collect::<Vec<_>>();
```
