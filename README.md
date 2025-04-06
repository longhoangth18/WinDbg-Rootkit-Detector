# 🛡️ Rootkit Detection & DKOM Analysis via WinDbg

## 🧠 What is **DKOM**?

**DKOM (Direct Kernel Object Manipulation)** is a highly stealthy and dangerous technique used to **hide processes, drivers, files, or kernel modules**. It is commonly employed by **advanced rootkits and malware**.

---

## ⚙️ How DKOM Works (with Processes)

Windows manages active processes using a **doubly linked list** (`LIST_ENTRY`) in the kernel:

```c
PsActiveProcessHead
```

Each process is represented by an EPROCESS structure, which includes a pointer called ActiveProcessLinks. This points to the previous and next processes in the global process list.

👉 A Rootkit Using DKOM Will:
Unlink the process from PsActiveProcessHead

→ Tools like Task Manager, tasklist, Process Explorer rely on this list → the process becomes invisible.

However, the kernel still maintains a pointer to the EPROCESS object → it can still be discovered using WinDbg or forensic memory analysis tools like Volatility.




## 🎯 Objective

Develop a rootkit model that:

- Hides the **process `longth-tool.exe`** from process managers like Task Manager, Process Explorer, etc.
- Hides the **executable file `longth-tool.exe`** from the file system
- Uses **DKOM (Direct Kernel Object Manipulation)** to directly modify kernel structures
## In this post I temporarily demo the hunting tool I developed.
![image](https://github.com/user-attachments/assets/a3fc254b-ab40-426d-bbd7-5c5d71118739)
## This is the PID of the current process and directory.
![image](https://github.com/user-attachments/assets/701264dc-c8b1-4d1c-b444-8f12f358c7d7)
![image](https://github.com/user-attachments/assets/c8fec807-696e-4403-a9b4-6a91f85147ac)
## 🔍 After Performing Rootkit Process and File Hiding

Once the rootkit has been deployed and DKOM manipulation is complete, perform the following steps to verify and analyze the stealth effect:

### ✅ 1. Validate Process Hiding

#### 🔹 Task Manager / Process Explorer / process hacker

- Open Task Manager (`Ctrl + Shift + Esc`) or Process Explorer.
- Confirm that `longth-tool.exe` is **not visible**, despite being actively running in the background.
![image](https://github.com/user-attachments/assets/5df28ef6-5147-4f06-9fb5-7fcdd32c0aa0)
---

## 🐞 Enable Kernel Debugging with WinDbg

To analyze DKOM-based rootkit behavior using **WinDbg**, you must enable **kernel debugging** on the target machine.

### 🛠️ Step 1: Enable Kernel Debugging Mode

Run the following command **as Administrator** on the target Windows system:

```powershell
bcdedit /debug on
```
Then reboot the system:
```powershell
shutdown /r /t 0
```
💡 Tip: This enables kernel-mode debugging using default settings (usually over COM port or USB for VM debugging).
🧩 Step 2: Connect WinDbg to the Target System
## 🧪 Analyzing Hidden Processes with WinDbg

After enabling kernel debugging (`bcdedit /debug on`) and rebooting, follow these steps to start WinDbg and inspect hidden processes.

---

### 🐞 Step-by-Step: Start WinDbg

1. **Open WinDbg as Administrator**
   - On your host/debugger machine.
   - Use **WinDbg (x64)** if you're debugging a 64-bit OS.

2. **Start Kernel Debugging**
   - Go to:  
     `File → Kernel Debug...`
   - Choose the appropriate transport method (COM / USB / Network).
   - Match settings with the target machine (`bcdedit /dbgsettings`).

3. **Wait for Connection**
   - Once the target system boots, WinDbg will show:
     ```
     Microsoft (R) Windows Debugger
     Copyright ...
     ```

---

### 🔍 Run the Process Listing Command

- After listing all loaded processes using `!process 0 0`, pay attention to processes with suspicious or duplicated names and compare them with the **current system's known PIDs**.

```assembly
PROCESS ffffe3849009c300  
    SessionId: 1  Cid: 2104    Peb: 975585000  ParentCid: 17e8  
    DirBase: 08b4f000  ObjectTable: ffffaa84136416c0  HandleCount: 103.  
    Image: longth-tool.exe  
```

# Process Analysis: EPROCESS Structure and Key Details

## `PROCESS ffffd78da57e9080`
- This is the **address of the EPROCESS structure** in the kernel.
- Every process on Windows has an EPROCESS structure located in kernel memory.

## `SessionId: none`

| `SessionId` | Common Meaning |
| --- | --- |
| `0`, `1`, `2` | Normal GUI user processes (e.g., explorer.exe, chrome.exe) |
| `none` | System processes, service processes, hidden processes, early boot processes |
- `none` means the process **does not belong to any user session** (e.g., not part of session 1, 2... like you would see in GUI login).
- This typically applies to:
    - **System services**
    - **Injected processes**
    - **Rootkit-hidden or stealthy processes**

## `Cid: 2bb4` (Client ID)
- Represents the Process ID (PID).
- In decimal: `0x2bb4` = **11188**
- This can be used to trace the process, you could try checking with `tasklist /fi "PID eq 11188"` to see if it's visible.

## `Peb: 7dfe290000`
- PEB = **Process Environment Block**, is a user-space memory region containing process information (such as image name, heap, loaded modules...).
- The presence of a PEB indicates the process has been **fully created** — meaning it's not a "ghost" or garbage entry.

## `DirBase: 1afb59000`
- This is the page directory base address, which helps the kernel access the process's memory.
- If `DirBase` is valid, it means the process is currently running/has actual memory.
- It’s important when you want to **switch contexts to analyze the process's memory**.

## PEB Check
```assembly
lkd> .process /p ffffe3849009c300; !peb 975585000
Implicit process is now ffffe384`9009c300
PEB at 0000000975585000
    InheritedAddressSpace:    No
    ReadImageFileExecOptions: No
    BeingDebugged:            No
    ImageBaseAddress:         00007ff74a6a0000
    NtGlobalFlag:             0
    NtGlobalFlag2:            0
    Ldr                       00007ff99499a4c0
    Ldr.Initialized:          Yes
    Ldr.InInitializationOrderModuleList: 0000013fde0e2520 . 0000013fde148960
    Ldr.InLoadOrderModuleList:           0000013fde0e2690 . 0000013fde148810
    Ldr.InMemoryOrderModuleList:         0000013fde0e26a0 . 0000013fde148820
                    Base TimeStamp                     Module
            7ff74a6a0000 67d2b49a Mar 13 17:34:02 2025 C:\Users\Atomic_Redteam\Pictures\longth-tool.exe
            7ff994830000 bd2c3c23 Jul 29 00:06:43 2070 C:\Windows\SYSTEM32\ntdll.dll
            7ff992de0000 61e69688 Jan 18 17:29:28 2022 C:\Windows\System32\KERNEL32.DLL
            7ff9922a0000 812662a7 Aug 30 17:21:27 2038 C:\Windows\System32\KERNELBASE.dll
            7ff994540000 efa6b327 May 29 22:24:23 2097 C:\Windows\System32\USER32.dll
            7ff992860000 0dcd0213 May 04 03:26:59 1977 C:\Windows\System32\win32u.dll
            7ff993270000 a19db164 Dec 04 00:05:40 2055 C:\Windows\System32\GDI32.dll
            7ff991f70000 f981a3e7 Aug 26 15:08:07 2102 C:\Windows\System32\gdi32full.dll
            7ff992200000 39255ccf May 19 22:25:03 2000 C:\Windows\System32\msvcp_win.dll
            7ff992080000 2bd748bf Apr 23 08:39:11 1993 C:\Windows\System32\ucrtbase.dll
            7ff9929b0000 15fd8d3b Sep 10 09:51:39 1981 C:\Windows\System32\ADVAPI32.dll
            7ff992ae0000 564f9f39 Nov 21 05:31:21 2015 C:\Windows\System32\msvcrt.dll
            7ff9946e0000 4782ccda Jan 08 08:07:38 2008 C:\Windows\System32\sechost.dll
            7ff992c50000 4c237b59 Jun 24 22:35:53 2010 C:\Windows\System32\RPCRT4.dll
            7ff9932b0000 3a0e9944 Nov 12 20:21:08 2000 C:\Windows\System32\IMM32.DLL
            7ff994430000 897a522a Feb 02 20:03:38 2043 C:\Windows\System32\SHLWAPI.dll
            7ff98f9a0000 e799fe58 Feb 16 20:23:36 2093 C:\Windows\system32\uxtheme.dll
            7ff9940d0000 c94441ae Jan 01 09:27:58 2077 C:\Windows\System32\combase.dll
            7ff992890000 53a97e2b Jun 24 20:33:31 2014 C:\Windows\System32\MSCTF.dll
            7ff992b80000 61567b6b Oct 01 10:07:23 2021 C:\Windows\System32\OLEAUT32.dll
    SubSystemData:     0000000000000000
    ProcessHeap:       0000013fde0e0000
    ProcessParameters: 0000013fde0e1c60
    CurrentDirectory:  'C:\Users\Atomic_Redteam\Pictures\'
    WindowTitle:  'C:\Users\Atomic_Redteam\Pictures\longth-tool.exe'
    ImageFile:    'C:\Users\Atomic_Redteam\Pictures\longth-tool.exe'
    CommandLine:  '"C:\Users\Atomic_Redteam\Pictures\longth-tool.exe" '
    DllPath:      '< Name not readable >'
    Environment:  0000013fde147070
        =::=::\
        ALLUSERSPROFILE=C:\ProgramData
        APPDATA=C:\Users\Atomic_Redteam\AppData\Roaming
        CommonProgramFiles=C:\Program Files\Common Files
        CommonProgramFiles(x86)=C:\Program Files (x86)\Common Files
        CommonProgramW6432=C:\Program Files\Common Files
        COMPUTERNAME=ATOMIC-REDTEAM
        ComSpec=C:\Windows\system32\cmd.exe
        DriverData=C:\Windows\System32\Drivers\DriverData
        FPS_BROWSER_APP_PROFILE_STRING=Internet Explorer
        FPS_BROWSER_USER_PROFILE_STRING=Default
        HOMEDRIVE=C:
        HOMEPATH=\Users\Atomic_Redteam
        LOCALAPPDATA=C:\Users\Atomic_Redteam\AppData\Local
        LOGONSERVER=\\ATOMIC-REDTEAM
        NUMBER_OF_PROCESSORS=4
        OneDrive=C:\Users\Atomic_Redteam\OneDrive
        OS=Windows_NT
        Path=C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Users\Atomic_Redteam\AppData\Local\Microsoft\WindowsApps;
        PATHEXT=.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC
        PROCESSOR_ARCHITECTURE=AMD64
        PROCESSOR_IDENTIFIER=AMD64 Family 25 Model 80 Stepping 0, AuthenticAMD
        PROCESSOR_LEVEL=25
        PROCESSOR_REVISION=5000
        ProgramData=C:\ProgramData
        ProgramFiles=C:\Program Files
        ProgramFiles(x86)=C:\Program Files (x86)
        ProgramW6432=C:\Program Files
        PSModulePath=C:\Program Files\WindowsPowerShell\Modules;C:\Windows\system32\WindowsPowerShell\v1.0\Modules
        PUBLIC=C:\Users\Public
        SESSIONNAME=Console
        SystemDrive=C:
        SystemRoot=C:\Windows
        TEMP=C:\Users\ATOMIC~1\AppData\Local\Temp
        TMP=C:\Users\ATOMIC~1\AppData\Local\Temp
        USERDOMAIN=ATOMIC-REDTEAM
        USERDOMAIN_ROAMINGPROFILE=ATOMIC-REDTEAM
        USERNAME=Atomic_Redteam
        USERPROFILE=C:\Users\Atomic_Redteam
        windir=C:\Windows
        _PYI_APPLICATION_HOME_DIR=C:\Users\ATOMIC~1\AppData\Local\Temp\_MEI84522
        _PYI_ARCHIVE_FILE=C:\Users\Atomic_Redteam\Pictures\longth-tool.exe
        _PYI_PARENT_PROCESS_LEVEL=0
```
## !peb Command Output Analysis

### Process Details
- **PEB at**: `0000000975585000`
- **Process Information**:
    - **ImageBaseAddress**: `00007ff74a6a0000`
    - **ImageFile**: `C:\Users\Atomic_Redteam\Pictures\longth-tool.exe`
    - **Current Directory**: `C:\Users\Atomic_Redteam\Pictures\`
    - **Window Title**: `C:\Users\Atomic_Redteam\Pictures\longth-tool.exe`
    - **Command Line**: `"C:\Users\Atomic_Redteam\Pictures\longth-tool.exe"`

### Loaded Modules:
The PEB provides information about loaded modules, which can indicate whether the process is using known libraries and if any unusual or potentially malicious libraries are loaded.

| **Base Address** | **Timestamp**        | **Module**                      |
| ---------------- | -------------------- | -------------------------------- |
| `7ff74a6a0000`    | Mar 13 17:34:02 2025 | `C:\Users\Atomic_Redteam\Pictures\longth-tool.exe` |
| `7ff994830000`    | Jul 29 00:06:43 2070 | `C:\Windows\SYSTEM32\ntdll.dll` |
| `7ff992de0000`    | Jan 18 17:29:28 2022 | `C:\Windows\System32\KERNEL32.DLL` |
| `7ff9922a0000`    | Aug 30 17:21:27 2038 | `C:\Windows\System32\KERNELBASE.dll` |
| `7ff994540000`    | May 29 22:24:23 2097 | `C:\Windows\System32\USER32.dll` |
| `7ff992860000`    | May 04 03:26:59 1977 | `C:\Windows\System32\win32u.dll` |
| `7ff993270000`    | Dec 04 00:05:40 2055 | `C:\Windows\System32\GDI32.dll` |
| `7ff991f70000`    | Aug 26 15:08:07 2102 | `C:\Windows\System32\gdi32full.dll` |

### Process Environment Variables:
The process has several environment variables that provide insight into the system configuration and the environment in which the process is running.

| **Environment Variable** | **Value** |
| ------------------------ | --------- |
| `ALLUSERSPROFILE`        | `C:\ProgramData` |
| `APPDATA`                | `C:\Users\Atomic_Redteam\AppData\Roaming` |
| `ComSpec`                | `C:\Windows\system32\cmd.exe` |
| `TEMP`                   | `C:\Users\ATOMIC~1\AppData\Local\Temp` |
| `USERNAME`               | `Atomic_Redteam` |

### Critical Flags:
- **BeingDebugged**: `No` – The process is not being debugged.
- **InheritedAddressSpace**: `No` – The process does not inherit the address space from another process.

### Key Observations:
- **Image Base Address**: `00007ff74a6a0000` – This is the address where the image of the process is loaded in memory.
- **Loaded Modules**: The process has several system and application modules loaded, such as `ntdll.dll`, `KERNEL32.DLL`, `USER32.dll`, and others. The presence of `longth-tool.exe` as the base address suggests it is a custom or potentially suspicious application.
- **Environment Variables**: The environment variables suggest a typical Windows environment with standard directories and user paths.
  
### Suspicious Indicators:
- **Command Line**: The command line contains the execution path of `longth-tool.exe`, which might indicate a custom or third-party tool.
- **Unusual Timestamps**: The loaded modules like `ntdll.dll`, `KERNEL32.DLL`, and `USER32.dll` seem normal, but their timestamps (e.g., 2070 and 2097) could be unusual for a typical system and may indicate tampering or modifications, depending on the context.


# Process Memory Dump Analysis using WinDbg

## Objective:
To perform a memory dump of a specific process using WinDbg for forensic analysis. The memory is being dumped from the process with the base address `0x7ff74a6a0000` to analyze its contents.

### Steps to Dump Process Memory:

1. **Identify the Process**:
   The process is identified using the `.process /p ffffe3849009c300` command in WinDbg. This sets the target process for subsequent commands.

   - **Implicit Process**: `ffffe3849009c300`
```assembly
     lkd> .process /p ffffe3849009c300
Implicit process is now ffffe384`9009c300
lkd> !address 0x7ff74a6a0000
Mapping user range ...
Mapping system range ...
Mapping non addressable range ...
Mapping page tables...
Mapping hyperspace...
Mapping HAL reserved range...
Mapping User Probe Area...
Mapping system shared page...
Mapping system cache working set...
Mapping loader mappings...
Mapping system PTEs...
Mapping system paged pool...
Mapping session space...
Mapping dynamic system space...
Mapping PFN database...
Mapping non paged pool...
Mapping VAD regions...
Mapping module regions...
Mapping process, thread, and stack regions...
Mapping system cache regions...


Usage:                  
Base Address:           00000009`75593000
End Address:            00007fff`ffff0000
Region Size:            00007ff6`8aa5d000
VA Type:                UserRange
```
   
2. **Check Memory Mapping**:
   Using the `!address` command with the base address `0x7ff74a6a0000`, WinDbg maps out various system and user ranges of memory, such as:
   - **User Range**: The user-mode memory region.
   - **System Range**: The kernel-mode memory region.
   - **Non-Addressable Range**: Address space regions that cannot be accessed.
   - **Loader Mappings**: Memory mappings for loaded modules.
   - **Process/Thread/Stack Regions**: Areas specific to the running process.
   - **Module Regions**: Memory sections used by loaded modules.
   - **Paged and Non-Paged Pools**: Memory regions used for system data structures and buffers.

3. **Attempt to Dump Memory**:
   The `.writemem` command is used to write the process memory to a binary file (`longth_tool_memdump.bin`). The memory is being dumped from the base address `0x7ff74a6a0000` to an offset of `0x5A000`.

   **Command**:
   ```
   .writemem C:\Users\Atomic_Redteam\Desktop\longth_tool_memdump.bin 0x00007ff74a6a0000 0x00007ff74a6a0000+0x5A000
   ```

```
lkd> .writemem C:\Users\Atomic_Redteam\Desktop\longth_tool_memdump.bin 0x00007ff74a6a0000 0x00007ff74a6a0000+0x5A000
Writing 5a001 bytes........
Unable to read memory at 00007ff7`4a6a4000, file is incomplete
lkd> .writemem C:\Users\Atomic_Redteam\Desktop\longth_tool_memdump.bin 0x00007ff74a6a0000 0x00007ff74a6a0000+0x5A000
Writing 5a001 bytes........
Unable to read memory at 00007ff7`4a6a4000, file is incomplete
```
![image](https://github.com/user-attachments/assets/f4a5d985-ee83-492a-a9ab-9c9bf70edf48)

# Process Memory Dump Analysis

## Issue Overview:
The memory dump process for the target application `longth-tool.exe` was not fully successful. The generated dump file is incomplete due to an error when trying to read specific memory addresses. This issue is likely caused by the presence of a rootkit utilizing kernel-level mechanisms, which prevent full access to the process's memory space.

### Error Description:
- The memory dump command was executed with the intention of capturing the contents of the process memory, but the resulting file is incomplete.
- **Error Encountered**: `Unable to read memory at 00007ff7`4a6a4000`, resulting in an incomplete memory dump.
  
### Possible Cause:
The failure to read the entire memory region suggests the presence of a rootkit or malicious kernel-level mechanism. Rootkits often employ techniques to hide specific parts of memory or intercept memory access requests, making it difficult to fully capture the contents of a process’s memory.

## It took me a while to research the rootkit mechanism and found out
# Rootkit API Hooking for Hiding Files and Folders

## Overview:
Rootkits use API hooking techniques to hide files and folders from the operating system and applications. By intercepting calls to certain Windows API functions, the rootkit can modify the results returned by these functions, effectively making hidden files and folders invisible to the system and user.

### **1. Hooking Windows API to Hide Files and Folders**

Rootkits hook into key Windows APIs that the operating system and applications use to query the file and folder listings. These APIs include:

- **NtQueryDirectoryFile**: Used to retrieve the list of files and directories within a specified directory.
- **FindFirstFile/FindNextFile**: Used to enumerate files in a directory.
- **ZwQueryDirectoryFile**: A lower-level API used by Windows to query file directories.

📌 **How It Works:**

- When a process (e.g., Windows Explorer or cmd.exe) calls these APIs to list files or directories, the rootkit intercepts and filters out any files that match a predefined list of hidden items.
- If a file or folder name matches one of the hidden entries, the rootkit will remove it from the results before they are returned to the caller, making it invisible to the system and user.

---

### **2. How the API Hooking Works**

Rootkits hook into **ntdll.dll** and **kernel32.dll**, as these libraries provide the APIs that Windows uses to access files:

1. **Hooking NtQueryDirectoryFile:**
    - When an application calls this API to retrieve a file listing, the rootkit checks the name of each file.
    - If the file or folder name matches an entry in the hidden list, it is removed from the results before being returned.

2. **Hooking FindFirstFile/FindNextFile:**
    - When an application calls FindFirstFile to search for files, the rootkit inspects the results and removes any unwanted files from the list before it is returned.

3. **Hooking ZwQueryDirectoryFile:**
    - This lower-level API for querying file directories is also hooked by the rootkit to ensure all file queries are intercepted and filtered as needed.

### Displays a list of all DLLs (Dynamic Link Libraries) that have been loaded into a process's memory space.

```assembly
lkd> !dlls

0x13fde0e2690: C:\Users\Atomic_Redteam\Pictures\longth-tool.exe
      Base   0x7ff74a6a0000  EntryPoint  0x7ff74a6ac380  Size        0x0005a000    DdagNode     0x13fde0e27c0
      Flags  0x0000a2cc  TlsIndex    0x00000000  LoadCount   0xffffffff    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL

0x13fde0e2500: C:\Windows\SYSTEM32\ntdll.dll
      Base   0x7ff994830000  EntryPoint  0x00000000  Size        0x001f5000    DdagNode     0x13fde0e2630
      Flags  0x0000a2c4  TlsIndex    0x00000000  LoadCount   0xffffffff    NodeRefCount 0x00000000
             <unknown>
             LDRP_IMAGE_DLL

0x13fde0e2b50: C:\Windows\System32\KERNEL32.DLL
      Base   0x7ff992de0000  EntryPoint  0x7ff992df70d0  Size        0x000bd000    DdagNode     0x13fde0e2c80
      Flags  0x000ca2cc  TlsIndex    0x00000000  LoadCount   0xffffffff    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e3160: C:\Windows\System32\KERNELBASE.dll
      Base   0x7ff9922a0000  EntryPoint  0x7ff9922b0650  Size        0x002c8000    DdagNode     0x13fde0e3290
      Flags  0x0008a2cc  TlsIndex    0x00000000  LoadCount   0xffffffff    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e4190: C:\Windows\System32\USER32.dll
      Base   0x7ff994540000  EntryPoint  0x7ff994557f30  Size        0x001a0000    DdagNode     0x13fde0e42c0
      Flags  0x000ca2ec  TlsIndex    0x00000000  LoadCount   0x00000002    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e4590: C:\Windows\System32\win32u.dll
      Base   0x7ff992860000  EntryPoint  0x00000000  Size        0x00022000    DdagNode     0x13fde0e46c0
      Flags  0x0008a2ec  TlsIndex    0x00000000  LoadCount   0x00000001    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e4880: C:\Windows\System32\GDI32.dll
      Base   0x7ff993270000  EntryPoint  0x7ff9932748d0  Size        0x0002a000    DdagNode     0x13fde0e42c0
      Flags  0x000ca2ec  TlsIndex    0x00000000  LoadCount   0x00000002    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e4c60: C:\Windows\System32\gdi32full.dll
      Base   0x7ff991f70000  EntryPoint  0x7ff991f9fe60  Size        0x0010b000    DdagNode     0x13fde0e42c0
      Flags  0x000ca2ec  TlsIndex    0x00000000  LoadCount   0x00000002    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e50b0: C:\Windows\System32\msvcp_win.dll
      Base   0x7ff992200000  EntryPoint  0x7ff992215390  Size        0x0009d000    DdagNode     0x13fde0e51e0
      Flags  0x000ca2ec  TlsIndex    0x00000000  LoadCount   0x00000001    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e54b0: C:\Windows\System32\ucrtbase.dll
      Base   0x7ff992080000  EntryPoint  0x7ff992096110  Size        0x00100000    DdagNode     0x13fde0e55e0
      Flags  0x0008a2ec  TlsIndex    0x00000000  LoadCount   0x00000002    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e5940: C:\Windows\System32\ADVAPI32.dll
      Base   0x7ff9929b0000  EntryPoint  0x7ff9929c5600  Size        0x000ac000    DdagNode     0x13fde0e5a70
      Flags  0x0008a2ec  TlsIndex    0x00000000  LoadCount   0x00000003    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e69e0: C:\Windows\System32\msvcrt.dll
      Base   0x7ff992ae0000  EntryPoint  0x7ff992ae7850  Size        0x0009e000    DdagNode     0x13fde0e6b10
      Flags  0x0008a2ec  TlsIndex    0x00000000  LoadCount   0x00000001    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e6dc0: C:\Windows\System32\sechost.dll
      Base   0x7ff9946e0000  EntryPoint  0x7ff9946fcd40  Size        0x0009b000    DdagNode     0x13fde0e6ef0
      Flags  0x000ca2ec  TlsIndex    0x00000000  LoadCount   0x00000001    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0e7280: C:\Windows\System32\RPCRT4.dll
      Base   0x7ff992c50000  EntryPoint  0x7ff992caedb0  Size        0x0012b000    DdagNode     0x13fde0e73b0
      Flags  0x0008a2ec  TlsIndex    0x00000000  LoadCount   0x00000003    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED
    Unable to read Module Name

0x13fde0ef290: Unknown Module
      Base   0x7ff9932b0000  EntryPoint  0x7ff9932b14d0  Size        0x00030000    DdagNode     0x13fde0ef3c0
      Flags  0x0008a2cc  TlsIndex    0x00000000  LoadCount   0x00000002    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED
    Unable to read Module Name

0x13fde0e4e40: Unknown Module
      Base   0x7ff994430000  EntryPoint  0x7ff99443a7a0  Size        0x00055000    DdagNode     0x13fde0f01d0
      Flags  0x000ca2cc  TlsIndex    0x00000000  LoadCount   0x00000000    NodeRefCount 0x00000000
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde0f7990: C:\Windows\system32\uxtheme.dll
      Base   0x7ff98f9a0000  EntryPoint  0x7ff98f9c8c40  Size        0x0009e000    DdagNode     0x13fde141a50
      Flags  0x000ca2cc  TlsIndex    0x00000000  LoadCount   0x00000000    NodeRefCount 0x00000000
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_DONT_CALL_FOR_THREADS
             LDRP_PROCESS_ATTACH_CALLED

0x13fde148220: C:\Windows\System32\combase.dll
      Base   0x7ff9940d0000  EntryPoint  0x7ff9941c2d50  Size        0x00355000    DdagNode     0x13fde141810
      Flags  0x0008a2cc  TlsIndex    0x00000000  LoadCount   0x00000000    NodeRefCount 0x00000000
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde148940: C:\Windows\System32\MSCTF.dll
      Base   0x7ff992890000  EntryPoint  0x7ff9928d1740  Size        0x00115000    DdagNode     0x13fde141930
      Flags  0x0008a2cc  TlsIndex    0x00000000  LoadCount   0x00000000    NodeRefCount 0x00000000
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED

0x13fde148810: C:\Windows\System32\OLEAUT32.dll
      Base   0x7ff992b80000  EntryPoint  0x7ff992b9de00  Size        0x000cd000    DdagNode     0x13fde141e70
      Flags  0x0008a2cc  TlsIndex    0x00000000  LoadCount   0x00000000    NodeRefCount 0x00000000
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
             LDRP_PROCESS_ATTACH_CALLED
```

## !dlls command in WinDbg and get the following information:
```assembly
0x13fde0e2690: C:\Users\Atomic_Redteam\Pictures\longth-tool.exe
      Base   0x7ff74a6a0000  EntryPoint  0x7ff74a6ac380  Size        0x0005a000    DdagNode     0x13fde0e27c0
      Flags  0x0000a2cc  TlsIndex    0x00000000  LoadCount   0xffffffff    NodeRefCount 0x00000000
             <unknown>
             LDRP_LOAD_NOTIFICATIONS_SENT
             LDRP_IMAGE_DLL
```
## Key Concepts

### 1. Memory Attributes
- **Base**: `0x7ff74a6a0000` — This is the memory address where the executable file is loaded in memory.
  
- **EntryPoint**: `0x7ff74a6ac380` — The entry point of the application, which is the starting point of execution within the file.
  
- **Size**: `0x0005a000` — The size of the memory block allocated to the executable file (approximately 368 KB).

### 2. Loading Information
- **Flags**: `0x0000a2cc` — These are the load flags that give additional details about the loading state of the file. This flag can indicate that the file has been loaded under special conditions.
  
- **TlsIndex**: `0x00000000` — The Thread Local Storage (TLS) index. A value of `0x00000000` suggests that this file is not associated with TLS.
  
- **LoadCount**: `0xffffffff` — This is an abnormal value. Typically, this value would be a positive integer representing the number of times the file has been loaded. A value of `0xffffffff` may indicate a problem with the loading process or that the file has not been loaded properly.
  
- **NodeRefCount**: `0x00000000` — The reference count for the DLL. A value of `0x00000000` suggests that the file may not be frequently used or has not been referenced yet.

### 3. Load Status
- **LDRP_LOAD_NOTIFICATIONS_SENT**: This flag indicates that notifications regarding the loading of the DLL have been sent. It is part of the operating system's module loading mechanism.
  
- **LDRP_IMAGE_DLL**: This flag indicates that the file is recognized by the system as an image DLL, despite having a `.exe` extension. This behavior might occur if the executable exhibits DLL-like characteristics or actions.

## Analysis

The file `longth-tool.exe` being loaded into memory with flags such as `LDRP_IMAGE_DLL` could suggest that the file is attempting to disguise itself as a DLL to evade detection. This technique is common among rootkits and other malicious tools.

The abnormal value of `LoadCount` (0xffffffff) raises suspicion about irregularities in the loading process, possibly signaling the use of anti-analysis techniques or malicious behavior intended to remain undetected.

### Display detailed information about a module lmDvmlongth_tool
```assembly
lkd> lmDvmlongth_tool
Browse full module list
start             end                 module name
00007ff7`4a6a0000 00007ff7`4a6fa000   longth_tool   (deferred)             
    Image path: C:\Users\Atomic_Redteam\Pictures\longth-tool.exe
    Image name: longth-tool.exe
    Browse all global symbols  functions  data  Symbol Reload
    Timestamp:        Thu Mar 13 17:34:02 2025 (67D2B49A)
    CheckSum:         010EA700
    ImageSize:        0005A000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables:
```
## Analytical Anomalies:
- **"deferred"**: This may indicate that the module is trying to avoid analysis by delaying the loading of debug information. This is a common technique in malware or rootkits to avoid detection.
### List detailed information about kernel32.dl module
```assembly
lm m kernel32
Browse full module list
start             end                 module name
00007ff9`92de0000 00007ff9`92e9d000   KERNEL32   (deferred)             
lkd> lmD
start             end                 module name
00007ff7`4a6a0000 00007ff7`4a6fa000   longth_tool   (no symbols)           
00007ff9`8f9a0000 00007ff9`8fa3e000   uxtheme    (deferred)             
00007ff9`91f70000 00007ff9`9207b000   gdi32full   (deferred)             
00007ff9`92080000 00007ff9`92180000   ucrtbase   (deferred)             
00007ff9`92200000 00007ff9`9229d000   msvcp_win   (deferred)             
00007ff9`922a0000 00007ff9`92568000   KERNELBASE   (deferred)             
00007ff9`92860000 00007ff9`92882000   win32u     (deferred)             
00007ff9`92890000 00007ff9`929a5000   MSCTF      (deferred)             
00007ff9`929b0000 00007ff9`92a5c000   ADVAPI32   (deferred)             
00007ff9`92ae0000 00007ff9`92b7e000   msvcrt     (deferred)             
00007ff9`92b80000 00007ff9`92c4d000   OLEAUT32   (deferred)             
00007ff9`92c50000 00007ff9`92d7b000   RPCRT4     (deferred)             
00007ff9`92de0000 00007ff9`92e9d000   KERNEL32   (deferred)             
00007ff9`93270000 00007ff9`9329a000   GDI32      (deferred)             
00007ff9`932b0000 00007ff9`932e0000   IMM32      (deferred)             
00007ff9`940d0000 00007ff9`94425000   combase    (deferred)             
00007ff9`94430000 00007ff9`94485000   SHLWAPI    (deferred)             
00007ff9`94540000 00007ff9`946e0000   USER32     (deferred)             
00007ff9`946e0000 00007ff9`9477b000   sechost    (deferred)             
00007ff9`94830000 00007ff9`94a25000   ntdll      (pdb symbols)          C:\ProgramData\Dbg\sym\ntdll.pdb\1AEE50051B9801A042E4A92E9D14828D1\ntdll.pdb
ffffcadf`46c00000 ffffcadf`46ed8000   win32kbase   (deferred)             
ffffcadf`47200000 ffffcadf`4729a000   win32k     (deferred)             
ffffcadf`47930000 ffffcadf`47ce6000   win32kfull   (deferred)             
ffffcadf`47cf0000 ffffcadf`47d3a000   cdd        (deferred)             
fffff801`0cfd0000 fffff801`0cff8000   mcupdate_AuthenticAMD   (deferred)             
fffff801`0d000000 fffff801`0d006000   hal        (deferred)             
fffff801`0d010000 fffff801`0d01b000   kd         (deferred)             
fffff801`0d020000 fffff801`0d047000   tm         (deferred)             
fffff801`0d050000 fffff801`0d0ba000   CLFS       (deferred)             
fffff801`0d0c0000 fffff801`0d0da000   PSHED      (deferred)             
fffff801`0d0e0000 fffff801`0d0eb000   BOOTVID    (deferred)             
fffff801`0d0f0000 fffff801`0d203000   clipsp     (deferred)             
fffff801`0d210000 fffff801`0d27f000   FLTMGR     (deferred)             
fffff801`0d280000 fffff801`0d2a9000   ksecdd     (deferred)             
fffff801`0d2b0000 fffff801`0d313000   msrpc      (deferred)             
fffff801`0d320000 fffff801`0d32e000   cmimcext   (deferred)             
fffff801`0d330000 fffff801`0d341000   werkernel   (deferred)             
fffff801`0d350000 fffff801`0d35c000   ntosext    (deferred)             
fffff801`0d360000 fffff801`0d373000   WDFLDR     (deferred)             
fffff801`0d380000 fffff801`0d38f000   SleepStudyHelper   (deferred)             
fffff801`0d390000 fffff801`0d3a1000   WppRecorder   (deferred)             
fffff801`0d3b0000 fffff801`0d3ca000   SgrmAgent   (deferred)             
fffff801`0da00000 fffff801`0ea46000   nt         (pdb symbols)          C:\ProgramData\Dbg\sym\ntkrnlmp.pdb\992A9A48F30EC2C58B01A5934DCE2D9C1\ntkrnlmp.pdb
fffff801`12800000 fffff801`128e3000   CI         (deferred)             
fffff801`128f0000 fffff801`129a7000   cng        (deferred)             
fffff801`129b0000 fffff801`12a82000   Wdf01000   (deferred)             
fffff801`12a90000 fffff801`12ab6000   acpiex     (deferred)             
fffff801`12ac0000 fffff801`12b0c000   mssecflt   (deferred)             
fffff801`12b10000 fffff801`12bdc000   ACPI       (deferred)             
fffff801`12be0000 fffff801`12bec000   WMILIB     (deferred)             
fffff801`12bf0000 fffff801`12bfb000   IntelTA    (deferred)             
fffff801`12c00000 fffff801`12c6b000   intelpep   (deferred)             
fffff801`12c70000 fffff801`12c87000   WindowsTrustedRT   (deferred)             
fffff801`12c90000 fffff801`12c9b000   WindowsTrustedRTProxy   (deferred)             
fffff801`12ca0000 fffff801`12cb4000   pcw        (deferred)             
fffff801`12cc0000 fffff801`12ccb000   msisadrv   (deferred)             
fffff801`12cd0000 fffff801`12d47000   pci        (deferred)             
fffff801`12d50000 fffff801`12d65000   vdrvroot   (deferred)             
fffff801`12d70000 fffff801`12d9f000   pdc        (deferred)             
fffff801`12da0000 fffff801`12db9000   CEA        (deferred)             
fffff801`12dc0000 fffff801`12df1000   partmgr    (deferred)             
fffff801`12e00000 fffff801`12eaa000   spaceport   (deferred)             
fffff801`12eb0000 fffff801`12ebb000   intelide   (deferred)             
fffff801`12ec0000 fffff801`12ed3000   PCIIDEX    (deferred)             
fffff801`12ee0000 fffff801`12ef9000   volmgr     (deferred)             
fffff801`12f00000 fffff801`12f63000   volmgrx    (deferred)             
fffff801`12f70000 fffff801`12f88000   vsock      (deferred)             
fffff801`12f90000 fffff801`12fac000   vmci       (deferred)             
fffff801`12fb0000 fffff801`12fce000   mountmgr   (deferred)             
fffff801`12fd0000 fffff801`12fdd000   atapi      (deferred)             
fffff801`12fe0000 fffff801`1301c000   ataport    (deferred)             
fffff801`13020000 fffff801`13052000   storahci   (deferred)             
fffff801`13060000 fffff801`13113000   storport   (deferred)             
fffff801`13120000 fffff801`1314b000   stornvme   (deferred)             
fffff801`13150000 fffff801`1316c000   EhStorClass   (deferred)             
fffff801`13170000 fffff801`1318a000   fileinfo   (deferred)             
fffff801`13190000 fffff801`131d0000   Wof        (deferred)             
fffff801`131e0000 fffff801`134b9000   Ntfs       (deferred)             
fffff801`134c0000 fffff801`134cd000   Fs_Rec     (deferred)             
fffff801`134d0000 fffff801`1363f000   ndis       (deferred)             
fffff801`13640000 fffff801`136d8000   NETIO      (deferred)             
fffff801`136e0000 fffff801`13712000   ksecpkg    (deferred)             
fffff801`13720000 fffff801`13a0b000   tcpip      (deferred)             
fffff801`13a10000 fffff801`13a8f000   fwpkclnt   (deferred)             
fffff801`13a90000 fffff801`13ac0000   wfplwfs    (deferred)             
fffff801`13ad0000 fffff801`13b98000   fvevol     (deferred)             
fffff801`13ba0000 fffff801`13bab000   volume     (deferred)             
fffff801`13bb0000 fffff801`13c1d000   volsnap    (deferred)             
fffff801`13c20000 fffff801`13c4e000   SysmonDrv   (deferred)             
fffff801`13c50000 fffff801`13ca0000   rdyboost   (deferred)             
fffff801`13cb0000 fffff801`13cd6000   mup        (deferred)             
fffff801`13ce0000 fffff801`13cf2000   iorate     (deferred)             
fffff801`13d20000 fffff801`13d3c000   disk       (deferred)             
fffff801`13d40000 fffff801`13dac000   CLASSPNP   (deferred)             
fffff801`14600000 fffff801`1462b000   dump_stornvme   (deferred)             
fffff801`14650000 fffff801`1466d000   dump_dumpfve   (deferred)             
fffff801`14670000 fffff801`146a0000   cdrom      (deferred)             
fffff801`146b0000 fffff801`146c5000   filecrypt   (deferred)             
fffff801`146d0000 fffff801`146de000   tbs        (deferred)             
fffff801`146e0000 fffff801`146ea000   Null       (deferred)             
fffff801`146f0000 fffff801`146fa000   Beep       (deferred)             
fffff801`14700000 fffff801`14711000   vmrawdsk   (deferred)             
fffff801`14720000 fffff801`147b4000   csc        (deferred)             
fffff801`147c0000 fffff801`147d2000   nsiproxy   (deferred)             
fffff801`147e0000 fffff801`147f0000   mssmbios   (deferred)             
fffff801`14800000 fffff801`1480a000   gpuenergydrv   (deferred)             
fffff801`14810000 fffff801`1483c000   dfsc       (deferred)             
fffff801`14840000 fffff801`14855000   umbus      (deferred)             
fffff801`14860000 fffff801`148cc000   fastfat    (deferred)             
fffff801`148d0000 fffff801`148e7000   bam        (deferred)             
fffff801`148f0000 fffff801`1493e000   ahcache    (deferred)             
fffff801`14940000 fffff801`14952000   CompositeBus   (deferred)             
fffff801`14960000 fffff801`1496d000   kdnic      (deferred)             
fffff801`14990000 fffff801`149ae000   crashdmp   (deferred)             
fffff801`14a00000 fffff801`14a1b000   rspndr     (deferred)             
fffff801`14a70000 fffff801`14a99000   luafv      (deferred)             
fffff801`14aa0000 fffff801`14ad6000   wcifs      (deferred)             
fffff801`14ae0000 fffff801`14b62000   cldflt     (deferred)             
fffff801`14b70000 fffff801`14b8a000   storqosflt   (deferred)             
fffff801`14b90000 fffff801`14bb8000   bindflt    (deferred)             
fffff801`14bc0000 fffff801`14bd8000   lltdio     (deferred)             
fffff801`14be0000 fffff801`14bf8000   mslldp     (deferred)             
fffff801`14e00000 fffff801`14e11000   BasicRender   (deferred)             
fffff801`14e20000 fffff801`14e3c000   Npfs       (deferred)             
fffff801`14e40000 fffff801`14e51000   Msfs       (deferred)             
fffff801`14e60000 fffff801`14e7b000   CimFS      (deferred)             
fffff801`14e80000 fffff801`14ea2000   tdx        (deferred)             
fffff801`14eb0000 fffff801`14ec0000   TDI        (deferred)             
fffff801`14ed0000 fffff801`14ede000   ws2ifsl    (deferred)             
fffff801`14ee0000 fffff801`14f3c000   netbt      (deferred)             
fffff801`14f40000 fffff801`14f53000   afunix     (deferred)             
fffff801`14f60000 fffff801`15003000   afd        (deferred)             
fffff801`15010000 fffff801`1502a000   vwififlt   (deferred)             
fffff801`15030000 fffff801`1505b000   pacer      (deferred)             
fffff801`15060000 fffff801`15074000   ndiscap    (deferred)             
fffff801`15080000 fffff801`15094000   netbios    (deferred)             
fffff801`150a0000 fffff801`15141000   Vid        (deferred)             
fffff801`15150000 fffff801`15171000   winhvr     (deferred)             
fffff801`15180000 fffff801`151fb000   rdbss      (deferred)             
fffff801`15200000 fffff801`155a5000   dxgkrnl    (deferred)             
fffff801`155b0000 fffff801`155c8000   watchdog   (deferred)             
fffff801`155d0000 fffff801`155e6000   BasicDisplay   (deferred)             
fffff801`155f0000 fffff801`155fe000   npsvctrig   (deferred)             
fffff801`15600000 fffff801`1560a000   vm3dmp_loader   (deferred)             
fffff801`15610000 fffff801`15668000   vm3dmp     (deferred)             
fffff801`15670000 fffff801`15680000   usbuhci    (deferred)             
fffff801`15690000 fffff801`15709000   USBPORT    (deferred)             
fffff801`15710000 fffff801`15735000   HDAudBus   (deferred)             
fffff801`15740000 fffff801`157a6000   portcls    (deferred)             
fffff801`157b0000 fffff801`157d1000   drmk       (deferred)             
fffff801`157e0000 fffff801`15856000   ks         (deferred)             
fffff801`15860000 fffff801`1587a000   usbehci    (deferred)             
fffff801`15880000 fffff801`1590e000   e1i65x64   (deferred)             
fffff801`15910000 fffff801`159a8000   USBXHCI    (deferred)             
fffff801`159b0000 fffff801`159f4000   ucx01000   (deferred)             
fffff801`15a00000 fffff801`15a0b000   vmgencounter   (deferred)             
fffff801`15a10000 fffff801`15a1f000   CmBatt     (deferred)             
fffff801`15a20000 fffff801`15a30000   BATTC      (deferred)             
fffff801`15a40000 fffff801`15a7b000   amdppm     (deferred)             
fffff801`15a80000 fffff801`15a8d000   NdisVirtualBus   (deferred)             
fffff801`15a90000 fffff801`15a9c000   swenum     (deferred)             
fffff801`15aa0000 fffff801`15aae000   rdpbus     (deferred)             
fffff801`15ab0000 fffff801`15b35000   usbhub     (deferred)             
fffff801`15b40000 fffff801`15b4e000   USBD       (deferred)             
fffff801`15b50000 fffff801`15bf5000   UsbHub3    (deferred)             
fffff801`15c00000 fffff801`15c6f000   HdAudio    (deferred)             
fffff801`15c70000 fffff801`15c7f000   ksthunk    (deferred)             
fffff801`15c80000 fffff801`15cb3000   usbccgp    (deferred)             
fffff801`15cc0000 fffff801`15cd2000   hidusb     (deferred)             
fffff801`15ce0000 fffff801`15d1f000   HIDCLASS   (deferred)             
fffff801`15d20000 fffff801`15d33000   HIDPARSE   (deferred)             
fffff801`15d40000 fffff801`15d50000   mouhid     (deferred)             
fffff801`15d60000 fffff801`15d69000   vmusbmouse   (deferred)             
fffff801`15d70000 fffff801`15d91000   BTHUSB     (deferred)             
fffff801`15da0000 fffff801`15f24000   BTHport    (deferred)             
fffff801`15f30000 fffff801`15fa3000   dxgmms1    (deferred)             
fffff801`15fb0000 fffff801`16091000   dxgmms2    (deferred)             
fffff801`160a0000 fffff801`160bb000   monitor    (deferred)             
fffff801`160c0000 fffff801`160fd000   rfcomm     (deferred)             
fffff801`16100000 fffff801`16122000   BthEnum    (deferred)             
fffff801`16130000 fffff801`16156000   bthpan     (deferred)             
fffff801`16170000 fffff801`1617e000   dump_dumpstorport   (deferred)             
fffff801`16180000 fffff801`161a1000   i8042prt   (deferred)             
fffff801`161b0000 fffff801`161c4000   kbdclass   (deferred)             
fffff801`161d0000 fffff801`161d9000   vmmouse    (deferred)             
fffff801`161e0000 fffff801`161f3000   mouclass   (deferred)             
fffff801`16200000 fffff801`16388000   HTTP       (deferred)             
fffff801`16390000 fffff801`163b5000   bowser     (deferred)             
fffff801`163c0000 fffff801`16454000   mrxsmb     (deferred)             
fffff801`16460000 fffff801`1647a000   mpsdrv     (deferred)             
fffff801`16480000 fffff801`164c5000   mrxsmb20   (deferred)             
fffff801`164d0000 fffff801`164da000   vmmemctl   (deferred)             
fffff801`164e0000 fffff801`16533000   srvnet     (deferred)             
fffff801`16540000 fffff801`16554000   mmcss      (deferred)             
fffff801`16560000 fffff801`165b2000   mrxsmb10   (deferred)             
fffff801`165c0000 fffff801`16687000   srv2       (deferred)             
fffff801`16690000 fffff801`166b7000   Ndu        (deferred)             
fffff801`166c0000 fffff801`16796000   peauth     (deferred)             
fffff801`167a0000 fffff801`167b5000   tcpipreg   (deferred)             
fffff801`167c0000 fffff801`16856000   srv        (deferred)             
fffff801`16860000 fffff801`16872000   condrv     (deferred)             
fffff801`16880000 fffff801`168ac000   vmhgfs     (deferred)             
fffff801`168b0000 fffff801`168bb000   kprocesshacker   (deferred)             
fffff801`168c0000 fffff801`168c9000   kldbgdrv   (deferred)             
fffff801`16f30000 fffff801`16f86000   msquic     (deferred)             

Unloaded modules:
fffff801`149c0000 fffff801`149cf000   dump_storport.sys
fffff801`14600000 fffff801`1462c000   dump_stornvme.sys
fffff801`14650000 fffff801`1466e000   dump_dumpfve.sys
fffff801`14840000 fffff801`1485c000   dam.sys 
fffff801`13d00000 fffff801`13d11000   hwpolicy.sys
```
## Key Points to Note:

### Module `longth_tool` (No Symbols)

- **Missing Symbols**: The absence of symbols in the `longth_tool` module indicates that the executable file `longth-tool.exe` may not have debug information or a Program Database (PDB) file associated with it. This makes the analysis of the module more challenging and could be a sign of a tool attempting to hide its presence.

- **Memory Address**: The module is loaded into memory with the starting address `00007ff74a6a0000` and ending at `00007ff74a6fa000`, with a size of approximately 368 KB. While not particularly large, this is enough space to potentially harbor malicious code or employ anti-analysis techniques.

### Summary:
The lack of symbols in the `longth_tool` module could suggest that the file is designed to evade easy analysis, which should raise suspicion regarding the legitimacy of the tool.

### System Modules in "Deferred" State:
Modules such as `uxtheme`, `gdi32full`, `ucrtbase`, and `msvcp_win` are marked as "deferred," indicating they have not been fully loaded or used yet. This could also suggest that these modules are either not fully executed or are being affected by some form of concealment.

### Key Indicators for Potential Hooking:

- **Missing Symbols**: The absence of symbols is a typical anti-analysis strategy that makes it more difficult to understand or analyze the source code of the module. This could be a method to hide malicious activity.
  
- **Hooking Technique**: Malicious tools often use "hooking" techniques, where they attach functions at specific points in the system to alter the behavior of system calls without modifying the original code. The absence of symbols makes it difficult to detect and analyze such behaviors, increasing the likelihood of malicious code remaining undetected.

The combination of missing symbols and system modules being affected is a suspicious indicator that warrants further investigation to uncover any potentially harmful actions.

## Conclusion

Based on the analysis of the `longth_tool` module, several key indicators point to the likelihood of malicious activity, specifically the use of a rootkit.

1. **Missing Symbols**: The absence of symbol information for the `longth_tool` module makes it difficult to analyze and understand the internal workings of the executable. This is a common technique used by malware to evade detection and hinder analysis efforts.

2. **Potential Hooking**: The module's interaction with critical system libraries such as `ntdll` and `kernel32` raises suspicions of hooking techniques. Hooking allows malicious code to intercept and alter the behavior of system functions, often without being detected. This could be a method to hide malicious activities, including the concealment of processes and files.

3. **Process and File Hiding**: The observed behavior, including interaction with low-level system modules and the lack of symbols, suggests that `longth_tool` may be using anti-analysis techniques, potentially hiding processes and files from the system and security tools.

### Conclusion: Rootkit Activity
Given these signs—missing symbols, hooking techniques, and the potential for hiding processes and files—it is strongly suspected that `longth_tool` is behaving as a rootkit. Rootkits are sophisticated tools designed to maintain privileged access to a system while avoiding detection. The presence of such behaviors warrants immediate attention and further investigation to confirm the tool's malicious intent and prevent further compromise of the system.
