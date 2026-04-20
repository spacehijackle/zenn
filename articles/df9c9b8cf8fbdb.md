---
title: "Swing 宣言的UI化計画⑦ ～アンケートに欠かせないアレの巻①～"
emoji: "🚀️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [ "Java", "Swing", "GUI", "宣言的UI", "実験" ]
published: true
---

## 人生とは… 詰まるところ選択なのだ！

私はもう人生の後半真っ只中なので、これから大きな選択はあまりない気もするのですが、だがしかし！今後の人生においても何らかの選択の機会は何度も訪れるはずです。そう、人生は選択の連続なのだ！と、少し熱く語ったところで、今回のテーマは **“選択”** です。

`GUI`としての選択と言えば、チェック・ボックス、ラジオ・ボタン、リストですかね。今回はアンケートに欠かせないこれら選択の部品の内、チェック・ボックスとラジオ・ボタンを宣言的UI化してみます。

例えば、↓こんな感じ。

![](/images/articles/df9c9b8cf8fbdb/gender01.jpg)

まず、チェック・ボックスですが、これはこれまで解説してきた`SwingUI`の仕組みを知っていれば難しくはありません。`JCheckBox`を拡張した`Widget`インターフェース実装の`CheckBoxWT`を作成し、その`CheckBoxWT`を呼び出す`CheckBox`を作成すれば良いのです。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/widget/choice/CheckBoxWT.java

まず`CheckBoxWT`です。重要な箇所を抽出して解説します。

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
        this(isChecked, item, (t) -> t.toString());
    }

    public CheckBoxWT(UIValue<Boolean> isChecked, UIValue<T> item, Function<T, String> labeling)
    {
        super(labeling.apply(item.get()), isChecked.get());

        this.isChecked = isChecked;
        this.isChecked.addValueChangeListener(valChgListener);

        this.item = item;
        this.item.addValueChangeListener(valChgListener);

        this.labeling = labeling;
    }
}
```

２つ目のコンストラクタを見てください。第一引数は`UIValue<Boolean>`であり、これはチェックON/OFF状態を表す変数です。`UIValue`を介しているため、例えばロジック側でこの変数に対し、true/falseを設定することで、それに同期してチェック・ボックスの選択状態も変化します（`addValueChangeListener()`に設定された処理により、ON/OFF状態が反映）。これは後ほどサンプル・プログラムで確認します。

第二引数は`UIValue<T>`であり、チェック・ボックスに紐づける任意の値が当てはまります。例えば、ギターの種類に関するチェック・ボックスの場合、ギターの種類を表すオブジェクトが該当します。基本的には固定値でしょうが、これを動的に変化させる可能性も考慮し、`UIValue`を介すことにしました。こちらも後ほどサンプル・プログラムで確認します。

第三引数ですが、これは第二引数で指定された任意型を基にチェック横の文字列が設定できるようにしてあります。`Funciton`は`Consumer`等の仲間である標準で用意されている関数型インターフェースで、ここでは指定した任意型を引数とし、その結果を返り値として受けます。この返り値の実装は呼び出し元で行います。

```Java
    CheckBox.of(isChecked, item, (t) -> "label: " + t.toString())
