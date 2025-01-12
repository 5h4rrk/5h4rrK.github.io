+++
title = 'Decoding PE Files: A Step-by-Step Guide with WinDbg - Part 2'
date = 2025-01-12
tags = ['WinDbg', 'PE', "Portable Executable", "PE Parsing", 'Debugging', 'Reverse Engineering', 'Windows Internals']
summary = 'Hands on parsing PE files with WinDbg and explore their structures for debugging and reverse engineering.'
draft = false
+++

#### PE Import Directory Parsing 

```bash
0: kd> !process 0 0 msedge.exe
PROCESS ffff81822cc0a080
    SessionId: 1  Cid: 0f08    Peb: dbc4685000  ParentCid: 1b48
    DirBase: 3f0ae002  ObjectTable: ffffcc89acaf8e40  HandleCount: 175.
    Image: msedge.exe

0: kd> .process /p /r ffff81822cc0a080
Implicit process is now ffff8182`2cc0a080
.cache forcedecodeuser done
Loading User Symbols
.......................
```

```bash
0: kd> lm v m HTTP
Browse full module list
start             end                 module name
fffff800`15e50000 fffff800`15fd8000   HTTP       (no symbols)           
    Loaded symbol image file: HTTP.sys
    Mapped memory image file: c:\programdata\dbg\sym\HTTP.sys\43E8008A188000\HTTP.sys
    Image path: \SystemRoot\system32\drivers\HTTP.sys
    Image name: HTTP.sys
    Browse all global symbols  functions  data  Symbol Reload
    Image was built with /Brepro flag.
    Timestamp:        43E8008A (This is a reproducible build file hash, not a timestamp)
    CheckSum:         00188BE7
    ImageSize:        00188000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables: 
```

ImageName : HTTP.sys

ImageBaseAddress : `fffff80015e5000`

`/Brepo` flag : In the context of building C/C++ application, it is related to incremental linking and dependency tracking.

#### Parsing it Manually

##### _IMAGE_DOS_HEADER

```bash
0: kd> db fffff800`15e50000
fffff800`15e50000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
fffff800`15e50010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
fffff800`15e50020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
fffff800`15e50030  00 00 00 00 00 00 00 00-00 00 00 00 d8 00 00 00  ................

0: kd> dt combase!_IMAGE_DOS_HEADER  fffff800`15e50000 
   +0x000 e_magic          : 0x5a4d
   +0x002 e_cblp           : 0x90
   +0x004 e_cp             : 3
   +0x006 e_crlc           : 0
   +0x008 e_cparhdr        : 4
   +0x00a e_minalloc       : 0
   +0x00c e_maxalloc       : 0xffff
   +0x00e e_ss             : 0
   +0x010 e_sp             : 0xb8
   +0x012 e_csum           : 0
   +0x014 e_ip             : 0
   +0x016 e_cs             : 0
   +0x018 e_lfarlc         : 0x40
   +0x01a e_ovno           : 0
   +0x01c e_res            : [4] 0
   +0x024 e_oemid          : 0
   +0x026 e_oeminfo        : 0
   +0x028 e_res2           : [10] 0
   +0x03c e_lfanew         : 0n216

0: kd> ? 0n216
Evaluate expression: 216 = 00000000`000000d8
```

##### DOS STUB & RICH HEADER

```bash
fffff807`2a4a0040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
fffff807`2a4a0050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
fffff807`2a4a0060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
fffff807`2a4a0070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......
fffff807`2a4a0080  bd 0a c4 77 f9 6b aa 24-f9 6b aa 24 f9 6b aa 24  ...w.k.$.k.$.k.$
fffff807`2a4a0090  f9 6b ab 24 53 6a aa 24-ed 00 ab 25 f6 6b aa 24  .k.$Sj.$...%.k.$
fffff807`2a4a00a0  ed 00 ae 25 f0 6b aa 24-ed 00 a9 25 fc 6b aa 24  ...%.k.$...%.k.$
fffff807`2a4a00b0  ed 00 a7 25 96 6b aa 24-ed 00 55 24 f8 6b aa 24  ...%.k.$..U$.k.$
fffff807`2a4a00c0  ed 00 a8 25 f8 6b aa 24-52 69 63 68 f9 6b aa 24  ...%.k.$Rich.k.$
fffff807`2a4a00d0  00 00 00 00 00 00 00 00                          ........
```
##### _IMAGE_NT_HEADERS64

