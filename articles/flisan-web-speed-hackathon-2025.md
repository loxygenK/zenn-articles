---
title: "Web Speed Hackathon 2025 で 10 位を取れたものレギュ落ちした記録"
emoji: "🏎️"
type: "tech"
topics:
  - web
  - performance
  - hackathon
published: true
---

Web Speed Hackathon 2025 にはじめて出場しました!
結果は、*スコアに関しては* 390/1200 点（全体[^net-score-rank] 10 位）という、悪くはない数値でした!  ……ただし、レギュレーションチェックにおいて、**対象 15 人のうち、14 人がまとめて失格となる** [^regulation-check]という大事故を他の 13 名の方と一緒に起こしてしまい、無事順位対象外となりました。南無三。

[^net-score-rank]: スコアだけで比較した場合の順位（レギュレーションチェック完了直前の順位）です。
[^regulation-check]: 300 点以上の提出を対象に、Web ページがデグレしていないかのレギュレーションチェックが行われました。
今回の対象は 15 人だったのですが、*たった一人を除き*、各人が多種多様な理由でレギュ落ちしてしまい、失格となりました。

初めて参加してみたのですが、**非常に楽しかったですし、有意義な経験だったと感じます！**
普段ここまで全力で最適化に走ることもないので、いつもと違う作業感覚でしたし、デスマーチを走る機会もなかなかないので、命を燃やす感覚を久しぶりに感じられました。また、単純な楽しさだけではなく、最適化をする上での重要な技術、考え方などを実際に使って学ぶことができる上、触れたことがない技術スタックが使われていることがあるので、「こんなのあるんだ」というような発見があるので、勉強にもなりました。

**来年もまた出ます!** もしこの記事を読んでくださっている方が出場を迷っていれば、ぜひ出場することをおすすめします!

---

以降は、WSH2025 出場にさしあたって、何をしたかについてご紹介しようと思います。

## 事前準備: 前大会を試しに走る

WSH2024 のスコアサーバがなんとまだ走っており、試しに走ってみることにしました。こんなことをしてみました。

- **余分なリクエストを削りました。**
    - ドデカ preload が入ってたのでやめました。5,000ms でタイムアウトが設定されていましたが、100ms でも普通に命取りなので、かなり意味がない
    - サーバ側で、クライアントからのリクエストに応じて画像を拡大縮小/形式変換していたのをキャッシュしたり、そんなに倍率が変わらない場合はそのまんま返したりしました。
    - アイコンが `*` で `import` されていて、結果 tree-shake できずに全部送られてきていたのでやめました。
    - その他にもいろいろありました。
- **デカリソースを解消しました。**
    - Service Worker 側で並列リクエスト制限があったり JXL をローカルで変換していたりしていたのをやめました。
    - あまりにもデカすぎる base64 が入った SVG やモジュールを解消しました。[^trick]
    - 画像を全部 webp にしました。先述している各種処理も不要になるのでうれしい
    - あまり見られないモーダルが巨大なテキストコンテンツを表示しており、そのコンテンツのダウンロードに時間を要していたので、表示時にだけダウンロードするようにしました。[^preloading-might-helped]
- **余分な依存を解消しました。**
    - three.js をやめました。
    - **AI でファイルタイプを検出する仕組み**が採用されていて、しかも**モデルをローカルに落として、ローカルで解析していた**のでやめました。
    - 大きな Web フォントが配信されていた一方で、使われていなかったのでやめました。
- 正規表現が catastropic backtracking[^catastropic-backtracking] で大変なことになっていたので削りました。

…… ここまでやったところで、**VRT が通らないということに初めて気づきました**。どの変更が問題だったのか見当もつけられないまま、結局直せずにリタイアしてしまいました。だいたい 8 時間のランでしたが普通にめっちゃ疲れました。

[^trick]: `display: none` 付き base64 PNG が SVG に紛れ込んでいたのですが、私は `display: none` に気づかないどころかそっちが本体だと勘違いしてしまい（あまりにでかい base64 に圧倒されてそっちしか見てなかった）、結構なロスタイムとなりました。
[^preloading-might-helped]: これ今気づいたんですが、マウスポイント時にコンテンツをダウンロードするみたいなことができたらかなりいい感じにチューニングできそうですね。できるのかな
[^catastropic-backtracking]: 参照: https://www.regular-expressions.info/catastrophic.html

### 走った意味はあった?

**大有りでした！** やっぱり練習は大事です。

実際に走ってみて、ページを早くするための基礎知識や、スコア計測システムの使い方を学ぶことができました。この知識は当日の安心感に大きく貢献したので、実際に走った意味があったなあと強く感じました!

あわせて、**VRT は変更毎に丁寧に回すべき**という、重要なことを学びました。複数の変更をしてから VRT を回すと、万が一 VRT が落ちた際にどの変更が原因で落ちたのかのトラッキングが極めて困難になり、再度テストを通すまでのコストがかなり大きくかかってしまいます。テストランでは 8 時間の作業の末に VRT の存在に気づいて心が折れてしまいましたが、当日は 1 commit 毎に回そうと強く決意しました。

## Web Speed Hackathon 2025 を実際に走る

:::details 今回のお題 "AREMA" について

AREMA は架空の動画配信サービスです。今回はリポジトリ内にアセットして存在する static な動画ファイル（`.ts`）を配信するという形で擬似的に機能を再現しています。このような機能を備えています。

- 番組表機能
- 動画視聴機能 ... 動画コンテンツをいつでも閲覧できる。
- 番組視聴機能 ... 現在放送されている番組を閲覧できる。
- ログイン機能 ... ログインすると、プレミアム限定動画を閲覧できるようになる。

既存の動画配信プラットフォームとはおそらく関係ないです。たぶん。きっと。 

:::

:::message

以降は概ね時系列順で書いていますが、似たような変更に関する記述をまとめるため、一部時系列から外してまとめています。また、変更による細かいデグレの修正など、一部カバーしていないところがあります。
完全な変更ログ・コードベースは、リポジトリを見ていただければと思います!

https://github.com/loxygenK/web-speed-hackathon-2025

:::

テストランから 3 日後、ついに本大会当日を迎えました。

### 初動
時は 10 時 30 分、Web ページを爆速にする 30 時間の火蓋が切られました。

#### デプロイ

