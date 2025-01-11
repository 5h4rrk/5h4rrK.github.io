+++
title = 'PE Parsing with WinDbg'
date = 2025-01-11T17:00:50+05:30
tags = ['WinDbg', 'PE Parsing', 'Debugging', 'Reverse Engineering', 'Windows Internals']
summary = 'Hands on parsing PE files with WinDbg and explore their structures for debugging and reverse engineering.'
draft = true
+++

#### PE Structure 

```c
0: kd> dt combase!_IMAGE_DOS_HEADER 
   +0x000 e_magic          : Uint2B
   +0x002 e_cblp           : Uint2B
   +0x004 e_cp             : Uint2B
   +0x006 e_crlc           : Uint2B
   +0x008 e_cparhdr        : Uint2B
   +0x00a e_minalloc       : Uint2B
   +0x00c e_maxalloc       : Uint2B
   +0x00e e_ss             : Uint2B
   +0x010 e_sp             : Uint2B
   +0x012 e_csum           : Uint2B
   +0x014 e_ip             : Uint2B
   +0x016 e_cs             : Uint2B
   +0x018 e_lfarlc         : Uint2B
   +0x01a e_ovno           : Uint2B
   +0x01c e_res            : [4] Uint2B
   +0x024 e_oemid          : Uint2B
   +0x026 e_oeminfo        : Uint2B
   +0x028 e_res2           : [10] Uint2B
   +0x03c e_lfanew         : Int4B
```

```c
0: kd> dt combase!_IMAGE_NT_HEADERS64
   +0x000 Signature        : Uint4B
   +0x004 FileHeader       : _IMAGE_FILE_HEADER
   +0x018 OptionalHeader   : _IMAGE_OPTIONAL_HEADER64
```


```c
0: kd> dt combase!_IMAGE_FILE_HEADER
   +0x000 Machine          : Uint2B
   +0x002 NumberOfSections : Uint2B
   +0x004 TimeDateStamp    : Uint4B
   +0x008 PointerToSymbolTable : Uint4B
   +0x00c NumberOfSymbols  : Uint4B
   +0x010 SizeOfOptionalHeader : Uint2B
   +0x012 Characteristics  : Uint2B
```

```c
0: kd> dt combase!_IMAGE_OPTIONAL_HEADER64
   +0x000 Magic            : Uint2B
   +0x002 MajorLinkerVersion : UChar
   +0x003 MinorLinkerVersion : UChar
   +0x004 SizeOfCode       : Uint4B
   +0x008 SizeOfInitializedData : Uint4B
   +0x00c SizeOfUninitializedData : Uint4B
   +0x010 AddressOfEntryPoint : Uint4B
   +0x014 BaseOfCode       : Uint4B
   +0x018 ImageBase        : Uint8B
   +0x020 SectionAlignment : Uint4B
   +0x024 FileAlignment    : Uint4B
   +0x028 MajorOperatingSystemVersion : Uint2B
   +0x02a MinorOperatingSystemVersion : Uint2B
   +0x02c MajorImageVersion : Uint2B
   +0x02e MinorImageVersion : Uint2B
   +0x030 MajorSubsystemVersion : Uint2B
   +0x032 MinorSubsystemVersion : Uint2B
   +0x034 Win32VersionValue : Uint4B
   +0x038 SizeOfImage      : Uint4B
   +0x03c SizeOfHeaders    : Uint4B
   +0x040 CheckSum         : Uint4B
   +0x044 Subsystem        : Uint2B
   +0x046 DllCharacteristics : Uint2B
   +0x048 SizeOfStackReserve : Uint8B
   +0x050 SizeOfStackCommit : Uint8B
   +0x058 SizeOfHeapReserve : Uint8B
   +0x060 SizeOfHeapCommit : Uint8B
   +0x068 LoaderFlags      : Uint4B
   +0x06c NumberOfRvaAndSizes : Uint4B
   +0x070 DataDirectory    : [16] _IMAGE_DATA_DIRECTORY

```

```c
0: kd> dt combase!_IMAGE_DATA_DIRECTORY 
   +0x000 VirtualAddress   : Uint4B
   +0x004 Size             : Uint4B
```

#### Parsing the x64-bit Executable

