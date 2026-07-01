## Chương 49
# Memory Mapping

Chương này thảo luận về việc sử dụng system call `mmap()` để tạo memory mapping. Memory mapping có thể được dùng cho IPC cũng như nhiều mục đích khác.

---

## 49.1 Tổng Quan

System call `mmap()` tạo memory mapping mới trong virtual address space của process gọi. Mapping có thể là hai loại:

- **File mapping**: File mapping ánh xạ một vùng file trực tiếp vào virtual memory của process gọi. Sau khi file được map, nội dung của nó có thể được truy cập bằng các thao tác trên byte trong vùng bộ nhớ tương ứng. Các page của mapping được tải (tự động) từ file khi cần. Loại mapping này còn gọi là file-based mapping hoặc memory-mapped file.
- **Anonymous mapping**: Anonymous mapping không có file tương ứng. Thay vào đó, các page của mapping được khởi tạo về 0.

Bộ nhớ trong mapping của một process có thể được chia sẻ với mapping trong các process khác (tức là các page-table entry của mỗi process trỏ đến cùng page RAM). Điều này có thể xảy ra theo hai cách:

- Khi hai process map cùng vùng của file, chúng chia sẻ cùng physical page.
- Child process được tạo bởi `fork()` kế thừa bản sao mapping của parent, và các mapping này tham chiếu đến cùng physical page như mapping tương ứng trong parent.

Khi hai hoặc nhiều process chia sẻ cùng page, mỗi process có thể nhìn thấy thay đổi nội dung page của process khác, tùy thuộc vào loại mapping:

- **Private mapping (MAP_PRIVATE)**: Thay đổi nội dung mapping không hiển thị với process khác và, với file mapping, không được ghi vào file bên dưới. Kernel thực hiện điều này bằng kỹ thuật copy-on-write: bất cứ khi nào process cố sửa đổi nội dung page, kernel tạo bản sao mới riêng biệt của page đó cho process. Đây là lý do `MAP_PRIVATE` đôi khi gọi là private, copy-on-write mapping.
- **Shared mapping (MAP_SHARED)**: Thay đổi nội dung mapping hiển thị với process khác chia sẻ cùng mapping và, với file mapping, được ghi vào file bên dưới.

**Bảng 49-1:** Mục đích của các loại memory mapping

| Tính hiển thị | File | Anonymous |
|---------------|------|-----------|
| Private | Khởi tạo bộ nhớ từ nội dung file | Cấp phát bộ nhớ |
| Shared | Memory-mapped I/O; chia sẻ bộ nhớ giữa process (IPC) | Chia sẻ bộ nhớ giữa process (IPC) |

Bốn loại memory mapping:

- **Private file mapping**: Nội dung mapping được khởi tạo từ vùng file. Nhiều process map cùng file ban đầu chia sẻ cùng physical page, nhưng copy-on-write được dùng. Dùng chính để: cho phép nhiều process chia sẻ text segment (read-only), và map initialized data segment.

- **Private anonymous mapping**: Mỗi lời gọi `mmap()` tạo mapping mới riêng biệt. Dù child kế thừa mapping, copy-on-write đảm bảo parent và child không thấy thay đổi của nhau. Mục đích chính: cấp phát bộ nhớ zero-filled cho process.

- **Shared file mapping**: Tất cả process mapping cùng vùng file chia sẻ cùng physical page, được khởi tạo từ file. Sửa đổi nội dung mapping được ghi vào file. Phục vụ hai mục đích: memory-mapped I/O và chia sẻ bộ nhớ giữa process không liên quan (IPC).

- **Shared anonymous mapping**: Mỗi lời gọi `mmap()` tạo mapping mới riêng biệt, nhưng không copy-on-write. Sau `fork()`, parent và child chia sẻ cùng physical page, và thay đổi của một process hiển thị với process kia.

Mapping bị mất khi process thực hiện `exec()`, nhưng được kế thừa bởi child của `fork()`.

---

## 49.2 Tạo Mapping: mmap()

System call `mmap()` tạo mapping mới trong virtual address space của process gọi.

```c
#include <sys/mman.h>
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t offset);
          Returns starting address of mapping on success, or MAP_FAILED on error
```

Đối số `addr` chỉ ra virtual address mà mapping được đặt. Nếu chỉ định `NULL`, kernel chọn địa chỉ phù hợp — đây là cách ưa thích. Hoặc có thể chỉ định giá trị khác `NULL`, kernel coi như gợi ý.

Khi thành công, `mmap()` trả về starting address của mapping mới. Khi lỗi, trả về `MAP_FAILED`.

