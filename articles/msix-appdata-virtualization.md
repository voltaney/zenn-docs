---
title: "MSIXパッケージのAppData仮想化にまつわるリダイレクトの挙動"
emoji: "📁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["winui3", "csharp", "dotnet", "msix"]
published: true
---

## はじめに

パッケージ化されたアプリにおいて、**Windows 10 version 1903**以降、AppData（`%APPDATA%`）フォルダは仮想化されるようになりました[^1]。（下記は`AppData\Local`の例）

[^1]: https://learn.microsoft.com/ja-jp/windows/msix/desktop/desktop-to-uwp-behind-the-scenes#common-file-system-operations

- 従来： `C:\Users\<username>\AppData\Local`
- 仮想化： `C:\Users\<username>\AppData\Local\Packages\<ハッシュ値>\LocalCache\Local`

従来のようなパスを指定して読み書き操作を行った場合、**仮想化されたパスへ操作がリダイレクトされます**。ただし、**従来のAppDataフォルダにのみ既に実体が存在している場合、操作はリダイレクトされません**。

:::details リダイレクトに関するMSドキュメントの引用
[ドキュメント](https://learn.microsoft.com/ja-jp/windows/msix/desktop/desktop-to-uwp-behind-the-scenes#common-file-system-operations)

> 次のディレクトリの下に作成された新しいファイルとフォルダーは、ユーザーごと、パッケージごとのプライベートな場所にリダイレクトされます。
>
> - ローカル
> - Local\Microsoft
> - ローミング
> - Roaming\Microsoft
> - Roaming\Microsoft\Windows\Start Menu\Programs

:::

<!-- textlint-disable -->

本記事では実際にこの当たりの挙動を確認します。なお、パッケージ化された環境においては、AppData操作用に[`Windows.Storage.ApplicationData`API](https://learn.microsoft.com/ja-jp/uwp/api/windows.storage.applicationdata?view=winrt-26100)が用意されていますが、今回はあくまでもリダイレクトの挙動を確認することが目的なので多用しません。

<!-- textlint-enable -->

## 検証方法

1. 従来のAppDataパスを使って`MSIX_TEST\sample.txt`を新規作成・既に存在するなら追記。
   - `StreamWriter`で`append: true`を指定。
   - "テスト書き込み"という内容を追記。
1. `dir`コマンドを別プロセスで実行し、ファイルの実体があるかを確認。
1. 従来のAppDataパスと仮想化されたAppDataパスの両ファイルをAPIで確認。
   - `File.Exists`、`File.ReadAllText`を使って確認。

:::details 実行するコード(WinUI3)

```cs:MainWindow.xaml.cs
using System;
using System.Diagnostics;
using System.IO;

using Microsoft.UI.Xaml;

using Windows.Storage;

namespace PackageTypeCheck;

public sealed partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();

        // パスの設定
        var myAppName = "MSIX_TEST";
        var fileName = "sample.txt";
        var localAppDataRootPath = Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData);
        var localAppDataPath = Path.Combine(localAppDataRootPath, myAppName);
        var virtualizedLocalAppDataPath = Path.Combine(ApplicationData.Current.LocalCacheFolder.Path, "Local", myAppName);

        // 従来のAppDataパスを使ってファイルを作成・書き込み
        var sampleFilePath = Path.Combine(localAppDataPath, fileName);
        Console.WriteLine($"★ [{sampleFilePath}]に追記（無ければ作成）\n");
        Directory.CreateDirectory(localAppDataPath);
        using (var sw = new StreamWriter(sampleFilePath, append: true))
        {
            sw.WriteLine("テスト書き込み");
        }

        // dirコマンドでファイル一覧を表示
        Console.WriteLine("★ dirコマンドでファイル確認");
        // C:\Users\biz\AppData\Local\MSIX_TEST
        ListFilesByDirCommand(localAppDataPath);
        // 仮想化されたLocalAppData
        ListFilesByDirCommand(virtualizedLocalAppDataPath);

        // ファイルを確認
        Console.WriteLine($"★ ファイルを読み込み");
        // C:\Users\biz\AppData\Local\MSIX_TEST
        Console.WriteLine($"[{sampleFilePath}]");
        Console.WriteLine($"File.Exists：{File.Exists(sampleFilePath)}");
        Console.WriteLine("File.ReadAllText：");
        if (File.Exists(sampleFilePath))
        {
            Console.WriteLine($"{File.ReadAllText(sampleFilePath)}");
        }
        // 仮想化されたLocalAppData
        var virtualizedSampleFilePath = Path.Combine(virtualizedLocalAppDataPath, fileName);
        Console.WriteLine($"[{virtualizedSampleFilePath}]");
        Console.WriteLine($"File.Exists：{File.Exists(virtualizedSampleFilePath)}");
        Console.WriteLine("File.ReadAllText：");
        if (File.Exists(virtualizedSampleFilePath))
        {
            Console.WriteLine($"{File.ReadAllText(virtualizedSampleFilePath)}");
        }
    }

    // コマンドプロンプトでdirコマンドを実行し、ファイル一覧を標準出力
    void ListFilesByDirCommand(string path)
    {
        Console.WriteLine($"[{path}]のファイル一覧");
        var psi = new ProcessStartInfo("cmd.exe")
        {
            Arguments = $"/c dir /b \"{path}",
            RedirectStandardOutput = true,
            UseShellExecute = false
        };
        using var process = Process.Start(psi);
        string output = process!.StandardOutput.ReadToEnd();
        Console.WriteLine(output);
    }
}


```

:::

https://github.com/voltaney/Sample-MSIX-AppData-Virtualization

## 従来・仮想化どちらにもパスが存在しない場合

作成されたファイルは、従来パスには実体がない（dirコマンドで確認できない）が、リダイレクトが働いて読み書きができていることが分かります。

```log:標準出力
★ [C:\Users\biz\AppData\Local\MSIX_TEST\sample.txt]に追記（無ければ作成）

★ dirコマンドでファイル確認
[C:\Users\biz\AppData\Local\MSIX_TEST]のファイル一覧
ファイルが見つかりません

[C:\Users\biz\AppData\Local\Packages\b8710d43-ca04-4c6d-ad91-0c86f3ba55d3_tbq0xhjce8r70\LocalCache\Local\MSIX_TEST]のファイル一覧
sample.txt

★ ファイルを読み込み
[C:\Users\biz\AppData\Local\MSIX_TEST\sample.txt]
File.Exists：True
File.ReadAllText：
テスト書き込み

[C:\Users\biz\AppData\Local\Packages\b8710d43-ca04-4c6d-ad91-0c86f3ba55d3_tbq0xhjce8r70\LocalCache\Local\MSIX_TEST\sample.txt]
File.Exists：True
File.ReadAllText：
テスト書き込み
```

## 従来・仮想化どちらのパスも存在する場合

先のコードを一度実行した後に、従来パスに`MSIX_TEST\sample.txt`を下記内容で作成し、再度実行してみます。なお、先ほどの`StreamWriter`コンストラクタ内では`append: true`を指定しているので、ファイルが存在する場合は追記されます。

```txt:従来パスのsample.txt
手動で作成しました。
```

### 実行結果

```log:標準出力
★ [C:\Users\biz\AppData\Local\MSIX_TEST\sample.txt]に追記（無ければ作成）

★ dirコマンドでファイル確認
[C:\Users\biz\AppData\Local\MSIX_TEST]のファイル一覧
sample.txt

[C:\Users\biz\AppData\Local\Packages\b8710d43-ca04-4c6d-ad91-0c86f3ba55d3_tbq0xhjce8r70\LocalCache\Local\MSIX_TEST]のファイル一覧
sample.txt

★ ファイルを読み込み
[C:\Users\biz\AppData\Local\MSIX_TEST\sample.txt]
File.Exists：True
File.ReadAllText：
テスト書き込み
テスト書き込み

[C:\Users\biz\AppData\Local\Packages\b8710d43-ca04-4c6d-ad91-0c86f3ba55d3_tbq0xhjce8r70\LocalCache\Local\MSIX_TEST\sample.txt]
File.Exists：True
File.ReadAllText：
テスト書き込み
テスト書き込み
```

どちらにもファイルが存在していますが、仮想化された方のパスが優先して読み書きされていることが分かります。実際、従来パスの方の`sample.txt`の内容は変更されませんでした。

## 従来パスのみ存在する場合

仮想化されたフォルダのみ`sample.txt`を削除してから、再度実行してみます。

```log:標準出力
★ [C:\Users\biz\AppData\Local\MSIX_TEST\sample.txt]に追記（無ければ作成）

★ dirコマンドでファイル確認
[C:\Users\biz\AppData\Local\MSIX_TEST]のファイル一覧
sample.txt

[C:\Users\biz\AppData\Local\Packages\b8710d43-ca04-4c6d-ad91-0c86f3ba55d3_tbq0xhjce8r70\LocalCache\Local\MSIX_TEST]のファイル一覧
★ ファイルを読み込み
[C:\Users\biz\AppData\Local\MSIX_TEST\sample.txt]
File.Exists：True
File.ReadAllText：
手動で作成しました。テスト書き込み

[C:\Users\biz\AppData\Local\Packages\b8710d43-ca04-4c6d-ad91-0c86f3ba55d3_tbq0xhjce8r70\LocalCache\Local\MSIX_TEST\sample.txt]
File.Exists：False
File.ReadAllText：

```

従来パスが存在する場合はリダイレクトが行われずに、直接処理されていることが分かります。

## まとめ

以上から、パッケージ環境下におけるリダイレクトは執筆時点で下記のような挙動をすることが分かりました。

| 従来パス   | 仮想化パス | リダイレクト           |
| ---------- | ---------- | ---------------------- |
| 存在しない | 存在しない | 仮想化パスの操作になる |
| 存在する   | 存在する   | 仮想化パスの操作になる |
| 存在する   | 存在しない | 従来パスの操作になる   |

## 最後に

NLogでAppDataにログファイルを出力していたのですが、このような仕様があることを知らず、ログファイルを見つけるのに手間取りました。
個々のアプリに関連付いたファイルがないまぜになることを避ける（MS風に言えま **rot（ja-JSドキュメントでは劣化）** を避ける）ために、仮想化は導入されたようです。パッケージ化した場合には気を付けていきたいですね。
