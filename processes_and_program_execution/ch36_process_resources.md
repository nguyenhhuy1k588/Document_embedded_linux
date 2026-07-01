## Chương 36
# **TÀI NGUYÊN PROCESS**

Mỗi process tiêu thụ các tài nguyên hệ thống như bộ nhớ và thời gian CPU. Chương này trình bày các system call liên quan đến tài nguyên. Chúng ta bắt đầu với system call `getrusage()`, cho phép một process theo dõi các tài nguyên mà nó đã sử dụng hoặc các tài nguyên mà các process con của nó đã sử dụng. Tiếp theo, chúng ta xem xét các system call `setrlimit()` và `getrlimit()`, có thể được dùng để thay đổi và truy xuất các giới hạn về mức tiêu thụ tài nguyên của process gọi.

### **36.1 Mức Sử Dụng Tài Nguyên của Process**

System call `getrusage()` truy xuất các thống kê về nhiều tài nguyên hệ thống khác nhau được sử dụng bởi process gọi hoặc bởi tất cả các process con của nó.

```
#include <sys/resource.h>
```

int **getrusage**(int who, struct rusage \*res\_usage);

Trả về 0 nếu thành công, hoặc –1 nếu có lỗi

Đối số `who` chỉ định (các) process mà thông tin sử dụng tài nguyên cần được truy xuất. Nó nhận một trong các giá trị sau:

```
RUSAGE_SELF
```

Trả về thông tin về process gọi.

#### RUSAGE\_CHILDREN

Trả về thông tin về tất cả các process con của process gọi đã kết thúc và đã được chờ đợi.

```
RUSAGE_THREAD (từ Linux 2.6.26)
```

Trả về thông tin về thread gọi. Giá trị này là đặc thù của Linux.

