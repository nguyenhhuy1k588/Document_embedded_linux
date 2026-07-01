## Chương 63
# Các Mô Hình I/O Thay Thế (Alternative I/O Models)

Chương này thảo luận về ba thay thế cho model I/O thông thường:
- **I/O multiplexing** (system call `select()` và `poll()`);
- **Signal-driven I/O**; và
- **epoll API** đặc thù của Linux.

---

## 63.1 Tổng Quan

Model I/O blocking truyền thống đủ cho nhiều ứng dụng, nhưng không phải tất cả. Một số ứng dụng cần:
- Kiểm tra I/O có thể thực hiện trên file descriptor mà không block.
- Giám sát nhiều file descriptor để xem I/O có thể thực hiện trên bất kỳ descriptor nào.

**Nonblocking I/O** cho phép kiểm tra định kỳ, nhưng polling trong vòng lặp chặt lãng phí CPU time. **Multiple process/thread** là đắt đỏ và phức tạp.

Ba kỹ thuật thay thế:
- **I/O multiplexing**: Process giám sát đồng thời nhiều file descriptor. `select()` và `poll()` thực hiện I/O multiplexing.
- **Signal-driven I/O**: Process yêu cầu kernel gửi signal khi I/O có thể thực hiện trên file descriptor. Hiệu suất tốt hơn `select()`/`poll()` khi giám sát số lượng lớn descriptor.
- **epoll API**: Tương tự I/O multiplexing nhưng hiệu suất tốt hơn nhiều. Cho phép level-triggered hoặc edge-triggered notification.

**Lưu ý**: Không cơ chế nào thực sự thực hiện I/O — chúng chỉ cho biết file descriptor đã sẵn sàng.

### 63.1.1 Level-Triggered và Edge-Triggered Notification

Hai model readiness notification:
- **Level-triggered**: File descriptor được coi là sẵn sàng nếu có thể thực hiện I/O system call mà không block.
- **Edge-triggered**: Thông báo chỉ khi có I/O activity kể từ lần cuối giám sát.

**Bảng 63-1:** Model notification của các I/O API

| I/O model | Level-triggered? | Edge-triggered? |
|-----------|-----------------|----------------|
| `select()`, `poll()` | ● | |
| Signal-driven I/O | | ● |
| epoll | ● | ● |

Với **edge-triggered notification**, chương trình nên thực hiện càng nhiều I/O càng tốt mỗi khi nhận thông báo, thường dùng nonblocking file descriptor và lặp cho đến khi nhận `EAGAIN`/`EWOULDBLOCK`.

### 63.1.2 Dùng Nonblocking I/O với Alternative I/O Models

Nonblocking I/O thường dùng cùng edge-triggered notification. Ngay cả với level-triggered API, nên dùng nonblocking I/O vì trong môi trường multithreaded, trạng thái sẵn sàng của descriptor có thể thay đổi giữa lần kiểm tra và lần thực hiện I/O.

---

## 63.2 I/O Multiplexing

### 63.2.1 System Call select()

```c
#include <sys/time.h>
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
           struct timeval *timeout);
         Returns number of ready file descriptors, 0 on timeout, or –1 on error
```

- `readfds`: Tập file descriptor cần kiểm tra xem có thể đọc không.
- `writefds`: Tập file descriptor cần kiểm tra xem có thể ghi không.
- `exceptfds`: Tập file descriptor cần kiểm tra exceptional condition.
- `nfds`: Phải là số lớn hơn file descriptor lớn nhất trong bất kỳ tập nào.
- `timeout`: Nếu `NULL` → block vô hạn; nếu có giá trị → timeout.

Thao tác với `fd_set` dùng macro:
```c
void FD_ZERO(fd_set *fdset);     /* Khởi tạo tập rỗng */
void FD_SET(int fd, fd_set *fdset);   /* Thêm fd vào tập */
void FD_CLR(int fd, fd_set *fdset);   /* Xóa fd khỏi tập */
int  FD_ISSET(int fd, fd_set *fdset); /* Kiểm tra fd trong tập */
```

`FD_SETSIZE` = 1024 trên Linux. Các đối số `fd_set` là value-result — phải reinitialize trước mỗi lần gọi trong vòng lặp.

**Giá trị trả về**: `-1` lỗi, `0` timeout, số dương = số descriptor sẵn sàng.

