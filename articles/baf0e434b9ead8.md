---
title: "Swing 宣言的UI化計画④ ～Javaなのに名前付き引数に挑戦！～"
emoji: "🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: true
---

## どうしても名前付き引数が欲しい！
今回は前回の予告を変更し、先に避けては通れないアノ問題に対峙していこうと思います。

前回もちょっと触れたのですが…

> 次はパディングです。これも`Widget`インターフェースで定義してあるメソッドです。ボタン内側の余白を作ります。`SwiftUI`では上下左右のそれぞれに余白の値を設定することができますが、`SwingUI`では現状、上下左右に同じ値の余白しか作りません。まぁその内、作るつもりです、たぶん…

これまで上下左右に同じパディングしか与えない`Widget#padding(int)`のみの定義だったのは、どうしても面倒が先に立って、それを乗り越える気力が湧かなかったのが原因でした。とにかく面倒なんです。何が面倒かと言うと…

ところで似たモノとして、コンポーネント周りの余白ではなく、コンポーネント間の余白を確保する`Spacer`がありました。
この`Spacer`に余白領域をサイズ指定するには固定値の幅と高さ、もしくは幅いっぱい/高さいっぱい、という可能な限り最大化する指定方法がありました。基本的にこれらのパターンなのですが、実際にメソッドとして対応するには…

```Java
1. Spacer.of(int width, int height);    // 幅・高さで指定
2. Spacer.of(UISize width, int height)  // 幅が領域いっぱい、高さは固定値で指定
3. Spacer.of(int width, UISize height)  // 幅は固定値で、高さが領域いっぱい指定
4. Spacer.widthOnly(int width);         // 幅だけ指定
5. Spacer.widthOnly(UISize width);      // 幅だけ領域いっぱい指定
6. Spacer.heightOnly(int height);       // 高さだけ指定
7. Spacer.heightOnly(UISize height);    // 高さだけ領域いっぱい指定
8. Spacer.fill();                       // 幅・高さを領域いっぱい指定
```

これだけの公開メソッドがありました。特に`widthOnly`,`heightOnly`ですね。幅のみの指定とか、高さのみの指定、というメソッドが必要となり、これがメソッドを増やしてしまう原因なのです。たとえ幅だけの指定であっても`of()`で済ませたいんですよね。`Spacer.of(width: 10)`みたいな感じで。でも`Java`には名前付き引数の機能がない（加えて引数の初期値指定ができない）ので困って苦し紛れに幅のみのメソッドとか作っちゃうんです。でもこれが上下左右のパディングなんてなったら大変なのは火を見るよりも明らか… これが上下左右別々のパディング指定を実現させることを遠ざけてた原因なんです。しかし、これはいつまでも引き延ばせないんです。だから今回、やりますよ！

今回の目標は、**名前付き引数のように、見た目でどの箇所の余白を指定しているかが分かり、指定されなかった箇所はデフォルト値が適用される**です。

では実際に`Widget#padding()`で挑戦してみましょう。とは言え`Java`に名前付き引数の機能がない以上、全く同じレベルの機能は作れません。ということで、こんな感じで実現してみるのはどうでしょう。

```Java
Button.of("おーい、ボタンですよ！")
    .padding(Left.of(20), Right.of(40), Bottom.of(60))
```

このように、引数名の代わりにクラスで代替し区別すると、どの箇所にパディング指定するのか分かります。そして、上記には指定されていない`Top`に関してはデフォルト値が適用される、という仕様で、とりあえずOKとしましょう。


## 完ぺきではなくとも、割り切りましょう🤗
さて上記を実現するには、まず引数がいくつ指定されるか不明なので、可変長引数で対応することにします。ただ可変長引数は型が同じでないといけません。そのため、それぞれのクラスには`Gap`という親クラスを持たせ、その型で共通化することにします。

https://github.com/spacehijackle/SwingUI_04/blob/main/app/src/main/java/com/swingui/value/gap/Gap.java

`Gap`クラスは余白のギャップ（間隔）を保持するだけのクラスです。これに`Top`,`Bottom`,`Left`,`Right`のそれぞれのクラスが継承することで、`Gap`型として可変長引数の指定を可能にします。

```Java:Widget.java
public interface Widget<T extends JComponent>
{
    /**
     * パディングの設定をする。
     * 
     * @param gaps 各サイド（left, top, right, bottom）のパディング
     * @return 自身のインスタンス
     */
    T padding(Gap... gaps);

    //// ---------- 後略 ---------- ////
```