Đối số `res_usage` là một pointer đến một cấu trúc có kiểu `rusage`, được định nghĩa như trong [Listing 36-1](#page-21-0).

<span id="page-21-0"></span>**Listing 36-1:** Định nghĩa của cấu trúc `rusage`

```
struct rusage {
 struct timeval ru_utime; /* Thời gian CPU ở user mode đã dùng */
 struct timeval ru_stime; /* Thời gian CPU ở kernel mode đã dùng */
 long ru_maxrss; /* Kích thước tối đa của resident set (kilobytes)
 [dùng từ Linux 2.6.32] */
 long ru_ixrss; /* Kích thước bộ nhớ text chia sẻ tích phân
 (kilobyte-seconds) [không dùng] */
 long ru_idrss; /* Bộ nhớ data không chia sẻ đã dùng (tích phân)
 (kilobyte-seconds) [không dùng] */
 long ru_isrss; /* Bộ nhớ stack không chia sẻ đã dùng (tích phân)
 (kilobyte-seconds) [không dùng] */
 long ru_minflt; /* Soft page fault (không cần I/O) */
 long ru_majflt; /* Hard page fault (cần I/O) */
 long ru_nswap; /* Swap ra khỏi bộ nhớ vật lý [không dùng] */
 long ru_inblock; /* Thao tác đọc theo block qua file
 system [dùng từ Linux 2.6.22] */
 long ru_oublock; /* Thao tác ghi theo block qua file
 system [dùng từ Linux 2.6.22] */
 long ru_msgsnd; /* Số IPC message đã gửi [không dùng] */
 long ru_msgrcv; /* Số IPC message đã nhận [không dùng] */
 long ru_nsignals; /* Số signal đã nhận [không dùng] */
 long ru_nvcsw; /* Context switch tự nguyện (process
 nhường CPU trước khi hết time slice)
 [dùng từ Linux 2.6] */
 long ru_nivcsw; /* Context switch không tự nguyện (process
 có độ ưu tiên cao hơn sẵn sàng chạy hoặc
 hết time slice) [dùng từ Linux 2.6] */
};
```

Như được chỉ ra trong các chú thích tại [Listing 36-1](#page-21-0), trên Linux, nhiều trường trong cấu trúc `rusage` không được `getrusage()` (hoặc `wait3()` và `wait4()`) điền vào, hoặc chỉ được điền vào từ các phiên bản kernel mới hơn. Một số trường không được dùng trên Linux lại được dùng trên các implementation UNIX khác. Các trường này được cung cấp trên Linux để nếu chúng được implement vào một ngày nào đó trong tương lai, cấu trúc `rusage` không cần thay đổi theo cách sẽ làm hỏng các binary ứng dụng hiện có.

Mặc dù `getrusage()` xuất hiện trên hầu hết các implementation UNIX, nó chỉ được đặc tả yếu trong SUSv3 (chỉ đặc tả các trường `ru_utime` và `ru_stime`). Một phần, điều này là vì ý nghĩa của nhiều thông tin trong cấu trúc `rusage` phụ thuộc vào implementation.

Các trường `ru_utime` và `ru_stime` là các cấu trúc kiểu `timeval` (Mục 10.1), trả về số giây và microsecond của thời gian CPU mà một process đã tiêu thụ trong user mode và kernel mode, tương ứng. (Thông tin tương tự được truy xuất bởi system call `times()` được mô tả trong Mục 10.7.)

> Các file `/proc/PID/stat` đặc thù của Linux hiển thị một số thông tin sử dụng tài nguyên (thời gian CPU và page fault) về tất cả các process trên hệ thống. Xem trang manual `proc(5)` để biết thêm chi tiết.

Cấu trúc `rusage` được trả về bởi thao tác `RUSAGE_CHILDREN` của `getrusage()` bao gồm các thống kê sử dụng tài nguyên của tất cả các process hậu duệ của process gọi. Ví dụ, nếu chúng ta có ba process liên quan theo kiểu cha, con, và cháu, thì khi process con thực hiện `wait()` trên process cháu, các giá trị sử dụng tài nguyên của process cháu được cộng vào các giá trị `RUSAGE_CHILDREN` của process con; khi process cha thực hiện `wait()` cho process con, các giá trị sử dụng tài nguyên của cả process con lẫn process cháu được cộng vào các giá trị `RUSAGE_CHILDREN` của process cha. Ngược lại, nếu process con không thực hiện `wait()` trên process cháu, thì các mức sử dụng tài nguyên của process cháu không được ghi lại trong các giá trị `RUSAGE_CHILDREN` của process cha.

Đối với thao tác `RUSAGE_CHILDREN`, trường `ru_maxrss` trả về kích thước resident set tối đa trong số tất cả các process hậu duệ của process gọi (thay vì tổng của tất cả các hậu duệ).

> SUSv3 chỉ định rằng nếu `SIGCHLD` đang bị bỏ qua (để các process con không biến thành zombie mà có thể được chờ đợi), thì các thống kê về process con không nên được cộng vào các giá trị được trả về bởi `RUSAGE_CHILDREN`. Tuy nhiên, như đã lưu ý trong Mục 26.3.3, trong các kernel trước 2.6.9, Linux không tuân theo yêu cầu này — nếu `SIGCHLD` bị bỏ qua, thì các giá trị sử dụng tài nguyên cho các process con đã chết được bao gồm trong các giá trị được trả về cho `RUSAGE_CHILDREN`.

# **36.2 Giới Hạn Tài Nguyên của Process**

Mỗi process có một tập hợp các giới hạn tài nguyên có thể được sử dụng để hạn chế lượng tài nguyên hệ thống khác nhau mà process có thể tiêu thụ. Ví dụ, chúng ta có thể muốn đặt giới hạn tài nguyên cho một process trước khi thực thi một chương trình tùy ý, nếu chúng ta lo ngại rằng nó có thể tiêu thụ tài nguyên quá mức. Chúng ta có thể đặt giới hạn tài nguyên của shell bằng cách sử dụng lệnh tích hợp `ulimit` (hoặc `limit` trong C shell). Các giới hạn này được kế thừa bởi các process mà shell tạo ra để thực thi các lệnh của người dùng.

> Từ kernel 2.6.24, file `/proc/PID/limits` đặc thù của Linux có thể được dùng để xem tất cả các giới hạn tài nguyên của bất kỳ process nào. File này thuộc sở hữu của real user ID của process tương ứng và quyền của nó chỉ cho phép đọc bởi user ID đó (hoặc bởi một privileged process).

Các system call `getrlimit()` và `setrlimit()` cho phép một process lấy và thay đổi các giới hạn tài nguyên của nó.

```
#include <sys/resource.h>
int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);
                                         Cả hai trả về 0 nếu thành công, hoặc –1 nếu có lỗi
```

Đối số `resource` xác định giới hạn tài nguyên cần được truy xuất hoặc thay đổi. Đối số `rlim` được dùng để trả về các giá trị giới hạn tài nguyên (`getrlimit()`) hoặc để chỉ định các giá trị giới hạn tài nguyên mới (`setrlimit()`), và là một pointer đến một cấu trúc có hai trường:

```
struct rlimit {
 rlim_t rlim_cur; /* Soft limit (giới hạn thực tế của process) */
 rlim_t rlim_max; /* Hard limit (trần cho rlim_cur) */
};
```

Các trường này tương ứng với hai giới hạn liên quan cho một tài nguyên: soft limit (`rlim_cur`) và hard limit (`rlim_max`). (Kiểu dữ liệu `rlim_t` là một kiểu số nguyên.) Soft limit quyết định lượng tài nguyên có thể được tiêu thụ bởi process. Một process có thể điều chỉnh soft limit thành bất kỳ giá trị nào từ 0 đến hard limit. Đối với hầu hết các tài nguyên, mục đích duy nhất của hard limit là cung cấp giới hạn trần này cho soft limit. Một privileged process (`CAP_SYS_RESOURCE`) có thể điều chỉnh hard limit theo bất kỳ hướng nào (miễn là giá trị của nó vẫn lớn hơn soft limit), nhưng một unprivileged process chỉ có thể điều chỉnh hard limit xuống giá trị thấp hơn (không thể hoàn tác). Giá trị `RLIM_INFINITY` trong `rlim_cur` hoặc `rlim_max` có nghĩa là vô hạn (không có giới hạn đối với tài nguyên), cả khi được truy xuất qua `getrlimit()` lẫn khi được đặt qua `setrlimit()`.

Trong hầu hết các trường hợp, giới hạn tài nguyên được áp dụng cho cả privileged và unprivileged process. Chúng được kế thừa bởi các process con được tạo qua `fork()` và được bảo toàn qua `exec()`.

Các giá trị có thể được chỉ định cho đối số `resource` của `getrlimit()` và `setrlimit()` được tóm tắt trong [Bảng 36-1](#page-24-0) và được trình bày chi tiết trong Mục [36.3.](#page-27-1)

Mặc dù giới hạn tài nguyên là một thuộc tính per-process, trong một số trường hợp, giới hạn được đo không chỉ dựa trên mức tiêu thụ tài nguyên của process đó, mà còn dựa trên tổng tài nguyên được tiêu thụ bởi tất cả các process có cùng real user ID. Giới hạn `RLIMIT_NPROC`, đặt giới hạn về số process có thể được tạo, là một ví dụ điển hình về lý do của cách tiếp cận này. Áp dụng giới hạn này chỉ dựa trên số process con mà process đó đã tạo sẽ không hiệu quả, vì mỗi process con mà process đó tạo ra cũng sẽ có thể tạo thêm các process con, những process con này lại có thể tạo thêm process con, và cứ tiếp tục như vậy. Thay vào đó, giới hạn được đo dựa trên số lượng tất cả các process có cùng real user ID. Tuy nhiên, lưu ý rằng giới hạn tài nguyên chỉ được kiểm tra trong các process mà nó đã được đặt (tức là, bản thân process và các hậu duệ của nó, những process kế thừa giới hạn). Nếu một process khác thuộc cùng real user ID chưa đặt giới hạn (tức là, giới hạn là vô hạn) hoặc đã đặt giới hạn khác, thì khả năng tạo process con của process đó sẽ được kiểm tra theo giới hạn mà nó đã đặt.

Như chúng ta mô tả từng giới hạn tài nguyên bên dưới, chúng ta lưu ý những giới hạn được đo dựa trên tài nguyên được tiêu thụ bởi tất cả các process có cùng real user ID. Nếu không được chỉ định khác, thì giới hạn tài nguyên chỉ được đo dựa trên mức tiêu thụ tài nguyên của chính process.

Hãy lưu ý rằng, trong nhiều trường hợp, các lệnh shell để lấy và đặt giới hạn tài nguyên (`ulimit` trong bash và Korn shell, và `limit` trong C shell) sử dụng các đơn vị khác với những đơn vị được sử dụng trong `getrlimit()` và `setrlimit()`. Ví dụ, các lệnh shell thường biểu thị các giới hạn về kích thước của các phân đoạn bộ nhớ khác nhau theo kilobyte.

<span id="page-24-0"></span>**Bảng 36-1:** Các giá trị `resource` cho `getrlimit()` và `setrlimit()`

| resource          | Giới hạn về                                                     | SUSv3 |
|-------------------|-----------------------------------------------------------------|-------|
| RLIMIT_AS         | Kích thước virtual memory của process (byte)                    | •     |
| RLIMIT_CORE       | Kích thước file core dump (byte)                                | •     |
| RLIMIT_CPU        | Thời gian CPU (giây)                                            | •     |
| RLIMIT_DATA       | Phân đoạn data của process (byte)                               | •     |
| RLIMIT_FSIZE      | Kích thước file (byte)                                          | •     |
| RLIMIT_MEMLOCK    | Bộ nhớ bị khóa (byte)                                          |       |
| RLIMIT_MSGQUEUE   | Byte được cấp phát cho POSIX message queue của real user ID     |       |
|                   | (từ Linux 2.6.8)                                                |       |
| RLIMIT_NICE       | Giá trị nice (từ Linux 2.6.12)                                  |       |
| RLIMIT_NOFILE     | Số file descriptor tối đa cộng thêm một                        | •     |
| RLIMIT_NPROC      | Số process cho real user ID                                     |       |
| RLIMIT_RSS        | Kích thước resident set (byte; chưa được implement)             |       |
| RLIMIT_RTPRIO     | Độ ưu tiên realtime scheduling (từ Linux 2.6.12)                |       |
| RLIMIT_RTTIME     | Thời gian CPU realtime (microsecond; từ Linux 2.6.25)           |       |
| RLIMIT_SIGPENDING | Số signal trong hàng đợi cho real user ID (từ                   |       |
|                   | Linux 2.6.8)                                                    |       |
| RLIMIT_STACK      | Kích thước phân đoạn stack (byte)                               | •     |

### **Chương trình ví dụ**

Trước khi đi vào chi tiết của từng giới hạn tài nguyên, chúng ta xem xét một ví dụ đơn giản về việc sử dụng giới hạn tài nguyên. [Listing 36-2](#page-25-0) định nghĩa hàm `printRlimit()`, hiển thị một thông báo, cùng với soft limit và hard limit cho một tài nguyên được chỉ định.

> Kiểu dữ liệu `rlim_t` thường được biểu diễn theo cùng cách với `off_t`, để xử lý biểu diễn của `RLIMIT_FSIZE`, giới hạn tài nguyên kích thước file. Vì lý do này, khi in các giá trị `rlim_t` (như trong [Listing 36-2](#page-25-0)), chúng ta ép kiểu chúng thành `long long` và sử dụng specifier `%lld` của `printf()`, như được giải thích trong Mục 5.10.

Chương trình trong [Listing 36-3](#page-25-1) gọi `setrlimit()` để đặt soft limit và hard limit về số process mà một người dùng có thể tạo (`RLIMIT_NPROC`), sử dụng hàm `printRlimit()` của [Listing 36-2](#page-25-0) để hiển thị các giới hạn trước và sau khi thay đổi, và sau đó tạo càng nhiều process càng tốt. Khi chúng ta chạy chương trình này, đặt soft limit thành 30 và hard limit thành 100, chúng ta thấy như sau:

```
$ ./rlimit_nproc 30 100
Initial maximum process limits: soft=1024; hard=1024
New maximum process limits: soft=30; hard=100
Child 1 (PID=15674) started
Child 2 (PID=15675) started
Child 3 (PID=15676) started
Child 4 (PID=15677) started
ERROR [EAGAIN Resource temporarily unavailable] fork
```

Trong ví dụ này, chương trình chỉ tạo được 4 process mới, vì đã có 26 process đang chạy cho người dùng này.

<span id="page-25-0"></span>**Listing 36-2:** Hiển thị giới hạn tài nguyên của process

```
–––––––––––––––––––––––––––––––––––––––––––––––––––– procres/print_rlimit.c
#include <sys/resource.h>
#include "print_rlimit.h" /* Declares function defined here */
#include "tlpi_hdr.h"
int /* Print 'msg' followed by limits for 'resource' */
printRlimit(const char *msg, int resource)
{
 struct rlimit rlim;
 if (getrlimit(resource, &rlim) == -1)
 return -1;
 printf("%s soft=", msg);
 if (rlim.rlim_cur == RLIM_INFINITY)
 printf("infinite");
#ifdef RLIM_SAVED_CUR /* Not defined on some implementations */
 else if (rlim.rlim_cur == RLIM_SAVED_CUR)
 printf("unrepresentable");
#endif
 else
 printf("%lld", (long long) rlim.rlim_cur);
 printf("; hard=");
 if (rlim.rlim_max == RLIM_INFINITY)
 printf("infinite\n");
#ifdef RLIM_SAVED_MAX /* Not defined on some implementations */
 else if (rlim.rlim_max == RLIM_SAVED_MAX)
 printf("unrepresentable");
#endif
 else
 printf("%lld\n", (long long) rlim.rlim_max);
 return 0;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– procres/print_rlimit.c
Listing 36-3: Đặt giới hạn tài nguyên RLIMIT_NPROC
–––––––––––––––––––––––––––––––––––––––––––––––––––– procres/rlimit_nproc.c
#include <sys/resource.h>
#include "print_rlimit.h" /* Declaration of printRlimit() */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 struct rlimit rl;
 int j;
 pid_t childPid;
```

```
 if (argc < 2 || argc > 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s soft-limit [hard-limit]\n", argv[0]);
 printRlimit("Initial maximum process limits: ", RLIMIT_NPROC);
 /* Set new process limits (hard == soft if not specified) */
 rl.rlim_cur = (argv[1][0] == 'i') ? RLIM_INFINITY :
 getInt(argv[1], 0, "soft-limit");
 rl.rlim_max = (argc == 2) ? rl.rlim_cur :
 (argv[2][0] == 'i') ? RLIM_INFINITY :
 getInt(argv[2], 0, "hard-limit");
 if (setrlimit(RLIMIT_NPROC, &rl) == -1)
 errExit("setrlimit");
 printRlimit("New maximum process limits: ", RLIMIT_NPROC);
 /* Create as many children as possible */
 for (j = 1; ; j++) {
 switch (childPid = fork()) {
 case -1: errExit("fork");
 case 0: _exit(EXIT_SUCCESS); /* Child */
 default: /* Parent: display message about each new child
 and let the resulting zombies accumulate */
 printf("Child %d (PID=%ld) started\n", j, (long) childPid);
 break;
 }
 }
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– procres/rlimit_nproc.c
```

#### **Giá trị giới hạn không thể biểu diễn**

Trong một số môi trường lập trình, kiểu dữ liệu `rlim_t` có thể không thể biểu diễn toàn bộ phạm vi giá trị có thể được duy trì cho một giới hạn tài nguyên cụ thể. Điều này có thể xảy ra trên một hệ thống cung cấp nhiều môi trường lập trình trong đó kích thước của kiểu dữ liệu `rlim_t` khác nhau. Những hệ thống như vậy có thể xuất hiện nếu một môi trường biên dịch large-file với `off_t` 64-bit được thêm vào một hệ thống trong đó `off_t` truyền thống là 32 bit. (Trong mỗi môi trường, `rlim_t` sẽ có cùng kích thước với `off_t`.) Điều này dẫn đến tình huống mà một chương trình với `rlim_t` nhỏ có thể, sau khi được exec bởi một chương trình với `off_t` 64-bit, kế thừa một giới hạn tài nguyên (ví dụ: giới hạn kích thước file) lớn hơn giá trị `rlim_t` tối đa.

Để hỗ trợ các ứng dụng portable xử lý khả năng một giới hạn tài nguyên có thể không thể biểu diễn, SUSv3 chỉ định hai hằng số để chỉ ra các giá trị giới hạn không thể biểu diễn: `RLIM_SAVED_CUR` và `RLIM_SAVED_MAX`. Nếu soft limit tài nguyên không thể biểu diễn trong `rlim_t`, thì `getrlimit()` sẽ trả về `RLIM_SAVED_CUR` trong trường `rlim_cur`. `RLIM_SAVED_MAX` thực hiện chức năng tương tự cho hard limit không thể biểu diễn được trả về trong trường `rlim_max`.

Nếu tất cả các giá trị giới hạn tài nguyên có thể có thể được biểu diễn trong `rlim_t`, thì SUSv3 cho phép một implementation định nghĩa `RLIM_SAVED_CUR` và `RLIM_SAVED_MAX` giống như `RLIM_INFINITY`. Đây là cách các hằng số này được định nghĩa trên Linux, ngụ ý rằng tất cả các giá trị giới hạn tài nguyên có thể có thể được biểu diễn trong `rlim_t`. Tuy nhiên, điều này không đúng trên các kiến trúc 32-bit như x86-32. Trên những kiến trúc đó, trong môi trường biên dịch large-file (tức là đặt macro kiểm tra tính năng `_FILE_OFFSET_BITS` thành 64 như được mô tả trong Mục 5.10), định nghĩa glibc của `rlim_t` rộng 64 bit, nhưng kiểu dữ liệu kernel để biểu diễn giới hạn tài nguyên là `unsigned long`, chỉ rộng 32 bit. Các phiên bản glibc hiện tại giải quyết tình huống này như sau: nếu một chương trình được biên dịch với `_FILE_OFFSET_BITS=64` cố gắng đặt giới hạn tài nguyên thành một giá trị lớn hơn có thể được biểu diễn trong `unsigned long` 32-bit, thì wrapper glibc cho `setrlimit()` âm thầm chuyển đổi giá trị thành `RLIM_INFINITY`. Nói cách khác, việc đặt giới hạn tài nguyên được yêu cầu không được thực hiện.

> Vì các tiện ích xử lý file thường được biên dịch với `_FILE_OFFSET_BITS=64` trong nhiều phiên bản x86-32, việc không thực thi các giới hạn tài nguyên lớn hơn giá trị có thể được biểu diễn trong 32 bit là một vấn đề có thể ảnh hưởng không chỉ đến các nhà lập trình ứng dụng, mà còn đến người dùng cuối.

> Có thể lập luận rằng wrapper `setrlimit()` của glibc nên báo lỗi nếu giới hạn tài nguyên được yêu cầu vượt quá khả năng của `unsigned long` 32-bit. Tuy nhiên, vấn đề cơ bản là giới hạn kernel, và hành vi được mô tả trong phần chính là cách tiếp cận mà các nhà phát triển glibc đã chọn để giải quyết nó.

# <span id="page-27-1"></span>**36.3 Chi Tiết Về Các Giới Hạn Tài Nguyên Cụ Thể**

<span id="page-27-0"></span>Trong phần này, chúng ta cung cấp thông tin chi tiết về từng giới hạn tài nguyên có sẵn trên Linux, lưu ý những giới hạn đặc thù của Linux.

#### **RLIMIT\_AS**

Giới hạn `RLIMIT_AS` chỉ định kích thước tối đa cho virtual memory (address space) của process, tính theo byte. Các nỗ lực (`brk()`, `sbrk()`, `mmap()`, `mremap()`, và `shmat()`) để vượt quá giới hạn này đều thất bại với lỗi `ENOMEM`. Trong thực tế, nơi phổ biến nhất mà một chương trình có thể gặp phải giới hạn này là trong các lời gọi đến các hàm trong gói `malloc`, sử dụng `sbrk()` và `mmap()`. Khi gặp phải giới hạn này, việc phát triển stack cũng có thể thất bại với các hậu quả được liệt kê bên dưới cho `RLIMIT_STACK`.

#### **RLIMIT\_CORE**

Giới hạn `RLIMIT_CORE` chỉ định kích thước tối đa, tính theo byte, cho các file core dump được tạo khi một process bị kết thúc bởi một số signal nhất định (Mục 22.1). Việc tạo file core dump sẽ dừng tại giới hạn này. Chỉ định giới hạn là 0 ngăn việc tạo file core dump, điều này đôi khi hữu ích vì file core dump có thể rất lớn và người dùng thường không biết phải làm gì với chúng. Một lý do khác để tắt core dump là bảo mật — để ngăn nội dung bộ nhớ của một chương trình bị đổ ra đĩa. Nếu giới hạn `RLIMIT_FSIZE` thấp hơn giới hạn này, file core dump bị giới hạn ở `RLIMIT_FSIZE` byte.

#### **RLIMIT\_CPU**

Giới hạn `RLIMIT_CPU` chỉ định số giây tối đa của thời gian CPU (trong cả system mode và user mode) mà process có thể sử dụng. SUSv3 yêu cầu signal `SIGXCPU` được gửi đến process khi đạt đến soft limit, nhưng để các chi tiết khác không được chỉ định. (Hành động mặc định cho `SIGXCPU` là kết thúc một process kèm core dump.) Có thể thiết lập handler cho `SIGXCPU` thực hiện bất kỳ xử lý nào mong muốn và sau đó trả quyền điều khiển cho chương trình chính. Sau đó, (trên Linux) `SIGXCPU` được gửi một lần mỗi giây của thời gian CPU đã tiêu thụ. Nếu process tiếp tục thực thi cho đến khi đạt đến hard CPU limit, thì kernel gửi cho nó một signal `SIGKILL`, luôn luôn kết thúc process.

Các implementation UNIX khác nhau về chi tiết cách chúng xử lý các process tiếp tục tiêu thụ thời gian CPU sau khi xử lý signal `SIGXCPU`. Hầu hết tiếp tục gửi `SIGXCPU` theo các khoảng thời gian đều đặn. Nếu muốn sử dụng signal này một cách portable, chúng ta nên lập trình ứng dụng sao cho khi nhận signal này lần đầu tiên, nó thực hiện bất kỳ dọn dẹp nào cần thiết và kết thúc. (Alternatively, chương trình có thể thay đổi giới hạn tài nguyên sau khi nhận signal.)

#### **RLIMIT\_DATA**

Giới hạn `RLIMIT_DATA` chỉ định kích thước tối đa, tính theo byte, của phân đoạn data của process (tổng của các phân đoạn initialized data, uninitialized data, và heap được mô tả trong Mục 6.3). Các nỗ lực (`sbrk()` và `brk()`) để mở rộng phân đoạn data (program break) vượt quá giới hạn này thất bại với lỗi `ENOMEM`. Cũng như `RLIMIT_AS`, nơi phổ biến nhất mà một chương trình có thể gặp phải giới hạn này là trong các lời gọi đến các hàm trong gói `malloc`.

### **RLIMIT\_FSIZE**

Giới hạn `RLIMIT_FSIZE` chỉ định kích thước tối đa của các file mà process có thể tạo, tính theo byte. Nếu một process cố gắng mở rộng file vượt quá soft limit, nó sẽ được gửi signal `SIGXFSZ`, và system call (ví dụ: `write()` hoặc `truncate()`) thất bại với lỗi `EFBIG`. Hành động mặc định cho `SIGXFSZ` là kết thúc một process và tạo core dump. Thay vào đó, có thể bắt signal này và trả quyền điều khiển cho chương trình chính. Tuy nhiên, bất kỳ nỗ lực nào khác để mở rộng file sẽ tạo ra cùng signal và lỗi đó.

#### **RLIMIT\_MEMLOCK**

Giới hạn `RLIMIT_MEMLOCK` (bắt nguồn từ BSD; không có trong SUSv3 và chỉ có trên Linux và BSD) chỉ định số byte tối đa của virtual memory mà một process có thể khóa vào bộ nhớ vật lý, để ngăn bộ nhớ bị swap ra. Giới hạn này ảnh hưởng đến các system call `mlock()` và `mlockall()`, và các tùy chọn khóa cho các system call `mmap()` và `shmctl()`. Chúng ta mô tả chi tiết trong Mục 50.2.

Nếu cờ `MCL_FUTURE` được chỉ định khi gọi `mlockall()`, thì giới hạn `RLIMIT_MEMLOCK` cũng có thể khiến các lời gọi sau đó đến `brk()`, `sbrk()`, `mmap()`, hoặc `mremap()` thất bại.

#### **RLIMIT\_MSGQUEUE**

Giới hạn `RLIMIT_MSGQUEUE` (đặc thù của Linux; từ Linux 2.6.8) chỉ định số byte tối đa có thể được cấp phát cho POSIX message queue cho real user ID của process gọi. Khi một POSIX message queue được tạo bằng `mq_open()`, byte được trừ từ giới hạn này theo công thức sau:

```
bytes = attr.mq_maxmsg * sizeof(struct msg_msg *) +
 attr.mq_maxmsg * attr.mq_msgsize;
```

Trong công thức này, `attr` là cấu trúc `mq_attr` được truyền dưới dạng đối số thứ tư cho `mq_open()`. Số hạng bao gồm `sizeof(struct msg_msg *)` đảm bảo rằng người dùng không thể xếp vào hàng đợi một số lượng không giới hạn các message có độ dài bằng không. (Cấu trúc `msg_msg` là một kiểu dữ liệu được sử dụng nội bộ bởi kernel.) Điều này cần thiết vì, mặc dù các message có độ dài bằng không không chứa dữ liệu, chúng vẫn tiêu thụ một số bộ nhớ hệ thống cho overhead quản lý.

Giới hạn `RLIMIT_MSGQUEUE` chỉ ảnh hưởng đến process gọi. Các process khác thuộc người dùng này không bị ảnh hưởng trừ khi chúng cũng đặt giới hạn này hoặc kế thừa nó.

#### **RLIMIT\_NICE**

Giới hạn `RLIMIT_NICE` (đặc thù của Linux; từ Linux 2.6.12) chỉ định giới hạn trần về giá trị nice có thể được đặt cho process này bằng cách sử dụng `sched_setscheduler()` và `nice()`. Giới hạn trần được tính là `20 – rlim_cur`, trong đó `rlim_cur` là soft limit tài nguyên `RLIMIT_NICE` hiện tại. Tham khảo Mục [35.1](#page-0-1) để biết thêm chi tiết.

#### **RLIMIT\_NOFILE**

Giới hạn `RLIMIT_NOFILE` chỉ định một số lớn hơn số file descriptor tối đa mà một process có thể cấp phát thêm một. Các nỗ lực (ví dụ: `open()`, `pipe()`, `socket()`, `accept()`, `shm_open()`, `dup()`, `dup2()`, `fcntl(F_DUPFD)`, và `epoll_create()`) cấp phát các descriptor vượt quá giới hạn này sẽ thất bại. Trong hầu hết các trường hợp, lỗi là `EMFILE`, nhưng đối với `dup2(fd, newfd)` thì là `EBADF`, và đối với `fcntl(fd, F_DUPFD, newfd)` với `newfd` lớn hơn hoặc bằng giới hạn, thì là `EINVAL`.

Các thay đổi đối với giới hạn `RLIMIT_NOFILE` được phản ánh trong giá trị được trả về bởi `sysconf(_SC_OPEN_MAX)`. SUSv3 cho phép, nhưng không yêu cầu, một implementation trả về các giá trị khác nhau cho một lời gọi `sysconf(_SC_OPEN_MAX)` trước và sau khi thay đổi giới hạn `RLIMIT_NOFILE`; các implementation khác có thể không hoạt động giống Linux trên điểm này.

> SUSv3 nêu rằng nếu một ứng dụng đặt soft hoặc hard limit `RLIMIT_NOFILE` thành một giá trị nhỏ hơn hoặc bằng số file descriptor cao nhất mà process hiện đang mở, hành vi không mong đợi có thể xảy ra.

> Trên Linux, chúng ta có thể kiểm tra file descriptor nào mà một process hiện đang mở bằng cách sử dụng `readdir()` để quét nội dung của thư mục `/proc/PID/fd`, chứa các symbolic link cho từng file descriptor hiện đang được process mở.

Kernel áp đặt giới hạn trần về giá trị mà giới hạn `RLIMIT_NOFILE` có thể được tăng lên. Trong các kernel trước 2.6.25, giới hạn trần này là một giá trị được hard-code được định nghĩa bởi hằng số kernel `NR_OPEN`, có giá trị là 1.048.576. (Cần rebuild kernel để tăng giới hạn trần này.) Từ kernel 2.6.25, giới hạn được định nghĩa bởi giá trị trong file `/proc/sys/fs/nr_open` đặc thù của Linux. Giá trị mặc định trong file này là 1.048.576; điều này có thể được sửa đổi bởi superuser. Các nỗ lực đặt soft hoặc hard limit `RLIMIT_NOFILE` cao hơn giá trị giới hạn trần sẽ tạo ra lỗi `EPERM`.

Cũng có một giới hạn trên toàn hệ thống về tổng số file có thể được mở bởi tất cả các process. Giới hạn này có thể được truy xuất và sửa đổi qua file `/proc/sys/fs/file-max` đặc thù của Linux. (Tham chiếu đến Mục 5.4, chúng ta có thể định nghĩa `file-max` chính xác hơn là giới hạn trên toàn hệ thống về số lượng open file description.) Chỉ các privileged process (`CAP_SYS_ADMIN`) mới có thể vượt quá giới hạn `file-max`. Trong một unprivileged process, một system call gặp phải giới hạn `file-max` thất bại với lỗi `ENFILE`.

### **RLIMIT\_NPROC**

Giới hạn `RLIMIT_NPROC` (bắt nguồn từ BSD; không có trong SUSv3 và chỉ có trên Linux và BSD) chỉ định số process tối đa có thể được tạo cho real user ID của process gọi. Các nỗ lực (`fork()`, `vfork()`, và `clone()`) để vượt quá giới hạn này thất bại với lỗi `EAGAIN`.

Giới hạn `RLIMIT_NPROC` chỉ ảnh hưởng đến process gọi. Các process khác thuộc người dùng này không bị ảnh hưởng trừ khi chúng cũng đặt hoặc kế thừa giới hạn này. Giới hạn này không được áp dụng cho privileged process (`CAP_SYS_ADMIN` hoặc `CAP_SYS_RESOURCE`).

> Linux cũng áp đặt giới hạn trên toàn hệ thống về số process có thể được tạo bởi tất cả người dùng. Trên Linux 2.4 và sau này, file `/proc/sys/kernel/threads-max` đặc thù của Linux có thể được dùng để truy xuất và sửa đổi giới hạn này.

> Để chính xác hơn, giới hạn tài nguyên `RLIMIT_NPROC` và file `threads-max` thực ra là giới hạn về số thread có thể được tạo, thay vì số process.

Cách giá trị mặc định cho giới hạn tài nguyên `RLIMIT_NPROC` được đặt đã thay đổi qua các phiên bản kernel. Trong Linux 2.2, nó được tính theo một công thức cố định. Trong Linux 2.4 và sau này, nó được tính bằng công thức dựa trên lượng bộ nhớ vật lý có sẵn.

> SUSv3 không chỉ định giới hạn tài nguyên `RLIMIT_NPROC`. Phương pháp được chỉ định bởi SUSv3 để truy xuất (nhưng không thay đổi) số process tối đa được phép cho một user ID là thông qua lời gọi `sysconf(_SC_CHILD_MAX)`. Lời gọi `sysconf()` này được hỗ trợ trên Linux, nhưng trong các phiên bản kernel trước 2.6.23, lời gọi không trả về thông tin chính xác — nó luôn trả về giá trị 999. Từ Linux 2.6.23 (và với glibc 2.4 và sau này), lời gọi này báo cáo chính xác giới hạn (bằng cách kiểm tra giá trị của giới hạn tài nguyên `RLIMIT_NPROC`).

> Không có cách portable để biết có bao nhiêu process đã được tạo cho một user ID cụ thể. Trên Linux, chúng ta có thể thử quét tất cả các file `/proc/PID/status` trên hệ thống và kiểm tra thông tin dưới mục `Uid` (liệt kê bốn process user ID theo thứ tự: real, effective, saved set, và file system) để ước tính số process hiện đang thuộc sở hữu của một người dùng. Tuy nhiên, hãy lưu ý rằng vào thời điểm chúng ta hoàn thành việc quét như vậy, thông tin này có thể đã thay đổi.

#### **RLIMIT\_RSS**

Giới hạn `RLIMIT_RSS` (bắt nguồn từ BSD; không có trong SUSv3, nhưng có sẵn rộng rãi) chỉ định số page tối đa trong resident set của process; tức là, tổng số virtual memory page hiện đang ở trong bộ nhớ vật lý. Giới hạn này được cung cấp trên Linux, nhưng hiện tại không có hiệu lực.

Trong các kernel Linux 2.4 cũ hơn (lên đến và bao gồm 2.4.29), `RLIMIT_RSS` đã có hiệu lực đối với hành vi của thao tác `madvise()` `MADV_WILLNEED` (Mục 50.4). Nếu thao tác này không thể được thực hiện do gặp phải giới hạn `RLIMIT_RSS`, lỗi `EIO` được trả về trong `errno`.

#### **RLIMIT\_RTPRIO**

Giới hạn `RLIMIT_RTPRIO` (đặc thù của Linux; từ Linux 2.6.12) chỉ định giới hạn trần về độ ưu tiên realtime có thể được đặt cho process này bằng cách sử dụng `sched_setscheduler()` và `sched_setparam()`. Tham khảo Mục [35.3.2](#page-8-1) để biết thêm chi tiết.

#### **RLIMIT\_RTTIME**

Giới hạn `RLIMIT_RTTIME` (đặc thù của Linux; từ Linux 2.6.25) chỉ định lượng thời gian CPU tối đa tính theo microsecond mà một process chạy theo realtime scheduling policy có thể tiêu thụ mà không ngủ (tức là thực hiện blocking system call). Hành vi nếu đạt đến giới hạn này cũng giống như đối với `RLIMIT_CPU`: nếu process đạt đến soft limit, thì signal `SIGXCPU` được gửi đến process, và các signal `SIGXCPU` tiếp theo được gửi cho mỗi giây CPU thêm được tiêu thụ. Khi đạt đến hard limit, signal `SIGKILL` được gửi. Tham khảo Mục [35.3.2](#page-8-1) để biết thêm chi tiết.

### **RLIMIT\_SIGPENDING**

Giới hạn `RLIMIT_SIGPENDING` (đặc thù của Linux; từ Linux 2.6.8) chỉ định số signal tối đa có thể được xếp vào hàng đợi cho real user ID của process gọi. Các nỗ lực (`sigqueue()`) để vượt quá giới hạn này thất bại với lỗi `EAGAIN`.

Giới hạn `RLIMIT_SIGPENDING` chỉ ảnh hưởng đến process gọi. Các process khác thuộc người dùng này không bị ảnh hưởng trừ khi chúng cũng đặt hoặc kế thừa giới hạn này.

Ban đầu được implement, giá trị mặc định cho giới hạn `RLIMIT_SIGPENDING` là 1024. Từ kernel 2.6.12, giá trị mặc định đã được thay đổi để giống với giá trị mặc định cho `RLIMIT_NPROC`.

Để kiểm tra giới hạn `RLIMIT_SIGPENDING`, số lượng signal đã xếp vào hàng đợi bao gồm cả signal realtime và standard signal. (Standard signal chỉ có thể được xếp vào hàng đợi một lần cho một process.) Tuy nhiên, giới hạn này chỉ được áp dụng cho `sigqueue()`. Ngay cả khi số signal được chỉ định bởi giới hạn này đã được xếp vào hàng đợi cho các process thuộc real user ID này, vẫn có thể sử dụng `kill()` để xếp một instance của mỗi signal (bao gồm cả realtime signal) chưa có trong hàng đợi của process vào hàng đợi.

Từ kernel 2.6.12 trở đi, trường `SigQ` của file `/proc/PID/status` đặc thù của Linux hiển thị số lượng hiện tại và tối đa của signal đã xếp vào hàng đợi cho real user ID của process.

#### **RLIMIT\_STACK**

Giới hạn `RLIMIT_STACK` chỉ định kích thước tối đa của process stack, tính theo byte. Các nỗ lực phát triển stack vượt quá giới hạn này dẫn đến việc tạo signal `SIGSEGV` cho process. Vì stack đã cạn kiệt, cách duy nhất để bắt signal này là bằng cách thiết lập một alternate signal stack, như được mô tả trong Mục 21.3.

> Từ Linux 2.6.23, giới hạn `RLIMIT_STACK` cũng xác định lượng không gian có sẵn để lưu trữ các đối số command-line và các biến môi trường của process. Xem trang manual `execve(2)` để biết chi tiết.

### **36.4 Tóm Tắt**

Các process tiêu thụ nhiều tài nguyên hệ thống khác nhau. System call `getrusage()` cho phép một process theo dõi một số tài nguyên được tiêu thụ bởi chính nó và bởi các process con của nó.

Các system call `setrlimit()` và `getrlimit()` cho phép một process đặt và truy xuất giới hạn về mức tiêu thụ tài nguyên của nó đối với nhiều tài nguyên khác nhau. Mỗi giới hạn tài nguyên có hai thành phần: soft limit, là thứ mà kernel áp dụng khi kiểm tra mức tiêu thụ tài nguyên của một process, và hard limit, đóng vai trò là giới hạn trần cho giá trị của soft limit. Một unprivileged process có thể đặt soft limit cho một tài nguyên thành bất kỳ giá trị nào trong phạm vi từ 0 đến hard limit, nhưng chỉ có thể giảm hard limit. Một privileged process có thể thực hiện bất kỳ thay đổi nào đối với giá trị giới hạn nào, miễn là soft limit nhỏ hơn hoặc bằng hard limit. Nếu một process gặp phải soft limit, nó thường được thông báo về thực tế đó bằng cách nhận signal hoặc thông qua lỗi của system call cố gắng vượt quá giới hạn.

# **36.5 Bài Tập**

- **36-1.** Viết một chương trình cho thấy rằng cờ `RUSAGE_CHILDREN` của `getrusage()` chỉ truy xuất thông tin về các process con mà `wait()` đã được thực hiện. (Cho chương trình tạo ra một process con tiêu thụ một lượng thời gian CPU, và sau đó cho process cha gọi `getrusage()` trước và sau khi gọi `wait()`.)
- **36-2.** Viết một chương trình thực thi một lệnh và sau đó hiển thị mức sử dụng tài nguyên của nó. Điều này tương tự với những gì lệnh `time(1)` làm. Vì vậy, chúng ta sẽ sử dụng chương trình này như sau:
  - \$ **./rusage** *command arg...*
- **36-3.** Viết các chương trình để xác định điều gì xảy ra nếu mức tiêu thụ tài nguyên của một process đã vượt quá soft limit được chỉ định trong một lời gọi đến `setrlimit()`.
