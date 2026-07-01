## Chương 50
# Các Thao Tác Virtual Memory

Chương này xem xét các system call thực hiện thao tác trên virtual address space của process:

- `mprotect()`: thay đổi bảo vệ trên vùng virtual memory.
- `mlock()` và `mlockall()`: lock vùng virtual memory vào physical memory, ngăn bị swap out.
- `mincore()`: xác định xem các page trong vùng virtual memory có đang nằm trong physical memory không.
- `madvise()`: cho phép process báo cho kernel biết về pattern sử dụng tương lai của vùng virtual memory.

---

## 50.1 Thay Đổi Bảo Vệ Memory: mprotect()

System call `mprotect()` thay đổi bảo vệ trên các virtual memory page trong phạm vi bắt đầu tại `addr` và kéo dài `length` byte.

```c
#include <sys/mman.h>
int mprotect(void *addr, size_t length, int prot);
                                             Returns 0 on success, or –1 on error
```

Giá trị trong `addr` phải là bội số của kích thước page hệ thống. Vì bảo vệ được đặt trên toàn bộ page, `length` thực chất được làm tròn lên bội số tiếp theo.

Đối số `prot` là bit mask chỉ định bảo vệ mới: `PROT_NONE` hoặc tổ hợp OR của `PROT_READ`, `PROT_WRITE`, và `PROT_EXEC`. Tất cả có cùng ý nghĩa như đối với `mmap()`.

Nếu process cố truy cập vùng memory theo cách vi phạm bảo vệ, kernel tạo signal `SIGSEGV`.