```

例えば上記のように呼び出せば、変数`item`を引数とし（この例では`t`）、これに対し、先頭に`label: `を、残りは`item`の`toString()`を結合し返却することで、チェック・ボックス横の文字列が設定されます。

ちなみに１つ目のコンストラクタでは、この文字列指定が省略され、`item`の`toString()`がデフォルトの実装となっています。

基本的には第二引数に`String`型を指定すれば、それがそのままチェック横の文字列となるので、通常はそれで良いでしょう。`toString()`の返却値をそのままチェック横の文字列にしたくないケースを考慮し、このような仕組みを設けました。

それでは、今度はチェック・ボックスを呼び出すフロント部分を見てみましょう。こちらは既に先ほどしれっと例で示してたんですけど、`CheckBox`クラスです。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/front/choice/CheckBox.java

こちらは単純に`CheckBoxWT`を呼び出すだけなので、特に詳しい解説は必要ないか、と思います。


## 一方、ラジオ・ボタンは厄介なのだ🥲

ラジオ・ボタンは単一選択なので、GUIプログラミングを知らない人からすると、複数選択可能なチェック・ボックスより簡単なんじゃ？と思う人がいるかもしれません。ギターで例えると、メロディを弾くよりコードで弾く方が難しいと思う人がいるように（ちなみに、そう思う理由は、一度に複数の指を使って弦を押さえるためだそう）。

ラジオ・ボタンの厄介なところは、単一選択のためにグルーピングが必要ということです。`Swing`においても`ButtonGroup`クラスを使って、対象となるラジオ・ボタンを`ButtonGroup`に追加することで、単一選択が可能になっています。簡単に説明すると、単一選択なので、選択中とは別のボタンがクリックされたら、それまで選択されてたボタンが非選択となり、新たにクリックされたボタンが選択される、というグルーピングの仕組みが必要であり、その動作を`ButtonGroup`がやってくれてるんですね。

もちろん`SwingUI`でも`ButtonGroup`を利用する訳ですが、問題は`ButtonGroup`はコンポーネントではない、ということなんですね。宣言的UIではコンポーネントを次々と重ねていく仕組みですから、非コンポーネントである`ButtonGroup`を直接宣言的UIの中で扱うことはできないのです。なので、`SwingUI`では`ButtonGroup`の機能を持つクラスをコンポーネント（`SwingUI`としてはウィジェット）として作成します。

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

まず、`RadioButtonGroupWT`を`JPanel`の子クラスとすることで、コンポーネントとして扱えるようにします。あとは、グループ配下の選択値を提供する`UIValue<T>`型の`selected`と、グルーピング配下にするラジオ・ボタン`RadioButtonWT`をリスト化した`List<RadioButtonWT<T>>`をコンストラクタの引数としています。

このリスト化したラジオ・ボタンをfor文でグルグル回して、`ButtonGroup`に追加していきます。これでグルーピングはＯＫです。また、その中に選択値と合致する任意データ（ラジオ・ボタンが保持している任意データ）があれば、選択中のラジオ・ボタンとして設定します（`RadioButtonWT#setSelected(boolean)`）。

そうそう、`Swing`のラジオ・ボタンである`JRadioButton`の拡張クラス（`SwingUI`用のラジオ・ボタンでもある）`RadioButtonWT`を忘れていました。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/widget/choice/RadioButtonWT.java

基本的に先ほど開設した`CheckBoxWT`とほぼ同じで、異なる点と言えば`CheckBoxWT`は選択用の変数を保持していましたが、ラジオ・ボタンの場合、選択状態は`RadioButtonGroup`が管理しているため、`RadioButtonWT`には選択用の変数を持たせていません。これはラジオ・ボタンをチェック・ボタンのように複数選択可の扱いにすることがなければ、問題はないはずです（そんなこと、しないでしょ？）。


## まだまだ続く、ラジオ・ボタンの厄介

それではラジオ・ボタンとそのグルーピングのフロント、つまり宣言的UIとして呼び出す対象のクラスを見ていきましょう。まず`RadioButtonGroupWT`のフロント、`RadioButtonGroup`です。先ほどの解説を思い出してもらうと、`RadioButtonGroupWT`は選択中を表す変数と、グルーピング配下のラジオ・ボタン`RadioButtonWT`のリストをコンストラクタの引数で受け取っていました。ではフロントもそれと同じ引数にするか？と言えば、それはＮＯです。

もし、下記のようにラジオ・ボタンを利用する際、縦にラジオ・ボタンを並べるというパターンしか考慮しなくても良いのならば、それでも良いでしょう。

![](/images/articles/df9c9b8cf8fbdb/radio_sample01.jpg)

しかし、実際にはラジオ・ボタンを横に並べたい時もあるでしょう。もしくは、ラジオ・ボタンの間に他のコンポーネントを挟み込みたいこともあるでしょう。つまり、フロントである`RadioButtonGroup`は、どのようにラジオ・ボタンが配置されるか？については呼び出し元に委ねるしかありません。だって、どのように配置したいのか分からないのですから。。。

と、言うことで、`RadioButtonGroup`はリスト化したラジオ・ボタンを直接引き受けるのではなく、ラジオ・ボタンを含むコンポーネントのコンテナ、つまり`VStack`や`HStack`でコンポーネントをまとめたコンテナを受け取り、それを`RadioButtonGroupWT`の上に乗せ、加えてコンテナ中に存在するラジオ・ボタンを検索し、それをリスト化の上、`RadioButtonGroupWT`のコンストラクタ引数として渡すのです。実際にソースを見てみると…

