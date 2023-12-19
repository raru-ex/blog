# GolangでGenericsを使ったID型の試行錯誤とuntyped constantsの学び

## 前置き

こんにちは、どせいさんです。
ぼくは株式会社バニッシュ・スタンダードという会社で、なんかむつかしいことをかんがえてます。

というわけで、今回はGenericsを使って「なんか良い感じ」のID型を定義する試行錯誤を共有したいと思います。
プロダクションで使えるかと言われると、正直「うぅん...?」という感じなのですがせっかくなので公開してしまいます。

まず前提として、うちのシステムではサロゲートキーに対応する値に対して個別のID型を定義しています。
よくある代入ミス防止とかのやつですね。

```go
type UserID uint64 // こういうやつ
```

基本的にこれで全然問題ないですしGolangっぽくて余計なこと考えなくて素敵なのですが、せっかくGenericsがあるのだから
なんかいい感じにできて書かなくてよくなったら楽なのになぁと思った、というのがモチベーションです。

元々テーブルのIDを楽に処理したいと思いつつも「せっかくだし」と欲張って広義のIDとして使えないかなぁ？ という意識を脳の片隅におきながら試行錯誤したので、少し話が分散する部分があると思いますが、ご容赦を。

ちなみにDBは僕がよく使うので、MySQLを想定しています。
うちの会社でもMySQLを使ってます。(Auroraだけど)

いくつかのパターンで実装してみて、それぞれに長所短所があるので、どれが好きかは好みが分かれそうです。
では、試行錯誤の流れにある程度乗りつつ、色々やったことを紹介してみたいと思います。

## パターン1: uint64固定で、ほどほどに厳しく緩いやつ

広義のIDとして使いたい、といいつつ最初からサロゲートキーを意識しまくった定義です。
対応範囲を広く取ろうとすると、僕の知能では煩雑になりすぎてコンパイル通せなかったので。

具体的な実装は以下。

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

type Book struct {
  ID ID[Book]
}

func NewUser(id ID[User]) User {
  return User{
    ID: id,
  }
}

func NewBook(id ID[Book]) Book {
  return Book{
    ID: id,
  }
}

func main() {
  uid := New[User](1)
  bid := New[Book](1)
  // compile error: fmt.Printf("user.id == book.id: %v", uid == bid)
  fmt.Printf("user.id == book.id: %v", uid.Int() == bid.Int())
  _ = NewUser(1)
  // success: var test ID[User] = 1
  // compile error: _ = NewUser(uint64(1))
  // compile error: _ = NewUser(bid)
  _ = NewUser(uid)
}

```

比較的シンプルに使えて、ある程度必要なコンパイルエラーは出してくれているので
自身が利用しているテーブル構成が常にサロゲートキーを持っているような構成であれば、意外と悪くないかもしれないです。

ただ気になる点はありまして、それは以下。

1. 整数リテラルがそのままID[T]へ代入できてしまうため、そのミスをコンパイルで落とせない
2. structに紐づかないIDで使えない
3. stringなど、uint64以外のIDを表現できない

### 整数リテラルがそのままID[T]へ代入できてしまうため、そのミスをコンパイルで落とせない。

気持ち的には結構気になるのですが、実際の業務コードをイメージしてみると意外と気にしなくてもいいのかもしれないと思っています。
というのも、コード上でIDをリテラルとして生成したい時なんて実はないかもしれないからです。

ID型が作られるのはDBからの返却時か、もしあったとしても画面から渡ってくるときくらいです。
そしてそれはもう値として渡されてくるので、コード上で新規に固定値で作るというシーンはなさそうです。

ちなみにテストコーデでは固定値で生成したいケースがありそうですが、それはそれで代入・生成できちゃうほうがむしろ便利です。

それでも不安があるとしたら、gomndのLinterで検出して警告を表示するようにしてしまえば、別に問題ない気がしました。

ちなみに、そもそもなんで `1` が代入可能なの？ という疑問についての調査は `パターン1で出た疑問` として後述しています。

### structに紐づかないIDで使えない

これはなんとなく不便そうと思ってるだけなのですが、そんなに気にしなくても良いかもです。
structに紐づかないIDというのがそもそも存在するシーンがあまり浮かばないからです。(結構あったらごめんなさい)
IDというものは識別子なので、何かに紐づいてないとおかしいはずだと思っていて、それはつまりstructに紐づくってことなんじゃないかと思います。

### stringなど、uint64以外のIDを表現できない

これは結構困る気がします。例えばuuidのように数値ではないものをID型として管理できません。

個別に `type UUID string` みたいなdefined type作れば対応はできるんですが、そうなるとgenericsなIDとそうじゃないIDが混在して煩雑です。
identifier.Codeを追加定義して文字列によるID型を作ることはは可能ですが、今後も似たようなものを増やしたくなる可能性があり、地獄へようこそ感が強いです。

もしこの実装を採用する場合には、運用上の落としどころして、EntityIDという型として利用するものがあるかもしれないです。
ID型というのは適用範囲が広すぎる命名なので、もう少し具体にして適用範囲を狭くすることで、お茶を濁す作戦です。

ただし先ほど記載したようにサロゲートキーとUUID, 人工的ではない(超訳)主キーを定義しているものが混在する場合には利用できませんね。


## パターン2: リテラルで代入可能なのを防いだもの

これはパターン1を修正して、リテラルでの代入をエラーで落とせるようにしたものです。

```go
type ID[T any] struct { // any => structに修正しました
	id uint64
}

