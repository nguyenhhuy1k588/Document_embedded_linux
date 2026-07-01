## Chương 28
# Tạo Process và Thực Thi Chương Trình Chi Tiết Hơn

Chương này mở rộng tài liệu được trình bày trong các Chương 24 đến 27 bằng cách đề cập đến nhiều chủ đề liên quan đến việc tạo process và thực thi chương trình. Chúng ta mô tả process accounting, một tính năng kernel ghi lại một bản ghi kế toán cho mỗi process trên hệ thống khi nó kết thúc. Sau đó chúng ta xem xét system call `clone()` đặc thù của Linux, là API cấp thấp được sử dụng để tạo thread trên Linux. Tiếp theo, chúng ta so sánh hiệu suất của `fork()`, `vfork()`, và `clone()`. Chúng ta kết thúc với tóm tắt về tác động của `fork()` và `exec()` trên các thuộc tính của một process.

## **28.1 Process Accounting**

Khi process accounting được bật, kernel ghi một bản ghi kế toán vào file process accounting trên toàn hệ thống khi mỗi process kết thúc. Bản ghi kế toán này chứa thông tin khác nhau do kernel duy trì về process, bao gồm trạng thái kết thúc của nó và lượng thời gian CPU mà nó đã tiêu thụ. File kế toán có thể được phân tích bởi các công cụ chuẩn (`sa(8)` tóm tắt thông tin từ file kế toán, và `lastcomm(1)` liệt kê thông tin về các lệnh đã thực thi trước đó) hoặc bởi các ứng dụng tùy chỉnh.

Trong các kernel trước 2.6.10, một bản ghi process accounting riêng biệt được viết cho mỗi thread được tạo bằng cách sử dụng cài đặt NPTL threading. Kể từ kernel 2.6.10, một bản ghi kế toán duy nhất được viết cho toàn bộ process khi thread cuối cùng kết thúc. Dưới cài đặt threading LinuxThreads cũ hơn, một bản ghi process accounting duy nhất luôn được viết cho mỗi thread.

Trong lịch sử, cách sử dụng chính của process accounting là tính phí người dùng cho việc tiêu thụ tài nguyên hệ thống trên các hệ thống UNIX nhiều người dùng. Tuy nhiên, process accounting cũng có thể hữu ích để lấy thông tin về một process mà không được giám sát và báo cáo bởi process cha của nó.

Mặc dù có sẵn trên hầu hết các cài đặt UNIX, process accounting không được chỉ định trong SUSv3. Định dạng của các bản ghi kế toán, cũng như vị trí của file kế toán, thay đổi phần nào giữa các cài đặt.

> Trên Linux, process accounting là một thành phần kernel tùy chọn được cấu hình qua tùy chọn `CONFIG_BSD_PROCESS_ACCT`.

### **Bật và tắt process accounting**

System call `acct()` được sử dụng bởi một process có đặc quyền (`CAP_SYS_PACCT`) để bật và tắt process accounting. System call này hiếm khi được sử dụng trong các chương trình ứng dụng. Thông thường, process accounting được bật mỗi khi khởi động lại hệ thống bằng cách đặt các lệnh thích hợp trong các script khởi động hệ thống.

```c
#define _BSD_SOURCE
#include <unistd.h>
int acct(const char *acctfile);
                                              Trả về 0 khi thành công, hoặc –1 khi có lỗi
```

Để bật process accounting, chúng ta cung cấp đường dẫn của một file thông thường hiện có trong `acctfile`. Một đường dẫn thông thường cho file kế toán là `/var/log/pacct` hoặc `/usr/account/pacct`. Để tắt process accounting, chúng ta chỉ định `acctfile` là NULL.

Chương trình trong Listing 28-1 sử dụng `acct()` để bật và tắt process accounting. Chức năng của chương trình này tương tự như lệnh shell `accton(8)`.