### 63.2.2 System Call poll()

```c
#include <poll.h>
int poll(struct pollfd fds[], nfds_t nfds, int timeout);
          Returns number of ready file descriptors, 0 on timeout, or –1 on error
```

```c
struct pollfd {
    int   fd;      /* File descriptor */
    short events;  /* Requested events bit mask */
    short revents; /* Returned events bit mask */
};
```

`timeout`: `-1` block vô hạn; `0` kiểm tra ngay lập tức; `>0` timeout milliseconds.

Bit mask quan trọng:

| Bit | events? | revents? | Mô tả |
|-----|---------|---------|-------|
| `POLLIN` | ● | ● | Có thể đọc dữ liệu |
| `POLLPRI` | ● | ● | Dữ liệu high-priority có thể đọc |
| `POLLRDHUP` | ● | ● | Peer socket đóng kết nối |
| `POLLOUT` | ● | ● | Có thể ghi dữ liệu |
| `POLLERR` | | ● | Đã xảy ra lỗi |
| `POLLHUP` | | ● | Hangup đã xảy ra |
| `POLLNVAL` | | ● | File descriptor không mở |

`poll()` không có giới hạn trên `FD_SETSIZE`, và dùng trường riêng `events`/`revents` nên không cần reinitialize.

### 63.2.3 Khi Nào File Descriptor Sẵn Sàng?

- **Regular file**: Luôn readable và writable.
- **Terminal/pseudoterminal**: Readable khi có input; writable khi có thể ghi.
- **Pipe read end**: Readable khi có dữ liệu; khi write end đóng: `POLLHUP` (poll) hoặc readable (select).
- **Socket**: Readable khi có input; `POLLIN` khi có connection trên listening socket.

### 63.2.4 So Sánh select() và poll()

**Khác biệt về API**:
- `select()`: Giới hạn `FD_SETSIZE`; phải reinitialize fd_set; timeout microsecond.
- `poll()`: Không giới hạn range; dùng trường riêng `events`/`revents`; timeout millisecond.

**Hiệu suất**: Tương tự với số lượng descriptor nhỏ. Với sparse descriptor set, `poll()` tốt hơn `select()`. Cả hai đều kém khi monitor số lượng lớn descriptor.

### 63.2.5 Vấn Đề với select() và poll()

Khi monitor số lượng lớn file descriptor, cả hai kém hiệu quả vì:
- Mỗi lần gọi, kernel phải kiểm tra tất cả descriptor được chỉ định.
- Mỗi lần gọi, phải truyền toàn bộ danh sách descriptor từ user space đến kernel.
- Sau khi gọi, phải kiểm tra mọi phần tử của data structure được trả về.

---

## 63.3 Signal-Driven I/O

Với signal-driven I/O, process yêu cầu kernel gửi signal khi I/O có thể thực hiện trên file descriptor. Các bước:

1. Thiết lập signal handler cho `SIGIO`.
2. Đặt owner của file descriptor: `fcntl(fd, F_SETOWN, getpid())`.
3. Bật nonblocking I/O: `O_NONBLOCK`.
4. Bật signal-driven I/O: `O_ASYNC`.

```c
flags = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, flags | O_ASYNC | O_NONBLOCK);
```

5. Process thực hiện các tác vụ khác cho đến khi signal được gửi.
6. Vì edge-triggered, thực hiện I/O cho đến khi nhận `EAGAIN`/`EWOULDBLOCK`.

### 63.3.2 Tinh Chỉnh Signal-Driven I/O

Để hiệu quả hơn khi monitor số lượng lớn descriptor:
- Dùng `fcntl(fd, F_SETSIG, sig)` để chỉ định realtime signal thay vì `SIGIO`.
- Dùng `SA_SIGINFO` trong `sigaction()` → handler nhận `siginfo_t` với `si_fd` và `si_code`.
- Nếu dùng realtime signal: nhiều notification có thể được xếp hàng.
- Có thể dùng `sigwaitinfo()`/`sigtimedwait()` thay vì signal handler để xử lý đồng bộ.

Cần xử lý signal-queue overflow: Thiết lập handler cho `SIGIO` để xử lý khi realtime signal queue đầy.

---

## 63.4 epoll API

epoll cung cấp hiệu suất tốt hơn `select()`/`poll()` và signal-driven I/O. Cho phép level-triggered hoặc edge-triggered notification. Là Linux-specific API (từ kernel 2.6).

