## Chương 12
# Thông Tin Hệ Thống và Process (System and Process Information)

Trong chương này, chúng ta xem xét các cách truy cập nhiều loại thông tin hệ thống và process. Trọng tâm chính của chương là thảo luận về file system `/proc`. Chúng ta cũng mô tả system call `uname()`, dùng để lấy các định danh hệ thống khác nhau.

---

## 12.1 File System /proc

Trong các triển khai UNIX cũ, thường không có cách dễ dàng để phân tích (hoặc thay đổi) nội tại các thuộc tính của kernel, để trả lời các câu hỏi như:

- Có bao nhiêu process đang chạy trên hệ thống và chúng thuộc về ai?
- Một process đang mở những file nào?
- Những file nào hiện đang bị lock, và process nào đang giữ các lock đó?
- Những socket nào đang được dùng trên hệ thống?

Một số triển khai UNIX cũ giải quyết vấn đề này bằng cách cho phép các chương trình có đặc quyền đào sâu vào các cấu trúc dữ liệu trong kernel memory. Tuy nhiên, cách tiếp cận này gặp phải nhiều vấn đề. Đặc biệt, nó đòi hỏi kiến thức chuyên sâu về các cấu trúc dữ liệu kernel, và các cấu trúc này có thể thay đổi từ phiên bản kernel này sang phiên bản khác, đòi hỏi các chương trình phụ thuộc vào chúng phải được viết lại.

Để cung cấp quyền truy cập dễ dàng hơn vào thông tin kernel, nhiều triển khai UNIX hiện đại cung cấp virtual file system `/proc`. File system này nằm dưới thư mục `/proc` và chứa các file khác nhau hiển thị thông tin kernel, cho phép các process đọc thông tin đó một cách thuận tiện, và trong một số trường hợp thay đổi nó, bằng các system call I/O file thông thường. File system `/proc` được gọi là virtual vì các file và subdirectory chứa trong đó không nằm trên đĩa. Thay vào đó, kernel tạo chúng "on the fly" khi các process truy cập chúng.

Trong mục này, chúng ta trình bày tổng quan về file system `/proc`. Trong các chương sau, chúng ta mô tả các file `/proc` cụ thể khi chúng liên quan đến chủ đề của từng chương. Mặc dù nhiều triển khai UNIX cung cấp file system `/proc`, SUSv3 không quy định file system này; các chi tiết mô tả trong sách này là đặc thù của Linux.

### 12.1.1 Lấy Thông Tin về một Process: /proc/PID

Đối với mỗi process trên hệ thống, kernel cung cấp một directory tương ứng tên `/proc/PID`, trong đó `PID` là ID của process. Trong directory này là các file và subdirectory khác nhau chứa thông tin về process đó. Ví dụ, ta có thể lấy thông tin về process `init`, luôn có process ID là 1, bằng cách xem các file dưới thư mục `/proc/1`.

Trong số các file trong mỗi thư mục `/proc/PID` có một file tên `status`, cung cấp nhiều thông tin về process:

```
$ cat /proc/1/status
Name:    init                    Tên lệnh đang chạy bởi process này
State:   S (sleeping)            Trạng thái của process này
Tgid:    1                       Thread group ID (PID truyền thống, getpid())
Pid:     1                       Thực ra là thread ID (gettid())
PPid:    0                       Parent process ID
TracerPid: 0                     PID của tracing process (0 nếu không bị theo dõi)
Uid:     0  0  0  0              Real, effective, saved set, và FS UID
Gid:     0  0  0  0              Real, effective, saved set, và FS GID
FDSize:  256                     Số slot file descriptor hiện được cấp phát
Groups:                          Supplementary group ID
VmPeak:  852 kB                  Kích thước virtual memory đỉnh
VmSize:  724 kB                  Kích thước virtual memory hiện tại
VmLck:   0 kB                    Locked memory
VmHWM:   288 kB                  Kích thước resident set đỉnh
VmRSS:   288 kB                  Kích thước resident set hiện tại
VmData:  148 kB                  Kích thước data segment
VmStk:   88 kB                   Kích thước stack
VmExe:   484 kB                  Kích thước text (executable code)
VmLib:   0 kB                    Kích thước code shared library
VmPTE:   12 kB                   Kích thước page table (từ 2.6.10)
Threads: 1                       Số thread trong thread group này
SigQ:    0/3067                  Signal đang xếp hàng hiện tại/tối đa (từ 2.6.12)
SigPnd:  0000000000000000        Signal đang chờ cho thread
ShdPnd:  0000000000000000        Signal đang chờ cho process (từ 2.6)
SigBlk:  0000000000000000        Signal bị block
SigIgn:  fffffffe5770d8fc        Signal bị bỏ qua
SigCgt:  00000000280b2603        Signal được bắt
CapInh:  0000000000000000        Inheritable capabilities
CapPrm:  00000000ffffffff        Permitted capabilities
CapEff:  00000000fffffeff        Effective capabilities
CapBnd:  00000000ffffffff        Capability bounding set (từ 2.6.26)
Cpus_allowed: 1                  CPU được phép, mask (từ 2.6.24)
Cpus_allowed_list: 0             Tương tự, định dạng list (từ 2.6.26)
Mems_allowed: 1                  Memory node được phép, mask (từ 2.6.24)
Mems_allowed_list: 0             Tương tự, định dạng list (từ 2.6.26)
voluntary_ctxt_switches: 6998    Voluntary context switch (từ 2.6.23)
nonvoluntary_ctxt_switches: 107  Involuntary context switch (từ 2.6.23)
Stack usage: 8 kB                High-water mark sử dụng stack (từ 2.6.32)
```