**Listing 28-1:** Bật và tắt process accounting

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/acct_on.c
#define _BSD_SOURCE
#include <unistd.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    if (argc > 2 || (argc > 1 && strcmp(argv[1], "--help") == 0))
        usageErr("%s [file]\n");
    if (acct(argv[1]) == -1)
        errExit("acct");
    printf("Process accounting %s\n",
           (argv[1] == NULL) ? "disabled" : "enabled");
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/acct_on.c
```

#### **Các bản ghi process accounting**

Khi process accounting được bật, một bản ghi `acct` được viết vào file kế toán khi mỗi process kết thúc. Cấu trúc `acct` được định nghĩa trong `<sys/acct.h>` như sau:

```c
typedef u_int16_t comp_t; /* Xem phần chính */
struct acct {
    char ac_flag;          /* Cờ kế toán (xem phần chính) */
    u_int16_t ac_uid;      /* User ID của process */
    u_int16_t ac_gid;      /* Group ID của process */
    u_int16_t ac_tty;      /* Controlling terminal cho process (có thể
                              là 0 nếu không có, ví dụ: cho một daemon) */
    u_int32_t ac_btime;    /* Thời gian bắt đầu (time_t; giây kể từ Epoch) */
    comp_t ac_utime;       /* Thời gian CPU người dùng (clock ticks) */
    comp_t ac_stime;       /* Thời gian CPU hệ thống (clock ticks) */
    comp_t ac_etime;       /* Thời gian đã trôi qua (real) (clock ticks) */
    comp_t ac_mem;         /* Mức sử dụng bộ nhớ trung bình (kilobytes) */
    comp_t ac_io;          /* Số bytes được truyền bởi read(2) và write(2)
                              (không sử dụng) */
    comp_t ac_rw;          /* Khối đọc/ghi (không sử dụng) */
    comp_t ac_minflt;      /* Minor page fault (đặc thù Linux) */
    comp_t ac_majflt;      /* Major page fault (đặc thù Linux) */
    comp_t ac_swaps;       /* Số lần swap (không sử dụng; đặc thù Linux) */
    u_int32_t ac_exitcode; /* Trạng thái kết thúc process */
#define ACCT_COMM 16
    char ac_comm[ACCT_COMM+1];
    /* Tên lệnh (kết thúc null) (basename của file exec cuối cùng) */
    char ac_pad[10]; /* Padding (dành cho sử dụng trong tương lai) */
};
```

Lưu ý các điểm sau liên quan đến cấu trúc `acct`:

- Các kiểu dữ liệu `u_int16_t` và `u_int32_t` là các số nguyên không dấu 16-bit và 32-bit.
- Trường `ac_flag` là một bit mask ghi lại các sự kiện khác nhau cho process. Các bit có thể xuất hiện trong trường này được hiển thị trong Bảng 28-1. Như được chỉ ra trong bảng, một số bit này không có trên tất cả các cài đặt UNIX.
- Trường `ac_comm` ghi lại tên của lệnh (file chương trình) cuối cùng được thực thi bởi process này. Kernel ghi giá trị này trên mỗi `execve()`. Trên một số cài đặt UNIX khác, trường này bị giới hạn ở 8 ký tự.
- Kiểu `comp_t` là một loại số dấu phẩy động. Các giá trị của kiểu này đôi khi được gọi là *compressed clock ticks*. Giá trị dấu phẩy động bao gồm một số mũ 3-bit, cơ số 8, theo sau là một phần mantissa 13-bit.
- Ba trường thời gian được định nghĩa với kiểu `comp_t` đại diện cho thời gian trong clock ticks hệ thống. Do đó, chúng ta phải chia các thời gian này cho giá trị được trả về bởi `sysconf(_SC_CLK_TCK)` để chuyển đổi chúng thành giây.
- Trường `ac_exitcode` giữ trạng thái kết thúc của process (được mô tả trong Mục 26.1.3).

**Bảng 28-1:** Các giá trị bit cho trường `ac_flag` của các bản ghi process accounting

| Bit   | Mô tả                                                                                  |
|-------|----------------------------------------------------------------------------------------|
| AFORK | Process được tạo bởi `fork()`, nhưng không thực hiện `exec()` trước khi kết thúc      |
| ASU   | Process đã sử dụng đặc quyền superuser                                                |
| AXSIG | Process bị kết thúc bởi một signal (không có trên một số cài đặt)                     |
| ACORE | Process tạo ra một core dump (không có trên một số cài đặt)                           |

Vì các bản ghi kế toán chỉ được viết khi các process kết thúc, chúng được sắp xếp theo thời gian kết thúc (một giá trị không được ghi trong bản ghi), thay vì theo thời gian bắt đầu process (`ac_btime`).

Nếu hệ thống gặp sự cố, không có bản ghi kế toán nào được viết cho bất kỳ process nào đang thực thi.

Vì việc viết bản ghi vào file kế toán có thể nhanh chóng tiêu thụ không gian đĩa, Linux cung cấp file ảo `/proc/sys/kernel/acct` để kiểm soát hoạt động của process accounting. File này chứa ba số, xác định (theo thứ tự) các tham số high-water, low-water, và frequency.

#### **Chương trình ví dụ**

Chương trình trong Listing 28-2 hiển thị các trường đã chọn từ các bản ghi trong một file process accounting. Phiên shell sau đây minh họa việc sử dụng chương trình này. Chúng ta bắt đầu bằng cách tạo một file process accounting mới, trống và bật process accounting:

```
$ su    Cần đặc quyền để bật process accounting
Password:
# touch pacct
# ./acct_on pacct    Process này sẽ là entry đầu tiên trong file kế toán
Process accounting enabled
# exit    Thôi là superuser
```

Bây giờ chúng ta chạy một loạt lệnh để thêm bản ghi vào file kế toán:

```
$ sleep 15 &
[1] 18063
$ ulimit -c unlimited    Cho phép core dump (shell built-in)
$ cat    Tạo một process
Nhấn Control-\ (tạo SIGQUIT, signal 3) để kill process cat
Quit (core dumped)
$
Nhấn Enter để thấy thông báo shell về việc hoàn thành sleep trước lệnh nhắc shell tiếp theo
[1]+ Done sleep 15
$ grep xxx badfile    grep thất bại với trạng thái 2
grep: badfile: No such file or directory
$ echo $?    Shell lấy trạng thái của grep (shell built-in)
2
```

Tiếp theo, chúng ta sử dụng chương trình trong Listing 28-2 để xem nội dung của file kế toán:

#### $ **./acct_view pacct**

| command | flags |  | term. | user | start time          | CPU  | elapsed |
|---------|-------|--|-------|------|---------------------|------|---------|
|         |       |  | status |      |                     | time | time    |
| acct_on | -S    |  | 0     | root | 2010-07-23 17:19:05 | 0.00 | 0.00    |
| bash    |       |  | 0     | root | 2010-07-23 17:18:55 | 0.02 | 21.10   |
| su      | -S    |  | 0     | root | 2010-07-23 17:18:51 | 0.01 | 24.94   |
| cat     | XC    |  | 0x83  | mtk  | 2010-07-23 17:19:55 | 0.00 | 1.72    |
| sleep   |       |  | 0     | mtk  | 2010-07-23 17:19:42 | 0.00 | 15.01   |
| grep    |       |  | 0x200 | mtk  | 2010-07-23 17:20:12 | 0.00 | 0.00    |
| echo    |       |  | 0     | mtk  | 2010-07-23 17:21:15 | 0.01 | 0.01    |
| t_fork  | F     |  | 0     | mtk  | 2010-07-23 17:21:36 | 0.00 | 0.00    |
| t_fork  |       |  | 0     | mtk  | 2010-07-23 17:21:36 | 0.00 | 3.01    |

Trong output, chúng ta thấy một dòng cho mỗi process được tạo trong phiên shell. Cột flags hiển thị các chữ cái đơn cho biết bit `ac_flag` nào được đặt trong mỗi bản ghi (xem Bảng 28-1).

**Listing 28-2:** Hiển thị dữ liệu từ file process accounting

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/acct_view.c
#include <fcntl.h>
#include <time.h>
#include <sys/stat.h>
#include <sys/acct.h>
#include <limits.h>
#include "ugid_functions.h" /* Khai báo của userNameFromId() */
#include "tlpi_hdr.h"
#define TIME_BUF_SIZE 100
static long long /* Chuyển đổi giá trị comp_t thành long long */
comptToLL(comp_t ct)
{
    const int EXP_SIZE = 3;   /* Số mũ 3-bit, cơ số 8 */
    const int MANTISSA_SIZE = 13; /* Tiếp theo là mantissa 13-bit */
    const int MANTISSA_MASK = (1 << MANTISSA_SIZE) - 1;
    long long mantissa, exp;
    mantissa = ct & MANTISSA_MASK;
    exp = (ct >> MANTISSA_SIZE) & ((1 << EXP_SIZE) - 1);
    return mantissa << (exp * 3); /* Lũy thừa 8 = dịch trái 3 bit */
}
int
main(int argc, char *argv[])
{
    int acctFile;
    struct acct ac;
    ssize_t numRead;
    char *s;
    char timeBuf[TIME_BUF_SIZE];
    struct tm *loc;
    time_t t;
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s file\n", argv[0]);
    acctFile = open(argv[1], O_RDONLY);
    if (acctFile == -1)
        errExit("open");
    printf("command flags term. user "
           "start time CPU elapsed\n");
    printf("              status "
           "          time time\n");
    while ((numRead = read(acctFile, &ac, sizeof(struct acct))) > 0) {
        if (numRead != sizeof(struct acct))
            fatal("partial read");
        printf("%-8.8s ", ac.ac_comm);
        printf("%c", (ac.ac_flag & AFORK) ? 'F' : '-') ;
        printf("%c", (ac.ac_flag & ASU)   ? 'S' : '-') ;
        printf("%c", (ac.ac_flag & AXSIG) ? 'X' : '-') ;
        printf("%c", (ac.ac_flag & ACORE) ? 'C' : '-') ;
#ifdef __linux__
        printf(" %#6lx ", (unsigned long) ac.ac_exitcode);
#else
        printf(" %#6lx ", (unsigned long) ac.ac_stat);
#endif
        s = userNameFromId(ac.ac_uid);
        printf("%-8.8s ", (s == NULL) ? "???" : s);
        t = ac.ac_btime;
        loc = localtime(&t);
        if (loc == NULL) {
            printf("???Unknown time??? ");
        } else {
            strftime(timeBuf, TIME_BUF_SIZE, "%Y-%m-%d %T ", loc);
            printf("%s ", timeBuf);
        }
        printf("%5.2f %7.2f ", (double) (comptToLL(ac.ac_utime) +
               comptToLL(ac.ac_stime)) / sysconf(_SC_CLK_TCK),
               (double) comptToLL(ac.ac_etime) / sysconf(_SC_CLK_TCK));
        printf("\n");
    }
    if (numRead == -1)
        errExit("read");
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/acct_view.c
```

## **Định dạng file Process accounting Version 3**

Bắt đầu từ kernel 2.6.8, Linux đã giới thiệu một phiên bản thay thế tùy chọn của file process accounting giải quyết một số hạn chế của file kế toán truyền thống. Để sử dụng phiên bản thay thế này, được gọi là Version 3, tùy chọn cấu hình kernel `CONFIG_BSD_PROCESS_ACCT_V3` phải được bật trước khi xây dựng kernel.

Khi sử dụng tùy chọn Version 3, sự khác biệt duy nhất trong hoạt động của process accounting là trong định dạng của các bản ghi được viết vào file kế toán. Định dạng mới được định nghĩa như sau:

```c
struct acct_v3 {
    char ac_flag;           /* Cờ kế toán */
    char ac_version;        /* Phiên bản kế toán (3) */
    u_int16_t ac_tty;       /* Controlling terminal cho process */
    u_int32_t ac_exitcode;  /* Trạng thái kết thúc process */
    u_int32_t ac_uid;       /* User ID 32-bit của process */
    u_int32_t ac_gid;       /* Group ID 32-bit của process */
    u_int32_t ac_pid;       /* Process ID */
    u_int32_t ac_ppid;      /* Parent process ID */
    u_int32_t ac_btime;     /* Thời gian bắt đầu (time_t) */
    float ac_etime;         /* Thời gian đã trôi qua (real) (clock ticks) */
    comp_t ac_utime;        /* Thời gian CPU người dùng (clock ticks) */
    comp_t ac_stime;        /* Thời gian CPU hệ thống (clock ticks) */
    comp_t ac_mem;          /* Mức sử dụng bộ nhớ trung bình (kilobytes) */
    comp_t ac_io;           /* Bytes đọc/ghi (không sử dụng) */
    comp_t ac_rw;           /* Khối đọc/ghi (không sử dụng) */
    comp_t ac_minflt;       /* Minor page fault */
    comp_t ac_majflt;       /* Major page fault */
    comp_t ac_swaps;        /* Số lần swap (không sử dụng; đặc thù Linux) */
#define ACCT_COMM 16
    char ac_comm[ACCT_COMM]; /* Tên lệnh */
};
```

Những sự khác biệt chính giữa cấu trúc `acct_v3` và cấu trúc `acct` Linux truyền thống:

- Trường `ac_version` được thêm vào. Trường này chứa số phiên bản của kiểu bản ghi kế toán này.
- Các trường `ac_pid` và `ac_ppid`, chứa process ID và parent process ID của process đã kết thúc, được thêm vào.
- Các trường `ac_uid` và `ac_gid` được mở rộng từ 16 lên 32 bit, để phù hợp với user và group ID 32-bit được giới thiệu trong Linux 2.4.
- Kiểu của trường `ac_etime` được thay đổi từ `comp_t` thành `float`, để cho phép ghi lại thời gian đã trôi qua dài hơn.

## **28.2 System Call clone()**

Giống như `fork()` và `vfork()`, system call `clone()` đặc thù của Linux tạo ra một process mới. Nó khác với hai lời gọi kia ở chỗ cho phép kiểm soát tốt hơn các bước xảy ra trong quá trình tạo process. Cách sử dụng chính của `clone()` là trong việc triển khai các thư viện threading. Vì `clone()` không di động, việc sử dụng trực tiếp của nó trong các chương trình ứng dụng thường nên tránh. Chúng ta mô tả nó ở đây vì đây là nền tảng hữu ích cho việc thảo luận về POSIX thread trong các Chương 29 đến 33, và cũng vì nó làm sáng tỏ thêm hoạt động của `fork()` và `vfork()`.

```c
#define _GNU_SOURCE
#include <sched.h>
int clone(int (*func) (void *), void *child_stack, int flags, void *func_arg, ...
          /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );
                      Trả về process ID của child khi thành công, hoặc –1 khi có lỗi
```

Giống như `fork()`, một process mới được tạo bằng `clone()` là một bản sao gần như chính xác của process cha. Không giống như `fork()`, child được clone không tiếp tục từ điểm lời gọi, mà thay vào đó bắt đầu bằng cách gọi hàm được chỉ định trong đối số `func`; chúng ta sẽ gọi đây là hàm child. Khi được gọi, hàm child được truyền giá trị được chỉ định trong `func_arg`. Sử dụng ép kiểu thích hợp, hàm child có thể tự do diễn giải đối số này; ví dụ, như một `int` hoặc như một con trỏ đến một cấu trúc.

> Trong kernel, `fork()`, `vfork()`, và `clone()` cuối cùng được triển khai bởi cùng một hàm (`do_fork()` trong `kernel/fork.c`). Ở mức này, cloning gần hơn nhiều với forking: `sys_clone()` không có các đối số `func` và `func_arg`, và sau lời gọi, `sys_clone()` trả về trong child theo cùng cách như `fork()`. Văn bản chính mô tả hàm wrapper `clone()` mà glibc cung cấp cho `sys_clone()`. Hàm wrapper này gọi `func` sau khi `sys_clone()` trả về trong child.

Process child được clone kết thúc khi `func` trả về (trong trường hợp đó giá trị trả về của nó là trạng thái thoát của process) hoặc khi process thực hiện lời gọi `exit()` (hoặc `_exit()`). Process cha có thể chờ child được clone theo cách thông thường bằng cách sử dụng `wait()` hoặc tương tự.

Vì một child được clone có thể (giống như `vfork()`) chia sẻ bộ nhớ của cha, nó không thể sử dụng stack của cha. Thay vào đó, người gọi phải cấp phát một khối bộ nhớ có kích thước phù hợp để sử dụng như stack của child và truyền một con trỏ đến khối đó trong đối số `child_stack`. Trên hầu hết các kiến trúc phần cứng, stack phát triển xuống dưới, vì vậy đối số `child_stack` nên trỏ đến đầu cao của khối được cấp phát.

Đối số `flags` của `clone()` phục vụ hai mục đích. Đầu tiên, byte thấp nhất của nó chỉ định signal kết thúc của child, là signal được gửi cho cha khi child kết thúc. Byte này có thể là 0, trong trường hợp đó không có signal nào được tạo ra.

Với `fork()` và `vfork()`, chúng ta không có cách nào để chọn signal kết thúc; nó luôn là `SIGCHLD`.

Các byte còn lại của đối số `flags` giữ một bit mask kiểm soát hoạt động của `clone()`. Chúng ta tóm tắt các giá trị bit mask này trong Bảng 28-2, và mô tả chúng chi tiết hơn trong Mục 28.2.1.

**Bảng 28-2:** Các giá trị bit mask `flags` của `clone()`

| Cờ                  | Hiệu ứng khi có                                              |
|----------------------|--------------------------------------------------------------|
| CLONE_CHILD_CLEARTID | Xóa ctid khi child gọi exec() hoặc _exit() (từ 2.6 trở đi)  |
| CLONE_CHILD_SETTID   | Ghi thread ID của child vào ctid (từ 2.6 trở đi)            |
| CLONE_FILES          | Parent và child chia sẻ bảng file descriptor mở             |
| CLONE_FS             | Parent và child chia sẻ thuộc tính liên quan đến file system |
| CLONE_IO             | Child chia sẻ I/O context của cha (từ 2.6.25 trở đi)        |
| CLONE_NEWIPC         | Child lấy namespace System V IPC mới (từ 2.6.19 trở đi)     |
| CLONE_NEWNET         | Child lấy network namespace mới (từ 2.4.24 trở đi)          |
| CLONE_NEWNS          | Child lấy bản sao của mount namespace của cha (từ 2.4.19)   |
| CLONE_NEWPID         | Child lấy process-ID namespace mới (từ 2.6.19 trở đi)       |
| CLONE_NEWUSER        | Child lấy user-ID namespace mới (từ 2.6.23 trở đi)          |
| CLONE_NEWUTS         | Child lấy UTS namespace mới (từ 2.6.19 trở đi)              |
| CLONE_PARENT         | Làm cha của child giống như cha của người gọi (từ 2.4)       |
| CLONE_PARENT_SETTID  | Ghi thread ID của child vào ptid (từ 2.6 trở đi)            |
| CLONE_PID            | Cờ lỗi thời chỉ được dùng bởi process khởi động hệ thống    |
| CLONE_PTRACE         | Nếu cha đang bị theo dõi, thì cũng theo dõi child           |
| CLONE_SETTLS         | tls mô tả thread-local storage cho child (từ 2.6 trở đi)    |
| CLONE_SIGHAND        | Parent và child chia sẻ các disposition signal               |
| CLONE_SYSVSEM        | Parent và child chia sẻ semaphore undo values (từ 2.6)       |
| CLONE_THREAD         | Đặt child vào cùng thread group với cha (từ 2.4 trở đi)     |
| CLONE_UNTRACED       | Không thể buộc CLONE_PTRACE trên child (từ 2.6 trở đi)      |
| CLONE_VFORK          | Parent bị treo cho đến khi child gọi exec() hoặc _exit()    |
| CLONE_VM             | Parent và child chia sẻ virtual memory                       |