epoll dùng **interest list** (danh sách descriptor cần monitor) và **ready list** (danh sách descriptor sẵn sàng).

### 63.4.1 Tạo epoll Instance: epoll_create()

```c
#include <sys/epoll.h>
int epoll_create(int size);
                                Returns file descriptor on success, or –1 on error
```

Trả về file descriptor tham chiếu đến epoll instance mới. Từ kernel 2.6.27, `epoll_create1(flags)` hỗ trợ thêm `EPOLL_CLOEXEC` flag.

### 63.4.2 Thay Đổi epoll Interest List: epoll_ctl()

```c
#include <sys/epoll.h>
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *ev);
                                             Returns 0 on success, or –1 on error
```

`op`:
- `EPOLL_CTL_ADD`: Thêm `fd` vào interest list.
- `EPOLL_CTL_MOD`: Thay đổi event setting cho `fd`.
- `EPOLL_CTL_DEL`: Xóa `fd` khỏi interest list.

```c
struct epoll_event {
    uint32_t     events;   /* epoll events (bit mask) */
    epoll_data_t data;     /* User data */
};

typedef union epoll_data {
    void     *ptr;
    int       fd;
    uint32_t  u32;
    uint64_t  u64;
} epoll_data_t;
```

### 63.4.3 Chờ Event: epoll_wait()

```c
#include <sys/epoll.h>
int epoll_wait(int epfd, struct epoll_event *evlist, int maxevents, int timeout);
          Returns number of ready file descriptors, 0 on timeout, or –1 on error
```

`evlist` chứa thông tin về các descriptor sẵn sàng. `timeout`: `-1` block vô hạn; `0` nonblocking; `>0` timeout milliseconds.

**Bảng 63-8:** Bit-mask values cho epoll events field

| Bit | Input? | Returned? | Mô tả |
|-----|--------|-----------|-------|
| `EPOLLIN` | ● | ● | Có thể đọc dữ liệu |
| `EPOLLPRI` | ● | ● | High-priority data |
| `EPOLLRDHUP` | ● | ● | Peer socket đóng |
| `EPOLLOUT` | ● | ● | Có thể ghi |
| `EPOLLET` | ● | | Edge-triggered notification |
| `EPOLLONESHOT` | ● | | Disable sau khi thông báo |
| `EPOLLERR` | | ● | Lỗi xảy ra |
| `EPOLLHUP` | | ● | Hangup xảy ra |

**EPOLLONESHOT**: Sau lần `epoll_wait()` đầu tiên thông báo sẵn sàng, descriptor bị đánh dấu inactive. Dùng `EPOLL_CTL_MOD` để re-enable.

### 63.4.4 Ngữ Nghĩa epoll chi tiết hơn

Interest list được liên kết với open file description (không phải file descriptor). Khi đóng file descriptor, open file description chỉ bị xóa khỏi interest list khi tất cả descriptor tham chiếu đến nó đều đóng.

### 63.4.5 Hiệu Suất epoll Versus I/O Multiplexing

| N descriptor | poll() CPU | select() CPU | epoll CPU |
|-------------|-----------|-------------|---------|
| 10 | 0.61s | 0.73s | 0.41s |
| 100 | 2.9s | 3.0s | 0.42s |
| 1000 | 35s | 35s | 0.53s |
| 10000 | 990s | 930s | 0.66s |

epoll hiệu quả vì kernel "nhớ" danh sách và chỉ trả về descriptor sẵn sàng, không cần kiểm tra tất cả mỗi lần gọi.

### 63.4.6 Edge-Triggered Notification

Mặc định epoll dùng **level-triggered notification**. Bật **edge-triggered** bằng flag `EPOLLET`:

```c
ev.data.fd = fd;
ev.events = EPOLLIN | EPOLLET;
epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
```

Với edge-triggered: Sau khi nhận thông báo, `epoll_wait()` thứ hai sẽ block vì không có input mới; với level-triggered, `epoll_wait()` thứ hai vẫn trả về sẵn sàng.

**Quy trình edge-triggered**:
1. Đặt tất cả descriptor được monitor là nonblocking.
2. Xây dựng interest list.
3. Lặp: gọi `epoll_wait()` → xử lý I/O cho mỗi descriptor cho đến khi nhận `EAGAIN`.

