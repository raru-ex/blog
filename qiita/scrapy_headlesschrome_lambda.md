# scrapyとheadless-chromeをlambdaで動かしてみる

今回はScalaではなくPythonです！  
個人的にpythonやスクレイピングに興味あったのでscrapyを利用したプログラムを実装してみました。  
ただその中でかなりいろんなハマりポイントにブチ当たってしまったので、記事として残していこうと思います。  

これによって誰か救われる人がいたら嬉しいです。  

結論としては、実際にシステムを稼働させることはできている状態です。  
細かい実装は興味がないと思うので、ハマり・苦しみポイントをまず解説。  
その後実際の実装をご紹介したいと思います。  

## 目次

- 利用技術・ツール
- ハマった罠・苦しんだこと一覧
- ハマった罠・苦しんだこと解決
  - 苦しんだこと
  - ハマった罠
- scrapyの実装
- serverless frameworkの設定

具体的な実装がみたい方は scrapyの実装以下へ  
python3.8のlambdaでheadless-chrome動かないよって方は、ハマった罠の最後の項目まで  

## 利用技術・ツール

- scrapy2.1.0
- selenium3.141
- headless-chromium v64-66
- ChromeDriver 2.37.544315
- AWS Lambda

headless-chromiumは既にバイナリを用意してくれている方がいたため、そちらを利用しています。  

リンクは以下になります。  
chrome: https://github.com/adieuadieu/serverless-chrome/releases/download/v1.0.0-37/stable-headless-chromium-amazonlinux-2017-03.zip
driver: https://chromedriver.storage.googleapis.com/index.html?path=2.37/

wgetなりで取得するなり、GUIから丁寧に配置するなりしてご利用ください。  

## ハマった罠・苦しんだこと一覧

今回は基本的なものから、普通に罠なものまで含めて大いにハマりまくりました。  
ハマれる場所は全部ハマったといっても過言ではありません。  
せっかくですので、それぞれ記載してみたいと思います。

### 苦しんだこと

- pythonでメソッドcallすると引数が多すぎると怒られる
- scrapyのsettingsの取得の仕方がわからない
- gmailのsmtp認証が通らない
- lambdaにファイルがアップできない
- lambdaにpipやaptで入れたパッケージの反映方法がわからない
- 既にあるプロジェクトへのserverless frameworkの設定方法がよくわからない
- scrapyでselenium経由でのページ情報取得方法がわからない

このあたりは調べると比較的すぐに解決できるものではありますが、なんか細々とつまづいてしまいました。  
特にscrapyのsettingsの取得方法はかなり時間を食われました。  

### ハマった罠

- jsで遅延ロードされているDOMが読めない
- headless-chromiumが起動しない
- localで動くheadless-chromiumがlambdaで動かない

このあたりはハマりました。  
DOMが読めないのはよくよく考えればわかるのですが、今回データ取得しようとしたサイトが遅延ロードだと思ってなかったのでビックリしました。  

またheadless-chromium関連は本当に地獄でした。  
特に一番最後。これが本当に地獄でした。  


## ハマった罠・苦しんだこと解決

冗長かもしれませんが、一つずつ解答します。  

### 苦しんだこと

#### pythonでメソッドcallすると引数が多すぎると怒られる

今回は以下のような実装でエラーになっていました。  

```python
class Hoge:
  def getText(str):
    return "hello " + text

hoge = Hoge()
hoge.getText("world")
```

その時のエラーがこちら  
`TypeError: getText() takes 1 positional argument but 2 were given`  

どうみても引数1つしか渡してないのに2つって言い張るわけです。  
これpythonではクラスにあるメソッドを呼ぶときに暗黙的に`self`が渡されるようで、その影響でエラーになっているみたいです。  

関数定義を `def getText(self, str):` に直してあげることで解決できました。  
普通ハマりますよね、これ。  


#### scrapyのsettingsの取得の仕方がわからない

