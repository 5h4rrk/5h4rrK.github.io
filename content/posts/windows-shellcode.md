+++
title = 'Windows Shellcode Development and Debugging with WinDbg - Part 1'
date = 2025-01-12T16:22:44+05:30
draft = false
tags = ['Windows Shellcode', 'Malware Analysis', 'Exploit Development', 'Assembly', 'WinDbg', 'Reverse Engineering', 'Debugging',  'Low-Level', 'Windows Internals']
summary = 'A comprehensive guide to understanding and creating Windows shellcode from scratch for exploit development. This article includes practical insights into using WinDbg for effective debugging and analysis.'
+++


### Exploring Strucutures 

#### TEB(Thread Environment Block)

```bash
0:000> dt nt!_TEB
ntdll!_TEB
   +0x000 NtTib            : _NT_TIB
   +0x038 EnvironmentPointer : Ptr64 Void
   +0x040 ClientId         : _CLIENT_ID
   +0x050 ActiveRpcHandle  : Ptr64 Void
   +0x058 ThreadLocalStoragePointer : Ptr64 Void
   +0x060 ProcessEnvironmentBlock : Ptr64 _PEB
   ....
   .... 

```

#### PEB(Process Environment Block)

```bash 
0:023> dt nt!_PEB -y LDR
    ntdll!_PEB
        +0x018 Ldr : Ptr64 _PEB_LDR_DATA     
```
##### PEB_LDR_DATA
```bash
0:023> dt nt!_PEB_LDR_DATA
ntdll!_PEB_LDR_DATA
   +0x000 Length           : Uint4B
   +0x004 Initialized      : UChar
   +0x008 SsHandle         : Ptr64 Void
   +0x010 InLoadOrderModuleList : _LIST_ENTRY
   +0x020 InMemoryOrderModuleList : _LIST_ENTRY
   +0x030 InInitializationOrderModuleList : _LIST_ENTRY
   +0x040 EntryInProgress  : Ptr64 Void
   +0x048 ShutdownInProgress : UChar
   +0x050 ShutdownThreadId : Ptr64 Void
```
##### LDR_DATA_TABLE_ENTRY

```bash
0:023> dt nt!_LDR_DATA_TABLE_ENTRY
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY
   +0x010 InMemoryOrderLinks : _LIST_ENTRY
   +0x020 InInitializationOrderLinks : _LIST_ENTRY
   +0x030 DllBase          : Ptr64 Void
   +0x038 EntryPoint       : Ptr64 Void
   +0x040 SizeOfImage      : Uint4B
   +0x048 FullDllName      : _UNICODE_STRING
   +0x058 BaseDllName      : _UNICODE_STRING
```

#### Parsing the PEB Structure

```bash
0:000> dt nt!_PEB (@rax)
ntdll!_PEB
   +0x000 InheritedAddressSpace : 0 ''
   +0x001 ReadImageFileExecOptions : 0 ''
   +0x002 BeingDebugged    : 0x1 ''
   +0x003 BitField         : 0x4 ''
   +0x003 ImageUsesLargePages : 0y0
   +0x003 IsProtectedProcess : 0y0
   +0x003 IsImageDynamicallyRelocated : 0y1
   +0x003 SkipPatchingUser32Forwarders : 0y0
   +0x003 IsPackagedProcess : 0y0
   +0x003 IsAppContainer   : 0y0
   +0x003 IsProtectedProcessLight : 0y0
   +0x003 IsLongPathAwareProcess : 0y0
   +0x004 Padding0         : [4]  ""
   +0x008 Mutant           : 0xffffffff`ffffffff Void
   +0x010 ImageBaseAddress : 0x00007ff6`14910000 Void
   +0x018 Ldr              : 0x00007ff8`d51108c0 _PEB_LDR_DATA
   +0x020 ProcessParameters : 0x0000019b`49df57b0 _RTL_USER_PROCESS_PARAMETERS

```

#### Parsing the PEB_LDR_DATA Structure
```bash
0:000> dt nt!_PEB_LDR_DATA 0x00007ff8`d51108c0
ntdll!_PEB_LDR_DATA
   +0x000 Length           : 0x58
   +0x004 Initialized      : 0x1 ''
   +0x008 SsHandle         : (null) 
   +0x010 InLoadOrderModuleList : _LIST_ENTRY [ 0x0000019b`49df62f0 - 0x0000019b`49df70d0 ]
   +0x020 InMemoryOrderModuleList : _LIST_ENTRY [ 0x0000019b`49df6300 - 0x0000019b`49df70e0 ]
   +0x030 InInitializationOrderModuleList : _LIST_ENTRY [ 0x0000019b`49df6060 - 0x0000019b`49df68e0 ]
   +0x040 EntryInProgress  : (null) 
   +0x048 ShutdownInProgress : 0 ''
   +0x050 ShutdownThreadId : (null) 

