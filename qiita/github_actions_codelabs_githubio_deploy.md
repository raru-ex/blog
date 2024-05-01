# Github Actionsでgithub.ioに別リポジトリのCodelabsをデプロイする

今回はいくつかの情報が入り混じった内容になってしまいますが、以下の内容を記事にしてみようと思います。  

- Github Actionsを利用したgithub.ioページのリリース
- Github Actionsを利用したCodelabsの自動ビルド
- Codelabs -> github.ioへの連携

## 今回の目的/構成

今回はCodelabsコンテンツを作成しているリポジトリとgithub.ioのリポジトリが分かれています。  
github.ioではCodelabsのコンテンツを配信しており、Codelabsのコンテンツを更新したタイミングで自動反映させるようにすることが目的です。  

箇条書きにすると、以下のような形ですね。  

- Codelabs用リポジトリ
- github.io用リポジトリ
  - Angularで作成されている
  - Codelabs用リポジトリで作成されたコンテンツを配置している

今回はこのデプロイを簡略化していきます。  

## github.ioページのActions

まず、github.ioを自動でリリースするようなactionsを作成していきます。  
最終的なactionsは以下です。  

```yaml:.github/workflows/release.yaml
name: Build and Deploy to Github Pages

on:
  push:
    branches:
      - master
jobs:
  build-and-release:
    runs-on: ubuntu-latest
    if: "!contains(github.event.head_commit.message, 'ci skip')"

    strategy:
      matrix:
        node-version: [10.16.x]

    steps:
      - name: Checkout Branch
        uses: actions/checkout@v2
        with:
          ref: ${{ github.ref }}
      - name: Use Node.js ${{ matrix.node-version }}
        uses: actions/setup-node@v1
        with:
          node-version: ${{ matrix.node-version }}
      - name: Build and Release Angular
        run: ./bin/release.sh
      - name: Deploy to Github Pages
        run: |
          git config --global user.name 'Github Actions'
          git config --global user.email 'actions@github.com'
          git add -A
          git commit -m "[actions - ci ckip] update GithubPages contents"
          git push origin master
```

慣れている方は見ただけで内容がわかると思いますが、簡単に構成要素を補足してみます。  

### Actionsの補足

```yaml
on:
  push:
    branches:
      - master
```

まず、今回はmasterへのpushをトリガーにActionsを作成しています。  
この設定だと直接プッシュや、PullRequestからマージされたときなどに実行されます。  


```yaml
jobs:
  build-and-release:
    runs-on: ubuntu-latest
    if: "!contains(github.event.head_commit.message, 'ci skip')"
```

ここではjobの定義を行っています。  
ビルドしてリリースしたいので `build-and-release` という名前のjobにしました

今回は実行用OSに`ubuntu-latest`を指定して、特定の条件に一致する場合にのみ後述の処理が実行されるようにしています。  
`if: "!contains(github.event.head_commit.message, 'ci skip')"` とすることで、コミットメッセージに`ci skip`という文字が入っている場合には、actionsを実行しないようにしているわけですね。  


```yaml
strategy:
  matrix:
    node-version: [10.16.x]
```

これは変数定義です。  
利用するnodeのバージョンを指定しています。  

```yaml
steps:
  - name: Checkout Branch
    uses: actions/checkout@v2
    with:
      ref: ${{ github.ref }}
```

ここからはjobのステップを記載しています。  
まず作業対象のブランチをcheckoutしています。  
`github.ref` をすることで、イベントが発火したときのブランチが取得できるので、今回だと常に`master`ブランチが対象になります。  

```yaml
- name: Use Node.js ${{ matrix.node-version }}
  uses: actions/setup-node@v1
  with:
    node-version: ${{ matrix.node-version }}
```

次にnodeのsetupです。  
先ほど変数定義したnodeのバージョンを指定して、nodeをsetupしています。  


```yaml
- name: Build and Release Angular
  run: ./bin/release.sh
```

次に、リポジトリの`bin`フォルダに配置されているリリース用のシェルスクリプトを実行しています。  
actionsに全てを記載するのは辛いので、このようにスクリプトを別で用意しておくと楽で良いです。  


```yaml
- name: Deploy to Github Pages
  run: |
    git config --global user.name 'Github Actions'
    git config --global user.email 'actions@github.com'
    git add -A
    git commit -m "[actions - ci ckip] update GithubPages contents"
    git push origin master
```

最後に`release.sh`によって更新されたファイルを`master`ブランチにpushして反映しています。  

### release.sh

あくまで私の環境特化になりますが、参考までにrelease.shを載せておきます。  

前提  

- フォルダ構成は気にせず諦めている
- angularを利用している

```sh
#!/bin/bash

angularDir=./src/github-pages

build_release() {
  # buildのためにangularプロジェクトフォルダへ移動
  cd $angularDir

  # buildに必要なものを用意
  if type "ng" > /dev/null 2>&1; then
    yarn global add @angular/cli
  fi
  yarn install

  # build
  yarn run build  --prod --output-hashing=none

  # 元のフォルダ(プロジェクトルート)に戻る
  cd -
  # 過去にアップロードされたファイルを削除
  rm main-es* polyfills-es* runtime-es* styles.* 3rdpartylicenses.txt index.html favicon.ico

  # ファイル強制配置
  \cp -pfr $angularDir/dist/github-pages/* .
}

build_release
```

