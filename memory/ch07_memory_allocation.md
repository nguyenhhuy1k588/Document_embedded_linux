## Chương 7
# Cấp Phát Bộ Nhớ (Memory Allocation)

Nhiều chương trình hệ thống cần có khả năng cấp phát thêm bộ nhớ cho các cấu trúc dữ liệu động (ví dụ: danh sách liên kết và binary tree), có kích thước phụ thuộc vào thông tin chỉ có tại runtime. Chương này mô tả các hàm dùng để cấp phát bộ nhớ trên heap hoặc stack.

---

## 7.1 Cấp Phát Bộ Nhớ Trên Heap

Process có thể cấp phát bộ nhớ bằng cách tăng kích thước heap — một segment virtual memory liên tục có kích thước thay đổi, bắt đầu ngay sau uninitialized data segment của process, và lớn hoặc co lại khi bộ nhớ được cấp phát và giải phóng. Giới hạn hiện tại của heap được gọi là **program break**.

Để cấp phát bộ nhớ, C program thường dùng họ hàm `malloc`, được mô tả ngay sau. Trước tiên, chúng ta mô tả `brk()` và `sbrk()`, là nền tảng của các hàm `malloc`.

### 7.1.1 Điều Chỉnh Program Break: brk() và sbrk()

Thay đổi kích thước heap thực ra chỉ đơn giản là yêu cầu kernel điều chỉnh ý tưởng của nó về vị trí program break của process. Ban đầu, program break nằm ngay sau cuối uninitialized data segment.

Sau khi program break tăng lên, chương trình có thể truy cập bất kỳ địa chỉ nào trong vùng được cấp phát mới, nhưng chưa có physical memory page nào được cấp phát. Kernel tự động cấp phát physical page mới khi process cố gắng truy cập địa chỉ trong các trang đó lần đầu tiên.

```c
#include <unistd.h>
int brk(void *end_data_segment);
                                            Returns 0 on success, or –1 on error
void *sbrk(intptr_t increment);
             Returns previous program break on success, or (void *) –1 on error
```

System call `brk()` đặt program break đến vị trí được chỉ định bởi `end_data_segment`. Virtual memory được cấp phát theo đơn vị page, nên `end_data_segment` thực chất được làm tròn lên đến page boundary tiếp theo.

Lời gọi `sbrk()` điều chỉnh program break bằng cách thêm `increment` vào nó. Trả về địa chỉ trước đó của program break. Lời gọi `sbrk(0)` trả về cài đặt hiện tại của program break mà không thay đổi nó.

### 7.1.2 Cấp Phát Bộ Nhớ Trên Heap: malloc() và free()

Nói chung, C program dùng họ hàm `malloc` để cấp phát và giải phóng bộ nhớ trên heap. Chúng có các ưu điểm so với `brk()` và `sbrk()`: được chuẩn hóa, dễ dùng hơn trong threaded program, cung cấp giao diện đơn giản cho phép cấp phát bộ nhớ theo đơn vị nhỏ, và cho phép tùy ý deallocate block bộ nhớ.

```c
#include <stdlib.h>
void *malloc(size_t size);
               Returns pointer to allocated memory on success, or NULL on error
```

Hàm `malloc()` cấp phát `size` byte từ heap và trả về pointer đến đầu block bộ nhớ mới. Bộ nhớ được cấp phát không được khởi tạo. Block bộ nhớ trả về luôn được align ở byte boundary phù hợp cho bất kỳ kiểu dữ liệu C nào (thường là 8-byte hoặc 16-byte).

Nếu không thể cấp phát bộ nhớ, `malloc()` trả về `NULL` và đặt `errno`. Mọi lời gọi đến `malloc()` đều phải kiểm tra giá trị trả về.

```c
#include <stdlib.h>
void free(void *ptr);
```

Hàm `free()` deallocate block bộ nhớ được trỏ bởi đối số `ptr`, phải là địa chỉ được trả về trước đó bởi `malloc()` hoặc các hàm cấp phát heap khác.

Nói chung, `free()` không hạ thấp program break, mà thêm block bộ nhớ vào danh sách free block được tái sử dụng bởi các lời gọi `malloc()` tương lai. Lý do:

- Block được giải phóng thường ở giữa heap thay vì cuối, nên không thể hạ program break.
- Giảm số lần gọi `sbrk()`.
- Trong nhiều trường hợp, hạ program break không giúp ích cho các chương trình cấp phát nhiều bộ nhớ.

Nếu đối số `free()` là `NULL`, lời gọi không làm gì cả. Dùng `ptr` sau khi gọi `free()` — ví dụ như truyền vào `free()` lần thứ hai — là lỗi dẫn đến kết quả không thể đoán trước.

### 7.1.3 Triển Khai của malloc() và free()

Triển khai `malloc()` đơn giản: đầu tiên quét danh sách block bộ nhớ đã giải phóng bởi `free()` trước đó để tìm block có kích thước lớn hơn hoặc bằng yêu cầu. Nếu block chính xác đúng kích thước thì trả về cho caller. Nếu lớn hơn thì tách ra. Nếu không có block nào đủ lớn, `malloc()` gọi `sbrk()` để cấp phát thêm bộ nhớ.

Về triển khai `free()`: khi `malloc()` cấp phát block, nó cấp phát thêm byte để giữ integer chứa kích thước block. Integer này nằm ở đầu block; địa chỉ thực trả về cho caller trỏ đến vị trí ngay sau giá trị độ dài này.

```
+-----------------+------------------------------+
| Length of block |   Memory for use by caller   |
|       (L)       |                              |
+-----------------+------------------------------+
          ^
          |
          +-- Address returned by malloc()
```

Khi block được đặt vào free list (danh sách liên kết đôi), `free()` dùng chính các byte của block để thêm vào danh sách.

Để tránh các lỗi liên quan đến cấp phát bộ nhớ, cần tuân theo các quy tắc sau:

- Sau khi cấp phát block bộ nhớ, không được chạm vào bất kỳ byte nào ngoài phạm vi block đó.
- Là lỗi khi gọi `free()` trên cùng bộ nhớ hơn một lần.
- Không bao giờ gọi `free()` với giá trị pointer không được tạo ra bởi hàm trong malloc package.
- Trong long-running program, phải đảm bảo deallocate bộ nhớ sau khi dùng xong — nếu không heap sẽ lớn dần đến giới hạn virtual memory (**memory leak**).

### Công cụ và thư viện debug malloc

glibc cung cấp các công cụ debug malloc:
- **`mtrace()` và `muntrace()`**: Cho phép bật/tắt tracing lời gọi cấp phát bộ nhớ. Dùng kết hợp với biến môi trường `MALLOC_TRACE`.
- **`mcheck()` và `mprobe()`**: Cho phép kiểm tra tính nhất quán của các block bộ nhớ.
- **`MALLOC_CHECK_`**: Biến môi trường kiểm soát cách chương trình phản hồi với lỗi cấp phát.

Các thư viện debug malloc như Electric Fence, dmalloc, Valgrind, và Insure++ cung cấp API giống `malloc` chuẩn nhưng kiểm tra lỗi nhiều hơn.

### Kiểm soát và giám sát malloc package

- **`mallopt()`**: Sửa đổi các tham số kiểm soát thuật toán của `malloc()`.
- **`mallinfo()`**: Trả về structure chứa thống kê về bộ nhớ được cấp phát.

### 7.1.4 Các Phương Pháp Khác Cấp Phát Bộ Nhớ Trên Heap

#### calloc() và realloc()

Hàm `calloc()` cấp phát bộ nhớ cho mảng các item giống nhau và **khởi tạo về 0**.

```c
#include <stdlib.h>
void *calloc(size_t numitems, size_t size);
               Returns pointer to allocated memory on success, or NULL on error
```

Ví dụ:
```c
struct myStruct *p;
p = calloc(1000, sizeof(struct myStruct));
if (p == NULL)
    errExit("calloc");
```

Hàm `realloc()` dùng để thay đổi kích thước (thường là tăng) block bộ nhớ đã cấp phát.

```c
#include <stdlib.h>
void *realloc(void *ptr, size_t size);
               Returns pointer to allocated memory on success, or NULL on error
```

Khi thành công, `realloc()` trả về pointer đến vị trí của block đã thay đổi kích thước — có thể khác với vị trí trước. Khi thất bại, trả về `NULL` và không chạm vào block được trỏ bởi `ptr`.

