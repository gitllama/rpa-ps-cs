# rpa-ps1-cs

UWSCの作者がお亡くなりになりupdateされなくなって久しいです。世間でRPAが一般化してきたこともあり、乗り換えることとするわけですが痒いところに手が届かないのでHackする方法を考えましょう。

## RPA (PowerAutomate Desktop)

Windowsの標準となったので、PowerAutomate Desktop (PAD)を使用します。

PowerAutomateではありませんPowerAutomate Desktopです。ややこしいですね。

個人的には重いしネットに接続されてないと動かないしアクションの作りもあまり好きになれません。しかし、弊社社員に使わせるには```GUI```/```環境構築が必要ない```というのはそれにあまりあるメリットです。標準機能は正義。

## Powershell

PowerAutomate DesktopではscriptとしてPowershell、Javascript、Python、VBScriptが使用できます（ユーザーが独自にアクションを作成・追加できるカスタムコネクタ機能はPowerAutomateの方だけの機能です）。

そこで、Powershellを核とします。私のコスモはJavascriptが将来切り捨てられるとささやいています。

## C# Script

Powershellから、C#って簡単に呼べるの知ってました？便利ですよね。

Powershellの構文はなかなか癖がありますが、これでストレスフリーにコードが書けますし、.Netのライブラリも呼び出しできます。Nugetからもライブラリ呼べます。ちなみに今回は環境構築などをなるべく必要としない、ポータブルな環境を目指してますのでDLLを作ってから呼び出しやファイル構成を増やすことはなるべく避けます。

### ポータビリティーに関しての補足

Javascript、Pythonの場合、CDNやnpm/PyPi/Anacondaでライブラリを追加できます。しかし正面突破ではCDNは使えませんし、ScriptのパッケージマネージャーのPathってどういう扱いになるんですかね（確認してないからよくわかりません）？

その点、C# Scriptは```#r```ディレクティブでcode内から直接nugetからアセンブリを読み込むことができます。