#### **Chương trình ví dụ**

Listing 28-3 hiển thị một ví dụ đơn giản về việc sử dụng `clone()` để tạo một process con. Chương trình chính thực hiện các bước sau:

- Mở một file descriptor (cho `/dev/null`) mà sẽ được đóng bởi child.
- Đặt giá trị cho đối số `flags` của `clone()` thành `CLONE_FILES` nếu một đối số dòng lệnh được cung cấp, để cha và child sẽ chia sẻ một bảng file descriptor duy nhất. Nếu không có đối số dòng lệnh nào được cung cấp, `flags` được đặt thành 0.
- Cấp phát một stack để sử dụng bởi child.
- Gọi `clone()` để tạo child. Đối số thứ ba (bit mask) bao gồm signal kết thúc. Đối số thứ tư (`func_arg`) chỉ định file descriptor được mở trước đó.
- Chờ child kết thúc.
- Kiểm tra xem file descriptor có còn mở không bằng cách thử `write()` đến nó. Chương trình báo cáo liệu `write()` thành công hay thất bại.

Thực thi của child được clone bắt đầu trong `childFunc()`, nhận (trong đối số `arg`) file descriptor được mở bởi chương trình chính. Child đóng file descriptor này và sau đó kết thúc bằng cách thực hiện `return`.

**Listing 28-3:** Sử dụng `clone()` để tạo một process con

```c
––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_clone.c
#define _GNU_SOURCE
#include <signal.h>
#include <sys/wait.h>
#include <fcntl.h>
#include <sched.h>
#include "tlpi_hdr.h"
#ifndef CHILD_SIG
#define CHILD_SIG SIGUSR1 /* Signal được tạo ra khi kết thúc child được clone */
#endif
static int /* Hàm khởi động cho child được clone */
childFunc(void *arg)
{
    if (close(*((int *) arg)) == -1)
        errExit("close");
    return 0; /* Child kết thúc bây giờ */
}
int
main(int argc, char *argv[])
{
    const int STACK_SIZE = 65536; /* Kích thước stack cho child được clone */
    char *stack;    /* Bắt đầu của buffer stack */
    char *stackTop; /* Cuối của buffer stack */
    int s, fd, flags;
    fd = open("/dev/null", O_RDWR); /* Child sẽ đóng fd này */
    if (fd == -1)
        errExit("open");
    /* Nếu argc > 1, child chia sẻ bảng file descriptor với cha */
    flags = (argc > 1) ? CLONE_FILES : 0;
    /* Cấp phát stack cho child */
    stack = malloc(STACK_SIZE);
    if (stack == NULL)
        errExit("malloc");
    stackTop = stack + STACK_SIZE; /* Giả sử stack phát triển xuống dưới */
    /* Bỏ qua CHILD_SIG, trong trường hợp đó là signal có mặc định kết thúc process;
       nhưng không bỏ qua SIGCHLD (bị bỏ qua theo mặc định), vì điều đó sẽ ngăn
       tạo zombie. */
    if (CHILD_SIG != 0 && CHILD_SIG != SIGCHLD)
        if (signal(CHILD_SIG, SIG_IGN) == SIG_ERR)
            errExit("signal");
    /* Tạo child; child bắt đầu thực thi trong childFunc() */
    if (clone(childFunc, stackTop, flags | CHILD_SIG, (void *) &fd) == -1)
        errExit("clone");
    /* Cha tiếp tục đến đây. Chờ child; __WCLONE là
       cần thiết cho child thông báo với signal khác SIGCHLD. */
    if (waitpid(-1, NULL, (CHILD_SIG != SIGCHLD) ? __WCLONE : 0) == -1)
        errExit("waitpid");
    printf("child has terminated\n");
    /* Việc close() file descriptor trong child có ảnh hưởng đến cha không? */
    s = write(fd, "x", 1);
    if (s == -1 && errno == EBADF)
        printf("file descriptor %d has been closed\n", fd);
    else if (s == -1)
        printf("write() on file descriptor %d failed "
               "unexpectedly (%s)\n", fd, strerror(errno));
    else
        printf("write() on file descriptor %d succeeded\n", fd);
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_clone.c
```

Khi chúng ta chạy chương trình trong Listing 28-3 mà không có đối số dòng lệnh, chúng ta thấy như sau:

```
$ ./t_clone    Không sử dụng CLONE_FILES
child has terminated
write() on file descriptor 3 succeeded    close() của child không ảnh hưởng đến cha
```

Khi chúng ta chạy chương trình với một đối số dòng lệnh, chúng ta có thể thấy rằng hai process chia sẻ bảng file descriptor:

```
$ ./t_clone x    Sử dụng CLONE_FILES
child has terminated
file descriptor 3 has been closed    close() của child ảnh hưởng đến cha
```

## **28.2.1 Đối Số flags của clone()**

Đối số `flags` của `clone()` là sự kết hợp (OR) của các giá trị bit mask được mô tả trong các trang sau. Thay vì trình bày các cờ này theo thứ tự bảng chữ cái, chúng ta trình bày chúng theo thứ tự giúp giải thích dễ dàng hơn, và bắt đầu với những cờ được sử dụng trong việc triển khai POSIX thread. Từ quan điểm triển khai thread, nhiều cách sử dụng từ "process" bên dưới có thể được thay thế bằng "thread".

Tại thời điểm này, đáng để nhận xét rằng, ở một mức độ nào đó, chúng ta đang chơi chữ khi cố gắng phân biệt giữa thuật ngữ thread và process. Điều hữu ích là giới thiệu thuật ngữ kernel scheduling entity (KSE), được sử dụng trong một số tài liệu để chỉ các đối tượng được xử lý bởi kernel scheduler. Thực ra, thread và process chỉ đơn giản là KSE cung cấp mức độ chia sẻ thuộc tính lớn hơn và nhỏ hơn (virtual memory, file descriptor mở, signal disposition, process ID, v.v.) với các KSE khác.

## **Chia sẻ bảng file descriptor: CLONE_FILES**

Nếu cờ `CLONE_FILES` được chỉ định, cha và child chia sẻ cùng một bảng file descriptor mở. Điều này có nghĩa là việc cấp phát hoặc giải phóng file descriptor (`open()`, `close()`, `dup()`, `pipe()`, `socket()`, v.v.) trong một trong hai process sẽ hiển thị với process kia. Nếu cờ `CLONE_FILES` không được đặt, thì bảng file descriptor không được chia sẻ, và child nhận được một bản sao của bảng cha tại thời điểm lời gọi `clone()`.

Đặc tả của POSIX thread yêu cầu tất cả các thread trong một process chia sẻ cùng các file descriptor mở.

#### **Chia sẻ thông tin liên quan đến file system: CLONE_FS**