scrapyにはspider, pipeline, middlewareなどのクラスがあるのですが、spider以外でのsettingsの取得方法がわかりませんでした。  
spiderでは `self.settings` で取得可能です。  

いろいろみてみたのですが、結論pipelineでは以下のように取得が可能です。  

```python
# from_crawlerからsettingを取得して読み込むため
def __init__(self, settings):
    self.settings = settings
    self.mailer = GMailSender(settings)

# settingsを取得するため
@classmethod
def from_crawler(cls, crawler):
    return cls(crawler.settings)
```

staticメソッド的にfrom_crawlerが内部的にコールされているので、そこでcrawlerを取得してsettingsを取得して、__init__に渡してあげることで解決しています。  
middlewareもこんな感じで対応できると思います。  

他には以下のような方法でも取得できます。  

```python
from scrapy.utils.project import get_project_settings
get_project_settings()
```

Scalaに慣れすぎていて、コンストラクタ用のメソッドで変数を初期化するのに違和感がありました。  

#### gmailのsmtp認証が通らない

これは単純でアプリケーションからgmailを利用する場合には、gmailアカウント側でアプリケーション認証を許可してあげる必要がありました。  
またpythonでいい感じのメール送信ライブラリあるだろうと思ったのですが、思ったほど簡単そうなものがなくtelnetを思い出させてくれる趣のある実装にたどり着きました。  

とはいえ、いろいろ探してみたところ最終的には以下のような雰囲気のコードになりました。  

```python
msg = MIMEText(body, "text")
msg['Subject'] = subject
msg['To'] = to
msg['Cc'] = cc
msg['From'] = self.settings['MAIL_FROM']

smtp = smtplib.SMTP('smtp.gmail.com', 587)
smtp.starttls()
smtp.login(self.settings['MAIL_USER'], self.settings['MAIL_PASS'])

smtp.send_message(msg)
smtp.close()
```

これであればシンプルでよいですね。  


#### lambdaにファイルがアップできない

容量オーバーでした。  
headless-chromiumとscrapyで160MBほどになるのですが、lambdaは250MBの制限があります。  
私のプロジェクトフォルダの組み方が悪かったために、余計なものまでパッケージングしてしまい380MBほどまで肥大化してしまいました。  

今回はserverless frameworkを利用していたので、以下のようにexclude設定を行うことで解決できました。  

```
package:
  exclude:
    - '**/__pycache__/**'
    - .DS_Store
    - .aws/
    - .bash_history
    - .bashrc
    - .cache/
    - .cache/**
    - .config/
    - .config/**
    - .git/
    - .git/**
    - .gitignore
    - .idea/
    - .idea/**
    - .local/
    - .local/**
    - .npm/
    - .npm/**
    - .pki/
    - .pki/**
    - .profile
    - .python_history
    - .serverless/
    - .serverless/**
    - .serverlessrc
    - .viminfo
    - node_modules/
    - node_modules/**
    - docker/
    - docker/**
```

やけくそ感があって味わい深いですね。  


#### lambdaにpipやaptで入れたパッケージの反映方法がわからない

これはserverless frameworkのpluginにある、serverless-python-requirementsで解決をしました。  
厳密にいうと、aptで入れたものは対応できなかったのですがpip側に対象のパッケージが存在したのでpipでインストールをすることで解決しました。  

serverless-python-requirementsはpipで導入したパッケージをlambda上でいい感じにパスを通しつつインストールしてくれるプラグインみたいです。  
serverlessの実行フォルダにrequirements.txtというファイルがあると、それを参考にパッケージを導入してくれます。  

`$ pip freeze > requirements.txt` のようにして記載しても良いのですが、これだととにかく全部入ってしまうので個別に追加することをお勧めします。  

私は今回以下のようになっています。  

```
Scrapy==2.1.0
scrapy-selenium==0.0.7
selenium==3.141.0
lxml
```

