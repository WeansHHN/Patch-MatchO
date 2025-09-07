
# iOS No Jailbreak Patch & Hook Framework

## Giới thiệu
Source này cung cấp hệ thống **patch offset** và **hook function** cho iOS **không jailbreak** (NoJB), hỗ trợ patch trực tiếp vào file Mach-O đã load trong memory (ví dụ `UnityFramework`, `anogs`, ...).  
Nó sử dụng cơ chế **Static Inline Hook** và **Code Patch** để thay thế function hoặc sửa byte code mà không cần quyền root.

## Các Thành Phần Chính

### Struct `StaticInlineHookBlock`
```c
typedef struct {
  uint64_t hook_vaddr;          // Virtual address của function cần hook
  uint64_t hook_size;           // Kích thước code cần hook
  uint64_t code_vaddr;          // Virtual address của code thay thế
  uint64_t code_size;           // Kích thước code thay thế

  uint64_t patched_vaddr;       // Địa chỉ sau khi patch
  uint64_t original_vaddr;      // Địa chỉ code gốc
  uint64_t instrument_vaddr;    // Địa chỉ bridge để debug / instrument

  uint64_t patch_size;          // Kích thước patch
  uint64_t patch_hash;          // Hash để kiểm tra integrity

  void *target_replace;         // Con trỏ function thay thế
  void *instrument_handler;     // Con trỏ instrument handler
} StaticInlineHookBlock;
```

---

### Macro tiện ích

#### `WeansHHN(x, y, z)`
Hook nhanh vào **UnityFramework**:
```c
WeansHHN(OFFSET, MyReplaceFunction, OriginalFunctionPtr);
```
- `OFFSET` → Offset của function cần hook  
- `MyReplaceFunction` → Hàm thay thế  
- `OriginalFunctionPtr` → Con trỏ function gốc (dùng để gọi lại nếu cần)

Ví dụ:
```c
void (*orig_update)(void *instance);
void my_update(void *instance) {
    // Code custom
    orig_update(instance);
}

WeansHHN(0x123456, my_update, orig_update);
```

---

#### `WeansHHNAnogs(x, y, z)`
Tương tự `WeansHHN` nhưng áp dụng cho **anogs.framework**:
```c
WeansHHNAnogs(OFFSET, MyReplaceFunction, OriginalFunctionPtr);
```

---

## Cách Sử Dụng

### 1. Tìm offset cần hook hoặc patch
- Dùng IDA / Hopper / Ghidra / Dump.cs để tìm RVA (Relative Virtual Address).
- Đảm bảo offset tính từ **Base Address** của Mach-O (ví dụ `UnityFramework`).

### 2. Hook function
- Dùng `WeansHHN` cho UnityFramework  
- Dùng `WeansHHNAnogs` cho anogs.framework

### 3. Patch byte code trực tiếp
**⚠ Lưu ý quan trọng:**  
Trước khi dùng `ActiveCodePatch` để patch, bạn **bắt buộc phải tạo file vá** bằng `StaticInlineHookPatch`.  
Ví dụ:
```c
// Tạo file patch
StaticInlineHookPatch("Frameworks/UnityFramework.framework/UnityFramework", 0x123456, "C0035FD6");

// Áp dụng patch
ActiveCodePatch("Frameworks/UnityFramework.framework/UnityFramework", 0x123456, "C0035FD6");
```

### 4. Tắt patch khi không cần
```c
DeactiveCodePatch("Frameworks/UnityFramework.framework/UnityFramework", 0x123456, "C0035FD6");
```

---

## Cấu hình build

### Thêm thư viện `.a`
- Đảm bảo `libpatch.a` được thêm vào **Makefile**:
```makefile
$(TWEAK_NAME)_LDFLAGS = libpatch.a
$(TWEAK_NAME)_FILES = PatchNonJB.mm
```

---

## Ưu điểm
- Hỗ trợ **No Jailbreak**  
- Có thể hook và patch trực tiếp vào binary load trong memory  
- Dùng được cho game Unity, anogs, hoặc các framework khác

## Lưu ý
- Offset phải chính xác 100%, sai offset có thể crash app.  
- Patch byte cần đúng kiến trúc (ARM64).  
- Một số app có cơ chế chống patch/hook, cần bypass trước.  
- Nếu patch mà chưa tạo file vá bằng `StaticInlineHookPatch` → `ActiveCodePatch` sẽ không hoạt động.  
- Phải link đúng **libpatch.a** trong Makefile để compile thành công.

---

## Ví dụ đầy đủ
```c
#include "PatchNonJB.h"

void (*orig_function)(void *instance);
void my_function(void *instance) {
    NSLog(@"Function hooked!");
    orig_function(instance);
}

__attribute__((constructor))
static void init() {
    WeansHHN(0x123456, my_function, orig_function);
}
```

---

## Bản quyền
Code này chỉ dùng cho mục đích nghiên cứu và phát triển. Không khuyến khích sử dụng vào mục đích vi phạm pháp luật.