func New[T any](id uint64) ID[T] {
	return ID[T]{id: id}
}

func (v ID[T]) Int() uint64 {
	return v.id
}

func (v ID[T]) MarshalJSON() ([]byte, error) {
	return json.Marshal(v.id)
}
```

ID型がstructになっています。
これによりリテラルからの代入もコンパイルで弾くことができます。
またID(1)のような直接のキャストも不可能です。

ただ単純にstructにしてしまうと、json.MarshalでIDが意図した形式で出力されないため
MarshalJSON()メソッドを定義して、対応をしています。

こうすることで `{"ID":{},"Name":"Taro"}` とか `{"ID":{"ID":1},"Name":"Taro"}` となってしまう出力を `{"ID":1,"Name":"Taro"}` の形に修正することができます。
これは結構良さそうですが、一応以下のようなポイントがあります。

1. structに紐づかないIDで使えない
1. stringなど、uint64以外のIDを表現できない
1. テストコードを書く時もNewしてあげないといけない

1,2は前回と同様なので省略します。
3については、本質的には問題ではないのですが、ちょっと置き換えるときに面倒臭いというものです。

他にjson以外でもstructになっていることの弊害があるかもしれないので
もし「こんなので困るよ」というのがあったら教えていただきたいです。

## パターン3: 複数の型を扱えるようにする

これは2の形をベースにさらにuint64以外の型が扱えるように足掻いてみた形です。

```go
type idType interface {
	~uint64 | ~string
}
type ID[T any, E idType] struct {
	id E
}

func (v ID[_, E]) Underlying() E {
	return v.id
}

func New[T any, E idType](id E) ID[T, E] {
	return ID[T, E]{id: id}
}

// Bookの型定義やNew関数, json省略
type User struct {
	ID ID[User, uint64]
}

type Session struct {
	ID ID[Session, string]
}

func main() {
	uid := New[User, uint64](1)
	bid := New[Book, uint64](1)
	sid := New[Book, string]("sessionid")
	// エラー: var test ID[User] = 1
	// エラー: _ = NewUser(1)
	// エラー: _ = NewUser(bid)
	// エラー: fmt.Printf("user.id == book.id: %v", uid == bid)
	// エラー: fmt.Printf("user.id == book.id: %v", uid.Underlying() == sid.Underlying())
	fmt.Printf("user.id == book.id: %v", uid.Underlying() == bid.Underlying()) // true
}
```

こうすることでIDとして使いたい型をidTypeに増やしていくことでカバーできる範囲を拡張できるようになりました。
最初に目指していたものとしては要件を結構満たせる感じになったと思います。

ただこれも個人的に気になるポイントがあります。

1. 記載が冗長
1. 第六感がいつか事故るという

1, 2も主観でありノーロジックなのですが、なんかこれ書くの面倒くさそうだな...と思いました。
あと、これはこれで適用可能な範囲が広すぎてID以外に適用されたり、型定義が煩雑になって可読性が下がりそうだな...と思います。

実際にプロダクションコードを置き換えてみたわけではないので感覚でしかなく、要件としては結構満たせているので
試しに個人開発で使ってみてもいいのですが、うぅんやっぱりなんか面倒臭い気がする...

でもこの実装にたどり着いたときには凄く嬉しかったです。
凄く苦労したので。

### パターン3に至るまでの試行錯誤

書き終わってみると大したことないんですが、途中で苦しんでたことを残しておきます。

最初は僕はこれを以下のように書こうとしていました。

```go
type idType interface {
	~uint64 | ~string
}