```c
0: kd> !process 0 0 notepad.exe
PROCESS ffffd38430e31340
    SessionId: 1  Cid: 1c84    Peb: 7848079000  ParentCid: 098c
    DirBase: 0ccb7002  ObjectTable: ffff90016b7d5e80  HandleCount: 243.
    Image: notepad.exe

0: kd> .process /p /r ffffd38430e31340
Implicit process is now ffffd384`30e31340
.cache forcedecodeuser done
Loading User Symbols
.........................................

************* Symbol Loading Error Summary **************
Module name            Error
SharedUserData         No error - symbol load deferred

You can troubleshoot most symbol related issues by turning on symbol loading diagnostics (!sym noisy) and repeating the command that caused symbols to be loaded.
You should also verify that your symbol search path (.sympath) is correct.
```


```c
00007ffb`562d0000 00007ffb`56623000   combase    (private pdb symbols)  c:\programdata\dbg\sym\combase.pdb\53D725336520305E7FA650F9504238661\combase.pdb
```

DllName: `Combase.dll`

ImageBaseAddress: `0x0007ffb562d0000`

```c

1: kd> dt combase!_IMAGE_DOS_HEADER 00007ffb`562d0000
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
   +0x03c e_lfanew         : 0n264
```

```c
1: kd> ? 0n264
Evaluate expression: 264 = 00000000`00000108

1: kd> dt combase!_IMAGE_NT_HEADERS64 00007ffb`562d0108
   +0x000 Signature        : 0x4550
   +0x004 FileHeader       : _IMAGE_FILE_HEADER
   +0x018 OptionalHeader   : _IMAGE_OPTIONAL_HEADER64
```

```c
1: kd> dt combase!_IMAGE_FILE_HEADER 0007ffb`562d0000 + 0x108 + 0x04
   +0x000 Machine          : 0x8664
   +0x002 NumberOfSections : 8
   +0x004 TimeDateStamp    : 0xe7bf5429
   +0x008 PointerToSymbolTable : 0
   +0x00c NumberOfSymbols  : 0
   +0x010 SizeOfOptionalHeader : 0xf0
   +0x012 Characteristics  : 0x2022

```

```c

1: kd> ? 00007ffb`562d0000 + 0x108 + 0x18
Evaluate expression: 140717459308832 = 00007ffb`562d0120

1: kd> dt combase!_IMAGE_OPTIONAL_HEADER64 00007ffb`562d0120
   +0x000 Magic            : 0x20b
   +0x002 MajorLinkerVersion : 0xe ''
   +0x003 MinorLinkerVersion : 0x14 ''
   +0x004 SizeOfCode       : 0x237e00
   +0x008 SizeOfInitializedData : 0x118200
   +0x00c SizeOfUninitializedData : 0
   +0x010 AddressOfEntryPoint : 0xf3d80
   +0x014 BaseOfCode       : 0x1000
   +0x018 ImageBase        : 0x00007ffb`562d0000
   +0x020 SectionAlignment : 0x1000
   +0x024 FileAlignment    : 0x200
   +0x028 MajorOperatingSystemVersion : 0xa
   +0x02a MinorOperatingSystemVersion : 0
   +0x02c MajorImageVersion : 0xa
   +0x02e MinorImageVersion : 0
   +0x030 MajorSubsystemVersion : 0xa
   +0x032 MinorSubsystemVersion : 0
   +0x034 Win32VersionValue : 0
   +0x038 SizeOfImage      : 0x353000
   +0x03c SizeOfHeaders    : 0x400
   +0x040 CheckSum         : 0x3593d4
   +0x044 Subsystem        : 3
   +0x046 DllCharacteristics : 0x4160
   +0x048 SizeOfStackReserve : 0x40000
   +0x050 SizeOfStackCommit : 0x1000
   +0x058 SizeOfHeapReserve : 0x100000
   +0x060 SizeOfHeapCommit : 0x1000
   +0x068 LoaderFlags      : 0
   +0x06c NumberOfRvaAndSizes : 0x10
   +0x070 DataDirectory    : [16] _IMAGE_DATA_DIRECTORY

```

```c
1: kd> ? 00007ffb`562d0000 + 0x108 + 0x18 + 0x70
Evaluate expression: 140717459308944 = 00007ffb`562d0190

