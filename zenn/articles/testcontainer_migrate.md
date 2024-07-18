---
title: "testcontainersとgolang-migrateでDBを使ったテストを試してみる"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

DBを利用したテストって、どうするのが楽なのか悩みませんか？
僕は悩みます。

テスト対象のDBをどう用意するのか、データの生成とクリーンアップどうしようかなとか...
というわけで、今回は個人開発で試してみたtestcontainersの導入と使ってみた感想を書いていこうと思います。

ゴリゴリにチューニングしたりガッツリ動作確認をしてるような内容ではないので、軽く使いたい方とか向けの記事になります。

## 環境

testcontainers自体は環境に依存するものではないんですが、今回はmigrateツールがGo製なの一応環境を記載します。

言語: Go
DB: Postgres
Migrateツール: https://github.com/golang-migrate/migrate
seed: testfixtures

## 動かしてみる

早速testcontainersを使うためのコードを書いてみます。

### テスト用のDBを持つコンテナを作成する

```go
type pgtestcontainers struct {
	postgres.PostgresContainer
	DSN      string
	TearDown func()
}

func PrepareContainer(ctx context.Context, t *testing.T) pgtestcontainers {
	dbName := "your-db-name"
	dbUser := "root"
	dbPassword := "root"

	// 後述
	scripts, err := readInitScripts()
	if err != nil {
		t.Fatal(err)
	}

	// ここでコンテナを作成しています。
	container, err := postgres.RunContainer(ctx,
		testcontainers.WithImage("postgres:16"),
		postgres.WithInitScripts(scripts...),
		postgres.WithDatabase(dbName),
		postgres.WithUsername(dbUser),
		postgres.WithPassword(dbPassword),
		testcontainers.WithWaitStrategy(
			wait.ForLog("database system is ready to accept connections").
				WithOccurrence(2).WithStartupTimeout(5*time.Second),
		),
	)
	if err != nil {
		log.Fatalf("failed to start container: %s", err)
	}

	// これはDBのteardown用の関数を外に公開するために定義
	teardown := func() {
		if err := container.Terminate(ctx); err != nil {
			t.Fatalf("failed to terminate container: %s", err)
		}
	}

	// testcontainersで生成されたDBへ接続するためのDSNを取っておきます。
	connStr, err := container.ConnectionString(ctx, "sslmode=disable")
	if err != nil {
		t.Fatal(err)
	}

	return pgtestcontainers{
		PostgresContainer: *container,
		DSN:               connStr,
		TearDown:          teardown,
	}
}
```

こんな感じです。
コードコメントにも記載していますが、呼び出し側でtestcontainersにより生成されたDBへ接続するための情報やDBのteardownを行いたいケースがあるためstructに詰めて返却しています。
teardownって名前をつけてますが、普通にterminateのママのほうがわかりやすいですね。

これだけだとイメージがわかないですね。
呼び出し元と使われ方もサンプルを書いてみます。

### 作成してコンテナを利用してみる

呼び出し元のコードを一部省略して記載してみます。

```go
// 途中省略
for _, tt := range tests {
	tt := tt

	t.Run(tt.name, func(t *testing.T) {
		t.Parallel() // 並列で回せる。やったね

		ctx := context.Background()
		// ここでコンテナを生成
		container := testhelper.PrepareContainer(ctx, t)
		// cleanup処理にteardownを設定
		t.Cleanup(container.TearDown)

		// 取得したDSNを利用してテスト用のhandlerを生成
		handler := bun.NewTestHandler(t, container.DSN)
		r := repositoryimpl.NewRepository(handler)

		// seedを入れてる
		testhelper.LoadFixture(t, container.DSN, testfixtures.Directory(tt.fixtures))

		got, err := r.Find(ctx, tt.args.key)
		if (err != nil) != tt.wantErr {
			t.Errorf("Repository.Find() error = %v, wantErr %v", err, tt.wantErr)
			return
		}

		opts := cmp.Options{
			cmp.AllowUnexported(model.User{}),
		}

		if diff := cmp.Diff(got, tt.want, opts...); diff != "" {
			t.Errorf("Repository.Find() diff = %v", diff)
		}
	})
}
```

testcontainersはテストごとにcontainerを用意してくれるため`t.Parallel()`でテストを並列に実行することができます。
DBのテストは工夫しないと直列になりがちなので、これは嬉しいですね。
cleanupではコンテナをterminateしています。

### DBをmigrateする

さて、今の状態だとDBがmigrateされていないのでテストが動きません。
なので、DBをMigrateしていきたいと思います。

