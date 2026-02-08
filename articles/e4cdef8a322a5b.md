---
title: "Swing 宣言的UI化計画⑥ ～'再びの'Javaなのに名前付き引数に挑戦！～"
emoji: "🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: false
---

## 気づいちゃいました…🫣

「今回は何をしようかなー」と考えてて、前回紹介したテキスト・コンポーネントに下線や打ち消し線を描画しようと決めました。ただ`Swing`の`JLabel`ではその機能がない（実はHTML記述を使った裏技的!?な方法もあるのですが、自ら線を描画する"正攻法"を採用）ので、それを実現することにしました。

で、その機能をテキスト・コンポーネントである`LabelWT`に追加したとして、当然ながらそれを呼び出すメソッドが必要です。下線においても、打ち消し線においても、線の属性を与えなければなりません。線の属性とは具体的に、線種（スタイル）、線の色、です。つまり呼び出しイメージは以下のようになります。

```Java
    Text.of("アンダーライン")
        .underline(LineStyle.Bold, LineColor.of(Color.red))
```

もちろん、本家`SwiftUI`がそうであるように、デフォルト引数が使いたいワケです。

```Java
    // 線種: 二重線, 色: 黒(デフォルト)
    Text.of("アンダーライン")
        .underline(LineStyle.Double)

    // 線種: 実線(デフォルト), 色: 赤
    Text.of("アンダーライン")
        .underline(LineColor.of(Color.red))

    // 線種: 実線(デフォルト), 色: 黒(デフォルト)
    Text.of("アンダーライン")
        .underline()
```

そうなると、以前の記事にあったよう、「どうしても名前付き引数が欲しい（デフォルト引数が欲しい）」となり、その時のやり方が使えそうです。

```Java
    // 幅: 80, 高さ: 40
    Button.of("Width & Height")
        .frame(Width.of(80), Height.of(40))

    // 幅: 80, 高さ: フォント高に合わせた高さ
    Button.of("Width Only")
        .frame(Width.of(80))
```

では、あの時、どんな仕組みで実現していたか、というと…

```Java:LabelWT.java
    public LabelWT<T> frame(UILength... sizes)
    {
        return WidgetHelper.frame(this, sizes);
    }
```

`frame()`メソッドの引数に注目です。`Width`,`Height`は共通の親クラス`UILength`を持たせ、その親の型で`Java`の可変数引数の機能を使って実現していました。

```Java:UILength
public class UILength
{
    public final int length;

    private UILength(int length)
    {
        this.length = length;
    }

    public static class Width extends UILength
    {
        private Width(int length)
        {
            super(length);
        }

        public static Width of(int length)
        {
            return new Width(length);
        }
    }

    public static class Height extends UILength
    {
        private Height(int length)
        {
            super(length);
        }

        public static Height of(int length)
        {
            return new Height(length);
        }
    }
}
```

今回もコレと同じことを考えたんですが、すぐに行き詰りました。。。

親クラスとして`LineAttr`を作成し、その子クラスに線種（スタイル）を定義し… 「アレっ、線種って列挙型にしたら、継承使えないじゃん！」って気付いたんです。それに…

`Width`,`Height`は共通の値として親クラスに`length`という整数値のフィールドを持っていました。どちらも共通のフィールド変数が使えたからね、それで良かったんです。だけど、「線種と線色、って別々の値じゃん！」となるわけです。

つまり、これまでのやり方は不完全だった、ということです。

そこで改めて考え直しました。共通の型で可変長引数、という基本はそのままに、でも列挙型であろうとも、各々が異なるフィールド変数を持とうとも問題のないやり方… つまりこうです。

列挙型でも対応できるよう、`LineAttr`は親クラスではなく、インターフェースとします。そして、そのインターフェースを各々のクラスが`implements`する形にすれば、問題はクリアできそうです。

```Java:LineAttr
public interface LineAttr
{
    public enum LineStyle implements LineAttr
    {
        Solid,
        Bold,
        Double,
    }

    public enum LineColor implements LineAttr
    {
        public final Color color;  //← 各々クラスの属性は自身で保持

        private LineColor(Color color)
        {
            this.color = color;
        }

        public static LineColor of(Color color)
        {
            return new LineColor(color);
        }
    }
}
```