Đối số `length` chỉ định kích thước mapping tính bằng byte. Kernel tạo mapping theo đơn vị page, nên `length` thực chất được làm tròn lên.

Đối số `prot` là bit mask chỉ định bảo vệ cho mapping:

**Bảng 49-2:** Giá trị bảo vệ memory

| Giá trị | Mô tả |
|---------|-------|
| `PROT_NONE` | Vùng không thể được truy cập |
| `PROT_READ` | Nội dung vùng có thể đọc |
| `PROT_WRITE` | Nội dung vùng có thể sửa đổi |
| `PROT_EXEC` | Nội dung vùng có thể thực thi |

Đối số `flags` là bit mask kiểm soát các khía cạnh khác nhau của thao tác mapping. Phải bao gồm chính xác một trong hai: `MAP_PRIVATE` hoặc `MAP_SHARED`.

Các đối số còn lại `fd` và `offset` được dùng với file mapping. `fd` là file descriptor xác định file cần map. `offset` chỉ định điểm bắt đầu mapping trong file, phải là bội số của kích thước page hệ thống.

### Bảo vệ memory chi tiết hơn

Nếu process cố truy cập vùng memory theo cách vi phạm bảo vệ, kernel gửi signal `SIGSEGV`. Có thể dùng page `PROT_NONE` làm guard page ở đầu hoặc cuối vùng bộ nhớ.

Bảo vệ memory có thể thay đổi bằng system call `mprotect()` (Mục 50.1).

**Listing 49-1:** Sử dụng `mmap()` để tạo private file mapping

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– mmap/mmcat.c
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    char *addr;
    int fd;
    struct stat sb;
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s file\n", argv[0]);
    fd = open(argv[1], O_RDONLY);
    if (fd == -1)
        errExit("open");
    if (fstat(fd, &sb) == -1)
        errExit("fstat");
    addr = mmap(NULL, sb.st_size, PROT_READ, MAP_PRIVATE, fd, 0);
    if (addr == MAP_FAILED)
        errExit("mmap");
    if (write(STDOUT_FILENO, addr, sb.st_size) != sb.st_size)
        fatal("partial/failed write");
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– mmap/mmcat.c
```

---

## 49.3 Unmap Vùng Đã Map: munmap()

System call `munmap()` thực hiện thao tác ngược của `mmap()`, xóa mapping khỏi virtual address space của process.

```c
#include <sys/mman.h>
int munmap(void *addr, size_t length);
                                             Returns 0 on success, or –1 on error
```

Đối số `addr` là starting address của address range cần unmap, phải align theo page boundary. Ví dụ:

```c
addr = mmap(NULL, length, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);
if (addr == MAP_FAILED)  errExit("mmap");
/* Code làm việc với vùng đã map */
if (munmap(addr, length) == -1)  errExit("munmap");
```

Trong quá trình unmap, kernel xóa bất kỳ memory lock nào mà process giữ cho address range được chỉ định. Tất cả mapping của process tự động được unmap khi nó kết thúc hoặc thực hiện `exec()`.

Để đảm bảo nội dung của shared file mapping được ghi vào file bên dưới, phải gọi `msync()` trước khi unmap.

---

## 49.4 File Mapping

Để tạo file mapping:
1. Lấy file descriptor cho file, thường qua `open()`.
2. Truyền file descriptor đó làm đối số `fd` trong lời gọi `mmap()`.

Sau khi gọi `mmap()`, có thể đóng file descriptor mà không ảnh hưởng đến mapping.

### 49.4.1 Private File Mapping

Hai cách dùng phổ biến nhất của private file mapping:

1. Cho phép nhiều process thực thi cùng chương trình hoặc dùng cùng shared library chia sẻ cùng text segment (read-only), được map từ phần tương ứng của file thực thi.
2. Map initialized data segment của executable hoặc shared library.

Cả hai loại mapping này thường vô hình với chương trình, vì chúng được tạo bởi program loader và dynamic linker.

### 49.4.2 Shared File Mapping

Khi nhiều process tạo shared mapping của cùng vùng file, chúng chia sẻ cùng physical page. Sửa đổi nội dung mapping được ghi vào file.

**Memory-mapped I/O**: Vì nội dung shared file mapping được khởi tạo từ file và bất kỳ sửa đổi nào được ghi lại tự động vào file, ta có thể thực hiện file I/O đơn giản bằng cách truy cập byte trong bộ nhớ. Kỹ thuật này gọi là memory-mapped I/O.

Memory-mapped I/O có thể có lợi thế hiệu năng vì:
- Loại bỏ transfer giữa kernel buffer cache và user-space buffer.
- Giảm yêu cầu bộ nhớ khi nhiều process thực hiện I/O trên cùng file.

Tuy nhiên, lợi ích hiệu năng chủ yếu khi truy cập ngẫu nhiên lặp đi lặp lại trong file lớn. Với sequential access, `mmap()` ít hoặc không có lợi ích so với `read()`/`write()`.

**IPC dùng shared file mapping**: Phân biệt loại shared memory này với System V shared memory object là sửa đổi nội dung được ghi vào file bên dưới — hữu ích khi cần nội dung shared memory tồn tại qua khởi động lại.

**Listing 49-2:** Sử dụng `mmap()` để tạo shared file mapping

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– mmap/t_mmap.c
#include <sys/mman.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
#define MEM_SIZE 10
int
main(int argc, char *argv[])
{
    char *addr;
    int fd;
    if (argc < 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s file [new-value]\n", argv[0]);
    fd = open(argv[1], O_RDWR);
    if (fd == -1)  errExit("open");
    addr = mmap(NULL, MEM_SIZE, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)  errExit("mmap");
    if (close(fd) == -1)  errExit("close");
    printf("Current string=%.*s\n", MEM_SIZE, addr);
    if (argc > 2) {
        if (strlen(argv[2]) >= MEM_SIZE)
            cmdLineErr("'new-value' too large\n");
        memset(addr, 0, MEM_SIZE);
        strncpy(addr, argv[2], MEM_SIZE - 1);
        if (msync(addr, MEM_SIZE, MS_SYNC) == -1)
            errExit("msync");
        printf("Copied \"%s\" to shared memory\n", argv[2]);
    }
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– mmap/t_mmap.c
```