Nếu cờ `CLONE_FS` được chỉ định, thì cha và child chia sẻ thông tin liên quan đến file system—umask, thư mục root, và thư mục làm việc hiện tại. Điều này có nghĩa là các lời gọi đến `umask()`, `chdir()`, hoặc `chroot()` trong một trong hai process sẽ ảnh hưởng đến process kia. Nếu cờ `CLONE_FS` không được đặt, thì cha và child có các bản sao riêng biệt của thông tin này (như với `fork()` và `vfork()`).

Việc chia sẻ thuộc tính được cung cấp bởi `CLONE_FS` được yêu cầu bởi POSIX thread.

#### **Chia sẻ các disposition signal: CLONE_SIGHAND**

Nếu cờ `CLONE_SIGHAND` được đặt, thì cha và child chia sẻ cùng bảng signal disposition. Việc sử dụng `sigaction()` hoặc `signal()` để thay đổi disposition của signal trong một trong hai process sẽ ảnh hưởng đến disposition của signal đó trong process kia. Nếu cờ `CLONE_SIGHAND` không được đặt, thì các signal disposition không được chia sẻ; thay vào đó, child nhận được một bản sao của bảng signal disposition của cha (như với `fork()` và `vfork()`). Cờ `CLONE_SIGHAND` không ảnh hưởng đến signal mask và tập hợp các signal đang chờ xử lý của process, luôn riêng biệt cho hai process. Từ Linux 2.6 trở đi, `CLONE_VM` cũng phải được bao gồm trong `flags` nếu `CLONE_SIGHAND` được chỉ định.

Việc chia sẻ signal disposition được yêu cầu bởi POSIX thread.

#### **Chia sẻ virtual memory của cha: CLONE_VM**

Nếu cờ `CLONE_VM` được đặt, thì cha và child chia sẻ cùng các virtual memory page (như với `vfork()`). Các cập nhật bộ nhớ hoặc lời gọi `mmap()` hoặc `munmap()` bởi một trong hai process sẽ hiển thị với process kia. Nếu cờ `CLONE_VM` không được đặt, thì child nhận được bản sao của virtual memory của cha (như với `fork()`).

Chia sẻ cùng virtual memory là một trong những thuộc tính định nghĩa của thread, và được yêu cầu bởi POSIX thread.

## **Thread group: CLONE_THREAD**

Nếu cờ `CLONE_THREAD` được đặt, thì child được đặt vào cùng thread group với cha. Nếu cờ này không được đặt, child được đặt vào thread group mới của riêng nó.

Thread group được giới thiệu trong Linux 2.4 để cho phép các thư viện threading hỗ trợ yêu cầu của POSIX thread rằng tất cả các thread trong một process chia sẻ một process ID duy nhất (tức là `getpid()` trong mỗi thread nên trả về cùng một giá trị). Một thread group là một nhóm KSE chia sẻ cùng thread group identifier (TGID), như được hiển thị trong Hình 28-1.

Kể từ Linux 2.4, `getpid()` trả về TGID của thread gọi. Nói cách khác, một TGID là cùng một thứ với một process ID.

Mỗi thread trong một thread group được phân biệt bởi một thread identifier (TID) duy nhất. Linux 2.4 đã giới thiệu một system call mới, `gettid()`, để cho phép một thread lấy thread ID của chính nó. Một thread ID được biểu diễn bằng cùng kiểu dữ liệu được sử dụng cho một process ID, `pid_t`. Thread ID là duy nhất trên toàn hệ thống, và kernel đảm bảo rằng không có thread ID nào sẽ giống với bất kỳ process ID nào trên hệ thống, ngoại trừ khi một thread là thread group leader cho một process.

Thread đầu tiên trong một thread group mới có một thread ID giống với thread group ID của nó. Thread này được gọi là thread group leader.

> Thread ID mà chúng ta đang thảo luận ở đây không giống với thread ID (kiểu dữ liệu `pthread_t`) được sử dụng bởi POSIX thread. Các định danh sau được tạo và duy trì nội bộ (trong user space) bởi một cài đặt POSIX thread.

Tất cả các thread trong một thread group có cùng parent process ID—đó là của thread group leader. Chỉ sau khi tất cả các thread trong một thread group đã kết thúc, một signal `SIGCHLD` (hoặc signal kết thúc khác) mới được gửi cho process cha đó. Các ngữ nghĩa này tương ứng với yêu cầu của POSIX thread.

Khi một thread `CLONE_THREAD` kết thúc, không có signal nào được gửi đến thread đã tạo ra nó bằng cách sử dụng `clone()`. Tương ứng, không thể sử dụng `wait()` (hoặc tương tự) để chờ một thread được tạo bằng cách sử dụng `CLONE_THREAD`. Để phát hiện việc kết thúc của một thread được tạo bằng cách sử dụng `CLONE_THREAD`, một cơ chế đồng bộ đặc biệt, được gọi là futex, được sử dụng.

Nếu bất kỳ thread nào trong một thread group thực hiện `exec()`, thì tất cả các thread ngoại trừ thread group leader đều bị kết thúc, và chương trình mới được exec trong thread group leader. Trong chương trình mới, `gettid()` sẽ trả về thread ID của thread group leader. Trong quá trình `exec()`, signal kết thúc mà process này nên gửi cho cha của nó được đặt lại thành `SIGCHLD`.

Nếu một trong các thread trong một thread group tạo một child bằng cách sử dụng `fork()` hoặc `vfork()`, thì bất kỳ thread nào trong nhóm có thể theo dõi child đó bằng cách sử dụng `wait()` hoặc tương tự.

## **Hỗ trợ thư viện threading: CLONE_PARENT_SETTID, CLONE_CHILD_SETTID, và CLONE_CHILD_CLEARTID**

Các cờ `CLONE_PARENT_SETTID`, `CLONE_CHILD_SETTID`, và `CLONE_CHILD_CLEARTID` được thêm vào trong Linux 2.6 để hỗ trợ việc triển khai POSIX thread. Các cờ này ảnh hưởng đến cách `clone()` xử lý các đối số `ptid` và `ctid` của nó.

Nếu cờ `CLONE_PARENT_SETTID` được đặt, thì kernel ghi thread ID của thread child vào vị trí được trỏ bởi `ptid`. Thread ID được sao chép vào `ptid` trước khi bộ nhớ của cha được sao chép. Điều này có nghĩa là, ngay cả khi cờ `CLONE_VM` không được chỉ định, cả cha và child đều có thể thấy thread ID của child trong vị trí này.

Nếu cờ `CLONE_CHILD_SETTID` được đặt, thì `clone()` ghi thread ID của thread child vào vị trí được trỏ bởi `ctid`. Việc đặt `ctid` chỉ được thực hiện trong bộ nhớ của child, nhưng điều này sẽ ảnh hưởng đến cha nếu `CLONE_VM` cũng được chỉ định.

Nếu cờ `CLONE_CHILD_CLEARTID` được đặt, thì `clone()` xóa vị trí bộ nhớ được trỏ bởi `ctid` khi child kết thúc.

Đối số `ctid` là cơ chế mà cài đặt NPTL threading sử dụng để nhận thông báo về việc kết thúc của một thread. Thông báo như vậy được yêu cầu bởi hàm `pthread_join()`, là cơ chế POSIX thread mà một thread có thể chờ việc kết thúc của một thread khác.

Kernel xử lý vị trí được trỏ bởi `ctid` như một futex, một cơ chế đồng bộ hiệu quả. (Xem trang manual `futex(2)` để biết thêm chi tiết về futex.) Thông báo về việc kết thúc thread có thể được lấy bằng cách thực hiện system call `futex()` chặn chờ sự thay đổi trong giá trị tại vị trí được trỏ bởi `ctid`.

#### **Thread-local storage: CLONE_SETTLS**