```bash
0: kd> dt combase!_IMAGE_NT_HEADERS64 (fffff800`15e50000 + 0xd8)
   +0x000 Signature        : 0x4550
   +0x004 FileHeader       : _IMAGE_FILE_HEADER
   +0x018 OptionalHeader   : _IMAGE_OPTIONAL_HEADER64
```

##### _IMAGE_FILE_HEADER
```bash
0: kd> dt combase!_IMAGE_FILE_HEADER (fffff800`15e50000 + 0xd8 + 0x4)
   +0x000 Machine          : 0x8664
   +0x002 NumberOfSections : 0xe
   +0x004 TimeDateStamp    : 0x43e8008a
   +0x008 PointerToSymbolTable : 0
   +0x00c NumberOfSymbols  : 0
   +0x010 SizeOfOptionalHeader : 0xf0
   +0x012 Characteristics  : 0x22
```
###### DLL Characteristics

```bash
0: kd> .formats 0x22
Evaluate expression:
  Hex:     00000000`00000022
  Decimal: 34
  Decimal (unsigned) : 34
  Octal:   0000000000000000000042
  Binary:  00000000 00000000 00000000 00000000 00000000 00000000 00000000 00100010
  Chars:   ......."
  Time:    Thu Jan  1 05:30:34 1970
  Float:   low 4.76441e-044 high 0
  Double:  1.67982e-322
```

```bash
0: kd> ? (0x22 & 0x00f) == 0x40
Evaluate expression: 0 = 00000000`00000000
```
- IMAGE_DYNAMIC_BASE : **Disabled**

```bash
0: kd> ? (0x22 & 0x0f00) == 0x01
Evaluate expression: 0 = 00000000`00000000
```
- DATA_EXECUTION_PREVENTION(DEP) : **Disabled**

```bash
0: kd> ? (0x22 & 0x0f00) == 0x04
Evaluate expression: 0 = 00000000`00000000
```
- STRUCTURED_EXCEPTION_HANDLER: **Disabled**

##### _IMAGE_OPTIONAL_HEADER64

```bash
0: kd> dt combase!_IMAGE_OPTIONAL_HEADER64 (fffff800`15e50000 + 0xd8 + 0x18)
   +0x000 Magic            : 0x20b
   +0x002 MajorLinkerVersion : 0xe ''
   +0x003 MinorLinkerVersion : 0x14 ''
   +0x004 SizeOfCode       : 0x126c00
   +0x008 SizeOfInitializedData : 0x5b400
   +0x00c SizeOfUninitializedData : 0
   +0x010 AddressOfEntryPoint : 0x15f010
   +0x014 BaseOfCode       : 0x1000
   +0x018 ImageBase        : 0xfffff800`15e50000
   +0x020 SectionAlignment : 0x1000
   +0x024 FileAlignment    : 0x200
   +0x028 MajorOperatingSystemVersion : 0xa
   +0x02a MinorOperatingSystemVersion : 0
   +0x02c MajorImageVersion : 0xa
   +0x02e MinorImageVersion : 0
   +0x030 MajorSubsystemVersion : 0xa
   +0x032 MinorSubsystemVersion : 0
   +0x034 Win32VersionValue : 0
   +0x038 SizeOfImage      : 0x188000
   +0x03c SizeOfHeaders    : 0x600
   +0x040 CheckSum         : 0x188be7
   +0x044 Subsystem        : 1
   +0x046 DllCharacteristics : 0x4160
   +0x048 SizeOfStackReserve : 0x40000
   +0x050 SizeOfStackCommit : 0x1000
   +0x058 SizeOfHeapReserve : 0x100000
   +0x060 SizeOfHeapCommit : 0x1000
   +0x068 LoaderFlags      : 0
   +0x06c NumberOfRvaAndSizes : 0x10
   +0x070 DataDirectory    : [16] _IMAGE_DATA_DIRECTORY
```