Kết quả trên lấy từ kernel 2.6.32. Như được chỉ ra bởi các chú thích `since` kèm theo kết quả file, định dạng của file này đã phát triển theo thời gian, với các trường mới được thêm vào (và trong một số trường hợp bị xóa) trong các phiên bản kernel khác nhau. (Ngoài các thay đổi Linux 2.6 đã nêu ở trên, Linux 2.4 đã thêm các trường `Tgid`, `TracerPid`, `FDSize`, và `Threads`.)

Thực tế là nội dung của file này đã thay đổi theo thời gian gợi lên một điểm chung về việc sử dụng các file `/proc`: khi các file này bao gồm nhiều mục, chúng ta nên parse chúng một cách phòng thủ — trong trường hợp này, tìm kiếm khớp trên dòng chứa chuỗi cụ thể (ví dụ: `PPid:`), thay vì xử lý file theo số dòng (logic).

Bảng 12-1 liệt kê một số file khác được tìm thấy trong mỗi thư mục `/proc/PID`.

**Bảng 12-1:** Các file được chọn trong mỗi thư mục `/proc/PID`

| File | Mô tả (thuộc tính process) |
|------|---------------------------|
| `cmdline` | Các đối số command-line phân tách bằng `\0` |
| `cwd` | Symbolic link đến current working directory |
| `environ` | Danh sách môi trường NAME=value, phân tách bằng `\0` |
| `exe` | Symbolic link đến file đang thực thi |
| `fd` | Directory chứa symbolic link đến các file được process này mở |
| `maps` | Memory mapping |
| `mem` | Virtual memory của process (phải `lseek()` đến offset hợp lệ trước khi I/O) |
| `mounts` | Mount point cho process này |
| `root` | Symbolic link đến root directory |
| `status` | Nhiều thông tin (ví dụ: process ID, credentials, memory usage, signal) |
| `task` | Chứa một subdirectory cho mỗi thread trong process (Linux 2.6) |

### Thư mục /proc/PID/fd

Thư mục `/proc/PID/fd` chứa một symbolic link cho mỗi file descriptor mà process có mở. Mỗi symbolic link này có tên khớp với số descriptor; ví dụ, `/proc/1968/1` là symbolic link đến standard output của process 1968. Xem Mục 5.11 để biết thêm thông tin.

Để tiện lợi, bất kỳ process nào cũng có thể truy cập thư mục `/proc/PID` của chính nó bằng symbolic link `/proc/self`.

### Thread: thư mục /proc/PID/task

Linux 2.4 đã thêm khái niệm thread group để hỗ trợ đúng mô hình POSIX threading. Vì một số thuộc tính là riêng biệt cho các thread trong một thread group, Linux 2.4 đã thêm subdirectory `task` dưới thư mục `/proc/PID`. Đối với mỗi thread trong process này, kernel cung cấp subdirectory tên `/proc/PID/task/TID`, trong đó `TID` là thread ID của thread. (Đây là con số giống như kết quả trả về của lời gọi `gettid()` trong thread.)