後々お話ししますが、lxmlというのがミソなんですよねぇ

#### 既にあるプロジェクトへのserverless frameworkの設定方法がよくわからない

参考サイトを見ているとプロジェクトの作成から行っているのが多いので、既にあるプロジェクトにはどうすればいいのかよくわかりませんでした。  
推測は立っていたのですが、確信が持てないかんじです。  

結論: serverless.ymlがあればいい

他に特にやることはありません。  
lambdaを利用するときにはエントリーポイントになるメソッドが必要なので、それように処理が必要です。  

serverless frameworkを利用するときにはhandler.pyで用意されることが多いですね。  
これはserverless.ymlの設定次第なので、やりやすいように調整してください。  

#### scrapyでselenium経由でのページ情報取得方法がわからない

これはmiddlewareを利用して解決できます。  
部分的な実装にはなりますが、以下のような実装です。  

```python
def process_request(self, request, spider):
    # spiderで設定された接続先にリクエストを送る
    self.driver.get(request.url)

    # driverをcloseする前にデータを取得
    body = self.driver.page_source
    url = self.driver.current_url

    # close and quit
    self.driver.quit()

    return HtmlResponse(url=url, body=body, encoding='utf-8', request=request)
```

process_request -> spider.process という流れで処理が進むので、このタイミングでリクエストの処理をseleniumで行うように変更してあげればOKというわけですね。  



### ハマった罠

#### jsで遅延ロードされているDOMが読めない

scrapyだけで実装していると遅延ロードされるDOMが読めません。  
この解決のためにheadless-chromiumとseleniumを導入しました。  
これらを利用すると、DOMのロードが完了したあとのデータを取得することができるため問題が解決しました。  

#### headless-chromiumが起動しない

知っていれば大した話ではないのですが、headless-chromiumをseleniumから利用するためのchromedriverはchromiumのバージョンと合わせてあげる必要があります。  
そのためよくわからず個別にインストールすると、実はバージョンが一致しておらず動作しないということが発生します。  

これらを導入する際には気をつけてください。

#### localで動くheadless-chromiumがlambdaで動かない

今回の本題です。  
記事内での位置が微妙なので、目立ちづらいですが...  

今回localとlambdaで完全に同じソース、同じchromium、同じdriver、同じpythonバージョンを利用していたにも関わらずlambdaのchromiumがうまく動作しませんでした。  
エラーが発生するでもなく、唐突にプログラムが終了する形になっており正直原因がわからず。  
かなり長い時間試行錯誤してみたのですが、わからず。  

結局原因は何かというと、lambdaが動いているlinux環境に導入されているパッケージがローカル環境と違うというものでした。  
実はlambdaのpython実行環境は以下のようになっています。  

python3.8: amazon linux 2  
python3.7以下: amazon linux  
参照: https://aws.amazon.com/jp/about-aws/whats-new/2019/11/aws-lambda-now-supports-python-3-8/  

これに気づいて、なんとなく嫌な予感がしたのでpython3.7用にコードや依存関係を修正して再度実行してみたところ、まだ正常には動作しないのですがしっかりとエラ〜メッセージが表示されるように！  

このエラーが内容がxml系のパッケージ不足によるものでした。  
この解決のためにpipでlxmlを導入して、再度デプロイしたところ正常に動作しました。  

サーバーレスはサーバを意識しないといいますが、初めてのサーバレスはめちゃくちゃサーバを意識させられるような出合いになっちゃいました。  

amazon linuxだと入ってるパッケージがamazon linux2だと入っていないんでしょうね。  
chromeインストールに必要なパッケージは以下のようなものがあるので、この中の何かが足りていないと思われます。  

```
$ apt-get -y install libappindicator3-1 libgbm1 fonts-liberation libasound2 libnspr4 libnss3 libx11-xcb1 libxcb-dri3-0 xdg-utils
```

