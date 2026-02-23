# Hướng Dẫn Đọc File PAK Trên Server x64

Tài liệu này tập trung vào các vấn đề **tương thích x64** khi chuyển đổi server từ Win32 sang x64 để đọc file PAK. Giả định bạn đã hiểu cơ bản hệ thống PAK (KPakFile, KPakList, XPackFile) và luồng load bản đồ.

---

## Mục Lục

1. [Tổng Quan Khác Biệt x64 vs Win32](#1-tổng-quan-khác-biệt-x64-vs-win32)
2. [Hệ Thống Preprocessor & Property Sheets](#2-hệ-thống-preprocessor--property-sheets)
3. [Các Thay Đổi Mã Nguồn Cần Thiết](#3-các-thay-đổi-mã-nguồn-cần-thiết)
4. [Vấn Đề Inline Assembly (x86 ASM)](#4-vấn-đề-inline-assembly-x86-asm)
5. [SubWorld: Cấp Phát Heap Thay Vì Stack](#5-subworld-cấp-phát-heap-thay-vì-stack)
6. [DLL Loading: Tên File x64](#6-dll-loading-tên-file-x64)
7. [Thư Viện BDB x64 & Static CRT](#7-thư-viện-bdb-x64--static-crt)
8. [Build & Lib Staging](#8-build--lib-staging)


---

## 1. Tổng Quan Khác Biệt x64 vs Win32

Khi biên dịch server cho nền tảng x64, có **5 khác biệt chính** ảnh hưởng đến khả năng đọc PAK:

| Vấn Đề | Win32 Server | x64 Server |
|---------|-------------|------------|
| Macro `_SERVER` | Có (Core), Không (Engine) | Có (cả Core **lẫn** Engine) |
| Macro `_WIN64` | Không | Có (tự động bởi `SwordOnline.x64.props`) |
| Inline ASM (`__asm`) | Hỗ trợ | **Không hỗ trợ** — MSVC x64 không cho phép inline ASM |
| SubWorld allocation | Mảng tĩnh trên stack (256 slot) | Con trỏ, cấp phát heap (1024 slot) |
| DLL tên file | `Engine.dll`, `CoreServer.dll` | `Engine64.dll`, `CoreServer64.dll` |

**Điểm quan trọng nhất:** Trên Win32, Engine.vcxproj **không** define `_SERVER` cho config "Server Release\|Win32", nhưng trên x64, Engine.vcxproj **có** define `_SERVER` cho config "Release\|x64". Điều này ảnh hưởng trực tiếp đến PAK vì các guard `#ifndef _SERVER` trong KPakFile sẽ hành xử khác nhau giữa 2 nền tảng.

---

## 2. Hệ Thống Preprocessor & Property Sheets

### Chuỗi props được import

Mỗi project x64 import 2 property sheet theo thứ tự:

```
SwordOnline.Common.props    ← định nghĩa chung cho tất cả nền tảng
    │
    └── SwordOnline.x64.props   ← ghi đè/bổ sung cho x64
```

### SwordOnline.Common.props (áp dụng mọi nền tảng)

```xml
<!-- Preprocessor chung -->
WIN32; WINVER=0x0601; _WIN32_WINNT=0x0601; WIN32_LEAN_AND_MEAN; NOMINMAX;
_CRT_SECURE_NO_WARNINGS; _CRT_NONSTDC_NO_WARNINGS; _WINSOCK_DEPRECATED_NO_WARNINGS

<!-- Lib staging tự động theo config -->
Release + Win32 → Build\Lib\release\
Release + x64   → Build\Lib\release-x64\
Debug + Win32   → Build\Lib\debug\
Debug + x64     → Build\Lib\debug-x64\

<!-- DirectX SDK -->
IncludePath: dx9csdk\Include (ưu tiên) + $(DXSDK_DIR)Include (fallback)
LibraryPath: dx9csdk\Lib\x86  (Win32 mặc định)

<!-- CRT tĩnh -->
Release → /MT (MultiThreaded)
Debug   → /MTd (MultiThreadedDebug)
```

### SwordOnline.x64.props (chỉ áp dụng x64)

```xml
<!-- Thư mục intermediate riêng -->
IntDir = $(Platform)\$(Configuration)\     → x64\Release\ hoặc x64\Debug\

<!-- Ghi đè đường dẫn DirectX SDK cho x64 -->
LibraryPath: dx9csdk\Lib\x64 (ưu tiên) + $(DXSDK_DIR)Lib\x64

<!-- MACRO QUAN TRỌNG NHẤT -->
PreprocessorDefinitions: _WIN64; %(PreprocessorDefinitions)

<!-- Warning Level 4 — bắt lỗi truncation pointer/size -->
WarningLevel: Level4
DisableSpecificWarnings: 4996; 4018   (giữ 4244/4267 để phát hiện truncation)

<!-- Linker: bật ASLR, bỏ SAFESEH (x86-only) -->
RandomizedBaseAddress: true
```

### Bảng tổng hợp Preprocessor theo từng config

| Config | Project | Preprocessor cuối cùng |
|--------|---------|----------------------|
| Server Release \| Win32 | **Core.vcxproj** | `NDEBUG; _SERVER; WIN32; _WINDOWS; _USRDLL; CORE_EXPORTS` + Common.props |
| Server Release \| Win32 | **Engine.vcxproj** | `NDEBUG; WIN32; _WINDOWS; _USRDLL; ENGINE_EXPORTS; WINVER=0x0500` + Common.props (KHÔNG có `_SERVER`) |
| Release \| x64 | **Core.vcxproj** | `NDEBUG; _SERVER; WIN32; _WINDOWS; _USRDLL; CORE_EXPORTS` + Common.props + **x64.props (`_WIN64`)** |
| Release \| x64 | **Engine.vcxproj** | `NDEBUG; _SERVER; WIN32; _WINDOWS; _USRDLL; ENGINE_EXPORTS; WINVER=0x0500; _USELUALIB` + Common.props + **x64.props (`_WIN64`)** |

> **Chú ý:** Engine x64 có `_SERVER` nhưng Engine Win32 (Server Release) **không** có `_SERVER`. Đây là điểm khác biệt quan trọng ảnh hưởng đến ABI của KPakFile.

---

## 3. Các Thay Đổi Mã Nguồn Cần Thiết

### 3.1. Gỡ Guard `#ifndef _SERVER` Trong KPakFile

**Vấn đề ABI trên Win32:** Engine Win32 không define `_SERVER` → `m_PackRef` được bao gồm trong `KPakFile`. Core Win32 define `_SERVER` → `m_PackRef` bị loại khỏi `KPakFile`. **Kết quả:** `sizeof(KPakFile)` khác nhau giữa 2 DLL → crash khi truyền đối tượng qua ranh giới DLL.

**Vấn đề trên x64:** Cả Engine x64 và Core x64 đều define `_SERVER`, nên cả hai đều loại `m_PackRef` → ABI nhất quán nhưng **PAK không hoạt động** (thiếu `m_PackRef`, thiếu logic tìm file trong PAK).

**Giải pháp:** Gỡ **tất cả** guard `#ifndef _SERVER` trong KPakFile.h và KPakFile.cpp, **ngoại trừ** các hàm render SPR/JPEG (phụ thuộc DirectDraw).

#### KPakFile.h — 2 guard cần gỡ

```cpp
// GỠ guard này — cho phép include XPackFile.h vô điều kiện:
#include "XPackFile.h"          // ← trước đây bị bao bởi #ifndef _SERVER

// GỠ guard này — m_PackRef luôn tồn tại trong struct:
private:
    KFile               m_File;
    XPackElemFileRef    m_PackRef;  // ← trước đây bị bao bởi #ifndef _SERVER
```

> **GIỮ NGUYÊN** guard `#ifndef _SERVER` cho `KSGImageContent`, `SprGetHeader()`, `SprGetFrame()`, `get_jpg_image()`, `release_image()` (dòng 39-55) — đây là hàm render chỉ dành cho client.

#### KPakFile.cpp — 9 guard cần gỡ

| # | Hàm/Vùng Code | Mô Tả |
|---|---------------|-------|
| 1 | `#include "KPakList.h"` | Include header quản lý PAK |
| 2 | Constructor `KPakFile()` | Khởi tạo `m_PackRef.nPackIndex = -1; m_PackRef.uId = 0;` |
| 3 | `IsFileInPak()` | Kiểm tra file có nằm trong PAK không |
| 4 | `Open()` | Logic fallback: tìm file trên đĩa → PAK (mode 0) hoặc PAK → đĩa (mode 1) |
| 5 | `Read()` | Đọc từ PAK qua `g_pPakList->ElemFileRead()` |
| 6 | `Seek()` | Seek trong PAK (thao tác `m_PackRef.nOffset`) |
| 7 | `Tell()` | Trả về vị trí đọc hiện tại từ PAK |
| 8 | `Size()` | Trả về kích thước file từ PAK |
| 9 | `Close()` | Reset `m_PackRef` khi đóng file |

> **GIỮ NGUYÊN** guard `#ifndef _SERVER` cho tất cả hàm SPR/JPEG (dòng 31-229).

### 3.2. Kích Hoạt PAK Init Trong KSubWorld::LoadMap

File `Core\Src\KSubWorld.cpp`, dòng 641 — bỏ comment:

```cpp
// TRƯỚC:
//g_PakList.Open("\\package.ini");

// SAU:
g_PakList.Open("\\package.ini");
```

Biến `g_PakList` khai báo tại dòng 46, chỉ tồn tại trên server (`#ifdef _SERVER`):

```cpp
#ifdef _SERVER
KGlobalMissionArray g_GlobalMissionArray;
KPakList g_PakList;    // constructor tự gán g_pPakList = this
#endif
```

### 3.3. Sửa Macro MAPLIST_SETTING_FILE

File `Core\Src\CoreUseNameDef.h`, dòng 139:

```cpp
// TRƯỚC:
#define MAPLIST_SETTING_FILE "\\settings\\WorldSet.ini"

// SAU:
#define MAPLIST_SETTING_FILE "\\settings\\MapList.ini"
```

---

## 4. Vấn Đề Inline Assembly (x86 ASM)

MSVC **không hỗ trợ inline `__asm`** khi biên dịch x64. Engine có nhiều hàm dùng inline ASM, tất cả đã có nhánh C/C++ thay thế qua guard `#ifndef _WIN64`:

| File | Hàm | Vai Trò |
|------|-----|---------|
| `Engine\Src\KMemBase.cpp` | `g_MemCopy`, `g_MemCopyMmx`, `g_MemComp`, `g_MemFill`, `g_MemZero`, `g_MemXore` | Thao tác bộ nhớ nhanh |
| `Engine\Src\KStrBase.cpp` | `g_StrLen`, `g_StrEnd`, `g_StrCpy`, `g_StrCpyLen`, `g_StrUpper`, `g_StrLower` | Thao tác chuỗi nhanh |
| `Engine\Src\KColors.cpp` | `g_Red`, `g_Green`, `g_Blue`, `g_RGB555`, `g_RGB565`, `g_555To565`, `g_565To555` | Xử lý màu sắc |
| `Engine\Src\KPalette.cpp` | `g_Pal24ToPal16`, `g_Pal16ToPal24` | Chuyển đổi palette |
| `Engine\Src\KDrawDotFont.cpp` | `g_DrawDotFont` | Vẽ font bitmap |
| `Engine\Src\KSprite.cpp` | SPR rendering routines | Vẽ sprite |
| `Engine\Src\KImageRes.cpp` | Image resource routines | Xử lý ảnh |
| `JpgLib\Src\KJpegLib.h` | `READ_WORD` macro | Đọc JPEG |
| `MultiServer\Common\KSG_EncodeDecode.cpp` | `KSG_DecodeEncode_ASM` | Mã hóa/giải mã gói tin |

**Pattern chung:**

```cpp
#ifndef _WIN64
    __asm { /* x86 inline assembly — nhanh */ }
#else
    // C/C++ fallback — tương thích x64
#endif
```

**Liên quan đến PAK:** Hàm `g_MemCopy`, `g_MemZero`, `g_StrCpy` được dùng **gián tiếp** trong quá trình đọc PAK (copy buffer, xử lý chuỗi tên file). Trên x64, chúng tự động dùng nhánh C fallback — **không cần sửa gì thêm**, chỉ cần đảm bảo `_WIN64` được define (đã có sẵn trong `SwordOnline.x64.props`).

**Trường hợp đặc biệt — `KSG_EncodeDecode`:** Mã hóa/giải mã gói tin mạng. Trên x64, tự động gọi hàm C `KSG_DecodeEncode()` thay vì phiên bản ASM. Không ảnh hưởng PAK nhưng ảnh hưởng hiệu năng mạng (nhẹ).

---

## 5. SubWorld: Cấp Phát Heap Thay Vì Stack

### Vấn đề

`KSubWorld` là struct lớn. Trên Win32, mảng `SubWorld[MAX_SUBWORLD]` được cấp phát tĩnh (stack/BSS). Trên x64, `MAX_SUBWORLD` tăng lên 1024 — cấp phát tĩnh sẽ tràn bộ nhớ.

### Giải pháp đã có trong code

File `Core\Src\KSubWorld.h`:

```cpp
#ifdef _SERVER
  #ifdef _WIN64
    #define MAX_SUBWORLD  1024    // x64: 1024 bản đồ
  #else
    #define MAX_SUBWORLD  256     // Win32: 256 bản đồ
  #endif
#else
  #define MAX_SUBWORLD  1         // Client: 1 bản đồ
#endif
```

File `Core\Src\KSubWorld.cpp`, dòng 25-39:

```cpp
#if defined(_WIN64) && defined(_SERVER)
    KSubWorld* SubWorld = nullptr;                    // con trỏ, chưa cấp phát
    void g_AllocSubWorld() {
        if (!SubWorld)
            SubWorld = new KSubWorld[MAX_SUBWORLD];   // cấp phát heap khi init
    }
    void g_FreeSubWorld() {
        delete[] SubWorld;
        SubWorld = nullptr;                           // giải phóng khi shutdown
    }
#else
    KSubWorld SubWorld[MAX_SUBWORLD];                 // Win32: mảng tĩnh
#endif
```

File `Core\Src\KCore.cpp` — gọi alloc/free:

```cpp
// Khởi tạo (dòng 139-146):
#if defined(_WIN64) && defined(_SERVER)
extern void g_AllocSubWorld();
#endif
CORE_API void g_InitCore()
{
#if defined(_WIN64) && defined(_SERVER)
    g_AllocSubWorld();            // cấp phát trước khi load map
#endif
    // ... phần init khác ...
}

// Giải phóng (dòng 474-478):
void g_ReleaseCore()
{
#if defined(_WIN64) && defined(_SERVER)
    extern void g_FreeSubWorld();
    g_FreeSubWorld();
#endif
    // ... phần cleanup khác ...
}
```

File `Core\Src\KPathfinder.cpp` — tham chiếu SubWorld:

```cpp
#if defined(_WIN64) && defined(_SERVER)
extern KSubWorld* SubWorld;     // con trỏ (x64)
#else
extern KSubWorld SubWorld[];    // mảng tĩnh (Win32)
#endif
```

**Liên quan PAK:** Mỗi phần tử `SubWorld[i]` gọi `LoadMap(nId)` → trong đó gọi `g_PakList.Open()`. Nếu `g_AllocSubWorld()` không được gọi trước, `SubWorld` là `nullptr` → crash ngay lập tức. **Đảm bảo `g_InitCore()` được gọi trước bất kỳ `LoadMap()` nào.**

---

## 6. DLL Loading: Tên File x64

Tất cả module server dùng `#ifdef _WIN64` để load đúng phiên bản DLL x64:

| Module | Code | File |
|--------|------|------|
| GameServer | `CLibrary g_theHeavenLibrary("heaven64.dll")` | `GameServer\KSOServer.cpp:72` |
| GameServer | `CLibrary g_theRainbowLibrary("rainbow64.dll")` | `GameServer\KSOServer.cpp:73` |
| Goddess | `CLibrary g_theHeavenLibrary("heaven64.dll")` | `Goddess\Goddess.cpp:70` |
| Goddess | `LoadLibrary("FilterText64.dll")` | `Goddess\FilterTextLib.cpp:29` |
| S3Relay | `LoadLibrary("heaven64.dll")` | `S3Relay\HeavenLib.cpp:29` |
| S3Relay | `LoadLibrary("rainbow64.dll")` | `S3Relay\RainbowLib.cpp:29` |
| Bishop | `CLibrary m_theRainbowLib("rainbow64.dll")` | `Bishop\SmartClient.cpp:21` |
| Bishop | `CLibrary m_theHeavenLib("heaven64.dll")` | `Bishop\Network.cpp:17` |
| Bishop | `CLibrary m_theRainbowLib("rainbow64.dll")` | `Bishop\Network.cpp:19` |
| S3RelayServer | `LoadLibrary("heaven64.dll")` | `S3RELAYSERVER\main.cpp:149` |
| S3AccServer | `LoadLibrary("heaven64.dll")` | `S3AccServer\main.cpp:152` |

**Pattern:**

```cpp
#ifdef _WIN64
    HMODULE hModule = ::LoadLibrary("heaven64.dll");
#else
    HMODULE hModule = ::LoadLibrary("heaven.dll");
#endif
```

**Liên quan PAK:** PAK không load DLL riêng — PAK nằm trong Engine64.dll. Nhưng nếu thiếu bất kỳ DLL x64 nào (đặc biệt `rainbow64.dll`), server sẽ crash trước khi kịp load PAK.

---

## 7. Thư Viện BDB x64 & Static CRT

### Berkeley DB x64

- File lib: `Build/Lib/release-x64/libdb62s_x64.lib`
- Phải được build với `/MT` (static CRT) bằng toolset **v141** (VS2017)
- Nếu build BDB với `/MD` (dynamic CRT) → lỗi linker `__declspec(dllimport)` cho các hàm CRT
- Nếu thiếu `/D_LIB` → BDB headers generate `__declspec(dllimport)` thay vì static linkage

### Static CRT (/MT) — Tất Cả Project

Tất cả project trong solution dùng `/MT` (Release) hoặc `/MTd` (Debug):

```xml
<!-- SwordOnline.Common.props -->
<RuntimeLibrary>MultiThreaded</RuntimeLibrary>        <!-- Release: /MT -->
<RuntimeLibrary>MultiThreadedDebug</RuntimeLibrary>    <!-- Debug: /MTd -->
```

**Tác dụng:**
- Không cần cài VC++ Redistributable trên máy server đích
- Tất cả hàm CRT được link tĩnh vào binary
- **BẮT BUỘC** tất cả thư viện bên ngoài (.lib) cũng phải build với `/MT` — nếu không sẽ bị conflict CRT (`LNK4098` hoặc crash runtime)

### Thư viện bị ignore

```xml
<!-- Common.props Release linker -->
<IgnoreSpecificDefaultLibraries>LIBC.lib;msvcrt.lib;msvcprt.lib</IgnoreSpecificDefaultLibraries>
```

---

## 8. Build & Lib Staging

### Cấu hình build trong solution

| Solution Config | Platform | Output chính |
|----------------|----------|-------------|
| Client Release | Win32 | Game.exe + các DLL client |
| Server Release | Win32 | GameServer.exe + CoreServer.dll + Engine.dll |
| **Server Release** | **x64** | **GameServer64.exe + CoreServer64.dll + Engine64.dll** |
| Debug | Win32 | Phiên bản debug client |
| Server Debug | Win32 | Phiên bản debug server |

### Đường dẫn Lib Staging

| Config | LibStagingDir |
|--------|--------------|
| Release + Win32 | `Build/Lib/release/` |
| **Release + x64** | **`Build/Lib/release-x64/`** |
| Debug + Win32 | `Build/Lib/debug/` |
| Debug + x64 | `Build/Lib/debug-x64/` |

### Tên Output x64

| Project | TargetName x64 | Output |
|---------|---------------|--------|
| Engine | `Engine64` | Engine64.dll, Engine64.lib |
| Core | `CoreServer64` | CoreServer64.dll, CoreServer64.lib |
| GameServer | `GameServer64` | GameServer64.exe |
| Goddess | `Goddess64` | Goddess64.exe |
| Bishop | `Bishop64` | Bishop64.exe |
| S3Relay | `S3Relay64` | S3Relay64.exe |
| LuaLibDll | `LuaLibDll64` | LuaLibDll64.dll |
| FilterText | `FilterText64` | FilterText64.dll |
| Heaven | `heaven64` | heaven64.dll |
| Rainbow | `rainbow64` | rainbow64.dll |

### Linker Dependencies x64 (Engine)

Engine x64 link với phiên bản `64` của tất cả dependency:

```
JpgLib64.lib; lualibdll64.lib; KMp3Lib64.lib; legacy_stdio_definitions.lib
+ ddraw.lib; dsound.lib; dxguid.lib; winmm.lib; wsock32.lib; dinput8.lib
```

### Giới Hạn Người Chơi

Trên x64, Goddess gỡ bỏ giới hạn 100 người chơi của Win32:

```cpp
// Goddess.cpp:1167
#ifndef _WIN64
    // x86: 3 buffers x 4MB/player; giới hạn 100 để nằm trong 2GB virtual address space
    if (g_nMaxPlayerCount > 100)
        g_nMaxPlayerCount = 100;
#endif
// x64: không giới hạn — có đủ virtual address space
```

### InterlockedExchangePointer

Trên Win32 cũ, macro `InterlockedExchangePointer` không tồn tại — cần define thủ công. Trên x64, Windows SDK cung cấp sẵn:

```cpp
// OpaqueUserData.h:23
#if !defined(InterlockedExchangePointer) && !defined(_WIN64)
    #define InterlockedExchangePointer(Target, Value) \
        (PVOID)InterlockedExchange((PLONG)(Target), (LONG)(Value))
#endif
```

---
