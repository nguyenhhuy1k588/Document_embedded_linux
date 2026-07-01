## Chương 60
# <span id="page-90-0"></span>**SOCKET: THIẾT KẾ SERVER**

Chương này thảo luận về các nguyên tắc cơ bản trong việc thiết kế iterative server và concurrent server, đồng thời mô tả inetd — một daemon đặc biệt được thiết kế để hỗ trợ việc tạo ra các Internet server.

# **60.1 Iterative Server và Concurrent Server**

Hai mô hình thiết kế phổ biến cho network server sử dụng socket như sau:

-  Iterative: Server xử lý từng client một, hoàn thành toàn bộ yêu cầu của client đó trước khi chuyển sang client tiếp theo.
-  Concurrent: Server được thiết kế để xử lý nhiều client cùng một lúc.

Chúng ta đã thấy một ví dụ về iterative server sử dụng FIFO trong Mục 44.8 và một ví dụ về concurrent server sử dụng System V message queue trong Mục 46.8.

Iterative server thường chỉ phù hợp khi các yêu cầu của client có thể được xử lý nhanh chóng, vì mỗi client phải chờ đến khi tất cả các client trước đó đã được phục vụ xong. Một tình huống điển hình để sử dụng iterative server là khi client và server trao đổi một cặp request/response duy nhất.

Concurrent server phù hợp khi cần một lượng thời gian xử lý đáng kể cho mỗi request, hoặc khi client và server tiến hành một cuộc trao đổi kéo dài, gửi qua lại nhiều message. Trong chương này, chúng ta chủ yếu tập trung vào phương pháp truyền thống (và đơn giản nhất) để thiết kế concurrent server: tạo một child process mới cho mỗi client mới. Mỗi child process của server thực hiện tất cả các tác vụ cần thiết để phục vụ một client duy nhất, sau đó kết thúc. Vì mỗi process này có thể hoạt động độc lập, nên nhiều client có thể được phục vụ đồng thời. Nhiệm vụ chính của main server process (process cha) là tạo ra một child process mới cho mỗi client mới. (Một biến thể của cách tiếp cận này là tạo một thread mới cho mỗi client.)

Trong các mục tiếp theo, chúng ta xem xét các ví dụ về iterative server và concurrent server sử dụng Internet domain socket. Hai server này triển khai dịch vụ echo (RFC 862), một dịch vụ đơn giản trả lại bản sao của bất kỳ dữ liệu nào client gửi đến.

# **60.2 Một Iterative UDP echo Server**

Trong mục này và mục tiếp theo, chúng ta trình bày các server cho dịch vụ echo. Dịch vụ echo hoạt động trên cả UDP và TCP port 7. (Vì đây là reserved port, echo server phải được chạy với quyền superuser.)

