# choco60にマウスレイヤー設定を追加する

セパレート自作キーボードのchoco60を作ってみました。  
私はMacを利用しているのですが、セパレートキーボードを利用するトラックパッドに触れるため手の移動すら面倒くさくなってしまうんですね...  
というわけで、keyboardにマウス設定をしてみましたので、簡単に記録を残しておこうと思います。  

私自身がキーボード初心者で何で調べていいのか分からなかったので、そう言う方の助けになれば嬉しいです。  

## 前提

私たち初心者はよくわからず[QMK Configurator](https://config.qmk.fm/#/choco60/LAYOUT)を利用して設定を調整していくと思うのですが、choco60にマウスレイヤーを設定する場合にはここからでは設定できません。  
というのもchoco60はデフォルトではマウス操作機能がfalse設定されており、ビルド設定でマウス操作を有効にする必要があるからです。  

この辺をどう設定するか、を簡単に記載していきます。  
明確な手順書まで残していないので、特に初期セットアップ手順に一部間違いがあるかもしれませんがご了承ください。  
またこれはMacを前提としています。  


## qmkのセットアップ

キーボードの設定はこの`qmk`というものを利用して行っていくみたいです。  
具体的なインストール方法は[こちらのサイト](https://docs.qmk.fm/#/newbs_getting_started)に記載されています。  

### qmkのインストール

まずはqmkをインストールします。  

```sh
$ brew tap qmk/qmk
$ brew install qmk
$ qmk setup
```

キーボードを複数持っていたり、込み入ったことをするのであれば他にもやることがあるかもしれませんが、とりあえずこれだけで問題ありません。  
ちなみに今後何かのコマンドを打った時に依存しているライブラリが足りないよ！ というエラーが出るかもしれません。  
その時はコンソールの促すままにinstallしていただいて大丈夫です。  

### choco60の自分用keymap設定ファイルを作成する

qmkがインストールできたら、自分用の設定ファイル群を生成します。  

```sh
# -kbでキーボードの種類を設定。ここで設定できるのは ~/qmk_firmware/keyboards フォルダ以下にあるもの(だと思います
$ qmk new-keymap -kb choco60
Keymap Name:
```

Keymap Nameは自由に設定して問題ありません。  
今回は`mine`という名前にしたという前提で話を進めていきます。  

正常にファイルが作成できると以下のようになります。  

```sh
$ tree ~/qmk_firmware/keyboards/choco60/ -L 3
~/qmk_firmware/keyboards/choco60/
├── choco60.c
├── choco60.h
├── config.h
├── info.json
├── keymaps
│   ├── default
│   │   ├── config.h
│   │   ├── keymap.c
│   │   └── readme.md
│   └── mine
│       ├── config.h
│       ├── keymap.c
│       └── readme.md
├── readme.md
└── rules.mk
```

`mine`以下にあるのが、今回作成されたファイルです。  
この辺りのファイルを修正しながら、自分のキーマップを作成していきます。  

## マウス操作を有効にする

次はマウス操作を有効にする設定をしていきます。  
これは`rules.mk`の設定を修正する必要があります。  

```c:~/qmk_firmware/keyboards/choco60/rules.mk
MOUSEKEY_ENABLE = yes      # Mouse keys
```

こちら差分だけを記載していますが、MOUSEKEY_ENABLEの部分をyesに設定してもらえればOKです。  

## keymapを設定する

ここまで設定ができたら、keymapをQMK Configuratorで生成されたjson, hexではなくkaymap.cを利用して設定するようにしちゃいましょう。  
参考までに私の設定を以下に記載しておきます。  

```c:~/qmk_firmware/keyboards/choco60/mine/keymap.c
enum layer_names {
    _BASE,
    _FN,
    _MOUSE,
    _MOUSE_2
};

#define KC_FN      MO(_FN)
#define KC_MOUSE   TG(_MOUSE)
#define KC_MOUSE_2 MO(_MOUSE_2)

const uint16_t PROGMEM keymaps[][MATRIX_ROWS][MATRIX_COLS] = {
  [_BASE] = LAYOUT(
    KC_ESC,  KC_1,    KC_2,    KC_3,    KC_4,      KC_5,    KC_6,     KC_7,    KC_8,    KC_9,   KC_0,    KC_MINS, KC_EQL,  KC_BSLS, KC_GRV,
    KC_TAB,  KC_Q,    KC_W,    KC_E,    KC_R,      KC_T,    KC_Y,     KC_U,    KC_I,    KC_O,   KC_P,    KC_LBRC, KC_RBRC, KC_BSPC,
    KC_LCTL, KC_A,    KC_S,    KC_D,    KC_F,      KC_G,    KC_H,     KC_J,    KC_K,    KC_L,   KC_SCLN, KC_QUOT, KC_ENT,
    KC_LSFT, KC_Z,    KC_X,    KC_C,    KC_V,      KC_B,    KC_N,     KC_M,    KC_COMM, KC_DOT, KC_SLSH, KC_RSFT, KC_FN,
    KC_LALT, KC_LGUI, KC_SPC,  KC_SPC,  KC_MOUSE,  KC_RGUI, KC_RALT
  ),
  [_FN] = LAYOUT(
    _______, KC_F1,       KC_F2,       KC_F3,    KC_F4,   KC_F5,   KC_F6,   KC_F7,   KC_F8,   KC_F9,   KC_F10,  KC_F11,  KC_F12,  KC_INS,  KC_DEL,
    KC_CAPS, _______,     _______,     _______,  _______, _______, _______, _______, KC_PSCR, KC_SLCK, KC_PAUS, KC_UP,   _______, _______,
    _______, KC__VOLDOWN, KC__VOLUP,   KC__MUTE, _______, _______, _______, _______, KC_HOME, KC_PGUP, KC_LEFT, KC_RGHT, _______,
    _______, _______,     _______,     _______,  _______, _______, _______, _______, KC_END,  KC_PGDN, KC_DOWN, _______, _______,
    _______, _______,     _______,     _______,  _______, KC_STOP, RESET
  ),
  [_MOUSE] = LAYOUT(
    _______,    _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
    _______,    _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
    KC_MOUSE_2, _______, _______, _______, _______, _______, KC_MS_L, KC_MS_D, KC_MS_U, KC_MS_R, _______, _______, _______,
    _______,    _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
    _______,    _______, KC_BTN1, _______, _______, _______, _______
  ),
  [_MOUSE_2] = LAYOUT(
    _______,    _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
    _______,    _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
    _______,    _______, _______, _______, _______, _______, KC_WH_L, KC_WH_D, KC_WH_U, KC_WH_R, _______, _______, _______,
    _______,    _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______, _______,
    _______,    _______, KC_BTN2, _______, _______, _______, _______
  ),

};
```

慣れた方々はコメントでキーボードの見た目を上手に作って見やすくしているようなのですが、私はしんどかったので普通に記載しています。  
keymapの設定は公式サイトのこの辺りをみるとわかりやすいです。  
[基本的なkeymap](https://docs.qmk.fm/#/keycodes_basic)  
[マウス操作系 keymap](https://docs.qmk.fm/#/feature_mouse_keys)  

私は右のスペースをマウスレイヤー切り替えのトグル用にしています。  
またマウスレイヤー中に左コントロールを押すと、ホイール操作に移行するようにしています。  

このあたりはお好みで設定してください。  

## mouse操作の感度を調整する

このままだとマウスやホイールがカクカクしていて好みではなかったので、感度の設定をしていきます。  
以下は私のサンプルです。  

```c:~/qmk_firmware/keyboards/choco60/mine/config.h
#pragma once

// マウス入力から反応までの遅延
#undef  MOUSEKEY_DELAY
#define MOUSEKEY_DELAY 50

// マウス操作がトップスピードになるまでの時間
#undef  MOUSEKEY_TIME_TO_MAX
#define MOUSEKEY_TIME_TO_MAX 0

// 押しっぱなしの時の反応までのインターバル
#undef  MOUSEKEY_INTERVAL
#define MOUSEKEY_INTERVAL 0

// カーソルの移動スピード
#undef  MOUSEKEY_MAX_SPEED
#define MOUSEKEY_MAX_SPEED 3

#undef  MOUSEKEY_WHEEL_INTERVAL
#define MOUSEKEY_WHEEL_INTERVAL 50

#undef  MOUSEKEY_WHEEL_TIME_TO_MAX
#define MOUSEKEY_WHEEL_TIME_TO_MAX 0

#undef  MOUSEKEY_WHEEL_MAX_SPEED
#define MOUSEKEY_WHEEL_MAX_SPEED 1
```

カーソルのスピードは2とかでもいいかもしれないです。  
3だと細かい動きは無理です。  

time to maxが0だと常にトップスピードになってしまうので、こちら側を調整した方がいいのかなぁ？  
ここについてはみなさんが操作しやすい感度にするのが良いと思います！  

## コンパイルしてhexファイルを生成する

設定が完成したらコンパイルしてみます。  
makeをqmkのルートで実行してください。  

```sh
$ cd ~/qmk_firmware
$ make choco60:mine
```

これでqmk_firmware直下に`choco60_mine.hex`というような名前でhexファイルが生成されます。  

makeはkeyboard:keymap_dir_name のような形で指定するみたいです。  

## まとめ

実際問題マウス操作をそんなに使うかと言うと、私は使わないかもしれないです。  
開発に完全に集中していて、トラックパッド触るほどじゃないけどざっくりカーソル移動をさせたい時には便利そうです。  
せっかく自作したので、いろいろ試してみたいですもんね。  
