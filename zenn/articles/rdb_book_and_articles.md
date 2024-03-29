---
title: "漠然とした「RDBがわからない」から脱却するためのおすすめ書籍"
emoji: "🐷"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: [MySQL,RDBMS,Idea,技術書,書評,読書]
published: false
---

私かれこれ15年ほどシステム開発をしていましたが、お恥ずかしながらRDBがとても苦手でした。
なんとなくの正規化が出来れば困ることなく生存することができてしまったのですね。

いよいよそんな状態では困るということで、直近で脱RDB苦手をするために接した本や記事のことを紹介していきたいと思います。

## 対象者

ちょっと前の僕自身のことなのですが、以下のような人が主な対象になると思います。

- DBは普段使っており、実務でもそんなに困っていない
- SQLを書くことができ、実務でもそんなに困っていない
- なんとなくそれっぽい正規化はできており、そんなにダメだしされることもない
- 「パフォーマンスチューニングして」と言われたら困る

おそらくですが、エンジニア歴が浅くまだ実務の経験がない・薄い人でも同じような手順でRDBに関しては最低限の準備はできるのではないかと思います。
最低限の基準は振れ幅がありそうですが、エンジニア同士の会話を聞いててイメージが湧く、くらいの温度感のイメージです。

## 書籍・記事紹介

元も子もないのですが、本を何冊か読めば「漠然とした何もわからない」という状態からは脱却できます。
追加で間を埋めるような記事を読むと最低限の知識は頭に入りますので、以下に記載します。
※ 各書籍に関しての個人的な感想は後述します。

### 書籍