### 49.4.3 Trường Hợp Biên

Nếu kích thước mapping không phải bội số của page size, nó được làm tròn lên. Các byte trong phần làm tròn nhưng ngoài file được khởi tạo về 0 và không được ghi vào file.

Cố truy cập byte sau phần làm tròn (nhưng trong mapping) tạo ra `SIGBUS`. Truy cập ngoài mapping tạo ra `SIGSEGV`.

### 49.4.4 Tương Tác Bảo Vệ Memory và File Access Mode

Nguyên tắc chung: `PROT_READ` và `PROT_EXEC` yêu cầu file được mở `O_RDONLY` hoặc `O_RDWR`; `PROT_WRITE` yêu cầu `O_WRONLY` hoặc `O_RDWR`.

---

## 49.5 Đồng Bộ Vùng Đã Map: msync()

Kernel tự động ghi sửa đổi của `MAP_SHARED` mapping vào file bên dưới, nhưng không đảm bảo khi nào. System call `msync()` cho ứng dụng kiểm soát rõ ràng khi nào shared mapping được đồng bộ với file.

```c
#include <sys/mman.h>
int msync(void *addr, size_t length, int flags);
                                             Returns 0 on success, or –1 on error
```

Đối số `flags`:

- **`MS_SYNC`**: Thực hiện synchronous file write. Lời gọi block cho đến khi tất cả page đã sửa được ghi vào đĩa.
- **`MS_ASYNC`**: Thực hiện asynchronous file write. Các page đã sửa được ghi vào đĩa tại điểm nào đó sau đó và ngay lập tức hiển thị với process khác `read()` file tương ứng.
- **`MS_INVALIDATE`**: Vô hiệu hóa các bản sao cached của dữ liệu đã map. Sau khi đồng bộ, các page không nhất quán với file được đánh dấu invalid và được refresh từ file khi truy cập tiếp.

---

## 49.6 Flag mmap() Bổ Sung

**Bảng 49-3:** Giá trị bit-mask cho đối số `flags` của `mmap()`

| Giá trị | Mô tả | SUSv3 |
|---------|-------|-------|
| `MAP_ANONYMOUS` | Tạo anonymous mapping | |
| `MAP_FIXED` | Diễn giải chính xác đối số `addr` (Mục 49.10) | ● |
| `MAP_LOCKED` | Lock các page đã map vào bộ nhớ (từ Linux 2.6) | |
| `MAP_HUGETLB` | Tạo mapping dùng huge page (từ Linux 2.6.32) | |
| `MAP_NORESERVE` | Kiểm soát đặt chỗ swap space (Mục 49.9) | |
| `MAP_PRIVATE` | Sửa đổi dữ liệu đã map là private | ● |
| `MAP_POPULATE` | Populate page của mapping (từ Linux 2.6) | |
| `MAP_SHARED` | Sửa đổi dữ liệu đã map hiển thị với process khác | ● |
| `MAP_UNINITIALIZED` | Không xóa anonymous mapping (từ Linux 2.6.33) | |