行っていることはシンプルで、以下だけです。  

- angularのビルド
- 過去に古いファイルを全部強制削除
- 出力されたファイルを所定の場所に配置

ビルドのたびに`filename-hash.js`みたいな形でハッシュ値が付与されたファイルが生成されてしまうので、ファイルを削除しないとどんどんファイルが増えてしまいます。  
それを回避するために削除しています。  

今見ていて気付いたのですが、削除用コマンドは`\`も付いてないですし`-f`オプションもないですね。  



## CodelabsのActions

次にCodelabs用リポジトリのActionsです。  

```yaml:.github/workflows/deploy-to-github-pages.yaml
name: Build and Deploy to Github Pages

on:
  push:
    branches:
      - master

jobs:
  build-and-release:
    runs-on: ubuntu-latest
    if: "!contains(github.event.head_commit.message, 'ci skip')"

    steps:
      - name: Checkout Branch
        uses: actions/checkout@v2
        with:
          ref: ${{ github.ref }}
      - uses: actions/setup-go@v2-beta
        with:
          go-version: '^1.13.8'
      - name: install claat command
        run: go get github.com/googlecodelabs/tools/claat
      - name: Build Lessons
        run: ./bin/build_all_lesson.sh
      - name: clone github-pages repo
        env:
          TOKEN: ${{ secrets.TOKEN }}
        run: |
          git clone https://${GITHUB_ACTOR}:${TOKEN}@github.com/Christina-Inching-Triceps/Christina-Inching-Triceps.github.io.git
      - name: Deploy to Github Pages
        run: |
          cd Christina-Inching-Triceps.github.io
          cp -pr ../codelabs .
          git config --global user.name 'Github Actions'
          git config --global user.email 'actions@github.com'
          git add -A
          git commit -m "[actions] update codelabs contents"
          git push origin master
```

### Actionsの補足

github.ioのものと同じ部分は省略しちゃいます。  

```yaml
- uses: actions/setup-go@v2-beta
  with:
    go-version: '^1.13.8'
- name: install claat command
  run: go get github.com/googlecodelabs/tools/claat
- name: Build Lessons
  run: ./bin/build_all_lesson.sh
```

ここではgoのインストールとCodelabsビルドツールの`claat`をインストールしています。  
その後に例によって、スクリプトを利用して全てのコンテンツをビルドしてますね。  

```yaml
- name: clone github-pages repo
  env:
    TOKEN: ${{ secrets.TOKEN }}
  run: |
    git clone https://${GITHUB_ACTOR}:${TOKEN}@github.com/Christina-Inching-Triceps/Christina-Inching-Triceps.github.io.git
```

ここからがgithub.ioリポジトリへのCodelabs反映を行っている部分になります。  
evn部分では環境設定としてgithub上に設定できる`Secrets`から`TOKEN`を設定して利用しています。  

やっていること・やろうとしてることは「Actionsで利用しているOS上に,別リポジトリをcloneして普通に更新してpushする」という流れです。  

```yaml
- name: Deploy to Github Pages
  run: |
    cd Christina-Inching-Triceps.github.io
    cp -pr ../codelabs .
    git config --global user.name 'Github Actions'
    git config --global user.email 'actions@github.com'
    git add -A
    git commit -m "[actions] update codelabs contents"
    git push origin master
```

次のこちらではbuildしてupdateされているファイルをgithub.ioのリポジトリにコピーしてadd, commit, pushしています。  


### release用shell script

こちらも参考までにですが、簡単にビルド用のスクリプトを記載しておきます。  

コンテンツのビルド用スクリプト  

```sh
#!/bin/bash

# 最終的な成果物を出力する
outputDir=./codelabs
# ビルド用の一時ファイルの格納フォルダ
tmpDir=./tmp
# 対象外ファイル
ignoreFile=README.md

build() {
  if [ ! -e $outputDir ]; then 
    mkdir -p $outputDir
  fi

  if [ ! -e $tmpDir ]; then 
    mkdir -p $tmpDir
  fi

  # mdファイルを全てマージ
  for file in $1/*.md; do
    if [ $ignoreFile != ${file##*/} ]; then
      cat $file >> $tmpDir/mergedMd.md
    fi
  done

  # 画像をコピー
  cp -pr $1/images $tmpDir/
  # htmlを生成
  claat export -o $outputDir/ $tmpDir/mergedMd.md
  # 不要になったtmpファイルを削除
  rm -rf $tmpDir
}

# ビルド対象markdownファイルが存在するディレクトリを指定して利用する
build $1
```

全てのコンテンツをビルドするためのスクリプト

```sh
#!/bin/bash

# 全てのLessonをビルド
/bin/bash ./bin/build_codelab_src.sh lesson1/documents
/bin/bash ./bin/build_codelab_src.sh lesson2/documents
/bin/bash ./bin/build_codelab_src.sh lesson2.5/documents
```

## まとめ

長くなってしまいましたが、今回行ったのは以下の処理です。  

1. 各リポジトリのビルドAction作成
2. 片方の成果物をもう片方のリポジトリをcloneして追加・更新する

かっこいい連携ではありませんが、愚直に対応する方式でも対応可能なのです！！  

一度設定しておくと、なんだかんだで便利なので皆さんも是非Github Actionsを使ってみてください。  