**Quan trọng**: Phải dùng pointer trả về từ `realloc()` cho các tham chiếu tiếp theo đến block bộ nhớ:

```c
nptr = realloc(ptr, newsize);
if (nptr == NULL) {
    /* Xử lý lỗi */
} else {
    ptr = nptr;
}
```

Không gán trực tiếp `ptr = realloc(ptr, newsize)` vì nếu thất bại, `ptr` sẽ là `NULL`, làm block ban đầu không thể truy cập.

#### Cấp Phát Bộ Nhớ Align: memalign() và posix_memalign()

Các hàm `memalign()` và `posix_memalign()` cấp phát bộ nhớ bắt đầu tại địa chỉ được align ở ranh giới lũy thừa của 2 được chỉ định.

```c
#include <malloc.h>
void *memalign(size_t boundary, size_t size);
               Returns pointer to allocated memory on success, or NULL on error
```

`memalign()` cấp phát `size` byte bắt đầu tại địa chỉ align theo bội số của `boundary`, phải là lũy thừa của 2.

SUSv3 quy định hàm tương tự là `posix_memalign()`:

```c
#include <stdlib.h>
int posix_memalign(void **memptr, size_t alignment, size_t size);
                      Returns 0 on success, or a positive error number on error
```

`posix_memalign()` khác `memalign()` ở: địa chỉ trả về qua `memptr`; `alignment` phải là bội số lũy thừa của 2 của `sizeof(void *)`.

---

## 7.2 Cấp Phát Bộ Nhớ Trên Stack: alloca()

Giống các hàm trong malloc package, `alloca()` cấp phát bộ nhớ động. Tuy nhiên, thay vì lấy bộ nhớ từ heap, `alloca()` lấy từ stack bằng cách tăng kích thước stack frame.

```c
#include <alloca.h>
void *alloca(size_t size);
                                  Returns pointer to allocated block of memory
```

**Không cần — và không được — gọi `free()` để deallocate bộ nhớ được cấp phát bằng `alloca()`**. Bộ nhớ được tự động giải phóng khi stack frame bị xóa (tức là khi hàm gọi `alloca()` trả về).

Không thể dùng `realloc()` để thay đổi kích thước block được cấp phát bởi `alloca()`.

`alloca()` không phải là một phần của SUSv3, nhưng được cung cấp trên hầu hết UNIX.

Nếu stack overflow do gọi `alloca()`, hành vi chương trình không thể đoán trước. Đặc biệt, không nhận được `NULL` để thông báo lỗi.

Không thể dùng `alloca()` trong danh sách đối số hàm:
```c
func(x, alloca(size), z);   /* SAI! */
```
Phải viết:
```c
void *y = alloca(size);
func(x, y, z);              /* Đúng */
```

**Ưu điểm của `alloca()`**:
- Nhanh hơn `malloc()` vì được compiler triển khai như inline code.
- Không cần duy trì free list.
- Bộ nhớ tự động giải phóng khi hàm trả về — đơn giản hóa code.
- Đặc biệt hữu ích khi dùng `longjmp()` hoặc `siglongjmp()` để thực hiện nonlocal goto — tránh memory leak.

---

## 7.3 Tóm Tắt

Dùng họ hàm `malloc`, process có thể cấp phát và giải phóng bộ nhớ động trên heap. Khi xem xét triển khai của các hàm này, chúng ta thấy nhiều thứ có thể xảy ra sai khi chương trình xử lý sai block bộ nhớ, và có nhiều công cụ debug để giúp tìm nguồn gốc của các lỗi đó.

Hàm `alloca()` cấp phát bộ nhớ trên stack. Bộ nhớ này được tự động deallocate khi hàm gọi `alloca()` trả về.

---

## 7.4 Bài Tập

- **7-1.** Sửa đổi chương trình trong Listing 7-1 để in ra giá trị hiện tại của program break sau mỗi lần thực hiện `malloc()`. Điều này sẽ chứng minh rằng `malloc()` không dùng `sbrk()` để điều chỉnh program break mỗi lần gọi, mà thay vào đó định kỳ cấp phát các chunk lớn hơn.
- **7-2.** (Nâng cao) Triển khai `malloc()` và `free()`.
