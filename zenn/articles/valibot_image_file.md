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
import * as v from 'valibot';

const ImageFileSchema = v.pipe(
  v.instance(FileList),
  v.check((input) => input.length > 0),
  v.transform((input) => input[0]),
  v.mimeType(['image/jpeg', 'image/png']),
  v.maxSize(1024 * 1024 * 100),
);
```

なんと公式の方がエゴサして回答コードをくれました。
あぁ〜これでいいのかぁ。わかりやすいですね。
一生ついていきます！

**追記2**

画像を必須項目にしたくないという要件が実現できていたなかったため、それが実現できる方法を載せておきます。

```typescript
const IMAGE_TYPES = ["image/jpeg", "image/png"];
const IMAGE_MAX_SIZE = 10 * 1024 * 1024;

export const ImageFileSchema = v.pipe(
  v.instance(FileList), // input type="file"は未選択でもlength=0のFileListで渡ってくる
  v.transform<FileList, File[]>((input) => {
    const count = input.length;
    const result = [];
    for (let i = 0; i < count; i++) {
      result.push(input.item(i) as File);
    }
    return result;
  }),
  v.checkItems((item) => {
    return item.size <= IMAGE_MAX_SIZE;
  }, "アップロードする画像は10MB以下のものにしてください"),
  v.checkItems((item) => {
    console.log(item.type);
    console.log(IMAGE_TYPES.includes(item.type));
    return IMAGE_TYPES.includes(item.type);
  }, "JPEG, PNGいずれかの画像を選択してください"),
);
```
valibotに用意されているmimeType, maxSizeのActionは利用できなくなるためcheckItemsで自前で頑張るしかないようでした。
絶対何かあるでしょ、と思ってたので面食らったし時間がかかりました...

zodも似たような感じらしいですね。
諸悪の根源はFileListですね。間違いない

**追記3**

```typescript
export const OptionalImageFileSchema = v.pipe(
  v.unknown(),
  v.transform((v) => {
    return v as FileList;
  }),
  v.transform<FileList, File | null>((input) => {
    return input.item(0);
  }),
  v.check((item) => {
    return item === null || item.size <= IMAGE_MAX_SIZE;
  }, "アップロードする画像は100MB以下のものにしてください"),
  v.check((item) => {
    return item === null || IMAGE_TYPES.includes(item.type);
  }, "JPEG, PNGいずれかの画像を選択してください"),
);
```

FileListを直接instanceに指定するとサーバ側にそんな型ないよ、っていうエラーが出てしまうようでした。
一度unknownを挟むことでエラーが出なくなります。（なんでや

絶対に正攻法ではないのですが、もうちょっとしんどいので一旦これでお茶を濁します。
いつの日か追記4で対応するかもしれません。

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


