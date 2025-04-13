---
title: "汎用ホストでNLog設定をappsettings.jsonで済ませる備忘録"
emoji: "📝"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [dotnet, NLog, logging, csharp]
published: true
---

## はじめに

メジャーどころのロガーは、汎用ホストからも簡単に扱えるようAPIが提供されています。NLogもその1つです。
今回はTemplateStudioのひな形（汎用ホストが実装されている）を使って、`appsettings.json`に記述した設定からNLogを構成する方法を備忘録として残しておきます。

NLog公式リポジトリにもいくつかExampleプロジェクトがあるので、参考にしてみてください。

https://github.com/NLog/NLog.Extensions.Logging/tree/v5.4.0/examples/NetCore2

## 準備

下記パッケージをNuGetからインストールします。

- [NLog](https://www.nuget.org/packages/nlog/)
- [NLog.Extensions.Logging](https://www.nuget.org/packages/NLog.Extensions.Logging/)
  - 汎用ホストからNLogを使うための拡張メソッド群

## 実装

### 汎用ホストを構成

ホスト構成を設定する箇所に下記のような追記をします。TemplateStudioで言えば`App.xaml.cs`の`App`コンストラクタに該当します。
`CreateDafaultBuilder`を呼び出している時点で`appsettings.json`は既に読み込まれているので、その設定内容を`context.Configuration`から取得できます[^1]。

[^1]: https://learn.microsoft.com/ja-jp/dotnet/api/microsoft.extensions.hosting.host.createdefaultbuilder?view=net-8.0

```diff csharp:App.xaml.cs
+ using NLog.Extensions.Logging;

public App()
{
    InitializeComponent();

    Host = Microsoft.Extensions.Hosting.Host.
    CreateDefaultBuilder().
    UseContentRoot(AppContext.BaseDirectory).
    ConfigureServices((context, services) =>
    {
        // 省略: サービスやViewModelの登録
    }).
+    // ここでNLogの登録
+    ConfigureLogging((context, builder) =>
+    {
+        // デフォルトのコンソール・イベントロガーを削除
+        builder.ClearProviders();
+        // 最低ログレベルをTraceに設定
+        builder.SetMinimumLevel(LogLevel.Trace);
+        // NLogの設定を追加
+        // 既にappsettings.jsonがCreateDefaultBuilderで読み込まている。
+        builder.AddNLog(context.Configuration);
+    }).
    Build();
}
```

### appsettings.jsonにNLogの設定を追加

`appsettings.json`にNLogの設定を追加します。設定可能なプロパティは、下記の公式Wikiに丁寧に説明されています。

https://github.com/NLog/NLog.Extensions.Logging/wiki/NLog-configuration-with-appsettings.json

例として、下記のような`jsonl`形式のログを出力する設定を`appsettings.json`に記述してみます。

```log
{ "timestamp": "2025-04-12T23:49:27.501329", "level": "Info", "logger": "MyService", "message": "MyService is initiated." }
```

`LocalSettingsOptions`はTemplateStudioの別サービスが参照する設定なので今回は気にしないでください。

```diff json:appsettings.json
{
    "LocalSettingsOptions": {
        "ApplicationDataFolder": "MyAppTablet/ApplicationData",
        "LocalSettingsFile": "LocalSettings.json"
    },
+    "Logging": {
+        "NLog": {
+            "IncludeScopes": false,
+            "ParseMessageTemplates": true,
+            "CaptureMessageProperties": true
+        }
+    },
+    "NLog": {
+        "autoreload": false,
+        "internalLogLevel": "Warn",
+        "internalLogFile": "${userLocalApplicationDataDir}/MyApp/nlog_internal.log",
+        "throwConfigExceptions": true,
+        "targets": {
+            "file": {
+                "type": "AsyncWrapper",
+                "target": {
+                    "File": {
+                        "type": "File",
+                        "fileName": "${specialfolder:folder=LocalApplicationData:cached=true}/MyApp/app.log",
+                        "archiveAboveSize": 100000,
+                        "maxArchiveFiles": 1,
+                        "layout": {
+                            "type": "JsonLayout",
+                            "Attributes": [
+                                {
+                                    "name": "timestamp",
+                                    "layout": "${date:format=yyyy-MM-ddTHH\\:mm\\:ss.ffffff}"
+                                },
+                                {
+                                    "name": "level",
+                                    "layout": "${level}"
+                                },
+                                {
+                                    "name": "logger",
+                                    "layout": "${logger}"
+                                },
+                                {
+                                    "name": "message",
+                                    "layout": "${message:raw=true}"
+                                },
+                                {
+                                    "name": "properties",
+                                    "encode": false,
+                                    "layout": {
+                                        "type": "JsonLayout",
+                                        "includeallproperties": "true"
+                                    }
+                                },
+                                {
+                                    "name": "exception",
+                                    "encode": false,
+                                    "layout": "${exception:format=@}"
+                                }
+                            ]
+                        }
+                    }
+                }
+            }
+        },
+        "rules": [
+            {
+                "logger": "*",
+                "minLevel": "Trace",
+                "writeTo": "File"
+            }
+        ]
+    }
}
```

### ログ出力

ここからは一般的なDIコンテナ同様、`ILogger<T>`をコンストラクタで受け取るようにすることで、NLogを利用できるようになります。もちろん`Host.Services.GetService`なんかでも取得可能です。

```csharp
public class MyService(ILogger<MyService> logger)
{
    public void Start(){
        logger.LogInformation("MyService is starting");
    }
}

```

## 出力先の注意点

NLog全般に関わる話なので記事の本筋からはそれてしまいますが、メモとして残しておきます。

### InternalLogとLayout Rendererで使える変数

先の`appsettings.json`の例では、InternalLogとユーザログどちらも`AppData`に出力させているのですが、フォルダ指定に使える変数名が下記のように異なります。

```json
"internalLogFile": "${userLocalApplicationDataDir}/MyApp/nlog_internal.log",
"fileName": "${specialfolder:folder=LocalApplicationData:cached=true}/MyApp/app.log",
```

それぞれ下記Wikiに詳述されています。

https://github.com/nlog/NLog/wiki/Special-Folder-Layout-Renderer

https://github.com/NLog/NLog/wiki/Internal-Logging

### パッケージ化されたアプリにおけるAppDataの仮想化

パッケージ化されたアプリ(MSIX)の場合、`%AppData%`が仮想化されるので先の`appsettings.json`の設定におけるログ出力先も仮想化されることに注意です。
仮想化については下記記事でまとめています。

https://zenn.dev/voltaney/articles/msix-appdata-virtualization