---

## 49.7 Anonymous Mapping

Anonymous mapping là mapping không có file tương ứng.

Trên Linux có hai phương pháp tương đương để tạo anonymous mapping:

1. Chỉ định `MAP_ANONYMOUS` trong `flags` và chỉ định `fd` là `-1`.
2. Mở file `/dev/zero` và truyền file descriptor kết quả cho `mmap()`. `/dev/zero` là virtual device luôn trả về zero khi đọc.

Cả hai phương pháp đều khởi tạo byte của mapping thành 0.

**MAP_PRIVATE anonymous mapping**: Dùng để cấp phát block bộ nhớ riêng tư, zero-filled cho process. glibc `malloc()` dùng chúng để cấp phát block lớn hơn `MMAP_THRESHOLD` byte (mặc định 128 kB).

**MAP_SHARED anonymous mapping**: Cho phép process liên quan (ví dụ parent và child) chia sẻ vùng bộ nhớ mà không cần file được map tương ứng. Sau `fork()`, parent và child chia sẻ cùng physical page.

**Listing 49-3:** Chia sẻ anonymous mapping giữa parent và child

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– mmap/anon_mmap.c
#ifdef USE_MAP_ANON
#define _BSD_SOURCE
#endif
#include <sys/wait.h>
#include <sys/mman.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    int *addr;
#ifdef USE_MAP_ANON
    addr = mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE,
                MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    if (addr == MAP_FAILED)  errExit("mmap");