WSH2024 では Koyeb.app を使ってアプリをデプロイしていたので、今回もそうなのかなあと（何故か）思い込み、事前に Koyeb.app のデプロイ画面を準備しておいたのですが、**今回は Heroku でした**。

ところで、Heroku の無料枠は結構前になくなってしまいました。なので、自分の Heroku アカウントでデプロイしようとするとお金がかかってしまいます。そこで、運営様側によって**無料でデプロイできる環境が整えられていました!** 指定のリポジトリに PR を立てると GitHub Action 経由で運営様側の Heroku アカウントでデプロイでき、参加者側での金銭負担はゼロとなるようになっていました。

ただ、ログ確認は運営の方に問い合わせ、というデバッグ作業上のオーバーヘッドがあったのと[^not-that-much-overhead]、一つ基盤を挟むことで何かとイシューのもとになるのでは[^deploy-struggle]…… という不安があったため、私は自身の Heroku アカウントでデプロイをすることにしました。

[^not-that-much-overhead]: 運営の方が常に Ready だったのか、様子を見ているとオーバーヘッドは実際かなり小さそうでした。
[^deploy-struggle]: 実際、この世界に存在する「**本番当日になるとシステムの可用性が著しく低下する**」という魔の現象が再現したようで、運営の方々やシステム利用者がかなり苦しんでいた様子が伺えました。

#### リポジトリをクローンする

**デカい！！** 動画コンテンツがアセットとして存在するのでたぶんこれはしょうがないです。最初 `--depth=1` をつけ忘れて、「そりゃ重いわ」とフラグを付けて再度やり直したのですが、やっぱり重かったです。

#### VRT のスナップショットを取る

先日のテストランで VRT の重要性を学んでいたので、VRT のためのスナップショットを取りました。Arch Linux 上でやっていたせいで最初まともに Playwright 環境が構築できず困っていたのですが、AUR パッケージ `google-chrome` を入れたところ動いてくれました。テストランで Playwright は一回動かしてるはずなんですが、なんでだったんでしょう……

### 動かしてみる

デプロイがちゃんと動き、VRT の準備も整い、とりあえず安心してスタートラインに立つことができたので、実際にアプリケーションの動作の様子を見てみました。

WSH2024 では表示が完了するのを待つ気にもならないほど遅かったのですが、今大会は、たしかに遅いものの**別にレンダリングはされており**、少し拍子抜けしました[^invalid]。
問題点を見つけられるだろうか…… と不安でしたが、Network タブを見てみると、いろいろな問題がすぐに見つかりました。

- HTML も JS もデカい
- API からの JSON がデカい
- SVG がデカい
- JPEG がデカいし直列で転送されて来てる上にレンダリングをブロックしてる

![Network タブの様子。HTML や JS ファイル、JSON などが 50 MB 前後の大きさで転送されていたり、大量の JSON が直列で転送されていたりしています。細部の詳細は後述しています。](/images/2025-03-01_wsh2025/image-1.png)
*いろいろとデカい！！！*

[^invalid]: 感覚がヤバいくらい狂ってる

### 計測できる程度に早くする

最初は **Lighthouse がまともに回せない**レベルで遅いので、とりあえず目につく明らかな問題点を解消して、計測を回せるようにします。

#### Webpack の設定調整 + Bundle Analyzer 導入
ソースマップがインラインされているなどの、比較的簡単に直せるところを直しつつ、Bundle Analyzer を導入しました。実際に見てみると、こんな問題に気づきました。

- **FFmpeg が 40MB バンドルされている!!**
- アイコンセットライブラリ由来の JSON がバンドルされている

![Bundle Analyzer のスクリーンショット。main.js のみが存在しており、その中の 6 割を ffmpeg 関係が占めています](/images/2025-03-01_wsh2025/image.png)
*ffmpeg デカすぎ　てかアイコンセットデカいの何?*

とりあえず脳内のキューに入れておいて、他にやろうとしていたことをやることにしました。

#### preload 解消
直列で JPEG が送られてきてて何だと思ったら preload でした（去年もあった）。応急措置として一旦全部消しました。

#### Wireit を Turborepo へ置換
ところで、WSH2025 のリポジトリ構造はモノリポになっていて、モノリポツールには **Wireit** が使われていました。ただ、設定が悪かったのか、**実行されているコマンドの出力が表示されない挙動になっており**、実行中に何が起こっているかがよくわかりませんでした。普通に設定を見直せば良かった気もするのですが、ChatGPT に「それ仕様なんですよね」と唆されたのもあり、気が狂って **Turborepo に置換しました**。どっちも全く詳しくなかったので置換は Cline にやってもらいました。

### 初回計測

ここらへんで初回の計測を走らせたところ、このようなリザルトでした。

https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/56#issuecomment-2744963702

![計測結果。15 項目ある中で、計測できている項目が 8 項目、計測不可が 7 項目。計測できているものも 100 点満点中 25 点が 2 つ、残りは 0 点台です。](/images/2025-03-01_wsh2025/image-7.png =600x)
*合計 63.25 / 1200.00 点、33 位*

**微妙！** ですが少なくとも、もっと早くできることは明白です！

### とにかく早くする！！

ここからは、思いついたことからいろいろ手をつけていき、たまに計測を回すということをしていました。

#### デカいリソースを潰す

ドデカい SVG がありましたが、何だと思ったら **SVG の中にフォントが Base64 で埋め込まれていました**。

```svg:SVG ペイロードの最初 400 バイト
$ cat ./public/logos/anime.svg | head -c 400
<svg xmlns="http://www.w3.org/2000/svg" width="280" height="60" viewBox="0 0 280 60" preserveAspectRatio="xMidYMid meet">
  <style type="text/css">
    @font-face {
      font-family: '無心';
      src: url('data:font/opentype;base64,T1RUTwANAIAAAwBQQ0ZGIEvXU5QAAp9YADtBrEdTVUItkqCcAAFWvAAArn5PUy8yfVtPnwAAAWAAAABgVk9SR9FnjIgAApeIAAAEKGNtYXDHpsNyAAABwAAAwtRoZWFkEPSENQAAANwAAAA2aGhlYQa2JqEAAAEUAAAA
```

フォントをファイルに起こそうか迷いましたが、ここ以外でフォントは使われてなさそうだったので、SVG をやめてラスタライズすることにしました。Figma に SVG をドラッグアンドドロップしてみると、うまくラスタライズしてくれたので、ひとつひとつ D&D して PNG への変換しました。

