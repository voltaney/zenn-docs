---
title: "WinUI3でPackaged/Unpackagedの両方に対応するための方策"
emoji: "📦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["winui3", "csharp", "dotnet"]
published: true
---

## はじめに

WinUI3のアプリケーションには、MSIXでアプリを配布するPackaged方式と、直接起動できる`.exe`形式で配布可能なUnpackaged方式の2つの方式があります。

アプリをどちらの方式にも対応させる場合、Packaged/Unpackaged形式を切り替えてビルドやテストを行う必要が出てきます。また、それぞれの形式によって処理を分岐させたいケースもあるでしょう。
本記事ではこれらを実現する方策を、[WinUI-Gallery](https://github.com/microsoft/WinUI-Gallery/)や[TemplateStudio](https://github.com/microsoft/TemplateStudio)のソースコードを参考にしながら紹介します。

なお、説明で使用しているプロジェクトは、Visual Studioが用意している下記のテンプレートプロジェクトから作成した直後のものです。
![winui3-template-project](/images/winui3-switch-package-type/winui3-template-project.png)
本記事で作成したプロジェクトを確認したい場合は、GitHubリポジトリを参照してください。

https://github.com/voltaney/Sample-WinUI3-PackageTypeCheck

## （前知識） Unpackaged形式にするには

前提として、WinUI3の新規プロジェクトを作成すると既定でPackaged形式になっています。これをUnpackaged形式にするためには`.csproj`に以下のプロパティ設定が最低限必要になります。

```xml:.csproj
<WindowsPackageType>None</WindowsPackageType>
```

これを設定すると、`launchSettings.json`にUnpackaged形式向けの起動プロファイルも自動的に追記されます。

```json:launchSettings.json
{
  "profiles": {
    "PackageTypeCheck (Package)": {
      "commandName": "MsixPackage"
    },
    "PackageTypeCheck (Unpackaged)": {
      "commandName": "Project"
    }
  }
}
```

このプロファイルを切り替えるだけで両形式とも起動できると楽なのですが、実際には先ほどの`WindowsPackageType`が効いてくるので、Packaged形式で実行しようとしてもエラーが出ます。

## パッケージ形式の切り替え方法

実際に両形式でパブリッシュされている良例が、MSがリリースしている[WinUI-Gallery](https://github.com/microsoft/WinUI-Gallery/)（WinUI3コンポーネントのギャラリー）です。
`.csproj`を見てみると、構成（DebugやRelease等）に`Unpackaged`版も追加することで`PropertyGroup`を切り替えていることが分かります。

[オリジナル](https://github.com/microsoft/WinUI-Gallery/blob/v2.5.1/WinUIGallery/WinUIGallery.csproj)の`.csproj`では他にも色々とコンディショナルな`PropertyGroup`が定義されていますが、ここでは`WindowsPackageType`の付け替え一点に絞って紹介します。

### ソリューション構成の追加

Visual Studioのメニューバーで**構成マネージャー**を選択します。

![構成マネージャー](/images/winui3-switch-package-type/config-manager.png)

**アクティブソリューション構成**のドロップダウンから**新規作成**を選択し、設定のコピー元を`Debug`とした状態で`Debug-Unpackaged`という構成を作成します。
`Release`についても同様に`Release-Unpackaged`を作成します。

![add-configration](/images/winui3-switch-package-type/add-configration.png)

### プロジェクトファイルにPropertyGroupを追加

作成した構成に基づいて、`.csproj`で下記ロジックを実現できるようにします。

1. Packaged形式かどうかを定義する`Packaged`というプロパティ（任意名で可）を`true`で追加する。
1. `Configuration`が`Debug-Unpackaged`または`Release-Unpackaged`の場合、`Packaged`を`false`にする。
1. `Packaged`が`true`の場合
   - `EnableMsixTooling`を`true`にする。
1. `Packaged`が`false`の場合
   - `EnableMsixTooling`を`false`にする。
   - `WindowsPackageType`を`None`にする。

実際の`.csproj`は下記のようになります。
なお、`Configurations`や`Publish`関連のプロパティは構成追加の時点で自動追記されているはずです。

```diff xml:.csproj
 <Project Sdk="Microsoft.NET.Sdk">
   <PropertyGroup>
     <OutputType>WinExe</OutputType>
     <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
     <RuntimeIdentifiers>win-x86;win-x64;win-arm64</RuntimeIdentifiers>
     <PublishProfile>win-$(Platform).pubxml</PublishProfile>
     <UseWinUI>true</UseWinUI>
-    <EnableMsixTooling>true</EnableMsixTooling>
     <Nullable>enable</Nullable>
+    <Configurations>Debug;Release;Debug-Unpackaged;Release-Unpackaged</Configurations>
+    <Packaged>true</Packaged>
+    <Packaged Condition="'$(Configuration)' == 'Debug-Unpackaged' Or '$(Configuration)' == 'Release-Unpackaged'">false</Packaged>
+  </PropertyGroup>
+
+  <PropertyGroup Condition="'$(Packaged)' != 'true'">
+    <EnableMsixTooling>false</EnableMsixTooling>
+    <WindowsPackageType>None</WindowsPackageType>
+  </PropertyGroup>
+
+  <PropertyGroup Condition="'$(Packaged)' == 'true'">
+    <EnableMsixTooling>true</EnableMsixTooling>
   </PropertyGroup>

   <ItemGroup>
   <!-- Publish Properties -->
   <PropertyGroup>
     <PublishReadyToRun Condition="'$(Configuration)' == 'Debug'">False</PublishReadyToRun>
+    <PublishReadyToRun Condition="'$(Configuration)'=='Debug-Unpackaged'">False</PublishReadyToRun>
     <PublishReadyToRun Condition="'$(Configuration)' != 'Debug'">True</PublishReadyToRun>
     <PublishTrimmed Condition="'$(Configuration)' == 'Debug'">False</PublishTrimmed>
+    <PublishTrimmed Condition="'$(Configuration)'=='Debug-Unpackaged'">False</PublishTrimmed>
     <PublishTrimmed Condition="'$(Configuration)' != 'Debug'">True</PublishTrimmed>
   </PropertyGroup>
 </Project>

```

### 構成を切り替えて実行

以上の設定後、起動プロファイルに合わせて構成マネージャーを選択することで、Packaged/Unpackagedを切り替えて実行できるようになります。

![unpackaged-example](/images/winui3-switch-package-type/unpackaged-example.png)
_Unpackaged形式_

![packaged-example](/images/winui3-switch-package-type/packaged-example.png)
_Packaged形式_

## パッケージ形式による実装分岐（プリプロセッサ）

[WinUIEx](https://github.com/dotMorten/WinUIEx)（WinUI3のWindowを拡張したライブラリ）の[サンプルコード](https://github.com/dotMorten/WinUIEx/blob/v2.5.1/src/WinUIExSample/App.xaml.cs)なんかではこの方法が取られています。
パッケージ形式に基づいて`.csproj`で特定の定数を定義し、`#if`ディレクティブで分岐させる方法です。

### `.csproj`で定数を定義

ここでは例としてPackaged形式の時に`PACKAGED`という定数を定義します。

```diff xml:.csproj
 <PropertyGroup Condition="'$(Packaged)' == 'true'">
  <EnableMsixTooling>true</EnableMsixTooling>
+  <DefineConstants>PACKAGED</DefineConstants>
 </PropertyGroup>
```

:::details WinUIExのExampleコードの場合、どんな定義か
`WindowsPackageType`が`None`の時に`UNPACKAGED`定数を定義しています。

```xml:.csproj
<DefineConstants Condition="'$(WindowsPackageType)'=='None'">UNPACKAGED;$(DefineConstants)</DefineConstants>
```

https://github.com/dotMorten/WinUIEx/blob/v2.5.1/src/WinUIExSample/WinUIExSample.csproj

:::

### #ifで分岐

下記の具合で分岐させることができます。Visual Studioでは実行されない`#if`ブロックはグレイの文字列で表示されますが、構成を切り替えると自動的にエディタの表示も切り替わります。

```csharp:MainWindow.xaml.cs
using Microsoft.UI.Xaml;
namespace PackageTypeCheck;

public sealed partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
#if PACKAGED
        // Packaged app
        MyText.Text = "Packaged App";
#else
        // Unpackaged app
        MyText.Text = "Unpackaged App";
#endif
    }
}
```

## パッケージ形式による実装分岐（ランタイム）

**WinUI-Gallery**と**TemplateStudio**どちらもこの方式を採用しており、ヘルパクラスとして提供されています。本項ではこれらの実装は行わず、それぞれのプロジェクトの実装を簡単に紹介するにとどめます。

### 動作原理

Windows APIには呼び出し元のパッケージ情報を取得する関数（後述）が定義されています。この関数は、非パッケージのプロセスから呼び出された場合、`APPMODEL_ERROR_NO_PACKAGE`（**15700**）というリターンコードを返す仕様になっています[^1]。
このリターンコードを確認することで、ランタイムでもパッケージ形式を判断できるようになります。

[^1]: https://learn.microsoft.com/ja-jp/windows/win32/api/appmodel/nf-appmodel-getcurrentpackagefullname#return-value

### WinUI-Gallery

`api-ms-win-appmodel-runtime-l1-1-1.dll`を介して[GetCurrentPackageId](https://learn.microsoft.com/ja-jp/windows/win32/api/appmodel/nf-appmodel-getcurrentpackageid)関数を呼び出し、返り値が15700か判断しています。

https://github.com/microsoft/WinUI-Gallery/blob/v2.5.1/WinUIGallery/Helper/NativeHelper.cs#L20-L46

### TemplateStudio

`kernel32.dll`を介して[GetCurrentPackageFullName](https://learn.microsoft.com/ja-jp/windows/win32/api/appmodel/nf-appmodel-getcurrentpackagefullname#return-value)関数を呼び出し、返り値が15700か判断しています。

https://github.com/microsoft/TemplateStudio/blob/v5.5/code/TemplateStudioForWinUICs/Templates/Proj/Default/Param_ProjectName/Helpers/RuntimeHelper.cs#L6-L20
