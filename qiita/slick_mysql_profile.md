# Slick3.3以降でMySQLのtimestamp等の日付関連をLocalDateTimeへマッピングする

Slick3.3から[java.time系のデータ型へのサポートが追加された](https://scala-slick.org/doc/3.3.1/upgrade.html#support-for-java.time-columns)のですが、その時のデータの扱い方がRDBによって大きく差がある状態になっています。  
特にMySQLはほとんどのデータがStringとして扱われているため、slick3.2以下で行っていたような`java.sql.Timestamp`と`LocalDateTime`のMappedColumnTypeを利用したものが動作しなくなっちゃいました。  

対応方法はいくつかあるのですが、参考になる情報がウェブ上に多くないため誰かの参考になればと思い記事にしてみます。  
一番良いと思っている方法をメインで紹介しますが、他の方法についてもおまけ的に後ろの方に記載しようと思います。  

## 公式推奨実装

最初に結論を書いてしまいます。  
公式サイトが英語なので今まで読み飛ばしてしまっていたのですが、公式が推奨の対応方法を書いてくれていました。  

記載があるのは[さっきのリンクと同じここ](https://scala-slick.org/doc/3.3.1/upgrade.html#support-for-java.time-columns)なのですが、具体的には以下の部分です。  

```
If you need to customise these formats, you can by extending a Profile and overriding the appropriate methods. For an example of this see: https://github.com/d6y/instant-etc/blob/master/src/main/scala/main.scala#L9-L45. Also of use will be an example of a full mapping, such as: https://github.com/slick/slick/blob/v3.3.0/slick/src/main/scala/slick/jdbc/JdbcTypesComponent.scala#L187-L365.
```

Profileを拡張子なさい、って書いてありますよね。  
今回で言うと`MySQLProfile`を継承して日付関連のマッピングをoverrideすれば良いと言うことですね。  

では、エラーの原因を追いつつ実装をしてみます。  
最終的な実装だけ知りたい方は[こちら](#profileを実装)から飛んでください。  

### 単純にマッピングを行おうとした場合に発生するエラー調査

実装の前に、そもそもどんなエラーになってしまうかを説明します。  
実際に出力したエラーログは以下です。  

```sh
play.api.http.HttpErrorHandlerExceptions$$anon$1: Execution exception[[DateTimeParseException: Text '2020-03-15 13:15:00' could not be parsed at index 10]]
        at play.api.http.HttpErrorHandlerExceptions$.throwableToUsefulException(HttpErrorHandler.scala:332)
        at play.api.http.DefaultHttpErrorHandler.onServerError(HttpErrorHandler.scala:251)
        at play.core.server.AkkaHttpServer$$anonfun$2.applyOrElse(AkkaHttpServer.scala:421)
        at play.core.server.AkkaHttpServer$$anonfun$2.applyOrElse(AkkaHttpServer.scala:417)
        at scala.concurrent.impl.Promise$Transformation.run(Promise.scala:453)
        at akka.dispatch.BatchingExecutor$AbstractBatch.processBatch(BatchingExecutor.scala:55)
        at akka.dispatch.BatchingExecutor$BlockableBatch.$anonfun$run$1(BatchingExecutor.scala:92)
        at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.scala:18)
        at scala.concurrent.BlockContext$.withBlockContext(BlockContext.scala:94)
        at akka.dispatch.BatchingExecutor$BlockableBatch.run(BatchingExecutor.scala:92)
Caused by: java.time.format.DateTimeParseException: Text '2020-03-15 13:15:00' could not be parsed at index 10
        at java.time.format.DateTimeFormatter.parseResolved0(DateTimeFormatter.java:1949)
        at java.time.format.DateTimeFormatter.parse(DateTimeFormatter.java:1851)
        at java.time.LocalDateTime.parse(LocalDateTime.java:492)
        at java.time.LocalDateTime.parse(LocalDateTime.java:477)
        at slick.jdbc.MySQLProfile$JdbcTypes$$anon$4.getValue(MySQLProfile.scala:404)
        at slick.jdbc.MySQLProfile$JdbcTypes$$anon$4.getValue(MySQLProfile.scala:389)
        at slick.jdbc.SpecializedJdbcResultConverter$$anon$1.read(SpecializedJdbcResultConverters.scala:26)
        at slick.jdbc.SpecializedJdbcResultConverter$$anon$1.read(SpecializedJdbcResultConverters.scala:24)
        at slick.relational.ProductResultConverter.read(ResultConverter.scala:54)
        at slick.relational.ProductResultConverter.read(ResultConverter.scala:44)
```

まずわかりやすく、ここでparseエラーが出ています。  
`Execution exception[[DateTimeParseException: Text$ '2020-03-15 13:15:00' could not be parsed at index 10]]`

index 10なので、入力された日付データでいうと ` ` この空白部分です。  
空白でパースエラーとは一体...?  
という、気持ちになりますね？  はい、なりました。

続きが気になるので、引き続きコードを追ってみます。  
次のヒントは `at slick.jdbc.MySQLProfile$JdbcTypes$$anon$4.getValue(MySQLProfile.scala:404)` のメッセージ。  

MySQLProfileの該当箇所周辺をみてみます。  

```scala:MySQLProfile.scala
override val localDateTimeType : LocalDateTimeJdbcType = new LocalDateTimeJdbcType {
  override def sqlType : Int = {
    /**
     * [[LocalDateTime]] will be persisted as a [[java.sql.Types.VARCHAR]] in order to
     * avoid losing precision, because MySQL stores [[java.sql.Types.TIMESTAMP]] with
     * second precision, while [[LocalDateTime]] stores it with a millisecond one.
     */
    java.sql.Types.VARCHAR
  }
  override def setValue(v: LocalDateTime, p: PreparedStatement, idx: Int) : Unit = {
    p.setString(idx, if (v == null) null else v.toString)
  }
  override def getValue(r: ResultSet, idx: Int) : LocalDateTime = {
    r.getString(idx) match {
      case null => null
      case iso8601String => LocalDateTime.parse(iso8601String) // <- ここです
    }
  }
  override def updateValue(v: LocalDateTime, r: ResultSet, idx: Int) = {
    r.updateString(idx, if (v == null) null else v.toString)
  }
  override def valueToSQLLiteral(value: LocalDateTime) : String = {
    stringToMySqlString(value.toString)
  }
}
```

`// <- ここです` って書いてある場所が、404行目です。  
getValueではDBから取得した値を`getString`で文字列として受け取り、そのまま`LocalDateTime.parse`に渡しているということですね。  

特にフォーマットを指定していないので、parseメソッドのデフォルトフォーマットで対応できるもの以外は全て落ちてしまうような実装になっているわけですね。  
ちなみにデフォルトのフォーマットがどんな感じかと言うと`parse(text, DateTimeFormatter.ISO_LOCAL_DATE_TIME)`こんな感じになっています。  

`ISO_LOCAL_DATE_TIME`が`yyyy-MM-ddTHH:mm:ss`形式になっているので`2020-03-15 13:15:00`では`T`が足らずにエラーになってしまうと言うわけです。  
index 10でエラーになる謎が解決できましたね。  

つまり、このパース処理を「いい感じ」に直してあげれば問題が解決すると言うことです！  

### Profileを実装

原因が判明したので公式情報に則って、MySQLProfileを拡張した独自Profileを作成していきます。  
独自に作成、と聞くと難しく感じますがMySQLProfileの実装のうち必要な場所をコピペしてきて、parse処理を直すだけです。  
MySQLProfileの実装はこちらです。  
[MySQLProfile.scala#L389-L415](https://github.com/slick/slick/blob/master/slick/src/main/scala/slick/jdbc/MySQLProfile.scala#L389-L415)

では、これを参考に実装した処理を以下に記載します。  

```scala:MyProfile.scala
import java.time.format.DateTimeFormatter
import java.time.LocalDateTime
import java.time.format.DateTimeFormatterBuilder
import java.time.temporal.ChronoField

/* LocalDateTimeをプロダクトに適した形に処理できるようにProfile設定を独自に拡張 */
trait MyDBProfile extends slick.jdbc.JdbcProfile with slick.jdbc.MySQLProfile {
  import java.sql.{PreparedStatement, ResultSet}
  import slick.ast.FieldSymbol

  @inline
  private[this] def stringToMySqlString(value : String) : String = {
    value match {
      case null => "NULL"
      case _ =>
        val sb = new StringBuilder
        sb append '\''
        for(c <- value) c match {
          case '\'' => sb append "\\'"
          case '"' => sb append "\\\""
          case 0 => sb append "\\0"
          case 26 => sb append "\\Z"
          case '\b' => sb append "\\b"
          case '\n' => sb append "\\n"
          case '\r' => sb append "\\r"
          case '\t' => sb append "\\t"
          case '\\' => sb append "\\\\"
          case _ => sb append c
        }
        sb append '\''
        sb.toString
    }
  }

  override val columnTypes = new JdbcTypes

  // Customise the types...
  class JdbcTypes extends super.JdbcTypes {

    // PostgresのProfileを参考にミリ秒も含めて対応できるformatterを実装
    private[this] val formatter = {
      new DateTimeFormatterBuilder()
        .append(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
        .optionalStart()
        .appendFraction(ChronoField.NANO_OF_SECOND,0,9,true)
        .optionalEnd()
        .toFormatter()
    }

    override val localDateTimeType : LocalDateTimeJdbcType = new LocalDateTimeJdbcType {
      override def sqlType : Int = {
        java.sql.Types.VARCHAR
      }

      override def setValue(v: LocalDateTime, p: PreparedStatement, idx: Int) : Unit = {
        p.setString(idx, if (v == null) null else v.toString)
      }
      override def getValue(r: ResultSet, idx: Int) : LocalDateTime = {
        r.getString(idx) match {
          case null       => null
          // 文字列から日付型にパースできるようにparseにformatterを渡す
          case dateString => LocalDateTime.parse(dateString, formatter)
        }
      }
      override def updateValue(v: LocalDateTime, r: ResultSet, idx: Int) = {
        r.updateString(idx, if (v == null) null else v.toString)
      }
      override def valueToSQLLiteral(value: LocalDateTime) : String = {
        stringToMySqlString(value.toString)
      }
    }
  }
}

object MyDBProfile extends MyDBProfile
```

自前でformatterを実装していますが、これはPostgreSQL用のProfileから拝借してきた実装になります。  
このように実装することでDB側の日時がミリ秒まで設定されていても正常にパースが行えるようになります。  

LocalDateTimeのparseはミリ秒については一律で設定する方法がなく、ミリ秒の数だけ`SSSSSS`みたいに書かないといけないのでLocalDateTime側のメソッドで処理してもらえると助かりますね。  
また0-9までの指定になっているのは、MySQL側でミリ秒以下が1つでもあると0埋めして9桁で受け取ろうとするからです。  
MySQL自体は6桁までの精度しかない(はず)です。  

ここで作成したprofileをslickを利用するモデルクラス等で利用してもらえば、エラーもなくマッピングができるようになります。  
codegenで自動生成されたTablesファイルで恐縮ですが、こんなイメージです。  

```scala
object Tables extends {
  val profile = slick.jdbc.MySQLProfile
} with Tables

trait Tables {
  val profile: slick.jdbc.JdbcProfile
  import profile.api._

  implicit def GetResultModel(implicit e0: GetResult[Long], e1: GetResult[String], e2: GetResult[LocalDateTime]): GetResult[Model] = GetResult{
    prs => import prs._
    Model.tupled((Some(<<[Long]), <<[String], <<[LocalDateTime], <<[LocalDateTime], <<[LocalDateTime]))
  }

  class Model(_tableTag: Tag) extends profile.api.Table[Model](_tableTag, Some("db"), "model") {
    def * = (id, content, postedAt, createdAt, updatedAt) <> (
      (x: (Long, String, LocalDateTime, LocalDateTime, LocalDateTime)) => {
        Model(Some(x._1), x._2 ,x._3, x._4, x._5)
      },
      (model: Model) => {
        Some((model.id.getOrElse(0L), model.content, model.postedAt, model.createdAt, model.updatedAt))
      }
    )

    def ? = ((Rep.Some(id), Rep.Some(content), Rep.Some(postedAt), Rep.Some(createdAt), Rep.Some(updatedAt))).shaped.<>({r=>import r._; _1.map(_=> Model.tupled((Option(_1.get), _2.get, _3.get, _4.get, _5.get)))}, (_:Any) =>  throw new Exception("Inserting into ? projection not supported."))

    val id:        Rep[Long]          = column[Long]("id", O.AutoInc, O.PrimaryKey)
    val content:   Rep[String]        = column[String]("content", O.Length(120,varying=true))
    val postedAt:  Rep[LocalDateTime] = column[LocalDateTime]("posted_at")
    val createdAt: Rep[LocalDateTime] = column[LocalDateTime]("created_at")
    val updatedAt: Rep[LocalDateTime] = column[LocalDateTime]("updated_at")
  }

  lazy val query = new TableQuery(tag => new Model(tag))
}
```

このようにモデル側では特にマッピングで何かする必要はなく、Stringなどの型と同じように利用することができます。  
※ mapping以外の実装については気にしないでください。  

この方法だとProfileにさえ実装を記載していれば、他のファイルではそれを意識しなくていいので非常に楽ですね。  
これで実装は完了です。  

独自にProfileを作成すると書くと、大変そうですがparse処理を直してるだけと言い直すと、めっちゃ簡単な気がしますよね。実際簡単でシンプルです。  

## おまけ

ここから先はおまけです。  
すごく簡単にではありますが、他の実装方法を紹介します。  
が、個人的にはお勧めしません。  

## MappedColumnTypeを利用した方法

通常であればslickはこの方法で外部の型をマッピングできるようにしていきます。  
ただし、今回については例外でこの方法ではシンプルに実装できませんでした。  

この実装については[こちら](https://qiita.com/lightstaff/items/cb247f98b9bb12213de9)の記事を参考にさせていただきました。

### 試しに作ってみた実装

実装を見た方が早いので、コードを載せます。  

```scala:SlickMappedMySQLDateTime.scala
import java.time.format.DateTimeFormatter
import java.time.LocalDateTime

/* MySQLDateTimeを直接マッピング対象にできないので、mappingに利用するクラスを作成する */
case class MySQLDateTime(v: String) {
  def toLocalDateTime: LocalDateTime = LocalDateTime.parse(v, MySQLDateTime.format)
}

object MySQLDateTime {
  val format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")
  def apply(time: LocalDateTime): MySQLDateTime = MySQLDateTime(time.format(format))
}

// mixinできるようにtraitにしてる。
trait LocalDateTimeColumMapper {
  val profile: slick.jdbc.JdbcProfile
  import profile.api._

  // いわゆるMappedColumnType
  implicit lazy val localDateTimeMapper = MappedColumnType.base[MySQLDateTime, String] (
    { ldt => ldt.v },
    { str => MySQLDateTime(str)}
  )
}
```

まず`MySQLDateTime`という`case class`を実装しています。  
ここが使いづらい原因になっている箇所です。  

本当は直接LocalDateTimeに紐づけたいのですが、MySQLProfileでLocalDateTimeへのマッピングが用意されているので先にそちらの処理が呼び出されてしまうようでした。  

```scala
MappedColumnType.base[LocalDateTime, String] (
    { ldt => ldt.toString },
    { str => LocalDateTime.parse(str, formatter)}
  )
```

例えばこんなMappingをして、以下のように書くとどうなるか。  
`val createdAt: Rep[LocalDateTime] = column[LocalDateTime]("created_at")`

この場合Profileに定義されているLocalDateTimeのマッピングが先に処理されて、その後にMappedColumnTypeの処理に移ろうとするみたいでした。  
なので結局はLocalDateTime.parseの箇所で落ちてしまうわけですね。  

かといってStringに対してのmappingにしてしまうと、普通にStringで使いたいものについても変換がかかってしまいます。  
そのためLocalDateTimeを直接使うことができません。  

そのため一度経由するためのクラスとして独自の`MySQLDateTime`という型が必要になってしまったと言うわけです。  
既に若干冗長になってしまいました。  

しかし、この実装だとまたもうちょっと大変なところがあります。  
モデル側の実装を見てみましょう。  

```scala:Model.scala
def * = (id, content, postedAt, createdAt, updatedAt) <> (
  (x: (Long, String, MySQLDateTime, MySQLDateTime, MySQLDateTime)) => {
    Model(
      Some(x._1),
      x._2,
      x._3.toLocalDateTime,
      x._4.toLocalDateTime,
      x._5.toLocalDateTime
    )
  },
  (model: Model) => {
    Some((
      model.id.getOrElse(0L),
      model.content,
      MySQLDateTime(model.postedAt.toString),
      MySQLDateTime(model.createdAt.toString),
      MySQLDateTime(model.updatedAt.toString)
    ))
  }
)
```

実装を見るとわかりますが、せっかくMappedColumnTypeをしているのにモデル側でも取り回しの処理をケアしてあげないといけない状態になっています。  
ここはもしかしたら私の実装の仕方に問題があるのかも？　と思っていますが...  

このようにMappedColumnTypeでの実装だと、必要になるコード量は増えるのに結局ケアもしないといけないということで手間が倍に増えてしまいました。  
そのため今回のケースについては適切ではなさそうだなという気持ちになりました。  


## Stringのまま処理してモデルへの変換時に処理するパターン

次に一番シンプルな方法で実装してみます。  

```scala
package slick.samples

import java.time.LocalDateTime
import slick.jdbc.{GetResult}
import java.time.format.DateTimeFormatter
import models.Model

/* def *のtupleでマッピングをするサンプル実装 */
object Tables extends {
  val profile    = slick.jdbc.MySQLProfile
} with Tables

trait Tables {
  val profile: slick.jdbc.JdbcProfile
  val format = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")

  import profile.api._

  /* Slick3.3ではDATETIME, TIMESTAMPなどをStringで受け取るため、モデルとの相互変換部分で吸収する */
  implicit def GetResultModel(implicit e0: GetResult[Long], e1: GetResult[String]): GetResult[Model] = GetResult{
    prs => import prs._
    Model.tupled((
      <<[Option[Long]],
      <<[String],
      LocalDateTime.parse(<<[String], format),
      LocalDateTime.parse(<<[String], format),
      LocalDateTime.parse(<<[String], format)
    ))
  }

  /* Slick3.3ではDATETIME, TIMESTAMPなどをStringで受け取るため、モデルとの相互変換部分で吸収する */
  class Model(_tableTag: Tag) extends profile.api.Table[Model](_tableTag, Some("twitter_clone"), "model") {
    def * = (id, content, postedAt, createdAt, updatedAt) <> (
      (x: (Long, String, String, String, String)) => {
        Model(
          Some(x._1),
          x._2,
          LocalDateTime.parse(x._3, format),
          LocalDateTime.parse(x._4, format),
          LocalDateTime.parse(x._5, format)
        )
      },
      (model: Model) => {
        Some((model.id.getOrElse(0L), model.content, model.postedAt.toString, model.createdAt.toString, model.updatedAt.toString))
      }
    )

    def ? = ((Rep.Some(id), Rep.Some(content), Rep.Some(postedAt), Rep.Some(createdAt), Rep.Some(updatedAt))).shaped.<>({r=>import r._; _1.map(_=> Model.tupled((Option(_1.get), _2.get, LocalDateTime.parse(_3.get, format), LocalDateTime.parse(_4.get, format), LocalDateTime.parse(_5.get, format))))}, (_:Any) =>  throw new Exception("Inserting into ? projection not supported."))

    val id:        Rep[Long]   = column[Long]("id", O.AutoInc, O.PrimaryKey)
    val content:   Rep[String] = column[String]("content", O.Length(120,varying=true))
    val postedAt:  Rep[String] = column[String]("posted_at")
    val createdAt: Rep[String] = column[String]("created_at")
    val updatedAt: Rep[String] = column[String]("updated_at")
  }

  lazy val query = new TableQuery(tag => new Model(tag))
}
```

これは`def *`などなど、変換が必要な場所それぞれで丁寧に処理していくパターンですね。  
対応方法としてはシンプルでわかりやすいのかなと思います。  

ただ、全てのモデルや全ての日付型のマッピングでコツコツ実装をしてあげないといけないので精神力が必要になります。  


## まとめ

今までずっとどうしたらいいのか悩んでいたのですが、やはり公式をしっかり読むのが一番ですね。  
もっと良い方法をご存知の方がいたら教えていただけますと幸いです。  

しかし、PostgreSQLなどは日付のparseに適切なformatterが用意されているのにMySQLではなんで日付がそのまま使われているのでしょうか。  
Zoned Timestampなどが入ってきたときなど、微妙に一つのデータ型で管理されているフォーマットパターンが多すぎて対応しきれなかったのですかね？？
Oracleについては、こういうところは丁寧にデータが用意されており流石なのかな？ と思いました。