JPEG に関しては脳死で Webp に置換したのですが…… 1.2 MB 程度と、まだかなり大きいです。そういうもんなのか？と疑問に思いながら、DevTools で img タグの `src` 属性の値をポイントしたところで、ようやく**画像横幅が 6000px くらいある**ことに気づきました。

![DevTools のスクリーンショット。画像が 6144px あると書いてあります](/images/2025-03-01_wsh2025/image-5.png =500x)
*でっけ*

画像自体と縮めないと、いつまでもサイズがデカいまんまなので、画像を縮めることにしました。FFmpeg を使って、400px、1200px に縮めた画像を作り、不必要に大きい画像を持ってこないようにしました（普通に 1200px 一つ生やせば足りたかも）。

#### JS をチャンク分けする [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/aaa42e8e6e89c41d3cedb04b968c9fda3d2ab049)

静的リソースがデカいのはだいたいどうにかなったので、JS がデカいのをどうにかしようとなりました。とにかく FFmpeg がデカすぎるので、将来的には絶対に剥がす決意を固めつつ、とりあえず応急措置として、せめて不要時には FFmpeg が転送されないように設定することにしました。

FFmpeg がどこで使われているかを見てみると、動画のサムネイル生成で使われていそうです。トップページでは使われないし、ルーティングロジックでも `import()` で dynamic import していますから最初に FFmpeg が降ってくるのは適切ではありません。

ところで、Webpack は既定ではチャンクが分かれるようになっていたはずですが、Bundle Analyzer の画面を見ていると、ただ一つ `main.js` といデカいチャンクが鎮座している構造になっていました。何らかの理由でチャンクを切る設定が有効になっていないと思い、それを治すことにしました。

有効化する方法を Google で調べ、`optimization.splitChunks.chunks` を `all` にするとか変なことをしていましたが…… **実際はチャンク数を絞るためのプラグインが存在するだけ**で、数値を小さくするかプラグインを取り除くだけでチャンク化されるというオチでした。`optimization.splitChunks.chunks` は触る必要ないし、変に触ると動かなくなるというものだったので、こいつはデフォルト値に戻りました。

```diff js:workspaces/client/webpack.config.js
  plugins: [
-   new webpack.optimize.LimitChunkCountPlugin({ maxChunks: 1 }),
+   new webpack.optimize.LimitChunkCountPlugin({ maxChunks: 9999 }),
    new webpack.EnvironmentPlugin({ API_BASE_URL: '/api', NODE_ENV: '' }),
    ANALYZE && new BundleAnalyzerPlugin(),
  ],
```

#### サーバ側でのデータ取得無効化 [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/ee4f39a63ca72cc02d761bb07243f37a81fa8709)

サーバ側でのプリフェッチが中途半端に実装されていたのですが……

- HTML に埋め込まれるデータがめちゃくちゃ重い
- クライアント側での Hydration 処理が未記述で、データが使われない
- 結局同じデータをクライアントで API から取得している

というような状態で、悲しくも無駄になっている状態でした。
動かすようにしても良かったのですが、他にやりたいことがあったので、一旦サーバ側で情報を取得するのをやめて、完全にクライアント側で全部やるようにしました。

#### デカすぎる API レスポンス (RecommendedModule) を削る

先述の通り、**57 MB**というめちゃくちゃデカい JSON を返してくるエンドポイントが存在します。
このエンドポイントから返るオブジェクト **`RecommendedModule`** は、ドメインルール上、3 重ほどの集約構造があるなど大きな構造になっていて、JSON にすると MB 単位になってしまう…… というような状態でした。`RecommendedModule` の形式を根本から見直す…… というところまでは行けなかったのですが、レスポンスの大きさを削るために、いろいろ施策を打ってみました。

##### いらないフィールドを削る [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/9ccd33ef9c2e05c0fe1c68d60ddfe1aabed98a44)

削れないフィールドがないか調べていると、長めの説明文が使われない場面が多いことがわかりました。
その他にも未使用なフィールドがちらほらあったので、**Zod の `.strip()` で無理くりレスポンスを削りました**。

API の型定義は、クライアント・サーバ間で、Zod スキーマの形で共有されており、そのスキーマを調整すればクライアント側の方で過不足がすぐわかるようになっていました。それで型エラーを見つつ、必要なフィールドだけ `pick` して調整しました。結果 1 MB 程度にまで JSON のサイズが落ち込みました（まだデカいけど）。

##### DB からの取得数を絞る [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/724e0ed72be3392ba47859d548f4933b1bfc60b9)

`RecommendedModules` をリスト形式で表示する画面が数枚あり、エンドポイントからは複数の `RecommendedModule` が配列の形で返ってきていました。
一方で、配列の先頭要素だけしか使っていない画面もありました。そこで、エンドポイントに `limit` クエリパラメータを作って、1 つだけ持ってこれるように修正して無駄を省きました。

#### 画像を置換読み込み（あとから結局やめた）

画面範囲外の部分も画像が読み込まれていて重かったので、`react-lazy-load-image-component` で遅延読み込みするようにしました。その後、やりたかったことは `loading="lazy"` で普通に解決できるということを知ってこのライブラリは剥がしてそちらに置換しました。
……ただ、遅延読み込み付きだと VRT 環境下で画像を読み込むのが間に合わなかったので、結局画像遅延はやめました。

#### CSS への実装置換

いくつか、CSS でできるやんってところを JavaScript でやっていたので、修正しました。


- `<Ellipsis />` → `-webkit-line-clamp` [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/984d7dcbf4a414cbd07130fd6787316057dd6eb2)
- `<AspectRatio />` → `aspect-ratio` [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/984d7dcbf4a414cbd07130fd6787316057dd6eb2)
- `<Hoverable />` → `&:hover {}` / `hover:` [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/07aaa0348d14b0075c5ec214b6b802303036df8c)
- `useScrollSnap()` → `scroll-snap` / `scroll-padding` [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/46f22753a94422637e5f8f5784f22dc66f7616fc)

直後の内容と被るのですが、250ms 毎のポーリングが絡む処理だったので、置き換えてあげることで結構軽くなった印象です。

#### 定期実行処理の最適化 