type ID[T any, E idType] any

func (v ID[_, E]) Underlying() E {
	switch t := v.(type) {
	case uint64:
		return uint64(v)
	case string:
		return string(v)
	}
	panic("error")
}
```

この実装はいくつか問題があり、エラーが発生します。

####  invalid receiver type ID[_, E] (pointer or interface type)

まず一つ目のエラーがこちら。
僕はこれを「ID[_, E]のEがinterfaceだからエラーです」という意味だと勘違いしていました。

それを確認するために以下を試したところ、コンパイルできちゃったんですね。

```go
type Integer interface {
	~int | ~uint64
}

type test[T Integer, E Integer] struct {
	V  T
	V2 E
}

func (v test[_, E]) Hello() E {
	return 2 * v.V2
}
```

つまり型パラメータがinterfaceなのは問題がないということです。
そりゃそうですよね、constraints.Integerだって型パラメータの制約に使えるんだから...

Golangだとレシーバのときに制限があったりするので、レシーバであることも含めて型パラメータ影響してるかと思ったのですが、全然関係なかったです。

これはGolangの[リファレンス](https://go.dev/ref/spec#Receiver)に書かれている `A receiver base type cannot be a pointer or interface type and it must be defined in the same package as the method.` というのに該当してるみたいです。
つまりID型が結局 `any(interface{])` であるためにレシーバに指定できなかった、ということです。

コンパイルが通る方だと `type test` はstructですもんね。

#### cannot use type switch on type parameter value v.id (variable of type E constrained by idType)

2つ目のエラーがこちらです。
switchで型を確定させてreturnしようとしてるところで、E (idType) に対して uint64, stringが適用されないというエラーでした。

Eはunderlyingがuint64, stringのどちらかなんだから別にいいじゃないと思っていたのですが
これは普通にアホで、Eが決まるとreturnが決まるので素直に以下でよかったです。

```go
func (v ID[_, E]) Underlying() E {
	return v.id
}
```

一度思い込むと、こんなこともわからなくなるんですね。
全然わからなかったです！

# パターン1で出た疑問

で、改めてパターン1のところでNewUser(1)として数値リテラルが渡されたときに、コンパイルエラーにならないのかという疑問を掘っていきたいと思います。

といっても、正直全然よくわからないのでgolangにおける代入可能性の検証について調べてみます。
Golangでは a = b のような代入があるとき以下のルールで検証するらしいです。
引用: https://go.dev/ref/spec#Assignability

```
A value x of type V is assignable to a variable of type T ("x is assignable to T") if one of the following conditions applies:

1. V and T are identical.
2. V and T have identical underlying types but are not type parameters and at least one of V or T is not a named type.
3. V and T are channel types with identical element types, V is a bidirectional channel, and at least one of V or T is not a named type.
4. T is an interface type, but not a type parameter, and x implements T.
5. x is the predeclared identifier nil and T is a pointer, function, slice, map, channel, or interface type, but not a type parameter.
6. x is an untyped constant representable by a value of type T.

Additionally, if x's type V or T are type parameters, x is assignable to a variable of type T if one of the following conditions applies:

7. x is the predeclared identifier nil, T is a type parameter, and x is assignable to each type in T's type set.
8. V is not a named type, T is a type parameter, and x is assignable to each type in T's type set.
9. V is a type parameter and T is not a named type, and values of each type in V's type set are assignable to T.
```

これを見ると、今回のケースでは2のパターンに合致して代入に成功しているように見えます。
※ 2の意訳: 同じunderlying typeを持つ型V, Tの場合、片方がdefined typeではないときには代入できる

ただそれだとなんでuint64(1)ではエラーになるのかよくわからないですよね。
uint64はプリミティブ型なはずなので、明示したところでdefined typeではないから代入できても良さそうですが。。。

よくわからないのでもうちょっと調べてみたら、golangには以下の型定義があることがわかりました。

```go
type uint64 uint64
```

どうもuint64には、uint64を基底にしたdefined typeも存在しているらしいです。
つまりuint64()と明示的に記載するときには、uint64はdefined typeとなっているみたいですね。
そのためuint64(1)とした場合には異なるdefined type同士の代入となり、エラーになったということでしょうか。

ただそう思ってたのですが、色々意見をもらって試してみたら以下のような挙動がわかりました。

```go
func main() {
	var id ID = 1
	fmt.Printf("%T\n", id) // main.ID

	var i = 1
	fmt.Printf("%T\n", i) // int
}