#else
    int fd;
    fd = open("/dev/zero", O_RDWR);
    if (fd == -1)  errExit("open");
    addr = mmap(NULL, sizeof(int), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (addr == MAP_FAILED)  errExit("mmap");
    if (close(fd) == -1)  errExit("close");
#endif
    *addr = 1;
    switch (fork()) {
    case -1:
        errExit("fork");
    case 0:
        printf("Child started, value = %d\n", *addr);
        (*addr)++;
        if (munmap(addr, sizeof(int)) == -1)  errExit("munmap");
        exit(EXIT_SUCCESS);
    default:
        if (wait(NULL) == -1)  errExit("wait");
        printf("In parent, value = %d\n", *addr);
        if (munmap(addr, sizeof(int)) == -1)  errExit("munmap");
        exit(EXIT_SUCCESS);
    }
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– mmap/anon_mmap.c
```

---

## 49.8 Remapping Vùng Đã Map: mremap()

Trên hầu hết UNIX, sau khi mapping được tạo, vị trí và kích thước của nó không thể thay đổi. Linux cung cấp system call (không portable) `mremap()` cho phép thay đổi này.

```c
#define _GNU_SOURCE
#include <sys/mman.h>
void *mremap(void *old_address, size_t old_size, size_t new_size, int flags, ...);
                        Returns starting address of remapped region on success,
                                                            or MAP_FAILED on error
```

Đối số `old_address` và `old_size` chỉ định vị trí và kích thước của mapping hiện có. `new_size` chỉ định kích thước mới mong muốn.

Đối số `flags`:
- **`MREMAP_MAYMOVE`**: Kernel có thể di chuyển mapping trong virtual address space nếu cần.
- **`MREMAP_FIXED`**: Chỉ dùng kết hợp với `MREMAP_MAYMOVE`; chỉ định địa chỉ cụ thể để di chuyển mapping.

Trên Linux, `realloc()` dùng `mremap()` để reallocate block lớn được `malloc()` cấp phát qua `mmap()` `MAP_ANONYMOUS`, tránh copy byte.

---

## 49.9 MAP_NORESERVE và Swap Space Overcommitting

Một số ứng dụng tạo mapping lớn (thường private anonymous) nhưng chỉ dùng một phần nhỏ. Kernel có thể đặt chỗ swap space cho các page của mapping chỉ khi chúng thực sự cần — gọi là **lazy swap reservation**. Điều này cho phép tổng virtual memory dùng bởi ứng dụng vượt quá tổng kích thước RAM + swap space (**swap space overcommitting**).

Nếu tất cả ứng dụng đều cố dùng toàn bộ mapping, RAM và swap space sẽ cạn kiệt. Kernel giảm áp lực bộ nhớ bằng cách kill một hoặc nhiều process.

Kiểm soát đặt chỗ swap space qua:
- Flag `MAP_NORESERVE` trong lời gọi `mmap()`
- File `/proc/sys/vm/overcommit_memory` với các giá trị:
  - `0`: Từ chối overcommit rõ ràng (subject to `MAP_NORESERVE`)
  - `1`: Cho phép overcommit trong mọi trường hợp
  - `2`: Strict overcommitting — giới hạn tổng allocation theo: `[swap size] + [RAM size] * overcommit_ratio / 100`

### OOM Killer

Khi memory bị cạn kiệt, kernel giảm áp lực bằng cách kill process. Code kernel được gọi là **OOM (out-of-memory) killer**. Để kill process được chọn, OOM killer gửi signal `SIGKILL`.

File `/proc/PID/oom_score` cho thấy trọng số kernel gán cho process nếu cần gọi OOM killer. File `/proc/PID/oom_adj` có thể dùng để ảnh hưởng đến `oom_score` (giá trị từ -16 đến +15; -17 xóa process hoàn toàn khỏi danh sách ứng cử).

---

## 49.10 Flag MAP_FIXED

Chỉ định `MAP_FIXED` trong đối số `flags` của `mmap()` buộc kernel diễn giải chính xác địa chỉ trong `addr`. `addr` phải được align theo page.

Nói chung, ứng dụng portable nên bỏ qua `MAP_FIXED` và chỉ định `addr` là `NULL`. Tuy nhiên, có một trường hợp ứng dụng portable có thể dùng `MAP_FIXED`: tạo nhiều mapping liên tiếp theo cách portable.

---

## 49.11 Nonlinear Mapping: remap_file_pages()

File mapping được tạo bởi `mmap()` là linear: có sự tương ứng tuần tự giữa page của file và page của vùng bộ nhớ. Một số ứng dụng cần nonlinear mapping — nơi page của file xuất hiện theo thứ tự khác trong bộ nhớ liên tiếp.

Từ kernel 2.6, Linux cung cấp system call `remap_file_pages()` để tạo nonlinear mapping mà không tạo nhiều VMA:

```c
#define _GNU_SOURCE
#include <sys/mman.h>
int remap_file_pages(void *addr, size_t size, int prot, size_t pgoff, int flags);
                                             Returns 0 on success, or –1 on error
```

Cách dùng:
1. Tạo mapping với `mmap()`.
2. Dùng một hoặc nhiều lời gọi `remap_file_pages()` để sắp xếp lại tương ứng giữa page bộ nhớ và page file.

`remap_file_pages()` là đặc thù của Linux; không được quy định trong SUSv3.

---

## 49.12 Tóm Tắt

System call `mmap()` tạo memory mapping mới trong virtual address space của process. `munmap()` thực hiện thao tác ngược.

Mapping có thể là file-based hoặc anonymous. File mapping maps nội dung vùng file vào virtual address space. Anonymous mapping (dùng `MAP_ANONYMOUS` hoặc map `/dev/zero`) không có vùng file tương ứng; byte được khởi tạo về 0.

Mapping có thể là private (`MAP_PRIVATE`) hoặc shared (`MAP_SHARED`). Private file mapping: thay đổi không hiển thị với process khác và không được ghi vào file. Shared file mapping: thay đổi hiển thị với process khác và được ghi vào file.

Kernel tự động ghi sửa đổi `MAP_SHARED` mapping vào file, nhưng không đảm bảo khi nào. Ứng dụng có thể dùng `msync()` để kiểm soát khi nào.

Memory mapping phục vụ nhiều mục đích:
- Cấp phát bộ nhớ riêng tư cho process (private anonymous mapping)
- Khởi tạo nội dung text và data segment (private file mapping)
- Chia sẻ bộ nhớ giữa process liên quan qua `fork()` (shared anonymous mapping)
- Memory-mapped I/O, kết hợp chia sẻ bộ nhớ giữa process không liên quan (shared file mapping)

Hai signal liên quan: `SIGSEGV` khi truy cập vi phạm bảo vệ hoặc vùng chưa map; `SIGBUS` với file mapping khi truy cập vùng không có tương ứng trong file.

`mremap()` cho phép resize mapping hiện có. `remap_file_pages()` cho phép tạo nonlinear file mapping.

---

## 49.13 Bài Tập

- **49-1.** Viết chương trình tương tự `cp(1)` dùng `mmap()` và `memcpy()` để copy file.
- **49-2.** Viết lại các chương trình trong Listing 48-2 và 48-3 để dùng shared memory mapping thay vì System V shared memory.
- **49-3.** Viết chương trình xác nhận signal `SIGBUS` và `SIGSEGV` được gửi trong các trường hợp mô tả trong Mục 49.4.3.
- **49-4.** Viết chương trình dùng kỹ thuật `MAP_FIXED` để tạo nonlinear mapping.
