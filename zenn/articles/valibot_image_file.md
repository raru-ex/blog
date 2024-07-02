---
title: "react-hook-formとvalibotを使って画像ファイルのschemaを定義する"
emoji: "📚"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

個人開発で興味本位でvalibotを利用しているのですが、変にハマってしまったので記録しておきます。
もっといいやり方をご存知の方がいたらコメントで教えていただけると嬉しすぎます。

## 利用バージョン

- react-hook-form: 7.51.5
- valibot: 0.35.0

## 結論

最終的に実装したものが以下です。

```typescript
export const ImageFileSchema = v.pipe(
  v.custom<FileList>((v) => {
    return v != null && typeof v === "object" && v.constructor === FileList && (<FileList>v).length > 0;
  }, "アップロードできるのは画像ファイルのみです"),
  v.transform<FileList, File>((v) => {
    return v[0];
  }),
  v.mimeType(["image/jpeg", "image/png"], "JPEG, PNGいずれかの画像を選択してください"),
  v.maxSize(1024 * 1024 * 100, "アップロードする画像は100MB以下のものにしてください"),
);
```
エラーメッセージとかは気にしないでください。

**追記**

```typescript
export const ImageFileSchema = v.pipe(
  v.instance(FileList),
  v.transform<FileList, File>((v) => {
    return v[0];
  }),
  v.mimeType(["image/jpeg", "image/png"], "JPEG, PNGいずれかの画像を選択してください"),
  v.maxSize(1024 * 1024 * 100, "アップロードする画像は100MB以下のものにしてください"),
);
```

## ハマったところ

1. react-hook-formはinput type="file"をresolverで解決するとFileListを当て込む
1. valibotにはFileListのvalidate用の定義が用意されてない(はず)
1. エラーメッセージをカスタムしてたせいで、詳細なエラーが見えてなかった

普段フロントエンドをあまり触っていなかったのと、意外とファイルを送るコードを書く機会がなかったのでreact-hook-formがFileListをセットしてくることを知りませんでした。
multiple属性を設定しなくてもFileListになるんですね...

またFileListだと判明したにvalibotにFileListに適用できるデフォルト定義がないか探していたのですが、意外となさそうでした。
少なくとも僕は見つけられなかったので、もし知ってる方がいたら教えていただきたいです。

こんなよく使う定義がデフォルトで用意されたないわけ無いでしょう、って思ってるんですけどね...

そして余談ですが、無駄にハマった理由は`アップロードできるのは画像ファイルのみです` という仮のエラーメッセージを当てていたため、本来出るはずであった`Invalid type: Expected Blob but received FileList` というメッセージが隠れてしまっていたことでした。

## まとめ

今回はメモレベルの内容でしたが、誰かの役に立てばいいなぁと思います。
valibotは比較的最近出てきたものだと思うので、やはりまだ情報が少ないですね。
大人しくzodを使っておけばよかったかもしれないとか思っちゃったのは内緒です。

人柱が必要そうなのでしばらく使って柱になってみようと思います。