```

#### Parsing the LDR_DATA_TABLE_ENTRY Structure
```bash
0:000>  dt nt!_LDR_DATA_TABLE_ENTRY 0x0000019b`49df6300
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x0000019b`49df6050 - 0x00007ff8`d51108e0 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ] <-----------  While Travesing, it points to 0x10 Offset .So, subtract it
```

The InMemoryOrderLinks is located at 0x10 offset. Subtract it with 0x10 for proper alignment.

```bash
0:000> dt nt!_LDR_DATA_TABLE_ENTRY 0x0000019b`49df6300-10
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x0000019b`49df6040 - 0x00007ff8`d51108d0 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x0000019b`49df6050 - 0x00007ff8`d51108e0 ] 
   +0x020 InInitializationOrderLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ]
   +0x030 DllBase          : 0x00007ff6`14910000 Void
   +0x038 EntryPoint       : 0x00007ff6`14911000 Void
   +0x040 SizeOfImage      : 0x5000
   +0x048 FullDllName      : _UNICODE_STRING "F:\temp\execute.exe"
   +0x058 BaseDllName      : _UNICODE_STRING "execute.exe"
   +0x068 FlagGroup        : [4]  "???"
    ....
   +0x0c0 SwitchBackContext : 0x00007ff8`d50b1004 Void
   +0x0c8 BaseAddressIndexNode : _RTL_BALANCED_NODE
```

```bash
0:000>  dt nt!_LDR_DATA_TABLE_ENTRY 0x0000019b`49df6050-10
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x0000019b`49df68c0 - 0x0000019b`49df62f0 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x0000019b`49df68d0 - 0x0000019b`49df6300 ]
   +0x020 InInitializationOrderLinks : _LIST_ENTRY [ 0x0000019b`49df70f0 - 0x00007ff8`d51108f0 ]
   +0x030 DllBase          : 0x00007ff8`d4f40000 Void
   +0x038 EntryPoint       : (null) 
   +0x040 SizeOfImage      : 0x263000
   +0x048 FullDllName      : _UNICODE_STRING "C:\WINDOWS\SYSTEM32\ntdll.dll"
   +0x058 BaseDllName      : _UNICODE_STRING "ntdll.dll"
```

```bash
0:000>  dt nt!_LDR_DATA_TABLE_ENTRY 0x0000019b`49df68d0-10
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x0000019b`49df70d0 - 0x0000019b`49df6040 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x0000019b`49df70e0 - 0x0000019b`49df6050 ]
   +0x020 InInitializationOrderLinks : _LIST_ENTRY [ 0x00007ff8`d51108f0 - 0x0000019b`49df70f0 ]
   +0x030 DllBase          : 0x00007ff8`d3930000 Void
   +0x038 EntryPoint       : 0x00007ff8`d395e120 Void
   +0x040 SizeOfImage      : 0xc8000
   +0x048 FullDllName      : _UNICODE_STRING "C:\WINDOWS\System32\KERNEL32.DLL"
   +0x058 BaseDllName      : _UNICODE_STRING "KERNEL32.DLL"
```

```bash
0:000>  dt nt!_LDR_DATA_TABLE_ENTRY 0x0000019b`49df70e0-10
ntdll!_LDR_DATA_TABLE_ENTRY
   +0x000 InLoadOrderLinks : _LIST_ENTRY [ 0x00007ff8`d51108d0 - 0x0000019b`49df68c0 ]
   +0x010 InMemoryOrderLinks : _LIST_ENTRY [ 0x00007ff8`d51108e0 - 0x0000019b`49df68d0 ]
   +0x020 InInitializationOrderLinks : _LIST_ENTRY [ 0x0000019b`49df68e0 - 0x0000019b`49df6060 ]
   +0x030 DllBase          : 0x00007ff8`d2310000 Void
   +0x038 EntryPoint       : 0x00007ff8`d2436da0 Void
   +0x040 SizeOfImage      : 0x3b2000
   +0x048 FullDllName      : _UNICODE_STRING "C:\WINDOWS\System32\KERNELBASE.dll"
   +0x058 BaseDllName      : _UNICODE_STRING "KERNELBASE.dll"
   +0x068 FlagGroup        : [4]  "???"
```


```bash
0: kd> dt nt!_UNICODE_STRING
   +0x000 Length           : Uint2B
   +0x002 MaximumLength    : Uint2B
   +0x008 Buffer           : Ptr64 Wchar
```