Nếu cờ `CLONE_SETTLS` được đặt, thì đối số `tls` trỏ đến một cấu trúc `user_desc` mô tả buffer thread-local storage để được sử dụng cho thread này. Cờ này được thêm vào trong Linux 2.6 để hỗ trợ cài đặt NPTL của thread-local storage (Mục 31.4).

#### **Chia sẻ System V semaphore undo values: CLONE_SYSVSEM**

Nếu cờ `CLONE_SYSVSEM` được đặt, thì cha và child chia sẻ một danh sách System V semaphore undo value (Mục 47.8). Nếu cờ này không được đặt, thì cha và child có các danh sách undo riêng biệt, và danh sách undo của child ban đầu là rỗng.

Cờ `CLONE_SYSVSEM` có sẵn từ kernel 2.6 trở đi, và cung cấp các ngữ nghĩa chia sẻ được yêu cầu bởi POSIX thread.

#### **Mỗi process mount namespace: CLONE_NEWNS**

Từ kernel 2.4.19 trở đi, Linux hỗ trợ khái niệm về mỗi process mount namespace. Một mount namespace là tập hợp các mount point được duy trì bởi các lời gọi đến `mount()` và `umount()`. Mount namespace ảnh hưởng đến cách các pathname được giải quyết đến các file thực tế, cũng như hoạt động của các system call như `chdir()` và `chroot()`.

Theo mặc định, cha và child chia sẻ một mount namespace. Một process có đặc quyền (`CAP_SYS_ADMIN`) có thể chỉ định cờ `CLONE_NEWNS` để child nhận được bản sao của mount namespace của cha. Sau đó, các thay đổi đối với namespace bởi một process không hiển thị trong process kia.

Chỉ định cả `CLONE_NEWNS` và `CLONE_FS` trong cùng một lời gọi đến `clone()` là vô nghĩa và không được phép.

#### **Làm cha của child giống với cha của người gọi: CLONE_PARENT**

Theo mặc định, khi chúng ta tạo một process mới với `clone()`, cha của process đó (được trả về bởi `getppid()`) là process gọi `clone()` (như với `fork()` và `vfork()`). Nếu cờ `CLONE_PARENT` được đặt, thì cha của child sẽ là cha của người gọi.

Cờ `CLONE_PARENT` có sẵn trong Linux 2.4 và sau đó.

#### **Làm PID của child giống với PID của cha: CLONE_PID (lỗi thời)**

Nếu cờ `CLONE_PID` được đặt, thì child có cùng process ID như cha. Chỉ process khởi động hệ thống (process ID 0) mới có thể chỉ định cờ này; nó được sử dụng khi khởi tạo một hệ thống đa bộ xử lý.

Cờ `CLONE_PID` không có nghĩa là sử dụng trong các ứng dụng người dùng. Trong Linux 2.6, nó đã bị xóa, và được thay thế bởi `CLONE_IDLETASK`.

### **Process tracing: CLONE_PTRACE và CLONE_UNTRACED**

Nếu cờ `CLONE_PTRACE` được đặt và process gọi đang bị theo dõi, thì child cũng bị theo dõi.

Từ kernel 2.6 trở đi, cờ `CLONE_UNTRACED` có thể được đặt, nghĩa là một process theo dõi không thể buộc `CLONE_PTRACE` trên child này. Cờ `CLONE_UNTRACED` được sử dụng nội bộ bởi kernel trong việc tạo các kernel thread.

#### **Treo process cha cho đến khi child thoát hoặc exec: CLONE_VFORK**

Nếu cờ `CLONE_VFORK` được đặt, thì việc thực thi của process cha bị treo cho đến khi child giải phóng các tài nguyên virtual memory của nó thông qua lời gọi `exec()` hoặc `_exit()` (như với `vfork()`).

## **Các cờ clone() mới để hỗ trợ container**

Một số giá trị cờ `clone()` mới được thêm vào trong Linux 2.6.19 và sau đó: `CLONE_IO`, `CLONE_NEWIPC`, `CLONE_NEWNET`, `CLONE_NEWPID`, `CLONE_NEWUSER`, và `CLONE_NEWUTS`. (Xem trang manual `clone(2)` để biết chi tiết về các cờ này.)

Hầu hết các cờ này được cung cấp để hỗ trợ việc triển khai container. Một container là một hình thức virtualization nhẹ, trong đó các nhóm process chạy trên cùng một kernel có thể được cách ly với nhau trong các môi trường xuất hiện như các máy riêng biệt. Container cũng có thể được lồng vào nhau, cái này bên trong cái kia. Cách tiếp cận container tương phản với full virtualization, trong đó mỗi môi trường được virtualize chạy một kernel riêng biệt.

Để triển khai container, các nhà phát triển kernel đã phải cung cấp một lớp indirection trong kernel xung quanh mỗi tài nguyên hệ thống toàn cục—như process ID, networking stack, các định danh được trả về bởi `uname()`, các đối tượng System V IPC, và các user và group ID namespace—để mỗi container có thể cung cấp phiên bản riêng của các tài nguyên này.

#### **Cách sử dụng cờ clone()**

Một cách gần đúng, chúng ta có thể nói rằng `fork()` tương ứng với lời gọi `clone()` với `flags` được chỉ định là chỉ `SIGCHLD`, trong khi `vfork()` tương ứng với lời gọi `clone()` chỉ định `flags` như sau:

```c
CLONE_VM | CLONE_VFORK | SIGCHLD
```

Kể từ phiên bản 2.3.3, hàm wrapper `fork()` của glibc được cung cấp như một phần của cài đặt NPTL threading bỏ qua system call `fork()` của kernel và gọi `clone()`. Hàm wrapper này gọi bất kỳ fork handler nào được thiết lập bởi người gọi bằng cách sử dụng `pthread_atfork()` (xem Mục 33.3).

Cài đặt NPTL threading sử dụng `clone()` (với tất cả bảy đối số) để tạo thread bằng cách chỉ định `flags` như sau:

```c
CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD |
CLONE_SETTLS | CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID | CLONE_SYSVSEM
```

## **28.2.2 Mở Rộng waitpid() Cho Các Child Được Clone**

Để chờ các child được tạo bởi `clone()`, các giá trị bổ sung (đặc thù Linux) sau đây có thể được bao gồm trong đối số bit mask `options` cho `waitpid()`, `wait3()`, và `wait4()`:

```
__WCLONE
```

Nếu được đặt, thì chờ các clone child. Nếu không được đặt, thì chờ các non-clone child. Trong ngữ cảnh này, một clone child là cái gửi signal khác `SIGCHLD` đến cha của nó khi kết thúc.

`__WALL` (kể từ Linux 2.4)

Chờ tất cả các child, bất kể loại (clone hoặc non-clone).

`__WNOTHREAD` (kể từ Linux 2.4)

Theo mặc định, các lời gọi wait không chỉ chờ các child của process gọi, mà còn chờ các child của bất kỳ process nào khác trong cùng thread group với người gọi. Chỉ định cờ `__WNOTHREAD` giới hạn việc chờ đối với các child của process gọi.

Các cờ này không thể được sử dụng với `waitid()`.

## **28.3 Tốc Độ Tạo Process**

Bảng 28-3 hiển thị một số so sánh tốc độ cho các phương pháp tạo process khác nhau. Kết quả được lấy bằng cách sử dụng một chương trình test thực thi một vòng lặp liên tục tạo một process con và sau đó chờ nó kết thúc. Bảng so sánh các phương pháp khác nhau bằng cách sử dụng ba kích thước bộ nhớ process khác nhau, như được chỉ ra bởi giá trị Total virtual memory.

**Bảng 28-3:** Thời gian cần để tạo 100.000 process bằng cách sử dụng `fork()`, `vfork()`, và `clone()`

