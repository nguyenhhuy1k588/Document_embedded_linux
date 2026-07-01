## Chương 57
# SOCKETS: UNIX DOMAIN

Chương này trình bày về việc sử dụng UNIX domain socket, cho phép giao tiếp giữa các process trên cùng một hệ thống. Chúng ta sẽ thảo luận về cách dùng cả stream socket và datagram socket trong UNIX domain. Ngoài ra, chúng ta sẽ mô tả cách dùng quyền file để kiểm soát truy cập vào UNIX domain socket, cách dùng `socketpair()` để tạo một cặp UNIX domain socket đã kết nối, và Linux abstract socket namespace.

# **57.1 Địa Chỉ UNIX Domain Socket: struct sockaddr\_un**

Trong UNIX domain, địa chỉ socket có dạng một đường dẫn (pathname), và cấu trúc địa chỉ socket đặc thù cho domain này được định nghĩa như sau:

```
struct sockaddr_un {
 sa_family_t sun_family; /* Always AF_UNIX */
 char sun_path[108]; /* Null-terminated socket pathname */
};
```

Tiền tố `sun_` trong các trường của cấu trúc `sockaddr_un` không liên quan đến Sun Microsystems; nó bắt nguồn từ "socket unix".

SUSv3 không chỉ định kích thước của trường `sun_path`. Các implementation BSD cổ điển dùng 108 và 104 byte, và một implementation đương đại (HP-UX 11) dùng 92 byte. Các ứng dụng portable nên code theo giá trị thấp hơn này, và dùng `snprintf()` hoặc `strncpy()` để tránh buffer overrun khi ghi vào trường này.

Để bind một UNIX domain socket vào một địa chỉ, chúng ta khởi tạo một cấu trúc `sockaddr_un`, rồi truyền con trỏ (đã ép kiểu) đến cấu trúc này làm đối số `addr` cho `bind()`, và chỉ định `addrlen` là kích thước của cấu trúc, như được minh họa trong Listing 57-1.

**Listing 57-1:** Binding a UNIX domain socket

```
 const char *SOCKNAME = "/tmp/mysock";
 int sfd;
 struct sockaddr_un addr;
 sfd = socket(AF_UNIX, SOCK_STREAM, 0); /* Create socket */
 if (sfd == -1)
 errExit("socket");
 memset(&addr, 0, sizeof(struct sockaddr_un)); /* Clear structure */
 addr.sun_family = AF_UNIX; /* UNIX domain address */
 strncpy(addr.sun_path, SOCKNAME, sizeof(addr.sun_path) - 1);
 if (bind(sfd, (struct sockaddr *) &addr, sizeof(struct sockaddr_un)) == -1)
 errExit("bind");
```

Việc dùng lời gọi `memset()` trong Listing 57-1 đảm bảo tất cả các trường của cấu trúc đều có giá trị 0. (Lệnh `strncpy()` tiếp theo tận dụng điều này bằng cách chỉ định đối số cuối cùng là một giá trị nhỏ hơn kích thước của trường `sun_path`, để đảm bảo trường này luôn có byte null kết thúc.) Việc dùng `memset()` để xóa toàn bộ cấu trúc, thay vì khởi tạo từng trường riêng lẻ, đảm bảo rằng các trường không chuẩn do một số implementation cung cấp cũng được khởi tạo về 0.

> Hàm `bzero()` có nguồn gốc từ BSD là một lựa chọn thay thế cho `memset()` khi xóa nội dung của một cấu trúc. SUSv3 chỉ định `bzero()` và hàm liên quan `bcopy()` (tương tự `memmove()`), nhưng đánh dấu cả hai hàm là LEGACY, lưu ý rằng `memset()` và `memmove()` được ưa thích hơn. SUSv4 loại bỏ các đặc tả của `bzero()` và `bcopy()`.