インターフェースですが、実装すべきメソッドはゼロです。型を揃えるだけですからね。ちなみにインターフェースの中に各々内部クラスを置いていますが、必ずしも中に置く必要はありません。ただ、線属性の一部、といったイメージが伝わりやすいので、中に置いています。

それでは実際にテキストに下線を引くメソッドを見てみましょう。

```Java:LabelWT.java
    public LabelWT<T> underline(LineAttr attr...)
    {
        // デフォルト属性を作成し、指定された属性で上書き
        LineStyle style = LineStyle.Solid;
        LineColor color = LineColor.of(getForeground());
        for(LineAttr attr : attrs)
        {
            if(attr instanceof LineStyle) style = (LineStyle)attr;
            if(attr instanceof LineColor) color = (LineColor)attr;
        }

        // 下線の描画
        return rendering(new UnderlineLabelRenderer(style, color));
    }
```

この辺りの処理は以前と同じです。まずデフォルトの線種、線色を設定し、引数で指定されていたら、その値で上書き、という流れです。

もちろん、言語レベルでデフォルト引数に対応している場合に比べ少々面倒が発生しますが、まぁまぁ許容範囲ではないか、と考えてる次第でして… やっぱ面倒か！😝

ちなみに「デフォルト値作成～引数値で上書き」部分は、その処理を`LineAttr`に移動しても良いか、と思います。特に今回は下線だけでなく、打ち消し線にも対応する予定であり、打ち消し線も「デフォルト値作成～引数値で上書き」まで全く同じ処理です。**DRYの原則**の考え方に従うと、下線でも打ち消し線でも、共通処理として利用できるよう、まとめて実装するほうが良いでしょう。

ただし、スタイルと色の２つの値を返却することになるので、ひとつのクラスにまとめなきゃなりません。`Swift`のようにタプル（新たなクラスを必要とせず、異なる型の複数の値をまとめて一つの値のように扱う構文）があれば… と思ったところでどうしようもありません。。。

```Java:LineAttr.java
public interface LineAttr
{
    // ---------- 中略 ---------- //

    public static class Fields
    {
        /** 線スタイル */
        public final LineStyle style;

        /** 線カラー */
        public final LineColor color;

        private Fields(LineStyle style, LineColor color)
        {
            this.style = style;
            this.color = color;
        }

        public static Fields of(LabelWT<?> label, LineAttr... attrs)
        {
            // デフォルト属性を作成し、指定された属性で上書き
            LineStyle style = LineStyle.Solid;
            LineColor color = LineColor.of(label.getForeground());
            for(LineAttr attr : attrs)
            {
                if(attr instanceof LineStyle) style = (LineStyle)attr;
                if(attr instanceof LineColor) color = (LineColor)attr;
            }

            return new Fields(style, color);
        }
    }
}
```

インターフェースなのに、メソッド定義はひとつもナシで、内部クラスばかり定義するのは邪道だ、と思われる方もいるでしょうね…

さて、このようにしておけば、さきほどの下線を引くメソッドの処理は以下のようになります。

```Java:LabelWT.java
    public LabelWT<T> underline(LineAttr attr...)
    {
        LineAttr.Fields fields = LineAttr.Fields.of(this, attrs);

        // 下線の描画
        return rendering(new UnderlineLabelRenderer(fields.style, fields.color, () -> repaint()));
    }
```

これで、よりパワーアップした「Javaなのに名前付き引数に挑戦！」になりました！


## 肝心の線を引く処理とは…

自分としては既に今回のテーマを終えた気持ちにはなっているのですが、肝心の線の描画について何一つ触れてないも同然なので、ここで説明してきたいと思います。

先ほども言及したように、`Swing`のテキスト・コンポーネント`JLabel`に下線や打ち消し線を描画する処理（メソッド）はありません。ではどうするのか、と言うと、実は`Swing`の各コンポーネントには自由に追加描画できる仕組みが提供されています。この機能を提供するのが`JComponent#paintComponent(Graphics)`です。継承した子クラスがこのメソッドをオーバーライドし、引数の`Graphics`クラスを用いて線などの描画をすることが可能なんです。