| Phương pháp tạo process | Total Virtual Memory 1.70 MB | Total Virtual Memory 2.70 MB | Total Virtual Memory 11.70 MB |
|-------------------------|------------------------------|------------------------------|-------------------------------|
| `fork()` | 22.27 (7.99) / 4544 | 26.38 (8.98) / 4135 | 126.93 (52.55) / 1276 |
| `vfork()` | 3.52 (2.49) / 28955 | 3.55 (2.50) / 28621 | 3.53 (2.51) / 28810 |
| `clone()` | 2.97 (2.14) / 34333 | 2.98 (2.13) / 34217 | 2.93 (2.10) / 34688 |
| `fork() + exec()` | 135.72 (12.39) / 764 | 146.15 (16.69) / 719 | 260.34 (61.86) / 435 |
| `vfork() + exec()` | 107.36 (6.27) / 969 | 107.81 (6.35) / 964 | 107.97 (6.38) / 960 |

*(Thời gian tính bằng giây / Tỷ lệ process/giây)*

Ba hàng dữ liệu đầu tiên hiển thị thời gian cho việc tạo process đơn giản (không exec chương trình mới trong child). Trong mỗi trường hợp, các process con thoát ngay sau khi chúng được tạo, và cha chờ mỗi child kết thúc trước khi tạo tiếp.

Hàng đầu tiên chứa các giá trị cho system call `fork()`. Từ dữ liệu, chúng ta có thể thấy rằng khi process lớn hơn, `fork()` mất nhiều thời gian hơn. Những sự khác biệt về thời gian này hiển thị thời gian bổ sung cần thiết để sao chép các bảng page ngày càng lớn hơn cho child và đánh dấu tất cả các mục data, heap, và stack segment page là read-only.

Hàng dữ liệu thứ hai cung cấp cùng thống kê cho `vfork()`. Chúng ta thấy rằng khi kích thước process tăng, thời gian vẫn không thay đổi—vì không có bảng page hoặc page nào được sao chép trong quá trình `vfork()`, kích thước virtual memory của process gọi không có hiệu ứng.

Hàng dữ liệu thứ ba hiển thị thống kê cho việc tạo process bằng cách sử dụng `clone()` với các cờ sau:

```c
CLONE_VM | CLONE_VFORK | CLONE_FS | CLONE_SIGHAND | CLONE_FILES
```

Hai cờ đầu tiên mô phỏng hành vi của `vfork()`. Các cờ còn lại chỉ định rằng cha và child nên chia sẻ các thuộc tính file-system (umask, thư mục root, và thư mục làm việc hiện tại), bảng signal disposition, và bảng file descriptor mở.

Sự khác biệt giữa `fork()` và `vfork()` khá rõ rệt. Tuy nhiên, các điểm sau cần ghi nhớ:

- Cột dữ liệu cuối cùng, nơi `vfork()` nhanh hơn hơn 30 lần so với `fork()`, tương ứng với một process lớn. Các process điển hình sẽ nằm ở đâu đó gần hai cột đầu tiên của bảng.
- Vì thời gian cần để tạo process thường nhỏ hơn nhiều so với thời gian cần cho một `exec()`, sự khác biệt ít rõ ràng hơn nhiều nếu `fork()` hoặc `vfork()` được theo sau bởi một `exec()`. Điều này được minh họa bởi cặp hàng dữ liệu cuối cùng trong Bảng 28-3.

## **28.4 Tác Động của exec() và fork() Trên Các Thuộc Tính Process**

Một process có nhiều thuộc tính, một số đã được mô tả trong các chương trước, và một số khác chúng ta khám phá trong các chương sau. Liên quan đến các thuộc tính này, hai câu hỏi nảy sinh:

- Điều gì xảy ra với các thuộc tính này khi một process thực hiện `exec()`?
- Các thuộc tính nào được kế thừa bởi một child khi `fork()` được thực hiện?

Bảng 28-4 tóm tắt câu trả lời cho các câu hỏi này. Cột `exec()` chỉ ra các thuộc tính được bảo tồn trong quá trình `exec()`. Cột `fork()` chỉ ra các thuộc tính được kế thừa (hoặc trong một số trường hợp, được chia sẻ) bởi một child sau `fork()`.

**Bảng 28-4:** Tác động của `exec()` và `fork()` trên các thuộc tính process