type ID int
```

挙動がわかったも何も、割と普通のコードではあるんですが
これを見て「普通に型推論が効いてるだけなのでは...?」と思いました。
ただそうなったときに、それはそれでわからないことが出てきます。

型推論するとしてもリテラルの1自体には何かしらの型が割り当てられてないとおかしい気がします。
それはint, int32, uint64なにかわからないですが、何かしらのint型が割り当てられてるはずです。

しかし関数の定義やstructの定義にある各int型の定義元に飛んでみると、全部defined typeとして宣言されていました。（ジャンプが常に真なのかという問いもありますが）
これが正しいとするとdefinedじゃない、underlyingなintはどう呼び出したらいいのでしょうか。
以下のルールにある `not a named type` というのは結局どうやってコード上で表現するのか全然わからなかったです。

```
2. V and T have identical underlying types but are not type parameters and at least one of V or T is not a named type.
```

コード上で表現できないならルールが必要ないはずですが、ルールは存在します。
つまり `not a named type` は存在するはずです。
でもコード上それをどうやって書くことが可能なのかよくわかりませんでした。

で、引き続き調べてみたら、golangには `Untyped Constants` というものがあることがわかりました。
参照: https://speakerdeck.com/dqneo/go-specification-untyped-constants?slide=12

確証はないのですが、今回の件はこれなんじゃないかな...
つまり、この疑問だった挙動は以下なのではないかと推測したということです。

1. id: ID[T] という引数に対して、 1というintegerなuntyped constantsが指定される
1. `V and T have identical underlying types but are not type parameters and at least one of V or T is not a named type.` のルールにより代入が可能
1. ID[T]型の変数に代入されたため、当然型はID[T]になる

untyped constantsなのにunderlying typeがuint64なのかという疑問は残るが、そういうことなんじゃないかとおもいました。

これが正しいにしろ間違ってるにしろ、正しい答えをご存知の方がいたら教えてもらえると、とても喜びます。

# まとめ

いくつか試行錯誤してみたのですが、個人的な好きもの順は パターン2 > パターン1 > パターン3です。
ただこれは個人の好みで、実際にコードで使うとしたらパターン1になるかもなぁと思わなくもなかったり...。悩ましい。

ちょっと3つ目のパターンは開発進めるとどうなるのかが気になりますね。
煩雑になるのか、意外と運用できるのか。
でも基本的にはuint64が多くなると思うので、毎回2つの型指定するの面倒臭いですよね。

しかし、そもそもこんなに頑張る意味があるのだろうか？　という気持ちにはなっているのが正直なところです。
ID型を個別に定義するコストはそんなに大きいのか？　と言われると「うぅん別に定型作業として一瞬だしな」という気持ちがあります。

Genericsは追加されましたが、ちょっとした楽をするためにはいいのですが、このレベルで使おうとすると
まだ少し無理やり運用に乗せている感が拭えないため、おとなしく各EntityのためのIDを個別に定義しちゃえばいいのかもしれないですね。

そしてなんか僕はGoのGenericsを使ってるとハマって時間がかかってしまいます。
これをスラスラ書けると良さそうですが、この「いい感じの書き方を探すために時間が溶けまくる」というのがないのがGolangのいいところだと個人的には思っているので
そういうところも悩んでしまってるポイントなのかもしれないですね。

でもやっぱりGenericsは便利なので、隙あらば便利に使えるところを探して楽しんでいこうと思いました。

余談ですが `identifier.New[HyperLongLongLongNameStructure]()` みたいになると面倒臭いなぁって気持ちになるので `id.New()` にしたら型定義書くのと文字数変わらないので、短くしたい気持ちが出ます。
IDEの補完もあるし、id packageが微妙すぎるのでやることはないでしょうけど...