```Java:コンポーネントを継承したクラス
    @Override
    protected void paintComponent(Graphics g)
    {
        super.paintComponent(g);  //← この呼び出しにより、テキスト等のコンポーネント描画が行われる。

        // コンポーネント上に対角線（赤線）を描画
        g.setColor(Color.RED);
        g.drawLine(0, 0, getWidth(), getHeight());  //← 左上(x,y)=(0,0)から右下(x,y)=(幅,高さ)まで
    }
```

`LabelWT`では、この仕組みを汎用的に提供するための仕組みを作りました。

```Java:LabelRenderer.java
/**
 * ラベル描画用レンダラー定義
 */
public interface LabelRenderer
{
    /**
     * ラベル下の描画処理
     */
    void paintUnder(LabelWT<?> target, Graphics g);

    /**
     * ラベル上の描画処理
     */
    void paintOver(LabelWT<?> target, Graphics g);
}
```

このインターフェース定義は、引数の`Graphics`を使って、テキスト・コンポーネントに追加描画するためのものです。`paintUnder()`はテキストの下側に描画する際に実装するもので、`paintOver()`はテキストの上に描画する場合に実装します。

```Java:LabelWT.java
    /**
     * レンダラーを設定する。
     * 
     * @param renderer レンダラー
     */
    public LabelWT<T> rendering(LabelRenderer renderer)
    {
        // レンダラー追加
        this.renderers.add(renderer);

        // 再描画
        repaint();

        return this;
    }

    @Override
    public void paintComponent(Graphics g)
    {
        // ラベル下にレンダリング
        renderers.forEach(renderer -> renderer.paintUnder(this, g));

        super.paintComponent(g);

        // ラベル上にレンダリング
        renderers.forEach(renderer -> renderer.paintOver(this, g));
    }
```

`LabelWT#rendering(LabelRenderer)`で`LabelRenderer`の実装を引数で取り、リスト形式で保持した後、再描画（`repaint()`）を呼び出すと、`paintComponent(Graphics)`が呼び出され、リストから取り出した`LabelRenderer`の実装が次々に描画を行います。ちなみに「ラベル下にレンダリング」とありますが、イメージしづらいかもしれません。これは例えば影を描画する場合を想定しています。まず`LabelRenderer#paintUnder()`で先に影を描画した後で`super.paintComponent(Graphics)`が呼び出されることでテキストが描画され、結果的に文字の下に影がある様子が表現されます。

今回の下線や打ち消し線はそれ専用のメソッドを用意し、そのメソッドの中から`rendering(LabelRenderer)`を呼び出すようにしています。もちろん、自ら`LabelRenderer`を実装し、それを`rendering(LabelRenderer)`で直接呼び出す事は可能です。

```Java:LabelWT.java
    /**
     * 下線を引く。
     */
    public LabelWT<T> underline(LineAttr... attrs)
    {
        LineAttr.Fields fields = LineAttr.Fields.of(this, attrs);
        return rendering(new UnderlineLabelRenderer(fields.style, fields.color));
    }
```

以上のような**下ごしらえ**をすることで、冒頭でも記述したような下線の描画が可能になるわけですね。

```Java
    // 線種: 二重線, 色: 黒(デフォルト)
    Text.of("アンダーライン")
        .underline(LineStyle.Double)

    // 線種: 実線(デフォルト), 色: 赤
    Text.of("アンダーライン")
        .underline(LineColor.of(Color.red))

    // 線種: 実線(デフォルト), 色: 黒(デフォルト)
    Text.of("アンダーライン")
        .underline()
```

ちなみに`UnderlineLabelRenderer`は`LabelRenderer`の実装で下線を描画するクラスです。なお、`UnderlineLabelRenderer`の実装は地道な描画処理なので、詳しくは解説しません。

これとは別に、打ち消し線もあります。この実装クラスは`StrikethroughLabelRenderer`で、仕組みは下線描画の場合と同様です。


## 実行してみよう！

それでは実際に下線と打ち消し線を呼び出してみます。

https://github.com/spacehijackle/SwingUI_06/blob/main/app/src/main/java/swingui_06/decoration/StaticTextDecoration.java


![](/images/articles/e4cdef8a322a5b/static01.jpg)

３種の下線と下線+打ち消し線のコンボの例です。最後のコンボはウィンドウ幅に対するテキスト・コンポーネントの幅が狭いために３点リーダーになっていますが、ウィンドウを広げると…

![](/images/articles/e4cdef8a322a5b/static02.jpg)