1: kd>  dx -r2 (*(ntdll!_IMAGE_DATA_DIRECTORY (*)[16]) 0x00007ffb562d0190)
(*(ntdll!_IMAGE_DATA_DIRECTORY (*)[16]) 0x00007ffb562d0190)                 [Type: _IMAGE_DATA_DIRECTORY [16]]
    [0]              [Type: _IMAGE_DATA_DIRECTORY] // EXPORT DIRECTORY
        [+0x000] VirtualAddress   : 0x2f5c10 [Type: unsigned long]
        [+0x004] Size             : 0x3cfc [Type: unsigned long]

    [1]              [Type: _IMAGE_DATA_DIRECTORY] // IMPORT DIRECTORY
        [+0x000] VirtualAddress   : 0x2f990c [Type: unsigned long]
        [+0x004] Size             : 0x410 [Type: unsigned long]

    [2]              [Type: _IMAGE_DATA_DIRECTORY] // RESOURCE DIRECTORY
        [+0x000] VirtualAddress   : 0x331000 [Type: unsigned long]
        [+0x004] Size             : 0x13d38 [Type: unsigned long]

    [3]              [Type: _IMAGE_DATA_DIRECTORY] // EXCEPTION DIRECTORY
        [+0x000] VirtualAddress   : 0x305000 [Type: unsigned long]
        [+0x004] Size             : 0x2aad4 [Type: unsigned long]

    [4]              [Type: _IMAGE_DATA_DIRECTORY] // SECURITY DIRECTORY
        [+0x000] VirtualAddress   : 0x34ce00 [Type: unsigned long]
        [+0x004] Size             : 0x9d28 [Type: unsigned long]

    [5]              [Type: _IMAGE_DATA_DIRECTORY] // BASE RELOCATION DIRECTORY
        [+0x000] VirtualAddress   : 0x345000 [Type: unsigned long]
        [+0x004] Size             : 0xd820 [Type: unsigned long]

    [6]              [Type: _IMAGE_DATA_DIRECTORY] // DEBUG DIRECTORY
        [+0x000] VirtualAddress   : 0x298950 [Type: unsigned long]
        [+0x004] Size             : 0x70 [Type: unsigned long]

    [7]              [Type: _IMAGE_DATA_DIRECTORY] // DESCRIPTION DIRECTORY
        [+0x000] VirtualAddress   : 0x0 [Type: unsigned long]
        [+0x004] Size             : 0x0 [Type: unsigned long]

    [8]              [Type: _IMAGE_DATA_DIRECTORY] // SPECIAL DIRECTORY
        [+0x000] VirtualAddress   : 0x0 [Type: unsigned long]
        [+0x004] Size             : 0x0 [Type: unsigned long]

    [9]              [Type: _IMAGE_DATA_DIRECTORY]  // THREAD STORAGE DIRECTORY
        [+0x000] VirtualAddress   : 0x273528 [Type: unsigned long]
        [+0x004] Size             : 0x28 [Type: unsigned long]

    [10]             [Type: _IMAGE_DATA_DIRECTORY] // LOAD CONFIGURATION DIRECTORY
        [+0x000] VirtualAddress   : 0x2543c0 [Type: unsigned long]
        [+0x004] Size             : 0x118 [Type: unsigned long]

    [11]             [Type: _IMAGE_DATA_DIRECTORY] // BOUND IMPORT DIRECTORY
        [+0x000] VirtualAddress   : 0x0 [Type: unsigned long]
        [+0x004] Size             : 0x0 [Type: unsigned long]

    [12]             [Type: _IMAGE_DATA_DIRECTORY] // Import Address Table Directory
        [+0x000] VirtualAddress   : 0x27c2e8 [Type: unsigned long]
        [+0x004] Size             : 0x1278 [Type: unsigned long]

    [13]             [Type: _IMAGE_DATA_DIRECTORY] // Delay Import Directory
        [+0x000] VirtualAddress   : 0x2f3b68 [Type: unsigned long]
        [+0x004] Size             : 0x5a0 [Type: unsigned long]

    [14]             [Type: _IMAGE_DATA_DIRECTORY] //  COR20 Header Directory
        [+0x000] VirtualAddress   : 0x0 [Type: unsigned long]
        [+0x004] Size             : 0x0 [Type: unsigned long]

    [15]             [Type: _IMAGE_DATA_DIRECTORY] // Reserved Directory
        [+0x000] VirtualAddress   : 0x0 [Type: unsigned long]
        [+0x004] Size             : 0x0 [Type: unsigned long]
