---
title: "react-beautiful-dnd + TypeScript でそこそこ楽にドラッグアンドドロップでの並び替えをできるようにする"
emoji: "🖱️"
type: "tech"
topics: ["react", "typescript", "dnd"]
published: false
---

# 🖱react-beatiful-dnd?
https://github.com/atlassian/react-beautiful-dnd

`react-beautilful-dnd` は React でドラッグアンドドロップでの並び替えの実装を簡単に済ませることができるようにするライブラリです。
1 リストの中での要素の並び替えはもちろん、複数リスト間での要素の並び替えも簡単に実装することができます!

```bash:Install
# use npm instead if you are using npm
yarn add react-beautiful-dnd
yarn add @types/react-beautiful-dnd -D 
```

どんなことができるのかは、👇の storybook を見るとつかめると思います!
https://react-beautiful-dnd.netlify.com

## ⚙️しくみの簡単な説明
主に、`react-beautiful-dnd` から提供される `DragDropContext`, `Droppable`, `Draggable` を使用して実装します。 `Droppable` と `Draggable` に 固有 ID を割り振り、それでドラッグされた要素やドラッグ先を識別します。イベントは `DragDropContext` に渡したイベントハンドラでまとめて受け取ります。

リポジトリの README の以下の図がめちゃくちゃわかりやすいです👇
https://github.com/atlassian/react-beautiful-dnd#api-%EF%B8%8F

:::details お手製の詳細図
![](https://storage.googleapis.com/zenn-user-upload/7b100f4d11dfdfe94434d463.png)
:::

# 🧩この記事での環境
```bash
create-react-app zenn-react-beautiful-dnd --template typescript
cd zenn-react-beautiful-dnd
yarn add react-beautiful-dnd styled-components
yarn add @types/react-beautiful-dnd @types/styled-components -D
```
をした環境を記事を書いています。

# 🚚D&D だけできるようにする
`DragDropContext`, `Droppable`, `Draggable` を配置して、とりあえずドラッグとドロップのイベントを受け取れるようにします。

## 最小構成
:::message
`react-beautiful-dnd` を使うためのコンポーネント構造を理解しやすくするために、コンポーネント分けなどはせずにわあああああっと書いています。後ほど分けていきます!
:::
```tsx:App.tsx
import React from 'react';
import { DragDropContext, Draggable, Droppable, DropResult } from "react-beautiful-dnd";
import styled from 'styled-components';

const DroppableWrapper = styled.div`
  margin: 1em;
  padding: 0.5em;

  border: 1px solid #444;
  background-color: #eeeeff;

  width: 33%;
`;

const DraggableWrapper = styled.div`
  margin: 0.5em;
  padding: 3em;

  border: 1px solid #777;
  border-radius: 15px;
  background-color: #f7f7ff;
`;

class App extends React.Component {
  constructor(props: {}) {
    super(props);

    this.handleDragAndDrop = this.handleDragAndDrop.bind(this);
  }

  handleDragAndDrop(result: DropResult) {
    console.log(result);
  }

  render() {
    return (
      <div>
        <DragDropContext onDragEnd={this.handleDragAndDrop}>
          <Droppable droppableId="droppable-1">
            {(provided) => (
              <DroppableWrapper
                {...provided.droppableProps}
                ref={provided.innerRef}
              >
                <Draggable draggableId="draggable-1" index={1}>
                  {(provided) => (
                    <DraggableWrapper
                      {...provided.draggableProps}
                      {...provided.dragHandleProps}
                      ref={provided.innerRef}
                    >
                      Droppable
                    </DraggableWrapper>
                  )}
                </Draggable>
                {provided.placeholder}
              </DroppableWrapper>
            )}
          </Droppable>
        </DragDropContext>
      </div>
    );
  }
}

export default App;
```

`yarn start` すると、ネストされた箱が表示されていると思います。小さい方の箱を動かすとドラッグアンドドロップで動かせることが分かると思います! また、DevTools を開くとイベントの情報が出力されていることも分かると思います。

![](https://storage.googleapis.com/zenn-user-upload/7d617d2ccac3f7bbc4df4e61.gif)
*エモエモアニメーションもデフォルトでついてくる*

### 構造
`react-beautiful-dnd` を使用するときの構造の詳細は以下のようになっています。
/* --- */

`Draggable` と `Droppable` の `children` (必要なタグの中身) は **関数** であることに注意が必要です。

#### 関数の中
関数の引数には、 **`provided`** と **`snapshot`** が渡されます。`provided` は、設定が必要な `props` や `ref` などが、`snapshot` は現在の D&D の状態が保存されています。`snapshot` には、例えば `Draggable` であれば、今ドラッグされている最中か、どの `Droppable` からドラッグされたかなどの情報が渡されます。

`provided` の値は、次のように設定します:

- **`Droppable` の中**
  - `provided.droppableProps` ... `<Droppable>` の中直下のコンポーネントに collapse
  - `provided.innerRef` ... `<Droppable>` の中直下のコンポーネントに引数 `ref` の値として
  - `provided.placeholder` ... `<Droppable> の中直下 **の中** に置く

```tsx:Droppable
<Droppable>
  (provided) => (
    <SomeComponent {...provided.droppableProps} ref={provided.innerRef}>
      {/* ここに Draggable などが入る */}
      {provided.placeholder}
    </SomeComponent>
  )
</Droppable>
```

- **`Draggable` の中**
  - `provided.draggableProps` ... `<Draggable>` の中直下のコンポーネントに collapse
  - `provided.innerRef` ... `<Droppable>` の中直下のコンポーネントに引数 `ref` の値として
  - `provided.dragHandleProps` ... `<Draggable>` の中で、ドラッグハンドルとして使いたいコンポーネントに collapse

:::message
`provided.dragHandleProps` を `<Droppable>` の中直下のコンポーネント **以外** に渡すとそれをドラッグのためのハンドルとして使用することができます。
`<Droppable>` の中直下のコンポーネントに `provided.draggableProps` と一緒に指定すると、そのコンポーネント全体がハンドルになります。
:::

```tsx:Draggable
<Draggable>
  (provided) => (
    <OtherComponent
      {...provided.draggable}
      {...provided.dragHandleProps}
      ref={provided.innerRef}
    >
      {/* ここに何かしらが入る */}
    </OtherComponent>
  )
</Draggable>
```

:::message
👆 をミスると、DevTools の Console タブに以下のようなエラーメッセージが表示されます。
![](https://storage.googleapis.com/zenn-user-upload/b494385007c76c40811075a6.png)
:::