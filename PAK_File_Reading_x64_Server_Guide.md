# Hướng Dẫn Chuyển Đổi & Đọc File PAK Trên Server x64

## Mục Lục

1. [Tổng Quan Vấn Đề](#1-tổng-quan-vấn-đề)
2. [Thay Đổi Mã Nguồn (Source Code)](#2-thay-đổi-mã-nguồn-source-code)
   - 2.1. Gỡ bỏ guard `#ifndef _SERVER` trong KPakFile
   - 2.2. Kích hoạt PAK init trong KSubWorld::LoadMap
   - 2.3. Sửa đường dẫn MapList.ini
3. [Mở Rộng Giới Hạn PAK Từ 32 Lên 64 (Client)](#3-mở-rộng-giới-hạn-pak-từ-32-lên-64-client)
   - 3.1. Tại sao cần mở rộng
   - 3.2. Các file cần sửa
   - 3.3. Chi tiết thay đổi
   - 3.4. Lưu ý ABI
4. [Build & Triển Khai](#4-build--triển-khai)
5. [Cấu Trúc Thư Mục Server x64](#5-cấu-trúc-thư-mục-server-x64)
6. [Định Dạng Nhị Phân File PAK](#6-định-dạng-nhị-phân-file-pak)
7. [Xử Lý Lỗi Thường Gặp](#7-xử-lý-lỗi-thường-gặp)

---

## 1. Tổng Quan Vấn Đề

Mặc định, engine Sword3 **vô hiệu hóa** chức năng đọc PAK trên server thông qua các guard biên dịch `#ifndef _SERVER`. Điều này có nghĩa:

- Server không thể đọc tài nguyên từ file `.pak` — chỉ đọc file rời trên ổ đĩa.
- Biến thành viên `m_PackRef` trong `KPakFile` bị loại khỏi bản build server → **sizeof(KPakFile) không nhất quán** giữa Engine.dll và CoreServer.dll.
- Lời gọi `g_PakList.Open("\\package.ini")` trong `KSubWorld::LoadMap()` bị comment.
- Macro `MAPLIST_SETTING_FILE` trỏ sai file (`WorldSet.ini` thay vì `MapList.ini`).

Ngoài ra, trên client, `MAX_PAK` gốc chỉ là **32**, trong khi `package.ini` thực tế có thể liệt kê **36 file PAK trở lên** (index 0-35). Các file PAK ở index 32+ (ví dụ: `ui.pak`, `script.pak`, `sound.pak`, `font.pak`) sẽ bị **âm thầm bỏ qua** mà không báo lỗi.

Để server x64 đọc được PAK **và** client hỗ trợ đủ 64 file PAK, cần thực hiện tất cả các bước bên dưới.

---

## 2. Thay Đổi Mã Nguồn (Source Code)

### 2.1. Gỡ Bỏ Guard `#ifndef _SERVER` Trong KPakFile

#### File: `Engine\Src\KPakFile.h`

**Trước:**
```cpp
#ifndef _SERVER
#include "XPackFile.h"
#endif
// ...
class ENGINE_API KPakFile
{
    // ...
private:
    KFile       m_File;
#ifndef _SERVER
    XPackElemFileRef m_PackRef;
#endif
};
```

**Sau:**
```cpp
#include "XPackFile.h"
// ...
class ENGINE_API KPakFile
{
    // ...
private:
    KFile       m_File;
    XPackElemFileRef m_PackRef;
};
```

> **Lưu ý quan trọng:** Giữ nguyên guard `#ifndef _SERVER` cho các hàm render SPR/JPEG (dòng 39-55 trong KPakFile.h) vì chúng phụ thuộc vào kiểu DirectDraw (`SPRHEAD`, `SPRFRAME`) không có trên server.

#### File: `Engine\Src\KPakFile.cpp`

Gỡ bỏ **9 khối guard** `#ifndef _SERVER` bao quanh các đoạn code sau:

| # | Vị Trí | Nội Dung Được Gỡ Guard |
|---|--------|------------------------|
| 1 | Đầu file | `#include "KPakList.h"` — giờ include vô điều kiện |
| 2 | Constructor | Khởi tạo `m_PackRef.nPackIndex = -1; m_PackRef.uId = 0;` |
| 3 | `IsFileInPak()` | `return (m_PackRef.nPackIndex >= 0 && m_PackRef.uId);` |
| 4 | `Open()` | Logic tìm file trong PAK qua `g_pPakList->FindElemFile()` |
| 5 | `Read()` | Đường code đọc từ PAK qua `g_pPakList->ElemFileRead()` |
| 6 | `Seek()` | Đường code seek trong PAK (thao tác trên `m_PackRef.nOffset`) |
| 7 | `Tell()` | Trả về `m_PackRef.nOffset` khi file nằm trong PAK |
| 8 | `Size()` | Trả về `m_PackRef.nSize` khi file nằm trong PAK |
| 9 | `Close()` | Reset `m_PackRef.nPackIndex = -1; m_PackRef.uId = 0;` |

> Giữ nguyên guard `#ifndef _SERVER` cho các hàm SPR/JPEG (dòng 31-229).

#### Ví dụ chi tiết — hàm `Open()` sau khi gỡ guard:

```cpp
BOOL KPakFile::Open(const char* pszFileName)
{
    if (pszFileName == NULL || pszFileName[0] == 0)
        return false;

    bool bOk = false;
    Close();

    if (m_nPakFileMode == 0)    // 0 = ưu tiên đọc từ đĩa
    {
        bOk = (m_File.Open((char*)pszFileName) != FALSE);
        if (bOk == false && g_pPakList)     // <-- trước đây bị guard
        {
            bOk = g_pPakList->FindElemFile(pszFileName, m_PackRef);
        }
    }
    else    // 1 = ưu tiên đọc từ PAK
    {
        if (g_pPakList)                      // <-- trước đây bị guard
            bOk = g_pPakList->FindElemFile(pszFileName, m_PackRef);
        if (bOk == false)
            bOk = (m_File.Open((char*)pszFileName) != FALSE);
    }
    return bOk;
}
```

---

### 2.2. Kích Hoạt PAK Init Trong KSubWorld::LoadMap

#### File: `Core\Src\KSubWorld.cpp`

**Bước 1 — Khai báo biến toàn cục** (dòng 44-46, đã có sẵn):

```cpp
#ifdef _SERVER
KGlobalMissionArray g_GlobalMissionArray;
KPakList g_PakList;     // <-- instance PAK list cho server
#endif
```

Khi `g_PakList` được khởi tạo, constructor của `KPakList` tự động gán `g_pPakList = this`, cho phép tầng Engine truy cập.

**Bước 2 — Bỏ comment lời gọi Open** (dòng 641):

```cpp
// TRƯỚC (bị comment):
//g_PakList.Open("\\package.ini");

// SAU (kích hoạt):
g_PakList.Open("\\package.ini");
```

> **Lưu ý về x64:** Trên x64 server, mảng `SubWorld` được cấp phát trên **heap** (không phải stack) thông qua `g_AllocSubWorld()` được gọi trong `KCore::Init()`. Điều này tự động xảy ra khi cả `_WIN64` và `_SERVER` đều được define.

---

### 2.3. Sửa Đường Dẫn MapList.ini

#### File: `Core\Src\CoreUseNameDef.h` (dòng 139)

```cpp
// TRƯỚC (sai tên file):
#define MAPLIST_SETTING_FILE "\\settings\\WorldSet.ini"

// SAU (đúng tên file):
#define MAPLIST_SETTING_FILE "\\settings\\MapList.ini"
```

File `MapList.ini` chứa ánh xạ từ **ID bản đồ** sang **đường dẫn thư mục** trong `\maps\`:

```ini
[List]
Count=998
1=phong_tuong\phuong_tuong
1_name=Phường Tương
2=phong_tuong\hoa_son
2_name=Hoa Sơn
...
```

Khi `LoadMap(nId)` được gọi, nó tra cứu key `nId` trong section `[List]` để lấy đường dẫn thư mục, rồi đọc các file `.wor` và `.obj` từ `\maps\{đường_dẫn}\`.

---

## 3. Mở Rộng Giới Hạn PAK Từ 32 Lên 64 (Client)

### 3.1. Tại Sao Cần Mở Rộng

Engine gốc định nghĩa `MAX_PAK = 32`. Vòng lặp trong `KPakList::Open()` chỉ đọc tối đa 32 entry (index 0-31) từ `package.ini`:

```cpp
for (int i = 0; i < MAX_PAK; i++)  // MAX_PAK = 32 → chỉ đọc 0..31
{
    itoa(i, szKey, 10);
    if (!IniFile.GetString(SECTION, szKey, "", szBuffer, sizeof(szBuffer)))
        break;
    // ... mở file PAK ...
}
```

Nếu `package.ini` có 36 entry (index 0-35), thì **4 file PAK cuối bị âm thầm bỏ qua** — không có thông báo lỗi. Ví dụ với `package.ini` thực tế của client:

```ini
[Package]
Path=\\data
0=vltkpatch.pak
1=vltkcache.pak
...
31=settings.pak
32=ui.pak          ← BỊ BỎ QUA (index >= 32)
33=script.pak      ← BỊ BỎ QUA
34=sound.pak       ← BỊ BỎ QUA
35=font.pak        ← BỊ BỎ QUA
```

Hậu quả: thiếu giao diện UI, thiếu script Lua, thiếu âm thanh, thiếu font — nhưng có thể bị che bởi cơ chế fallback đọc file rời trên đĩa (nếu file đó tồn tại ngoài PAK).

### 3.2. Các File Cần Sửa

Cần sửa **3 file header** trong `Engine\Src\`:

| File | Dòng | Giá Trị Cũ | Giá Trị Mới | Tác Động |
|------|------|-------------|-------------|----------|
| `KPakList.h` | 37 | `#define MAX_PAK 32` | `#define MAX_PAK 64` | Thay đổi `sizeof(KPakList)` — mảng `m_PakFilePtrList[MAX_PAK]` tăng kích thước |
| `KZipList.h` | 16 | `#define MAX_PAK 32` | `#define MAX_PAK 64` | Thay đổi `sizeof(KZipList)` — mảng `m_ZipFile[MAX_PAK]` tăng kích thước |
| `XPackFile.h` | 83 | `#define MAX_XPACKFILE_CACHE 10` | `#define MAX_XPACKFILE_CACHE 16` | Tăng cache LRU cho tỉ lệ hit tốt hơn khi có nhiều PAK |

Và thêm log cảnh báo trong **1 file cpp**:

| File | Vị Trí | Nội Dung Thêm |
|------|--------|---------------|
| `KPakList.cpp` | Sau vòng lặp load (sau dòng 168) | Log cảnh báo khi PAK count > 56 |

### 3.3. Chi Tiết Thay Đổi

#### 3.3.1. `Engine\Src\KPakList.h` — dòng 37

```cpp
// TRƯỚC:
private:
    #define MAX_PAK     32
    XPackFile*          m_PakFilePtrList[MAX_PAK];

// SAU:
private:
    #define MAX_PAK     64  // NOTE: also defined in KZipList.h - keep in sync
    XPackFile*          m_PakFilePtrList[MAX_PAK];
```

#### 3.3.2. `Engine\Src\KZipList.h` — dòng 16

```cpp
// TRƯỚC:
#define MAX_PAK     32

// SAU:
#define MAX_PAK     64  // NOTE: also defined in KPakList.h - keep in sync
```

> **Tại sao phải sửa cả KZipList.h?** Mặc dù hệ thống KZipList hiện không được sử dụng (biến `g_pZipList` không bao giờ được khởi tạo), nhưng cần giữ đồng bộ để tránh nhầm lẫn bảo trì sau này. Hai file định nghĩa `MAX_PAK` **độc lập** — không có kiểm tra lúc biên dịch nếu chúng lệch nhau.

#### 3.3.3. `Engine\Src\XPackFile.h` — dòng 83

```cpp
// TRƯỚC:
    #define MAX_XPACKFILE_CACHE     10

// SAU:
    #define MAX_XPACKFILE_CACHE     16
```

> **Tại sao tăng cache?** Khi số lượng PAK tăng gấp đôi (32→64), nhiều file PAK hơn cạnh tranh cùng cache LRU. Tăng từ 10 lên 16 slot giúp duy trì tỉ lệ cache hit hợp lý.

#### 3.3.4. `Engine\Src\KPakList.cpp` — thêm log cảnh báo

Thêm **sau vòng lặp for** (sau dòng 168), trước `bResult = true`:

```cpp
            // ... (kết thúc vòng lặp for) ...
            }

            // Thêm đoạn này:
            if (m_nPakNumber > 56)
            {
                g_DebugLog("[WARN] PAK limit approaching: %d of %d (MAX_PAK=%d)",
                    m_nPakNumber, MAX_PAK, MAX_PAK);
            }
            g_DebugLog("PakList: %d PAK files loaded", m_nPakNumber);

            bResult = true;
```

### 3.4. Lưu Ý ABI — Bắt Buộc Build Lại Toàn Bộ

Thay đổi `MAX_PAK` và `MAX_XPACKFILE_CACHE` **thay đổi kích thước struct** của `KPakList`, `KZipList`, và mảng cache static trong `XPackFile`. Nếu một DLL/EXE được build với giá trị cũ nhưng DLL khác dùng giá trị mới, sẽ xảy ra **lệch bộ nhớ (memory corruption)** dẫn đến crash.

**Chuỗi include bị ảnh hưởng:**

```
KPakList.h (định nghĩa MAX_PAK, kích thước m_PakFilePtrList[MAX_PAK])
  ├── KPakList.cpp    (Engine)
  ├── KFilePath.cpp   (Engine)
  ├── KImageRes.cpp   (Engine)
  ├── KPakFile.cpp    (Engine)
  ├── KSubWorld.cpp   (Core)
  ├── KSubWorldSet.cpp(Core)
  └── S3Client.cpp    (S3Client) ← khởi tạo KPakList g_PakList

KZipList.h (định nghĩa MAX_PAK, kích thước m_ZipFile[MAX_PAK])
  ├── KZipList.cpp    (Engine)
  └── KZipFile.cpp    (Engine)

XPackFile.h (định nghĩa MAX_XPACKFILE_CACHE, kích thước mảng cache static)
  ├── KPakList.h      (ảnh hưởng tất cả file include KPakList.h)
  ├── KPakFile.h      (ảnh hưởng Engine consumers)
  └── XPackFile.cpp   (Engine)
```

> **BẮT BUỘC:** Clean build toàn bộ solution sau khi thay đổi. Xóa thư mục `Build/Lib/` và rebuild tất cả cấu hình (Client Release, Server Release, Server Release x64, Debug). Kiểm tra timestamp của Engine.dll, CoreServer.dll/CoreServer64.dll, và Game.exe/S3Client.exe phải cùng thời điểm build.

---

## 4. Build & Triển Khai

### Quy Trình Build

1. Mở solution `JXAll.sln`
2. **Clean** toàn bộ solution (Build → Clean Solution)
3. Xóa thủ công thư mục `Build/Lib/` để đảm bảo không còn lib cũ
4. Build từng cấu hình theo thứ tự:
   - **Client Release** (Win32) → tạo Game.exe + các DLL client
   - **Server Release** (Win32) → tạo GameServer.exe + CoreServer.dll + Engine.dll
   - **Server Release x64** → tạo GameServer64.exe + CoreServer64.dll + Engine64.dll
5. Kiểm tra log build — không được có warning về macro redefinition

### Bảng DLL x64

| Thành Phần | Win32 | x64 |
|------------|-------|-----|
| Game Server | GameServer.exe | **GameServer64.exe** |
| Core DLL | CoreServer.dll | **CoreServer64.dll** |
| Engine DLL | engine.dll | **Engine64.dll** |
| Lua DLL | LuaLibDll.dll | **LuaLibDll64.dll** |
| Filter DLL | FilterText.dll | **FilterText64.dll** |
| Rainbow DLL | Rainbow.dll | **rainbow64.dll** |
| Heaven DLL | heaven.dll | **heaven64.dll** |

### Bảng Lib x64

| Lib | Win32 | x64 |
|-----|-------|-----|
| Common | Common.lib | **Common64.lib** |
| Engine | Engine.lib | **Engine64.lib** |
| Core | CoreServer.lib | **CoreServer64.lib** |
| BDB | libdb62s.lib | **libdb62s_x64.lib** |

### Static CRT (/MT)

Tất cả project dùng cờ `/MT` (static CRT) — **không cần** cài VC++ Redistributable trên máy đích. BDB x64 cũng được build với `/MT`.

---

## 5. Cấu Trúc Thư Mục Server x64

Tương đối so với thư mục chạy `GameServer64.exe`:

```
GameServer64.exe
CoreServer64.dll
Engine64.dll
LuaLibDll64.dll
FilterText64.dll
rainbow64.dll
heaven64.dll
ExpandPackage.dll
|
├── package.ini                    ← CẤU HÌNH PAK (bắt buộc)
├── pak/                           ← THƯ MỤC CHỨA FILE PAK
│   ├── maps.pak
│   ├── update_map.pak             (tùy chọn)
│   ├── qianchonglou.mps           (tùy chọn)
│   └── ...
|
├── settings/
│   ├── MapList.ini                ← ÁNH XẠ ID BẢN ĐỒ (bắt buộc)
│   └── ...các file cấu hình khác
|
└── maps/                          ← FILE BẢN ĐỒ RỜI (dự phòng)
    ├── map_folder_1/
    │   ├── 0_0.rgn
    │   └── ...
    └── map_folder_2/
        └── ...
```

### Định Dạng `package.ini`

```ini
[Package]
Path=\pak
0=maps.pak
1=update_map.pak
2=qianchonglou.mps
```

- **`Path`**: Đường dẫn tương đối đến thư mục chứa file PAK (từ thư mục gốc server).
- **`0`, `1`, `2`, ...**: Tên file PAK, load theo thứ tự. Tối đa **64 file** (sau khi mở rộng).
- Log kết quả: `"PakList Open : {đường_dẫn} ... Ok"` hoặc `"... Fail"`.

### Cơ Chế Fallback

Khi `KPakFile::Open("somefile")` được gọi:

| Mode | Thứ Tự Tìm Kiếm |
|------|-----------------|
| **0** (mặc định) | Đĩa trước → PAK sau |
| **1** | PAK trước → Đĩa sau |

Nghĩa là: nếu không có file PAK, server tự động đọc file rời từ `\maps\` mà không cần thay đổi gì.

---

## 6. Định Dạng Nhị Phân File PAK

Nếu bạn cần tạo hoặc kiểm tra file PAK, đây là cấu trúc nhị phân:

### Header (32 byte)

```
Offset  Kích Thước  Trường               Mô Tả
0x00    4 byte      cSignature           Magic bytes: "PACK" (0x4B434150)
0x04    4 byte      uCount               Số lượng file con trong PAK
0x08    4 byte      uIndexTableOffset    Offset byte đến bảng index
0x0C    4 byte      uDataOffset          Offset byte đến vùng dữ liệu
0x10    4 byte      uCrc32               Checksum CRC32
0x14    12 byte     cReserved            Dự trữ (toàn zero)
```

### Bảng Index (16 byte mỗi entry)

Nằm tại vị trí `uIndexTableOffset`, gồm `uCount` entry, **sắp xếp theo uId** (binary search):

```
Offset  Kích Thước  Trường               Mô Tả
0x00    4 byte      uId                  ID file (hash tên file)
0x04    4 byte      uOffset              Offset dữ liệu trong PAK
0x08    4 byte      lSize                Kích thước gốc (chưa nén)
0x0C    4 byte      lCompressSizeFlag    Byte cao: phương pháp nén
                                          3 byte thấp: kích thước sau nén
```

### Phương Pháp Nén (byte cao của `lCompressSizeFlag`)

| Giá Trị | Tên | Mô Tả |
|---------|-----|-------|
| `0x00` | TYPE_NONE | Không nén |
| `0x01` | TYPE_UCL | Nén UCL (NRV2B) |
| `0x02` | TYPE_BZIP2 | Nén BZIP2 |
| `0x10` | TYPE_FRAME | Nén theo frame (chỉ cho file SPR) |
| `0x20` | TYPE_UPL_NEW | Nén kiểu mới |

### Thuật Toán Hash Tên File (`FileNameToId`)

Engine dùng hàm hash tùy chỉnh để chuyển đổi tên file thành ID 32-bit:

```cpp
unsigned long FileNameToId(const char* pszFileName)
{
    unsigned long id = 0;
    int index = 0;
    const char *ptr = pszFileName;
    while (*ptr)
    {
        if (*ptr >= 'A' && *ptr <= 'Z')
            id = (id + (++index) * (*ptr + 'a' - 'A')) % 0x8000000b * 0xffffffef;
        else
            id = (id + (++index) * (*ptr)) % 0x8000000b * 0xffffffef;
        ptr++;
    }
    return (id ^ 0x12345678);
}
```

- Tên file được **chuyển thành lowercase** trước khi hash.
- Đường dẫn sử dụng `\` (backslash) làm phân cách.
- Bảng index trong PAK **phải được sắp xếp theo uId** để binary search hoạt động.

---

## 7. Xử Lý Lỗi Thường Gặp

### 7.1. File PAK Không Load Được

**Triệu chứng:** Log server hiện `"PakList Open : {path} ... Fail"`.

**Kiểm tra:**
1. `package.ini` có tồn tại trong thư mục gốc server không?
2. Giá trị `Path=` có đúng đường dẫn tương đối không?
3. Các file `.pak` / `.mps` có thực sự tồn tại trong thư mục đó không?
4. Process server có quyền đọc file không?
5. File PAK có đúng signature "PACK" (4 byte đầu) không? — dùng hex editor kiểm tra.

### 7.2. Bản Đồ Không Load Được

**Triệu chứng:** `LoadMap()` trả về FALSE.

**Kiểm tra:**
1. File `settings\MapList.ini` phải tồn tại (không phải `WorldSet.ini`).
2. ID bản đồ cần load phải có entry tương ứng trong section `[List]` của `MapList.ini`.
3. Dữ liệu bản đồ phải tồn tại trong file PAK hoặc dạng file rời trong `\maps\`.

### 7.3. Crash Do Lệch ABI

**Triệu chứng:** Server crash khi khởi động hoặc khi truy cập file PAK. Có thể crash trong `KPakList::Close()` hoặc `KPakList::FindElemFile()`.

**Nguyên nhân:** Engine64.dll và CoreServer64.dll được build với giá trị `MAX_PAK` hoặc `sizeof(KPakFile)` khác nhau.

**Khắc phục:**
1. Xóa `Build/Lib/release-x64/`
2. Clean build toàn bộ cấu hình "Server Release x64"
3. Kiểm tra timestamp tất cả DLL phải cùng thời điểm

### 7.4. Client Thiếu Tài Nguyên (UI, Script, Font, Sound)

**Triệu chứng:** Thiếu giao diện, không có âm thanh, Lua script không chạy.

**Nguyên nhân:** `MAX_PAK` vẫn là 32 nhưng `package.ini` có >32 entry. Các file PAK ở index 32+ bị bỏ qua.

**Khắc phục:** Thực hiện bước 3 (mở rộng MAX_PAK từ 32 lên 64) và rebuild lại client.

**Kiểm chứng:** Sau khi fix, so sánh debug log trước/sau — các file trước đây đọc từ đĩa (fallback) giờ sẽ đọc từ PAK.

### 7.5. Lỗi Biên Dịch Do Encoding GBK

**Triệu chứng:** Lỗi compiler trên các dòng comment tiếng Trung sau khi sửa file.

**Nguyên nhân:** File `KPakList.h`, `KZipList.h` dùng encoding **GBK** (không phải UTF-8). Editor UTF-8 làm hỏng các byte GBK.

**Khắc phục:** Dùng Python binary patch thay vì editor thông thường:

```python
# Thay đổi MAX_PAK trong file GBK mà không hỏng encoding
import sys

filepath = sys.argv[1]  # Đường dẫn file .h
with open(filepath, 'rb') as f:
    content = f.read()

content = content.replace(
    b'#define MAX_PAK\t\t32',
    b'#define MAX_PAK\t\t64\t// NOTE: also defined in KZipList.h - keep in sync'
)

with open(filepath, 'wb') as f:
    f.write(content)
```

### 7.6. Cảnh Báo Giới Hạn PAK

**Triệu chứng:** Log hiện `"[WARN] PAK limit approaching: N of 64 (MAX_PAK=64)"`.

**Ý nghĩa:** Đang dùng hơn 56 trong 64 slot. Cân nhắc gộp các file PAK nhỏ lại hoặc tăng `MAX_PAK` (nhớ sửa cả 2 file header và rebuild toàn bộ).

---

## Tham Khảo Nhanh

```
package.ini        → Liệt kê file PAK cần load (tối đa 64)
MapList.ini        → Ánh xạ ID bản đồ → đường dẫn thư mục
g_PakList          → Instance KPakList phía server (trong KSubWorld.cpp)
g_pPakList         → Con trỏ export từ Engine (gán bởi constructor KPakList)
KPakFile::Open()   → Phân giải file từ đĩa hoặc PAK một cách trong suốt
MAX_PAK = 64       → Số lượng file PAK tối đa (sau mở rộng)
LRU cache = 16     → Số slot cache file đã giải nén (sau mở rộng)
Signature = "PACK" → 4 byte magic header của file PAK (0x4B434150)
/MT                → Static CRT — không cần VC++ Redistributable
```