で、この`Widget#padding(Gap...)`を`SwingUI`用の各拡張コンポーネントが実装する訳なのですが…

```Java:ButtonWT.java
public class ButtonWT extends JButton implements Widget<ButtonWT>
{
    //// ---------- 中略 ---------- ////

    @Override
    public ButtonWT padding(Gap... gaps)
    {
        return WidgetHelper.padding(this, gaps);
    }

    //// ---------- 後略 ---------- ////
```

実装内容はコンポーネント共通なので、ヘルパークラスにお願いします。

```Java:WidgetHelper.java
public class WidgetHelper
{
    /**
     * 指定コンポーネントのパディングの設定をする。
     * 
     * @param <T> JComponentの継承クラス
     * @param target 対象コンポーネント
     * @param gaps 四方（left, top, right, bottom）のパディング
     * @return 対象コンポーネント
     */
    public static <T extends JComponent> T padding(T target, Gap... gaps)
    {
        // 四方のパディング取得
        AllSidesGap sides = AllSidesGap.of(gaps);

        // パディング設定
        target.setBorder
        (
            BorderFactory.createCompoundBorder
            (
                target.getBorder(),
                BorderFactory.createEmptyBorder
                (
                    sides.top.gap, sides.left.gap, sides.bottom.gap, sides.right.gap
                )
            )
        );
        return target;
    }
}
```

重要なのはパディングを作る仕組み、ではなく、上下左右のパティングの値を決定する仕組みです。メソッドの最初に`AllSidesGap`を作成している箇所がありますが、このクラスによって上下左右のパディング値を決定します。

```Java:AllSidesGap.java
public class AllSidesGap
{
    public final Left left;     // 左側の間隔
    public final Top top;       // 上部の間隔
    public final Right right;   // 右側の間隔
    public final Bottom bottom; // 下部の間隔

    private AllSidesGap(Left left, Top top, Right right, Bottom bottom)
    {
        this.left   = left;
        this.top    = top;
        this.right  = right;
        this.bottom = bottom;
    }

    /**
     * 指定された間隔値で {@code AllSidesGap} を生成する。
     * 
     * @param gaps 間隔値（Left, Top, Right, Bottom）
     * @return {@code AllSidesGap}
     */
    public static AllSidesGap of(Gap... gaps)
    {
        // 四方のパディング決定
        // ※デフォルト値を設定した後、指定された値で上書き
        AllSidesGap paddings = defaults();
        Left   left   = paddings.left;
        Top    top    = paddings.top;
        Right  right  = paddings.right;
        Bottom bottom = paddings.bottom;
        for(Gap gap : gaps)
        {
            if(gap instanceof Left)   left   = (Left)gap;
            if(gap instanceof Top)    top    = (Top)gap;
            if(gap instanceof Right)  right  = (Right)gap;
            if(gap instanceof Bottom) bottom = (Bottom)gap;
        }

        return new AllSidesGap(left, top, right, bottom);
    }

    /**
     * デフォルトの間隔値で {@code AllSidesGap} を生成する。
     * 
     * @return {@code AllSidesGap}
     */
    public static AllSidesGap defaults()
    {
        Left   left   = Left.of(UIDefaults.COMPONENT_GAP);
        Top    top    = Top.of(UIDefaults.COMPONENT_GAP);
        Right  right  = Right.of(UIDefaults.COMPONENT_GAP);
        Bottom bottom = Bottom.of(UIDefaults.COMPONENT_GAP);

        return new AllSidesGap(left, top, right, bottom);
    }

    //// ---------- 後略 ---------- ////
```

`AllSidesGap.of(Gap...)`を見てください。まず、上下左右のデフォルト値を設定し、可変長引数で指定された箇所のみ上書きします。これで上下左右のパディング値を決定することができました。簡単でしょ？面倒だけど…

現状の実装の問題点としては、同じ個所のパディング値を複数指定してもエラーが発生しないこと。やったとしても何事もなかったかのように後勝ちです。まぁ、良いのではないでしょうか。大目に見ましょう😅


## ついでにやっておきましょう。
上記は上下左右のパディング値でしたが、`SwiftUI`でもそうであるように、パディングの指定は上下左右だけでなく、水平方向/垂直方向の指定も可能にします。

```Java
Button.of("牡丹じゃないよ、ボタンだよ")
    .padding(Horizontal.of(30));
```

`Horizontal`/`Vertical`というクラスを使って左右、または上下のパディングを指定します。これらのクラスの親クラスとして`Symmetry`を作成しました。ちなみに発音は"シンメトリー"ではなく、"シメトリー"です😝