**Ngăn starvation**: Duy trì danh sách descriptor đã sẵn sàng; thực hiện I/O giới hạn trên mỗi descriptor trước khi chuyển sang descriptor khác.

---

## 63.5 Chờ Đồng Thời Signal và File Descriptor

Vấn đề race condition: Signal có thể đến trước `select()` nhưng sau khi thiết lập handler.

### 63.5.1 System Call pselect()

```c
#define _XOPEN_SOURCE 600
#include <sys/select.h>
int pselect(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
            struct timespec *timeout, const sigset_t *sigmask);
         Returns number of ready file descriptors, 0 on timeout, or –1 on error
```

`pselect()` thực hiện atomically: unblock `sigmask`, gọi `select()`, restore signal mask. Timeout là `timespec` (nanosecond precision). Không thay đổi `timeout` khi trả về.

Linux cũng cung cấp `ppoll()` và `epoll_pwait()` là các biến thể tương tự.

### 63.5.2 Self-Pipe Trick

Vì `pselect()` không available trên tất cả UNIX, dùng **self-pipe trick** cho portable application:

1. Tạo pipe, đặt cả hai đầu nonblocking.
2. Thêm read end của pipe vào `readfds` của `select()`.
3. Signal handler ghi một byte vào write end của pipe.
4. `select()` trong vòng lặp restart nếu bị interrupt.
5. Kiểm tra read end của pipe để biết signal đã đến.
6. Đọc tất cả byte từ pipe; thực hiện hành động phản hồi signal.

```c
static int pfd[2];
static void
handler(int sig)
{
    int savedErrno = errno;
    if (write(pfd[1], "x", 1) == -1 && errno != EAGAIN)  errExit("write");
    errno = savedErrno;
}
/* ... */
if (pipe(pfd) == -1)  errExit("pipe");
FD_SET(pfd[0], &readfds);
/* Đặt cả hai đầu pipe thành nonblocking */
/* Cài signal handler */
while ((ready = select(nfds, &readfds, NULL, NULL, pto)) == -1 && errno == EINTR)
    continue;
if (FD_ISSET(pfd[0], &readfds)) {
    /* Signal đã đến: đọc hết byte từ pipe, xử lý signal */
}
```

Kỹ thuật self-pipe cũng dùng được với `poll()` và `epoll_wait()`.

---

## 63.6 Tóm Tắt

Ba kỹ thuật thay thế để monitor nhiều file descriptor:

**I/O multiplexing** (`select()`/`poll()`): Portable, long-standing; truyền đầy đủ danh sách descriptor mỗi lần gọi; hiệu suất kém khi monitor nhiều descriptor.

**Signal-driven I/O**: Hiệu suất tốt hơn khi monitor nhiều descriptor; dùng edge-triggered model; yêu cầu signal handling phức tạp; dùng realtime signal cho queuing và siginfo_t.

**epoll**: Linux-specific; hiệu suất tốt nhất; hỗ trợ cả level-triggered và edge-triggered; tránh phức tạp của signal handling.

Level-triggered = thông báo khi I/O có thể thực hiện; edge-triggered = thông báo khi có I/O activity mới. Edge-triggered thường dùng với nonblocking I/O.

`pselect()` và self-pipe trick giải quyết bài toán chờ đồng thời signal và file descriptor.

---

## 63.7 Bài Tập

- **63-1.** Sửa đổi `poll_pipes.c` để dùng `select()` thay vì `poll()`.
- **63-2.** Viết echo server xử lý cả TCP lẫn UDP client.
- **63-3.** Viết chương trình dùng `select()` để giám sát cả terminal lẫn System V message queue.
- **63-4.** Điều gì xảy ra nếu đọc tất cả byte từ pipe trước khi thực hiện hành động phản hồi signal?
- **63-5.** Sửa đổi `self_pipe.c` để dùng `poll()` thay vì `select()`.
- **63-6.** Viết chương trình dùng `epoll_create()` rồi ngay lập tức gọi `epoll_wait()` với interest list rỗng. Điều gì xảy ra?
- **63-7.** Khi nhiều descriptor sẵn sàng và `maxevents` nhỏ, `epoll_wait()` trả về descriptor nào?
- **63-8.** Sửa `demo_sigio.c` để dùng realtime signal thay vì `SIGIO`, hiển thị `si_fd` và `si_code`.
