---
title: "useReducer で華麗にステート管理をする"
emoji: "🔄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - react
  - reducer
  - typescript
published: true
---

フライさんです．
[限界開発鯖アドベントカレンダー](https://qiita.com/advent-calendar/2021/approvers)の 16 日目の記事です．[15 日目は Saigyo_HBK さんの YOLOX+motpyで始めるMultiple Object Tracking(MOT) ](https://qiita.com/Saigyo_HBK/items/1116b87a1bb202d1d680) でした．精度すごい…

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

### 🧵 複雑なコンポーネント

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

@[codesandbox](https://codesandbox.io/embed/cityselector-by-setstate-di524?fontsize=14&hidenavigation=1&module=%2Fsrc%2FcitySelector%2Fview.tsx&theme=dark&view=editor)
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

  fetchCities((selectedPrefIndex + 1).toString().padStart(2, "0")).then((c) => {
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

   fetchCities((selectedPrefIndex + 1).toString().padStart(2, "0")).then((cities) => {
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

## ✨ ここがうれしい Reducer
Reducer はその性質から，**コンポーネントの関数とステート管理を分離できる** という利点があります．そうすると，Function Component 自体の実装が書いてあるファイルの情報量が削減されるほか，**ステート管理のテストも格段にしやすくなります**．

:::message
以降の Reducer を用いた実装でも，**`controller.ts` と `view.ts` にファイルが分かれています**．Embed の左上のハンバーガーメニューをクリックすると，別のファイルの内容を表示することができます．
:::

## 🔎 具体例
一度具体的なコードの例を見てみましょう．
**-1 と +1，リセットができるカウンタ**を Reducer を用いて実装する例を考えてみます．
[Sandbox の URL をここに貼っておきます．](https://codesandbox.io/s/counter-by-setreducer-jvjb2?file=/src/counter/view.tsx)

:::details setState で実装した例です．
@[codesandbox](https://codesandbox.io/embed/counter-by-setstate-w8h08?fontsize=14&module=%2Fsrc%2Fcounter%2FCounter.tsx&theme=dark&view=editor)
:::

### 🪄 ステート管理部分を見てみる

@[codesandbox](https://codesandbox.io/embed/counter-by-setreducer-jvjb2?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fcounter%2Fcontroller.ts&theme=dark&view=editor)

コードの上から順に，Reducer パターンに現れる要素を見てみます．

#### 📦 State
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

#### 📒 Action
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

#### 🍳 reducer
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


#### ✨ 構造のまとめ

まとめると，

- **State** 管理したい状態の構造を定義する
- **Action** に `type` と `args` をもたせ，操作とその引数の組を定義する
- **reducer** の中で，Action の `type` を基に処理を選び，`args` を参考に処理をする

というのが Reducer パターンの実装の構造のおおまかなまとめになります．

### 🎨 表示部分を見てみる
これまで，ステート管理に限定された部分を見てきました．実際にこの Reducer パターンを React の Function Component で使用する際はどのような実装になるのでしょうか．

@[codesandbox](https://codesandbox.io/embed/counter-by-setreducer-jvjb2?fontsize=14&hidenavigation=1&module=%2Fsrc%2Fcounter%2Fview.tsx&theme=dark&view=editor)

このようになります．同じように上から構造を見ていきます．

#### 🙋 `useReducer`
```tsx
const [state, dispatch] = React.useReducer(reducer, initialState);
```

`useReducer` は **Function Component で Reducer パターンを利用する**ときに用いられるフックです．
`reducer` (関数) と `initialState` (初期状態) を渡すことで，**現在の状態が保存される `state`** と **"dispatch" を行う関数 `dispatch`** の 2 つをもらうことができます．

#### 🚀 `dispatch`
```tsx
// dispatch({ type: "DECREMENT", args: { delta: 1 } });
dispatch({
  type: "DECREMENT",
  args: {
    delta: 1
  }
});
```

先述したように，**`dispatch` 関数を用いて "dispatch" を行います**．dispatch を行ったあと，reducer の返り値は **自動で state に反映されて useState と同じように再レンダリングが走ります**．

Dispatch は，**現在の state と，`type` 及び `args` を持つ "Action" を "reducer" に渡す**ことを言うのでした．ここでは，`type` が `"DECREMENT"`, `args` が `{ delta: 1 }` な **Action** を渡して Dispatch を行っていることがわかります．

勘のいい人は，「あれ? **state 渡してないけど**」ということに気づくかもしれません．実は，**dispatch() は自動で現在の state を Action と一緒に reducer に送ってくれます**．なので，開発者は Action を渡せば自動で state も含めた Dispatch を行ってくれる…ということになります．

#### ✨ 構造のまとめ

まとめると，

- **`useReducer`** で `state` と `dispatch` をもらう．
- **`dispatch`** で **Action** を渡し，reducer に dispatch してもらう．
  - reducer が計算した新 State は自動で state に反映され，再レンダリングが走る．
  - `dispatch` を呼んだ時，state は自動で reducer に送られるので開発者は **Action** だけ渡せば良い．

というのが Reducer を `useReducer` を用いて Function Component から使用する際の構造のまとめです．

### 🔬 Reducer パターンの動きをトレースしてみる
ここで，ボタンを押されたときの Reducer パターンの動作をトレースしてみます．[別タブで CodeSandbox を開いていただくといいかもしれません．](https://codesandbox.io/s/counter-by-setreducer-jvjb2?file=/src/counter/view.tsx)

:::message
Reducer わかりましたという方は，もちろんここは飛ばして頂いても大丈夫です．
:::

**カウントが 5 のとき，"Increment" ボタンが押された** としましょう．
このときの State は **`{ count: 5 }`** になります．

"Increment" ボタンは，ここで定義されています．

```tsx:view.tsx L28
<input type="button" value="Increment" onClick={handleIncrement} />
```
ここがクリックされるので `handleIncrement` 関数が呼び出されます．

```tsx:view.tsx L16
const handleIncrement = () => {
  dispatch({ type: "INCREMENT", args: { delta: 1 } });
};
```
`handleIncrement` が取り扱うのは `dispatch` 関数．`type` は `INCREMENT`，`args` は `{ delta: 1 }` な Action が渡されています．
`dispatch` 関数が呼び出されると，**自動で state が補完されて `reducer` が呼び出される**のでした．

ここで舞台は移って `controller.ts` に移ります．

```tsx: controller.ts L17
export const reducer = (state: State, action: Action): State => {
```

ありました．`reducer` 関数です．`state` は `useReducer` によって補完してもらえているので，ここの引数には…

- `state =`**`{ count: 5 }`**
- `action =`**`{ type: "INCREMENT", args: { delta: 1 } }`**

が入ります．

```tsx: controller.ts L18
switch(action.type) {
  case "INCREMENT":
    return { count: state.count + action.args.delta };
  case "DECREMENT":
    return { count: state.count - action.args.delta };
  case "RESET":
    return { count: 0 };
}
```

reducer 関数の中に鎮座していたのは `switch` です．reducer は **`action.type` を基に処理を選ぶ**のでした．
`action.type` の値は `"INCREMENT"` です．そうすると，以下の処理が実行されることになります．

```tsx: controller.ts L20
// case "INCREMENT":
     return { count: state.count + action.args.delta };
```

`state.count` は `5`，`action.args.delta` は `1` です．`5 + 1` は `6` なので，**この reducer 関数からは `{ count: 6 }` が返ります**．

ここで，`dispatch` 関数は `reducer` 関数を呼び出した後，その返り値を **自動的に `state` に反映してくれる**のでした．

というわけで，無事に `{ count: 5 }` は `{ count: 6 }` に更新され，表示上のカウントも 5 から 6 に増えました．めでたしめでたし．

# 🧵 複雑なコンポーネントを Reducer で実装してみる
Reducer パターンを導入する前に提示したコンポーネントを，Reducer パターンで実装してみましょう．
管理する State の内容などは特に setState バージョンと変化はありません．

@[codesandbox](https://codesandbox.io/embed/cityselector-by-setreducer-cmibo?fontsize=14&hidenavigation=1&module=%2Fsrc%2FcitySelector%2Fview.tsx&theme=dark&view=editor)

コード量は増えてしまいました…が，**愚直に書かれたステート操作がなくなった** ので `useState` の例よりもすんなりとコードが読めるのではないかと思います．

:::message
コード内で View が API を取り扱うという設計上まずいことが起こっていますが，簡略化のために直接ここで API を叩いてしまいました…
Reducer パターンの別の要素である「Middleware」を用いると，このような非同期処理の操作も Reducer パターンに入れ込むことができ，View のコードがより見やすくなりますが，今回は `useReducer` の導入に留めたかったのでこのような形になりました．そのうち Middleware についての記事も書くかもしれません．
:::

# 📝 まとめ
### `useReducer` のうれしいところ

- **View から愚直なステート操作のロジックを取り除ける!**
  - 代わりに **Action を渡しての Dispatch** に置き換わる!
- **ステート操作を別ファイルに切り出せる!**
  - Function Component を定義しているファイルの情報量が減る!
  - テストがしやすくなる!

### Reducer パターン

- **reducer** がキモ!
  1. dispatch をする．
      - **操作の種類を表す `type` と操作の引数を表す `args` を持った Action** 
      - 現在の **State**

      これらを reducer に渡すことを **dispatch** という．
  2. `action` を基に，`state` の値をいじり，**新しい State を計算する**．
  3. その State をコンポーネントに反映させる．

- **`useReducer` フック**は **dispatch 周りの面倒を見てくれる**ので便利!

# 👋 おわりに
初めての技術記事で至らぬ点が多すぎると思いますが，ここまで読んでくださりありがとうございました!
なにかご提案やマサカリがあれば飛ばしてくださると勉強になるのでぜひよろしくお願いします 🙏


[^1]: 都道府県リストは，最初からこちら側で持っていて状態が変化し得ないのでステートとしては列挙しませんでした．

[^2]: 最初はこちら側で市町村区のデータを**持っていない状態**ですが，後から取得して**持っている状態**になり，**状態が変化し得る**ので，市町村区リストはステートとして管理しなければなりません．また，都道府県が選択されるたびにリストの中身も**変化**したりします

[^3]: 読んでくださっている方の中にも，ステート管理のコードを見て顔をしかめた方がいるかもしれません．

[^4]: ステートを司るクラスを作ってそれを Function Component で使うという手法が思いつきましたが，「それをするなら Class Component でよくない?」と思ってしまいました．ただ，プロジェクトの中の別のコードが全部 Function Component であるような状況だと，Class Component を書くのは多少勇気がいるかもしれません．

[^5]: `string` は実際には `string` 型リテラルの Union 型になると思います．また，`any` を使っていますが，「ここは何でも良いよ」という意味であって，**`any` を使おうねという意味ではもちろんありません**．

[^6]: このように Action を定義してあげると，TypeScript の文脈に基づく型推論の物凄さをふんだんに活用することができます．後述のコードについている注釈で詳しく述べています．

<!-- vim: set wrap: -->
