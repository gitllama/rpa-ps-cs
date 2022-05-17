# Webからインクルード

PADはどうせネットにつながないといけないのだから、いっそのことJavascriptをCDNから読むように、ネットからコードをインクルードできるようにしてみます

CDNに登録なんて当然していないので、今回はgithubから直にコードを読み込むこととします

## PAD + PS1

```Web.InvokeWebService.InvokeWebService```アクションを使用する場合、以下のようになります。ps1ではなく、csxコードを呼ぶ場合は```Invoke-Expression```の代わりに```Add-Type -TypeDefinition```を使用するとよいでしょう。

また、```%hoge%```は文字列の置換が行われているだけなので、実際は```$code= "%hoge%"```としなくても```%hoge%```だけでも実行することは可能です。

```powershell
Web.InvokeWebService.InvokeWebService Url: $'''https://raw.githubusercontent.com/gitllama/rpa-ps1-cs/main/lib/test.ps1''' Method: Web.Method.Get Accept: $'''application/xml''' ContentType: $'''application/xml''' ConnectionTimeout: 30 FollowRedirection: True ClearCookies: False FailOnErrorStatus: False EncodeRequestBody: True UserAgent: $'''Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.8.1.21) Gecko/20100312 Firefox/3.6''' Encoding: Web.Encoding.AutoDetect AcceptUntrustedCertificates: False ResponseHeaders=> WebServiceResponseHeaders Response=> WebServiceResponse StatusCode=> StatusCode
Scripting.RunPowershellScript Script: $'''$code = @\'
%WebServiceResponse%
\'@
Invoke-Expression $code''' ScriptOutput=> JsonOutput ScriptError=> ScriptError

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

powershell scriptの1ステップで記述する場合は```Invoke-webrequest```を利用し以下のように記述できます

```powershell
Scripting.RunPowershellScript Script: $'''(Invoke-webrequest -URI \"https://raw.githubusercontent.com/gitllama/rpa-ps1-cs/main/lib/test.ps1\").Content | Invoke-Expression
''' ScriptOutput=> JsonOutput ScriptError=> ScriptError

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

キャッシュが効いてるわけではなさそうなので、一度読みこんだコードは変数に保存しておく・一度コンパイルかけてテンポラリーとしてローカル保存するなど、高速化には一工夫が必要です。

## PAD + javascript

PADを使用する場合、javascriptはそれほど使い勝手がいいわけではありませんが、ブラウザ操作（Elementの差し替え）の際は使用することとなります。

javascript上の外部コードを実行するには以下のような方法があります。

```javascript
var code = 'function Test(v, b) { return v + b }';
eval(code);
WScript.StdOut.Write(Test(1, 2));
```

```javascript
var code = ";(function (globalObject){ \
'use strict'; \
function Test(v, b) { return v - b } \
globalObject.Test = Test; \
})(this);";
var b = new Function(code);
b();
WScript.StdOut.Write(Test(3,2));
```

```powershell
Web.InvokeWebService.InvokeWebService Url: $'''https://raw.githubusercontent.com/gitllama/rpa-ps1-cs/main/lib/test.js''' Method: Web.Method.Get Accept: $'''application/xml''' ContentType: $'''application/xml''' ConnectionTimeout: 30 FollowRedirection: True ClearCookies: False FailOnErrorStatus: False EncodeRequestBody: True UserAgent: $'''Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.8.1.21) Gecko/20100312 Firefox/3.6''' Encoding: Web.Encoding.AutoDetect AcceptUntrustedCertificates: False ResponseHeaders=> WebServiceResponseHeaders Response=> WebServiceResponse StatusCode=> StatusCode
Scripting.RunJavascript JavascriptCode: $'''%WebServiceResponse%
WScript.StdOut.Write(Test(3,2));''' ScriptOutput=> JavascriptOutput ScriptError=> ScriptError2

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

```Scripting.RunJavascript```では```XMLHttpRequest()```が使用できません。また、```ActiveXObject(\"Msxml2.XMLHTTP\")```, ```ActiveXObject(\"Microsoft.XMLHTTP\")```もどうも正常に動かなさそうだったので、```Web.InvokeWebService.InvokeWebService```を使用する必要があります。

また、CDNからコードを読み込んでそもまま使う場合も互換を取りにくくて、あまり使い勝手がよいとは言えません。