### Windows Shellcode

**Creating the simplest shellcode imaginable**

```rust
bits 64 
section .text
    global start 
start:
    mov rax, gs:[0x60]      ; TEB
    mov rax, [rax + 0x18]   ; PEB.Ldr
    mov rax, [rax + 0x20]   ; PEB.Ldr->InMemoryOrderModuleList 
    mov rax, [rax]          ; _LDR_DATA_TABLE_ENTRY
    sub rax, 0x10           ; _LDR_DATA_TABLE_ENTRY.InMemoryOrderLinks
    add rax, 0x58           ; BaseDll->_UNICODE_STRING 
    mov rax, [rax + 0x08]   ; BaseDll->_UNICODE_STRING.Buffer
    xor rcx, rcx 
    ret 
```

##### Building and Linking

Build it with Visual Studio 

```bash
nasm -f win64 -g -F cv8 -o shellcode.obj shellcode.asm
link /SUBSYSTEM:CONSOLE /ENTRY:start /OUT:execute.exe /DEBUG  shellcode.obj 
```

##### Debugging with WinDbg

Open it in WinDbg.

```bash
0:000> lm
start             end                 module name
00007ff7`6b490000 00007ff7`6b495000   execute  C (pdb symbols)
00007ff8`cd030000 00007ff8`cd0cc000   apphelp    (deferred)             
00007ff8`d2310000 00007ff8`d26c2000   KERNELBASE   (deferred)             
00007ff8`d3930000 00007ff8`d39f8000   KERNEL32   (deferred)             
00007ff8`d4f40000 00007ff8`d51a3000   ntdll      (pdb symbols)     
```

Inspect all the functions 

```bash
0:000> x execute!*
00007ff7`6b491000 execute!start (start)
```

Set a breakpoint at entrypoint 

```bash
0:000> bp $exentry

0:000> g
Breakpoint 0 hit
execute!start:
00007ff6`46f41000 65488b042560000000 mov   rax,qword ptr gs:[60h] gs:00000000`00000060=????????????????
```

Set a breakpoint at start function

```bash
0:000> bp 00007ff7`6b491000
0:000> bl
     0 e Disable Clear  00007ff7`6b491000      0001 (0001)  0:**** execute!start