Dưới mỗi subdirectory `/proc/PID/task/TID` là một tập file và directory giống hệt những gì tìm thấy dưới `/proc/PID`. Vì các thread chia sẻ nhiều thuộc tính, nhiều thông tin trong các file này giống nhau cho mỗi thread trong process. Tuy nhiên, ở những nơi có ý nghĩa, các file này hiển thị thông tin riêng biệt cho mỗi thread. Ví dụ, trong các file `/proc/PID/task/TID/status` cho một thread group, `State`, `Pid`, `SigPnd`, `SigBlk`, `CapInh`, `CapPrm`, `CapEff`, và `CapBnd` là một số trường có thể khác biệt cho mỗi thread.

### 12.1.2 Thông Tin Hệ Thống Dưới /proc

Nhiều file và subdirectory dưới `/proc` cung cấp quyền truy cập vào thông tin toàn hệ thống. Một số được hiển thị trong Hình 12-1.

Bảng 12-2 tóm tắt mục đích chung của các subdirectory `/proc`.

**Bảng 12-2:** Mục đích của các subdirectory `/proc` được chọn

| Thư mục | Thông tin được hiển thị bởi các file trong thư mục này |
|---------|--------------------------------------------------------|
| `/proc` | Nhiều thông tin hệ thống |
| `/proc/net` | Thông tin trạng thái về networking và socket |
| `/proc/sys/fs` | Cài đặt liên quan đến file system |
| `/proc/sys/kernel` | Nhiều cài đặt kernel chung |
| `/proc/sys/net` | Cài đặt networking và socket |
| `/proc/sys/vm` | Cài đặt quản lý bộ nhớ (memory management) |
| `/proc/sysvipc` | Thông tin về các đối tượng System V IPC |

**Hình 12-1:** Các file và subdirectory được chọn dưới `/proc`

```
proc/
├── filesystems, kallsyms, loadavg, locks, meminfo,
│   partitions, stat, swaps, uptime, version, vmstat
├── net/
│   └── sockstat, sockstat6, tcp, tcp6, udp, udp6
├── sys/
│   ├── fs/
│   │   ├── file-max
│   │   ├── inotify/
│   │   └── mqueue/
│   ├── kernel/
│   │   └── acct, core_pattern, hostname, msgmax, msgmnb,
│   │       msgmni, pid_max, sem, shmall, shmmax, shmmni
│   ├── net/
│   │   ├── core/
│   │   │   └── somaxconn
│   │   ├── ipv4/
│   │   │   └── ip_local_port_range
│   │   ├── ipv6/
│   │   └── unix/
│   └── vm/
│       └── overcommit_memory, overcommit_ratio
└── sysvipc/
    └── msg, sem, shm
```

### 12.1.3 Truy Cập Các File /proc

Các file dưới `/proc` thường được truy cập bằng shell script (hầu hết các file `/proc` chứa nhiều giá trị có thể dễ dàng được parse bằng ngôn ngữ script như Python hay Perl). Ví dụ, ta có thể sửa đổi và xem nội dung của file `/proc` bằng các lệnh shell như sau:

```
# echo 100000 > /proc/sys/kernel/pid_max
# cat /proc/sys/kernel/pid_max
100000
```

Các file `/proc` cũng có thể được truy cập từ chương trình bằng các system call I/O file thông thường. Một số hạn chế áp dụng khi truy cập các file này:

- Một số file `/proc` là read-only; tức là, chúng chỉ tồn tại để hiển thị thông tin kernel và không thể dùng để sửa đổi thông tin đó. Điều này áp dụng cho hầu hết các file dưới thư mục `/proc/PID`.
- Một số file `/proc` chỉ có thể đọc bởi chủ sở hữu file (hoặc bởi process có đặc quyền). Ví dụ, tất cả các file dưới `/proc/PID` thuộc sở hữu của người dùng sở hữu process tương ứng, và trên một số file này (ví dụ: `/proc/PID/environ`), quyền đọc chỉ được cấp cho chủ sở hữu file.
- Ngoại trừ các file trong các subdirectory `/proc/PID`, hầu hết các file dưới `/proc` thuộc sở hữu của `root`, và các file có thể sửa đổi chỉ có thể được sửa bởi `root`.

### Truy cập file trong /proc/PID

Các thư mục `/proc/PID` là volatile. Mỗi thư mục này xuất hiện khi một process có process ID tương ứng được tạo và biến mất khi process đó kết thúc. Điều này có nghĩa là nếu ta xác định được một thư mục `/proc/PID` cụ thể tồn tại, ta cần xử lý sạch khả năng rằng process đã kết thúc, và thư mục `/proc/PID` tương ứng đã bị xóa, vào lúc ta cố gắng mở file trong thư mục đó.

