+++
title = "Prefetch Files"
date = "2025-06-12T09:54:09+05:30"
author = "5h4rrk"
tags = ["Windows Prefetch", "Windows Forensics", "Digital Forensics", "Prefetch File Analysis", "Prefetch Parser", "Windows Performance"]
description = "An analysis of Windows Prefetch files, their structure, forensic significance, and decompression process."
draft=false
+++

# Prefetch Files 

Prefetch files are created by Windows to speed up the loading process by caching the neccessary data. When we fire up a process, it will cache the details like files accessed and stores it in small file i.e. prefetch files under `Prefetch` subfolder in `Windows`. When the application is opened for the next time, it will load the files accessed. Windows Prefetch files are located in `C:\Windows\Prefetch\*.pf`. From forensic perspective, it provides various valuable information like

- Executable Name 
- Full Executable Path
- Execution Count
- Last Run Time
- TimeStamp (when executed i.e. Last 8 timestamps)
- MACB Time 
- Executable Path Hash
- Volume Information
- Files accessed
- Directories accessed

![Prefetch](/posts/prefetch/image-1.png)

```python
prefetch_settings = {
    0: "Prefetch-Disabled",
    1: "Application-Prefetch-Enabled",
    2: "Boot-Prefetch-Enabled",
    3: "Application-Boot-Prefetch-Enabled"
}
```

- Maximum Number of Prefetch files
    - **WindowsXP-Windows7**: 128 files
    - **Windows8-Windows11**: 1024 files 

## Prefetch File Structure

Windows Prefetch files are compressed and stored in `.pf` format. To obtain the information, we need to decompress it first.

### Compressed Format

- Compressed Header ( 0x4 bytes)
- Uncompressed Size ( 0x4 bytes)
- CRC32             ( 0x4 bytes) // Optional
- Compressed buffer 

Compressed Header: `0x4D414D04` (`MAM\x04`)

CRC32 is an optional either it will present or absent.

#### Overview of Compressed Prefetch file

<div style="display: flex; gap: 10px; align-items: center;">
  <div>
    <p><strong>Without CRC32</strong></p>
    <a href="/posts/prefetch/image.png" target="_blank" rel="noopener">
      <img src="/posts/prefetch/image.png" alt="PrefetchFileStructure1" width="400"/>
    </a>
  </div>
  <div>
    <p><strong>With CRC32</strong></p>
    <a href="/posts/prefetch/image-2.png" target="_blank" rel="noopener">
      <img src="/posts/prefetch/image-2.png" alt="PrefetchFileStructure2" width="400"/>
    </a>
  </div>
</div>


#### Decompressing It

`PrefetchDecompress.cpp`

