---
title: "GoのEchoでrequestを独自型(uuid)にbindさせる"
emoji: "🍣"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

GoのEchoでuuidを直接requestから受け取りたかったのですが、パッとやり方がわからなかったので記録しておきます。
やってみると、めっちゃ普通で特に特筆することはないのですが誰かの参考になれが幸いですね。


## 結論

まずは最終的にできたコードを記載しておきます。
見てしまうと、なんだ実にGoらしい普通の実装やん、ってなりますよね。
実際それでしかないです。

```go
import (
	"github.com/google/uuid"
)

type UUID uuid.UUID

func (v *UUID) UnmarshalJSON(b []byte) error {
	id, err := uuid.ParseBytes(b)
	if err != nil {
		return err
	}
	*v = UUID(id)

	return nil
}

func (v *UUID) UnmarshalText(text []byte) error {
	id, err := uuid.ParseBytes(text)
	if err != nil {
		return err
	}
	*v = UUID(id)

	return nil
}
func (v *UUID) UnmarshalParam(param string) error {
	id, err := uuid.Parse(param)
	if err != nil {
		return err
	}
	*v = UUID(id)

	return nil
}

func (v *UUID) Underlying() uuid.UUID {
	return uuid.UUID(*v)
}
```

## Echoはどんな感じにデータをBindしているのか

詳細のコードはひたすら愚直にreflectなので、そんなに詳細は追わないのですがざっくりと見ていきたいと思います。

まずEchoが対応しているパラメータの受取方式が以下 (公式から引用

```
query - query parameter
param - path parameter (also called route)
header - header parameter
json - request body. Uses builtin Go json package for unmarshalling.
xml - request body. Uses builtin Go xml package for unmarshalling.
form - form data. Values are taken from query and request body. Uses Go standard library form parsing.
```

そしてこれがBindの処理です。

```go
func (b *DefaultBinder) Bind(i interface{}, c Context) (err error) {
	if err := b.BindPathParams(c, i); err != nil {
		return err
	}

	method := c.Request().Method
	if method == http.MethodGet || method == http.MethodDelete || method == http.MethodHead {
		if err = b.BindQueryParams(c, i); err != nil {
			return err
		}
	}
	return b.BindBody(c, i)
}
```

3つBind処理を呼んでいますが、対応してる方式に対して愚直に処理を呼んでコツコツバインドしている感じですね。


```go
case strings.HasPrefix(ctype, MIMEApplicationJSON):
	if err = c.Echo().JSONSerializer.Deserialize(c, i); err != nil {
	//...
case strings.HasPrefix(ctype, MIMEApplicationXML), strings.HasPrefix(ctype, MIMETextXML):
	if err = xml.NewDecoder(req.Body).Decode(i); err != nil {
	//...
case strings.HasPrefix(ctype, MIMEApplicationForm), strings.HasPrefix(ctype, MIMEMultipartForm):
	params, err := c.FormParams()
    //...
	if err = b.bindData(i, params, "form"); err != nil {
```

JSON, XMLに関しては最終的にはそれぞれ`UnmarshalJSON`, `UnmarshalText`を利用しており、formでの送信に関してはecho内で定義している`bindData`を呼び出しています。
この`bindData`は以下のように`BindPathParams`, `BindQueryParams`でも呼び出していて、json, xml以外はここで処理しているのが分かります。

```go
// BindPathParams binds path params to bindable object
func (b *DefaultBinder) BindPathParams(c Context, i interface{}) error {
	names := c.ParamNames()
	values := c.ParamValues()
	params := map[string][]string{}
	for i, name := range names {
		params[name] = []string{values[i]}
	}
	if err := b.bindData(i, params, "param"); err != nil {
		return NewHTTPError(http.StatusBadRequest, err.Error()).SetInternal(err)
	}
	return nil
}

// BindQueryParams binds query params to bindable object
func (b *DefaultBinder) BindQueryParams(c Context, i interface{}) error {
	if err := b.bindData(i, c.QueryParams(), "query"); err != nil {
		return NewHTTPError(http.StatusBadRequest, err.Error()).SetInternal(err)
	}
	return nil
}
```

そんで、bindDataですが何をしているかというとめちゃくちゃreflectで頑張って最終的には以下のようになります。

```go
func unmarshalInputToField(valueKind reflect.Kind, val string, field reflect.Value) (bool, error) {
	if valueKind == reflect.Ptr {
		if field.IsNil() {
			field.Set(reflect.New(field.Type().Elem()))
		}
		field = field.Elem()
	}

	fieldIValue := field.Addr().Interface()
	switch unmarshaler := fieldIValue.(type) {
	case BindUnmarshaler:
		return true, unmarshaler.UnmarshalParam(val)
	case encoding.TextUnmarshaler:
		return true, unmarshaler.UnmarshalText([]byte(val))
	}

	return false, nil
}
```

ここで`UnmarshalParam`というものが出てくるのですが、これはEchoにより定義されているものです。

```go
package echo
type BindUnmarshaler interface {
	UnmarshalParam(param string) error
}
```

なので、これをJSON, XMLと上記のメソッドを定義してあげればrequestを直接Bindして受け取ることができるというわけです。
Goって感じがしますね。

ちなみに以下のような定義も有ります。

```go
type bindMultipleUnmarshaler interface {
	UnmarshalParams(params []string) error
}
```

これは`type Hoge []int`みたいに配列型の型を定義したときに利用するものになります。
`UnmarshalParams`は未定義でも特にエラーにならなずに処理が無視されるようになっているので、この定義は不要なときは定義しなくて良いです(そもそも定義できないし)

簡単になりますが、以上になります。

## まとめ

最初はCustomBinderを作って、Bind処理を自前で作ってゴリゴリ書いていくのかな？　と思って、CustomBinderに処理を書こうとしてたんですがデフォルトで用意されてるBindの実装を眺めてたら(いやこれ絶対自分で焼き直したり、追加処理書く感じじゃないわ)ってなりました
色々処理を追っていったらUnmarshalがあったので(あぁ、そうだよね、Goってこうだよね)ってなりましたねぇ。

いやぁ、それなりにGoを書いてるつもりだったんですが、全然Go脳になってないですね。

話は変わりますが、formで送られてくるFileって楽にマッピングできないんですかね。
open, closeの管理などがあるので無理かなぁって思ってるんですが、いいやり方あるのかなぁ。