https://github.com/spacehijackle/SwingUI_04/blob/main/app/src/main/java/com/swingui/value/gap/Symmetry.java

これを基に、`Widget#padding(Symmetry...)`の定義を追加します。

```Java:Widget.java
public interface Widget<T extends JComponent>
{
    /**
     * パディングの設定をする。
     * 
     * @param gaps 各サイド（left, top, right, bottom）のパディング
     * @return 自身のインスタンス
     */
    T padding(Gap... gaps);

    /**
     * 水平, 垂直方向のパディングの設定をする。
     * 
     * @param symmetries 水平, 垂直方向のパディング
     * @return 自身のインスタンス
     */
    default T padding(Symmetry... symmetries)
    {
        AllSidesGap sides = AllSidesGap.of(symmetries);
        return padding(sides.left, sides.top, sides.right, sides.bottom);
    }

    //// ---------- 後略 ---------- ////
```

`AllSidesGap`には`Symmetry`用のメソッドも用意します。しかし、こちらは既に定義済みの`Widget#padding(Gap...)`を使ってのデフォルト実装です。`AllSidesGap.of(Symmetry...)`を追加して終わりです。

```Java:AllSidesGap.java
    /**
     * 指定された間隔値で {@code AllSidesGap} を生成する。
     * 
     * @param gaps 間隔値（Horizontal, Vertical）
     * @return {@code AllSidesGap}
     */
    public static AllSidesGap of(Symmetry... symmetries)
    {
        // 四方のパディング決定
        // ※デフォルト値を設定した後、指定された値で上書き
        AllSidesGap paddings = defaults();
        Left   left   = paddings.left;
        Top    top    = paddings.top;
        Right  right  = paddings.right;
        Bottom bottom = paddings.bottom;
        for(Symmetry symmetry : symmetries)
        {
            if(symmetry instanceof Horizontal)
            {
                Horizontal horizontal = (Horizontal)symmetry;
                left  = Left.of(horizontal.gap);
                right = Right.of(horizontal.gap);
            }
            else if(symmetry instanceof Vertical)
            {
                Vertical vertical = (Vertical)symmetry;
                top    = Top.of(vertical.gap);
                bottom = Bottom.of(vertical.gap);
            }
        }

        return new AllSidesGap(left, top, right, bottom);
    }
```

まぁ`Symmetry`も似たようなもんです。これでとりあえず準備オッケーです。実際に実行してみましょう。

![](/images/articles/baf0e434b9ead8/padding_sample.jpg =350x)
*新しくなったパディング指定*

```Java:Startup.java
    /**
     * パディングのパターンをテストする。
     */
    private void testPaddingPatterns()
    {
        Frame.of
        (
            "SwingUI Padding Sample",

            (f) ->
            {
                f.setResizable(true);  // 画面リサイズ可能
                f.setSize(400, 300);  // 初期画面サイズ指定
            },

            VStack.of
            (
                Spacing.of(24),

                Spacer.fill(),

                Button.of("Padding (←, ↑, →, ↓) : (20, none, 40, 60)")
                    .padding(Left.of(20), Right.of(40), Bottom.of(60))
                    .onClicked(self -> showInfoDialog(self, "Padding on all sides")),

                Button.of("Padding (Horizontal, Vertical) : (30, none)")
                    .padding(Horizontal.of(30))
                    .onClicked(self -> showInfoDialog(self, "Padding on horizontal/vertical sides")),

                Spacer.fill()
            )
        );
    }
```

`Top`や`Vertical`が指定されてなくてもデフォルトのパディングがちゃんと取れてますね。願いが叶ったー、と言いつつ名前付き引数と比較したらアレですけど… まぁ良しとしましょう。使う側からしたら、より良くなったハズですから。。。


## 勢いで他も変えちゃう！
もうこうなったら勢いですからね。同じ問題を持つ`Spacer`も`Widget#frame(int, int)`もやっちゃいましょう。

:::message
以下の説明はクラス名等の見直しにより、書き直しました。
既に書き直し前の説明を読んでしまった方は、記事の最後にある`追記(2026.01.01)`をご覧ください。
:::

`Spacer`は`widthOnly(int)`とか`hightOnly(int)`なんて言う、幅or高さのみを指定するのに専用メソッドを作っていたのが問題でした。なので、これを`Spacer.of()`に一本化します（`Spacer.fill()`除く）。

これまで列挙型として最大限の幅（もしくは高さ）を表していた`UISize`という型がありましたが、これを廃止し、前出の`Gap`のような役割のクラス`UILength`を追加します。