```c
// PrefetchDecompress.cpp : This file contains the 'main' function. Program
// execution begins and ends there.
#include <Windows.h>
#include <winternl.h>
#include <stdio.h>
import std;

#define COMPRESSION_FORMAT_NONE (0x0000)        // winnt
#define COMPRESSION_FORMAT_DEFAULT (0x0001)     // winnt
#define COMPRESSION_FORMAT_LZNT1 (0x0002)       // winnt
#define COMPRESSION_FORMAT_XPRESS (0x0003)      // winint
#define COMPRESSION_FORMAT_XPRESS_HUFF (0x0004) // winint
#define COMPRESSION_ENGINE_STANDARD (0x0000)    // winnt
#define COMPRESSION_ENGINE_MAXIMUM (0x0100)     // winnt
#define COMPRESSION_ENGINE_HIBER (0x0200)       // winnt

typedef ULONG32 u32;
typedef ULONG64 u64;
typedef UCHAR u8;

typedef struct {
  u32 signature;
  u32 size;
  BYTE* compressedPayload;
  BYTE* payloadPtr;
} COMPRESSED_PREFETCH;

typedef NTSTATUS(WINAPI *RtlGetCompressionWorkSpaceSize_t)(
    USHORT CompressionFormatAndEngine, PULONG CompressBufferWorkSpaceSize,
    PULONG CompressFragmentWorkSpaceSize);

typedef NTSTATUS(WINAPI *RtlDecompressBufferEx_t)(
    USHORT CompressionFormat, PUCHAR UncompressedBuffer,
    ULONG UncompressedBufferSize, PUCHAR CompressedBuffer,
    ULONG CompressedBufferSize,
    PULONG FinalCompressedSize, PVOID WorkSpace);

void CleanUp(HMODULE hNtdll, HANDLE hFile) {
  if (hFile && hFile != INVALID_HANDLE_VALUE)
    CloseHandle(hFile);
}

void InitPrefetchBuff(COMPRESSED_PREFETCH *prefetch, SIZE_T req_size) {

  prefetch->compressedPayload = (BYTE *)VirtualAlloc(
      nullptr, 0x00100000, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

  prefetch->payloadPtr = (BYTE*) VirtualAlloc(
      nullptr, 0x00100000, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

  if (prefetch->compressedPayload == NULL) {
    std::println("[-] Failed to allocate buffer");
  }

  if (prefetch->payloadPtr == NULL) {
    std::println("[-] Failed to allocate buffer");
  }
}

void CleanPrefetch(COMPRESSED_PREFETCH* prefetch, SIZE_T size) {
    prefetch->size = 0x0;
    prefetch->signature = 0x0;
    VirtualFree(prefetch->payloadPtr, 0x00100000, MEM_DECOMMIT | MEM_FREE);
    VirtualFree(prefetch->compressedPayload, 0x00100000, MEM_DECOMMIT | MEM_FREE);
}

int main() {
  std::println("[+] Windows Prefetch ");
  HANDLE hFile = CreateFileA(
      R"(E:\files\Wireshark.pf)",
      GENERIC_READ, FILE_SHARE_READ, nullptr, OPEN_EXISTING,
      FILE_ATTRIBUTE_NORMAL, nullptr);

  if (hFile == INVALID_HANDLE_VALUE) {
    std::println("[?] Failed To Open File with ErrorCode {}", GetLastError());
    return GetLastError();
  }
  ULONG CompressBufferWorkSpaceSize = 0x0;
  ULONG CompressFragmentWorkSpaceSize = 0x0;

  // Load the library

HMODULE hNtdll = GetModuleHandleA("ntdll.dll");

  if (hNtdll == NULL) {
    std::println("Unable to Obtain handle");
    CloseHandle(&hFile);
    return GetLastError();
  }
  auto RtlGetCompressionWorkSpaceSizeFuncPtr = (RtlGetCompressionWorkSpaceSize_t) GetProcAddress(hNtdll, "RtlGetCompressionWorkSpaceSize");
  auto RtlDecompressBufferExPtr = (RtlDecompressBufferEx_t) GetProcAddress(hNtdll, "RtlDecompressBufferEx");

  if ((u64)RtlGetCompressionWorkSpaceSizeFuncPtr == NULL) {
    std::println("Unable to resolve RtlGetCompressionWorkSpaceSize");
    CleanUp(hNtdll, hFile);
    return GetLastError();
  }
  if ((u64)RtlDecompressBufferExPtr == NULL) {
    std::println("Unable to resolve RtlDecompressBufferEx");
    CleanUp(hNtdll, hFile);
    return GetLastError();
  }
  std::println("[OFFSET] RtlGetCompressionWorkSpaceSize @ {:#x} ",(u64)RtlGetCompressionWorkSpaceSizeFuncPtr);
  std::println("[OFFSET] RtlDecompressBufferExPtr @ {:#x} ",(u64)RtlDecompressBufferExPtr);

  NTSTATUS status = RtlGetCompressionWorkSpaceSizeFuncPtr( COMPRESSION_FORMAT_XPRESS_HUFF, &CompressBufferWorkSpaceSize, &CompressFragmentWorkSpaceSize);
  if (status == 0x0) {
    std::println("[+] Success: \n\tCompressBufferWorkSpaceSize = {:#x}\n\tCompressFragmentWorkSpaceSize = {:#x}",(u64)CompressBufferWorkSpaceSize, (u64)CompressFragmentWorkSpaceSize);
  }

  COMPRESSED_PREFETCH prefetch;
  ULONG bytesRead;
  u32 sig = 0x0;
  u32 size = 0x0;

  u64 CompressedSize = (u64)GetFileSize(hFile, NULL) - 0x8; // TotalFileSize - HeaderSize
  ReadFile(hFile, &sig, sizeof(UINT32), &bytesRead, NULL);
  ReadFile(hFile, &size, sizeof(UINT32), &bytesRead, NULL);

  prefetch.size = size;
  prefetch.signature = sig;

  InitPrefetchBuff(&prefetch, CompressBufferWorkSpaceSize);
  BOOL flag = ReadFile(hFile, prefetch.compressedPayload, CompressBufferWorkSpaceSize - 8, &bytesRead, NULL);

  if (flag == 0x0) {
    std::println("File Read Failed Error Code {:#x}", GetLastError());
  }

  std::println("Signature {:#x}", prefetch.signature);
  std::println("Decompressed Size {:#x}", prefetch.size);
  std::println("Compressed Size {:#x}", CompressedSize);
  
  
  // Decompressing
  ULONG FinalUncompressedSize = 0x0;
  PVOID workspace = VirtualAlloc(nullptr, CompressBufferWorkSpaceSize, MEM_COMMIT | MEM_RESERVE, PAGE_READWRITE);

  if (!workspace) {
    std::println("[-] Failed to allocate workspace");
    CleanUp(hNtdll, hFile);
    return GetLastError();
  }
 
  NTSTATUS decompress_flag = RtlDecompressBufferExPtr(
      COMPRESSION_FORMAT_XPRESS_HUFF, prefetch.payloadPtr, prefetch.size, prefetch.compressedPayload,
      CompressedSize, &FinalUncompressedSize,
      workspace);

  if (decompress_flag == 0x0) {
    std::println("Decompressed Successfully !");
  }
  HANDLE hDecomFile = CreateFileA(
      R"(E:\files\Wireshark_decompressed.pf)",
      GENERIC_WRITE, 0x0, nullptr, CREATE_ALWAYS,
      FILE_ATTRIBUTE_NORMAL, nullptr);

  if (hDecomFile != INVALID_HANDLE_VALUE) {
    ULONG bytesWritten = 0x0;
    auto status = WriteFile(hDecomFile, prefetch.payloadPtr, FinalUncompressedSize, &bytesWritten, 0x0);
    if (status != 0x0) {
      std::println("File Written Successfully");
    }
  }
  VirtualFree(workspace, 0, MEM_RELEASE);
  CleanPrefetch(&prefetch, CompressBufferWorkSpaceSize);
  std::println("Done");
}
```

![DecompressedFile](/posts/prefetch/image-3.png)