##### Export Directory

```bash
0: kd> dt combase!_IMAGE_DATA_DIRECTORY (fffff800`15e50000 + 0xd8 + 0x18 + 0x70)
   +0x000 VirtualAddress   : 0
   +0x004 Size             : 0

```

##### Import Directory

```bash
0: kd> ?? sizeof(combase!_IMAGE_DATA_DIRECTORY)
unsigned int64 8

0: kd> dt combase!_IMAGE_DATA_DIRECTORY (fffff800`15e50000 + 0xd8 + 0x18 + 0x70 + 0x08)
   +0x000 VirtualAddress   : 0x87d90
   +0x004 Size             : 0xa0
```

##### _IMAGE_IMPORT_DESCRIPTOR

```c
typedef struct _IMAGE_IMPORT_DESCRIPTOR {
    union {
        DWORD Characteristics;    // 0 for terminating null import descriptor
        DWORD OriginalFirstThunk; // Import Address Name Table (INT)
    } DUMMYUNIONNAME;
    DWORD TimeDateStamp;          // Timestamp when the DLL was bound
    DWORD ForwarderChain;         // Index of the first forwarder chain, or -1
    DWORD Name;                   // ASCII string containing the DLL name
    DWORD FirstThunk;             // Import Address Table (IAT)
} IMAGE_IMPORT_DESCRIPTOR, *PIMAGE_IMPORT_DESCRIPTOR;

```

```bash
0: kd> dt combase!_IMAGE_IMPORT_DESCRIPTOR (fffff800`15e50000 + 0x87d90)
   +0x000 Characteristics  :  0x880d0  
   +0x000 OriginalFirstThunk : 0x880d0 // Points to Import Name Table
   +0x004 TimeDateStamp    : 0
   +0x008 ForwarderChain   : 0
   +0x00c Name             : 0x8aa38 // DLL Name
   +0x010 FirstThunk       : 0x872a0 //Points to Import Address Table