**Listing 50-1:** Thay đổi bảo vệ memory bằng `mprotect()`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– vmem/t_mprotect.c
#define _BSD_SOURCE
#include <sys/mman.h>
#include "tlpi_hdr.h"
#define LEN (1024 * 1024)
#define SHELL_FMT "cat /proc/%ld/maps | grep zero"
#define CMD_SIZE (sizeof(SHELL_FMT) + 20)
int
main(int argc, char *argv[])
{
    char cmd[CMD_SIZE];
    char *addr;
    /* Tạo anonymous mapping với tất cả truy cập bị từ chối */
    addr = mmap(NULL, LEN, PROT_NONE, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (addr == MAP_FAILED)
        errExit("mmap");
    printf("Before mprotect()\n");
    snprintf(cmd, CMD_SIZE, SHELL_FMT, (long) getpid());
    system(cmd);
    /* Thay đổi bảo vệ để cho phép read và write */
    if (mprotect(addr, LEN, PROT_READ | PROT_WRITE) == -1)
        errExit("mprotect");
    printf("After mprotect()\n");
    system(cmd);
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– vmem/t_mprotect.c
```

---

## 50.2 Memory Locking: mlock() và mlockall()

Trong một số ứng dụng, hữu ích khi lock một phần hoặc toàn bộ virtual memory của process để đảm bảo nó luôn nằm trong physical memory. Lý do:

1. **Cải thiện hiệu năng**: Truy cập vào locked page không bao giờ bị trễ bởi page fault.
2. **Bảo mật**: Nếu virtual memory page chứa dữ liệu nhạy cảm không bao giờ bị swap out, không có bản sao nào được ghi vào đĩa.

### RLIMIT_MEMLOCK resource limit

Giới hạn tài nguyên `RLIMIT_MEMLOCK` định nghĩa giới hạn số byte mà process có thể lock vào bộ nhớ.

- Trước Linux 2.6.9: chỉ process có đặc quyền (`CAP_IPC_LOCK`) có thể lock bộ nhớ.
- Từ Linux 2.6.9: process không có đặc quyền có thể lock lượng nhỏ bộ nhớ lên đến giới hạn mềm của `RLIMIT_MEMLOCK`. Với process có đặc quyền, `RLIMIT_MEMLOCK` bị bỏ qua.

Giá trị mặc định cho cả giới hạn mềm và cứng của `RLIMIT_MEMLOCK` là 8 page (32.768 byte trên x86-32).

`RLIMIT_MEMLOCK` ảnh hưởng đến:
- `mlock()` và `mlockall()`;
- flag `MAP_LOCKED` của `mmap()`;
- thao tác `SHM_LOCK` của `shmctl()`.

### Lock và unlock vùng bộ nhớ

```c
#include <sys/mman.h>
int mlock(void *addr, size_t length);
int munlock(void *addr, size_t length);
                                         Both return 0 on success, or –1 on error
```

System call `mlock()` lock tất cả page của virtual address range bắt đầu tại `addr` và kéo dài `length` byte. `addr` không cần align theo page — kernel lock các page bắt đầu từ page boundary tiếp theo bên dưới `addr`.

Sau lời gọi `mlock()` thành công, tất cả page trong phạm vi được đảm bảo lock và nằm trong physical memory. `mlock()` thất bại nếu không đủ physical memory hoặc vi phạm giới hạn `RLIMIT_MEMLOCK`.

System call `munlock()` thực hiện ngược `mlock()`, xóa memory lock. Các page đã unlock không được đảm bảo ngừng nằm trong RAM.

Memory lock tự động bị xóa khi:
- Process kết thúc;
- Các page đã lock được unmap qua `munmap()`;
- Các page đã lock bị phủ bằng flag `MAP_FIXED` của `mmap()`.

### Chi tiết ngữ nghĩa memory locking

- Memory lock **không kế thừa** qua `fork()` và **không được bảo toàn** qua `exec()`.
- Khi nhiều process chia sẻ tập page (ví dụ: `MAP_SHARED` mapping), các page đó vẫn locked miễn là ít nhất một process giữ lock.
- Memory lock **không nest** cho một process đơn lẻ: gọi `mlock()` nhiều lần trên cùng range chỉ thiết lập một lock.
- Vì lock theo đơn vị page, không hợp lý khi áp dụng `mlock()`/`munlock()` độc lập cho các cấu trúc dữ liệu khác nhau trên cùng virtual page.

### Lock và unlock toàn bộ bộ nhớ process

```c
#include <sys/mman.h>
int mlockall(int flags);
int munlockall(void);
                                         Both return 0 on success, or –1 on error
```

`mlockall()` lock tất cả page đã map trong virtual address space của process, tất cả page sẽ map trong tương lai, hoặc cả hai, tùy thuộc vào `flags`:

- **`MCL_CURRENT`**: Lock tất cả page hiện được map, bao gồm text, data segment, memory mapping, và stack. Sau lời gọi thành công, tất cả page được đảm bảo nằm trong bộ nhớ.
- **`MCL_FUTURE`**: Lock tất cả page map vào address space sau này. Thao tác cấp phát bộ nhớ sau có thể thất bại nếu hết RAM hoặc gặp giới hạn `RLIMIT_MEMLOCK`.

`munlockall()` unlock tất cả page và xóa hiệu ứng của bất kỳ lời gọi `mlockall(MCL_FUTURE)` nào trước đó.

**Listing 50-2:** Sử dụng `mlock()` và `mincore()`

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– vmem/memlock.c
#define _BSD_SOURCE
#include <sys/mman.h>
#include "tlpi_hdr.h"
static void
displayMincore(char *addr, size_t length)
{
    unsigned char *vec;
    long pageSize, numPages, j;
    pageSize = sysconf(_SC_PAGESIZE);
    numPages = (length + pageSize - 1) / pageSize;
    vec = malloc(numPages);
    if (vec == NULL)  errExit("malloc");
    if (mincore(addr, length, vec) == -1)  errExit("mincore");
    for (j = 0; j < numPages; j++) {
        if (j % 64 == 0)
            printf("%s%10p: ", (j == 0) ? "" : "\n", addr + (j * pageSize));
        printf("%c", (vec[j] & 1) ? '*' : '.');
    }
    printf("\n");
    free(vec);
}
int
main(int argc, char *argv[])
{
    char *addr;
    size_t len, lockLen;
    long pageSize, stepSize, j;
    if (argc != 4 || strcmp(argv[1], "--help") == 0)
        usageErr("%s num-pages lock-page-step lock-page-len\n", argv[0]);
    pageSize = sysconf(_SC_PAGESIZE);
    if (pageSize == -1)  errExit("sysconf(_SC_PAGESIZE)");
    len      = getInt(argv[1], GN_GT_0, "num-pages")       * pageSize;
    stepSize = getInt(argv[2], GN_GT_0, "lock-page-step")  * pageSize;
    lockLen  = getInt(argv[3], GN_GT_0, "lock-page-len")   * pageSize;
    addr = mmap(NULL, len, PROT_READ, MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (addr == MAP_FAILED)  errExit("mmap");
    printf("Allocated %ld (%#lx) bytes starting at %p\n",
           (long) len, (unsigned long) len, addr);
    printf("Before mlock:\n");
    displayMincore(addr, len);
    for (j = 0; j + lockLen <= len; j += stepSize)
        if (mlock(addr + j, lockLen) == -1)
            errExit("mlock");
    printf("After mlock:\n");
    displayMincore(addr, len);
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– vmem/memlock.c
```

Ví dụ chạy — lock 3 page liên tiếp trong mỗi nhóm 8 page của 32 page:
```
# ./memlock 32 8 3
Allocated 131072 (0x20000) bytes starting at 0x4014a000
Before mlock:
0x4014a000: ................................
After mlock:
0x4014a000: ***.....***.....***.....***.....
```

Dấu `*` biểu diễn page đang nằm trong bộ nhớ; dấu `.` là page không nằm trong bộ nhớ.

---

## 50.3 Xác Định Memory Residence: mincore()

System call `mincore()` là complement của các system call memory locking. Nó báo cáo page nào trong virtual address range đang nằm trong RAM, và do đó sẽ không gây page fault nếu truy cập.

```c
#define _BSD_SOURCE
#include <sys/mman.h>
int mincore(void *addr, size_t length, unsigned char *vec);
                                          Returns 0 on success, or –1 on error
```

Địa chỉ trong `addr` phải align theo page. Thông tin về memory residency được trả về trong `vec`, phải là mảng `(length + PAGE_SIZE - 1) / PAGE_SIZE` byte. Bit ít có ý nghĩa nhất của mỗi byte được đặt nếu page tương ứng đang nằm trong bộ nhớ.

Thông tin trả về bởi `mincore()` có thể thay đổi giữa lúc gọi và lúc kiểm tra `vec`. Các page duy nhất được đảm bảo vẫn nằm trong bộ nhớ là những page được lock bằng `mlock()` hoặc `mlockall()`.

---

## 50.4 Báo Trước Pattern Sử Dụng Memory: madvise()

System call `madvise()` dùng để cải thiện hiệu năng ứng dụng bằng cách thông báo cho kernel về pattern sử dụng dự kiến của process đối với các page trong phạm vi được chỉ định. Kernel có thể dùng thông tin này để cải thiện hiệu quả I/O.

```c
#define _BSD_SOURCE
#include <sys/mman.h>
int madvise(void *addr, size_t length, int advice);
                                             Returns 0 on success, or –1 on error
```

Giá trị `addr` phải align theo page. Đối số `advice` là một trong:

- **`MADV_NORMAL`**: Hành vi mặc định. Page được chuyển theo cluster (bội số nhỏ của page size). Dẫn đến một số read-ahead và read-behind.

- **`MADV_RANDOM`**: Page trong vùng này sẽ được truy cập ngẫu nhiên, nên read-ahead không có lợi. Kernel nên fetch tối thiểu dữ liệu trong mỗi lần đọc.

- **`MADV_SEQUENTIAL`**: Page trong phạm vi này sẽ được truy cập một lần, tuần tự. Kernel có thể đọc trước tích cực và page có thể được giải phóng nhanh sau khi truy cập.

- **`MADV_WILLNEED`**: Đọc trước page trong vùng này để chuẩn bị cho truy cập tương lai.

- **`MADV_DONTNEED`**: Process gọi không cần page trong vùng này nằm trong bộ nhớ nữa. Trên Linux, với `MAP_PRIVATE` region: các page đã map bị loại bỏ rõ ràng — sửa đổi bị mất, truy cập tiếp theo gây page fault khởi tạo lại page. **Ứng dụng portable không nên dựa vào ngữ nghĩa destructive của Linux cho `MADV_DONTNEED`.**

SUSv3 chuẩn hóa API này dưới tên khác `posix_madvise()` với các hằng số có tiền tố `POSIX_`: `POSIX_MADV_NORMAL`, `POSIX_MADV_RANDOM`, `POSIX_MADV_SEQUENTIAL`, `POSIX_MADV_WILLNEED`, và `POSIX_MADV_DONTNEED`.

---

## 50.5 Tóm Tắt

Trong chương này, chúng ta xem xét các thao tác khác nhau có thể thực hiện trên virtual memory của process:

- `mprotect()`: thay đổi bảo vệ trên vùng virtual memory.
- `mlock()` và `mlockall()`: lock một phần hoặc toàn bộ virtual address space vào physical memory.
- `mincore()`: báo cáo page nào trong vùng virtual memory đang nằm trong physical memory.
- `madvise()` và `posix_madvise()`: cho phép process báo trước cho kernel về pattern sử dụng bộ nhớ dự kiến.

---

## 50.6 Bài Tập

- **50-1.** Xác minh ảnh hưởng của giới hạn tài nguyên `RLIMIT_MEMLOCK` bằng cách viết chương trình đặt giá trị cho giới hạn này rồi cố lock nhiều bộ nhớ hơn giới hạn.
- **50-2.** Viết chương trình để xác minh thao tác `MADV_DONTNEED` của `madvise()` cho writable `MAP_PRIVATE` mapping.