| Thuộc tính process | exec() | fork() | Interface ảnh hưởng thuộc tính; ghi chú bổ sung |
|--------------------|--------|--------|--------------------------------------------------|
| **Không gian địa chỉ process** | | | |
| Text segment | Không | Chia sẻ | Process con chia sẻ text segment với cha. |
| Stack segment | Không | Có | Hàm vào/ra; `alloca()`, `longjmp()`, `siglongjmp()`. |
| Data và heap segment | Không | Có | `brk()`, `sbrk()`. |
| Biến environment | Xem ghi chú | Có | `putenv()`, `setenv()`; sửa đổi trực tiếp `environ`. Bị ghi đè bởi `execle()` và `execve()` và được bảo tồn bởi các lời gọi `exec()` còn lại. |
| Ánh xạ bộ nhớ | Không | Có; xem ghi chú | `mmap()`, `munmap()`. Cờ `MAP_NORESERVE` của ánh xạ được kế thừa qua `fork()`. Các ánh xạ được đánh dấu bằng `madvise(MADV_DONTFORK)` không được kế thừa qua `fork()`. |
| Khóa bộ nhớ | Không | Không | `mlock()`, `munlock()`. |
| **Định danh và thông tin xác thực process** | | | |
| Process ID | Có | Không | |
| Parent process ID | Có | Không | |
| Process group ID | Có | Có | `setpgid()`. |
| Session ID | Có | Có | `setsid()`. |
| Real ID | Có | Có | `setuid()`, `setgid()`, và các lời gọi liên quan. |
| Effective và saved set ID | Xem ghi chú | Có | `setuid()`, `setgid()`, và các lời gọi liên quan. Chương 9 giải thích cách `exec()` ảnh hưởng đến các ID này. |
| Supplementary group ID | Có | Có | `setgroups()`, `initgroups()`. |
| **File, file I/O, và thư mục** | | | |
| File descriptor mở | Xem ghi chú | Có | `open()`, `close()`, `dup()`, `pipe()`, `socket()`, v.v. File descriptor được bảo tồn qua `exec()` trừ khi được đánh dấu close-on-exec. Descriptor trong child và cha tham chiếu đến cùng open file description; xem Mục 5.4. |
| Cờ close-on-exec | Có (nếu tắt) | Có | `fcntl(F_SETFD)`. |
| File offset | Có | Chia sẻ | `lseek()`, `read()`, `write()`, `readv()`, `writev()`. Child chia sẻ file offset với cha. |
| Cờ trạng thái file mở | Có | Chia sẻ | `open()`, `fcntl(F_SETFL)`. Child chia sẻ cờ trạng thái file mở với cha. |
| Thao tác asynchronous I/O | Xem ghi chú | Không | `aio_read()`, `aio_write()`, và các lời gọi liên quan. Các thao tác đang chờ xử lý bị hủy trong quá trình `exec()`. |
| Directory stream | Không | Có; xem ghi chú | `opendir()`, `readdir()`. |
| **File system** | | | |
| Thư mục làm việc hiện tại | Có | Có | `chdir()`. |
| Thư mục root | Có | Có | `chroot()`. |
| Mặt nạ tạo chế độ file | Có | Có | `umask()`. |
| **Signal** | | | |
| Signal disposition | Xem ghi chú | Có | `signal()`, `sigaction()`. Trong quá trình `exec()`, các signal với disposition được đặt thành mặc định hoặc bỏ qua không thay đổi; các signal được bắt quay trở lại disposition mặc định. Xem Mục 27.5. |
| Signal mask | Có | Có | Gửi signal, `sigprocmask()`, `sigaction()`. |
| Tập hợp signal đang chờ xử lý | Có | Không | Gửi signal; `raise()`, `kill()`, `sigqueue()`. |
| Alternate signal stack | Không | Có | `sigaltstack()`. |
| **Bộ đếm thời gian** | | | |
| Interval timer | Có | Không | `setitimer()`. |
| Bộ đếm thời gian bởi `alarm()` | Có | Không | `alarm()`. |
| POSIX timer | Không | Không | `timer_create()` và các lời gọi liên quan. |
| **POSIX thread** | | | |
| Thread | Không | Xem ghi chú | Trong quá trình `fork()`, chỉ thread gọi được sao chép trong child. |
| Trạng thái và kiểu thread cancelability | Không | Có | Sau một `exec()`, kiểu và trạng thái cancelability được đặt lại thành `PTHREAD_CANCEL_ENABLE` và `PTHREAD_CANCEL_DEFERRED`, tương ứng. |
| Mutex và condition variable | Không | Có | Xem Mục 33.3. |
| **Ưu tiên và lập lịch** | | | |
| Giá trị nice | Có | Có | `nice()`, `setpriority()`. |
| Chính sách và ưu tiên lập lịch | Có | Có | `sched_setscheduler()`, `sched_setparam()`. |
| **Tài nguyên và thời gian CPU** | | | |
| Giới hạn tài nguyên | Có | Có | `setrlimit()`. |
| Thời gian CPU process và child | Có | Không | Được trả về bởi `times()`. |
| Sử dụng tài nguyên | Có | Không | Được trả về bởi `getrusage()`. |
| **Interprocess communication** | | | |
| Shared memory segment System V | Không | Có | `shmat()`, `shmdt()`. |
| POSIX shared memory | Không | Có | `shm_open()` và các lời gọi liên quan. |
| POSIX message queue | Không | Có | `mq_open()` và các lời gọi liên quan. Descriptor trong child và cha tham chiếu đến cùng open message queue description. Child không kế thừa các đăng ký message notification của cha. |
| POSIX named semaphore | Không | Chia sẻ | `sem_open()` và các lời gọi liên quan. Child chia sẻ tham chiếu đến cùng semaphore với cha. |
| POSIX unnamed semaphore | Không | Xem ghi chú | `sem_init()` và các lời gọi liên quan. Nếu các semaphore nằm trong một vùng shared memory, thì child chia sẻ semaphore với cha; ngược lại, child có bản sao riêng của semaphore. |
| Điều chỉnh System V semaphore | Có | Không | `semop()`. Xem Mục 47.8. |
| File lock | Có | Xem ghi chú | `flock()`. Child kế thừa tham chiếu đến cùng lock với cha. |
| Record lock | Xem ghi chú | Không | `fcntl(F_SETLK)`. Lock được bảo tồn qua `exec()` trừ khi file descriptor tham chiếu đến file được đánh dấu close-on-exec; xem Mục 55.3.5. |
| **Linh tinh** | | | |
| Locale | Không | Có | `setlocale()`. Như một phần của khởi tạo runtime C, tương đương với `setlocale(LC_ALL, "C")` được thực thi sau khi một chương trình mới được exec. |
| Môi trường floating-point | Không | Có | Khi một chương trình mới được exec, trạng thái của môi trường floating-point được đặt lại thành mặc định; xem `fenv(3)`. |
| Controlling terminal | Có | Có | |
| Exit handler | Không | Có | `atexit()`, `on_exit()`. |
| **Đặc thù Linux** | | | |
| File-system ID | Xem ghi chú | Có | `setfsuid()`, `setfsgid()`. Các ID này cũng thay đổi mỗi khi effective ID tương ứng thay đổi. |
| timerfd timer | Có | Xem ghi chú | `timerfd_create()`; child kế thừa file descriptor tham chiếu đến cùng timer với cha. |
| Capability | Xem ghi chú | Có | `capset()`. Việc xử lý capability trong quá trình `exec()` được mô tả trong Mục 39.5. |
| Capability bounding set | Có | Có | |
| Cờ capabilities securebits | Xem ghi chú | Có | Tất cả cờ securebits được bảo tồn trong quá trình `exec()` ngoại trừ `SECBIT_KEEP_CAPS`, luôn bị xóa. |
| CPU affinity | Có | Có | `sched_setaffinity()`. |
| SCHED_RESET_ON_FORK | Có | Không | Xem Mục 35.3.2. |
| CPU được phép | Có | Có | Xem `cpuset(7)`. |
| Memory node được phép | Có | Có | Xem `cpuset(7)`. |
| Memory policy | Có | Có | Xem `set_mempolicy(2)`. |
| File lease | Có | Xem ghi chú | `fcntl(F_SETLEASE)`. Child kế thừa tham chiếu đến cùng lease với cha. |
| Directory change notification | Có | Không | API dnotify, có sẵn qua `fcntl(F_NOTIFY)`. |
| `prctl(PR_SET_DUMPABLE)` | Xem ghi chú | Có | Trong quá trình `exec()`, cờ `PR_SET_DUMPABLE` được đặt, trừ khi exec một chương trình set-user-ID hoặc set-group-ID, trong trường hợp đó nó bị xóa. |
| `prctl(PR_SET_PDEATHSIG)` | Có | Không | |
| `prctl(PR_SET_NAME)` | Không | Có | |
| oom_adj | Có | Có | Xem Mục 49.9. |
| coredump_filter | Có | Có | Xem Mục 22.1. |

## **28.5 Tóm Tắt**

Khi process accounting được bật, kernel ghi một bản ghi kế toán vào một file cho mỗi process kết thúc trên hệ thống. Bản ghi này chứa thống kê về các tài nguyên được sử dụng bởi process.

Giống như `fork()`, system call `clone()` đặc thù của Linux tạo một process mới, nhưng cho phép kiểm soát tốt hơn các thuộc tính được chia sẻ giữa cha và child. System call này được sử dụng chủ yếu để triển khai các thư viện threading.

Chúng ta đã so sánh tốc độ tạo process bằng cách sử dụng `fork()`, `vfork()`, và `clone()`. Mặc dù `vfork()` nhanh hơn `fork()`, sự khác biệt thời gian giữa các system call này nhỏ so với thời gian cần cho một process con thực hiện `exec()` tiếp theo.

Khi một process con được tạo qua `fork()`, nó kế thừa các bản sao của (hoặc trong một số trường hợp chia sẻ) một số thuộc tính process nhất định từ cha, trong khi các thuộc tính process khác không được kế thừa. Ví dụ, một child kế thừa các bản sao của bảng file descriptor cha và signal disposition, nhưng không kế thừa interval timer, record lock, hoặc tập hợp các signal đang chờ xử lý của cha. Tương ứng, khi một process thực hiện `exec()`, một số thuộc tính process vẫn không thay đổi, trong khi những thuộc tính khác được đặt lại về giá trị mặc định.

### **Thông tin thêm**

Tham khảo các nguồn thông tin thêm được liệt kê trong Mục 24.6. Chương 17 của [Frisch, 2002] mô tả quản trị process accounting. [Bovet & Cesati, 2005] mô tả việc triển khai system call `clone()`.

## **28.6 Bài Tập**

**28-1.** Viết một chương trình để xem tốc độ của các system call `fork()` và `vfork()` trên hệ thống của bạn nhanh đến mức nào. Mỗi process con nên thoát ngay lập tức, và process cha nên `wait()` trên mỗi child trước khi tạo tiếp theo. So sánh sự khác biệt tương đối cho hai system call này với những kết quả trong Bảng 28-3. Lệnh shell built-in `time` có thể được sử dụng để đo thời gian thực thi của một chương trình.