```

```bash
0: kd> da (fffff800`15e50000 + 0x8aa38)
fffff800`15edaa38  "ntoskrnl.exe"
```

##### NTOSKRNL.EXE

```bash
0: kd> dqs (fffff800`15e50000 + 0x880d0) L15C
fffff800`15ed80d0  00000000`000893ec
fffff800`15ed80d8  00000000`0008940a
fffff800`15ed80e0  00000000`00089426
fffff800`15ed80e8  00000000`00089446
fffff800`15ed80f0  00000000`0008945a
fffff800`15ed80f8  00000000`00089472
fffff800`15ed8100  00000000`00089480
.....
fffff800`15ed8b88  00000000`0008b438
fffff800`15ed8b90  00000000`0008b460
fffff800`15ed8b98  00000000`0008b488
fffff800`15ed8ba0  00000000`0008b4dc
fffff800`15ed8ba8  00000000`00000000
```

###### _IMAGE_IMPORT_BY_NAME

```bash
// Entry 0x00
0: kd> dt combase!_IMAGE_IMPORT_BY_NAME (fffff800`15e50000 + 00000000`000893ec)
   +0x000 Hint             : 0x815
   +0x002 Name             : [1]  "R"

0: kd> da (fffff800`15e50000 + 00000000`000893ec)
fffff800`15ed93ec  "..RtlEnumerateGenericTableAvl"

// Entry 0x1
0: kd> dt combase!_IMAGE_IMPORT_BY_NAME (fffff800`15e50000 + 00000000`0008940a)
   +0x000 Hint             : 0x4c4
   +0x002 Name             : [1]  "K"

0: kd> da (fffff800`15e50000 + 00000000`0008940a)
fffff800`15ed940a  "..KeQueryHighestNodeNumber"

//Entry 0x02 
0: kd> dt combase!_IMAGE_IMPORT_BY_NAME (fffff800`15e50000 + 00000000`00089426)
   +0x000 Hint             : 0x88f
   +0x002 Name             : [1]  "R"

0: kd> da (fffff800`15e50000 + 00000000`00089426)
fffff800`15ed9426  "..RtlInitializeGenericTableAvl"

// Entry 0x15a
0: kd> dt combase!_IMAGE_IMPORT_BY_NAME (fffff800`15e50000 + 00000000`0008b4dc)
   +0x000 Hint             : 0xc8
   +0x002 Name             : [1]  "E"

0: kd> da (fffff800`15e50000 + 00000000`0008b4dc)
fffff800`15edb4dc  "."

0: kd> da (fffff800`15e50000 + 00000000`0008b4dc+0x02)
fffff800`15edb4de  "ExFreeCacheAwareRundownProtectio"
fffff800`15edb4fe  "n"
```

###### FirstThunk Parsing 

```bash
// ImageBaseAddress + FirstThunk

: kd> db (fffff800`15e50000 +0x872a0)
fffff800`15ed72a0  50 f1 96 10 00 f8 ff ff-50 b1 8e 10 00 f8 ff ff  P.......P.......
fffff800`15ed72b0  30 bd 90 10 00 f8 ff ff-80 40 87 10 00 f8 ff ff  0........@......
fffff800`15ed72c0  70 74 86 10 00 f8 ff ff-90 51 86 10 00 f8 ff ff  pt.......Q......
fffff800`15ed72d0  30 74 9e 10 00 f8 ff ff-30 1a 8f 10 00 f8 ff ff  0t......0.......
fffff800`15ed72e0  60 6c bf 10 00 f8 ff ff-40 ca c8 10 00 f8 ff ff  `l......@.......
fffff800`15ed72f0  30 a4 f9 10 00 f8 ff ff-f0 a3 93 10 00 f8 ff ff  0...............
fffff800`15ed7300  10 52 94 10 00 f8 ff ff-60 1e 8d 10 00 f8 ff ff  .R......`.......
fffff800`15ed7310  60 5e 84 10 00 f8 ff ff-90 45 9e 10 00 f8 ff ff  `^.......E......

0: kd> dq (fffff800`15e50000 +0x872a0)
fffff800`15ed72a0  fffff800`1096f150 fffff800`108eb150
fffff800`15ed72b0  fffff800`1090bd30 fffff800`10874080
fffff800`15ed72c0  fffff800`10867470 fffff800`10865190
fffff800`15ed72d0  fffff800`109e7430 fffff800`108f1a30
fffff800`15ed72e0  fffff800`10bf6c60 fffff800`10c8ca40
fffff800`15ed72f0  fffff800`10f9a430 fffff800`1093a3f0
fffff800`15ed7300  fffff800`10945210 fffff800`108d1e60
fffff800`15ed7310  fffff800`10845e60 fffff800`109e4590

nt!RtlEnumerateGenericTableAvl:
fffff800`1096f150 4883ec28        sub     rsp,28h
fffff800`1096f154 84d2            test    dl,dl
fffff800`1096f156 7405            je      nt!RtlEnumerateGenericTableAvl+0xd (fffff800`1096f15d)
fffff800`1096f158 4883613800      and     qword ptr [rcx+38h],0
fffff800`1096f15d 488d5138        lea     rdx,[rcx+38h]
fffff800`1096f161 e8cac1f9ff      call    nt!RtlEnumerateGenericTableWithoutSplayingAvl (fffff800`1090b330)
fffff800`1096f166 4883c428        add     rsp,28h
fffff800`1096f16a c3              ret

.....
.....