```
Trace it in Debugger 

```bash
0:000> t
execute!start+0x9:
00007ff6`46f41009 488b4018        mov     rax,qword ptr [rax+18h] ds:0000008f`f4b28018={ntdll!PebLdr (00007ff8`d51108c0)}
0:000> t
execute!start+0xd:
00007ff6`46f4100d 488b4020        mov     rax,qword ptr [rax+20h] ds:00007ff8`d51108e0=000001d7b1ebaf70
0:000> t
execute!start+0x11:
00007ff6`46f41011 488b00          mov     rax,qword ptr [rax] ds:000001d7`b1ebaf70=000001d7b1ebacc0
0:000> t
execute!start+0x14:
00007ff6`46f41014 4883e810        sub     rax,10h
0:000> t
execute!start+0x18:
00007ff6`46f41018 4883c058        add     rax,58h
0:000> t
execute!start+0x1c:
00007ff6`46f4101c 488b4008        mov     rax,qword ptr [rax+8] ds:000001d7`b1ebad10={ntdll!`string (00007ff8`d50bc388)}
0:000> r
rax=000001d7b1ebad08 rbx=0000000000000000 rcx=0000008ff4b28000
rdx=00007ff646f41000 rsi=0000000000000000 rdi=0000000000000000
rip=00007ff646f4101c rsp=0000008ff493f848 rbp=0000000000000000
 r8=0000008ff4b28000  r9=00007ff646f41000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
execute!start+0x1c:
00007ff6`46f4101c 488b4008        mov     rax,qword ptr [rax+8] ds:000001d7`b1ebad10={ntdll!`string (00007ff8`d50bc388)}
0:000> dt nt!_UNICODE_STRING @rax
ntdll!_UNICODE_STRING
 "ntdll.dll"
   +0x000 Length           : 0x12
   +0x002 MaximumLength    : 0x14
   +0x008 Buffer           : 0x00007ff8`d50bc388  "ntdll.dll"

0:000> t
execute!start+0x20:
00007ff6`46f41020 4831c9          xor     rcx,rcx

0:000> du @rax
00007ff8`d50bc388  "ntdll.dll"

0:000> t
execute!start+0x23:
00007ff6`46f41023 c3              ret
```

#### Extending the Shellcode

**Extending the above code**

- Added traversing the loaded modules  
- Iterating over each module name.

```rust
bits 64 
section .text
    global start 

start:
    mov rax, gs:[0x60]       ; TEB
    mov rax, [rax + 0x18]    ; PEB.Ldr
    mov rax, [rax + 0x20]    ; PEB.Ldr->InMemoryOrderModuleList 
    
repeat:
    mov rax, [rax]           ; _LDR_DATA_TABLE_ENTRY
    mov rcx, [rax + 0x40]    ; _LDR_DATA_TABLE_ENTRY.SizeOfImage
    cmp rcx, 0x0        
    je done
    sub rax, 0x10            ; _LDR_DATA_TABLE_ENTRY.InMemoryOrderLinks
    add rax, 0x58            ; BaseDll->_UNICODE_STRING 
    jmp find_names

find_names:
    mov rcx, [rax]          ; BaseDll->_UNICODE_STRING.Length
    and rcx , 0x0fff        ; BaseDll->_UNICODE_STRING.Length & 0x0fff (DWORD) // Not Neccessary !!
    mov rbx , [rax  + 0x08] ; BaseDll->_UNICODE_STRING.Buffer

iterate_chars:  
     sub rcx, 0x02          ; UTF-16 Chars
     cmp rcx, 0x00          ; Counter Check
     je next_iteration      
     jmp iterate_chars

next_iteration:
    sub rax, 0x48   ; _LDR_DATA_TABLE_ENTRY.InMemoryOrderLinks
    mov rax, [rax ] ; _LIST_ENTRY 
    jmp repeat

done:
    ret 
```

##### Debugging with WinDbg

Again attach a debugger

Set a breakpoint at `execute!start+0x42`

```bash
0:000> bp execute!start+0x42
0:000> bl
     0 e Disable Clear  00007ff7`bcf01000      0001 (0001)  0:**** execute!start
     1 e Disable Clear  00007ff7`bcf01042      0001 (0001)  0:**** execute!start+0x42

0:000> g
Breakpoint 0 hit
execute!start:
00007ff7`bcf01000 65488b042560000000 mov   rax,qword ptr gs:[60h] gs:00000000`00000060=????????????????

0:000> g
Breakpoint 1 hit
execute!start+0x42:
00007ff7`bcf01042 4883e848        sub     rax,48h
```

Inspect the registers

```bash
0:000> r
rax=00000222ed066098 rbx=00007ff8d50bc388 rcx=0000000000000000
rdx=00007ff7bcf01000 rsi=0000000000000000 rdi=0000000000000000
rip=00007ff7bcf01042 rsp=000000483a6ffd38 rbp=0000000000000000
 r8=000000483a544000  r9=00007ff7bcf01000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
execute!start+0x42:
00007ff7`bcf01042 4883e848        sub     rax,48h
```

```bash
0:000> dt nt!_UNICODE_STRING @rax
ntdll!_UNICODE_STRING
 "ntdll.dll"
   +0x000 Length           : 0x12
   +0x002 MaximumLength    : 0x14
   +0x008 Buffer           : 0x00007ff8`d50bc388  "ntdll.dll"
```
Unicode Buffer is at 0x08. Add 0x08 to rax.

```bash
0:000> du poi(@rax+0x08)
00007ff8`d50bc388  "ntdll.dll"
```

Continue it

```bash
0:000> g
Breakpoint 1 hit
execute!start+0x42:
00007ff7`bcf01042 4883e848        sub     rax,48h

0:000> du poi(@rax+0x08)
00000222`ed0672e8  "KERNELBASE.dll"

```

Continue

```bash
0:000> g
Breakpoint 1 hit
execute!start+0x42:
00007ff7`bcf01042 4883e848        sub     rax,48h

0:000> du poi(@rax+0x08)
00000222`ed065e10  "execute.exe"
```

Again Continue

```bash
0:000> g
Breakpoint 1 hit
execute!start+0x42:
00007ff7`bcf01042 4883e848        sub     rax,48h

0:000> du poi(@rax+0x08)
00000222`ed066ad8  "KERNEL32.DLL"
```

Process Terminates

```bash
0:000> g
ntdll!NtTerminateProcess+0x14:
00007ff8`d509fca4 c3              ret
```

Check all the loaded modules 

```bash
0:000> lm
start             end                 module name
00007ff7`bcf00000 00007ff7`bcf05000   execute  C (pdb symbols)          
00007ff8`d2310000 00007ff8`d26c2000   KERNELBASE   (deferred)             
00007ff8`d3930000 00007ff8`d39f8000   KERNEL32   (pdb symbols)          
00007ff8`d4f40000 00007ff8`d51a3000   ntdll      (pdb symbols)          
   
```