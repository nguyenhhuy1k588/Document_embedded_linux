## Chương 3
# **CÁC KHÁI NIỆM LẬP TRÌNH HỆ THỐNG**

Chương này đề cập đến nhiều chủ đề là điều kiện tiên quyết để lập trình hệ thống. Chúng ta bắt đầu bằng cách giới thiệu system call và mô tả chi tiết các bước xảy ra trong quá trình thực thi chúng. Sau đó chúng ta xem xét library function và cách chúng khác với system call, kết hợp với mô tả về (GNU) C library.

Mỗi khi thực hiện system call hoặc gọi library function, chúng ta nên luôn kiểm tra giá trị trả về để xác định xem lời gọi có thành công hay không. Chúng ta mô tả cách thực hiện kiểm tra như vậy, và trình bày một tập hàm được dùng trong hầu hết các chương trình ví dụ trong cuốn sách này để chẩn đoán lỗi từ system call và library function.

## **3.1 System Call**

Một system call là một điểm vào có kiểm soát vào kernel, cho phép process yêu cầu kernel thực hiện một số hành động thay mặt cho process. Kernel cung cấp một loạt dịch vụ có thể truy cập cho chương trình thông qua system call application programming interface (API). Các dịch vụ này bao gồm, ví dụ, tạo process mới, thực hiện I/O và tạo pipe cho interprocess communication.

Trước khi đi vào chi tiết về cách system call hoạt động, chúng ta lưu ý một số điểm chung:

- Một system call thay đổi trạng thái bộ xử lý từ user mode sang kernel mode, để CPU có thể truy cập protected kernel memory.
- Tập hợp các system call là cố định. Mỗi system call được nhận dạng bằng một số duy nhất.
- Mỗi system call có thể có một tập argument chỉ định thông tin cần truyền từ user space đến kernel space và ngược lại.

Từ góc độ lập trình, việc gọi system call trông giống như gọi hàm C. Tuy nhiên, đằng sau đó, nhiều bước xảy ra trong quá trình thực thi system call. Để minh họa điều này, chúng ta xem xét các bước theo thứ tự chúng xảy ra trên một phiên bản cài đặt phần cứng cụ thể, x86-32:

1. Chương trình ứng dụng thực hiện system call bằng cách gọi một wrapper function trong thư viện C.
2. Wrapper function phải cung cấp tất cả argument của system call cho routine xử lý system call trap. Wrapper function sao chép các argument vào các thanh ghi cụ thể.
3. Vì tất cả system call đều vào kernel theo cùng một cách, kernel cần một phương pháp nhận dạng system call. Wrapper function sao chép system call number vào một thanh ghi CPU cụ thể (`%eax`).
4. Wrapper function thực thi một lệnh trap (`int 0x80`), khiến bộ xử lý chuyển từ user mode sang kernel mode và thực thi code được trỏ đến bởi vị trí 0x80 của trap vector của hệ thống.
5. Để phản hồi trap, kernel gọi routine `system_call()` để xử lý trap. Handler này:
   a) Lưu giá trị thanh ghi vào kernel stack.
   b) Kiểm tra tính hợp lệ của system call number.
   c) Gọi service routine system call thích hợp.
   d) Khôi phục giá trị thanh ghi từ kernel stack và đặt giá trị trả về của system call lên stack.
   e) Trả về wrapper function, đồng thời đưa bộ xử lý về user mode.
6. Nếu giá trị trả về của service routine system call chỉ ra lỗi, wrapper function đặt biến toàn cục `errno` bằng giá trị này. Wrapper function sau đó trả về caller, cung cấp giá trị trả về integer chỉ ra thành công hay thất bại.

```
+--------------------------- USER MODE -----------------------------------+
|      Chương trình ứng dụng         glibc wrapper function               |
|                                                                         |
|    +----------------------+        +----------------------------------+ |
|    |   ...                |        |                                  | |
|    |   execve(path, --------> execve(path, argv, envp)                | |
|    |          argv, envp) |        |   {   ...                        | |
|    |   ... <--------------         |       int 0x80  ------> kernel   | |
|    |                      |        |   }                              | |
|    +----------------------+        +----------------------------------+ |
|_________________________________________________________________________|

+--------------------------- KERNEL MODE ---------------------------------+
|   System call service routine         Trap handler                      |
|                                                                         |
|    +----------------------+        +----------------------------------+ |
|    |                      |        |   system_call:                   | |
|    |   sys_execve() <------------- |   call sys_call_table            | |
|    |   {                  |        |            [__NR_execve]         | |
|    |   ...                |        |                                  | |
|    |   return error; ----->         |                                  | |
|    |   }                  |        |                                  | |
|    +----------------------+        +----------------------------------+ |
|_________________________________________________________________________|
```

