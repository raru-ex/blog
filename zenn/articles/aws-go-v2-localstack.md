---
title: "Goでlocalstackを使ってS3と触れ合おうとしてハマった思い出と解決"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

aws-sdk-go-v2とlocalstackを使ってローカル開発を行おうとしたら変にハマったので、思い出として記しておこうと思います。

あとついでに最後に他に設定したものとかもチョロチョロ記載しておこうと思います。
いつか誰かの役に立ちますように

## ハマったものと、どうすりゃいいのか

早速ですがハマったものと、何が原因でどうしたのかを記述していきます。

### NoSuchBucketエラーでPutObjectができない

#### 現象

私の手元では以下のような状態で、期待する動作が得られなくなっていました。

- Localstackに対するListBucketは正常に動作するのにPutは動かない
- S3本体に対するPutObjectは正常に動作する
- awslocal(aws cli)からバケットを確認するがバケットは存在している

#### 原因

1. clientの設定でforth path styleを指定していなかった
1. EndpointResolerV2の書き方を普通に間違えていた

通常この問題が発生した場合、原因は1のみであるケースが多そうなのですが、僕の場合はEndpointResolverの書き方を間違えていたので余計に原因が分かりづらくなっていました。

#### 解決方法

##### ForthPathStyleを設定する/EndpointResolerV2を正しく記述する

ForthPathStyleを設定するのにEndpointResolverV2の記述が絡んでくるので、両方セットで記載してしまいます。

```go
func NewClient(cfg config.AWS) s3.Client { // config.AWSは独自型です。なんか環境変数とか入ってる感じです
	var opts []func(*awsConfig.LoadOptions) error
	if cfg.Endpoint != "" {
		opts = append(opts, awsConfig.WithSharedConfigProfile("localstack"))
	}
	
	c, err := awsConfig.LoadDefaultConfig(ctx, opts...)
	if err != nil {
		panic(err)
	}
	
	return s3.NewFromConfig(c, func(o *s3.Options) {
		o.Region = cfg.Region
		o.EndpointResolverV2 = &resolverV2{
			cfg: cfg,
		}
	})
}

type resolverV2 struct {
	cfg config.AWS
}

func (r *resolverV2) ResolveEndpoint(
	ctx context.Context,
	params s3.EndpointParameters,
) (smithyendpoints.Endpoint, error) {
	if r.cfg.Endpoint != "" {
		params.Endpoint = aws.String(r.cfg.Endpoint)
		params.ForcePathStyle = aws.Bool(true) // ここでPathStyleをForceしている
	}

	return s3.NewDefaultEndpointResolverV2().ResolveEndpoint(ctx, params)
}
```

#### 私が何を間違えていたのか

まずS3のバケットのアクセス方法にPathStyleというものと、ドメイン方式のものがあるというのを知りませんでした。
Bucketへのアクセス方法な以下のような2通りがあるみたいです。

- Path:   `https://domain.com/{bucket_name}/{key}`
- Domain: `https://{bucket_name}.domain.com/{key}`

localstackでは(少なくとも初期状態では)Domain方式に対応していないため、PathStyleを強制するように設定する必要があったようです。
localstackの問題なのか`/etc/hosts`に記載がないからなのかは調べてないのですが、どっちにしても不便なのでlocalstackではPathStyleを使うのが良いでしょう。

もうひとつがResolverの書き方です。
僕がもともと書いていた[ここを参考にした](https://aws.github.io/aws-sdk-go-v2/docs/configuring-sdk/endpoints/#with-endpointresolverv2)コードはこんな感じです。


```go
type resolverV2 struct {
	cfg config.AWS
}

func (r *resolverV2) ResolveEndpoint(ctx context.Context, params s3.EndpointParameters) (
        smithyendpoints.Endpoint, error,
    ) {
    if /* input params or caller context indicate we must route somewhere */ {
		if r.cfg.Endpoint != "" {
			u, err := url.Parse(r.cfg.Endpoint)
			if err != nil {
				return smithyendpoints.Endpoint{}, err
			}
			return smithyendpoints.Endpoint{
				URI: *u,
			}, nil
		}
	}

	// delegate back to the default v2 resolver otherwise
	return s3.NewDefaultEndpointResolverV2().ResolveEndpoint(ctx, params)
}
```

コードを書きながら(params捨ててるけどいいんか？？)と思ってたんですが、全然良くなかったです。
このせいで`NewFromConfig`で設定している他の設定を捨ててしまっているっぽくて、何かと問題が出ていました。

そのため以下のように設定を追記/上書きする形にすることで正常に動作するようになりました。

```go
if r.cfg.Endpoint != "" {
	params.Endpoint = aws.String(r.cfg.Endpoint)
	params.ForcePathStyle = aws.Bool(true) // ここでPathStyleをForceしている
}

return s3.NewDefaultEndpointResolverV2().ResolveEndpoint(ctx, params)
```

ボケ〜っとしながらコードを書くと碌なことになりませんね。

## localstackを動かすまでにやったこと、調べたこと

### docker-compose

```yaml
  localstack:
    container_name: "localstack"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"            # LocalStack Gateway
      - "127.0.0.1:4510-4559:4510-4559"  # external services port range
    environment:
      - DISABLE_CORS_CHECKS=1
      - DEBUG=1
      - AWS_ACCESS_KEY_ID=test
      - AWS_SECRET_ACCESS_KEY=test
      - AWS_DEFAULT_REGION=ap-northeast-1
      - SERVICES=s3
      - DATA_DIR=/tmp/localstack/data
    volumes:
      - "./docker/localstack/data:/var/lib/localstack"
      - "./docker/localstack/init:/etc/localstack/init/ready.d"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

基本的には公式サイトの設定を拝借しているのですが、いくつか追加で設定しています。

1. DISABLE_CORS_CHECKS
1. AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY

DISABLE_CORS_CHECKSはCORSで処理が弾かれるの防ぐためです。ローカルではただ開発がしたいだけなので、現状は外しています。

AWS_ACCESS_KEY_ID、AWS_SECRET_ACCESS_KEYなどはpresignedURLを利用するときにハッシュ値の演算に利用されいてるようなので明示してわかりやすくしています。
未指定だと`test`が設定されるようなのですが、そんなの知ってる人しか知らないので...

### 指定したプロファイルのcredentialsを読み込む

上記でcredentialsを指定することにしたので、localでlocalstackを使うときにはそれ用のcredentialsを利用したくなりました。
そのために前述してはいますが、クライアントを生成するときに以下のコードを追加しました。

```go
var opts []func(*awsConfig.LoadOptions) error
if cfg.Endpoint != "" {
	opts = append(opts, awsConfig.WithSharedConfigProfile("localstack"))
}

c, err := awsConfig.LoadDefaultConfig(ctx, opts...)
if err != nil {
	panic(err)
}
```

こうすることで以下のように設定されたcredentialsを読みに行ってくれるようになります。

```sh
$ cat ~/.aws/credentials
[localstack]
aws_access_key_id = test
aws_secret_access_key = test
```

## 感想

この問題で5-6時間ほど苦しんだのですが、見返すと（コード見たらわかるじゃん）って感じがしますよね。
全然わかんなくて発狂してました。

またAWSにもlocalstackにも詳しくなったわけでもないので、認識違いしてることとかあるかもしれないので、そのときは教えてもらえると助かります。

あと僕の設定の問題かもしれないんですが、Localstackが全然ろくなログを出してくれないのでもっと詳しいログを希望したいところです。
見方あるんですかねぇ...



