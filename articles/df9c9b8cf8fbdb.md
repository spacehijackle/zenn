---
title: "Swing 宣言的UI化計画⑦ ～アンケートに欠かせないアレの巻～"
emoji: "🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: false
---

## 人生とは… 詰まるところ選択なのだ！

私はもう人生の後半真っ只中なので、これから大きな選択はあまりない気もするのですが、だがしかし！今後の人生においても何らかの選択の機会は何度も訪れるはずです。そう、人生は選択の連続なのだ！と、少し熱く語ったところで、今回のテーマは **“選択”** です。

`GUI`としての選択と言えば、チェック・ボックス、ラジオ・ボタン、リストですかね。今回はアンケートに欠かせないこれら選択の部品の内、チェック・ボックスとラジオ・ボタンを宣言的UI化してみます。

まず、チェック・ボックスですが、これはそれほど難しくはありません。`JCheckBox`を拡張した`Widget`インターフェース実装の`CheckBoxWT`を作成し、その`CheckBoxWT`を呼び出す`CheckBox`を作成すれば良いのです。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/widget/choice/CheckBoxWT.java

重要な箇所を抽出して解説します。

```Java:CheckBoxWT.java
public class CheckBoxWT<T> extends JCheckBox implements Widget<CheckBoxWT<T>>
{
    // 選択状態
    private UIValue<Boolean> isChecked = new UIValue<>(false);

    // 選択対象保持データ
    private UIValue<T> item;

    // ラベル生成関数
    private Function<T, String> labeling;

    // 対象値変更リスナー（ウィジェット更新の呼出）
    private final ValueChangeListener valChgListener = () -> WidgetHelper.invokeToRefresh(CheckBoxWT.this);

    public CheckBoxWT(UIValue<Boolean> isChecked, UIValue<T> item)
    {
        this(isChecked, item, (t) -> t != null ? t.toString() : "");
    }

    public CheckBoxWT(UIValue<Boolean> isChecked, UIValue<T> item, Function<T, String> labeling)
    {
        super(item != null ? labeling.apply(item.get()) : "", isChecked.get());

        this.isChecked = isChecked;
        this.isChecked.addValueChangeListener(valChgListener);

        this.item = item;
        this.item.addValueChangeListener(valChgListener);

        this.labeling = labeling;
    }
}
```

２つ目のコンストラクタを見てください。第一引数は`UIValue<Boolean>`であり、これはチェックON/OFF状態を表す変数です。`UIValue`を介しているため、例えばロジック側でこの変数に対し、true/falseを設定することで、それに同期してチェック・ボックスの選択状態も変化します。これは後ほどサンプル・プログラムで確認します。

第二引数は`UIValue<T>`であり、チェック・ボックス横の文字列を表しています。基本的には固定値でしょうが、これを動的に変化させる可能性も考慮し、このような型にしました。こちらも後ほどサンプル・プログラムで確認します。

第三引数ですが、これは第二引数で指定された任意型を基にチェック横の文字列が設定できるようにしてあります。`Funciton`は`Consumer`等の仲間である関数型インターフェースで、ここでは指定した任意型を引数とし、その結果を返り値として受けます。この返り値の実装は呼び出し元で行います。

```Java
    CheckBox.of(isChecked, item, (t) -> "label: " + t.toString())
```

例えば上記のように呼び出せば、変数`item`を引数とし（この例では`t`）、これに対し、先頭に`label: `を、残りは`item`の`toString()`を結合し返却することで、チェック・ボックス横の文字列が設定されます。

ちなみに１つ目のコンストラクタでは、この文字列指定が省略され、`item`の`toString()`がデフォルトの実装となっています。

ところで、`Function`を使わず、直接文字列を指定したらエエやん。と思われる方もいるでしょう。私も若干そう思います。ただ、このチェック横の文字列は指定された任意型に紐づいているだろう、という前提に立てば、その任意型の値を基に文字列を指定するよ、って意図がハッキリしませんか？もし`Function`を使わなかったら、間違えて任意型と関係のない文字列を指定してしまう、なんて懸念はありませんか？
…まぁ、この点はまた改めて考えます。

それでは、実際にチェック・ボックスを呼び出すフロント部分を見てみましょう。こちらは既に先ほど例で示したんですけど、`CheckBox`クラスです。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/front/choice/CheckBox.java

こちらは単純に`CheckBoxWT`を呼び出すだけなので、特に詳しい解説は必要ないか、と思います。


## 一方、ラジオ・ボタンは厄介なのだ🥲

ラジオ・ボタンは単一選択なので、何も知らない人からすると、複数選択可能なチェック・ボックスより簡単なんじゃ？と思う人がいるかもしれません。ギターで例えると、メロディを弾くよりコードで弾く方が難しいと思う人がいるように（ちなみに、そう思う理由は、一度に複数の指を使って弦を押さえるためだそう）。

ラジオ・ボタンの厄介なところは、単一選択のためにグルーピングが必要ということです。`Swing`においても`ButtonGroup`クラスを使って、対象となるラジオ・ボタンを`ButtonGroup`に追加することで、単一選択が可能になっています。つまり、単一選択なので、選択中とは別のボタンがクリックされたら、それまで選択されてたボタンが非選択となり、新たにクリックされたボタンが選択される、というグルーピングの仕組みが必要であり、その動作を`ButtonGroup`がやってくれてるんですね。

もちろん`SwingUI`でも`ButtonGroup`を利用する訳ですが、問題は`ButtonGroup`はコンポーネントではない、ということなんですね。宣言的UIではコンポーネントを次々と重ねていく仕組みですから、非コンポーネントである`ButtonGroup`を直接扱うことはできないのです。なので、`SwingUI`では`ButtonGroup`の機能を持つクラスをコンポーネント（あるいはウィジェット）として作成します。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/widget/choice/RadioButtonGroupWT.java

ここでも、重要な部分を抜き出してみました。

```Java:RadioButtonGroupWT.java
public class RadioButtonGroupWT<T> extends JPanel implements Widget<RadioButtonGroupWT<T>>
{
    // 選択値
    private UIValue<T> selected;
	
    // 管理下のラジオボタン
    private List<RadioButtonWT<T>> radios;

    // 対象値変更リスナー（ウィジェット更新の呼出）
    private final ValueChangeListener valChgListener = () -> WidgetHelper.invokeToRefresh(RadioButtonGroupWT.this);

    public RadioButtonGroupWT(UIValue<T> selected, List<RadioButtonWT<T>> radios)
    {
        super(new FlowLayout(FlowLayout.CENTER, 0, 0));

        this.selected = selected;
        this.selected.addValueChangeListener(valChgListener);

        this.radios = radios;

        // ラジオボタンのグループ追加と選択値に合わせてラジオボタンの選択
        ButtonGroup group = new ButtonGroup();
        for(RadioButtonWT<T> radio : radios)
        {
            group.add(radio);
            radio.setSelected(Objects.equals(selected.get(), radio.getItem().get()));
        }
    }
}
```

まず、`RadioButtonGroupWT`を`JPanel`の子クラスとすることで、コンポーネントとして扱えるようにします。あとは、グループ配下の選択値を提供する`UIValue<T>`型の`selected`と、配下にするラジオ・ボタン`RadioButtonWT`をリスト化した`List<RadioButtonWT<T>>`をコンストラクタの引数としています。

このリスト化したラジオ・ボタンをfor文でグルグル回して、`ButtonGroup`に追加していきます。これでグルーピングはＯＫです。また、その中に選択値と合致するラジオ・ボタンが保持する任意データがあれば、選択中のラジオ・ボタンとして設定します。


## さいごに