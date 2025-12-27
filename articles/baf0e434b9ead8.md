---
title: "Swing 宣言的UI化計画④ ～名前付き引数に挑戦！～"
emoji: "🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: false
---

## どうしても名前付き引数が欲しい！
今回は予定変更し、あの問題に対峙していこうと思います。

前回もちょっと触れたのですが…

> 次はパディングです。これもWidgetインターフェースで定義してあるメソッドです。ボタン内側の余白を作ります。SwiftUIでは上下左右のそれぞれに余白の値を設定することができますが、SwingUIでは現状、上下左右に同じ値の余白しか作りません。まぁその内、作るつもりです、たぶん…

これまで上下左右に同じパディングしか与えない`Widget#padding(int)`のみの定義だったのは、どうしても面倒が先に立って、それを乗り越える気力が湧かなかったのが原因でした。とにかく面倒なんです。何が面倒かと言うと…

ところで似たモノとして、コンポーネント周りの余白ではなく、コンポーネント間の余白を確保する`Spacer`がありました。
この`Spacer`に余白領域をサイズ指定するには固定値の幅と高さ、もしくは幅いっぱい/高さいっぱい、という可能な限り最大化する指定方法がありました。基本的にはこのパターンではあるのですが、実際にメソッドとして対応するには…

```Java
1. Spacer.of(int width, int height);  // 幅・高さで指定
2. Spacer.widthOnly(int width);       // 幅だけ指定
3. Spacer.widthOnly(UISize width);    // 幅だけ領域いっぱい指定
4. Spacer.heightOnly(int height);     // 高さだけ指定
5. Spacer.heightOnly(UISize height);  // 高さだけ領域いっぱい指定
6. Spacer.fill();                     // 幅・高さを領域いっぱい指定
```

これだけの公開メソッドがありました。特に`widthOnly`,`heightOnly`ですかね。幅のみの指定とか、高さのみの指定、というメソッドが必要となり、これが公開メソッドを増やしてしまう原因なのです。幅と高さだけでもこれだけ必要なのに、これが上下左右となればカオスですよ、これは。そしてこの問題の根本は`Java`に名前付き引数の機能がない（加えて引数の初期値指定ができない）ことでした。。。

今回の目標は、**名前付き引数のように、見た目でどの箇所の余白を指定しているかが分かり、指定されなかった箇所はデフォルト値が適用される**です。

では実際に`Widget#padding()`で挑戦してみましょう。とは言え`Java`に名前付き引数の機能がない以上、全く同じレベルの機能は作れません。ということで、こんな感じで実現してみるのはどうでしょう。

```Java
Button.of("おーい、ボタンですよ！")
    .padding(Left.of(20), Right.of(40), Bottom.of(60))
```

このように、引数名の代わりにクラスで代替し区別すると、どの箇所にパディング指定するのか分かります。そして、上記には指定されていない`Top`に関してはデフォルト値が適用される、ということであれば、とりあえずOKとしましょう。


## 完ぺきではなくとも、割り切りましょう🤗
さて上記を実現するには、まず引数にいくつ指定されるか不明なので、可変引数で対応することにします。ただ可変引数は型が同じでないといけません。そのため、それぞれのクラスは`Gap`という親クラスを持たせ、その型で共通化することにします。

https://github.com/spacehijackle/SwingUI_04/blob/main/app/src/main/java/com/swingui/value/gap/Gap.java

`Gap`クラスは余白のギャップ（間隔）を保持するだけのクラスです。これに`Top`,`Bottom`,`Left`,`Right`のそれぞれのクラスが継承することで、`Gap`として可変引数の指定を可能にします。

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

実装内容はコンポーネント共通なので、ヘルパークラスに投げます。

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

`AllSidesGap.of(Gap...)`を見てください。まず、上下左右のデフォルト値を設定し、可変引数で指定された箇所のみ上書きします。これで上下左右のパディング値を決定することができました。簡単でしょ？面倒だけど…

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

`AllSidesGap`には`Symmetry`用のメソッドも用意します。

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

`Spacer`は`widthOnly(int)`とか`hightOnly(int)`なんて言う、幅or高さのみを指定するのに専用メソッドを作っていたのが問題でした。なので、これを`Spacer.of()`に一本化します（`Spacer.fill()`除く）。

これまで列挙型として最大限の幅（もしくは高さ）を表していた`UISize`という型がありましたが、これを前出の`Gap`のような役割のクラスに変更です。

https://github.com/spacehijackle/SwingUI_04/blob/main/app/src/main/java/com/swingui/value/UISize.java

子クラスに`Width`,`Height`を作り、これを`Spacer.of()`の引数とします。

```Java:Spacer.java
public class Spacer
{
    /**
     * 指定した幅・高さのスペース領域を確保する。
     * 
     * @param sizes 幅・高さのサイズ
     * @return {@code PanelWT}
     */
    public static PanelWT of(UISize... sizes)
    {
        // 幅・高さスペースの決定
        Width  width  = Width.of(0);
        Height height = Height.of(0);
        for(UISize size : sizes)
        {
            if(size instanceof Width)  width =  (Width)size;
            if(size instanceof Height) height = (Height)size;
        }

        // 幅・高さスペースの設定
        if(isInfinite(width) || isInfinite(height))
        {
            // 幅または高さが最大限の場合、柔軟なスペース領域を確保
            return flexible(new Dimension(width.length, height.length));
        }
        else
        {
            // 幅・高さに最大限の値を含まない場合、固定のスペース領域を確保
            return fixed(new Dimension(width.length, height.length));
        }
    }

    //// ---------- 後略 ---------- ////
```