```go
func PrepareContainer(ctx context.Context, t *testing.T) pgtestcontainers {
	//....
	scripts, err := readInitScripts()
	if err != nil {
		t.Fatal(err)
	}
	// ....
}

func readInitScripts() ([]string, error) {
	projectRoot, err := file.GetProjectRoot()
	if err != nil {
		return nil, err
	}

	scriptsDir := projectRoot + "/docker/postgres/migrations/"
	entries, err := os.ReadDir(scriptsDir)
	if err != nil {
		return nil, err
	}

	scripts := make([]string, 0, len(entries))
	for _, e := range entries {
		// migration upのファイルのみを対象にする
		if strings.HasSuffix(e.Name(), "up.sql") {
			scripts = append(scripts, scriptsDir+e.Name())
		}
	}

	return scripts, nil
}
```

このコードでtestcontainersで用意されたDBにmigrateを行っています。

#### golang-migrateを採用した理由

1. migrateファイルがup, downで分かれているから
1. migrateファイル以外にschema.sqlみたいなのを用意したくなかったから

1については`readInitScripts`の以下の処理をみてもらうとわかります。

```go
if strings.HasSuffix(e.Name(), "up.sql") { 
	// ...
}
```

`migrate`ではupのファイルだけを全部流せば、本番と同じDBのスキーマが表現できるので、こんな感じにすることができて楽でした。
他に存在しているmigrateツールは一つのファイルの中に以下のようにして定義するものが多くて、migrateのファイルを指定しづらかったんですよね。

```sql
-- +migrate Up
create table...
-- +migrate Down
drop table...
```

これだと生成もdropも同時に実行されるので、都合が悪かったのです。

2についてですが、migrateファイルを管理してるのに別でschemaを管理すると絶対に手運用をミスるので、ほかを用意したくなかったというのが理由です。
migrateの実行をhookにして、schema.sqlを生成するようななんかこういい感じのやり方もあったのかもしれないのですが、今はそんなに頑張りたくないかなという...

ともあれこれで動かせるようになりました。

## 余談

### TestHandlerの実装

TestHandlerという関数がしれっと定義されているので、一応載せておきます。
postgresで外部キー制約を設定しながら開発してるのですが、テストのときには外部キー制約に黙っててほしかったので制約を無効にするように設定しています。

めちゃくちゃ無理やりなのですが、他のやり方は重厚すぎるものしか見つけられなかったので仕方なし。

```go
// テスト用のHandlerを作成する
// 外部キー制約は無視する
func NewTestHandler(t *testing.T, dsn string) *bun.DB {
	t.Helper()

	sqldb := sql.OpenDB(pgdriver.NewConnector(pgdriver.WithDSN(dsn)))
	// テスト時には外部キー制約が面倒なので無効化
	sqldb.Exec("SET session_replication_role = replica;")

	return bun.NewDB(sqldb, pgdialect.New())
}

// 外部キーの動作まで確認する場合に利用するhandlerを生成する
func NewTestHandlerEnableFK(t *testing.T, dsn string) *bun.DB {
	t.Helper()
	sqldb := sql.OpenDB(pgdriver.NewConnector(pgdriver.WithDSN(dsn)))
	return bun.NewDB(sqldb, pgdialect.New())
}
```

### 他のmigrateツールとのコラボ

ちなみにmigrateツールにはflywayっていう有名なのがあるんですが、testcontainersとそれを連携させるようなものがあるみたいです。
[testcontainers-go-flyway](https://github.com/CyberOwlTeam/testcontainers-go-flyway)
testcontainersについて呟いていたら、これを開発しているサーバーフクロウさんチームの方からリプをいただきましたので紹介してみます。

サイバーフクロウさんチーム、いいですよね響きが

## 所感

- テストコードの数が少ないとオーバーヘッドが大きくてテストが遅い
  - 実装時に個別のテストを書いてるときにはtry&errorがちょっとモタついてストレスかもしれない
- 貧弱なPCだとテストを回すとPCが辛そう
- 並列で実行できるのでテスト全体を回すときには良さそう

またまだ並列数の制御とか、テストケース数 > 並列数になったときにコンテナ生成待ちでエラーにならないのかとかまで確認していないです。
多分よしなにやってくれると思うんですが、実務レベルだとちゃんと調整しないと詰まったりモタったりするんじゃないかなぁという気はしますよねぇ。

## まとめ

今回始めて使ってみたのですが、割と良いなぁと思う反面、実装時には（ちょっと遅いなぁ）って正直感じちゃいました。
これは僕が開発に使ってるPCがミニPCだからかもしれないですね。(Macも持ってるんですけどね)(Windowsもあるよ)

ほんとただの感想なんですが、自分全然大したことしてねぇよなぁって気持ちなので、なんか大したことしたいですね。

