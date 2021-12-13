---
title: "useReducer で華麗にステート管理をする"
emoji: "🔄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - react
  - reducer
published: false
---

フライさんです．限界開発鯖アドベントカレンダーです．

:::message
この記事では，**Function Component** を取り扱っています．
言語は TypeScript を使っています．
:::

# 🚚 ステート管理
みなさん，ステート管理はしていますか? 僕はよくしています．

## 🔄 useState を使ったステート管理と限界

普通 React の Function Component でステートを管理するとき，**`useState`** を採用することが多いと思います．

```tsx:useState を使ったステート管理．ありふれたやり方
import React from "react";

export const SomeComponent: React.VFC = () => {
  const [text, setText] = React.useState("");

  return (
    <input
      type="text"
      onChange={(e) => setText(e.target.value)}
    />
  );
}
```

ある程度の規模のコンポーネントなら `useState` で困らないと思います．が，これが複雑になってくると少し困ります．

### 複雑なコンポーネント

例えば，**都道府県と市町村区を選択する** コンポーネントを考えてみましょう．都道府県のリストはこちら側で持っていますが，**市町村区は国が提供してくれてる API から取得します**．


:::details シーケンス図です．

![](https://storage.googleapis.com/zenn-user-upload/e210274e21d7-20211213.png)

:::

このコンポーネントを作るにあたっては，以下の 3 つのステート[^1]が必要になりそうです．

- **選択された都道府県**
  デフォルトでは一番上 (1 番目，0-indexed で 0 番目) が選択されている．
- **API から取得した市町村区リスト**[^2]
  まだ取得していない場合は `undefined`．
- **選択された市町村区**
  選択されていない場合は `undefined`．

これを考慮して，**`useState` を用いてコンポーネントを実装する**とこうなります．

@[codesandbox](https://codesandbox.io/embed/setstate-cityselector-zuuux?fontsize=14&hidenavigation=1&theme=dark&view=editor)
:::message
右にあるハンドルを左にドラッグすると実際のコンポーネントが表示されます．
:::

私が問題だなあと考えているのは，「このコード，**ステート管理に費やすコード量が多くない?**」ということです．コード量が多いと単純にコードの見通しが悪くなり，読むのに頭を使い疲れるという問題があります[^3]．またステートの変更が愚直に書かれていて，「何がしたいのか」がひと目でわかりにくいです．

```tsx:上のコードからステートに関わる部分を抽出
const [selectedPrefIndex, setSelectedPrefIndex] = React.useState(0);
const [cities, setCities] = React.useState<City[] | undefined>(undefined);
const [selectedCityIndex, setSelectedCityIndex] = React.useState<number | undefined>(undefined);

React.useEffect(() => {
  setCities(undefined);
  setSelectedCityIndex(undefined);

  fetchCities(selectedPrefIndex).then((c) => {
    setCities(c);
    setSelectedCityIndex(0);
  });
}, [selectedPrefIndex]);
```

このコンポーネントのコードの見通しをよくしたくなったとき，**このステートの変更を別の関数に切り出す** という手法が考えられます．こんな感じで書けるとうれしいです．

```diff tsx:こうやりたい
 const [selectedPrefIndex, setSelectedPrefIndex] = React.useState(0);
 const [cities, setCities] = React.useState<City[] | undefined>(undefined);
 const [selectedCityIndex, setSelectedCityIndex] = React.useState<number | undefined>(undefined);

 React.useEffect(() => {
+  clearCityList();
 
   fetchCities(selectedPrefIndex).then((cities) => {
+    updateCityList(cities);
   });
 }, [selectedPrefIndex]);
```

ただし実際に実装してみようとすると…あまり一筋縄では行かないことがわかります[^4]．それに，上部のステート定義も煩わしく感じられてしまいます．

この，

- **ステート管理を一括で，コンポーネント外のどこかしらでやりたい!**
- **ステート管理を関数呼び出し (的な感じ) でやりたい!**

という願いを叶えるものがあります．**Reducer パターン** と，それを React の Function Component で実現しやすくする **`useReducer` フック**です!


# 🔀 Reducer パターン

![Reducer パターンのアーキテクチャ](https://storage.googleapis.com/zenn-user-upload/1324426552f4-20211213.png)
*いまいちわかりにくい Reducer パターンのアーキテクチャの図*

Reducer パターンの大きな特徴は **reducer** の存在です (名前にもなっています)． reducer に **ステート計算の種類と引数にあたる "Action"** と **現在の "State"** を渡す (**dispatch**) と，それを基に **新しい State** を計算してもらえます．それを新しく state として保存する…という流れになります．

React では，この Reducer パターンを Function Component で取り扱いやすくするために，**dispatch した後の setState まで面倒を見てくれる `useReducer`** というフックが存在します．Function Component で Reducer パターンを用いる場合は，基本的にこのフックを利用することになると思います．

## 🔎 具体例
一度具体的なコードの例を見てみましょう．
このコードは，**-1 と +1，リセットができるカウンタ**を Reducer で実装した例です．

@[codepen](https://codepen.io/loxygenk/pen/ExvBapx?default-tab=js)

:::details setState で実装した例です．
@[codepen](https://codepen.io/loxygenk/pen/QWMXwrB?default-tab=js)
:::

:::message
見やすくするために森羅万象が 1 つのファイルに詰まっていますが，Reducer 部分を別のファイルに切り出すことも可能です．切り出してあげたほうが，レンダー部分とステート管理部分の分離を徹底できていいとおもいます．
:::

### 構造
コードの上から順に，Reducer パターンと useReducer に現れる要素を見てみます．

#### State
State は単純に，コンポーネントが持つ状態を説明しているのみです．
上記のコードでは，同時に初期状態 (`initialState`) も定義しています．

```tsx:State の定義
type State = {
  count: number
};
const initialState: State = {
  count: 0
};
```

#### Action
Action はこのように定義されています．

```tsx:Action の定義
type Action =
  {
    type: "INCREMENT",
    args: { delta: number }
  } |
  {
    type: "DECREMENT",
    args: { delta: number }
  } |
  {
    type: "RESET"
  };
```

一般化するとこのように書き表すことができます．[^5]

```tsx: Action の一般化
type Action = { type: string, args: any };
```

このように，**`type` で計算の種類を定義**して，**`args` でその計算の種類ごとの引数の型を定義**してやることで Action が定義されます．[^6]

#### reducer
実際の Reducer はこのように実装されています．[^7]

```tsx:Reducer の実装
const reducer = (state: State, action: Action): State => {
  switch(action.type) {
    case "INCREMENT":
      return { count: state.count + action.args.delta };
    case "DECREMENT":
      return { count: state.count - action.args.delta };
    case "RESET":
      return { count: 0 };
  }
}
```

返り値が `State` になっていることから，**reducer は State を返している** ということがわかります．また，**Action の `type`** を用いて計算処理を選び，**`args`** から計算に必要な情報を持ってきています．

:::details TypeScript のすごいところ
TypeScript は，文脈から行われる型推論が尋常じゃなくすごいです．

```tsx
// snip
  switch(action.type) {
    case "INCREMENT":
      return { count: state.count + action.args.delta };
// snip
```
このコードの `return` で使われている，`action.args` の型について考えてみましょう．
このとき，`action.type` の値は `INCREMENT` で確定しています．そうすると，`action` の型は…

```tsx
{
  type: "INCREMENT",
  args: { delta: number }
}
```

に決まりそうです．そうすると，`args` の型も `{ delta: number }` になりそうです．

TypeScript はすごいので，**ここまで考えてしっかりと `args` の型を `{ delta: number }` に推論してくれます**．なので，ここで `action.args.foo` などを参照しようとするとちゃんと怒られます．

また…
```tsx
// snip
    case "RESET":
      return { count: 0 };
// snip
```
についても同様で，`action.type` が `RESET` に決まるので，`action` の型は…

```tsx
{
  type: "RESET",
}
```

に決まり，`action.args` の型は **`never`** になります．
:::

#### useReducer と dispatch
```tsx:useReducer と dispatch
const [state, dispatch] = React.useReducer(reducer, initialState);

const handleDecrement = () => {
  dispatch({ type: "DECREMENT", args: { count: 1 } });
};
const handleIncrement = () => {
  dispatch({ type: "INCREMENT", args: { count: 1 } });
};
const handleReset = () => {
  dispatch({ type: "RESET" });
};
```

useReducer で `state` と `dispatch` をもらいます．`dispatch` 関数を呼び出し，そこに **Action** を渡すことで，**自動で `reducer` に `state` を渡してもらうことができ，また自動で計算結果が `state` に反映されます**．

### 構造のまとめ

まとめると，

- **State** 管理したい状態の構造を定義する
- **Action** に `type` と `args` をもたせ，操作とその引数の組を定義する
- **reducer** の中で，Action の `type` を基に処理を選び，`args` を参考に処理をする
- **`useReducer`** で `state` と `dispatch` をもらい，reducer を利用する

というのが Reducer パターンの実装の構造のおおまかなまとめになります．


[^1]: 都道府県リストは，最初からこちら側で持っていて状態が変化し得ないのでステートとしては列挙しませんでした．

[^2]: 最初はこちら側で市町村区のデータを**持っていない状態**ですが，後から取得して**持っている状態**になり，**状態が変化し得る**ので，市町村区リストはステートとして管理しなければなりません．また，都道府県が選択されるたびにリストの中身も**変化**したりします

[^3]: 読んでくださっている方の中にも，ステート管理のコードを見て顔をしかめた方がいるかもしれません．

[^4]: ステートを司るクラスを作ってそれを Function Component で使うという手法が思いつきましたが，「それをするなら Class Component でよくない?」と思ってしまいました．ただ，プロジェクトの中の別のコードが全部 Function Component であるような状況だと，Class Component を書くのは多少勇気がいるかもしれません．

[^5]: `string` は実際には `string` 型リテラルの Union 型になると思います．また，`any` を使っていますが，「ここは何でも良いよ」という意味であって，**`any` を使おうねという意味ではもちろんありません**．

[^6]: このように Action を定義してあげると，TypeScript の文脈に基づく型推論の物凄さをふんだんに活用することができます．後述のコードについている注釈で詳しく述べています．

<!-- vim: set wrap: -->
