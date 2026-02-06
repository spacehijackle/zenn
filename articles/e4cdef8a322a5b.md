---
title: "Swing 宣言的UI化計画⑥ ～'再びの'Javaなのに名前付き引数に挑戦！～"
emoji: "🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: false
---

## 気づいちゃいました…🫣

「今回は何をしようかなー」と考えてて、前回紹介したテキスト・コンポーネントに下線や打ち消し線を描画しようと決めました。ただ`Swing`の`JLabel`ではその機能がない（実はHTML記述を使ったやり方があるのですが、自ら線を描画する"正攻法"を採用）ので、それを実現することとなりました。

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

引数に注目です。`Width`,`Height`は共通の親クラス`UILength`を持たせ、その親の型で`Java`の可変数引数の機能を使って実現していました。

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

親クラスとして`LineAttr`を作成し、その子クラスに線種（スタイル）を定義し… 「アレっ、線種って列挙型にしたら、継承使えないじゃん！」って判明したんです。それに…

`Width`,`Height`は共通の値として親クラスに`length`という整数値のフィールドを持っていました。どちらも共通のフィールド変数が使えたからね、それで良かったんです。だけど、「線種と線色、って別々の値じゃん！」となるわけです。

つまり、これまでのやり方は不完全だった、ということです。

そこで改めて考え直しました。基本的に可変長引数で共通の型、という基本はそのままに、でも列挙型であろうとも、各々が異なるフィールド変数を持とうとも問題のないやり方… つまりこうです。

列挙型でも対応できるよう、`LineAttr`は親クラスではなく、インターフェースとします。そして、そのインターフェースを各々のクラスが実装する形にすれば、問題はクリアできそうです。

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
        public final Color color;

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

インターフェースですが、空メソッドです。型を揃えるだけですからね。ちなみにインターフェースの中に各々の実装クラスを置いていますが、必ずしも中に置く必要はありません。ただ、線属性の一部、といったイメージが伝わりやすいので、中に置いています。

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
        return rendering(new UnderlineLabelRenderer(style, color, () -> repaint()));
    }
```

ここの辺りの処理は以前と同じです。まずデフォルトの線種、線色を設定し、引数で指定されていたら、その値で上書き、という流れです。

もちろん、言語レベルでデフォルト引数に対応している場合に比べ面倒が発生しますが、まぁまぁ許容範囲ではないか、と考えてる次第でして… やっぱ面倒か！

ちなみに「デフォルト値作成～引数値で上書き」部分は、その処理を`LineAttr`に移動しても良いか、と思います。特に今回は下線だけでなく、打ち消し線にも対応する予定であり、打ち消し線も「デフォルト値作成～引数値で上書き」まで全く同じ処理です。**DRYの原則**の考え方に沿うと、下線でも打ち消し線でも、共通処理として利用できるよう、まとめて実装するほうが良いでしょう。

ただし、スタイルと色の２つの値を返すことになるので、ひとつのクラスにまとめなきゃなりません。`Swift`のようにタプル（新たなクラスを作成せず、異なる型の複数の値をまとめて一つの値のように扱う構文）があれば… と思ったところで仕方ありません。。。

```Java:LineAttr.java
public interface LineAttr
{
    // ---------- 中略 ---------- //

    public static class Fields
    {
        /** 線スタイル */
        public LineStyle style;

        /** 線カラー */
        public LineColor color;

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

インターフェースなのに、メソッド定義はひとつもナシで、内部クラスばかり定義するのは気持ち悪い、と思われる方もいるでしょうね…

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

## さいごに
