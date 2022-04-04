---
title: "Figma で component を活用する"
emoji: "👨‍🎨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["figma", "design"]
published: true
---

# 🧩 Component?
Component は，名前の通り**細かい部品を Figma 上で使いまわすことができる**機能です．

component のデザインは**利用箇所で同期されている**ので，今後のデザイン変更にも素早く対応することができます．簡潔に言うと DRY 原則をいい感じに守ることができます!
また，自ずと細かい部品ごとのデザインが作られるので，フロントエンドエンジニアの方々に**ページのデザインを待っている間にコンポーネントを実装してもらう**ということも可能です．

# 🪄 Component を作る
![](https://storage.googleapis.com/zenn-user-upload/d6da3d4539ac-20220403.gif =500x)
1. Component のデザインを作って
2. 右クリックして "Create component" を押す ( `Ctrl+Alt+K` でも🙆 )
3. Component が生える

このとき，component の名前が "Component 1" などになっていてわかりにくいので，名前をダブルクリックして「ボタン」などにするとよりよいと思います．

# ✨ Component を利用する

## 📋 複製して利用する

上の手順で作った component  (**main component**) を何らかの手順で複製すると，main component 自体がコピーされる代わりに，その main component の **instance** が作成されます．この 2 つは Layers パネルや Canvas 上のアイコンで見分けることができます．( 💠 的なアイコンが main component, ◇ 的なアイコンが instance です )

![](https://storage.googleapis.com/zenn-user-upload/2c5f8a78bb2e-20220403.gif =500x)

## 🗃 Asset から利用する

作成した component は，**左の Assets パネルからドラッグアンドドロップで呼び出すこともできます**．

![](https://storage.googleapis.com/zenn-user-upload/174b74d2c134-20220404.gif =500x)

# ✏️ Component を編集する
## 🌏 全ての instance を変更する
Main component を編集すると，**その main component の instance 全てで変更が反映されます**．

![](https://storage.googleapis.com/zenn-user-upload/2ce3a0c6818e-20220403.gif =300x)
*上が main component で下が instance．Main component 上で変更があるとそれが instance にも反映される．*

## 🧩 個々の instance を変更する
Instance を変更することもできます (**override** と呼ばれます)．

- Override された変更はその instance のみ影響があります．
- Override された変更は，**Main component 上で変更があっても維持されます**．

![](https://storage.googleapis.com/zenn-user-upload/e5012c610cc9-20220403.gif =300x)
*上が main component で下が instance．Instance 上でテキストを変更したあと (override) に main component でテキストを変更しても override のほうが強いので影響されない．*

上の例ではテキスト**の内容**を override しましたが，テキスト**のスタイル** は override していないので，main component でテキストのスタイルや位置を変更すると instance にも適用されます．

# 🎭 Component の見た目の種類を増やす
Component は**複数の見た目**を持つことができます (**Variant** といいます)．

## 🔬 Variant の仕組み
Component で variant を有効にすると，以下の設定項目が追加されます．

- **Property**
  その component で設定可能な内容．例えば `大きさ` など．
- **Value**
  そのプロパティで設定可能な値たち．例えば `大きさ` property の variant は`大` `中` `小`．

ボタンだと，このようなプロパティが考えられます．

| Property の名前 | Value 1       | Value 2       | Value 3      |
|:----------------|:--------------|:--------------|:-------------|
| `style`         | `primary`     | `secondary`   | `disabled`   |
| `icon`          | `nothing`     | `right arrow` | `trash can`  |

## ✨ Variant を使ってみる
説明を容易にするために，Variant を使いたいケースのストーリーを導入しておきます．

ここにメッセージダイアログの component を作りました．

![](https://storage.googleapis.com/zenn-user-upload/d9758a81c3bd-20220403.png =500x)

いまのところは成功時の component しかないのですが，他に `エラー` と `警告` も増やしたいと考えています．大きさ等も変わらず，唯一変わるのは枠線の色のみなので，variant を使って 1 つの component にまとめてしまうことにしました．

### Variant を有効にする

Variant は以下の手順で有効にできます．

1. Component をクリックする．
2. 右の Design パネルで "Variant" の右にあるプラスアイコンをクリックする
3. Variant が有効になります．

![](https://storage.googleapis.com/zenn-user-upload/e07e379020ca-20220403.gif)

Variant を有効にすると，自動的に 2 つ目の Variant が作成されます．

### Value の値を適切に設定する
デフォルトでは，以下のような property と value が設定されています．

| Property の名前 | Value 1       | Value   2     |
|:----------------|:--------------|:--------------|
| `Property 1`    | `Default`     | `Variant 2`   |

わかりにくいので，以下のように変更してみます．

| Property の名前 | Value 1       | Value 2       |
|:----------------|:--------------|:--------------|
| `種類`          | `成功`        | `エラー`      |

#### Value を変更する

1. Value を変更したい variant をクリックする．
2. 右の Design パネルで value を変更する．

![](https://storage.googleapis.com/zenn-user-upload/ba0a4ffd226b-20220403.gif)

#### Property を変更する

1. Canvas 上か Layers 上のコンポーネントの名前をクリックする．
2. 右の Design パネルで property の名前を変更する．

![](https://storage.googleapis.com/zenn-user-upload/914d798b59c0-20220403.gif)

:::message
ここの画面で，value の並び順や値を変更することもできます．
:::

### Variant ごとにデザインを変える
対応する variant を編集すると，variant ごとのデザインを作成することができます．

![](https://storage.googleapis.com/zenn-user-upload/e883710fd1e5-20220404.png =500x)

このようなデザインにしてみました．

### さらに Variant を増やす
任意の variant を複製すると，新しい variant と value が component に追加されます．
以下の GIF では，「警告」variant を新しく作っています．

![](https://storage.googleapis.com/zenn-user-upload/f9226222abcf-20220404.gif)

:::message
上の GIF では，左上のメニューから Duplicate していましたが，もちろん Ctrl+D や Ctrl+C, V でも同じことができます．
:::

### Variant を使う
通常のように component を呼び出したあとに，**右の Design パネルで value を選択することができます**．
![](https://storage.googleapis.com/zenn-user-upload/c8ef6859992b-20220404.gif)

# 👋 おわり
Component を使うと，**ボタンのデザインの改修をしたいけど全部のボタン編集するの面倒くさすぎる**とかがなくなるので，ぜひ活用してください!

---

#### 🧑‍💻 Author
フライさんです．Twemoji が好きです．
- 🦜 Twitter: https://twitter.com/loxygen_k
- 🐙 GitHub: https://github.com/loxygenK
- 📖 [この記事の GitHub 上の URL](https://github.com/loxygenK/zenn-articles/blob/main/articles/create-friendly-figma.md)

<!-- vim: set wrap: -->