### Chương trình ví dụ

Listing 12-1 minh họa cách đọc và sửa đổi file `/proc`. Chương trình này đọc và hiển thị nội dung của `/proc/sys/kernel/pid_max`. Nếu có đối số command-line, chương trình cập nhật file bằng giá trị đó. File này (mới trong Linux 2.6) chỉ định giới hạn trên cho process ID (Mục 6.2). Ví dụ:

```
$ su    Cần đặc quyền để cập nhật file pid_max
Password:
# ./procfs_pidmax 10000
Old value: 32768
/proc/sys/kernel/pid_max now contains 10000
```

**Listing 12-1:** Truy cập `/proc/sys/kernel/pid_max`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––– sysinfo/procfs_pidmax.c
#include <fcntl.h>
#include "tlpi_hdr.h"
#define MAX_LINE 100
int
main(int argc, char *argv[])
{
    int fd;
    char line[MAX_LINE];
    ssize_t n;
    fd = open("/proc/sys/kernel/pid_max", (argc > 1) ? O_RDWR : O_RDONLY);
    if (fd == -1)
        errExit("open");
    n = read(fd, line, MAX_LINE);
    if (n == -1)
        errExit("read");
    if (argc > 1)
        printf("Old value: ");
    printf("%.*s", (int) n, line);
    if (argc > 1) {
        if (write(fd, argv[1], strlen(argv[1])) != strlen(argv[1]))
            fatal("write() failed");
        system("echo /proc/sys/kernel/pid_max now contains "
               "`cat /proc/sys/kernel/pid_max`");
    }
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– sysinfo/procfs_pidmax.c
```

---

## 12.2 Định Danh Hệ Thống: uname()

System call `uname()` trả về nhiều thông tin nhận dạng về hệ thống host mà ứng dụng đang chạy trên đó, trong structure được trỏ bởi `utsbuf`.

```c
#include <sys/utsname.h>
int uname(struct utsname *utsbuf);
                                             Returns 0 on success, or –1 on error
```

Đối số `utsbuf` là pointer đến structure `utsname`, được định nghĩa như sau:

```c
#define _UTSNAME_LENGTH 65
struct utsname {
    char sysname[_UTSNAME_LENGTH];    /* Implementation name */
    char nodename[_UTSNAME_LENGTH];   /* Node name on network */
    char release[_UTSNAME_LENGTH];    /* Implementation release level */
    char version[_UTSNAME_LENGTH];    /* Release version level */
    char machine[_UTSNAME_LENGTH];    /* Hardware on which system
                                         is running */
#ifdef _GNU_SOURCE                    /* Following is Linux-specific */
    char domainname[_UTSNAME_LENGTH]; /* NIS domain name of host */
#endif
};
```

SUSv3 quy định `uname()`, nhưng để không xác định độ dài của các trường khác nhau trong structure `utsname`, chỉ yêu cầu rằng các chuỗi phải kết thúc bằng null byte. Trên Linux, các trường này mỗi trường dài 65 byte, bao gồm chỗ cho null byte kết thúc. Trên một số triển khai UNIX, các trường này ngắn hơn; trên các triển khai khác (ví dụ: Solaris), chúng có thể lên đến 257 byte.

Các trường `sysname`, `release`, `version`, và `machine` của structure `utsname` được kernel tự động đặt.

> Trên Linux, ba file trong thư mục `/proc/sys/kernel` cung cấp quyền truy cập vào cùng thông tin trả về trong các trường `sysname`, `release`, và `version` của structure `utsname`. Các file read-only này lần lượt là `ostype`, `osrelease`, và `version`. File khác, `/proc/version`, bao gồm cùng thông tin với các file này, và cũng bao gồm thông tin về bước biên dịch kernel (tức là tên người dùng thực hiện biên dịch, tên host mà biên dịch được thực hiện, và phiên bản gcc được dùng).

Trường `nodename` trả về giá trị được đặt bằng system call `sethostname()` (xem manual page để biết chi tiết về system call này). Thường thì tên này giống như tiền tố hostname từ DNS domain name của hệ thống.

Trường `domainname` trả về giá trị được đặt bằng system call `setdomainname()` (xem manual page để biết chi tiết). Đây là NIS (Network Information Services) domain name của host (không giống với DNS domain name của host).

System call `gethostname()`, là hàm đảo của `sethostname()`, lấy system hostname. System hostname cũng có thể xem và đặt bằng lệnh `hostname(1)` và file Linux-specific `/proc/hostname`.

System call `getdomainname()`, là hàm đảo của `setdomainname()`, lấy NIS domain name. NIS domain name cũng có thể xem và đặt bằng lệnh `domainname(1)` và file Linux-specific `/proc/domainname`.

Các system call `sethostname()` và `setdomainname()` hiếm khi được dùng trong chương trình ứng dụng. Thông thường, hostname và NIS domain name được thiết lập lúc boot bởi startup script.

Chương trình trong Listing 12-2 hiển thị thông tin trả về bởi `uname()`. Ví dụ kết quả:

```
$ ./t_uname
   Node name:   tekapo
   System name: Linux
   Release:     2.6.30-default
   Version:     #3 SMP Fri Jul 17 10:25:00 CEST 2009
   Machine:     i686
   Domain name:
```

**Listing 12-2:** Sử dụng `uname()`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– sysinfo/t_uname.c
#define _GNU_SOURCE
#include <sys/utsname.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    struct utsname uts;
    if (uname(&uts) == -1)
        errExit("uname");
    printf("Node name:   %s\n", uts.nodename);
    printf("System name: %s\n", uts.sysname);
    printf("Release:     %s\n", uts.release);
    printf("Version:     %s\n", uts.version);
    printf("Machine:     %s\n", uts.machine);
#ifdef _GNU_SOURCE
    printf("Domain name: %s\n", uts.domainname);
#endif
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– sysinfo/t_uname.c
```

---

## 12.3 Tóm Tắt

File system `/proc` cung cấp nhiều thông tin kernel cho các chương trình ứng dụng. Mỗi subdirectory `/proc/PID` chứa các file và subdirectory cung cấp thông tin về process có ID khớp với `PID`. Nhiều file và thư mục khác dưới `/proc` hiển thị thông tin toàn hệ thống mà các chương trình có thể đọc và, trong một số trường hợp, sửa đổi.

System call `uname()` cho phép ta khám phá triển khai UNIX và loại máy mà ứng dụng đang chạy trên đó.

---

## 12.4 Bài Tập

- **12-1.** Viết chương trình liệt kê process ID và tên lệnh cho tất cả các process đang được chạy bởi người dùng có tên được chỉ định trong đối số command-line. (Hàm `userIdFromName()` từ Listing 8-1 có thể hữu ích.) Điều này có thể được thực hiện bằng cách kiểm tra các dòng `Name:` và `Uid:` của tất cả các file `/proc/PID/status` trên hệ thống. Duyệt qua tất cả các thư mục `/proc/PID` trên hệ thống đòi hỏi dùng `readdir(3)`, được mô tả trong Mục 18.8. Đảm bảo chương trình của bạn xử lý đúng khả năng rằng thư mục `/proc/PID` biến mất giữa lúc chương trình xác định thư mục tồn tại và lúc nó cố gắng mở file `/proc/PID/status` tương ứng.

- **12-2.** Viết chương trình vẽ cây hiển thị quan hệ phân cấp cha-con của tất cả các process trên hệ thống, truy ngược về đến `init`. Với mỗi process, chương trình nên hiển thị process ID và lệnh đang thực thi. Đầu ra của chương trình nên tương tự như được tạo bởi `pstree(1)`, mặc dù không cần tinh vi như vậy. Process cha của mỗi process trên hệ thống có thể được tìm bằng cách kiểm tra dòng `PPid:` của tất cả các file `/proc/PID/status`. Cẩn thận xử lý khả năng process cha (và do đó thư mục `/proc/PID` của nó) biến mất trong quá trình quét tất cả thư mục `/proc/PID`.

- **12-3.** Viết chương trình liệt kê tất cả các process đang mở một pathname file cụ thể. Điều này có thể đạt được bằng cách kiểm tra nội dung của tất cả các symbolic link `/proc/PID/fd/*`. Điều này sẽ đòi hỏi vòng lặp lồng nhau dùng `readdir(3)` để quét tất cả các thư mục `/proc/PID`, và sau đó nội dung của tất cả các mục `/proc/PID/fd` trong mỗi thư mục `/proc/PID`. Để đọc nội dung của symbolic link `/proc/PID/fd/n` đòi hỏi dùng `readlink()`, được mô tả trong Mục 18.5.