```

- The last 4 steps can be achieved through using a single command 

```c
    dx -r4 (*(nt!_IMAGE_NT_HEADERS64*)0x0007ffb`562d0108)
```

#### Parsing Export Directory

```c
1: kd> dt combase!_IMAGE_DATA_DIRECTORY 0x00007ffb562d0190
   +0x000 VirtualAddress   : 0x2f5c10
   +0x004 Size             : 0x3cfc

// ImageBaseAddress + _IMAGE_DATA_DIRECTORY[0].VirtualAddress
1: kd> ? 00007ffb`562d0000 + 0x2f5c10
Evaluate expression: 140717462412304 = 00007ffb`565c5c10

1: kd> db 00007ffb`565c5c10
00007ffb`565c5c10  00 00 00 00 29 54 bf e7-00 00 00 00 2a 70 2f 00  ....)T......*p/.
00007ffb`565c5c20  01 00 00 00 4f 02 00 00-c9 01 00 00 38 5c 2f 00  ....O.......8\/.
00007ffb`565c5c30  74 65 2f 00 98 6c 2f 00-20 b6 19 00 80 81 12 00  te/..l/. .......
00007ffb`565c5c40  90 81 12 00 a0 81 12 00-b0 81 12 00 c0 81 12 00  ................

1: kd> dt combase!_IMAGE_EXPORT_DIRECTORY 00007ffb`565c5c10
   +0x000 Characteristics  : 0
   +0x004 TimeDateStamp    : 0xe7bf5429
   +0x008 MajorVersion     : 0
   +0x00a MinorVersion     : 0
   +0x00c Name             : 0x2f702a
   +0x010 Base             : 1
   +0x014 NumberOfFunctions : 0x24f
   +0x018 NumberOfNames    : 0x1c9
   +0x01c AddressOfFunctions : 0x2f5c38
   +0x020 AddressOfNames   : 0x2f6574
   +0x024 AddressOfNameOrdinals : 0x2f6c98

// ImageBaseAddress + _IMAGE_EXPORT_DIRECTORY.AddressOfNames
0: kd> ? 00007ffb`562d0000  + 0x2f6574
Evaluate expression: 140717462414708 = 00007ffb`565c6574

// Dump the DWORDS
0: kd> dds 00007ffb`565c6574 L2
00007ffb`565c6574  002f7134
00007ffb`565c6578  002f7148

// ImageBaseAddress + Entry1
0: kd> ?  00007ffb`562d0000  + 002f7134
Evaluate expression: 140717462417716 = 00007ffb`565c7134

// First Exported Name
0: kd> da 00007ffb`565c7134
00007ffb`565c7134  "CLIPFORMAT_UserFree"

// ImageBaseAddress + Entry2
0: kd> ?  00007ffb`562d0000  +002f7148
Evaluate expression: 140717462417736 = 00007ffb`565c7148

// Second Exported Name 
0: kd> da 00007ffb`565c7148
00007ffb`565c7148  "CLIPFORMAT_UserFree64"

```

```c
[Ordinal/Name Pointer] Table -- Ordinal Base 1
	          Ordinal   Hint Name
	[  88] +base[  89]  0000 CLIPFORMAT_UserFree
	[  90] +base[  91]  0001 CLIPFORMAT_UserFree64
	[  93] +base[  94]  0002 CLIPFORMAT_UserMarshal
	[ 104] +base[ 105]  0003 CLIPFORMAT_UserMarshal64
	[ 105] +base[ 106]  0004 CLIPFORMAT_UserSize
	[ 106] +base[ 107]  0005 CLIPFORMAT_UserSize64
	[ 107] +base[ 108]  0006 CLIPFORMAT_UserUnmarshal
	[ 108] +base[ 109]  0007 CLIPFORMAT_UserUnmarshal64
	[ 112] +base[ 113]  0008 CLSIDFromOle1Class
	[ 113] +base[ 114]  0009 CLSIDFromProgID
	[ 114] +base[ 115]  000a CLSIDFromProgIDEx
    ....
```

#### Parsing Kernel32 with Extension Command

```c
0:000> !dh -f kernel32

File Type: DLL
FILE HEADER VALUES
    8664 machine (X64)
       7 number of sections
A9F358B9 time date stamp Sun May  9 08:34:25 2060

       0 file pointer to symbol table
       0 number of symbols
      F0 size of optional header
    2022 characteristics
            Executable
            App can handle >2gb addresses
            DLL

OPTIONAL HEADER VALUES
     20B magic #
   14.30 linker version
   81000 size of code
   42000 size of initialized data
       0 size of uninitialized data
   125E0 address of entry point
    1000 base of code
         ----- new -----
00007ffe38910000 image base
    1000 section alignment
    1000 file alignment
       3 subsystem (Windows CUI)
   10.00 operating system version
   10.00 image version
   10.00 subsystem version
   C4000 size of image
    1000 size of headers
   D681C checksum
0000000000040000 size of stack reserve
0000000000001000 size of stack commit
0000000000100000 size of heap reserve
0000000000001000 size of heap commit
    4160  DLL characteristics
            High entropy VA supported
            Dynamic base
            NX compatible
            Guard
   9E6F0 [    E8F4] address [size] of Export Directory
   ACFE4 [     7F8] address [size] of Import Directory
   C2000 [     520] address [size] of Resource Directory
   BB000 [    54F0] address [size] of Exception Directory
   C3000 [    4168] address [size] of Security Directory
   C3000 [     390] address [size] of Base Relocation Directory
   8AD60 [      70] address [size] of Debug Directory
       0 [       0] address [size] of Description Directory
       0 [       0] address [size] of Special Directory
       0 [       0] address [size] of Thread Storage Directory
   824C0 [     140] address [size] of Load Configuration Directory
       0 [       0] address [size] of Bound Import Directory
   83C00 [    2AB8] address [size] of Import Address Table Directory
   9E2F8 [      80] address [size] of Delay Import Directory
       0 [       0] address [size] of COR20 Header Directory
       0 [       0] address [size] of Reserved Directory
```

```c
// _IMAGE_DOS_HEADER
0:000> db 00007ffe`38910000
00007ffe`38910000  4d 5a 90 00 03 00 00 00-04 00 00 00 ff ff 00 00  MZ..............
00007ffe`38910010  b8 00 00 00 00 00 00 00-40 00 00 00 00 00 00 00  ........@.......
00007ffe`38910020  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ffe`38910030  00 00 00 00 00 00 00 00-00 00 00 00 e8 00 00 00  ................
00007ffe`38910040  0e 1f ba 0e 00 b4 09 cd-21 b8 01 4c cd 21 54 68  ........!..L.!Th
00007ffe`38910050  69 73 20 70 72 6f 67 72-61 6d 20 63 61 6e 6e 6f  is program canno
00007ffe`38910060  74 20 62 65 20 72 75 6e-20 69 6e 20 44 4f 53 20  t be run in DOS 
00007ffe`38910070  6d 6f 64 65 2e 0d 0d 0a-24 00 00 00 00 00 00 00  mode....$.......

// _IMAGE_NT_HEADERS64
0:000> db 00007ffe`38910000 + e8
00007ffe`389100e8  50 45 00 00 64 86 07 00-b9 58 f3 a9 00 00 00 00  PE..d....X......
00007ffe`389100f8  00 00 00 00 f0 00 22 20-0b 02 0e 1e 00 10 08 00  ....... ........
00007ffe`38910108  00 20 04 00 00 00 00 00-e0 25 01 00 00 10 00 00  . .......%......
00007ffe`38910118  00 00 91 38 fe 7f 00 00-00 10 00 00 00 10 00 00  ...8............
00007ffe`38910128  0a 00 00 00 0a 00 00 00-0a 00 00 00 00 00 00 00  ................
00007ffe`38910138  00 40 0c 00 00 10 00 00-1c 68 0d 00 03 00 60 41  .@.......h....`A
00007ffe`38910148  00 00 04 00 00 00 00 00-00 10 00 00 00 00 00 00  ................
00007ffe`38910158  00 00 10 00 00 00 00 00-00 10 00 00 00 00 00 00  ................

// _IMAGE_OPTIONAL_HEADER64->_IMAGE_DATA_DIRECTORY[0].VirtualAddress : EXPORT DIRECTORY
0:000> db 00007ffe`38910000 + e8+0x18+0x70
00007ffe`38910170  f0 e6 09 00 f4 e8 00 00-e4 cf 0a 00 f8 07 00 00  ................
00007ffe`38910180  00 20 0c 00 20 05 00 00-00 b0 0b 00 f0 54 00 00  . .. ........T..
00007ffe`38910190  00 30 0c 00 68 41 00 00-00 30 0c 00 90 03 00 00  .0..hA...0......
00007ffe`389101a0  60 ad 08 00 70 00 00 00-00 00 00 00 00 00 00 00  `...p...........
00007ffe`389101b0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00007ffe`389101c0  c0 24 08 00 40 01 00 00-00 00 00 00 00 00 00 00  .$..@...........
00007ffe`389101d0  00 3c 08 00 b8 2a 00 00-f8 e2 09 00 80 00 00 00  .<...*..........
00007ffe`389101e0  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................

// _IMAGE_EXPORT_DIRECTORY
0:000> db 00007ffe`38910000 + 9e6f0
00007ffe`389ae6f0  00 00 00 00 b9 58 f3 a9-00 00 00 00 5e 28 0a 00  .....X......^(..
00007ffe`389ae700  01 00 00 00 87 06 00 00-87 06 00 00 18 e7 09 00  ................
00007ffe`389ae710  34 01 0a 00 50 1b 0a 00-83 28 0a 00 b9 28 0a 00  4...P....(...(..
00007ffe`389ae720  90 8a 01 00 50 47 01 00-80 12 02 00 60 aa 05 00  ....PG......`...
00007ffe`389ae730  e0 45 00 00 90 0f 02 00-a0 0f 02 00 64 29 0a 00  .E..........d)..
00007ffe`389ae740  f0 cc 03 00 80 ab 05 00-e0 ab 05 00 50 ac 03 00  ............P...
00007ffe`389ae750  60 6f 01 00 70 ac 03 00-90 f5 01 00 90 ac 03 00  `o..p...........
00007ffe`389ae760  c0 84 03 00 9d 2a 0a 00-dd 2a 0a 00 40 ad 04 00  .....*...*..@...

