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
- 標準入出力からの入力出力は困難
  - ipcやmemorymappingなどを使用したプロセス間通信を使用する
  - printfデバッグと切り分けしやすいと考えるとスマート

Robin
```powershell
Scripting.RunPowershellScript Script: $'''$id = Start-Process dotnet.exe -WindowStyle Normal -ArgumentList \"script c:\\Temp\\test.csx\" -PassThru
ConvertTo-Json $id''' ScriptOutput=> JsonOutput ScriptError=> ScriptError
Variables.ConvertJsonToCustomObject Json: JsonOutput CustomObject=> JsonAsCustomObject
WAIT 1
LOOP LoopIndex FROM 0 TO 5 STEP 1
    Scripting.RunPowershellScript Script: $'''Add-Type -TypeDefinition @\'
using System;
using System.Threading;
using System.IO.MemoryMappedFiles;

public class Class1{
  static string mapName = \"SharedMemoryTest\";

  public static void Set(int i){
    using(var mmf = MemoryMappedFile.OpenExisting(mapName))
    using(var accessor = mmf.CreateViewAccessor()){
      accessor.Write(0, i);
    }
  }
}
\'@ -Language CSharp

[Class1]::Set(%LoopIndex%)''' ScriptOutput=> JsonOutput ScriptError=> ScriptError
    WAIT 1
END

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

PowerShell部抜粋 (subprocess実行)
```powershell
$id = Start-Process dotnet.exe -WindowStyle Normal -ArgumentList "script c:\Temp\test.csx" -PassThru
ConvertTo-Json $id
```

```Start-Process``` のオプションの内訳はこんな感じ。

- ```-WindowStyle``` : Consoleを出力するかどうか。```Hidden```で隠せる
- ```-PassThru``` : 異常終了などでプロセスが残ってしまった場合、```Stop-Process```で終了させるためidは取っておきましょう

ここでも```dotnet script```を呼んでいますが、別のものでも構いません。

PowerShell部抜粋 (入出力部)
```powershell
Add-Type -TypeDefinition @'
using System;
using System.Threading;
using System.IO.MemoryMappedFiles;

public class Class1{
  static string mapName = "SharedMemoryTest";

  public static void Set(int i){
    using(var mmf = MemoryMappedFile.OpenExisting(mapName))
    using(var accessor = mmf.CreateViewAccessor()){
      accessor.Write(0, i);
    }
  }
}
'@ -Language CSharp

[Class1]::Set(%LoopIndex%)
```

ここでは値の変更のみを行っています。直接、csxのコードを記述する形としています。

subprocessで外部のcsxを呼んでいるので、同様に

- csxファイルをコマンドとして呼び出し
- csxファイルからcodeを読み込んで```Add-Type```
- ドット演算子でのps1ファイル読み込み
- ```Import-Module```でpsm1ファイルの読み込み

といった手段で外部化することができます。1ファイル構成にこだわるならsubprocessで使用したcsxファイルに対し

- コマンドラインオプションの実装
- ```Get-Content -Raw```と正規表現で特定ブロックのみ切り出し

という手も使用できます。特に後者は若干アバンギャルドですが、名前付きメモリマップドファイルや名前付きパイプなど共通のロジックや定数を利用する際にメリットがあります。

C:\Temp\test.csx
```csharp
using System;
using System.Threading;
using System.IO.MemoryMappedFiles;

static string mapName = "SharedMemoryTest";

using(var mmf = MemoryMappedFile.CreateNew(mapName, sizeof(int)))
using(var accessor = mmf.CreateViewAccessor()){
  accessor.Write(0, (int)1);
  while(true){
    int dst;
    accessor.Read(0,out dst);
    Console.WriteLine($"Value = {dst}");
    if(dst > 3) return;
    Thread.Sleep(1000);
  }
}
```

複雑な変更をするならちゃんとmutexで排他制御しましょう。
