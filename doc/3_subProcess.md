## subProcessの実行

機能を拡張する際にsubProcessを立ち上げBackgroundWorkerとして働いてもらうとできることが増えます。

- Script間の変数の共有 : ```System.IO.MemoryMappedFiles```
- 通信の監視 : ```System.IO.Ports```
- 並列でのTask処理

## subProcessの呼び方

戦略としては二つの

- CMD Session (```Cmd.Write```)で実行 
- powershell scriptから```Start-Process```で実行

CMD Sessionは標準入出力を使用しアクションから入力・結果取得が可能ですがConsoleを出す手段がないのでデバッグに苦労します。一方、Start-Processは結果の取得にひと工夫必要ですが、Consoleの出力・非出力をコントロールすることができます

共に、別ファイルからの呼び出しが必要となります。

### CMD Session

- 標準入出力を使用しアクションから入力・結果取得が可能 : ```Cmd.ReadFromCmdSession```, ```Cmd.WaitForText```
- Consoleが出ない
- ```WaitForText```での正規表現がまぁまぁうざったい

Robin

```
Cmd.Open.Open Session=> CmdSession
Cmd.Write Session: CmdSession Command: $'''dotnet script \"C:\\test.csx\"''' SendEnter: True
Cmd.WaitForText Session: CmdSession Text: $'''Press any key to continue''' IsRegEx: False IgnoreCase: True Timeout: 0
Cmd.ReadFromCmdSession.Read Session: CmdSession CmdOutput=> CmdOutput1
Cmd.Write Session: CmdSession Command: $'''hoge''' SendEnter: True
Cmd.ReadFromCmdSession.Read Session: CmdSession CmdOutput=> CmdOutput2
Cmd.Close Session: CmdSession

# [ControlRepository][PowerAutomateDesktop]

{
  "ControlRepositorySymbols": [],
  "ImageRepositorySymbol": {
    "Name": "imgrepo",
    "ImportMetadata": {},
    "Repository": "{\r\n  \"Folders\": [],\r\n  \"Images\": [],\r\n  \"Version\": 1\r\n}"
  }
}

```

C:\test.csx
```csharp
using System;

Console.WriteLine("Press any key to continue");
var val = Console.ReadLine();
Console.WriteLine($"input : {val}");
```

このような形で、```WaitForText``` ```ReadFromCmdSession``` ```Write```の繰り返しでフローを構成します。ここでは```dotnet script```を呼び出していますがConsoleで動くものであれば、呼び出しされるコードの形式は問いません。

標準入出力にアタッチしてくれるので手軽さはありますが、アクションはたくさん消費するのでうまくサブフローにまとめないとすぐにスパゲッティー化しそうです。

### Start-Process

- Consoleの出力・非出力を選択的に行える



 