```Java:RadioButtonGroup.java
public class RadioButtonGroup
{
    public static <E, T extends JComponent> RadioButtonGroupWT<E> of(UIValue<E> selected, Widget<T> container)
    {
        // 指定コンテナ上のラジオ・ボタンを検索
        RadioButtonFinder<E> finder = new RadioButtonFinder<>();
        finder.find((JComponent)container);

        // 抽出したラジオ・ボタンをグループ化し、指定されたコンポーネントを乗せる
        RadioButtonGroupWT<E> panel = new RadioButtonGroupWT<>(selected, finder.radios);
        panel.add((JComponent)container);

        return panel;
    }
}
```

第二引数は`Widget<T>`になっていますが、これは`VStack`や`HStack`を想定しています。まず、このコンテナ上のラジオ・ボタンを検索し、結果を`RadioButtonGroupWT`の引数に渡しています。その後、`RadioButtonGroupWT`上に指定されたコンテナを乗せています（`RadioButtonGroupWT`は`Swing`のコンテナである`JPanel`を継承したクラス）。これにより、ラジオ・ボタンをグルーピングできる上に、柔軟な配置も可能になります。

そう、そう。`RadioButtonWT`のフロントである`RadioButton`は、ただ`RadioButtonWT`を呼び出すだけです。

https://github.com/spacehijackle/SwingUI_07/blob/main/app/src/main/java/com/swingui/front/choice/RadioButton.java


## 機は熟した。後は実行あるのみ！

まずは、単純なチェック・ボックスとラジオ・ボタンのサンプル・プログラムを見ていきましょう。

![](/images/articles/df9c9b8cf8fbdb/survey01.jpg)

以下、チェック・ボックス部分を抽出してみました。

```Java:Survey.java
public class Survey
{
    //
    // 趣味の選択値（チェック・ボックス）
    //
    private final UIValue<Boolean> isPassbookGazingChecked = new UIValue<>(false);
    private final UIValue<Boolean> isNappingChecked = new UIValue<>(false);
    private final UIValue<Boolean> isZoningOutChecked = new UIValue<>(false);
    private final UIValue<Boolean> isPeopleWatchingChecked = new UIValue<>(false);
    private final UIValue<Boolean> isOtherChecked = new UIValue<>(false);

    //
    // 趣味の選択肢の活性/非活性状態（チェック・ボックス）
    //
    private final UIValue<Boolean> isPassbookGazingEnabled = new UIValue<>(true);
    private final UIValue<Boolean> isNappingEnabled = new UIValue<>(true);
    private final UIValue<Boolean> isZoningOutEnabled = new UIValue<>(true);
    private final UIValue<Boolean> isPeopleWatchingEnabled = new UIValue<>(true);

    void build()
    {
        Frame.of
        (
            "アンケート",

            VStack.of
            (
                UIAlignmentX.Leading,

                Text.of("趣味を選択してください（複数回答可）"),

                VStack.of
                (
                    UIAlignmentX.Leading,

                    CheckBox.of(isPassbookGazingChecked, "通帳を眺める")
                        .enabled(isPassbookGazingEnabled),
                    CheckBox.of(isNappingChecked, "ひたすら寝る")
                        .enabled(isNappingEnabled),
                    CheckBox.of(isZoningOutChecked, "ボーっとする")
                        .enabled(isZoningOutEnabled),
                    CheckBox.of(isPeopleWatchingChecked, "人間観察")
                        .enabled(isPeopleWatchingEnabled),
                    CheckBox.of(isOtherChecked, "その他")
                        .onCheckChanged(isChecked -> syncHobbyWhenOthersChanged(isChecked))
                )
                .padding(Left.of(8))
            )
        )
    }
}
```

チェック・ボックスである`CheckBox`を`VStack`で縦に並べています。`CheckBox.of()`の第一引数は、`UIValue<boolean>`であり、これが選択状態か否か、を表す変数です。初期状態では全てが`false`なため、全て未選択状態です。もしデフォルトで選択したい項目がある場合は、その項目に対応する変数に対し`true`を設定しておきます。

第二引数は任意型データですが、ここでは`String`型を指定しています。先ほど解説したのですが、この任意データ型の`toString()`がチェック・ボックス横の文字列になるのでしたよね。`String`型だと、指定した文字列がそのままチェック・ボックス横の文字列になっているように思えますが、実際はちょっと違う（`toString()`の返却値が設定）、ということです。結果的には同じですが（　＾ω＾）・・・