Khi dùng để bind một UNIX domain socket, `bind()` tạo một mục trong hệ thống file. (Do đó, một thư mục được chỉ định là một phần của pathname socket cần có thể truy cập và ghi được.) Quyền sở hữu file được xác định theo các quy tắc thông thường cho việc tạo file (Mục 15.3.1). File được đánh dấu là socket. Khi `stat()` được áp dụng cho pathname này, nó trả về giá trị `S_IFSOCK` trong thành phần file-type của trường `st_mode` trong cấu trúc `stat` (Mục 15.1). Khi liệt kê bằng `ls –l`, một UNIX domain socket hiển thị với kiểu `s` trong cột đầu tiên, và `ls –F` thêm dấu bằng (=) vào sau pathname socket.

> Mặc dù UNIX domain socket được nhận dạng bằng pathname, nhưng I/O trên các socket này không liên quan đến các thao tác trên thiết bị vật lý bên dưới.

Các điểm sau đáng lưu ý khi binding một UNIX domain socket:

- Chúng ta không thể bind một socket vào một pathname đã tồn tại (`bind()` sẽ thất bại với lỗi `EADDRINUSE`).
- Thông thường, nên bind một socket vào một pathname tuyệt đối, để socket nằm ở một địa chỉ cố định trong hệ thống file. Việc dùng pathname tương đối là có thể nhưng không phổ biến, vì nó đòi hỏi ứng dụng muốn `connect()` đến socket này phải biết thư mục làm việc hiện tại của ứng dụng thực hiện `bind()`.
- Một socket chỉ có thể bind vào một pathname; ngược lại, một pathname chỉ có thể bind vào một socket.
- Chúng ta không thể dùng `open()` để mở một socket.
- Khi socket không còn cần thiết, mục pathname của nó có thể (và thường nên) được xóa bằng `unlink()` (hoặc `remove()`).

Trong hầu hết các chương trình ví dụ, chúng ta bind UNIX domain socket vào pathname trong thư mục `/tmp`, vì thư mục này thường có mặt và có thể ghi được trên mọi hệ thống. Điều này giúp người đọc dễ dàng chạy các chương trình này mà không cần phải sửa pathname của socket trước. Tuy nhiên, hãy lưu ý rằng đây thường không phải là kỹ thuật thiết kế tốt. Như đã chỉ ra trong Mục 38.7, việc tạo file trong các thư mục có thể ghi công khai như `/tmp` có thể dẫn đến các lỗ hổng bảo mật. Ví dụ, bằng cách tạo một pathname trong `/tmp` với cùng tên được dùng bởi socket ứng dụng, chúng ta có thể tạo ra một cuộc tấn công từ chối dịch vụ đơn giản. Các ứng dụng thực tế nên `bind()` UNIX domain socket vào pathname tuyệt đối trong các thư mục được bảo mật đúng cách.

# **57.2 Stream Socket trong UNIX Domain**

Chúng ta sẽ trình bày một ứng dụng client-server đơn giản sử dụng stream socket trong UNIX domain. Chương trình client (Listing 57-4) kết nối đến server, và dùng kết nối đó để truyền dữ liệu từ đầu vào chuẩn đến server. Chương trình server (Listing 57-3) chấp nhận kết nối từ client, và chuyển tất cả dữ liệu được gửi qua kết nối bởi client đến đầu ra chuẩn. Server là một ví dụ đơn giản về iterative server—một server xử lý một client tại một thời điểm trước khi chuyển sang client tiếp theo. (Chúng ta sẽ xem xét thiết kế server chi tiết hơn trong Chương 60.)

Listing 57-2 là file header được dùng bởi cả hai chương trình này.

**Listing 57-2:** Header file for us\_xfr\_sv.c and us\_xfr\_cl.c ––––––––––––––––––––––––––––––––––––––––––––––––––––––––– **sockets/us\_xfr.h** #include <sys/un.h> #include <sys/socket.h> #include "tlpi\_hdr.h" #define SV\_SOCK\_PATH "/tmp/us\_xfr" #define BUF\_SIZE 100 ––––––––––––––––––––––––––––––––––––––––––––––––––––––––– **sockets/us\_xfr.h**

