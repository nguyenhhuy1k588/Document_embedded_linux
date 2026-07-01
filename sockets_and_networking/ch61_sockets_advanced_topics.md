## Chương 61
# Socket: Các Chủ Đề Nâng Cao (Sockets: Advanced Topics)

Chương này xem xét nhiều chủ đề nâng cao hơn liên quan đến lập trình socket, bao gồm:
- Partial read và write trên stream socket
- Sử dụng `shutdown()` để đóng một nửa kênh hai chiều
- Các system call I/O dành riêng cho socket: `recv()` và `send()`
- System call `sendfile()` để xuất dữ liệu hiệu quả
- Chi tiết hoạt động của giao thức TCP
- Sử dụng `netstat` và `tcpdump` để giám sát và debug
- `getsockopt()` và `setsockopt()` để lấy và thay đổi socket option

---

## 61.1 Partial Read và Write trên Stream Socket

Partial read có thể xảy ra nếu có ít byte hơn số yêu cầu trong socket buffer.

Partial write có thể xảy ra nếu không đủ buffer space và:
- Signal handler interrupt lời gọi `write()` sau khi chuyển một số byte.
- Socket hoạt động ở nonblocking mode (`O_NONBLOCK`).
- Asynchronous error xảy ra sau khi chỉ một số byte được chuyển.

Hai hàm xử lý partial I/O bằng cách restart lời gọi:

```c
#include "rdwrn.h"
ssize_t readn(int fd, void *buffer, size_t count);
ssize_t writen(int fd, void *buffer, size_t count);
```

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/rdwrn.c
#include <unistd.h>
#include <errno.h>
#include "rdwrn.h"
ssize_t
readn(int fd, void *buffer, size_t n)
{
    ssize_t numRead;
    size_t  totRead;
    char   *buf;
    buf = buffer;
    for (totRead = 0; totRead < n; ) {
        numRead = read(fd, buf, n - totRead);
        if (numRead == 0)   return totRead;
        if (numRead == -1) {
            if (errno == EINTR)  continue;
            else                 return -1;
        }
        totRead += numRead;
        buf     += numRead;
    }
    return totRead;
}
ssize_t
writen(int fd, const void *buffer, size_t n)
{
    ssize_t numWritten;
    size_t  totWritten;
    const char *buf;
    buf = buffer;
    for (totWritten = 0; totWritten < n; ) {
        numWritten = write(fd, buf, n - totWritten);
        if (numWritten <= 0) {
            if (numWritten == -1 && errno == EINTR)  continue;
            else                                     return -1;
        }
        totWritten += numWritten;
        buf        += numWritten;
    }
    return totWritten;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/rdwrn.c
```

---

## 61.2 System Call shutdown()

Gọi `close()` trên socket đóng cả hai nửa của kênh hai chiều. `shutdown()` cho phép đóng chỉ một nửa.

```c
#include <sys/socket.h>
int shutdown(int sockfd, int how);
                                             Returns 0 on success, or –1 on error
```

Giá trị `how`:
- **`SHUT_RD`**: Đóng nửa đọc. `read()` tiếp theo trả về EOF. Vẫn có thể ghi vào socket. Không thể dùng hiệu quả với TCP socket.
- **`SHUT_WR`**: Đóng nửa ghi (**socket half-close**). Peer sẽ thấy EOF sau khi đọc hết dữ liệu outstanding. `write()` tiếp theo trả về `SIGPIPE` và `EPIPE`. Đây là cách dùng phổ biến nhất của `shutdown()`.
- **`SHUT_RDWR`**: Đóng cả hai nửa.

Không giống `close()`, `shutdown()` đóng kênh socket bất kể có file descriptor nào khác tham chiếu đến socket không. Lưu ý: `shutdown()` không đóng file descriptor; phải gọi thêm `close()`.

---

## 61.3 System Call recv() và send()

```c
#include <sys/socket.h>
ssize_t recv(int sockfd, void *buffer, size_t length, int flags);
                    Returns number of bytes received, 0 on EOF, or –1 on error
ssize_t send(int sockfd, const void *buffer, size_t length, int flags);
                                    Returns number of bytes sent, or –1 on error
```

Flag cho `recv()`:
- **`MSG_DONTWAIT`**: Thực hiện recv() nonblocking; nếu không có dữ liệu, trả về ngay với `EAGAIN`.
- **`MSG_OOB`**: Nhận out-of-band data.
- **`MSG_PEEK`**: Lấy bản sao byte từ socket buffer nhưng không xóa khỏi buffer.
- **`MSG_WAITALL`**: Block cho đến khi đủ `length` byte. Vẫn có thể trả về ít hơn trong một số trường hợp.

Flag cho `send()`:
- **`MSG_DONTWAIT`**: Thực hiện send() nonblocking.
- **`MSG_MORE`** (từ Linux 2.4.4): Tương tự `TCP_CORK` nhưng trên per-call basis.
- **`MSG_NOSIGNAL`**: Không tạo `SIGPIPE` khi peer đóng kết nối; thay vào đó `send()` thất bại với `EPIPE`.
- **`MSG_OOB`**: Gửi out-of-band data.

---

## 61.4 System Call sendfile()

Ứng dụng như web server cần truyền nội dung file qua socket. Cách thông thường (read → write trong vòng lặp) không hiệu quả vì cần hai transfer: file → kernel buffer → user space → socket.

`sendfile()` truyền dữ liệu trực tiếp từ file đến socket mà không qua user space — **zero-copy transfer**.

```c
#include <sys/sendfile.h>
ssize_t sendfile(int out_fd, int in_fd, off_t *offset, size_t count);
                            Returns number of bytes transferred, or –1 on error
```

`out_fd` phải là socket; `in_fd` phải là file có thể áp dụng `mmap()` (thường là regular file).

**TCP_CORK socket option**: Để kết hợp header HTTP (ghi bằng `write()`) và nội dung file (qua `sendfile()`) vào cùng một TCP segment, dùng option `TCP_CORK`:

```c
int optval = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_CORK, &optval, sizeof(optval));
write(sockfd, ...);    /* Ghi HTTP header */
sendfile(sockfd, ...); /* Gửi nội dung file */
optval = 0;
setsockopt(sockfd, IPPROTO_TCP, TCP_CORK, &optval, sizeof(optval));
```

---

## 61.5 Lấy Địa Chỉ Socket

```c
#include <sys/socket.h>
int getsockname(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
int getpeername(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
                                        Both return 0 on success, or –1 on error
```

`getsockname()`: Trả về địa chỉ mà socket được bind. Hữu ích khi muốn biết ephemeral port number được kernel gán.

`getpeername()`: Trả về địa chỉ của peer socket — hữu ích với TCP khi server muốn biết địa chỉ client.

---

## 61.6 Xem Xét Chi Tiết TCP

### 61.6.1 Định Dạng TCP Segment

TCP segment chứa các trường: source/destination port, sequence number, acknowledgement number, header length, control bits (CWR, ECE, URG, ACK, PSH, RST, SYN, FIN), window size, checksum, urgent pointer, options, và data.

### 61.6.2 Sequence Number và Acknowledgement TCP

Mỗi byte được truyền qua TCP connection được gán sequence number logic. TCP dùng **positive acknowledgement**: khi segment nhận thành công, ACK được gửi ngược lại với `ack_number = seq_number + 1`.

### 61.6.3 TCP State Machine và State Transition Diagram

TCP endpoint được mô hình như state machine với các trạng thái:
- **LISTEN**: Chờ connection request từ peer.
- **SYN_SENT**: Đã gửi SYN, chờ reply.
- **SYN_RECV**: Nhận SYN, đã trả SYN/ACK, chờ ACK.
- **ESTABLISHED**: Kết nối thiết lập, có thể trao đổi dữ liệu.
- **FIN_WAIT1/FIN_WAIT2**: Đã gửi FIN (active close), chờ ACK.
- **CLOSING**: Cả hai bên gửi FIN đồng thời.
- **TIME_WAIT**: Đã hoàn thành active close, chờ hết timeout.
- **CLOSE_WAIT/LAST_ACK**: Passive close.

### 61.6.4 Thiết Lập Kết Nối TCP

TCP dùng **three-way handshake**:
1. Client gửi SYN với initial sequence number (M).
2. Server trả SYN/ACK (xác nhận M+1 và gửi sequence number của server N).
3. Client gửi ACK (xác nhận N+1).

### 61.6.5 Kết Thúc Kết Nối TCP

Bốn bước:
1. Active close: gửi FIN.
2. Passive close: ACK.
3. Passive close: gửi FIN.
4. Active close: ACK.

### 61.6.6 Gọi shutdown() trên TCP Socket

`SHUT_WR` hoặc `SHUT_RDWR` kích hoạt chuỗi kết thúc kết nối TCP. `SHUT_RD` không được khuyến nghị cho TCP socket do hành vi không nhất quán giữa các implementation.

### 61.6.7 Trạng Thái TIME_WAIT

TIME_WAIT tồn tại để:
1. **Đảm bảo kết thúc kết nối đáng tin cậy**: Cho phép resend ACK cuối nếu bị mất.
2. **Đảm bảo old duplicate segment hết hạn**: Ngăn new connection incarnation nhận segment cũ.

Thời gian: **2MSL** (MSL = 30 giây trên Linux → TIME_WAIT = 60 giây). Có thể gây lỗi `EADDRINUSE` khi server restart — giải quyết bằng `SO_REUSEADDR`.

---

## 61.7 Giám Sát Socket: netstat

`netstat` hiển thị trạng thái Internet và UNIX domain socket. Các option hữu ích:

| Option | Mô tả |
|--------|-------|
| `-a` | Hiển thị tất cả socket kể cả listening |
| `-l` | Chỉ hiển thị listening socket |
| `-n` | Hiển thị địa chỉ và port dưới dạng số |
| `-p` | Hiển thị PID và tên chương trình |
| `--inet` | Chỉ Internet domain socket |

```
$ netstat -a --inet
Proto  Recv-Q Send-Q  Local Address     Foreign Address   State
tcp    0      0       *:50000           *:*               LISTEN
tcp    0      0       localhost:32776   localhost:58000   TIME_WAIT
tcp    34767  0       localhost:55000   localhost:32773   ESTABLISHED
```

---

## 61.8 Sử Dụng tcpdump Để Giám Sát TCP Traffic

`tcpdump` là công cụ debug cho phép superuser giám sát Internet traffic. Với mỗi TCP segment, hiển thị:

```
src > dst: flags data-seqno ack window urg <options>
```

Ví dụ ba-way handshake:
```
IP pukaki.60391 > tekapo.55555: S 3412991013:3412991013(0) win 5840
IP tekapo.55555 > pukaki.60391: S 1149562427:1149562427(0) ack 3412991014 win 5792
IP pukaki.60391 > tekapo.55555: . ack 1 win 5840
```

---

## 61.9 Socket Option

```c
#include <sys/socket.h>
int getsockopt(int sockfd, int level, int optname, void *optval,
               socklen_t *optlen);
int setsockopt(int sockfd, int level, int optname, const void *optval,
               socklen_t optlen);
                                     Both return 0 on success, or –1 on error
```

`level` thường là `SOL_SOCKET` (option ở mức sockets API). Socket option được liên kết với open file description — file descriptor được duplicate chia sẻ cùng option.

---

## 61.10 SO_REUSEADDR Socket Option

`SO_REUSEADDR` giải quyết lỗi `EADDRINUSE` ("Address already in use") khi server restart cố bind socket vào port đang có TCP trong TIME_WAIT state.

```c
int sockfd, optval;
sockfd = socket(AF_INET, SOCK_STREAM, 0);
optval = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval));
bind(sockfd, &addr, addrlen);
listen(sockfd, backlog);
```

Hầu hết TCP server nên bật option này. Nó relaxes ràng buộc về local port reuse trong khi vẫn cho TIME_WAIT phục vụ mục đích đáng tin cậy.

---

## 61.11 Kế Thừa Flag và Option qua accept()

Socket mới được tạo bởi `accept()` **không** kế thừa:
- Status flag của open file description (`O_NONBLOCK`, `O_ASYNC`).
- File descriptor flag (`FD_CLOEXEC`).
- Thuộc tính signal-driven I/O (`F_SETOWN`, `F_SETSIG`).

Socket mới **có** kế thừa hầu hết socket option được đặt bằng `setsockopt()`.

---

## 61.12 TCP Versus UDP

Lý do chọn UDP thay TCP:
- UDP server có thể nhận từ nhiều client mà không cần tạo kết nối → overhead thấp hơn.
- Cho simple request-response, UDP nhanh hơn vì không cần connection establishment (tiết kiệm 1 RTT).
- UDP hỗ trợ broadcasting và multicasting.
- Một số ứng dụng (streaming audio/video) chấp nhận loss hơn là delay từ TCP recovery.

---

## 61.13 Các Tính Năng Nâng Cao

### 61.13.1 Out-of-Band Data

Out-of-band data cho phép sender đánh dấu dữ liệu high-priority. Gửi/nhận bằng flag `MSG_OOB`. Với TCP, tối đa 1 byte có thể là out-of-band tại một thời điểm. Hiện nay không được khuyến nghị; thay thế bằng một cặp socket riêng biệt.

### 61.13.2 sendmsg() và recvmsg()

Các system call I/O socket đa năng nhất, hỗ trợ scatter-gather I/O và truyền ancillary data.

### 61.13.3 Truyền File Descriptor

Dùng `sendmsg()`/`recvmsg()` với UNIX domain socket để truyền file descriptor giữa các process. Thực chất truyền tham chiếu đến cùng open file description.

### 61.13.4 Nhận Sender Credential

Dùng ancillary data qua UNIX domain socket để nhận credential (user ID, group ID, process ID) của sender.

### 61.13.5 Sequenced-Packet Socket

Kết hợp tính năng của stream và datagram socket: connection-oriented, bảo toàn message boundary, reliable. Tạo bằng `SOCK_SEQPACKET`.

### 61.13.6 SCTP và DCCP

- **SCTP**: Protocol transport reliable như TCP nhưng bảo toàn message boundary, hỗ trợ multistream (từ Linux 2.6).
- **DCCP**: Protocol datagram với congestion control nhưng không reliable (từ Linux 2.6.14).

---

## 61.14 Tóm Tắt

Partial read/write có thể xảy ra trên stream socket; `readn()`/`writen()` xử lý điều này.

`shutdown()` cho phép đóng một hoặc cả hai nửa kết nối hai chiều.

`recv()`/`send()` cung cấp chức năng I/O socket-specific qua đối số `flags`.

`sendfile()` truyền file đến socket hiệu quả qua zero-copy.

`getsockname()`/`getpeername()` lấy địa chỉ local và peer.

TIME_WAIT state là phần quan trọng của đảm bảo độ tin cậy TCP; `SO_REUSEADDR` giúp tránh `EADDRINUSE`.

`netstat` và `tcpdump` là công cụ hữu ích để debug.

`getsockopt()`/`setsockopt()` lấy và thay đổi socket option.

---

## 61.15 Bài Tập

- **61-1.** Vấn đề gì xảy ra nếu chương trình `is_echo_cl.c` dùng một process thay vì `fork()`?
- **61-2.** Triển khai `pipe()` bằng `socketpair()`. Dùng `shutdown()` để đảm bảo pipe là unidirectional.
- **61-3.** Triển khai thay thế `sendfile()` bằng `read()`, `write()`, và `lseek()`.
- **61-4.** Viết chương trình dùng `getsockname()` để chứng minh nếu gọi `listen()` trên TCP socket không `bind()` trước, socket được gán ephemeral port.
- **61-5.** Viết client và server cho phép client thực thi shell command tùy ý trên server.
- **61-6.** Triển khai framework dùng hai socket connection (một cho normal data, một cho priority data) thay vì out-of-band data.