この中で“その他”について、`onCheckChanged`により、チェックのON/OFFのイベントを拾えるようにしています。これは“その他”を選択した場合、他の選択肢は非活性とするためのトリガーとなります。実際に呼ばれた場合、以下のメソッドが呼び出されます。

```Java:Survey.java
    private void syncHobbyWhenOthersChanged(boolean isChecked)
    {
        if(isChecked)
        {
            // 「その他」が選択された場合、他の選択肢を全てオフにし、選択できないようにする
            isPassbookGazingChecked.set(false);
            isNappingChecked.set(false);
            isZoningOutChecked.set(false);
            isPeopleWatchingChecked.set(false);

            isPassbookGazingEnabled.set(false);
            isNappingEnabled.set(false);
            isZoningOutEnabled.set(false);
            isPeopleWatchingEnabled.set(false);
        }
        else
        {
            // 「その他」の選択が外された場合、他の選択肢を選択できるようにする
            isPassbookGazingEnabled.set(true);
            isNappingEnabled.set(true);
            isZoningOutEnabled.set(true);
            isPeopleWatchingEnabled.set(true);
        }
    }
```

実際にいくつかの項目が選択されている状態で、“その他”を選択した時、以下のようにそれまでの選択状態が解除され、加えて“その他”以外の項目が非活性になります。

![](/images/articles/df9c9b8cf8fbdb/survey02.jpg)
![](/images/articles/df9c9b8cf8fbdb/survey03.jpg)

次にラジオ・ボタンです。今度はラジオ・ボタン周りのソースを抽出してみます。

```Java:Survey.java
public class Survey
{
    // 年齢の選択値（ラジオボタン）
    private final UIValue<String> selectedAge = UIValue.of(null);

    void build()
    {
        Frame.of
        (
            "アンケート",

            VStack.of
            (
                UIAlignmentX.Leading,

                Text.of("年齢を選択してください（単一回答）"),

                VStack.of
                (
                    UIAlignmentX.Leading,

                    RadioButtonGroup.of
                    (
                        selectedAge,
                        VStack.of
                        (
                            UIAlignmentX.Leading,

                            RadioButton.of("お子ちゃま"),
                            RadioButton.of("20代"),
                            RadioButton.of("30代"),
                            RadioButton.of("40代"),
                            RadioButton.of("50代"),
                            RadioButton.of("60代以上")
                        )
                    )
                )
                .padding(Left.of(8)),

                Spacer.of(Height.of(8)),

                Button.of("送 信")
                    .frame(Width.Infinite, Height.of(32))
                    .onClicked(self ->
                    {
                        checkBeforeSubmit();
                        JOptionPane.showMessageDialog(self.getRootPane(), "送信完了");
                    })
            )
            .padding(24)
            .frame(Width.of(300))
        );
    }
}
```

チェック・ボックスの時と違うのは、やはり単一選択のためのグルーピングである`RadioButtonGroup`があることですね。第一引数には選択データを表す変数を指定し、第二引数でラジオ・ボタンを含むコンテナ（ここでは`VStack`）を指定しています。

ちなみに第一引数である`UIValue<String>`は、初期状態で`null`を設定しているため、全ての項目が未選択状態ですね。

最後に“送信ボタン”を押すと、送信前のチェック処理が走ります。

```Java:Survey.java
    private void checkBeforeSubmit()
    {
        // 趣味の選択がない場合、適当に選択
        if(!isPassbookGazingChecked.get() && !isNappingChecked.get()
        && !isZoningOutChecked.get() && !isPeopleWatchingChecked.get() && !isOtherChecked.get())
        {
            isNappingChecked.set(true);
        }

        // 年齢の選択がない場合、適当に選択
        if(selectedAge.get() == null)
        {
            selectedAge.set("60代以上");
        }
    }
```

中身はかなり乱暴な処理ですが、ロジック側から選択値を設定できることの確認がしたかっただけ、なんです💦

![](/images/articles/df9c9b8cf8fbdb/survey04.jpg)


## さいごに

さて、今回はアンケート等で欠かせないチェック・ボックスとラジオ・ボタンについて解説しました。実はまだサンプル・プログラムがあって、このまま続けるつもりでしたが、急に書き続けるのが辛くなって次回に持ち越すことにしました🥲 生成AIもイイですが、早く若返りのクスリが実用化されて欲しいです（　＾ω＾）・・・

今回のソース一式は[こちら](https://github.com/spacehijackle/SwingUI_07/tree/main)


## 次回記事
https://zenn.dev/spacehijackle/articles/ed4353c15520ab