python3.8でもこの辺りのものを順番に検証したら、動作するかもしれません。  
私はその前に力尽きてしまったのでpython3.7で済ませるようにしました。  


## scrapyの実装

ここからはパパッと実装を紹介させていただきます。  
scrapyは使い方を知っていれば実装することは多くないので、コード量はそんなに多くありません。  
全量を載せるのは憚られるので、一部省略して記載させていただきます。  

spiderについては特に説明するほどのものもないので割愛します。  

### middleware

`middleware`
```python
# -*- coding: utf-8 -*-

from scrapy import signals
from pprint import pprint
from scrapy.http import HtmlResponse
from selenium import webdriver
from selenium.webdriver.chrome.options import Options


class CrawlSeleniumDownloaderMiddleware:

    def __init__(self):
        pprint('========== setup headless chrome  ==========')

        # headless chrome起動のオプション設定
        options = Options()
        options.binary_location = '/opt/headless-chromium'
        options.add_argument('--headless')
        options.add_argument('--no-sandbox')
        options.add_argument('--single-process')
        options.add_argument('--disable-application-cache')
        options.add_argument('--disable-dev-shm-usage')
        options.add_argument('--disable-gpu')

        # chrome-driverからchromeを起動
        self.driver = webdriver.Chrome('/opt/chromedriver', chrome_options=options)

        pprint('========== executed headless chrome  ==========')

    @classmethod
    def from_crawler(cls, crawler):
        ... 変更なし ...

    def process_request(self, request, spider):
        pprint('========== start request  ==========')
        # spiderで設定された接続先にリクエストを送る
        self.driver.get(request.url)

        # driverをcloseする前にデータを取得
        body = self.driver.page_source
        url = self.driver.current_url

        # close and quit
        self.driver.quit()

        return HtmlResponse(url=url, body=body, encoding='utf-8', request=request)

    def close_selenium(self):
        self.deriver.quit()

    def process_response(self, request, response, spider):
        ... 変更なし ...

    def process_exception(self, request, exception, spider):
        ... 変更なし ...


    def spider_opened(self, spider):
        ... 変更なし ...
```

ここでポイントは2つです。  
1つめはchromiumのオプション。  
以下は必須なので指定をしてください。  

```
options.add_argument('--headless')
options.add_argument('--no-sandbox')
```

2つ目が各バイナリの配置場所

```
options.binary_location = '/opt/headless-chromium'
self.driver = webdriver.Chrome('/opt/chromedriver', chrome_options=options)
```

lambdaのレイヤーに配置されたものは opt 以下に配置されるようなので、このパスを指定してください。  
もちろんレイヤーにあげるときにご自身が作成したフォルダ構成には依存するので、その場合には調整が必要です。  

### pipeline

pipelineについても特別なことはしていないのですが、以下のような感じになっています。  

```python3
class CrawlPipeline:
    # 念のため少し幅をとる
    DEFAULT_PRICE = 30000 * 1.1

    # from_crawlerからsettingを取得して読み込むため
    def __init__(self, settings):
        self.settings = settings
        self.mailer = GMailSender(settings)

    # settingsを取得するため
    @classmethod
    def from_crawler(cls, crawler):
        return cls(crawler.settings)

    # itemを処理するためにscrapy側から呼び出されるトリガーのメソッド
    def process_item(self, item, spider):
        pprint('========== start pipeline process ==========')
        # 定価以下の場合には通知
        if self.is_default_value(item.get('price')):
            self.send_mail(self.create_mail_body(item))

        return item
```

値段でなんとなく何をしたいのかバレそうですね。  

最後に設定ファイルになります。  

