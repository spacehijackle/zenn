---
title: "Swing 宣言的UI化計画②"
emoji: "️🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: true
---

## 前回の振り返り
前回は`Swing`の基幹コンポーネントとでも呼ぶべき`JPanel`を拡張し、`PanelWT`を作成して基本的なレイアウトを実現しました。と言いつつ、色々迷走してすみませんでした。

https://zenn.dev/spacehijackle/articles/afbf49830d82f8

今回も始める前にお断りしておきます。。。

`SwiftUI`に近づきたい一心でコンポーネント全体を中央に寄せるように変えたのですが、それをやると、今回のテーマで問題が発生することが… ま、ということです。現状、`VStack`で`.frame(int, int)`を指定しない場合、ウィンドウ中央でなく、上部で偏った状態になります。

![](/images/articles/1ffaaec492b98e/VStack.jpg =350x)
*現状はこれです、の図*

もちろん、この`SwingUI`自体、実験と言いつつ、実際にはある程度できている状態であり、記事化に合わせて都度コードを見直そうというつもりでいましたが、その行為が逆に自分自身を混乱させる結果になってしまった、という次第でありまして…


## 今回の実験
今回取り組むのは、前回に引き続き`PanelWT`になります。前回はこれを使ってコンポーネントを縦/横に並べるベースを作ったのですが、今回はコンポーネント間（もしくはウィンドウ・フレームとの間）の余白を確保する`Spacer`です。例えば先に挙げた図では、コンポーネントが全体的に上部に偏っていますが、これを中心に寄せることを可能にするのが`Spacer`です。`SwiftUI`でも同じ名称の`View`(`Swing`でいう所のコンポーネント)がありますね。アレです。

早速、見ていきましょう。

![](/images/articles/1ffaaec492b98e/SpacerInVStack.jpg =350x)
*`Spacer`でコンポーネント配置を制御！*

縦方向`VStack`では中心に、横方向`HStack`には右に少し偏っていますが、これも中心近くに寄っています。では、この図を実現しているソースを見てみましょう。

https://github.com/spacehijackle/SwingUI_02/blob/main/app/src/main/java/swingui_02/Startup.java

`Startup#buildUpInVertical()`を見てください。前回とほぼ同じソースなのですが、`Spacer`が付く記述がありますよね？これがコンポーネント間（もしくはウィンドウ・フレームとの間）の余白を生じさせています。

まず、図で示すところの上部に大きなグレーの領域がありますが、これが`VStack`以降に最初に出てくる`Spacer.fill()`に対応しています。`Spacer.fill()`は確保可能な領域いっぱいに、自身のサイズを広げようとします。ただし、ここで注意！なのですが、基本的にコンポーネントのサイズはベースとなるレイアウトに依存するため、必ずいっぱいに広がるワケではないのですが、`VStack`で採用している`BoxLayout`は空いた領域に子コンポーネントをいっぱいに広げようとするため、それが可能になっています。なので、正確には、いっぱいに広がろうとする`PanelWT`と`BoxLayout`が組み合わさって可能になる、というのがより正確な表現になると思います。

![](/images/articles/1ffaaec492b98e/SpacerInVStack.jpg =350x)
*再掲:`Spacer`でコンポーネント配置を制御！*

次に文字列`RIGHT`の右側の余白です。これに対応するのは`HStack`内の`Spacer.of(80, 10)`になります。幅80px, 縦10pxで領域が確保されます。これは固定値の領域のため、ウィンドウ・サイズを大きくしても、小さくしても、変化することはありません。

次は`HStack`内の文字列`LEFT`の左側の余白です。これはグレー領域が無いのですが、実際には高さゼロの領域が存在しています。`Spacer.widthOnly(UISize.Infinite)`というのが、それです。`widthOnly`とあるのは幅だけ指定し、高さはゼロ、という意味です。つまり幅はいっぱいに取って、でも高さは他の横並びのコンポーネント次第でイイよ、ということですね。ちなみに`UISize.Infinite`とは列挙型`UISize`に定義した、とにかく空いてる領域いっぱいに伸ばして！という意味の定数です。実際にはソースにもある通り、`Integer.MAX_VALUE`です。

https://github.com/spacehijackle/SwingUI_02/blob/main/app/src/main/java/com/swingui/constant/UISize.java

この`Infinite`は`SwiftUI`にも同様の列挙型があります。いずれにせよ、領域いっぱい、というのは具体的な数値では表現できないので、このような対処をしているわけです。