いくつかの処理が 250ms 毎に実行されており、処理に必要なパラメータは変化していないのに無駄に処理をしている場面があったので修正しました。

##### `useCarouselItemWidth()` [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/984d7dcbf4a414cbd07130fd6787316057dd6eb2#diff-f6383cfdc5e6d2682f1dd155306026a7adce1a98c487929a6a71ad11ac6b3474)

AREMA にはカルーセルが存在しており、各要素の大きさは画面サイズで決まります。この要素サイズの計算は JavaScript で実装されており、250ms 毎に再計算されるようになっていましたが、`ResizeObserver` を使い、イベント発火に応じてレイアウト計算を行うように調整しました[^heavier-maybe]。
本当はレイアウト計算処理を CSS で実装できれば良かったのですが、ぱっと上手くできなかったので一旦 JavaScript のロジックをそのまま使いました[^heavy-css-tech]。

[^heavier-maybe]: これなんですが、もともとはたかだか 250ms 毎に行われていたのが、リサイズイベントが発火されるたびに実行されるようになったので、リサイズ中に関しては元よりも実行頻度が高くなったことになります。Debounce とかすべきだったかも……
[^heavy-css-tech]: 競技終了後の解説によると、**メディアクエリの中で、`calc()` を使ってヘビーな算数をする**というすごいことをする必要があったみたいなので、頑張らなくて良かった〜〜となりました。これはできない、、！！

##### `useCurrentUnixtimeMs()` (ref: [#1](https://github.com/loxygenK/web-speed-hackathon-2025/commit/ffb5a98a91ff34bfbff4ea6e6e9ef24713c135f0), [#2](https://github.com/loxygenK/web-speed-hackathon-2025/commit/c46d51832dee742ab879a07dd827978cd36c7286))

現在の時間を state として持って、250ms 毎に状態を変える実装になっていました。
分が変わる毎に処理を実行したいというモチベーションで使われていそうだったので、分毎にレンダリングを発火できるように調整しました。

##### 番組表の列リサイズ処理 [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/ffb5a98a91ff34bfbff4ea6e6e9ef24713c135f0#diff-2cb14e5936a698fd93ae6bd463dbe18cca060f2ab6c137b532a397c9a265381b)

AREMA には、番組表の列を拡縮できるという、便利なのか便利じゃないのかよくわからない機能が存在します。

![AREMA 内に実装されている、番組表の列拡縮について案内する訴求モーダル。](/images/2025-03-01_wsh2025/image-6.png =300x)
*インタラクトしないとわからない仕様に関して、競技者にモーダルで周知するの
かしこいかもと思いました*

番組表の列をリサイズしたとき、各項目内の画像が見切れた場合はそもそも画像を表示しないようにする、という処理が実装されていたのですが、その処理も 250ms で定期実行されていました。こちらも、リサイズ時にだけ発火するように修正しました[^heavier-maybe]。

---

以上に関して、250ms の定期実行を解消しました!

#### UnoCSS の Runtime を剥がす [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/17b31bb1f0f2261874a0b162caf7c0947dc88a11)

UnoCSS でスタイルが当たっていたのですが、静的解析ではなく、ランタイムで className を監視し動的に CSS を生成するという形が取られていました。

最初から CSS が存在するわけではなく、JS での処理で作るので、CSS がない瞬間が最初存在し、**スタイルが当たっていない状態でページが表示される**ということが起こっていました。その上、UnoCSS の設定の中にアイコンセットの設定が存在しており、設定データといっしょにそれがバンドルされる結果、**アイコンセット由来のデカ JSON** がついてきてしまい、バンドルサイズが大きい原因にもなっていました。

静的解析で上手くいかないこともないだろうと考え、UnoCSS Runtime は剥がし、静的生成するように変更しました。UnoCSS の静的生成は `@unocss/cli` に任せ、`build` 時に追加でコマンドを実行するようにしました。ランタイムに依存している部分（→ 動的解析を行っている部分）は、`style` props にに置換しました。

----

ここで一度計測を回してみたところ、**265 点、5 位**という、悪くない数値でした!

https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/56#issuecomment-2745271339

----


#### スキーマ・バックエンドの修正

##### Type Validator 三銃士を解消する [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/341149413fc7287394da3c0a1789befa6b62c323)

`@sinclair/typebox`、 `valibot`、そして `zod` という、**Type Validator 三銃士が全部バンドルされていました**。なんだこれはと思い、各パッケージの使用箇所を見てみると、**Zod スキーマを `@sinclair/typemap` によって TypeBox か ValiBot に変換し（あるいは変換しなかったり）、それを `TypeMap` に変換する**という非常に壮大なことをしていました。もちろん、今回はそんなことすることないので、Zod のスキーマをそのまんま使う形にしました。スキーマ自体は Zod ですべて定義されていたので、その変換処理を取り除くだけで Zod 以外の依存はなくなりました。

…… ところで、同じような三銃士が実はもう一つ存在していました。どうやら、動画プレイヤーライブラリも 3 つ存在したらしく、**動画プレイヤー三銃士**も全部バンドルされていました。Bundle Analyzer にもしっかり、ドデカく表示されていたのですが、**全く気づかず、競技終了まで対応されませんでした**。不覚……

##### Drizzle をバンドルから吹き飛ばす [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/1fb7be24e69c00dedf784e0daf90ac6308e4c47d)

バックエンドで SQLite を触るために Drizzle ORM が採用されていたのですが、**何故かそれがバンドルされていました**。もちろんブラウザでデータベースを操作することはないのでこれは変です。
よく見ると、**スキーマ定義の行のすぐ下で、スキーマと Drizzle ORM でのテーブルスキーマとの矛盾がないかを検査しており**、型上のロジックながら依存が発生しており、バンドルされているようでした。

```typescript:スキーマファイルの一部抜粋
import { createSelectSchema } from 'drizzle-zod';
import { z } from 'zod';

import * as databaseSchema from '@wsh-2025/schema/src/database/schema';

function assertSchema<T>(_actual: z.ZodType<NoInfer<T>>, _expected: z.ZodType<T>): void {}

// このスキーマだけ必要ですが……
const channel = z.object({
  id: z.string().openapi({ format: 'uuid' }),
  logoUrl: z.string().openapi({ example: 'https://image.example.com/assets/d13d2e22-a7ff-44ba-94a3-5f025f2b63cd.png' }),
  name: z.string().openapi({ example: 'AREMA NEWS' }),
});
// ここで使っており、Drizzle ORM への依存が実質的に発生している
assertSchema(channel, createSelectSchema(databaseSchema.channel));

// 以下省略
```

