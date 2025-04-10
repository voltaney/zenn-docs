---
title: "WinUI3のFlyoutのスタイルを変更する"
emoji: "🐦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["winui3", "csharp", "dotnet", "xaml"]
published: false
---

## はじめに

WinUI3の`Flyout`のスタイル指定方法が他のコンポーネントに比べて少し特殊だったので備忘録です。結論をお急ぎの方は[Flyout自体のスタイルの変更方法](#Flyout自体のスタイルの変更方法)セクションへ。

## Flyoutとは

WinUI3では`Button`に`Flyout`属性を付けることができ、クリック時でポップアップ（Flyout）が表示されるようにできます。しかもXAMLだけの実装でOK。

![flyout-example2](/images/winui3-flyout-style/flyout-example2.png)

```xml
<Button>
    Additional Info1
    <Button.Flyout>
        <Flyout Placement="Bottom">
            <TextBlock TextWrapping="WrapWholeWords">
                こんにちは 🎉
            </TextBlock>
        </Flyout>
    </Button.Flyout>
</Button>
```

## デフォルトスタイルの問題点

大変便利なのですが、下記のように長い文章を表示させようとすると、`TextWrapping`を有効にしていても改行されずに横スクロースされてしまいます。

![flyout-example1](/images/winui3-flyout-style/flyout-example1.png)

また、`Flyout`の`Content`でレイアウト系のコンポーネントを入れて幅を固定させようとしても、`Flyout`自体の幅は特定の幅までしか広がらず、スクロールバーに変わってしまいます。

![flyout-example3](/images/winui3-flyout-style/flyout-example3.png)

## Flyout自体のスタイルの変更方法

実は`Flyout`自体はコントロールやUIElementでは無いのでStyle適用のためのプロパティが存在しません[^1]。代わりに`Flyout`のコンテンツ表示を担当する`FlyoutPresenter`のスタイルを変更する必要があります。

例えば、下記例では水平スクロールを無効化してテキストを`MaxWidth`で折り返すように実装しています。ついでに`Padding`も指定しています。

```diff xml
 <Button>
    Additional Info4
    <Button.Flyout>
        <Flyout Placement="Bottom">
+            <Flyout.FlyoutPresenterStyle>
+                <Style TargetType="FlyoutPresenter">
+                    <Setter Property="ScrollViewer.HorizontalScrollMode" Value="Disabled" />
+                    <Setter Property="ScrollViewer.HorizontalScrollBarVisibility" Value="Disabled" />
+                    <Setter Property="MaxWidth" Value="400" />
+                    <Setter Property="Padding" Value="10" />
+                </Style>
+            </Flyout.FlyoutPresenterStyle>
            <TextBlock TextWrapping="WrapWholeWords">
                Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Aenean commodo ligula eget dolor. Aenean massa.
                Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Donec quam felis, ultricies nec, pellentesque eu, pretium quis, sem.
            </TextBlock>
        </Flyout>
    </Button.Flyout>
 </Button>
```

![flyout-example4](/images/winui3-flyout-style/flyout-example4.png)

変更可能なプロパティ一覧は下記を参考に。

https://learn.microsoft.com/ja-jp/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.controls.flyoutpresenter?view=windows-app-sdk-1.6#properties

[^1]: https://learn.microsoft.com/ja-jp/windows/windows-app-sdk/api/winrt/microsoft.ui.xaml.controls.flyout.flyoutpresenterstyle?view=windows-app-sdk-1.6#--

## 余談（Backdropの指定）

`Flyout`には`SystemBackdrop`プロパティも存在します。下記のように設定することでポップアップの背景をMicaやAcrylicにできるはずです。以前試した際はうまく反映されなかった気がするのですが、一応共有です。

```diff xml
 <Button>
    Additional Info4
    <Button.Flyout>
        <Flyout Placement="Bottom">
            <Flyout.FlyoutPresenterStyle>
                <Style TargetType="FlyoutPresenter">
                    <Setter Property="ScrollViewer.HorizontalScrollMode" Value="Disabled" />
                    <Setter Property="ScrollViewer.HorizontalScrollBarVisibility" Value="Disabled" />
                    <Setter Property="TabNavigation" Value="Cycle" />
                    <Setter Property="MaxWidth" Value="600" />
                    <Setter Property="Padding" Value="10" />
                </Style>
            </Flyout.FlyoutPresenterStyle>
+            <Flyout.SystemBackdrop>
+                <MicaBackdrop />
+            </Flyout.SystemBackdrop>
            <TextBlock TextWrapping="WrapWholeWords">
                Lorem ipsum dolor sit amet, consectetuer adipiscing elit. Aenean commodo ligula eget dolor. Aenean massa.
                Cum sociis natoque penatibus et magnis dis parturient montes, nascetur ridiculus mus. Donec quam felis, ultricies nec, pellentesque eu, pretium quis, sem.
            </TextBlock>
        </Flyout>
    </Button.Flyout>
 </Button>
```