ちゃんと末尾まで描画されていますね。ちなみにウィンドウ幅によりテキストの描画が３点リーダーになったり、そうでなかったりの調整しているのは`JLabel`元々の機能であり、テキストの幅に応じて線の長さが変わるのは`UnderlineLabelRenderer`や`StrikethroughLabelRenderer`の描画の実装に依っています。

以上、線描画の仕組みを解説しました。これからは更に機能強化です。

`UIValue`を使って動的に線属性を変える処理を実装してみます。そのため、`LabelWT`にメソッドを追加です。

```Java:LabelWT.java
    /**
     * 下線を引く。
     */
    public LabelWT<T> underline()
    {
        return underline(UIValue.of(LineStyle.Solid));
    }

    /**
     * 下線を引く（動的変更不可）。
     */
    public LabelWT<T> underline(LineAttr... attrs)
    {
        return underline(WidgetHelper.wrapInUIValues(attrs));
    }

    /**
     * 下線を引く（動的変更可）。
     */
    public LabelWT<T> underline(UIValue<? extends LineAttr>... attrs)
    {
        LineAttr.Fields fields = LineAttr.Fields.of(this, attrs);
        return rendering(new UnderlineLabelRenderer(fields.style, fields.color, () -> repaint()));
    }
```

３つめのメソッド`LabelWT#underline(UIValue<? extends LineAttr>...)`により、引数に線種や線色を保持した`UIValue`を受け取り、これを基に下線の描画を行います。ちなみに`UnderlineLabelRenderer()`の最後の引数ですが、`UIValue`に保持した線属性に変更があった場合にコンポーネントの再描画をするためのコールバックです。

```Java:UnderlineLabelRenderer.java
    public UnderlineLabelRenderer(UIValue<LineStyle> style, UIValue<LineColor> color, Runnable repaint)
    {
        this.style = style;
        this.color = color;

        // 線種に変更があった際にラベルの再描画するよう、リスナーの設定
        this.style.addValueChangeListener(() -> repaint.run());
        this.color.addValueChangeListener(() -> repaint.run());
    }
```

`LabelWT#underline(LineAttr... attrs)`は先のサンプル・コードにもあったように、`UIValue`を介してではなく、直接線種や線色を指定された場合に対応するために設けてあります。中身の処理は引数で指定された線種等を`UIValue`に包んで`LabelWT#underline(UIValue<? extdends LineAttr>...)`に渡すだけです。汎用化したメソッドで対応しています。

実は、このように同名の２つの可変長引数のメソッドを作ると、引数に何も指定しなかった時（`Text.of("text").underline()`）に`Java`が`LabelWT#underline(LineAttr... attrs)`を呼び出せばイイのか、`LabelWT#underline(UIValue<? extdends LineAttr>...)`を呼び出せばイイのか分からず、エラーになります。このために引数なしメソッド（`LabelWT#underline()`）も必要になります。う～ん、メンドウですね。。。

https://github.com/spacehijackle/SwingUI_06/blob/main/app/src/main/java/swingui_06/decoration/DynamicTextDecoration.java

![](/images/articles/e4cdef8a322a5b/dynamic01.jpg)

線種と線色を`UIValue`経由で引数指定しているので、その変化に応じてテキストの下線が変化します。

![](/images/articles/e4cdef8a322a5b/dynamic02.jpg)
![](/images/articles/e4cdef8a322a5b/dynamic03.jpg)


## さいごに

今回はテキスト・コンポーネントに機能を追加することをメインにしようと考えていたものの、気が付くと`Java`のデフォルト引数が無い問題に再び挑戦するハメになりました。結果的に前回に比べ、より適用範囲が増えたと考えていますが、やはり言語レベルでサポートされないと面倒な事には変わりないです。

もし、今後言語レベルで名前付き引数、デフォルト引数がサポートされたとしても、古いバージョンでコンパイルされたJARライブラリを利用した場合、そのライブラリ（クラスファイル）では引数名の情報は削除されているから、そのライブラリのメソッドは引数名を指定して呼び出すことはできないハズです。そういう事情もあってサポートされない気もしています。まぁ実際のところ、何が障害になっているのか分かりませんが（　＾ω＾）・・・

今回のソース一式は[こちら](https://github.com/spacehijackle/SwingUI_06/tree/main)