`assertSchema` の定義と参照、`databaseSchema` の `import` をすべて別ファイルに移し、Drizzle ORM を依存の流れから断ち切りました（今思うと、`import type` を使えば足りたかもしれません）。

ちなみに、単純作業だった上に数が多かったので Cline に任せました。ただ、微妙に一発では上手くいかなかったので手作業のほうが早かったかもしれません。

#### gzip 圧縮を有効にする [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/4e4964a11c884129702b51f973f89853ab86f64c)

ここまで来ると、Lighthouse を回せるようになりました。
そこで、Lighthouse を回してみると、「テキスト圧縮しなよ」という指摘がありました。たしかに圧縮されていなかったので、`@fastify/compress` で gzip 圧縮を有効にしました。

ところが、ただミドルウェアを食わせると**サーバから空のレスポンスが返ってくる**ようになってしまい、上手く動きません。いろいろ調べていると、**各エンドポイントで `return` を書くのを忘れていると、レスポンスが空になる**という挙動があるということを知りました。たしかに、各エンドポイントに `return` がありません。

https://github.com/fastify/fastify-compress/issues/237

`return` を書いてあげると無事動くようになりました。

----

ここで Webpack の出力とちゃんと向き合ったところ、かつて 50 MB くらいあったバンドルサイズが、これまでの作業を経て **719 KiB にまで落ちている**ことがわかりました! いいですね。一般論的にはもっと縮めるべきなのですが、それでも相対的に見ると劇的な変化です（だいたい FFmpeg と UnoCSS 吹っ飛ばして数十 MB が消えたからってだけですが）。

![私の Discord でのスクリーンショットです。Webpack の出力のスクリーンショットを添付して喜んでいます。](/images/2025-03-01_wsh2025/image-9.png =400x)
*私用の Discord スレッドで、私が喜んでいる様子*

また、計測を回したのですが、めちゃくちゃ上振れして 500 点台（全体 2 位）になりました[^sudden-raise-might-be-error]。ただ、残念ながらまぐれだったようで、この数値をまた見ることは叶いませんでした。

![私の Discord でのスクリーンショットです。"DUDE I'M VERSTAPPEN AT THIS POINT NGL"（日本語に訳すと、「やばおれフェルスタッペンかも」）と言っています。「合計 547.14 点、暫定 2 位」という旨が書いてあるスクリーンショットが添付されています。](/images/2025-03-01_wsh2025/image-8.png =400x)

[^sudden-raise-might-be-error]: 競技終了数日後の運営の方のツイート曰く、この上振れは React Router の Error Boundary が計測されたことによると思われる、とのことでした。（参照: https://x.com/sor4chi/status/1904760789619400910 ）ただ、エラーの心当たりがなく、詳しい原因は執筆時点でまだわからずです。

----

#### 正規表現修正

無効なメールアドレスが入力された場合にめちゃくちゃ重くなる正規表現が使われていました。`..` がメールアドレスのユーザー名部分（`@` より前）に存在して、その周辺の文字数が多いと重くなるみたいです。

E メールアドレスは Zod の `.email()`、パスワードは同等のよりシンプルな正規表現に置換しました [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/4eca604b1e5cdfd3278c4bfbe7f3d500857139fe)。メールアドレスに関しては、正規表現をまるっきり変えてしまうとレギュ落ちする……? という恐怖があったので、最後の方にだいたい等価でより安全な正規表現に置換しました [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/644f2b0ffe4b4926b90241ef4d0f3a4dc741388f)。

そんなに意味があるわけではないのですが、ベンチマークツール "JS Benchmark" を使って、元の正規表現と新しい正規表現、Zod の `.email()` に関して速度上問題ないか、Catastropic Backtracing が起こってないかを検証しました。やってみたところ、正常なメールアドレスに関しては速度が劣りますが、Catastropic Backtracing は起こってなさそうでした。
なお、Zod の `.email()` はどのメールアドレスに対しても、ただの正規表現に比べると速度面で劣るようでした。ただ、そんな何回も実行するものではないので、変に正規表現を使わずに `.email()` で良かったかなあとは思います。

