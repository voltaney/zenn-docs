---
title: "WinUI3のPacakged/Unpackagedで実装を分岐させる方策"
emoji: "🎉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["winui3", "csharp", "dotnet"]
published: false
---

## はじめに

WinUI3のアプリケーションには、MSIXでアプリを配布するPackaged方式と`.exe`形式で直接起動可能なUnpackaged方式の2つの方式があります。

アプリをどちらの方式にも対応させたい場合、Packaged/Unpackaged形式を適宜切り替えてデバッグ・テストしたくなります。また、それぞれの形式によって処理を分岐させたいケースもあるでしょう。
本記事ではこれらを実現する方策を、[WinUI-Gallery](https://github.com/microsoft/WinUI-Gallery/)と[TemplateStudio](https://github.com/microsoft/TemplateStudio)のソースコードを参考にしながら紹介します。

## Packaged/Unpackagedの切り替え

### Unpackaged形式にするには

前提として、WinUI3の新規プロジェクトを作成すると既定でPackaged形式になっています。これをUnpackaged形式にするためには`.csproj`に以下のプロパティ設定が最低でも必要になります。

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

[オリジナル](https://github.com/microsoft/WinUI-Gallery/blob/main/WinUIGallery/WinUIGallery.csproj)の`.csproj`では他にも色々とコンディショナルな`PropertyGroup`が定義されていますが、ここでは`WindowsPackageType`の付け替え一点にしぼり紹介します。