https://github.com/spacehijackle/SwingUI_04/blob/main/app/src/main/java/com/swingui/value/size/UILength.java

子クラスに`Width`,`Height`を作り、これを`Spacer.of()`の引数とします。ちなみに最大限を表す定数`Infinite`を`Width`と`Height`のそれぞれに定義し、これまでの列挙型`UISize.Infinite`は`UISize.Width.Infinite`,`UISize.Height.Infinite`の定数に変更です。

次に先に出てきた`AllSidesGap`のように、幅・高さを保持するクラス`WxHSize`を作成します。ちなみに`WxH`は、よく製品のサイズ表記で使われてるアレをマネています。

https://github.com/spacehijackle/SwingUI_04/blob/main/app/src/main/java/com/swingui/value/size/WxHSize.java

`WxHSize#of()`ですが、先のパディングの例`AllSidesGap`ではデフォルト値を自クラスのメソッドで生成できてたので良かったのですが、`WxHSize`に関しては、これから対応しようとしている`Spacer`と`Widget#frame()`のデフォルト値が異なるため、これらデフォルト値を外部から渡す仕様になっています。

```Java:HxWSize.java
public class WxHSize
{
    //// ---------- 中略 ---------- ////

    /**
     * デフォルト値と指定されたサイズで {@code WxHSize} を生成する。
     *
     * @param defaults 幅・高さのデフォルト値
     * @param lengths 指定されたサイズ
     * @return {@code WxHSize}
     */
    public static WxHSize of(WxHSize defaults, UILength... lengths)
    {
        Width  width  = defaults.width;
        Height height = defaults.height;
        for(UILength length : lengths)
        {
            if(length instanceof Width)  width  = (Width)length;
            if(length instanceof Height) height = (Height)length;
        }

        return new WxHSize(width, height);
    }
}
```

それでは`Spacer`に適用してみます。

```Java:Spacer.java
public class Spacer
{
    /**
     * 指定した幅・高さのスペース領域を確保する。
     * 
     * @param lengths 幅・高さのサイズ
     * @return {@code PanelWT}
     */
    public static PanelWT of(UILength... lengths)
    {
        // 幅・高さスペースの取得
        WxHSize size = WxHSize.of(WxHSize.zero(), lengths);

        // 幅・高さスペースの設定
        if(isInfinite(size.width) || isInfinite(size.height))
        {
            // 幅または高さが最大限の場合、柔軟なスペース領域を確保
            return flexible(new Dimension(size.width.length, size.height.length));
        }
        else
        {
            // 幅・高さに最大限の値を含まない場合、固定のスペース領域を確保
            return fixed(new Dimension(size.width.length, size.height.length));
        }
    }

    //// ---------- 後略 ---------- ////
```

`Spacer`の場合、デフォルト値は幅・高さ共にゼロなので、`SxHSize#zero()`を使ってデフォルト値を与えています。これにより、`WxHSize.of()`は最初に幅・高さのデフォルト値を確保し、第２引数以降で指定された`Width`/`Height`の値が、必要に応じてそのデフォルト値を上書きしている、というパディングの時と同じ流れになります。

それではもうひとつ、`UILength`を使って`Widget#frame(int, int)`を`Widget#frame(UILength...)`に書き換えです。

```Java:Widget.java
public interface Widget<T extends JComponent>
{
    //// ---------- 中略 ---------- ////

    /**
     * 自身のサイズの設定をする。
     * 
     * @param lengths 幅・高さサイズ
     * @return 自身のインスタンス
     */
    T frame(UILength... lengths);

    //// ---------- 後略 ---------- ////
```

このインターフェースの実装は`Widget`を実装している`SwingUI`用コンポーネントが対象です。先にもあったよう、ヘルパークラスにお願いです（ここでは`ButtonWT`を例に説明）。

```Java:ButtonWT
public class ButtonWT extends JButton implements Widget<ButtonWT>
{
    //// ---------- 中略 ---------- ////

    @Override
    public ButtonWT frame(UILength... lengths)
    {
        return WidgetHelper.frame(this, lengths);
    }

    //// ---------- 後略 ---------- ////
```