- [有効なメールアドレスでの計測結果 (jsbenchmark.com)。](https://jsbenchmark.com/#eyJjYXNlcyI6W3siaWQiOiI1TnlIR1BrV0F1cTUxeDZyay1RWVMiLCJjb2RlIjoiL14oW0EtWjAtOV8rLV0rXFwuPykqW0EtWjAtOV8rLV1AKFtBLVowLTldW0EtWjAtOS1dKlxcLikrW0EtWl17Mix9JC9pLnRlc3QoREFUQSkiLCJuYW1lIjoiUmVnZXgiLCJhc3luYyI6bnVsbCwiZGVwZW5kZW5jaWVzIjpbXX0seyJpZCI6InZfZzhsS2hBMjh5OGpCbVV0MTNoRyIsImNvZGUiOiIvXltBLVowLTlfKy1dKyhcXC5bQS1aMC05XystXSspKkBbQS1aMC05XSsoXFwuW0EtWjAtOV1bQS1aMC05LV0qKSpbQS1aXXsyfSQvaS50ZXN0KERBVEEpIiwibmFtZSI6Ik15IFJlZ2V4IiwiYXN5bmMiOm51bGwsImRlcGVuZGVuY2llcyI6W119LHsiaWQiOiJPZlpDaWZRZ21pNlZoXzZuOW9KVWoiLCJjb2RlIjoiei5zdHJpbmcoKS5lbWFpbCgpLnNhZmVQYXJzZShEQVRBKS5zdWNjZXNzIiwibmFtZSI6IlpvZCBQYXJzZXIiLCJhc3luYyI6bnVsbCwiZGVwZW5kZW5jaWVzIjpbeyJ1cmwiOiJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL3pvZEAzLjI0LjIvK2VzbSIsIm5hbWUiOiJ6IiwiZXNtIjp0cnVlfV19XSwiY29uZmlnIjp7Im5hbWUiOiJCYXNpYyBleGFtcGxlIiwicGFyYWxsZWwiOnRydWUsImdsb2JhbFRlc3RDb25maWciOnsiZGVwZW5kZW5jaWVzIjpbXX0sImRhdGFDb2RlIjoicmV0dXJuIFwidmFsaWQtYnV0LXZlZWVlZWVlZWVlZWVlZWVyeS1sb25nLWVtYWlsLWFkZHJlc3NAaGVyZS1pcy1sb29vb29vb29vb25nLXRvby5jb21cIjsifX0) これは元正規表現が一番早い。

- [Catastropic Backtracking が起こる場合の計測結果 (jsbenchmark.com)。](https://jsbenchmark.com/#eyJjYXNlcyI6W3siaWQiOiI1TnlIR1BrV0F1cTUxeDZyay1RWVMiLCJjb2RlIjoiL14oW0EtWjAtOV8rLV0rXFwuPykqW0EtWjAtOV8rLV1AKFtBLVowLTldW0EtWjAtOS1dKlxcLikrW0EtWl17Mix9JC9pLnRlc3QoREFUQSkiLCJuYW1lIjoiUmVnZXgiLCJhc3luYyI6bnVsbCwiZGVwZW5kZW5jaWVzIjpbXX0seyJpZCI6InZfZzhsS2hBMjh5OGpCbVV0MTNoRyIsImNvZGUiOiIvXltBLVowLTlfKy1dKyhcXC5bQS1aMC05XystXSspKkBbQS1aMC05XSsoXFwuW0EtWjAtOV1bQS1aMC05LV0qKSpbQS1aXXsyfSQvaS50ZXN0KERBVEEpIiwibmFtZSI6Ik15IFJlZ2V4IiwiYXN5bmMiOm51bGwsImRlcGVuZGVuY2llcyI6W119LHsiaWQiOiJPZlpDaWZRZ21pNlZoXzZuOW9KVWoiLCJjb2RlIjoiei5zdHJpbmcoKS5lbWFpbCgpLnNhZmVQYXJzZShEQVRBKS5zdWNjZXNzIiwibmFtZSI6IlpvZCBQYXJzZXIiLCJhc3luYyI6bnVsbCwiZGVwZW5kZW5jaWVzIjpbeyJ1cmwiOiJodHRwczovL2Nkbi5qc2RlbGl2ci5uZXQvbnBtL3pvZEAzLjI0LjIvK2VzbSIsIm5hbWUiOiJ6IiwiZXNtIjp0cnVlfV19XSwiY29uZmlnIjp7Im5hbWUiOiJCYXNpYyBleGFtcGxlIiwicGFyYWxsZWwiOnRydWUsImdsb2JhbFRlc3RDb25maWciOnsiZGVwZW5kZW5jaWVzIjpbXX0sImRhdGFDb2RlIjoicmV0dXJuIFwidGhpcy1kb3VibGUtcGVyaW9kLWJlZm9yZS1sb25nLXRleHQuLm1ha2VzLXBhcnNpbmctaGVhdnlAZXhhbXBsZS5jb21cIjsifX0) 元の正規表現は動かずですが、対策した正規表現は耐える

#### スケジューラーを消す [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/0db40c2efd480faa50c7654f7b81a0259b31f642)

処理 (Task) の優先度を元にいい感じにスケジューリングできる Prioritizied Task Scheduling API という概念が存在するらしく、fetch 処理がそれによって 1000ms の delay 付きでスケジューリングされていました。

https://developer.mozilla.org/en-US/docs/Web/API/Prioritized_Task_Scheduling_API

「こんなのあるんだ！」とためになった一方で、**今回は我々の処理をとにかく実行してほしいのでスケジューリングは不要です**。Delay も同じくです！というわけで、スケジューリングは行わず、直接その場で `fetch` を実行するように変更しました。

#### 不要なリクエスト削減 [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/0db40c2efd480faa50c7654f7b81a0259b31f642)

*(上と同じコミットです)*

モーダル内で実行している API 取得処理が、モーダルが非表示なのにもかかわらず実行されていたので、非表示時はモーダルのレンダリング自体を止めて余分な処理を走らせないようにしました[^good-thing-modal-has-no-animation]。

[^good-thing-modal-has-no-animation]: 今回は大丈夫だったのですが、もしモーダルの表示/非表示時にアニメーションがあると、単純にレンダリングを止めるというアプローチが取れなくなります。これは単純に `{ isOpen && <Modal /> }` をすると、閉じるときのアニメーションが実行されないまま DOM から消えてしまい、アニメーションが実装できなくなるためです。AREMA にはそれなかったので、単純なアプローチで解決できました。よかった。

#### サムネイル生成処理をサーバへ移動 [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/a3db0661becefe155e0158805614b28b8469dfd5)

ここで、**クライアント側でしているサムネイル生成処理をサーバに移す**という大仕事に取り掛かりました！

アプリケーション操作内で動画の内容が変わる…… とかはないので、サムネイルをビルド時に生成することにしました。
デプロイ時、Heroku でビルド処理を行っていたので、Heroku の Buildpack に FFmpeg 用のものを追加して使えるようにしました。この時、**メインの環境の buildpack は一番下に配置しないといけない**ということになかなか気づけず、かなり時間を溶かしました。
ただ今考えると、Heroku でサムネイル生成するんじゃなくて、普通に Git に生成結果をコミットすべきだった気がします。なんでそうしなかったんだ

---

ここで再度計測を回したところ、**397.35 点** でした！点数こそ向上していますが、順位は変わらず**5 位**でした。点数のインフレを感じます。

https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/56#issuecomment-2745906011

---

#### `RecommendedModule` を ~~無限スクロール~~ ユーザーインタラクト後に段階的に取得

`RecommendedModule` が複数個配信されている…… ということを先述しましたが、各 `RecommendedModule` はそれなりに縦幅を取るので、First view で見えない `RecommendedModule` がいくつか存在します。初回の fetch は本当に必要なものだけに留めたいです。というわけで、**`IntersectionObserver` を使った無限スクロールの形**で、段階的に `RecommendedModule` を取るようにしました[^observing-last-div] [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/ebf7c7739c15c07f206c03dea5c285d8f1542b5a)。

[^observing-last-div]: 普段、丁寧にリスト最後の要素を頑張って observe…… とかしているのですが、今回は**リストの最後に空の `div` を置き、それを observe する**という、なかなかにオシャレ（個人比）な実装をすることができました。

しかし、VRT で、**スクロール処理からスクショまでが早く、fetch が間に合わない**という問題が起こってしまいました。無限スクロールに関わるパラメータを微調整すればどうにかならないか、、と思いいろいろ試しましたが、普通に無理でした（し、そういう微調整はよくない）。

なので、スクロールされてから取りに行くのではなく、「**ユーザから何らかの操作があった**時点で取りに行く」という挙動に修正しました[(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/dd0adb5e5c33aa346fe821fb50f11d547a7c0b41)。無事 VRT も動くようになりましたし、初回 fetch のペイロード量も減らすことができました！

#### サーバ側でのデータ取得処理の復活 [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/730a9053434284a2956755ce2dbbec28117f895c)

応急措置としてサーバ側でのデータ取得処理を止めていましたが、ここらへんで復活させました。また、Hydration の処理もちゃんと書いて、データを活用できるようにしました。
また、サーバ側で取っていればローカルで取る必要もないので、ローカルで必要ない fetch 処理はしないようにしました。

ところで、とりあえず PoC として一ページだけ復活させていたのですが、その後全ページで有効化することはありませんでした。後述しますが体力が切れてしまって………

#### Cloudflare への移行

ここまで、一通り Heroku で計測をしていたのですが、このあたりで「**サーバがアメリカにあるの、主に静的コンテンツの配信がキツい！！**」と感じ始め、**Cloudflare を噛ませてそこを計測対象としました**。この段階ではキャッシュの設定はしておらず、最後の最後でキャッシュの設定を入れました。

Cloudflare 移行前後でのスコアの比較はしなかったのですが、Cloudflare に移行しても大きなスコアの変化はなく、392.62 点でした。直接 Heroku にアクセスしていたときの最後の計測が 397.35 点なので、-5 点となりました。

https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/195#issuecomment-2745996758

ところで、この計測あたりから、今まで落ちていたユーザーフローテストが通るようになりました！

#### 無駄なグローバル状態の削減 [(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/5bde9e5a3f18cac495cb75348772dfa8d995a9d2)

マウスポインタ位置がグローバルに保存されていたのでやめました。グローバルである必要はないので、グローバルステートとしてではなくて `useState()` の形で保持するようにしています。

#### 仕様に適合するように修正

ページ遷移時のローダーが表示されていないところがあったので修正しました[(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/58c9a6d404a8ff9d0eb5a07c2254660fa6ea2b1b)。もともとは `react-router` の `state` を見ていたのですが、`state` の指定がされていないところがあってワークしていなかったので、`useNavigation().state` を見る形にして修正しました。
ただ、一部、逆に「ローダーを出してはいけない」というところがあったので、その部分に対応するためにロジックをちょっと修正しています[(ref)](https://github.com/loxygenK/web-speed-hackathon-2025/commit/53fb5f5c68e1aebbc3969382b412dbe5e6a5825a)。

### 最終計測

大方以上の作業を行い、競技終了の 45 分前にコードフリーズ、最後の計測に移りました。
何度か計測を行ったのですが、**途中で 40 点下振れるという事故を起こしました**。それなりに後悔しましたが、スコアサーバからの「来年度もよろしくお願いします」というメッセージと捉えて耐えました。

| 試行回数 | 点数 | 順位 | 計測記録 |
| --- | --- | --- | --- |
| (最終計測直前) | 425.71 | 5 | [ref](https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/195#issuecomment-2746059151) |
| #1 | 423 | 7 | [ref](https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/195#issuecomment-2746098665) |
| #2 | 384 | 11 | [ref](https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/195#issuecomment-2746105233) |
| #3 | 391 | 10 | [ref](https://github.com/CyberAgentHack/web-speed-hackathon-2025-scoring-tool/issues/195#issuecomment-2746107931) |

最後に少し上がりましたが、**この計測は競技終了の数秒前**でした。マジで危なかった……

---

以上の流れで最適化を行い、*スコアは* **391.18 点 / 1200 点**、全体 10 位となりました!

### レギュレーションチェック

無事に計測を完了し、疲れた〜〜〜〜とゆっくりしていたのですが**レギュレーションチェックのアナウンスが流れて急にそわそわし始めました**。「レギュチェに時間かかってるのでもう少しお待ち下さい」というアナウンスが流れ、他に WSH に出ていた人と「**だいたいでいいじゃないですかー！！**」と震えていました。そして運命のレギュレーションチェックですが、冒頭で先述したように、**対象 15 人中 14 人がレギュレーションチェック違反となり、私も一緒に落ちました。**

#### 単純な仕様考慮漏れが原因でした

アナウンスされた原因は以下の通りでした:

```markdown
10: loxygenK
番組終了後遷移時にスクロール位置が保存されてない
```

**ああ…… そういえば**。完全に忘れていました。ローダーのステート管理を見直している場合ではなかったようです。
`regulations.md` 3 回くらい読んだつもりだったのですが普通に頭から抜けてました。どうして………

## 競技中のアクシデント … VRT が通らない

基本的にスムーズに最適化を進めることができましたが、途中、テストランで心を折られた、 **VRT が通らなくなってその原因も不明**…… という大アクシデントが起こってしまいました。

#### いつの間にかめちゃくちゃ落ちるようになってた

1 日目の 14:00 (T+4:00) のことでした。**ここまで来て、VRT がいつの間にか通らなくなっていることに気づきました**。さらに悪いことに、コミットごとに VRT を回すのを忘れており、**どの変更が影響したのかがわかりません**。困りました。テストランで学習したはずなのに。
なんとか `git bisect` し続けた結果、**`react-lazy-load-imagae-component` での画像遅延読み込みが悪影響を及ぼしている**ということに気づきました。とりあえず一旦画像遅延読み込みをやめて VRT を通すと、また動くようになりました。**復旧に 3 時間かかりました。**

#### スクショのタイミングが揺れる?

とりあえず VRT は動いたのですが、**画像読み込みが途中の段階でレンダリングされる**など、タイミング的な部分で結果が揺れることが多々ありました。
いろいろためしていると、**`--worker` 数を過大に設定している**ということに気づきました。CPU のコア数を指定していたのですが、ワーカーが多すぎても、高負荷環境下のレンダリングとなってしまい結果が揺れてしまうようです[^too-many-workers]。
いろいろ試している中で、デフォルト値がいい感じの値ということに気づいたので、特に何も指定せずに VRT を回すようにしました。結果、テスト結果はかなり揺れにくくなりましたし、テスト時間も（体感?）早くなり、VRT を回しやすくなりました！

[^too-many-workers]: ブラウザってシングルスレッドなものじゃないので、それはそう……

#### レンダリングが上手くいかない (気がする)

ヘッダーの部分で、透明度が絡むグラデーションが存在していたのですが、この部分に微妙に差異が発生していました。また、上下位置がズレるという差異も存在していました。

これに関しては原因がかなりわからず、最終的に Arch Linux で作業をしているのが悪い？ということを疑ったので、途中から環境を macOS に切り替えました。そこからは VRT 結果は基本的に安定したので、普通に無難な環境で回すのが一番だなぁということをしみじみ感じました。

#### ポストモーテム

それ以降は `--worker` を変に指定することなく、頻繁に VRT を回すようになりました。お陰で、原因不明な VRT 落ちもなくなり、落ちた場合はすぐに原因の特定/修正に移ることができました。
また、途中で、リモートサーバの Arch Linux から環境を macOS に切り替えたことで、**UI モードを活用できるようになりました**。なんで VRT が落ちたかが非常にわかりやすくなりました。

## 振り返り

以上が、Web Speed Hackathon 2025 でやったことでした。

### できなかったこと

いくつか、他の方のお話を伺って発見した中でできなかったことがありました。

- **Webpack の、他バンドラへの移管はできませんでした。**
  Rspack とかに切り替えた方がそれなりにいたみたいです。たしかに、乗り換えればイテレーションが回しやすくなるので、そっちに時間を投資すべきだったかなあと思います。

- **40 MB の m3u8 が存在することに気づきませんでした。**
  全く気づきませんでした……！せっかく頑張って FFmpeg 取っ払ったのに……！

- **ビデオプレイヤー三銃士が存在することに気づきませんでした。**
  前述した通りです。本当に Bundle Analyzer に出てたんですけどね……

- **SSR はできませんでした。**

悔やまれますね……

### 感想

冒頭と内容が被ってしまいますが、**非常に楽しかったです！** 久しぶりの限界開発ができましたし、技術的な面/ソフトスキル的な面の両面で学びがありました。また、開発の中でいくつかドラマもあり（VRT が直せなくなったり、競技数秒前に上振れるとか）、開発で感情が動かされるという、しばらくできなかった経験もすることができました。
また、これで終わり…… のような書き方をしていますが、他の方の参加記を読んだり、感想戦をしたりで、競技終了後も Web Speed Hackathon 2025 / AREMA を通して様々なことを学ばせていただけると思うので、非常に楽しみにしています！このような、複数の方が一つのことに取り組んで知見を共有できるのは、このようなイベントの醍醐味だなあと感じます。似たイベントに ISUCON があるので、WSH を通じてそちらも気になっています!

**もしこれを読んでいる方が参加を迷っているならば、ぜひ参加されることをおすすめします！**

### 学び

先述しましたが、実際に初めて走ってみて、単純な技術スタックだけではなくソフトスキルに関しても様々な学びや気付きを得ることができました。

技術スタックに関しては、例えばこのような学びがありました。

- **Standard Schema**
  複数の Type validator ライブラリがある中で、横断して使えるようにするために統一のインターフェースが仕様として定められているんだそうです。
- **Zustand、並びに他のグローバルステートの運用の仕方**
  `service` とか `store` とか、ディレクトリが切られてしっかり運用されていました。グローバルステートを運用するにあたってのコードベースのイメージが今までなかったので、一つの例としてためになりました。
- **フレームワークがない中での Hydration**
  SSR に関しては実現できず、ちゃんと学べてはいませんが……… 感想戦で取り組もうと思うので、楽しみです。
- **動画プレイヤーライブラリ**
  3 つ紹介されていました。他の方の話によると、「今回は 1 つまとめるべきだけど、業務では動画の種類に応じてプレイヤーを切り替えることもある」んだそうです。
- **Performance / Lighthouse の使い方**
  今まで以上にヘビーにこれらの機能を使いました。**Performance の Call tree** だとか、**Lighthouse の Timespan** とか、いろいろな機能を学べました。

Web Speed Hackathon のコードベースでは、**技術スタックのショーケース……?** と感じるような~~変な~~実装がなにかとみられることがあります。だいたい取り除いてしまいますが、ためにはなります！Web Speed Hackathon のここすきポイントです。

また、ソフトスキル面に関しても、このような気づきを得ることができました。

- **そんなに急ぐことはない**
  ハッカソンだとだいたい時間との勝負が起こりますが、Web Speed Hackathon は、相当高度なことをしなければそのような時間制約はあまり強く起こらないことがわかりました。なので、休憩は普通に取るべきですし、急いで作業をする必要もなく、一つ一つ着実に改善を重ねていくことが大事なのではないかなと思います。
- **不適切な意思決定が何箇所かあった**
  こうして振り返ってみると、サムネイルの毎ビルド生成を始めとして、いろいろな場所で変な決断をしているところがありました。今回の WSH でいろいろ勉強できたので、次回はもっと効率的に進めたいですね。
- **気づくべき問題点に気付けなかった**
  40 MB m3u8 に気付けなかった、、など、痛い見落としがありました。「そんなに急ぐことはない」に関連しますが、どうせ急いでも時間は余るので、丁寧にしっかり検証するべきだなあと感じます。
- **推測するな、計測せよ**
  今回は初めてだったのであまり意識していませんでしたが、前述の「気づくべき問題点に気付けなかった」というのは、この部分の意識欠落も原因としてあるのではないかと考えています。次回以降はもっと計測結果の理解と分析に時間をかけたいですね。
- **寝るべき**
  今回の Web Speed Hackathon では**睡眠を取らずに参加した**のですが、**2 日間体調不良を引きずり大後悔しました**。というか当日も最後の方適切な意思決定ができなかったのでパフォーマンスに悪影響を与えています。絶対寝るべきでし。もしこの記事を読まれている方が出場する際は、ぜひ睡眠を取ってください。

---

以上、はじめての Web Speed Hackathon の参加記でした。
駄文でしたが…… 最後まで読んでいただき大変ありがとうございました！**来年もまた出ます！**
