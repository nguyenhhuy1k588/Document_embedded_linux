## Chương 23
# Timer và Ngủ (Timers and Sleeping)

Timer cho phép process lên lịch thông báo cho chính nó xảy ra tại một thời điểm trong tương lai. Sleeping cho phép process (hoặc thread) tạm dừng thực thi trong một khoảng thời gian. Chương này mô tả các giao diện để thiết lập timer và để ngủ.

---

## 23.1 Interval Timer

System call `setitimer()` thiết lập interval timer — timer hết hạn tại một điểm trong tương lai và (tùy chọn) ở các khoảng thời gian đều đặn sau đó.

```c
#include <sys/time.h>
int setitimer(int which, const struct itimerval *new_value,
              struct itimerval *old_value);
                                         Returns 0 on success, or –1 on error
```

Sử dụng `setitimer()`, process có thể thiết lập ba loại timer khác nhau bằng cách chỉ định `which` là:

- **`ITIMER_REAL`**: Tạo timer đếm ngược theo real time (wall clock). Khi hết hạn, signal `SIGALRM` được tạo ra.
- **`ITIMER_VIRTUAL`**: Tạo timer đếm ngược theo process virtual time (user-mode CPU time). Khi hết hạn, signal `SIGVTALRM` được tạo ra.
- **`ITIMER_PROF`**: Tạo profiling timer đếm theo process time (tổng user-mode và kernel-mode CPU time). Khi hết hạn, signal `SIGPROF` được tạo ra.

Disposition mặc định của tất cả timer signal là kết thúc process. Phải thiết lập handler cho signal được timer tạo ra.

Các đối số `new_value` và `old_value` là pointer đến `itimerval`:

```c
struct itimerval {
    struct timeval it_interval; /* Interval cho periodic timer */
    struct timeval it_value;    /* Giá trị hiện tại (thời gian đến hết hạn tiếp theo) */
};
```

`it_value` chỉ định thời gian đến khi timer hết hạn. `it_interval` chỉ định khoảng thời gian lặp lại (nếu cả hai trường là 0 thì chỉ hết hạn một lần).

Mỗi process chỉ có một timer mỗi loại. Gọi `setitimer()` với cả hai trường `new_value.it_value` bằng 0 sẽ vô hiệu hóa timer hiện có.

`old_value` (nếu không `NULL`) trả về cài đặt trước đó của timer.

System call `getitimer()` trả về trạng thái hiện tại của timer:

```c
#include <sys/time.h>
int getitimer(int which, struct itimerval *curr_value);
                                            Returns 0 on success, or –1 on error
```

Timer được thiết lập bởi `setitimer()` (và `alarm()`) được bảo toàn qua `exec()`, nhưng không kế thừa bởi child process qua `fork()`.

### Giao diện timer đơn giản hơn: alarm()

System call `alarm()` cung cấp giao diện đơn giản để thiết lập real-time timer hết hạn một lần:

```c
#include <unistd.h>
unsigned int alarm(unsigned int seconds);
                   Always succeeds, returning number of seconds remaining on
                     any previously set timer, or 0 if no timer previously was set
```

`alarm()` trả về số giây còn lại trước khi timer trước đó hết hạn, hoặc 0 nếu không có timer. Đặt `alarm(0)` vô hiệu hóa timer hiện có.

---

## 23.2 Lên Lịch và Độ Chính Xác của Timer

Tùy thuộc vào tải hệ thống và scheduling của process, process có thể không được lên lịch để chạy cho đến một khoảng thời gian ngắn sau khi timer thực sự hết hạn. Tuy nhiên, việc hết hạn của periodic timer vẫn đều đặn — timer không subject to creeping errors.

### High-resolution timer

Từ kernel 2.6.21, Linux tùy chọn hỗ trợ high-resolution timer. Nếu được bật (`CONFIG_HIGH_RES_TIMERS`), độ chính xác của các timer và sleep interface không còn bị giới hạn bởi jiffy. Trên phần cứng hiện đại, độ chính xác xuống đến microsecond là điển hình.

---

## 23.3 Đặt Timeout trên Blocking Operation

Một cách dùng real-time timer là đặt giới hạn trên cho thời gian mà blocking system call có thể vẫn bị block. Các bước:

1. Gọi `sigaction()` để thiết lập handler cho `SIGALRM`, bỏ qua `SA_RESTART` để system call không được restart.
2. Gọi `alarm()` hoặc `setitimer()` để thiết lập timer.
3. Thực hiện blocking system call.
4. Sau khi system call trả về, gọi `alarm()` hoặc `setitimer()` lại để tắt timer.
5. Kiểm tra xem blocking system call có thất bại với `errno` là `EINTR` không.