**Hình 3-1:** Các bước trong quá trình thực thi system call

Thông tin nêu trên minh họa điểm quan trọng là ngay cả đối với một system call đơn giản, cũng phải thực hiện khá nhiều công việc, và do đó system call có một overhead nhỏ nhưng đáng kể.

## **3.2 Library Function**

Một library function đơn giản là một trong số nhiều hàm cấu thành thư viện C chuẩn. Mục đích của các hàm này rất đa dạng, bao gồm các tác vụ như mở file, chuyển đổi thời gian sang định dạng có thể đọc được và so sánh hai chuỗi ký tự.

Nhiều library function không sử dụng bất kỳ system call nào (ví dụ: các hàm thao tác chuỗi). Mặt khác, một số library function được xây dựng trên các system call. Ví dụ, library function `fopen()` sử dụng system call `open()` để thực sự mở file. Thường thì library function được thiết kế để cung cấp interface thân thiện hơn với caller so với system call bên dưới. Ví dụ, hàm `printf()` cung cấp output formatting và data buffering, trong khi system call `write()` chỉ xuất một khối byte.

# **3.3 Thư Viện C Chuẩn; GNU C Library (glibc)**

Có các phiên bản cài đặt khác nhau của thư viện C chuẩn trên các phiên bản UNIX khác nhau. Phiên bản cài đặt được dùng phổ biến nhất trên Linux là GNU C library (glibc).

## **Xác định phiên bản glibc trên hệ thống**

Đôi khi, chúng ta cần xác định phiên bản glibc trên một hệ thống. Từ shell, chúng ta có thể làm điều này bằng cách chạy file shared library của glibc như thể nó là một chương trình executable:

```
$ /lib/libc.so.6
GNU C Library stable release version 2.10.1, by Roland McGrath et al.
...
```

Có hai phương tiện để chương trình ứng dụng xác định phiên bản GNU C library hiện có trên hệ thống: kiểm tra các hằng số hoặc gọi library function. Từ version 2.0 trở đi, glibc định nghĩa hai hằng số, `__GLIBC__` và `__GLIBC_MINOR__`, có thể được kiểm tra tại compile time. Một chương trình cũng có thể gọi hàm `gnu_get_libc_version()` để xác định phiên bản glibc có sẵn tại runtime.

```c
#include <gnu/libc-version.h>
const char *gnu_get_libc_version(void);
```

# **3.4 Xử Lý Lỗi Từ System Call và Library Function**

Hầu hết mọi system call và library function đều trả về một loại giá trị status nào đó cho biết lời gọi có thành công hay thất bại. Giá trị status này nên luôn được kiểm tra để xem lời gọi có thành công hay không.

#### **Xử lý lỗi system call**

Trang manual cho mỗi system call ghi lại các giá trị trả về có thể của lời gọi, cho thấy giá trị nào chỉ ra lỗi. Thông thường, lỗi được chỉ ra bằng giá trị trả về là –1. Vì vậy, một system call có thể được kiểm tra với code như sau:

```c
fd = open(pathname, flags, mode); /* system call để mở file */
if (fd == -1) {
    /* Code để xử lý lỗi */
}
```

Khi một system call thất bại, nó đặt biến integer toàn cục `errno` thành một giá trị dương nhận dạng lỗi cụ thể. Bao gồm header file `<errno.h>` cung cấp khai báo của `errno`, cũng như một tập hằng số cho các error number khác nhau. Tất cả các tên tượng trưng này bắt đầu bằng `E`. Đây là một ví dụ đơn giản về việc sử dụng `errno` để chẩn đoán lỗi system call:

```c
cnt = read(fd, buf, numbytes);
if (cnt == -1) {
    if (errno == EINTR)
        fprintf(stderr, "read bị gián đoạn bởi signal\n");
    else {
        /* Đã xảy ra lỗi khác */
    }
}
```

Các system call và library function thành công không bao giờ reset `errno` về 0, vì vậy biến này có thể có giá trị khác không do lỗi từ lần gọi trước. Do đó, khi kiểm tra lỗi, chúng ta nên luôn kiểm tra giá trị trả về của hàm trước, rồi mới kiểm tra `errno` để xác định nguyên nhân lỗi.

Một hành động phổ biến sau khi system call thất bại là in ra thông báo lỗi dựa trên giá trị `errno`. Các library function `perror()` và `strerror()` được cung cấp cho mục đích này.

Hàm `perror()` in chuỗi được trỏ đến bởi argument `msg` của nó, theo sau là thông báo tương ứng với giá trị hiện tại của `errno`.

```c
#include <stdio.h>
void perror(const char *msg);
```

Một cách đơn giản để xử lý lỗi từ system call là:

```c
fd = open(pathname, flags, mode);
if (fd == -1) {
    perror("open");
    exit(EXIT_FAILURE);
}
```

Hàm `strerror()` trả về chuỗi lỗi tương ứng với error number được cung cấp trong argument `errnum` của nó.

```c
#include <string.h>
char *strerror(int errnum);
```

## **Xử lý lỗi từ library function**

Các library function khác nhau trả về các kiểu dữ liệu và giá trị khác nhau để chỉ ra thất bại. Đối với mục đích của chúng ta, library function có thể được chia thành các loại sau:

- Một số library function trả về thông tin lỗi theo cùng cách như system call: giá trị trả về –1, với `errno` chỉ ra lỗi cụ thể.
- Một số library function trả về giá trị khác –1 khi có lỗi, nhưng vẫn đặt `errno` để chỉ ra điều kiện lỗi cụ thể. Ví dụ, `fopen()` trả về con trỏ NULL khi có lỗi.
- Các library function khác không sử dụng `errno` chút nào. Phương pháp xác định sự tồn tại và nguyên nhân lỗi phụ thuộc vào hàm cụ thể.

# **3.5 Ghi Chú Về Các Chương Trình Ví Dụ Trong Cuốn Sách**

## **3.5.1 Command-Line Option và Argument**

Nhiều chương trình ví dụ trong cuốn sách này dựa vào command-line option và argument để xác định hành vi của chúng. Để phân tích các option này, chúng ta sử dụng library function chuẩn `getopt()`.

## **3.5.2 Các Hàm Chung và Header File**

### **Header file chung**

Listing 3-1 là header file được hầu hết mọi chương trình trong cuốn sách này sử dụng. Header file này bao gồm nhiều header file khác được sử dụng bởi nhiều chương trình ví dụ, định nghĩa kiểu dữ liệu Boolean và định nghĩa macro để tính giá trị nhỏ nhất và lớn nhất của hai giá trị số.

```c
/* lib/tlpi_hdr.h */
#ifndef TLPI_HDR_H
#define TLPI_HDR_H
#include <sys/types.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include "get_num.h"
#include "error_functions.h"
typedef enum { FALSE, TRUE } Boolean;
#define min(m,n) ((m) < (n) ? (m) : (n))
#define max(m,n) ((m) > (n) ? (m) : (n))
#endif
```

### **Các hàm chẩn đoán lỗi**

Để đơn giản hóa xử lý lỗi trong các chương trình ví dụ của chúng ta, chúng ta sử dụng các hàm chẩn đoán lỗi sau:

```c
#include "tlpi_hdr.h"
void errMsg(const char *format, ...);
void errExit(const char *format, ...);
void err_exit(const char *format, ...);
void errExitEN(int errnum, const char *format, ...);
```

Hàm `errMsg()` in một thông báo trên standard error. Nó in văn bản lỗi tương ứng với giá trị hiện tại của `errno` — bao gồm tên lỗi, như EPERM, cộng mô tả lỗi như được trả về bởi `strerror()` — theo sau là đầu ra có định dạng được chỉ định trong danh sách argument.