```Java:WidgetHelper.java
public class WidgetHelper
{
    //// ---------- 中略 ---------- ////

    /**
     * 指定コンポーネントのサイズの設定をする。
     * 
     * @param <T> JComponentの継承クラス
     * @param target 対象コンポーネント
     * @param lengths 幅・高さサイズ
     * @return 対象コンポーネント
     */
    public static <T extends JComponent> T frame(T target, UILength... lengths)
    {
        // 幅・高さ決定
        WxHSize defaults = WxHSize.from(target.getPreferredSize());
        WxHSize size = WxHSize.of(defaults, lengths);

        // サイズ設定
        target.setMaximumSize(new Dimension(size.width.length, size.height.length));
        target.setMinimumSize(new Dimension(size.width.length, size.height.length));
        target.setPreferredSize(new Dimension(size.width.length, size.height.length));
        return target;
    }
}
```

こちらの場合、デフォルト値はその対象コンポーネントの"適切な"サイズです。`Swing`の各コンポーネントは、それぞれの適切なサイズを持っていて、例えばボタンの場合、設定されるボタンの文字列のフォントサイズに応じて算出されます。で、この適切なサイズは`JComponent#getPreferredSize()`（返り値は`Dimension`）で取得できるので、`WxHSize#from(Dimension)`を使ってデフォルト値を生成します。ちなみに`Dimension`は`Swing`で使われる、幅と高さを保持するクラスです。これでデフォルト値に対し、指定された`Width`/`Height`で上書きする、というこれまでの流れと同じになりますね。

それでは、`Spacer`と`Widget#frame(UILength...)`を実際に呼び出して確認です。

![](/images/articles/baf0e434b9ead8/frame_sample01.jpg =350x)
*`Spacer`,`Widget#frame(UISize...)`の確認*

```Java:Startup.java
    /**
     * {@link Spacer} / {@link Widget#frame(UISize...)} のパターンをテストする。
     */
    private void testUISizePatterns()
    {
        Frame.of
        (
            "SwingUI Spacer Sample",

            (f) ->
            {
                f.setResizable(true);  // 画面リサイズ可能
                f.setSize(400, 300);  // 初期画面サイズ指定
            },

            VStack.of
            (
                Spacer.of(Height.Infinite),

                text("── Width / Height Only ──"),

                HStack.of
                (
                    Button.of("Width: 100")
                        .frame(Width.of(100)),

                    Spacer.of(Width.of(30)),

                    Button.of("Height: 40")
                        .frame(Height.of(40))
                ),

                Spacer.of(Height.Infinite),

                text("── Fixed & Infinite ──"),

                Button.of("Width: Infinite, Height: Fixed")
                    .frame(Width.Infinite, Height.of(60))
            )
            .padding(UIDefaults.COMPONENT_GAP)
        );
    }
```

`Spacer`が`Width`のみ、`Height`のみでも`Spacer.of()`で対応できていますね。

ところで一番下のボタンですが、その上の`Spacer`が高さが最大限取っているため、ウィンドウの高さを伸長してもウィンドウの下部に引っ付いています。また、ボタン自身の幅を最大限に設定しているため、ウィンドウの横幅を伸ばしても、その幅に合わせてボタンも伸びます。これでOKですね。

![](/images/articles/baf0e434b9ead8/frame_sample02.jpg =450x)
*ウィンドウを大きくした時の`Spacer`,`Widget#frame(UISize...)`の確認*


## さいごに
今回は長らく後回しにしてきた問題に立ち向かいました。とりあえず当初の目標は達成ですが、実際のところ、やっぱり欲しい**名前付き引数機能**です。導入したところで問題あるんですかね？過去のバージョンは'名前なし'として新しいJREで対応できるでしょ？ま、`SwingUI`は`Java8`以上のサポートを想定しているので関係はないですけどね…

今回のソース一式は[こちら](https://github.com/spacehijackle/SwingUI_04/tree/main)


## 追記 (2026.01.01)
全く元日早々、修正なんてしたくなかったんですが、こればっかりは仕方ありません。

まず、幅と高さを保持する`UISize`を`UILength`にクラス名を変えました。これは`Size`と名付けると、普通、幅と高さの２つの属性を持つクラスを想像するだろう、と考えた結果です。実のところ、属性としては長さだけなんで違和感があったんですね。

それから`WxHSize`を追加しました。当初、`Spacer`や`Widget#frame()`はデフォルト値が違うし、`AllSidesGap`のようなクラスは作らず、それぞれの呼び出し側がデフォルト値とその上書き処理をやればイイじゃん、と思ってたんですが、冷静に考えると、それは違うな… と。つまりデフォルト値に対し、上書くロジックが分散するじゃないか…、と。

Don't Repeat Yourself...


## 次回記事
https://zenn.dev/spacehijackle/articles/9b334991d26717