**Listing 23-2:** Thực hiện `read()` với timeout

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– timers/timed_read.c
#include <signal.h>
#include "tlpi_hdr.h"
#define BUF_SIZE 200
static void
handler(int sig)
{
    printf("Caught signal\n");  /* UNSAFE (xem Mục 21.1.2) */
}
int
main(int argc, char *argv[])
{
    struct sigaction sa;
    char buf[BUF_SIZE];
    ssize_t numRead;
    int savedErrno;
    sa.sa_flags = (argc > 2) ? SA_RESTART : 0;
    sigemptyset(&sa.sa_mask);
    sa.sa_handler = handler;
    if (sigaction(SIGALRM, &sa, NULL) == -1)
        errExit("sigaction");
    alarm((argc > 1) ? getInt(argv[1], GN_NONNEG, "num-secs") : 10);
    numRead = read(STDIN_FILENO, buf, BUF_SIZE - 1);
    savedErrno = errno;
    alarm(0);                   /* Đảm bảo timer tắt */
    errno = savedErrno;
    if (numRead == -1) {
        if (errno == EINTR)
            printf("Read timed out\n");
        else
            errMsg("read");
    } else {
        printf("Successful read (%ld bytes): %.*s",
               (long) numRead, (int) numRead, buf);
    }
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– timers/timed_read.c
```

---

## 23.4 Tạm Dừng Thực Thi Trong Khoảng Thời Gian Cố Định (Sleeping)

### 23.4.1 Low-Resolution Sleeping: sleep()

Hàm `sleep()` tạm dừng thực thi của process gọi trong số giây được chỉ định trong đối số `seconds` hoặc cho đến khi signal được bắt.

```c
#include <unistd.h>
unsigned int sleep(unsigned int seconds);
                                Returns 0 on normal completion, or number of
                                     unslept seconds if prematurely terminated
```

Trả về 0 nếu sleep hoàn thành. Nếu bị interrupt bởi signal, trả về số giây còn lại chưa ngủ.

### 23.4.2 High-Resolution Sleeping: nanosleep()

Hàm `nanosleep()` thực hiện tác vụ tương tự `sleep()`, nhưng cung cấp độ phân giải tốt hơn.

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int nanosleep(const struct timespec *request, struct timespec *remain);
                                      Returns 0 on successfully completed sleep,
                                              or –1 on error or interrupted sleep
```

```c
struct timespec {
    time_t tv_sec;  /* Seconds */
    long   tv_nsec; /* Nanoseconds (0 to 999,999,999) */
};
```

`nanosleep()` được quy định rõ ràng trong SUSv3 là không được triển khai bằng signal, nên ta có thể portable kết hợp `nanosleep()` với `alarm()` hoặc `setitimer()`.

Nếu bị interrupt bởi signal handler, `nanosleep()` trả về `-1` với `errno` là `EINTR`, và nếu `remain` không `NULL`, buffer được trỏ trả về thời gian còn lại chưa ngủ. Ta có thể dùng giá trị này để restart lời gọi.

---

## 23.5 POSIX Clock

POSIX clock cung cấp API để truy cập clock với độ chính xác nanosecond. Chương trình dùng API này phải biên dịch với `-lrt`.

### 23.5.1 Lấy Giá Trị Clock: clock_gettime()

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int clock_gettime(clockid_t clockid, struct timespec *tp);
int clock_getres(clockid_t clockid, struct timespec *res);
                                         Both return 0 on success, or –1 on error
```

**Bảng 23-1:** Loại POSIX.1b clock

| Clock ID | Mô tả |
|---------|-------|
| `CLOCK_REALTIME` | System-wide real-time clock có thể đặt được |
| `CLOCK_MONOTONIC` | Monotonic clock không thể đặt được (từ lúc hệ thống khởi động) |
| `CLOCK_PROCESS_CPUTIME_ID` | Per-process CPU-time clock (từ Linux 2.6.12) |
| `CLOCK_THREAD_CPUTIME_ID` | Per-thread CPU-time clock (từ Linux 2.6.12) |

- `CLOCK_REALTIME`: System-wide clock đo wall-clock time, có thể thay đổi.
- `CLOCK_MONOTONIC`: Hữu ích cho ứng dụng không bị ảnh hưởng bởi thay đổi đột ngột của system clock.
- `CLOCK_PROCESS_CPUTIME_ID`: Đo CPU time user và system của process gọi.
- `CLOCK_THREAD_CPUTIME_ID`: Tương tự nhưng cho individual thread.

### 23.5.2 Đặt Giá Trị Clock: clock_settime()

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int clock_settime(clockid_t clockid, const struct timespec *tp);
                                             Returns 0 on success, or –1 on error
```

Chỉ process có đặc quyền (`CAP_SYS_TIME`) mới có thể đặt `CLOCK_REALTIME`.

### 23.5.3 Lấy Clock ID của Process hoặc Thread Cụ Thể

```c
#define _XOPEN_SOURCE 600
#include <time.h>
int clock_getcpuclockid(pid_t pid, clockid_t *clockid);
                       Returns 0 on success, or a positive error number on error

int pthread_getcpuclockid(pthread_t thread, clockid_t *clockid);
                       Returns 0 on success, or a positive error number on error
```

### 23.5.4 High-Resolution Sleeping Cải Tiến: clock_nanosleep()

```c
#define _XOPEN_SOURCE 600
#include <time.h>
int clock_nanosleep(clockid_t clockid, int flags,
                    const struct timespec *request, struct timespec *remain);
                                     Returns 0 on successfully completed sleep,
                       or a positive error number on error or interrupted sleep
```

Điểm phân biệt `clock_nanosleep()` với `nanosleep()`:

1. Nếu `flags` là `TIMER_ABSTIME`, `request` chỉ định thời gian tuyệt đối — tránh vấn đề oversleeping khi sleep bị interrupt.
2. Có thể chọn clock để đo sleep interval bằng `clockid`.

Ví dụ ngủ 20 giây theo `CLOCK_REALTIME` dùng absolute time:

```c
struct timespec request;
if (clock_gettime(CLOCK_REALTIME, &request) == -1)
    errExit("clock_gettime");
request.tv_sec += 20;
s = clock_nanosleep(CLOCK_REALTIME, TIMER_ABSTIME, &request, NULL);
```

---

## 23.6 POSIX Interval Timer

Timer `setitimer()` có một số hạn chế:
- Chỉ có thể đặt một timer mỗi loại.
- Chỉ có thể thông báo qua signal.
- Không thể biết timer overrun.
- Giới hạn ở microsecond.

POSIX.1b định nghĩa API để khắc phục các hạn chế này, được triển khai trong Linux 2.6.

POSIX timer API gồm các bước:
1. `timer_create()` tạo timer mới và định nghĩa phương thức thông báo.
2. `timer_settime()` arm (bắt đầu) hoặc disarm (dừng) timer.
3. `timer_delete()` xóa timer khi không cần.

POSIX timer không kế thừa bởi child process qua `fork()`. Chương trình dùng API này phải biên dịch với `-lrt`.

### 23.6.1 Tạo Timer: timer_create()

```c
#define _POSIX_C_SOURCE 199309
#include <signal.h>
#include <time.h>
int timer_create(clockid_t clockid, struct sigevent *evp, timer_t *timerid);
                                             Returns 0 on success, or –1 on error
```

Đối số `evp` xác định cách chương trình được thông báo khi timer hết hạn, qua structure `sigevent` với trường `sigev_notify`:

**Bảng 23-2:** Giá trị cho trường `sigev_notify` của structure `sigevent`

| Giá trị `sigev_notify` | Phương thức thông báo | SUSv3 |
|------------------------|----------------------|-------|
| `SIGEV_NONE` | Không thông báo; giám sát bằng `timer_gettime()` | ● |
| `SIGEV_SIGNAL` | Gửi signal `sigev_signo` cho process | ● |
| `SIGEV_THREAD` | Gọi `sigev_notify_function` như start function của thread mới | ● |
| `SIGEV_THREAD_ID` | Gửi signal đến thread `sigev_notify_thread_id` | |

Chỉ định `evp` là `NULL` tương đương với `sigev_notify` = `SIGEV_SIGNAL`, `sigev_signo` = `SIGALRM`.

### 23.6.2 Arm và Disarm Timer: timer_settime()

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int timer_settime(timer_t timerid, int flags, const struct itimerspec *value,
                  struct itimerspec *old_value);
                                         Returns 0 on success, or –1 on error
```

Structure `itimerspec` dùng `timespec` thay vì `timeval`:

```c
struct itimerspec {
    struct timespec it_interval; /* Interval cho periodic timer */
    struct timespec it_value;    /* Hết hạn đầu tiên */
};
```

Nếu `flags` là 0, `value.it_value` là tương đối. Nếu là `TIMER_ABSTIME`, là thời gian tuyệt đối. Để disarm: đặt cả hai trường của `value.it_value` về 0.

### 23.6.3 Lấy Giá Trị Hiện Tại của Timer: timer_gettime()

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int timer_gettime(timer_t timerid, struct itimerspec *curr_value);
                                             Returns 0 on success, or –1 on error
```

### 23.6.4 Xóa Timer: timer_delete()

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int timer_delete(timer_t timerid);
                                             Returns 0 on success, or –1 on error
```

### 23.6.5 Thông Báo qua Signal

Khi nhận timer signal (qua signal handler hoặc `sigwaitinfo()`), structure `siginfo_t` cung cấp thêm thông tin:
- `si_signo`: Signal được tạo bởi timer này.
- `si_code`: Được đặt thành `SI_TIMER`.
- `si_value`: Giá trị được cung cấp trong `evp.sigev_value` khi tạo timer.

Thường gán `evp.sigev_value.sival_ptr` địa chỉ của `timerid` để signal handler có thể lấy ID của timer đã tạo ra signal.

### 23.6.6 Timer Overrun

Nếu timer hết hạn nhiều lần trước khi signal liên quan được bắt hoặc chấp nhận, các instance bổ sung của signal không được xếp hàng. Thay vào đó, ta có thể lấy **timer overrun count** — số lần hết hạn thêm giữa lúc signal được tạo và lúc nhận.

```c
#define _POSIX_C_SOURCE 199309
#include <time.h>
int timer_getoverrun(timer_t timerid);
                         Returns timer overrun count on success, or –1 on error
```

Hàm `timer_getoverrun()` là async-signal-safe, an toàn để gọi từ signal handler.

### 23.6.7 Thông Báo qua Thread

Flag `SIGEV_THREAD` cho phép nhận thông báo hết hạn timer qua hàm được gọi trong thread riêng biệt. Khi timer hết hạn, hàm được gọi như start function của thread mới với `sigev_value` làm đối số.

---

## 23.7 Timer Thông Báo qua File Descriptor: timerfd API

Từ kernel 2.6.25, Linux cung cấp timerfd API cho phép đọc thông báo hết hạn timer từ file descriptor — hữu ích vì có thể giám sát cùng các descriptor khác bằng `select()`, `poll()`, và `epoll`.

```c
#include <sys/timerfd.h>
int timerfd_create(int clockid, int flags);
                                Returns file descriptor on success, or –1 on error
```

`clockid` có thể là `CLOCK_REALTIME` hoặc `CLOCK_MONOTONIC`. `flags` hỗ trợ `TFD_CLOEXEC` và `TFD_NONBLOCK` (từ Linux 2.6.27).

```c
#include <sys/timerfd.h>
int timerfd_settime(int fd, int flags, const struct itimerspec *new_value,
                    struct itimerspec *old_value);
                                        Returns 0 on success, or –1 on error

int timerfd_gettime(int fd, struct itimerspec *curr_value);
                                             Returns 0 on success, or –1 on error
```

`flags` có thể là 0 (thời gian tương đối) hoặc `TFD_TIMER_ABSTIME` (thời gian tuyệt đối).

**Đọc từ timerfd file descriptor:**

Buffer truyền vào `read()` phải đủ lớn để giữ unsigned 8-byte integer (`uint64_t`). Nếu một hoặc nhiều hết hạn đã xảy ra, `read()` trả về ngay với số hết hạn. Nếu không có, `read()` block đến khi hết hạn tiếp theo xảy ra.

**Tương tác với fork() và exec():** Trong `fork()`, child kế thừa bản sao file descriptor. Qua `exec()`, file descriptor được bảo toàn (trừ khi đánh dấu close-on-exec).

---

## 23.8 Tóm Tắt

Process có thể dùng `setitimer()` hoặc `alarm()` để thiết lập timer, nhận signal sau khoảng thời gian nhất định. Một cách dùng timer là đặt giới hạn trên cho thời gian blocking system call.

Linux 2.6 triển khai các extension POSIX.1b định nghĩa API cho high-precision clock và timer. POSIX.1b timer cho phép: tạo nhiều timer; chọn signal; lấy timer overrun count; nhận thông báo qua thread.

timerfd API cung cấp giao diện tạo timer cho phép đọc thông báo qua file descriptor, có thể giám sát bằng `select()`, `poll()`, và `epoll`.

---

## 23.9 Bài Tập

- **23-1.** Triển khai `alarm()` sử dụng `setitimer()`.
- **23-2.** Thay `nanosleep()` bằng `clock_gettime()` và `clock_nanosleep()` với `TIMER_ABSTIME` để tránh vấn đề oversleeping khi signal được gửi với tốc độ cao.
- **23-3.** Viết chương trình để chứng minh rằng nếu đối số `evp` của `timer_create()` là `NULL`, điều này tương đương với chỉ định `sigev_notify = SIGEV_SIGNAL`, `sigev_signo = SIGALRM`.
- **23-4.** Sửa đổi chương trình trong Listing 23-5 để dùng `sigwaitinfo()` thay vì signal handler.
