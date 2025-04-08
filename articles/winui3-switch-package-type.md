---
title: "WinUI3のPackaged/Unpackagedで実装分岐"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["winui3", "csharp", "dotnet"]
published: false
---

## はじめに

WinUI3のアプリケーションには、MSIXでアプリを配布するPackaged方式と、直接起動できる`.exe`形式で配布可能なUnpackaged方式の2つの方式があります。

アプリをどちらの方式にも対応させたい場合、Packaged/Unpackaged形式を適宜切り替えてデバッグ・テストしたくなります。また、それぞれの形式によって処理を分岐させたいケースもあるでしょう。
本記事ではこれらを実現する方策を、[WinUI-Gallery](https://github.com/microsoft/WinUI-Gallery/)と[TemplateStudio](https://github.com/microsoft/TemplateStudio)のソースコードを参考にしながら紹介します。

なお、説明で使用しているプロジェクトは、Visual Studioが用意している下記のテンプレートプロジェクトから作成した直後のものです。
![winui3-template-project](/images/winui3-switch-package-type/winui3-template-project.png)
実際のコードを確認したい場合は、GitHubリポジトリを参照してください。

https://github.com/voltaney/Sample-WinUI3-PackageTypeCheck

## Packaged/Unpackagedの切り替え

### Unpackaged形式にするには

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

### WinUI-Galleryが採用している切り替え方式

実際に両形式でパブリッシュされている良例が、MSがリリースしている[WinUI-Gallery](https://github.com/microsoft/WinUI-Gallery/)（WinUI3コンポーネントの紹介ソフト）です。
`.csproj`を見てみると、構成（DebugやRelease等）に`Unpackaged`版も追加することで`PropertyGroup`を切り替えていることが分かります。

[オリジナル](https://github.com/microsoft/WinUI-Gallery/blob/main/WinUIGallery/WinUIGallery.csproj)の`.csproj`では他にも色々とコンディショナルな`PropertyGroup`が定義されていますが、ここでは`WindowsPackageType`の付け替え一点に絞って紹介します。

#### 1. 構成の追加

Visual Studioのメニューバーで**構成マネージャー**を選択します。

![構成マネージャー](/images/winui3-switch-package-type/config-manager.png)

**アクティブソリューション構成**のドロップダウンから**新規作成**を選択し、設定のコピー元を`Debug`とした状態で`Debug-Unpackaged`という構成を作成します。
`Release`についても同様に`Release-Unpackaged`を作成します。

![add-configration](/images/winui3-switch-package-type/add-configration.png)

#### 2. PropertyGroupの追加

次に、先ほど作成した構成に合わせて`PropertyGroup`を追記します。
`Configurations`に関しては構成追加の時点で自動追記されているはずです。

https://github.com/voltaney/Sample-WinUI3-PackageTypeCheck/blob/4bb0a34ceef71f70dea1d443a4cdd178c5631911/PackageTypeCheck/PackageTypeCheck.csproj#L1-L25

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

```xml:.csproj
<PropertyGroup>
  <OutputType>WinExe</OutputType>
  <TargetFramework>net8.0-windows10.0.19041.0</TargetFramework>
  <TargetPlatformMinVersion>10.0.17763.0</TargetPlatformMinVersion>
  <RootNamespace>PackageTypeCheck</RootNamespace>
  <ApplicationManifest>app.manifest</ApplicationManifest>
  <Platforms>x86;x64;ARM64</Platforms>
  <RuntimeIdentifiers>win-x86;win-x64;win-arm64</RuntimeIdentifiers>
  <PublishProfile>win-$(Platform).pubxml</PublishProfile>
  <UseWinUI>true</UseWinUI>
  <Nullable>enable</Nullable>
  <Configurations>Debug;Release;Debug-Unpackaged;Release-Unpackaged</Configurations>
  <Packaged>true</Packaged>
  <Packaged Condition="'$(Configuration)' == 'Debug-Unpackaged' Or '$(Configuration)' == 'Release-Unpackaged'">false</Packaged>
</PropertyGroup>

<PropertyGroup Condition="'$(Packaged)' != 'true'">
  <EnableMsixTooling>false</EnableMsixTooling>
  <WindowsPackageType>None</WindowsPackageType>
</PropertyGroup>

<PropertyGroup Condition="'$(Packaged)' == 'true'">
  <EnableMsixTooling>true</EnableMsixTooling>
  <DefineConstants>PACKAGED</DefineConstants>
</PropertyGroup>
```
