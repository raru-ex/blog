---
title: "GolangでGenericsを使ったID型を利用してみたら思ったより微妙だった話"
emoji: ""
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

## 前置き

これは以前に自身が書いた[GolangでGenericsを使ったID型の試行錯誤とuntyped constantの学び](https://zenn.dev/vs_blog/articles/542ec8795d64d9)へのアンサー記事になります。
Genericsが悪いわけでも汎用的なID型が悪いわけでもなく、私のID型を表現する実装方法に問題があるかもしれないので、その点ご了承ください。

## 何が微妙だったのか

具体的な例は後ほど記載していきますが、何が微妙だったのかを端的に以下に記載します。

** IDを識別子として利用するモデルは概念ごとに複数存在するが、そこで統一的に利用できるID型を定義できなかった **

定義できなかったはちょっと言い過ぎなんですが、概ねこれが理由です。
後続にこれを具体的に書いていきたいと思います。

## 前提の説明

利用する前提によって結論は変わっていくと思うので、今回僕の開発における前提を揃えていきたいと思います。

### 今回使っていたID型

前回の記事の焼き直しになりますが、今回使ってみたID型の実装を以下に記載します。

```go
type ID[T any] uint64

func New[T any](id uint64) ID[T] {
  return ID[T](id)
}

func (v ID[T]) Int() uint64 {
  return uint64(v)
}

type User struct {
  ID ID[User]
}

func NewUser(id ID[User]) User {
  return User{
    ID: id,
  }
}

func main() {
  uid := New[User](1)
}

```

こんな感じで特定のStructに関連づける形で固有のIDを定義できるようにしたものです。

IDというのは`識別子`なので、何かを識別するために利用されます。
「何か」とはシステム上はオブジェクト(struct)になるため、structに紐づくことには問題ない気がしてたんですけどね

ただ元記事でも「絶対に何かしらのモデルに紐づけないといけない」ということに、なんとなく不安を感じていたようなので意外と人間の第六感はバカにならないなぁとしみじみします。

### アーキテクチャ

なんか大仰な感じですが、たいしたことはないです。
今回のプロジェクトは`オニオンアーキテクチャ`を参考に実装されています。

そのため依存の方向性はそれに合わせて意識していきたいという形です。

本題ではないのと、試行錯誤して苦しんでいるので多めに見てほしいのですが、参考までにフォルダ構成を記載します。

```shell
.
├── bin/
├── config/
├── core/
│   ├── domain/
│   │   ├── entity/ # <--- 苦しみが漏れ出ているフォルダ
│   │   ├── model/
│   │   └── repository/
│   ├── type/
│   │   ├── convert/
│   │   ├── enum/
│   │   └── identifier/
│   ├── usecase/
│   └── utility/
│       └── localtime/
├── di/
├── docker/
│   └── postgres/
│       ├── Dockerfile/
│       └── migrations/
├── infrastructure/
│   ├── crypto/
│   ├── external/
│   │   └── oidc/
│   │       └── value/
│   └── persistence/
│       └── bun/
│           ├── repositoryimpl/
│           └── table/
└── presentation/
    ├── echo/
    │   ├── response/
    │   ├── server/
    ├── http/
    └── v1/
       └── controller/
```

## 微妙だった具体的な実装

具体的な題材がないとイメージがわかないので、今回遭遇したケースと同じような機能をイメージしていきたいと思います。

### アイテム登録機能とアイテム一覧機能を作成する

この機能を作成する時、以下のような感じでいくつかのモデルを定義したくなります。

1. infrastructure/persistence/bun/table/item.go
2. core/domain/model/item.go
3. core/domain/model/list_item.go

※ DDD云々は本題ではないのでdomain modelについては細かい話は無視

それぞれのモデルは例えば以下の感じの定義です

```go

// infrastructure/persistence/bun/table/item.go
type TableItem struct {
	ID uint64 `なんかORMとかに使うタグ`
	Name string `なんかORMとかに使うタグ`
}

// core/domain/model/item.go
type CreateItemModel struct {
	Name string
}

// core/domain/model/list_item.go
type ListItemModel struct {
	ID identifier.ID[???]
	Name string
}
```

さて、早くも`???`という謎の型が登場しました。
みなさんはここに何を入れたくなりましたか？

僕は`identifier.ID[table.TableItem]`としたくなりました。
しかしそれをすると困ったことになります。

coreのレイヤーであるListItemModelが、より外側にあるinfrastructureに依存してしまいます。
じゃあテーブル定義をcoreに持ってくればいいかというと、ORMのライブラリの知識がcoreに侵入してしまうのでcoreに入れるわけにはいきません。

じゃあ自分自身をGenericsに入れたIDにしてしまえばいいじゃないかということで以下のようにしたとしてみます。

```go
// core/domain/model/list_item.go
type ListItemModel struct {
	ID identifier.ID[ListItemModel]
	Name string
}
```

こうすれば問題なさそうですね。
解決です。しかし不思議ですね、なんだかもう苦しい気持ちになってきました


### アイテム作成のRepositoryを実装しようとしてみる

次にアイテム作成用のRepositoryを作っていきたいと思います。
ここで作成するのは以下2つ。

1. infrastructure/persistence/bun/repositoryimpl/item_repository.go
2. core/domain/repository/item_repository.go

``` go
// infrastructure/persistence/bun/repositoryimpl/item_repository.go
// returnの型については諸説ありますが、ここではIDで考える
type ItemRepository interface {
	Save(context.Context, model.CreateItemModel) (identifier.ID[???], error)
}

// core/domain/repository/item_repository.go
type ItemRepository struct {}

func (r ItemRepository )Save(context.Context, model.CreateItemModel) (identifier.ID[???], error) {}
```

ここにも`???`な悩みポイントが発生しました
ここにはどんな型を入れたくなりますか？

僕はやっぱり`identifier.ID[table.TableItem]`としたくなりました。
しかしこれもinterface側の定義がcoreにあるため行えません。

じゃあ何を指定したらいいんだろう。

他の選択肢だと`CreateItemModel`が浮かびますが、このモデル自体はIDのプロパティを持っていません。
かといって`ListItemModel`では意味がよくわかりません。

**もうほぼ破綻しました**

しかし苦し紛れに実装を続けてみます。
例えば以下のようにすることでコンパイルを通しながら実装を進めることができます。

```go
type ItemRepository interface {
	Save(context.Context, model.CreateItemModel) (identifier.ID[model.ItemCreateModel], error)
}
```

IDのプロパティがなくても、型自体は定義できるのだから適切?な形で定義したらいいのです。

#### おまけの試行錯誤

ちなみに私は別の方法で実装を勧めて見ました。

```shell
├── core/
│   ├── domain/
│   │   ├── entity/ # <--- 苦しみが漏れ出ているフォルダ
```

```go
// core/domain/entity/item.go
type Item struct {
	ID identifier.ID[Item]
	Name string
}
```

ORMライブラリに依存しているtable定義とは別に、純粋なテーブルと一致するEntityという名前のモデルを作る形ですね。
一応これで色々と「型の問題」は解決できます。
ただ`そもそもentityってそういうものだっけ？`とか`いやいやtable定義と同じ意味のものを二重管理してるじゃん`とか辛さしかないので、根本的な解決にはなりません。

### 苦しみながらも実装を続ける