Trong các trang tiếp theo, chúng ta trình bày source code của server và client, sau đó thảo luận chi tiết về các chương trình này và trình bày ví dụ về cách sử dụng.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/us_xfr_sv.c
#include "us_xfr.h"
#define BACKLOG 5
int
main(int argc, char *argv[])
{
 struct sockaddr_un addr;
 int sfd, cfd;
 ssize_t numRead;
 char buf[BUF_SIZE];
 sfd = socket(AF_UNIX, SOCK_STREAM, 0);
 if (sfd == -1)
 errExit("socket");
 /* Construct server socket address, bind socket to it,
 and make this a listening socket */
 if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT)
 errExit("remove-%s", SV_SOCK_PATH);
 memset(&addr, 0, sizeof(struct sockaddr_un));
 addr.sun_family = AF_UNIX;
 strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);
 if (bind(sfd, (struct sockaddr *) &addr, sizeof(struct sockaddr_un)) == -1)
 errExit("bind");
 if (listen(sfd, BACKLOG) == -1)
 errExit("listen");
 for (;;) { /* Handle client connections iteratively */
 /* Accept a connection. The connection is returned on a new
 socket, 'cfd'; the listening socket ('sfd') remains open
 and can be used to accept further connections. */
 cfd = accept(sfd, NULL, NULL);
 if (cfd == -1)
 errExit("accept");
 /* Transfer data from connected socket to stdout until EOF */
 while ((numRead = read(cfd, buf, BUF_SIZE)) > 0)
 if (write(STDOUT_FILENO, buf, numRead) != numRead)
 fatal("partial/failed write");
 if (numRead == -1)
 errExit("read");
```

```
 if (close(cfd) == -1)
 errMsg("close");
 }
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/us_xfr_sv.c
Listing 57-4: A simple UNIX domain stream socket client
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/us_xfr_cl.c
#include "us_xfr.h"
int
main(int argc, char *argv[])
{
 struct sockaddr_un addr;
 int sfd;
 ssize_t numRead;
 char buf[BUF_SIZE];
 sfd = socket(AF_UNIX, SOCK_STREAM, 0); /* Create client socket */
 if (sfd == -1)
 errExit("socket");
 /* Construct server address, and make the connection */
 memset(&addr, 0, sizeof(struct sockaddr_un));
 addr.sun_family = AF_UNIX;
 strncpy(addr.sun_path, SV_SOCK_PATH, sizeof(addr.sun_path) - 1);
 if (connect(sfd, (struct sockaddr *) &addr,
 sizeof(struct sockaddr_un)) == -1)
 errExit("connect");
 /* Copy stdin to socket */
 while ((numRead = read(STDIN_FILENO, buf, BUF_SIZE)) > 0)
 if (write(sfd, buf, numRead) != numRead)
 fatal("partial/failed write");
 if (numRead == -1)
 errExit("read");
 exit(EXIT_SUCCESS); /* Closes our socket; server sees EOF */
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/us_xfr_cl.c
```

Chương trình server được hiển thị trong Listing 57-3. Server thực hiện các bước sau:

- Tạo một socket.
- Xóa bất kỳ file nào có cùng pathname với pathname chúng ta muốn bind socket vào.
- Tạo cấu trúc địa chỉ cho socket của server, bind socket vào địa chỉ đó, và đánh dấu socket là listening socket.
- Thực thi một vòng lặp vô hạn để xử lý các yêu cầu client đến. Mỗi lần lặp thực hiện các bước sau:
  - Chấp nhận kết nối, lấy socket mới `cfd` cho kết nối.
  - Đọc tất cả dữ liệu từ socket đã kết nối và ghi ra đầu ra chuẩn.
  - Đóng socket đã kết nối `cfd`.

Server phải được kết thúc thủ công (ví dụ: bằng cách gửi cho nó một signal). Chương trình client (Listing 57-4) thực hiện các bước sau:

- Tạo một socket.
- Tạo cấu trúc địa chỉ cho socket của server và kết nối đến socket tại địa chỉ đó.
- Thực thi một vòng lặp sao chép đầu vào chuẩn của nó vào kết nối socket. Khi gặp end-of-file trong đầu vào chuẩn, client kết thúc, kết quả là socket của nó bị đóng và server thấy end-of-file khi đọc từ socket ở đầu kia của kết nối.

Phiên làm việc shell dưới đây minh họa cách sử dụng các chương trình này. Chúng ta bắt đầu bằng cách chạy server ở chế độ nền:

```
$ ./us_xfr_sv > b &
[1] 9866
$ ls -lF /tmp/us_xfr Examine socket file with ls
srwxr-xr-x 1 mtk users 0 Jul 18 10:48 /tmp/us_xfr=
```

Sau đó chúng ta tạo file test để làm đầu vào cho client, và chạy client:

```
$ cat *.c > a
$ ./us_xfr_cl < a Client takes input from test file
```

Lúc này, client đã hoàn thành. Bây giờ chúng ta kết thúc server và kiểm tra rằng output của server khớp với input của client:

```
$ kill %1 Terminate server
 [1]+ Terminated ./us_xfr_sv >b Shell sees server's termination
$ diff a b
$
```

Lệnh `diff` không tạo ra output, cho thấy rằng các file input và output là giống hệt nhau.

Lưu ý rằng sau khi server kết thúc, pathname socket vẫn tồn tại. Đây là lý do tại sao server dùng `remove()` để xóa bất kỳ instance nào của pathname socket trước khi gọi `bind()`. (Giả sử chúng ta có quyền phù hợp, lời gọi `remove()` này sẽ xóa bất kỳ loại file nào có pathname này, dù nó không phải là socket.) Nếu không làm vậy, thì lời gọi `bind()` sẽ thất bại nếu lần gọi server trước đó đã tạo ra pathname socket này.

# **57.3 Datagram Socket trong UNIX Domain**

Trong phần mô tả chung về datagram socket mà chúng ta đã trình bày trong Mục 56.6, chúng ta đã nêu rằng giao tiếp sử dụng datagram socket là không đáng tin cậy. Điều này đúng với datagram được truyền qua mạng. Tuy nhiên, đối với UNIX domain socket, việc truyền datagram được thực hiện trong kernel và là đáng tin cậy. Tất cả các message được giao theo thứ tự và không bị trùng lặp.

#### **Kích thước datagram tối đa cho UNIX domain datagram socket**

SUSv3 không chỉ định kích thước tối đa cho datagram được gửi qua UNIX domain socket. Trên Linux, chúng ta có thể gửi datagram khá lớn. Các giới hạn được kiểm soát thông qua tùy chọn socket `SO_SNDBUF` và các file `/proc` khác nhau, như được mô tả trong trang manual `socket(7)`. Tuy nhiên, một số implementation UNIX khác áp đặt giới hạn thấp hơn, chẳng hạn như 2048 byte. Các ứng dụng portable sử dụng UNIX domain datagram socket nên cân nhắc áp đặt giới hạn trên thấp cho kích thước datagram được sử dụng.

### **Chương trình ví dụ**

Listing 57-6 và Listing 57-7 minh họa một ứng dụng client-server đơn giản sử dụng UNIX domain datagram socket. Cả hai chương trình này đều sử dụng file header được hiển thị trong Listing 57-5.

```
Listing 57-5: Header file used by ud_ucase_sv.c and ud_ucase_cl.c
––––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/ud_ucase.h
#include <sys/un.h>
#include <sys/socket.h>
#include <ctype.h>
#include "tlpi_hdr.h"
#define BUF_SIZE 10 /* Maximum size of messages exchanged
 between client to server */
#define SV_SOCK_PATH "/tmp/ud_ucase"
––––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/ud_ucase.h
```

Chương trình server (Listing 57-6) trước tiên tạo một socket và bind nó vào một địa chỉ well-known. (Trước đó, server unlink pathname trùng với địa chỉ đó, phòng trường hợp pathname đã tồn tại.) Server sau đó vào một vòng lặp vô hạn, dùng `recvfrom()` để nhận datagram từ client, chuyển đổi văn bản nhận được sang chữ hoa, và trả văn bản đã chuyển đổi về cho client bằng địa chỉ lấy được qua `recvfrom()`.

Chương trình client (Listing 57-7) tạo một socket và bind socket vào một địa chỉ, để server có thể gửi phản hồi của nó. Địa chỉ client được tạo unique bằng cách đưa process ID của client vào pathname. Client sau đó lặp, gửi từng đối số dòng lệnh của nó như một message riêng biệt đến server. Sau khi gửi mỗi message, client đọc phản hồi server và hiển thị nó trên đầu ra chuẩn.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/ud_ucase_sv.c
#include "ud_ucase.h"
int
main(int argc, char *argv[])
{
 struct sockaddr_un svaddr, claddr;
 int sfd, j;
 ssize_t numBytes;
 socklen_t len;
 char buf[BUF_SIZE];
 sfd = socket(AF_UNIX, SOCK_DGRAM, 0); /* Create server socket */
 if (sfd == -1)
 errExit("socket");
 /* Construct well-known address and bind server socket to it */
 if (remove(SV_SOCK_PATH) == -1 && errno != ENOENT)
 errExit("remove-%s", SV_SOCK_PATH);
 memset(&svaddr, 0, sizeof(struct sockaddr_un));
 svaddr.sun_family = AF_UNIX;
 strncpy(svaddr.sun_path, SV_SOCK_PATH, sizeof(svaddr.sun_path) - 1);
 if (bind(sfd, (struct sockaddr *) &svaddr, sizeof(struct sockaddr_un)) == -1)
 errExit("bind");
 /* Receive messages, convert to uppercase, and return to client */
 for (;;) {
 len = sizeof(struct sockaddr_un);
 numBytes = recvfrom(sfd, buf, BUF_SIZE, 0,
 (struct sockaddr *) &claddr, &len);
 if (numBytes == -1)
 errExit("recvfrom");
 printf("Server received %ld bytes from %s\n", (long) numBytes,
 claddr.sun_path);
 for (j = 0; j < numBytes; j++)
 buf[j] = toupper((unsigned char) buf[j]);
 if (sendto(sfd, buf, numBytes, 0, (struct sockaddr *) &claddr, len) !=
 numBytes)
 fatal("sendto");
 }
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/ud_ucase_sv.c
```

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/ud_ucase_cl.c
#include "ud_ucase.h"
int
main(int argc, char *argv[])
{
 struct sockaddr_un svaddr, claddr;
 int sfd, j;
 size_t msgLen;
 ssize_t numBytes;
 char resp[BUF_SIZE];
 if (argc < 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s msg...\n", argv[0]);
 /* Create client socket; bind to unique pathname (based on PID) */
 sfd = socket(AF_UNIX, SOCK_DGRAM, 0);
 if (sfd == -1)
 errExit("socket");
 memset(&claddr, 0, sizeof(struct sockaddr_un));
 claddr.sun_family = AF_UNIX;
 snprintf(claddr.sun_path, sizeof(claddr.sun_path),
 "/tmp/ud_ucase_cl.%ld", (long) getpid());
 if (bind(sfd, (struct sockaddr *) &claddr, sizeof(struct sockaddr_un)) == -1)
 errExit("bind");
 /* Construct address of server */
 memset(&svaddr, 0, sizeof(struct sockaddr_un));
 svaddr.sun_family = AF_UNIX;
 strncpy(svaddr.sun_path, SV_SOCK_PATH, sizeof(svaddr.sun_path) - 1);
 /* Send messages to server; echo responses on stdout */
 for (j = 1; j < argc; j++) {
 msgLen = strlen(argv[j]); /* May be longer than BUF_SIZE */
 if (sendto(sfd, argv[j], msgLen, 0, (struct sockaddr *) &svaddr,
 sizeof(struct sockaddr_un)) != msgLen)
 fatal("sendto");
 numBytes = recvfrom(sfd, resp, BUF_SIZE, 0, NULL, NULL);
 if (numBytes == -1)
 errExit("recvfrom");
 printf("Response %d: %.*s\n", j, (int) numBytes, resp);
 }
 remove(claddr.sun_path); /* Remove client socket pathname */
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/ud_ucase_cl.c
```

Phiên làm việc shell dưới đây minh họa cách sử dụng chương trình server và client:

```
$ ./ud_ucase_sv &
[1] 20113
$ ./ud_ucase_cl hello world Send 2 messages to server
Server received 5 bytes from /tmp/ud_ucase_cl.20150
Response 1: HELLO
Server received 5 bytes from /tmp/ud_ucase_cl.20150
Response 2: WORLD
$ ./ud_ucase_cl 'long message' Send 1 longer message to server
Server received 10 bytes from /tmp/ud_ucase_cl.20151
Response 1: LONG MESSA
$ kill %1 Terminate server
```

Lần gọi thứ hai của chương trình client được thiết kế để chứng minh rằng khi lời gọi `recvfrom()` chỉ định độ dài (`BUF_SIZE`, được định nghĩa trong Listing 57-5 với giá trị 10) ngắn hơn kích thước message, message sẽ bị cắt bớt im lặng. Chúng ta có thể thấy điều này đã xảy ra, vì server in ra message nói rằng nó chỉ nhận được 10 byte, trong khi message được gửi bởi client bao gồm 12 byte.

# **57.4 Quyền UNIX Domain Socket**

Quyền sở hữu và các quyền của file socket xác định process nào có thể giao tiếp với socket đó:

- Để kết nối đến một UNIX domain stream socket, cần có quyền ghi trên file socket.
- Để gửi datagram đến một UNIX domain datagram socket, cần có quyền ghi trên file socket.

Ngoài ra, cần có quyền thực thi (tìm kiếm) trên mỗi thư mục trong pathname socket.

Theo mặc định, một socket được tạo (bởi `bind()`) với tất cả quyền được cấp cho owner (user), group và other. Để thay đổi điều này, chúng ta có thể gọi `umask()` trước lời gọi `bind()` để tắt các quyền mà chúng ta không muốn cấp.

Một số hệ thống bỏ qua các quyền trên file socket (SUSv3 cho phép điều này). Do đó, chúng ta không thể dùng quyền file socket một cách portable để kiểm soát truy cập vào socket, mặc dù chúng ta có thể dùng một cách portable các quyền trên thư mục chứa socket cho mục đích này.

# **57.5 Tạo Một Cặp Socket Đã Kết Nối: socketpair()**

Đôi khi, sẽ rất hữu ích khi một single process tạo ra một cặp socket và kết nối chúng với nhau. Điều này có thể được thực hiện bằng hai lời gọi `socket()`, một lời gọi `bind()`, và sau đó là các lời gọi `listen()`, `connect()`, và `accept()` (cho stream socket), hoặc một lời gọi `connect()` (cho datagram socket). System call `socketpair()` cung cấp một cách viết tắt cho thao tác này.

```
#include <sys/socket.h>
int socketpair(int domain, int type, int protocol, int sockfd[2]);
                                             Returns 0 on success, or –1 on error
```

System call `socketpair()` này chỉ có thể được dùng trong UNIX domain; tức là, `domain` phải được chỉ định là `AF_UNIX`. (Hạn chế này áp dụng trên hầu hết các implementation, và là hợp lý, vì cặp socket được tạo trên một host duy nhất.) Kiểu socket có thể được chỉ định là `SOCK_DGRAM` hoặc `SOCK_STREAM`. Đối số `protocol` phải được chỉ định là 0. Mảng `sockfd` trả về các file descriptor tham chiếu đến hai socket đã kết nối.

Khi chỉ định `type` là `SOCK_STREAM`, điều này tạo ra tương đương của một bidirectional pipe (còn được gọi là stream pipe). Mỗi socket có thể được dùng để đọc và ghi, và các kênh dữ liệu riêng biệt chảy theo mỗi hướng giữa hai socket. (Trên các implementation có nguồn gốc từ BSD, `pipe()` được implement như một lời gọi đến `socketpair()`.)

Thông thường, một cặp socket được dùng theo cách tương tự như một pipe. Sau lời gọi `socketpair()`, process tạo ra một process con thông qua `fork()`. Process con kế thừa các bản sao của các file descriptor của process cha, bao gồm các descriptor tham chiếu đến cặp socket. Do đó, process cha và process con có thể dùng cặp socket cho IPC.

Một điểm khác biệt giữa việc dùng `socketpair()` và tạo thủ công một cặp socket đã kết nối là các socket không được bind vào bất kỳ địa chỉ nào. Điều này có thể giúp chúng ta tránh một lớp các lỗ hổng bảo mật, vì các socket không hiển thị với bất kỳ process nào khác.

> Bắt đầu từ kernel 2.6.27, Linux cung cấp một cách dùng thứ hai cho đối số `type`, bằng cách cho phép hai flag không chuẩn được OR với kiểu socket. Flag `SOCK_CLOEXEC` khiến kernel kích hoạt flag close-on-exec (`FD_CLOEXEC`) cho hai file descriptor mới. Flag này hữu ích vì các lý do tương tự như flag `O_CLOEXEC` của `open()` được mô tả trong Mục 4.3.1. Flag `SOCK_NONBLOCK` khiến kernel đặt flag `O_NONBLOCK` trên cả hai open file description bên dưới, để các thao tác I/O tương lai trên socket sẽ là nonblocking. Điều này tiết kiệm các lời gọi `fcntl()` bổ sung để đạt được kết quả tương tự.

# **57.6 Linux Abstract Socket Namespace**

Cái gọi là abstract namespace là một tính năng dành riêng cho Linux cho phép chúng ta bind một UNIX domain socket vào một tên mà tên đó không được tạo trong hệ thống file. Điều này cung cấp một số lợi thế tiềm năng:

- Chúng ta không cần lo lắng về khả năng xung đột với các tên hiện có trong hệ thống file.
- Không cần phải unlink pathname socket khi chúng ta đã dùng xong socket. Tên abstract sẽ tự động bị xóa khi socket bị đóng.
- Chúng ta không cần tạo một pathname hệ thống file cho socket. Điều này có thể hữu ích trong môi trường chroot, hoặc nếu chúng ta không có quyền ghi vào hệ thống file.

Để tạo một abstract binding, chúng ta chỉ định byte đầu tiên của trường `sun_path` là một null byte (`\0`). Điều này phân biệt các tên socket abstract với pathname UNIX domain socket thông thường, bao gồm một chuỗi một hoặc nhiều byte non-null kết thúc bởi một null byte. Các byte còn lại của trường `sun_path` sau đó định nghĩa tên abstract cho socket. Các byte này được diễn giải trong toàn bộ, thay vì như một chuỗi kết thúc bằng null.

Listing 57-8 minh họa việc tạo một abstract socket binding.

**Listing 57-8:** Creating an abstract socket binding

```
––––––––––––––––––––––––––––––––––––––––––––– from sockets/us_abstract_bind.c
 struct sockaddr_un addr;
 memset(&addr, 0, sizeof(struct sockaddr_un)); /* Clear address structure */
 addr.sun_family = AF_UNIX; /* UNIX domain address */
 /* addr.sun_path[0] has already been set to 0 by memset() */
 strncpy(&addr.sun_path[1], "xyz", sizeof(addr.sun_path) - 2);
 /* Abstract name is "xyz" followed by null bytes */
 sockfd = socket(AF_UNIX, SOCK_STREAM, 0);
 if (sockfd == -1)
 errExit("socket");
 if (bind(sockfd, (struct sockaddr *) &addr,
 sizeof(struct sockaddr_un)) == -1)
 errExit("bind");
––––––––––––––––––––––––––––––––––––––––––––– from sockets/us_abstract_bind.c
```

Việc một null byte đầu tiên được dùng để phân biệt tên socket abstract với tên socket thông thường có thể dẫn đến một hậu quả bất thường. Giả sử biến `name` tình cờ trỏ đến một chuỗi có độ dài bằng không và chúng ta cố gắng bind một UNIX domain socket vào một `sun_path` được khởi tạo như sau:

```
strncpy(addr.sun_path, name, sizeof(addr.sun_path) - 1);
```

Trên Linux, chúng ta sẽ vô tình tạo ra một abstract socket binding. Tuy nhiên, một chuỗi code như vậy có lẽ là không cố ý (tức là, một bug). Trên các implementation UNIX khác, lời gọi `bind()` tiếp theo sẽ thất bại.

# **57.7 Tóm Tắt**

UNIX domain socket cho phép giao tiếp giữa các ứng dụng trên cùng một host. UNIX domain hỗ trợ cả stream socket và datagram socket.

Một UNIX domain socket được nhận dạng bằng một pathname trong hệ thống file. Quyền file có thể được dùng để kiểm soát truy cập vào một UNIX domain socket.

System call `socketpair()` tạo ra một cặp UNIX domain socket đã kết nối. Điều này tránh được nhu cầu phải gọi nhiều system call để tạo, bind và kết nối các socket. Một cặp socket thường được dùng theo cách tương tự như một pipe: một process tạo ra cặp socket và sau đó `fork()` để tạo ra một process con kế thừa các descriptor tham chiếu đến các socket. Hai process sau đó có thể giao tiếp qua cặp socket.

Linux-specific abstract socket namespace cho phép chúng ta bind một UNIX domain socket vào một tên không xuất hiện trong hệ thống file.

## **Tài liệu tham khảo**

Tham khảo các nguồn thông tin thêm được liệt kê trong Mục 59.15.

# **57.8 Bài Tập**

- **57-1.** Trong Mục 57.3, chúng ta đã lưu ý rằng UNIX domain datagram socket là đáng tin cậy. Viết các chương trình để chứng minh rằng nếu một sender truyền datagram đến một UNIX domain datagram socket nhanh hơn receiver đọc chúng, thì sender cuối cùng sẽ bị block, và vẫn bị block cho đến khi receiver đọc một số datagram đang chờ.
- **57-2.** Viết lại các chương trình trong Listing 57-3 (us\_xfr\_sv.c) và Listing 57-4 (us\_xfr\_cl.c) để sử dụng Linux-specific abstract socket namespace (Mục 57.6).
- **57-3.** Implement lại server và client số thứ tự của Mục 44.8 bằng UNIX domain stream socket.
- **57-4.** Giả sử chúng ta tạo hai UNIX domain datagram socket được bind vào các path `/somepath/a` và `/somepath/b`, và chúng ta kết nối socket `/somepath/a` đến `/somepath/b`. Điều gì xảy ra nếu chúng ta tạo socket datagram thứ ba và cố gắng gửi (`sendto()`) một datagram qua socket đó đến `/somepath/a`? Viết một chương trình để xác định câu trả lời. Nếu bạn có quyền truy cập vào các hệ thống UNIX khác, hãy kiểm tra chương trình trên các hệ thống đó để xem liệu câu trả lời có khác không.
