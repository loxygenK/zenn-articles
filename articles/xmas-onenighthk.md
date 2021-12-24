---
title: "彼女はいないかもしれないけど，インターネットには多くの仲間がいる"
emoji: "👨‍👩‍👧‍👦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["event", "hackathon"]
published: true
---

React + Gorilla で，**くりぼっち仲間を慰め合うクリスマスツリー** を作りました．PM 20:00 から開発がスタートし，今の時間は AM 4:37 です．[^1]

# 作ったもの
**くりぼっちのいない人同士が慰め合うためのクリスマスツリー**です．

![](https://storage.googleapis.com/zenn-user-upload/3eea216d835f-20211225.png =300x)
*木*

ここに木が存在します．それとテキストボックスが存在します．

![](https://storage.googleapis.com/zenn-user-upload/dd0fa1b0e541-20211225.png =300x)

このテキストボックスに，慰めの言葉を書き込み，Enter を押してあげると…

![](https://storage.googleapis.com/zenn-user-upload/007bea8a3ec2-20211225.png =200x)
![](https://storage.googleapis.com/zenn-user-upload/2943ed625c35-20211225.png =100x)

**くつ下** が出現しました．クリスマスツリーの飾りです．
このクリスマスツリーの飾りは…

![](https://storage.googleapis.com/zenn-user-upload/21e6a40ac3ac-20211225.png)

**慰めの言葉になっているのでした**．(ここで慰めの言葉へのリアクションを飛ばすこともできます．)

このクリスマスツリーは，慰めの言葉で飾られていく **クリボッチ仲間を慰め合うクリスマスツリー**です．

# サイト
**CORS と Cloud Run と Docker と docker-compose その他諸々に殴られて大変な目に合ってしまい，今の時間がハッカソン終了の 4 分前なのでサイトを貼り付けておきます…**．

https://frontend-theta-henna.vercel.app/

運営の人に許可が降りれば，別の記事などの形で詳細を綴ろうと考えています．

# 技術スタック

- フロントエンド: **React** (TypeScript)
- バックエンド: **Gorilla** (Golang) + Docker
- インフラ: GCP Cloud Build + Cloud Run

データベースを用いた永続化は**行っていません**．インメモリで永続化を行っているのでコンテナが止まると内容が揮発してしまいます．これについての言い訳は後述したいと思います．

[^1]: 助けてください