呼び出し側でメソッドの引数に名前を指定でき、かつデフォルト値を与えることができる言語であれば、固定値指定と同じメソッドを使って`Spacer.of(width: UISize.Infinite)`と記述できるところ（`height`はデフォルト値のゼロ）ではあるのですが、仕方ありません。`Java`では同様の記述ができないため、一々`.widthOnly()`という別メソッドを用意しないといけないのですね。。。

![](/images/articles/1ffaaec492b98e/SpacerInVStack.jpg =350x)
*再掲:`Spacer`でコンポーネント配置を制御！*

最後は`VStack`内の末尾にある、`Spacer.heightOnly(UISize.Infinite)`です。これは先ほどの`Spacer.widthOnly(UISize.Infinite)`の高さバージョンですね。この場合は、幅ゼロで高さが領域いっぱい、ということになります。

ということで、だいたいの余白領域の指定方法は分かったでしょうか？今回の例にはないのですが、もちろん文字列のコンポーネント間にこの`Spacer`を使うことも可能です。


## 核心の`Spacer`探究
`Spacer`の指定方法は、直感的にも分かりやすいんじゃないか、と思います。ではその`Spacer`がどのように実現されているか、を見ていきましょう。

https://github.com/spacehijackle/SwingUI_02/blob/main/app/src/main/java/com/swingui/front/layout/Spacer.java

たくさんメソッドがあって、少々分かりづらいのですが、まず押さえたいのは`Spacer#of(int, int)`です。幅、高さのふたつの引数の内、いずれかが`UISize.Infinite`を含む場合は、`Spacer#flexible(Dimension)`へ、そうでない場合は`Spacer#fixed(Dimension)`へ処理が移ります。ちなみに`Dimension`とは幅と高さを表す`Swing`（正確には`AWT`）のクラスです。単に幅と高さを保持するクラス、と捉えてください。

この移った先のメソッドですが、一つは領域いっぱいに広げるメソッド（→`flexible()`）であり、もう一つは固定領域のメソッド（→`fixed()`）です。この領域自体は`PanelWT`のコンポーネントであり、基本的にこのコンポーネントサイズを指定しているだけ、になります。この両メソッドの違いなのですが、わずかです。

```Java
    private static PanelWT flexible(Dimension dimension)
    {
        PanelWT panel = new PanelWT();
        panel.setMinimumSize(new Dimension(0, 0));
        panel.setMaximumSize(dimension);
        return panel;
    }
```
```Java
    private static PanelWT fixed(Dimension dimension)
    {
        PanelWT panel = new PanelWT();
        panel.setPreferredSize(dimension);
        panel.setMinimumSize(dimension);
        panel.setMaximumSize(dimension);
        return panel;
    }
```

`Swing`において、コンポーネントのサイズ指定は３種類あり、以下のようになります。

|メソッド名|意味|
----|----|
|setMinimumSize(Dimension)|これ以上、小さくしないで！の最小サイズを指定|
|setMaximumSize(Dimension)|これ以上、大きくしないで！の最大サイズを指定|
|setPreferredSize(Dimension)|ちょうど良い最小～最大の間の適切サイズを指定|

これらサイズ指定のメソッドはあくまでも配置されるレイアウト依存ではあるのですが、簡単に使い分けを説明すると、固定サイズを指定したければ、３つのメソッドで指定された同じ`Dimension`のサイズ指定をする。そうでない場合は`setMinimumSize()`で最小領域を、`setMaximumSize()`で指定されたサイズ指定をする。そんな感じです。ちなみに後者で`setPreferredSize()`指定をすると、ウィンドウを大きくしたり小さくしたりする過程で表示がオカシなことになりますのでご注意を（というか、私自身が嵌ってしまいました）…

基本的には上記で表した`Spacer`のメソッドでほぼ十分です。残りのメソッドは利用者の利便性のために設けているものです。`widthOnly(UISize)`, `widthOnly(int)`,`heightOnly(UISize)`,`heightOnly(int)`は幅や高さ、そして領域いっぱいか、固定値か、の指定の違いになります。ちなみに`fill()`は幅、高さ共に`UISize.Infinite`の具体値を`flexible()`に渡しています。


## さいごに
今回は地味だけど重要である余白を確保する仕組みについてご紹介しました。これでコンポーネントの配置はだいたい希望通りになると思います。次回はどうしようかな… ボタンかテキストになるかな、と考えています。より宣言的UIらしくなっていく予定なので、お楽しみに👋🏻