- [達人に学ぶDB設計徹底指南書](https://amzn.asia/d/faVMcwQ)
- [達人に学ぶSQL徹底指南書 第2版](https://amzn.asia/d/eaNClYA)
- [理論から学ぶデータベース実践入門](https://amzn.asia/d/2SmrVUF)
- [失敗から学ぶRDBの正しい歩き方](https://amzn.asia/d/0mQDapK)

### おすすめの順番

基本的に興味がある部分から参照すると良いと思いますが、一通り読み終わった後に個人的に「この順番で読めばよかったなぁ」と思ったものを記載します。

1. **達人に学ぶDB設計徹底指南書**
2. 理論から学ぶデータベース実践入門
3. 失敗から学ぶRDBの正しい歩き方
4. 達人に学ぶSQL徹底指南書

２以降はそんなに気にしなくていいのですが **達人に学ぶDB設計徹底指南書** はいちばん最初に読むべきだと思います。
タイトルが「達人」のため身構えてしまったのですが、この本が最も基礎的で網羅的でとっつき易い内容の本になっていました。

逆に **失敗から学ぶRDBの正しい歩き方** はある程度RDBがわかってからの方が理解し易いと感じました。
本自体はとてもわかりやすいのですが、前提になる知識が欠如してると間を埋めるために他で知識を拾ってくる必要がありました。
捕捉・説明もしてくれており丁寧で親切な作りなのですが、それを見ても脳内でデータや処理がイメージできないことがあったので、あえて優先度をつけるならというレベルではありますが。

4つ目の **達人に学ぶSQL徹底指南書** のものに関しては前半はRDBというよりはSQLに関する本なので、ちょっと他とテイストが違います。
ただRDBが苦手な人は、SQLも「動かせるけど実は苦手」な人が多いと思うので、最後に読んでみると良いのかなと思います。
第２部はRDBについてのお話なので **達人に学ぶDB設計徹底指南書** の学び直しにもなって良いですよ。

### 記事紹介

上記の書籍を読み進める間や、読んだあとに追加で参考にしていた記事の一部です。
普段MySQLを使うことが多いため、RDB全般よりはMySQLによって調べてました。
僕は特にINDEXに対するイメージが上手く出来ていなかったので、それに関するものが多くなっています。

書籍ではないのでここで簡単に紹介しつつ感想を書いてみます。

- [B-treeインデックス入門](https://qiita.com/kiyodori/items/f66a545a47dc59dd8839)
- [MySQL with InnoDB のインデックスの基礎知識とありがちな間違い](https://techlife.cookpad.com/entry/2017/04/18/092524)

1つ目の記事ではINDEXのイメージがかなり出来たため、特徴を覚えたりパフォーマンス特性と暗記しなくてもよくなりました。
また実務でINDEXを設計するときや当てに行くときに、自分で考えられるようになったのでとても助かりました。
複雑な話ではないので、もっと前から勉強したら良かったと思いました。

2つ目のものは実際の利用シーンについて書かれているため、1つ目で理解した（気になった）ものの検証にしつつ、関連する話題が理解できるかどうかの確認が出来てよかったです。

- [ネクストキーロックとは](https://softwarenote.info/p1067/)

ロックに関しては各書籍であまり具体的には学べていなかったのでMySQLに関してですが追加で確認してみていました。
MySQLだと外部キーを使うの避けたりするケースがあるのがなんでだっけ？　というのが毎回わからなくなり、その都度調べてるのですが、その途中でネクストキーロックのワード出てきたので調べてました。



## 各書籍に対する感想

### 達人に学ぶDB設計徹底指南書

名前から上級者向けなのか！？　と感じて後回しにしていたのですが、全然そんなことはなかったです。
むしろ **RDBに関して初心者がまず一番最初に読むべき本** をあげることがあれば、今後はこの本をオススメしていくでしょう。

僕が気に入ったポイントは以下

- RDBとしての目指すべき指針を示しつつも、現実問題実務ではどの落とし所に落としていくのか書いてあること
- 設計、正規化、パフォーマンス、実運用でよく出るけど困るデータパターンへの対応など広い範囲を網羅している
- 網羅性が高く各項目の文章量は多くないが、必要なポイントを抑えており対象の内容をしっかりと学ぶことができる
- B-Tree indexの説明が記載されている

個人的に非常にバランスが良い本棚と感じています。
内容もそうですが筆者のスタンスも理論と実務の間でバランスが良い印象を受けました。

あと僕はindexが全然良くわかってないがゆえに、DBに対して苦手意識を持っていたのですがそのindexへの言及がされていることが非常に気に入りました。
内容も全体を網羅するというよりは仕組みを端的に表しているため、概要を理解するのにちょうどよかったです。詳細に知りたい場合には物足りないと思いますが、今回の目的には十分でした。

逆にそんなに自分が求めていたものに対してはヒットしなかったのは、物理レイヤーに関しての説明やRAIDに関する章の部分がありました。
クラウドが主流になってきてるのもあり、大事なのは重々承知しつつ、求めている内容とは違うなと思いました。

が、RAIDに関しても他の人が話してるときに「意味わからん」って思ってたので、個人的には面白かったです。

### 理論から学ぶデータベース実践入門

RDBの「R」ってそもそもなんなんだっけ？　というところから学んでいくということで理論から学ぶ〜となっていると思われる本です。
先述の本でもその点に関して説明はありますが、こちらのほうがその部分の比重が大きな本となっています。

この本で僕が気に入ったのは以下のあたりです。

- RDBが述語論理と集合など一定の理論がベースになっていることが学べる
- 正規形というものがあり正規化の作業はこれを高次のものに分解する作業だと学べる
- 正規化にも理論があり、一定の規則に則って行うことができると学べる
- 履歴、グラフ、インデックス、トランザクションなど実務で悩む、ハマる点に関しての記載がある

僕は数学や述語理論などをそもそも正しく理解してないので、述語論理や集合論に関して記載の細かい部分が正しいのかどうかはわかりません。
なので、そこは評価できないです。
ただRDBがざっくりどんなものを基礎としているのかが分かることが非常に良かったです。

あとテーブル設計をしてて履歴やツリー構造などはよく出てくるけど、対応に苦心することが多いんですが、それに対する対応パターンが割と多く列挙されているのが読んでいて楽しかったです。
後半は比較的実践で出会ったり耳にするものが多く書かれているので、前半の理論が辛かったら読み飛ばして後半だけ読んでしまうのも良いのかもしれないなと思いました。

とはいえ、銀の弾丸はないため「答え」は示されていないため、今後の参考情報が欲しいなぁというモチベーションで向き合って読むのが良いかなと思いました。

これを最初に読むものとしてあまりオススメだと感じなかったのは、内容自体は詳細に記載されてはいるが答えはなく、理論が中心で「仕組み」への言及は多くないためかもしれません。
「なるほど、RDB（の機能）がなんとなくわかったぞ」となりづらいため、一度機能的な全体像を抑えてからの2冊目以降に良いのかなぁ〜と思いました。

また理論から学ぶ〜**入門**のため述語論理自体は知ってるよ、という人が理論を学ぶことを期待して買うと肩透かしを食らいそうです。

でも私のようなRDBを体系的に学んだことがない人はオススメできると思います。

### 失敗から学ぶRDBの正しい歩き方

表紙に記載されている「効かないINDEX」「強すぎる制約」「知らないロック」などなどRDBを運用していく中で発生する「失敗」を題材に、どうすべきだったのかを教えてくれる書籍。
平易でわかりやすい言葉遣いになっていて、他の書籍と比べて読み進めやすかったなという印象が強く残っています。

この本で僕がいいなとおもったのは以下。

- アンチパターンが産まれる背景や、アンチパターンに陥らないためにどんな手段があるか書かれている
- 運用観点が強めで監視、ロギングなどRDBそのものではないが重要なものにも触れられている
- システムから利用される場面をイメージできるようなストーリーが組まれている
- テンポの良い文章になっており、サクサク読み進められる
- 追加学習を行うために参考になる資料が多く記載されている

失敗から学ぶ本なので、実際に業務の中で遭遇しやすい問題がメインに記載されておりイメージがしやすいので「わかるなぁ」という気持ちで読み進められるのがよかったです。
ただこちらの本もおそらく対象者は初心者になっているため、高度なものを求めている人には薄く感じてしまうかもしれません。

とはいえ「よくない」と思っていつつも採用してしまうパターンも出てくると思うので、改めてどうたち振る舞うべきだったのか振り返るには良いかもしれません。

僕は最初にこの本を読んだのでIndexなどあまり良くわかっておらず「感の良い皆さんならもうおわかりですよね？」みたいな文章を見ながら（いや？　全然わからんが？？）となったりしてました。
感の良い皆さんになりたい皆さんは、先に簡単にでも他の書籍で肩慣らしをしておくと気持ちよくなれますよ。

### 達人に学ぶSQL徹底指南書

こちらは他の本とは違いSQLに関する指南書です。
とはいいつつも、SQLに関して集中的に書かれているのは全体の半分で、後半は「リレーショナル・データベースの世界」というRDBに関する話となっています。
後半部分については他の書籍とも重複するものが多いので、SQLに一度頭を持っていかれたあとのRDBのおさらいに丁度いいのかなと思いました。

この本で良さそうかなぁと思ったの以下のあたりです。

- SQLを書くときに無意識に手続きをイメージしてたものを繰り返し切り替えさせてくれる
- ウィンドウ関数についての記載が結構多く確保されている
- 過去の章に出てきたものが、あとの章でも繰り返し出てくるためステップアップで学びやすい

特にこの本だとCASE式の使い方や効力を取り上げてることが多く、正直普段使うことがないため良い学びになりました。
僕は集計やダッシュボード作成などでは重宝しそうなSQLを書くのが苦手なのですが、そういう人にはもってこいな内容で固められているような気がします。

また普段意識してなかったのですが、CASEが式であるように「あえて言うなら」関数型っぽい思想であるというのは、晴れやかな気分になりました。
違うとわかって考え方を変えようとしても手続きでイメージしてしまうことがあるんですが、とりあえず関数型っぽい発想でやればいいか、みたいな肩の荷が下りたラフな気持ちになれました。

RDBの世界という章もRDBの基礎を作った人たちの発言を多く引用しており、どんな思想で作られているのかが染み付くように書かれているので非常に読みやすかったです。

## まとめ

漠然とした「よくわからない」を解決するには、その界隈で比較的良いと言われている本を何個か見繕って読むのが結局はわかりやすいんだなぁという気づきを得ました。
ここでは書籍を読んでみましたが、その技術に関して公式がしっかりとコンテンツを用意してくれているのであればそれを読むのも良いと思います。

書籍ではあまり詳細がわからなかったものは、MySQLの公式ドキュメントを参照してみたりして理解が進むこともありましたので。
PostgreSQLもコンテンツが充実してるらしい？ので、ある程度大きい規模ものはしっかり公式のドキュメントに当たるのも良さそうです。

例えばフロントエンドなどは書籍がすぐ古くなりやすいので、公式読むのが一番いいんでしょうね。

当たり前に普段使っていて、なんとなく使うだけなら動いてしまうためRDBに実は苦手意識を持ってる人は意外にいるのではないかと思います。
そういう人に少しでも参考になればいいなと思います。
(結局本を頑張って読もうって話にしかなってないけど...)