Hàm `errExit()` hoạt động như `errMsg()`, nhưng cũng kết thúc chương trình bằng cách gọi `exit()` hoặc, nếu biến môi trường `EF_DUMPCORE` được định nghĩa, bằng cách gọi `abort()` để tạo core dump file.

Hàm `err_exit()` tương tự như `errExit()`, nhưng khác ở hai điểm:
- Nó không flush standard output trước khi in thông báo lỗi.
- Nó kết thúc process bằng cách gọi `_exit()` thay vì `exit()`.

Hàm `errExitEN()` giống như `errExit()`, ngoại trừ thay vì in văn bản lỗi tương ứng với giá trị hiện tại của `errno`, nó in văn bản tương ứng với error number được cung cấp trong argument `errnum`.

Để chẩn đoán các loại lỗi khác, chúng ta sử dụng `fatal()`, `usageErr()` và `cmdLineErr()`.

Hàm `fatal()` được dùng để chẩn đoán các lỗi chung, bao gồm lỗi từ library function không đặt `errno`.

Hàm `usageErr()` được dùng để chẩn đoán lỗi trong cách sử dụng command-line argument. Nó in chuỗi "Usage:" theo sau là đầu ra có định dạng trên standard error, rồi kết thúc chương trình.

# **3.6 Các Vấn Đề Về Tính Di Động**

## **3.6.1 Feature Test Macro**

Các tiêu chuẩn khác nhau chi phối hành vi của system call và library function API. Đôi khi, khi viết ứng dụng di động, chúng ta có thể muốn các header file khác nhau chỉ expose các định nghĩa tuân theo một tiêu chuẩn cụ thể. Để làm điều này, chúng ta định nghĩa một hoặc nhiều feature test macro khi biên dịch chương trình:

```c
#define _BSD_SOURCE 1
```

Hoặc sử dụng tùy chọn `-D` của compiler C:

```
$ cc -D_BSD_SOURCE prog.c
```

Các feature test macro sau được chỉ định bởi các tiêu chuẩn liên quan:

**`_POSIX_SOURCE`**: Nếu được định nghĩa, expose các định nghĩa tuân theo POSIX.1-1990 và ISO C (1990).

**`_POSIX_C_SOURCE`**: Kiểm soát mức độ tuân thủ POSIX.1. Nếu được định nghĩa với giá trị 200112, expose các định nghĩa cho base specification POSIX.1-2001. Nếu được định nghĩa với giá trị 200809, expose các định nghĩa cho base specification POSIX.1-2008.

**`_XOPEN_SOURCE`**: Nếu được định nghĩa với giá trị 600 hoặc lớn hơn, expose các extension SUSv3 XSI (UNIX 03). Nếu được định nghĩa với giá trị 700 hoặc lớn hơn, expose các extension SUSv4 XSI.

Các feature test macro dành riêng cho glibc:

**`_BSD_SOURCE`**: Expose các định nghĩa BSD.

**`_SVID_SOURCE`**: Expose các định nghĩa System V Interface Definition (SVID).

**`_GNU_SOURCE`**: Expose tất cả các định nghĩa cùng với nhiều GNU extension.

## **3.6.2 Các Kiểu Dữ Liệu Hệ Thống**

Nhiều kiểu dữ liệu cài đặt được biểu diễn bằng các kiểu C chuẩn, ví dụ, process ID, user ID và file offset. Để tránh các vấn đề về tính di động, SUSv3 chỉ định nhiều kiểu dữ liệu hệ thống chuẩn và yêu cầu phiên bản cài đặt định nghĩa và sử dụng các kiểu này một cách thích hợp. Mỗi kiểu này được định nghĩa bằng tính năng `typedef` của C.

Bảng 3-1 liệt kê một số kiểu dữ liệu hệ thống mà chúng ta sẽ gặp trong cuốn sách này:

**Bảng 3-1:** Các kiểu dữ liệu hệ thống được chọn lọc

