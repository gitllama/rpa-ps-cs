## TCPIP Sample

ということで、

- TCPIPの通信を受けた際にMemoryMappedへの書き込み
- MemoryMappedを観察してるPADへの反映

までのサンプルコード

MemoryMappedの共通コードはregionで挟まれた範囲を抜き出してPADで再利用してます

```powershell
Scripting.RunPowershellScript Script: $'''$hash = @{}
$hash[\"path\"] = \"c:\\Temp\\test.csx\"
$hash[\"ps_var\"] = $PSVersionTable.PSVersion
$hash[\"cs_var\"] = [System.Runtime.InteropServices.RuntimeInformation]::FrameworkDescription
$hash[\"process\"] = Start-Process dotnet.exe -WindowStyle Normal -ArgumentList \"script c:\\Temp\\test.csx SERVER\" -PassThru

ConvertTo-Json $hash''' ScriptOutput=> JsonOutput ScriptError=> ScriptError
Variables.ConvertJsonToCustomObject Json: JsonOutput CustomObject=> JsonAsCustomObject
WAIT 1
LOOP LoopIndex FROM 0 TO 1 STEP 0
    Scripting.RunPowershellScript Script: $'''$raw = Get-Content \'%JsonAsCustomObject.path%\' -Raw
$raw = ($raw -split \"#endregion MemoryMapping\")[0]
$raw = ($raw -split \"#region MemoryMapping\")[1]

Add-Type -TypeDefinition $raw -Language CSharp

$dst = [MemoryMap]::Value
$dst''' ScriptOutput=> JsonOutput ScriptError=> ScriptError
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

c:\Temp\test.csx
```csharp
#region MemoryMapping

using System;
using System.Threading;
using System.IO.MemoryMappedFiles;
using System.Net;
using System.Net.Sockets;

public class MemoryMap {

  public static string mapName = "SharedMemoryTest";
  public static string mutexName = "MutexTest";

  public static int Value{
    get {
      ind dst = 0;
      callmem((accessor)=>{ accessor.Read(0, out dst);
      return dst; 
    }
    set { callmem((accessor)=>{ accessor.Write(0, value); }); }
  }

  static void callmem(Action<MemoryMappedViewAccessor> act){   
    using(var mmf = MemoryMappedFile.OpenExisting(mapName))
    using(var mutex = new Mutex(false, mutexName))
    using(var accessor = mmf.CreateViewAccessor()){
      mutex.WaitOne();
      try{
        act(accessor);
      }finally{
        mutex.ReleaseMutex();  
      }
    }
  }

}

#endregion MemoryMapping

static (string ip, int port) addr = ("127.0.0.1", 9999);

switch(Args[0].ToUpper()){
  case "SERVER":
  {
    var task = Task.Run(()=>{ Server(addr); }).ContinueWith(t => { });
    task.Wait();
    break;
  }
  case "CLIENT":
  {
    Client(addr, Args[1]);
    break;
  }
  default:
    throw new Exception();
}

public static void Server((string ip, int port) addr){
  TcpListener server = null;
  try {
    Byte[] buffer = new Byte[1024];
    using(var mmf = MemoryMappedFile.CreateNew(MemoryMap.mapName, sizeof(int))){
      server = new TcpListener(IPAddress.Parse(addr.ip), addr.port);
      server.Start();
      Console.WriteLine("wait");
      while(true){
        using(var client = server.AcceptTcpClient())
        using(var stream = client.GetStream()){
          Console.WriteLine("connected");
          int i;
          string dst = "";
          while((i = stream.Read(buffer, 0, buffer.Length))!=0)  
          {
            var data = System.Text.Encoding.UTF8.GetString(buffer, 0, i); 
            Console.WriteLine($"received : {data}");
            dst += data;
          }
          MemoryMap.Value = Int32.Parse(dst);
          Console.WriteLine(MemoryMap.Value);
          client.Close();
        }
      }
    }
  }
  catch(SocketException e){ Console.WriteLine(e.Message); }
  finally{ server.Stop(); }
}

public static void Client((string ip, int port) addr, string src){
  try {
    using(var client = new TcpClient(addr.ip, addr.port))
    using(var stream = client.GetStream()){
      var data = System.Text.Encoding.UTF8.GetBytes(src); 
      stream.Write(data, 0, data.Length); 
      Console.WriteLine($"send: {src}"); 
      client.Close();
    }
  } catch(Exception e) { Console.WriteLine($"Exception: {e}"); }
}
```