UDP echo server liên tục đọc các datagram, trả lại bản sao của mỗi datagram cho người gửi. Vì server chỉ cần xử lý một message tại một thời điểm, thiết kế iterative server là đủ. File header cho server được trình bày trong [Listing 60-1.](#page-91-0)

<span id="page-91-0"></span>**Listing 60-1:** File header cho id\_echo\_sv.c và id\_echo\_cl.c

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/id_echo.h
#include "inet_sockets.h" /* Declares our socket functions */
#include "tlpi_hdr.h"
#define SERVICE "echo" /* Name of UDP service */
#define BUF_SIZE 500 /* Maximum size of datagrams that can
 be read by client and server */
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/id_echo.h
```

[Listing 60-2](#page-92-0) trình bày cách triển khai server. Lưu ý các điểm sau đây liên quan đến việc triển khai server:

-  Chúng ta sử dụng hàm `becomeDaemon()` trong Mục 37.2 để biến server thành một daemon.
-  Để rút gọn chương trình này, chúng ta sử dụng thư viện Internet domain socket được phát triển trong Mục [59.12.](#page-76-1)
-  Nếu server không thể gửi phản hồi cho client, nó ghi lại một message bằng `syslog()`.

Trong một ứng dụng thực tế, chúng ta có thể sẽ áp dụng một số giới hạn tần suất cho các message được ghi bằng `syslog()`, cả để ngăn chặn khả năng kẻ tấn công làm đầy system log và vì mỗi lần gọi `syslog()` rất tốn kém, vì (theo mặc định) `syslog()` lại gọi `fsync()`.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/id_echo_sv.c
#include <syslog.h>
#include "id_echo.h"
#include "become_daemon.h"
int
main(int argc, char *argv[])
{
 int sfd;
 ssize_t numRead;
 socklen_t addrlen, len;
 struct sockaddr_storage claddr;
 char buf[BUF_SIZE];
 char addrStr[IS_ADDR_STR_LEN];
 if (becomeDaemon(0) == -1)
 errExit("becomeDaemon");
 sfd = inetBind(SERVICE, SOCK_DGRAM, &addrlen);
 if (sfd == -1) {
 syslog(LOG_ERR, "Could not create server socket (%s)", strerror(errno));
 exit(EXIT_FAILURE);
 }
   /* Receive datagrams and return copies to senders */
 for (;;) {
 len = sizeof(struct sockaddr_storage);
 numRead = recvfrom(sfd, buf, BUF_SIZE, 0,
 (struct sockaddr *) &claddr, &len);
 if (numRead == -1)
 errExit("recvfrom");
 if (sendto(sfd, buf, numRead, 0, (struct sockaddr *) &claddr, len)
 != numRead)
 syslog(LOG_WARNING, "Error echoing response to %s (%s)",
 inetAddressStr((struct sockaddr *) &claddr, len,
 addrStr, IS_ADDR_STR_LEN),
 strerror(errno));
 }
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/id_echo_sv.c
```

Để kiểm tra server, chúng ta sử dụng chương trình client được trình bày trong [Listing 60-3.](#page-93-0) Chương trình này cũng sử dụng thư viện Internet domain socket được phát triển trong Mục [59.12.](#page-76-1) Đối số dòng lệnh đầu tiên của chương trình client là tên của host nơi server đang chạy. Client thực hiện một vòng lặp, trong đó nó gửi từng đối số dòng lệnh còn lại cho server dưới dạng các datagram riêng biệt, và đọc rồi in ra mỗi datagram phản hồi mà server gửi lại.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/id_echo_cl.c
#include "id_echo.h"
int
main(int argc, char *argv[])
{
 int sfd, j;
 size_t len;
 ssize_t numRead;
 char buf[BUF_SIZE];
 if (argc < 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s: host msg...\n", argv[0]);
   /* Construct server address from first command-line argument */
 sfd = inetConnect(argv[1], SERVICE, SOCK_DGRAM);
 if (sfd == -1)
 fatal("Could not connect to server socket");
   /* Send remaining command-line arguments to server as separate datagrams */
 for (j = 2; j < argc; j++) {
 len = strlen(argv[j]);
 if (write(sfd, argv[j], len) != len)
 fatal("partial/failed write");
 numRead = read(sfd, buf, BUF_SIZE);
 if (numRead == -1)
 errExit("read");
 printf("[%ld bytes] %.*s\n", (long) numRead, (int) numRead, buf);
 }
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/id_echo_cl.c
```

Dưới đây là ví dụ về những gì chúng ta thấy khi chạy server và hai phiên bản client:

```
$ su Need privilege to bind reserved port
Password:
# ./id_echo_sv Server places itself in background
# exit Cease to be superuser
$ ./id_echo_cl localhost hello world This client sends two datagrams
[5 bytes] hello Client prints responses from server
[5 bytes] world
$ ./id_echo_cl localhost goodbye This client sends one datagram
[7 bytes] goodbye
```

# **60.3 Một Concurrent TCP echo Server**

<span id="page-94-0"></span>Dịch vụ TCP echo cũng hoạt động trên port 7. TCP echo server chấp nhận một kết nối và sau đó lặp liên tục, đọc tất cả dữ liệu được truyền và gửi lại cho client trên cùng socket đó. Server tiếp tục đọc cho đến khi phát hiện end-of-file, lúc đó nó đóng socket của mình (để client cũng thấy end-of-file nếu nó vẫn đang đọc từ socket của nó).

Vì client có thể gửi một lượng dữ liệu không xác định cho server (và do đó việc phục vụ client có thể mất một lượng thời gian không xác định), thiết kế concurrent server là phù hợp để nhiều client có thể được phục vụ đồng thời. Cách triển khai server được trình bày trong [Listing 60-4](#page-95-0). (Chúng ta trình bày cách triển khai một client cho dịch vụ này trong Mục [61.2](#page-107-0).) Lưu ý các điểm sau đây về cách triển khai:

-  Server trở thành daemon bằng cách gọi hàm `becomeDaemon()` được trình bày trong Mục 37.2.
-  Để rút gọn chương trình này, chúng ta sử dụng thư viện Internet domain socket được trình bày trong [Listing 59-9](#page-79-1) (trang [1228\)](#page-79-1).
-  Vì server tạo ra một child process cho mỗi kết nối client, chúng ta phải đảm bảo rằng các zombie được thu hồi. Chúng ta thực hiện điều này trong một SIGCHLD handler.
-  Phần thân chính của server bao gồm một vòng lặp `for` chấp nhận một kết nối client và sau đó sử dụng `fork()` để tạo ra một child process gọi hàm `handleRequest()` để xử lý client đó. Trong khi đó, process cha tiếp tục vòng lặp `for` để chấp nhận kết nối client tiếp theo.

Trong một ứng dụng thực tế, chúng ta có thể sẽ bao gồm một số code trong server để đặt giới hạn trên cho số lượng child process mà server có thể tạo, nhằm ngăn chặn kẻ tấn công cố gắng thực hiện remote fork bomb bằng cách sử dụng dịch vụ để tạo ra nhiều process trên hệ thống đến mức nó trở nên không sử dụng được. Chúng ta có thể áp đặt giới hạn này bằng cách thêm code bổ sung trong server để đếm số lượng child đang thực thi (số lượng này sẽ được tăng lên sau một `fork()` thành công và giảm xuống khi mỗi child được thu hồi trong SIGCHLD handler). Nếu đạt đến giới hạn số lượng child, chúng ta có thể tạm thời dừng chấp nhận kết nối (hoặc thay vào đó, chấp nhận kết nối và sau đó đóng ngay lập tức).

-  Sau mỗi lần `fork()`, các file descriptor cho listening socket và connected socket được nhân bản trong child (Mục 24.2.1). Điều này có nghĩa là cả process cha và process con đều có thể giao tiếp với client bằng connected socket. Tuy nhiên, chỉ có process con cần thực hiện giao tiếp đó, vì vậy process cha đóng file descriptor cho connected socket ngay sau khi `fork()`. (Nếu process cha không làm điều này, thì socket sẽ không bao giờ thực sự được đóng; hơn nữa, process cha cuối cùng sẽ hết file descriptor.) Vì process con không chấp nhận kết nối mới, nó đóng bản sao file descriptor cho listening socket của mình.
-  Mỗi child process kết thúc sau khi xử lý một client duy nhất.

<span id="page-95-0"></span>**Listing 60-4:** Một concurrent server triển khai dịch vụ TCP echo

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_echo_sv.c
#include <signal.h>
#include <syslog.h>
#include <sys/wait.h>
#include "become_daemon.h"
#include "inet_sockets.h" /* Declarations of inet*() socket functions */
#include "tlpi_hdr.h"
#define SERVICE "echo" /* Name of TCP service */
#define BUF_SIZE 4096
static void /* SIGCHLD handler to reap dead child processes */
grimReaper(int sig)
{
 int savedErrno; /* Save 'errno' in case changed here */
 savedErrno = errno;
 while (waitpid(-1, NULL, WNOHANG) > 0)
 continue;
 errno = savedErrno;
}
/* Handle a client request: copy socket input back to socket */
static void
handleRequest(int cfd)
{
 char buf[BUF_SIZE];
 ssize_t numRead;
 while ((numRead = read(cfd, buf, BUF_SIZE)) > 0) {
 if (write(cfd, buf, numRead) != numRead) {
 syslog(LOG_ERR, "write() failed: %s", strerror(errno));
 exit(EXIT_FAILURE);
 }
 }
 if (numRead == -1) {
 syslog(LOG_ERR, "Error from read(): %s", strerror(errno));
 exit(EXIT_FAILURE);
 }
}
int
main(int argc, char *argv[])
{
 int lfd, cfd; /* Listening and connected sockets */
 struct sigaction sa;
 if (becomeDaemon(0) == -1)
 errExit("becomeDaemon");
```

```
 sigemptyset(&sa.sa_mask);
 sa.sa_flags = SA_RESTART;
 sa.sa_handler = grimReaper;
 if (sigaction(SIGCHLD, &sa, NULL) == -1) {
 syslog(LOG_ERR, "Error from sigaction(): %s", strerror(errno));
 exit(EXIT_FAILURE);
 }
 lfd = inetListen(SERVICE, 10, NULL);
 if (lfd == -1) {
 syslog(LOG_ERR, "Could not create server socket (%s)", strerror(errno));
 exit(EXIT_FAILURE);
 }
 for (;;) {
 cfd = accept(lfd, NULL, NULL); /* Wait for connection */
 if (cfd == -1) {
 syslog(LOG_ERR, "Failure in accept(): %s", strerror(errno));
 exit(EXIT_FAILURE);
 }
      /* Handle each client request in a new child process */
      switch (fork()) {
 case -1:
 syslog(LOG_ERR, "Can't create child (%s)", strerror(errno));
 close(cfd); /* Give up on this client */
 break; /* May be temporary; try next client */
 case 0: /* Child */
 close(lfd); /* Unneeded copy of listening socket */
 handleRequest(cfd);
 _exit(EXIT_SUCCESS);
 default: /* Parent */
 close(cfd); /* Unneeded copy of connected socket */
 break; /* Loop to accept next connection */
 }
 }
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_echo_sv.c
```

# **60.4 Các Mô Hình Concurrent Server Khác**

<span id="page-96-0"></span>Mô hình concurrent server truyền thống được mô tả trong mục trước là đủ cho nhiều ứng dụng cần xử lý đồng thời nhiều client qua TCP connection. Tuy nhiên, đối với các server có tải trọng rất cao (ví dụ, web server xử lý hàng nghìn request mỗi phút), chi phí tạo một child mới (hoặc thậm chí một thread) cho mỗi client đặt ra một gánh nặng đáng kể cho server (tham khảo Mục 28.3), và cần phải sử dụng các thiết kế thay thế. Chúng ta xem xét sơ lược một số lựa chọn thay thế này.

#### **Server preforked và prethreaded**

Server preforked và prethreaded được mô tả chi tiết trong Chương 30 của [Stevens et al., 2004]. Các ý tưởng chính như sau:

-  Thay vì tạo một child process mới (hoặc thread) cho mỗi client, server tạo sẵn một số lượng cố định child process (hoặc thread) ngay khi khởi động (tức là, trước khi nhận được bất kỳ yêu cầu nào của client). Các child này tạo thành một cái gọi là server pool.
-  Mỗi child trong server pool xử lý một client tại một thời điểm, nhưng thay vì kết thúc sau khi xử lý client, child lấy client tiếp theo cần phục vụ và phục vụ nó, và cứ thế tiếp tục.

Sử dụng kỹ thuật trên đòi hỏi một số quản lý cẩn thận trong ứng dụng server. Server pool nên đủ lớn để đảm bảo đáp ứng đầy đủ các yêu cầu của client. Điều này có nghĩa là process cha của server phải theo dõi số lượng child rảnh rỗi, và trong những thời điểm tải cao nhất, tăng kích thước pool để luôn có đủ child process sẵn sàng phục vụ ngay cho các client mới. Nếu tải giảm, thì kích thước server pool nên được thu nhỏ lại, vì có quá nhiều process trên hệ thống có thể làm giảm hiệu suất tổng thể của hệ thống.

Ngoài ra, các child trong server pool phải tuân theo một số giao thức để cho phép chúng chọn riêng từng kết nối client. Trên hầu hết các triển khai UNIX (bao gồm Linux), mỗi child trong pool chặn trong một lần gọi `accept()` trên listening descriptor là đủ. Nói cách khác, process cha của server tạo ra listening socket trước khi tạo bất kỳ child nào, và mỗi child kế thừa một file descriptor cho socket trong quá trình `fork()`. Khi có kết nối client mới đến, chỉ một trong các child sẽ hoàn thành lần gọi `accept()`. Tuy nhiên, vì `accept()` không phải là một system call nguyên tử trên một số triển khai cũ hơn, lần gọi có thể cần được bao quanh bởi một kỹ thuật loại trừ lẫn nhau (ví dụ: file lock) để đảm bảo rằng chỉ một child tại một thời điểm thực hiện lần gọi ([Stevens et al., 2004]).

> Có các lựa chọn thay thế cho việc để tất cả child trong server pool thực hiện các lần gọi `accept()`. Nếu server pool bao gồm các process riêng biệt, process cha của server có thể thực hiện lần gọi `accept()`, và sau đó chuyển file descriptor chứa kết nối mới cho một trong các process rảnh trong pool, sử dụng một kỹ thuật mà chúng ta mô tả ngắn gọn trong [Mục 61.13.3.](#page-135-1) Nếu server pool bao gồm các thread, thread chính có thể thực hiện lần gọi `accept()`, và sau đó thông báo cho một trong các server thread rảnh rằng có client mới trên connected descriptor.

### **Xử lý nhiều client từ một process duy nhất**

Trong một số trường hợp, chúng ta có thể thiết kế một server process duy nhất để xử lý nhiều client. Để làm điều này, chúng ta phải sử dụng một trong các mô hình I/O (I/O multiplexing, signal-driven I/O, hoặc epoll) cho phép một process duy nhất đồng thời giám sát nhiều file descriptor cho các sự kiện I/O. Các mô hình này được mô tả trong Chương 63.

Trong thiết kế single-server, server process phải đảm nhận một số tác vụ lập lịch thường được kernel xử lý. Trong giải pháp có một server process cho mỗi client, chúng ta có thể dựa vào kernel để đảm bảo rằng mỗi server process (và do đó mỗi client) được chia sẻ tài nguyên một cách công bằng trên host của server. Nhưng khi chúng ta sử dụng một server process duy nhất để xử lý nhiều client, server phải thực hiện một số công việc để đảm bảo rằng một hoặc một vài client không độc chiếm quyền truy cập vào server trong khi các client khác bị bỏ đói. Chúng ta nói thêm một chút về điểm này trong Mục 63.4.6.

## **Sử dụng server farm**

Các cách tiếp cận khác để xử lý tải client cao liên quan đến việc sử dụng nhiều hệ thống server — một server farm.

Một trong những cách tiếp cận đơn giản nhất để xây dựng server farm (được một số web server sử dụng) là DNS round-robin load sharing (hoặc load distribution), trong đó authoritative name server cho một zone ánh xạ cùng một tên miền đến một số địa chỉ IP (tức là, một số server chia sẻ cùng một tên miền). Các yêu cầu liên tiếp đến DNS server để phân giải tên miền trả về các địa chỉ IP này theo thứ tự khác nhau, theo cách round-robin. Thông tin thêm về DNS round-robin load sharing có thể tìm thấy trong [Albitz & Liu, 2006].

Round-robin DNS có ưu điểm là rẻ và dễ thiết lập. Tuy nhiên, nó đưa ra một số vấn đề. Một trong số đó là bộ nhớ đệm được thực hiện bởi các remote DNS server, có nghĩa là các yêu cầu trong tương lai từ client trên một host cụ thể (hoặc tập hợp host) bỏ qua round-robin DNS server và luôn được xử lý bởi cùng một server. Ngoài ra, round-robin DNS không có bất kỳ cơ chế tích hợp nào để đảm bảo cân bằng tải tốt (các client khác nhau có thể đặt tải khác nhau lên server) hoặc đảm bảo tính khả dụng cao (nếu một trong các server bị hỏng hoặc ứng dụng server đang chạy trên đó bị crash?). Một vấn đề khác mà chúng ta có thể cần xem xét — vấn đề mà nhiều thiết kế sử dụng nhiều máy server phải đối mặt — là đảm bảo server affinity; tức là, đảm bảo rằng một chuỗi yêu cầu từ cùng một client đều được chuyển đến cùng một server, để bất kỳ thông tin trạng thái nào server duy trì về client vẫn còn chính xác.

Một giải pháp linh hoạt hơn, nhưng cũng phức tạp hơn, là server load balancing. Trong kịch bản này, một load-balancing server duy nhất định tuyến các yêu cầu client đến đến một trong các thành viên của server farm. (Để đảm bảo tính khả dụng cao, có thể có một backup server tiếp quản nếu primary load-balancing server bị crash.) Điều này loại bỏ các vấn đề liên quan đến remote DNS caching, vì server farm trình bày một địa chỉ IP duy nhất (của load-balancing server) cho thế giới bên ngoài. Load-balancing server tích hợp các thuật toán để đo lường hoặc ước tính tải server (có thể dựa trên các số liệu do các thành viên của server farm cung cấp) và phân phối tải thông minh trên các thành viên của server farm. Load-balancing server cũng tự động phát hiện các lỗi trong các thành viên của server farm (và việc thêm server mới, nếu nhu cầu yêu cầu). Cuối cùng, load-balancing server cũng có thể hỗ trợ server affinity. Thông tin thêm về server load balancing có thể tìm thấy trong [Kopparapu, 2002].

# **60.5 Daemon inetd (Internet Superserver)**

<span id="page-98-0"></span>Nếu chúng ta xem qua nội dung của `/etc/services`, chúng ta thấy hàng trăm dịch vụ khác nhau được liệt kê. Điều này ngụ ý rằng một hệ thống về mặt lý thuyết có thể đang chạy một số lượng lớn server process. Tuy nhiên, hầu hết các server này thường chỉ làm gì đó ngoài việc chờ đợi các yêu cầu kết nối hoặc các datagram không thường xuyên. Tất cả các server process này vẫn chiếm các vị trí trong bảng process của kernel, và tiêu thụ một lượng memory và swap space, do đó đặt tải lên hệ thống.

Daemon `inetd` được thiết kế để loại bỏ nhu cầu chạy một số lượng lớn server được sử dụng không thường xuyên. Sử dụng `inetd` mang lại hai lợi ích chính:

-  Thay vì chạy một daemon riêng biệt cho mỗi dịch vụ, một process duy nhất — daemon `inetd` — giám sát một tập hợp các socket port được chỉ định và khởi động các server khác khi cần. Do đó, số lượng process đang chạy trên hệ thống được giảm thiểu.
-  Việc lập trình các server được khởi động bởi `inetd` được đơn giản hóa, vì `inetd` thực hiện một số bước thường được yêu cầu bởi tất cả các network server khi khởi động.

Vì nó giám sát một loạt các dịch vụ, gọi các server khác khi cần, `inetd` đôi khi được gọi là Internet superserver.

> Một phiên bản mở rộng của `inetd`, `xinetd`, được cung cấp trong một số bản phân phối Linux. Trong số những thứ khác, `xinetd` thêm một số cải tiến bảo mật. Thông tin về `xinetd` có thể tìm thấy tại http://www.xinetd.org/.

## **Hoạt động của daemon inetd**

Daemon `inetd` thường được khởi động trong quá trình boot hệ thống. Sau khi trở thành một daemon process (Mục 37.2), `inetd` thực hiện các bước sau:

- 1. Đối với mỗi dịch vụ được chỉ định trong file cấu hình của nó, `/etc/inetd.conf`, `inetd` tạo ra một socket thuộc loại phù hợp (tức là, stream hoặc datagram) và bind nó vào port được chỉ định. Mỗi TCP socket được đánh dấu thêm để cho phép các kết nối đến qua lần gọi `listen()`.
- 2. Sử dụng system call `select()` (Mục 63.2.1), `inetd` giám sát tất cả các socket được tạo trong bước trước để phát hiện datagram hoặc yêu cầu kết nối đến.
- 3. Lần gọi `select()` chặn cho đến khi một UDP socket có datagram sẵn để đọc hoặc một yêu cầu kết nối được nhận trên một TCP socket. Trong trường hợp TCP connection, `inetd` thực hiện `accept()` cho kết nối trước khi tiến hành bước tiếp theo.
- 4. Để khởi động server được chỉ định cho socket này, `inetd()` gọi `fork()` để tạo ra một process mới sau đó thực hiện `exec()` để khởi động chương trình server. Trước khi thực hiện `exec()`, child process thực hiện các bước sau:
  - a) Đóng tất cả các file descriptor kế thừa từ process cha, ngoại trừ cái dành cho socket nơi UDP datagram có sẵn hoặc TCP connection đã được chấp nhận.
  - b) Sử dụng các kỹ thuật được mô tả trong Mục 5.5 để nhân bản socket file descriptor trên các file descriptor 0, 1 và 2, và đóng chính socket file descriptor (vì nó không còn cần thiết nữa). Sau bước này, server đã được exec có thể giao tiếp trên socket bằng cách sử dụng ba standard file descriptor.
  - c) Tùy chọn, đặt user và group ID cho server đã được exec thành các giá trị được chỉ định trong `/etc/inetd.conf`.
- 5. Nếu một kết nối đã được chấp nhận trên một TCP socket trong bước 3, `inetd` đóng connected socket (vì nó chỉ cần trong server đã được exec).
- 6. `inetd` server quay lại bước 2.

## **File /etc/inetd.conf**

Hoạt động của daemon `inetd` được kiểm soát bởi một file cấu hình, thường là `/etc/inetd.conf`. Mỗi dòng trong file này mô tả một trong các dịch vụ được xử lý bởi `inetd`. [Listing 60-5](#page-100-0) hiển thị một số ví dụ về các mục trong file `/etc/inetd.conf` đi kèm với một bản phân phối Linux.

<span id="page-100-1"></span><span id="page-100-0"></span>**Listing 60-5:** Các dòng ví dụ từ /etc/inetd.conf

```
# echo stream tcp nowait root internal
# echo dgram udp wait root internal
ftp stream tcp nowait root /usr/sbin/tcpd in.ftpd
telnet stream tcp nowait root /usr/sbin/tcpd in.telnetd
login stream tcp nowait root /usr/sbin/tcpd in.rlogind
```

Hai dòng đầu tiên của [Listing 60-5](#page-100-0) được comment ra bằng ký tự `#` đầu dòng; chúng ta hiển thị chúng ở đây vì chúng ta sẽ đề cập đến dịch vụ echo sắp tới.

Mỗi dòng của `/etc/inetd.conf` bao gồm các trường sau, được phân tách bằng khoảng trắng:

-  Tên dịch vụ (Service name): Chỉ định tên của một dịch vụ từ file `/etc/services`. Kết hợp với trường protocol, điều này được sử dụng để tra cứu `/etc/services` để xác định số port nào `inetd` nên giám sát cho dịch vụ này.
-  Loại socket (Socket type): Chỉ định loại socket được sử dụng bởi dịch vụ này — ví dụ: `stream` hoặc `dgram`.
-  Protocol: Chỉ định protocol được sử dụng bởi socket này. Trường này có thể chứa bất kỳ Internet protocol nào được liệt kê trong file `/etc/protocols` (được tài liệu hóa trong trang hướng dẫn `protocols(5)`), nhưng hầu hết mọi dịch vụ đều chỉ định `tcp` (cho TCP) hoặc `udp` (cho UDP).
-  Flags: Trường này chứa `wait` hoặc `nowait`. Trường này chỉ định liệu server được exec bởi `inetd` có (tạm thời) tiếp quản quản lý socket cho dịch vụ này hay không. Nếu server được exec quản lý socket, thì trường này được chỉ định là `wait`. Điều này khiến `inetd` xóa socket này khỏi tập file descriptor mà nó giám sát bằng `select()` cho đến khi server được exec thoát (inetd phát hiện điều này qua một handler cho SIGCHLD). Chúng ta nói thêm một chút về trường này bên dưới.
-  Tên đăng nhập (Login name): Trường này bao gồm một tên người dùng từ `/etc/passwd`, tùy chọn theo sau là dấu chấm (`.`) và tên group từ `/etc/group`. Những thứ này xác định user và group ID mà server được exec chạy dưới. (Vì `inetd` chạy với effective user ID là root, các con của nó cũng có đặc quyền và do đó có thể sử dụng các lần gọi `setuid()` và `setgid()` để thay đổi thông tin xác thực của process nếu muốn.)
-  Chương trình server (Server program): Chỉ định pathname của chương trình server cần exec.
-  Đối số chương trình server (Server program arguments): Trường này chỉ định một hoặc nhiều đối số, được phân tách bằng khoảng trắng, được sử dụng làm danh sách đối số khi exec chương trình server. Đối số đầu tiên trong số này tương ứng với `argv[0]` trong chương trình được exec và do đó thường giống với phần basename của tên chương trình server. Đối số tiếp theo tương ứng với `argv[1]`, v.v.

Trong các dòng ví dụ được hiển thị trong [Listing 60-5](#page-100-0) cho các dịch vụ ftp, telnet và login, chúng ta thấy chương trình server và các đối số được thiết lập khác với những gì vừa mô tả. Cả ba dịch vụ này đều khiến `inetd` gọi cùng một chương trình, `tcpd(8)` (TCP daemon wrapper), thực hiện một số ghi nhật ký và kiểm tra kiểm soát truy cập trước khi exec chương trình thích hợp, dựa trên giá trị được chỉ định làm đối số chương trình server đầu tiên (có sẵn cho `tcpd` qua `argv[0]`). Thông tin thêm về `tcpd` có thể tìm thấy trong trang hướng dẫn `tcpd(8)` và trong [Mann & Mitchell, 2003].

Các stream socket (TCP) server được gọi bởi `inetd` thường được thiết kế để xử lý chỉ một kết nối client và sau đó kết thúc, để lại cho `inetd` công việc lắng nghe các kết nối tiếp theo. Đối với các server như vậy, flags nên được chỉ định là `nowait`. (Nếu thay vào đó, server được exec chấp nhận kết nối, thì nên chỉ định `wait`, trong trường hợp này `inetd` không chấp nhận kết nối, mà thay vào đó truyền file descriptor cho listening socket cho server được exec dưới dạng descriptor 0.)

Đối với hầu hết các UDP server, trường flags nên được chỉ định là `wait`. Một UDP server được gọi bởi `inetd` thường được thiết kế để đọc và xử lý tất cả các datagram đang chờ trên socket và sau đó kết thúc. (Điều này thường đòi hỏi một số loại timeout khi đọc socket, để server kết thúc khi không có datagram mới đến trong một khoảng thời gian xác định.) Bằng cách chỉ định `wait`, chúng ta ngăn chặn daemon `inetd` đồng thời cố gắng `select()` trên socket, điều này sẽ có hậu quả không mong muốn là `inetd` sẽ tranh nhau với UDP server để kiểm tra datagram và, nếu nó thắng cuộc đua, sẽ khởi động một phiên bản khác của UDP server.

> Vì hoạt động của `inetd` và định dạng file cấu hình của nó không được SUSv3 chỉ định, có một số biến thể (thường là nhỏ) trong các giá trị có thể được chỉ định trong các trường của `/etc/inetd.conf`. Hầu hết các phiên bản của `inetd` cung cấp ít nhất cú pháp mà chúng ta mô tả trong văn bản chính. Để biết thêm chi tiết, hãy xem trang hướng dẫn `inetd.conf(8)`.

Như một biện pháp hiệu quả, `inetd` tự triển khai một vài dịch vụ đơn giản, thay vì exec các server riêng biệt để thực hiện tác vụ. Dịch vụ echo UDP và TCP là ví dụ về các dịch vụ mà `inetd` triển khai. Đối với các dịch vụ như vậy, trường server program của bản ghi `/etc/inetd.conf` tương ứng được chỉ định là `internal`, và các đối số chương trình server bị bỏ qua. (Trong các dòng ví dụ trong [Listing 60-5](#page-100-0), chúng ta thấy rằng các mục dịch vụ echo đã bị comment ra. Để kích hoạt dịch vụ echo, chúng ta cần xóa ký tự `#` ở đầu các dòng này.)

Bất cứ khi nào chúng ta thay đổi file `/etc/inetd.conf`, chúng ta cần gửi signal SIGHUP cho `inetd` để yêu cầu nó đọc lại file:

# **killall -HUP inetd**

#### **Ví dụ: gọi dịch vụ TCP echo qua inetd**

Chúng ta đã lưu ý trước đó rằng `inetd` đơn giản hóa việc lập trình các server, đặc biệt là concurrent (thường là TCP) server. Nó thực hiện điều này bằng cách thực hiện các bước sau thay mặt cho các server mà nó gọi:

- 1. Thực hiện tất cả các khởi tạo liên quan đến socket, gọi `socket()`, `bind()`, và (đối với TCP server) `listen()`.
- 2. Đối với dịch vụ TCP, thực hiện `accept()` cho kết nối mới.

- 3. Tạo một process mới để xử lý UDP datagram hoặc TCP connection đến. Process được tự động thiết lập như một daemon. Chương trình `inetd` xử lý tất cả các chi tiết tạo process qua `fork()` và thu hồi các child đã chết qua một handler cho SIGCHLD.
- 4. Nhân bản file descriptor của UDP socket hoặc connected TCP socket trên các file descriptor 0, 1 và 2, và đóng tất cả các file descriptor khác (vì chúng không được sử dụng trong server đã được exec).
- 5. Exec chương trình server.

(Trong mô tả các bước trên, chúng ta giả định các trường hợp thông thường rằng trường flags của mục dịch vụ trong `/etc/inetd.conf` được chỉ định là `nowait` cho TCP service và `wait` cho UDP service.)

Như một ví dụ về cách `inetd` đơn giản hóa việc lập trình TCP service, trong [List](#page-102-0)[ing 60-6,](#page-102-0) chúng ta trình bày tương đương TCP echo server được gọi bởi `inetd` của TCP echo server từ [Listing 60-4.](#page-95-0) Vì `inetd` thực hiện tất cả các bước trên, tất cả những gì còn lại của server là code được thực thi bởi child process để xử lý yêu cầu client, có thể được đọc từ file descriptor 0 (`STDIN_FILENO`).

Nếu server nằm trong thư mục `/bin` (ví dụ), thì chúng ta cần tạo mục sau trong `/etc/inetd.conf` để `inetd` gọi server:

echo stream tcp nowait root /bin/is\_echo\_inetd\_sv is\_echo\_inetd\_sv

<span id="page-102-0"></span>**Listing 60-6:** TCP echo server được thiết kế để được gọi qua inetd

```
–––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_echo_inetd_sv.c
#include <syslog.h>
#include "tlpi_hdr.h"
#define BUF_SIZE 4096
int
main(int argc, char *argv[])
{
 char buf[BUF_SIZE];
 ssize_t numRead;
 while ((numRead = read(STDIN_FILENO, buf, BUF_SIZE)) > 0) {
 if (write(STDOUT_FILENO, buf, numRead) != numRead) {
 syslog(LOG_ERR, "write() failed: %s", strerror(errno));
 exit(EXIT_FAILURE);
 }
 }
 if (numRead == -1) {
 syslog(LOG_ERR, "Error from read(): %s", strerror(errno));
 exit(EXIT_FAILURE);
 }
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_echo_inetd_sv.c
```

# **60.6 Tóm tắt**

Iterative server xử lý từng client một, hoàn thành toàn bộ yêu cầu của client đó trước khi chuyển sang client tiếp theo. Concurrent server xử lý nhiều client đồng thời. Trong các tình huống tải cao, thiết kế concurrent server truyền thống tạo ra một child process mới (hoặc thread) cho mỗi client có thể không hoạt động đủ hiệu quả, và chúng ta đã phác thảo một loạt các cách tiếp cận khác để xử lý đồng thời số lượng lớn client.

Daemon Internet superserver, `inetd`, giám sát nhiều socket và khởi động các server thích hợp để phản hồi các UDP datagram hoặc TCP connection đến. Sử dụng `inetd` cho phép chúng ta giảm tải hệ thống bằng cách giảm thiểu số lượng network server process trên hệ thống, và cũng đơn giản hóa việc lập trình các server process, vì nó thực hiện hầu hết các bước khởi tạo được yêu cầu bởi một server.

### **Thông tin thêm**

Tham khảo các nguồn thông tin thêm được liệt kê trong Mục [59.15.](#page-86-0)

# **60.7 Bài tập**

- **60-1.** Thêm code vào chương trình trong [Listing 60-4](#page-95-0) (is\_echo\_sv.c) để đặt giới hạn cho số lượng child đang thực thi đồng thời.
- **60-2.** Đôi khi, có thể cần phải viết một socket server để nó có thể được gọi trực tiếp từ dòng lệnh hoặc gián tiếp qua `inetd`. Trong trường hợp này, một tùy chọn dòng lệnh được sử dụng để phân biệt hai trường hợp. Sửa đổi chương trình trong [Listing 60-4](#page-95-0) để, nếu nó được cho tùy chọn dòng lệnh `–i`, nó giả định rằng nó đang được gọi bởi `inetd` và xử lý một client duy nhất trên connected socket, mà `inetd` cung cấp qua `STDIN_FILENO`. Nếu tùy chọn `–i` không được cung cấp, thì chương trình có thể giả định rằng nó đang được gọi từ dòng lệnh, và hoạt động theo cách thông thường. (Thay đổi này chỉ yêu cầu thêm một vài dòng code.) Sửa đổi `/etc/inetd.conf` để gọi chương trình này cho dịch vụ echo.