0: kd> dq (fffff800`15e50000 +0x872a0 + 0x08 * (0x15a)) L1
fffff800`15ed7d70  fffff800`108162e0

0: kd> u fffff800`108162e0
nt!ExFreeCacheAwareRundownProtection:
fffff800`108162e0 4053            push    rbx
fffff800`108162e2 4883ec20        sub     rsp,20h
fffff800`108162e6 488bd9          mov     rbx,rcx
fffff800`108162e9 488b4908        mov     rcx,qword ptr [rcx+8]
fffff800`108162ed e8fed20400      call    nt!ExFreeHeapPool (fffff800`108635f0)
fffff800`108162f2 488bcb          mov     rcx,rbx
fffff800`108162f5 e8f6d20400      call    nt!ExFreeHeapPool (fffff800`108635f0)
fffff800`108162fa 4883c420        add     rsp,20h
```

`dqs` : display a sequence of quadwords along with symbols.

```bash
// ImageBaseAddress + FirstThunk
0: kd> dqs (fffff800`15e50000 + 0x872a0) L20
fffff800`15ed72a0  fffff800`1096f150 nt!RtlEnumerateGenericTableAvl
fffff800`15ed72a8  fffff800`108eb150 nt!KeQueryHighestNodeNumber
fffff800`15ed72b0  fffff800`1090bd30 nt!RtlInitializeGenericTableAvl
fffff800`15ed72b8  fffff800`10874080 nt!KeInitializeEvent
fffff800`15ed72c0  fffff800`10867470 nt!KeWaitForSingleObject
fffff800`15ed72c8  fffff800`10865190 nt!KeSetEvent
fffff800`15ed72d0  fffff800`109e7430 nt!wcschr
fffff800`15ed72d8  fffff800`108f1a30 nt!MmIsThisAnNtAsSystem
fffff800`15ed72e0  fffff800`10bf6c60 nt!RtlUpcaseUnicodeString
fffff800`15ed72e8  fffff800`10c8ca40 nt!RtlUpcaseUnicodeChar
fffff800`15ed72f0  fffff800`10f9a430 nt!NlsLeadByteInfo
fffff800`15ed72f8  fffff800`1093a3f0 nt!MmSizeOfMdl
fffff800`15ed7300  fffff800`10945210 nt!IoGetRequestorProcess
fffff800`15ed7308  fffff800`108d1e60 nt!KeStackAttachProcess
fffff800`15ed7310  fffff800`10845e60 nt!KeUnstackDetachProcess
fffff800`15ed7318  fffff800`109e4590 nt!vsnwprintf
fffff800`15ed7320  fffff800`10d321b0 nt!RtlUnicodeToUTF8N
fffff800`15ed7328  fffff800`109e4a60 nt!strncmp
fffff800`15ed7330  fffff800`10880060 nt!MmMapLockedPagesSpecifyCache
fffff800`15ed7338  fffff800`1085bfc0 nt!IoAllocateMdl
fffff800`15ed7340  fffff800`10932820 nt!MmBuildMdlForNonPagedPool
fffff800`15ed7348  fffff800`10ba29b0 nt!RtlQueryFeatureConfigurationChangeStamp
fffff800`15ed7350  fffff800`1099fae0 nt!RtlQueryFeatureConfiguration
fffff800`15ed7358  fffff800`10ba29c0 nt!RtlRegisterFeatureConfigurationChangeNotification
fffff800`15ed7360  fffff800`10f2fd90 nt!RtlUnregisterFeatureConfigurationChangeNotification
fffff800`15ed7368  fffff800`10a10a20 nt!ZwQueryWnfStateData
fffff800`15ed7370  fffff800`10fcb010 nt!ExAllocatePoolWithTag
fffff800`15ed7378  fffff800`10ba2990 nt!RtlNotifyFeatureUsage
fffff800`15ed7380  fffff800`1093ca40 nt!KeQueryMaximumProcessorCountEx
fffff800`15ed7388  fffff800`10d25dd0 nt!RtlGetVersion
fffff800`15ed7390  fffff800`10a0dfe0 nt!ZwOpenKey
fffff800`15ed7398  fffff800`10d31230 nt!RtlIsStateSeparationEnabled

```

Parsing `HTTP.sys` using [PEInsight](https://github.com/5h4rrk/PEInsight)

```bash
        ntoskrnl.exe
        ----------------
          Hint         Name                                                             |
        +-------------------------------------------------------------------------------+
        | 2069       | RtlEnumerateGenericTableAvl                                      |
        | 1220       | KeQueryHighestNodeNumber                                         |
        | 2191       | RtlInitializeGenericTableAvl                                     |
        | 1162       | KeInitializeEvent                                                |
        | 1335       | KeWaitForSingleObject                                            |
        | 1292       | KeSetEvent                                                       |
        | 3047       | wcschr                                                           |
        | 1431       | MmIsThisAnNtAsSystem                                             |
        | 2421       | RtlUpcaseUnicodeString                                           |
        | 2420       | RtlUpcaseUnicodeChar                                             |
        | 1496       | NlsLeadByteInfo                                                  |
        | 1479       | MmSizeOfMdl                                                      |
        | 849        | IoGetRequestorProcess                                            |
        | 1315       | KeStackAttachProcess                                             |
        | 1330       | KeUnstackDetachProcess                                           |
        | 2967       | _vsnwprintf                                                      |
        | 2414       | RtlUnicodeToUTF8N                                                |
        | 3025       | strncmp                                                          |
        | 1441       | MmMapLockedPagesSpecifyCache                                     |
        | 710        | IoAllocateMdl                                                    |
        ..........
        ..........
        ..........
        | 1258       | KeReleaseMutex                                                   |
        | 145        | ExAcquireRundownProtectionCacheAware                             |
        | 157        | ExAllocateCacheAwareRundownProtection                            |
        | 285        | ExReleaseRundownProtectionCacheAware                             |
        | 200        | ExFreeCacheAwareRundownProtection                                |
        +-------------------------------------------------------------------------------+
```

```bash
0: kd> ?? sizeof(_IMAGE_IMPORT_DESCRIPTOR)
unsigned int64 0x14
```
###### _IMAGE_IMPORT_DESCRIPTOR

```bash
0: kd> dt combase!_IMAGE_IMPORT_DESCRIPTOR  (fffff800`15e50000 + 0x87d90 + 0x14)
   +0x000 Characteristics  : 0x87e30
   +0x000 OriginalFirstThunk : 0x87e30 // Import Name Table
   +0x004 TimeDateStamp    : 0
   +0x008 ForwarderChain   : 0
   +0x00c Name             : 0x8aa62 // DLL Name
   +0x010 FirstThunk       : 0x87000 // Import Address Table
```

##### HAL.dll 

```bash
0: kd> da  (fffff800`15e50000 + 0x8aa62)
fffff800`15edaa62  "HAL.dll"
```


```bash
0: kd> dqs  (fffff800`15e50000 + 0x87e30)
fffff800`15ed7e30  00000000`0008aa46 
fffff800`15ed7e38  00000000`00000000 // Ends Entry for HAL.dll 
fffff800`15ed7e40  00000000`0008b1d6
fffff800`15ed7e48  00000000`0008b1b2

0: kd> db  (fffff800`15e50000 + 00000000`0008aa46)
fffff800`15edaa46  54 00 4b 65 51 75 65 72-79 50 65 72 66 6f 72 6d  T.KeQueryPerform
fffff800`15edaa56  61 6e 63 65 43 6f 75 6e-74 65 72 00 48 41 4c 2e  anceCounter.HAL.
fffff800`15edaa66  64 6c 6c 00 2c 00 46 72-65 65 43 6f 6e 74 65 78  dll.,.FreeContex
fffff800`15edaa76  74 42 75 66 66 65 72 00-4c 00 53 73 6c 47 65 74  tBuffer.L.SslGet
```

###### _IMAGE_IMPORT_BY_NAME

```bash
0: kd> dt combase!_IMAGE_IMPORT_BY_NAME (fffff800`15e50000 + 00000000`0008aa46)
   +0x000 Hint             : 0x54
   +0x002 Name             : [1]  "K"

// HINT
0: kd> da  (fffff800`15e50000 + 00000000`0008aa46)
fffff800`15edaa46  "T"

// NAME 
0: kd> da  (fffff800`15e50000 + 00000000`0008aa46 + 0x2)
fffff800`15edaa48  "KeQueryPerformanceCounter"

```
###### FirstThunk Parsing 

```bash
// ImageBaseAddress + FirstThunk
0: kd> dqs  (fffff800`15e50000 + 0x87000)
fffff800`15ed7000  fffff800`1083c380 nt!KeQueryPerformanceCounter
fffff800`15ed7008  00000000`00000000
fffff800`15ed7010  fffff800`128e1450 ndis!NdisGetJobObjectCompartmentId
fffff800`15ed7018  fffff800`12837a20 ndis!NdisGetThreadObjectCompartmentId
fffff800`15ed7020  00000000`00000000
```

Parsing `HTTP.sys` using [PEInsight](https://github.com/5h4rrk/PEInsight)

```bash
       HAL.dll
        ----------------
          Hint         Name                                                             |
        +-------------------------------------------------------------------------------+
        | 84         | KeQueryPerformanceCounter                                        |
        +-------------------------------------------------------------------------------+
```


#### Parsing it with Extension command

```bash
0: kd> !dh -f HTTP

File Type: EXECUTABLE IMAGE
FILE HEADER VALUES
    8664 machine (X64)
       E number of sections
43E8008A time date stamp Tue Feb  7 07:36:02 2006

       0 file pointer to symbol table
       0 number of symbols
      F0 size of optional header
      22 characteristics
            Executable
            App can handle >2gb addresses

OPTIONAL HEADER VALUES
     20B magic #
   14.20 linker version
  126C00 size of code
   5B400 size of initialized data
       0 size of uninitialized data
  15F010 address of entry point
    1000 base of code
         ----- new -----
fffff80015e50000 image base
    1000 section alignment
     200 file alignment
       1 subsystem (Native)
   10.00 operating system version
   10.00 image version
   10.00 subsystem version
  188000 size of image
     600 size of headers
  188BE7 checksum
0000000000040000 size of stack reserve
0000000000001000 size of stack commit
0000000000100000 size of heap reserve
0000000000001000 size of heap commit
    4160  DLL characteristics
            High entropy VA supported
            Dynamic base
            NX compatible
            Guard
       0 [       0] address [size] of Export Directory
   87D90 [      A0] address [size] of Import Directory
  162000 [   1D810] address [size] of Resource Directory
   7A000 [    C834] address [size] of Exception Directory
  180800 [    25F0] address [size] of Security Directory
  180000 [     97C] address [size] of Base Relocation Directory
   61590 [      54] address [size] of Debug Directory
       0 [       0] address [size] of Description Directory
       0 [       0] address [size] of Special Directory
       0 [       0] address [size] of Thread Storage Directory
   5C8A0 [     118] address [size] of Load Configuration Directory
       0 [       0] address [size] of Bound Import Directory
   87000 [     D80] address [size] of Import Address Table Directory
       0 [       0] address [size] of Delay Import Directory
       0 [       0] address [size] of COR20 Header Directory
       0 [       0] address [size] of Reserved Directory
```

```bash
0: kd> dps HTTP+87000
fffff800`15ed7000  fffff800`1083c380 nt!KeQueryPerformanceCounter
fffff800`15ed7008  00000000`00000000
fffff800`15ed7010  fffff800`128e1450 ndis!NdisGetJobObjectCompartmentId
fffff800`15ed7018  fffff800`12837a20 ndis!NdisGetThreadObjectCompartmentId
fffff800`15ed7020  00000000`00000000
fffff800`15ed7028  fffff800`129d8610 NETIO!NsiRegisterChangeNotification
fffff800`15ed7030  fffff800`129f00b0 NETIO!CancelMibChangeNotify2
fffff800`15ed7038  fffff800`129fab00 NETIO!NsiDeregisterChangeNotification
fffff800`15ed7040  fffff800`129d8420 NETIO!NotifyUnicastIpAddressChange
fffff800`15ed7048  fffff800`129f9530 NETIO!NmrClientDetachProviderComplete
fffff800`15ed7050  fffff800`129d16e0 NETIO!NmrClientAttachProvider
fffff800`15ed7058  fffff800`129f95f0 NETIO!NmrWaitForClientDeregisterComplete
fffff800`15ed7060  fffff800`129f9590 NETIO!NmrDeregisterClient
fffff800`15ed7068  fffff800`129d0bf0 NETIO!NmrRegisterClient
fffff800`15ed7070  fffff800`129d9890 NETIO!KfdFreeEnumHandle
fffff800`15ed7078  fffff800`129cc500 NETIO!KfdDerefFilterContext
.....
```

To dump the everything from executable 

```bash
!dh -a fffff800`15e50000 
```