```
# Obey robots.txt rules
ROBOTSTXT_OBEY = False
DOWNLOAD_DELAY = 2
RANDOMIZE_DOWNLOAD_DELAY = True
COOKIES_ENABLED = False

# メール用の設定を追加
MAIL_HOST = 'smtp.gmail.com'
MAIL_FROM = 'hogehoge@gmail.com'
MAIL_USER = 'hogehoge@gmail.com'
MAIL_PASS = 'fugafuga'
MAIL_PORT = 465
MAIL_TLS = True
MAIL_SSL = True

# 独自設定を追加
NOTIFICATION_ADDRESS = 'hogehoge@gmail.com'

# エラーの場合も処理をしたいので全て許可する
HTTPERROR_ALLOW_ALL = True

# Enable or disable downloader middlewares
# See https://docs.scrapy.org/en/latest/topics/downloader-middleware.html
DOWNLOADER_MIDDLEWARES = {
    'amazon_crawler.download_middlewares.CrawlSeleniumDownloaderMiddleware': 543,
}
# See https://docs.scrapy.org/en/latest/topics/item-pipeline.html
ITEM_PIPELINES = {
    'amazon_crawler.pipelines.CrawlPipeline': 300,
}
```

robots周辺はよくわからないままに設定しているのでスルーしてください。  
middleware, pipelineを有効にしている箇所やgmail設定あたりは、少しは参考になるかと思います。  

### handler.py

念のためhanderも記載しておきます。  
scrapyをコマンドではなくlambdaから起動する場合のやり方、ちょっと探しづらいですよね。  

```python
# -*- coding: utf-8 -*-

import scrapy
from scrapy.crawler import CrawlerProcess
from scrapy.utils.project import get_project_settings
from amazon_crawler.spiders.amazon_search_spider import AmazonSearchSpiderSpider
from pprint import pprint

def main(event, context):
    pprint('========== start lambda function ==========')
    process = CrawlerProcess(get_project_settings())

    process.crawl(AmazonSearchSpiderSpider)
    process.start()

    pprint('========== finished lambda function ==========')
```

今気づきましたが、SpiderSpiderってクラス名になってますね、うっかりです。

## serverless frameworkの設定

最後にserverlessの設定ファイルです。  
凝った設定はしていませんが、こんな感じで毎分動くlambdaが設定できました。  

```
service: amazon-crawler

provider:
  name: aws
  runtime: python3.7
  stage: production
  region: ap-northeast-1
  role: arn:aws:iam::--------------------------
  timeout: 120

plugins:
  - serverless-python-requirements

layers:
  chromedriver:
    path: headless-chrome
    description: chromedriver and headless-chromium
    CompatibleRuntimes:
      - python3.7

resources:
  Outputs:
    ChromedriverLayerExport:
      Value:
        Ref: ChromedriverLambdaLayer
      Export:
        Name: ChromedriverLambdaLayer

package:
  exclude:
    - '**/__pycache__/**'
    - .DS_Store
    - .aws/
    - .bash_history
    - .bashrc
    - .cache/
    - .cache/**
    - .config/
    - .config/**
    - .git/
    - .git/**
    - .gitignore
    - .idea/
    - .idea/**
    - .local/
    - .local/**
    - .npm/
    - .npm/**
    - .pki/
    - .pki/**
    - .profile
    - .python_history
    - .serverless/
    - .serverless/**
    - .serverlessrc
    - .viminfo
    - node_modules/
    - node_modules/**
    - docker/
    - docker/**

functions:
  amazon-crawler:
    handler: handler.main
    memorySize: 256
    layers:
      - arn:aws:lambda:ap-northeast-1:-----------:layer:chromedriver:1
    events:
      # every minutes
      - schedule: cron(* * * * ? *)
```

これでなんとなく全体感は伝わるのではないでしょうか。  

## まとめ

やや煩雑な記事にはなりますが、この内容でlambdaにscrapy+headless-chromiumなバッチリリースは可能になると思われます。  
元々２日くらいで作り終わるかなと思ってたのですが、かれこれ１週間もかかってしまいました。  

やはり自分で書いてデプロイまでしっかりやる、というのがいい勉強になりますね。  