0:000> db 00007ffe`38910000 + 9e718
00007ffe`389ae718  83 28 0a 00 b9 28 0a 00-90 8a 01 00 50 47 01 00  .(...(......PG..
00007ffe`389ae728  80 12 02 00 60 aa 05 00-e0 45 00 00 90 0f 02 00  ....`....E......
00007ffe`389ae738  a0 0f 02 00 64 29 0a 00-f0 cc 03 00 80 ab 05 00  ....d)..........
00007ffe`389ae748  e0 ab 05 00 50 ac 03 00-60 6f 01 00 70 ac 03 00  ....P...`o..p...
00007ffe`389ae758  90 f5 01 00 90 ac 03 00-c0 84 03 00 9d 2a 0a 00  .............*..
00007ffe`389ae768  dd 2a 0a 00 40 ad 04 00-e0 0b 02 00 d0 ac 03 00  .*..@...........
00007ffe`389ae778  b0 ac 03 00 70 2b 0a 00-ae 2b 0a 00 f6 2b 0a 00  ....p+...+...+..
00007ffe`389ae788  49 2c 0a 00 a1 2c 0a 00-f5 2c 0a 00 49 2d 0a 00  I,...,...,..I-..

0:000> db 00007ffe`38910000 + 0a2883
00007ffe`389b2883  4e 54 44 4c 4c 2e 52 74-6c 41 63 71 75 69 72 65  NTDLL.RtlAcquire
00007ffe`389b2893  53 52 57 4c 6f 63 6b 45-78 63 6c 75 73 69 76 65  SRWLockExclusive
00007ffe`389b28a3  00 41 63 71 75 69 72 65-53 52 57 4c 6f 63 6b 53  .AcquireSRWLockS
00007ffe`389b28b3  68 61 72 65 64 00 4e 54-44 4c 4c 2e 52 74 6c 41  hared.NTDLL.RtlA
00007ffe`389b28c3  63 71 75 69 72 65 53 52-57 4c 6f 63 6b 53 68 61  cquireSRWLockSha
00007ffe`389b28d3  72 65 64 00 41 63 74 69-76 61 74 65 41 63 74 43  red.ActivateActC
00007ffe`389b28e3  74 78 00 41 63 74 69 76-61 74 65 41 63 74 43 74  tx.ActivateActCt
00007ffe`389b28f3  78 57 6f 72 6b 65 72 00-41 63 74 69 76 61 74 65  xWorker.Activate
```