| Kiểu dữ liệu | Yêu cầu kiểu SUSv3 | Mô tả |
|--------------|---------------------|-------|
| `blkcnt_t` | signed integer | Số block của file (Mục 15.1) |
| `clock_t` | integer hoặc real-floating | Thời gian hệ thống tính bằng clock tick (Mục 10.7) |
| `dev_t` | arithmetic type | Số thiết bị, bao gồm số major và minor (Mục 15.1) |
| `gid_t` | integer | Numeric group identifier (Mục 8.3) |
| `ino_t` | unsigned integer | File i-node number (Mục 15.1) |
| `mode_t` | integer | File permission và type (Mục 15.1) |
| `nlink_t` | integer | Số (hard) link đến file (Mục 15.1) |
| `off_t` | signed integer | File offset hoặc kích thước (Mục 4.7 và 15.1) |
| `pid_t` | signed integer | Process ID, process group ID hoặc session ID |
| `rlim_t` | unsigned integer | Resource limit (Mục 36.2) |
| `sig_atomic_t` | integer | Kiểu dữ liệu có thể được truy cập atomically (Mục 21.1.3) |
| `sigset_t` | integer hoặc structure type | Signal set (Mục 20.9) |
| `size_t` | unsigned integer | Kích thước của object tính bằng byte |
| `ssize_t` | signed integer | Số byte hoặc chỉ báo lỗi (âm) |
| `time_t` | integer hoặc real-floating | Calendar time tính bằng giây kể từ Epoch (Mục 10.1) |
| `uid_t` | integer | Numeric user identifier (Mục 8.1) |

### **In giá trị kiểu dữ liệu hệ thống**

Khi in các giá trị của một trong các kiểu dữ liệu hệ thống số, chúng ta phải cẩn thận không đưa vào sự phụ thuộc biểu diễn trong lời gọi `printf()`. Giải pháp thông thường là sử dụng specifier `%ld` và luôn cast giá trị tương ứng sang `long`:

```c
pid_t mypid;
mypid = getpid();
printf("My PID is %ld\n", (long) mypid);
```

## **3.6.3 Các Vấn Đề Tính Di Động Khác**

### **Khởi tạo và sử dụng structure**

Thứ tự định nghĩa các field trong các structure chuẩn không được SUSv3 chỉ định. Do đó, không nên sử dụng structure initializer như sau:

```c
struct sembuf s = { 3, -1, SEM_UNDO };
```

Thay vào đó, sử dụng các câu lệnh gán tường minh:

```c
struct sembuf s;
s.sem_num = 3;
s.sem_op = -1;
s.sem_flg = SEM_UNDO;
```

Nếu sử dụng C99, có thể dùng cú pháp khởi tạo structure mới:

```c
struct sembuf s = { .sem_num = 3, .sem_op = -1, .sem_flg = SEM_UNDO };
```

# **3.7 Tóm Tắt**

System call cho phép các process yêu cầu dịch vụ từ kernel. Ngay cả system call đơn giản nhất cũng có overhead đáng kể so với lời gọi hàm trong user space, vì hệ thống phải tạm thời chuyển sang kernel mode để thực thi system call, và kernel phải xác thực argument và truyền dữ liệu giữa user memory và kernel memory.

Thư viện C chuẩn cung cấp nhiều library function thực hiện nhiều tác vụ. Một số library function sử dụng system call để thực hiện công việc của mình; các hàm khác thực hiện tác vụ hoàn toàn trong user space. Trên Linux, phiên bản cài đặt thư viện C chuẩn thường được dùng là glibc.

Hầu hết system call và library function trả về trạng thái chỉ ra xem lời gọi có thành công hay thất bại. Các giá trị trả về như vậy nên luôn được kiểm tra.

Khi biên dịch ứng dụng, chúng ta có thể định nghĩa nhiều feature test macro khác nhau để kiểm soát các định nghĩa được expose bởi header file. Điều này hữu ích khi chúng ta muốn đảm bảo chương trình tuân thủ một tiêu chuẩn chính thức hoặc tiêu chuẩn do phiên bản cài đặt định nghĩa.

## **3.8 Bài Tập**

**3-1.** Khi sử dụng system call `reboot()` dành riêng cho Linux để khởi động lại hệ thống, argument thứ hai, `magic2`, phải được chỉ định là một trong một tập số ma thuật (ví dụ: `LINUX_REBOOT_MAGIC2`). Ý nghĩa của các số này là gì? (Chuyển đổi chúng sang thập lục phân sẽ cung cấp gợi ý.)
