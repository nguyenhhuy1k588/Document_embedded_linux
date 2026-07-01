## Chương 56
# **SOCKET: GIỚI THIỆU**

Socket là một phương thức IPC cho phép trao đổi dữ liệu giữa các ứng dụng, trên cùng một host (máy tính) hoặc trên các host khác nhau được kết nối qua mạng. Triển khai đầu tiên được phổ biến rộng rãi của sockets API xuất hiện cùng với 4.2BSD vào năm 1983, và API này đã được chuyển sang hầu hết mọi triển khai UNIX, cũng như hầu hết các hệ điều hành khác.

> Sockets API được quy định chính thức trong POSIX.1g, được phê chuẩn năm 2000 sau khoảng một thập kỷ là bản nháp tiêu chuẩn. Tiêu chuẩn này đã được thay thế bởi SUSv3.

Chương này và các chương tiếp theo mô tả cách sử dụng socket, cụ thể như sau:

- Chương này cung cấp phần giới thiệu tổng quát về sockets API. Các chương tiếp theo giả định rằng người đọc đã hiểu các khái niệm chung được trình bày ở đây. Chúng ta không trình bày bất kỳ ví dụ code nào trong chương này. Các ví dụ code trong miền UNIX và Internet được trình bày ở các chương tiếp theo.
- Chương 57 mô tả UNIX domain socket, cho phép giao tiếp giữa các ứng dụng trên cùng một host.
- Chương 58 giới thiệu các khái niệm mạng máy tính khác nhau và mô tả các tính năng chính của giao thức mạng TCP/IP. Nó cung cấp nền tảng cần thiết cho các chương tiếp theo.
- Chương 59 mô tả Internet domain socket, cho phép các ứng dụng trên các host khác nhau giao tiếp qua mạng TCP/IP.
- Chương 60 thảo luận về thiết kế server sử dụng socket.
- Chương 61 bao gồm nhiều chủ đề nâng cao, bao gồm các tính năng bổ sung cho socket I/O, tìm hiểu chi tiết hơn về giao thức TCP, và cách sử dụng socket option để lấy và sửa đổi các thuộc tính khác nhau của socket.

Các chương này chỉ nhằm mục đích cung cấp cho người đọc nền tảng vững chắc trong việc sử dụng socket. Lập trình socket, đặc biệt cho giao tiếp mạng, là một chủ đề rất rộng lớn, và là chủ đề của nhiều cuốn sách riêng. Các nguồn thông tin thêm được liệt kê trong Mục 59.15.

# **56.1 Tổng quan**

Trong một kịch bản client-server điển hình, các ứng dụng giao tiếp bằng socket như sau:

- Mỗi ứng dụng tạo một socket. Socket là "thiết bị" cho phép giao tiếp, và cả hai ứng dụng đều cần một socket.
- Server bind socket của nó vào một địa chỉ được biết trước (well-known address) để các client có thể tìm thấy nó.

Một socket được tạo bằng system call `socket()`, trả về một file descriptor dùng để tham chiếu đến socket trong các system call tiếp theo:

```
fd = socket(domain, type, protocol);
```

Chúng ta mô tả các domain và type của socket trong các đoạn tiếp theo. Với tất cả các ứng dụng được mô tả trong cuốn sách này, `protocol` luôn được chỉ định là 0.

#### **Communication domain**

Socket tồn tại trong một communication domain, xác định:

- phương thức nhận dạng một socket (tức là định dạng của "địa chỉ" socket); và
- phạm vi giao tiếp (tức là giữa các ứng dụng trên cùng host hay giữa các ứng dụng trên các host khác nhau được kết nối qua mạng).

Các hệ điều hành hiện đại hỗ trợ ít nhất các domain sau:

- Domain UNIX (`AF_UNIX`) cho phép giao tiếp giữa các ứng dụng trên cùng một host. (POSIX.1g sử dụng tên `AF_LOCAL` như là từ đồng nghĩa với `AF_UNIX`, nhưng tên này không được dùng trong SUSv3.)
- Domain IPv4 (`AF_INET`) cho phép giao tiếp giữa các ứng dụng chạy trên các host được kết nối qua mạng Internet Protocol phiên bản 4 (IPv4).
- Domain IPv6 (`AF_INET6`) cho phép giao tiếp giữa các ứng dụng chạy trên các host được kết nối qua mạng Internet Protocol phiên bản 6 (IPv6). Mặc dù IPv6 được thiết kế là phiên bản kế tiếp của IPv4, nhưng giao thức sau vẫn đang được sử dụng phổ biến nhất hiện nay.

Bảng 56-1 tóm tắt các đặc điểm của các socket domain này.

Trong một số code, chúng ta có thể thấy các hằng số có tên như `PF_UNIX` thay vì `AF_UNIX`. Trong ngữ cảnh này, AF là viết tắt của "address family" (họ địa chỉ) và PF là viết tắt của "protocol family" (họ giao thức). Ban đầu, người ta hình dung rằng một họ giao thức có thể hỗ trợ nhiều họ địa chỉ. Trên thực tế, chưa bao giờ có họ giao thức nào hỗ trợ nhiều họ địa chỉ được định nghĩa, và tất cả các triển khai hiện có đều định nghĩa các hằng số `PF_` đồng nghĩa với các hằng số `AF_` tương ứng. (SUSv3 chỉ định các hằng số `AF_`, không phải `PF_`.) Trong cuốn sách này, chúng ta luôn sử dụng hằng số `AF_`. Thông tin thêm về lịch sử của các hằng số này có thể tìm thấy trong Mục 4.2 của [Stevens et al., 2004].

**Bảng 56-1:** Các socket domain

| Domain   | Giao tiếp<br>thực hiện | Giao tiếp<br>giữa các ứng dụng     | Định dạng địa chỉ                               | Cấu trúc<br>địa chỉ |
|----------|----------------------------|-------------------------------------------|----------------------------------------------|----------------------|
| AF_UNIX  | trong kernel              | trên cùng host                              | pathname                                     | sockaddr_un          |
| AF_INET  | qua IPv4                   | trên host được kết nối                        | Địa chỉ IPv4 32-bit +                        | sockaddr_in          |
|          |                            | qua mạng IPv4                       | Số port 16-bit                           |                      |
| AF_INET6 | qua IPv6                   | trên host được kết nối<br>qua mạng IPv6 | Địa chỉ IPv6 128-bit +<br>Số port 16-bit | sockaddr_in6         |

## **Loại socket**

Mọi triển khai socket đều cung cấp ít nhất hai loại socket: stream và datagram. Các loại socket này được hỗ trợ trong cả domain UNIX và Internet. Bảng 56-2 tóm tắt các thuộc tính của các loại socket này.

**Bảng 56-2:** Các loại socket và thuộc tính của chúng

| Thuộc tính                      | Loại socket |          |  |  |
|-------------------------------|-------------|----------|--|--|
|                               | Stream      | Datagram |  |  |
| Truyền tin cậy?            | Y           | N        |  |  |
| Bảo toàn ranh giới message? | N           | Y        |  |  |
| Hướng kết nối?          | Y           | N        |  |  |

Stream socket (`SOCK_STREAM`) cung cấp một kênh giao tiếp byte-stream đáng tin cậy, hai chiều giữa hai đầu cuối. Theo các thuật ngữ trong mô tả này, chúng ta có nghĩa là:

- Đáng tin cậy (Reliable) có nghĩa là chúng ta được đảm bảo rằng dữ liệu được truyền sẽ đến nguyên vẹn tại ứng dụng nhận, đúng như người gửi đã truyền (giả sử cả kết nối mạng lẫn bên nhận đều không bị lỗi), hoặc chúng ta sẽ nhận được thông báo về khả năng truyền thất bại.
- Hai chiều (Bidirectional) có nghĩa là dữ liệu có thể được truyền theo cả hai hướng giữa hai socket.
- Byte-stream có nghĩa là, như với pipe, không có khái niệm về ranh giới message (tham khảo Mục 44.1).

Một stream socket tương tự như việc sử dụng một cặp pipe để cho phép giao tiếp hai chiều giữa hai ứng dụng, với sự khác biệt là socket (trong Internet domain) cho phép giao tiếp qua mạng.

Stream socket hoạt động theo cặp kết nối. Vì lý do này, stream socket được mô tả là connection-oriented (hướng kết nối). Thuật ngữ peer socket đề cập đến socket ở đầu kia của kết nối; peer address biểu thị địa chỉ của socket đó; và peer application biểu thị ứng dụng sử dụng peer socket. Đôi khi, thuật ngữ remote (hoặc foreign) được dùng đồng nghĩa với peer. Tương tự, đôi khi thuật ngữ local được dùng để tham chiếu đến ứng dụng, socket, hoặc địa chỉ ở đầu này của kết nối. Một stream socket chỉ có thể được kết nối với một peer.

Datagram socket (`SOCK_DGRAM`) cho phép trao đổi dữ liệu dưới dạng các message gọi là datagram. Với datagram socket, ranh giới message được bảo toàn, nhưng việc truyền dữ liệu không đáng tin cậy. Các message có thể đến không theo thứ tự, bị trùng lặp, hoặc không đến được.

Datagram socket là một ví dụ về khái niệm tổng quát hơn là connectionless socket (socket không kết nối). Không giống như stream socket, datagram socket không cần phải kết nối với socket khác để sử dụng. (Trong Mục 56.6.2, chúng ta sẽ thấy rằng các datagram socket có thể được kết nối với nhau, nhưng điều này có ngữ nghĩa hơi khác so với các stream socket được kết nối.)

Trong Internet domain, datagram socket sử dụng User Datagram Protocol (UDP), và stream socket (thường) sử dụng Transmission Control Protocol (TCP). Thay vì sử dụng các thuật ngữ Internet domain datagram socket và Internet domain stream socket, chúng ta thường chỉ sử dụng các thuật ngữ UDP socket và TCP socket tương ứng.

## **Các system call của socket**

Các system call socket chính là:

- System call `socket()` tạo một socket mới.
- System call `bind()` bind một socket vào một địa chỉ. Thông thường, server sử dụng lệnh gọi này để bind socket của nó vào một well-known address để các client có thể tìm thấy socket.
- System call `listen()` cho phép stream socket chấp nhận các kết nối đến từ các socket khác.
- System call `accept()` chấp nhận một kết nối từ ứng dụng peer trên một listening stream socket, và tùy chọn trả về địa chỉ của peer socket.
- System call `connect()` thiết lập kết nối với socket khác.

Trên hầu hết các kiến trúc Linux (ngoại trừ Alpha và IA-64), tất cả các system call socket thực tế được triển khai như các hàm thư viện được ghép qua một system call đơn, `socketcall()`. (Đây là sản phẩm của việc phát triển ban đầu việc triển khai socket Linux như một dự án riêng biệt.) Tuy nhiên, chúng ta gọi tất cả các hàm này là system call trong cuốn sách này, vì đây là những gì chúng vốn là trong triển khai BSD gốc, cũng như trong nhiều triển khai UNIX đương đại khác.

Socket I/O có thể được thực hiện bằng cách sử dụng các system call `read()` và `write()` thông thường, hoặc bằng cách sử dụng một loạt các system call dành riêng cho socket (ví dụ: `send()`, `recv()`, `sendto()`, và `recvfrom()`). Theo mặc định, các system call này sẽ block nếu thao tác I/O không thể hoàn thành ngay lập tức. Nonblocking I/O cũng có thể thực hiện, bằng cách sử dụng thao tác `fcntl()` `F_SETFL` (Mục 5.3) để bật flag `O_NONBLOCK` trạng thái file đang mở.

> Trên Linux, chúng ta có thể gọi `ioctl(fd, FIONREAD, &cnt)` để lấy số byte chưa đọc có sẵn trên stream socket được tham chiếu bởi file descriptor `fd`. Đối với datagram socket, thao tác này trả về số byte trong datagram chưa đọc tiếp theo (có thể là 0 nếu datagram tiếp theo có độ dài bằng 0), hoặc 0 nếu không có datagram đang chờ. Tính năng này không được quy định trong SUSv3.

# **56.2 Tạo Socket: socket()**

System call `socket()` tạo một socket mới.

```
#include <sys/socket.h>
int socket(int domain, int type, int protocol);
                                Returns file descriptor on success, or –1 on error
```

Đối số `domain` chỉ định communication domain cho socket. Đối số `type` chỉ định loại socket. Đối số này thường được chỉ định là `SOCK_STREAM` để tạo stream socket, hoặc `SOCK_DGRAM` để tạo datagram socket.

Đối số `protocol` luôn được chỉ định là 0 cho các loại socket được mô tả trong cuốn sách này. Các giá trị protocol khác 0 được sử dụng với một số loại socket mà chúng ta không mô tả. Ví dụ, `protocol` được chỉ định là `IPPROTO_RAW` cho raw socket (`SOCK_RAW`).

Khi thành công, `socket()` trả về một file descriptor được sử dụng để tham chiếu đến socket mới được tạo trong các system call sau.

> Bắt đầu từ kernel 2.6.27, Linux cung cấp cách sử dụng thứ hai cho đối số `type`, bằng cách cho phép hai flag không chuẩn được OR với loại socket. Flag `SOCK_CLOEXEC` khiến kernel bật close-on-exec flag (`FD_CLOEXEC`) cho file descriptor mới. Flag này hữu ích vì những lý do tương tự như flag `O_CLOEXEC` của `open()` được mô tả trong Mục 4.3.1. Flag `SOCK_NONBLOCK` khiến kernel đặt flag `O_NONBLOCK` trên open file description bên dưới, để các thao tác I/O trong tương lai trên socket sẽ là nonblocking. Điều này tiết kiệm các lần gọi thêm đến `fcntl()` để đạt được kết quả tương tự.

# **56.3 Bind Socket vào Địa chỉ: bind()**

System call `bind()` bind một socket vào một địa chỉ.

```
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
                                             Returns 0 on success, or –1 on error
```

Đối số `sockfd` là một file descriptor thu được từ lần gọi trước đó đến `socket()`. Đối số `addr` là một con trỏ đến một cấu trúc chỉ định địa chỉ mà socket này sẽ được bind. Loại cấu trúc được truyền trong đối số này phụ thuộc vào socket domain. Đối số `addrlen` chỉ định kích thước của cấu trúc địa chỉ. Kiểu dữ liệu `socklen_t` được sử dụng cho đối số `addrlen` là một kiểu số nguyên được chỉ định bởi SUSv3.

Thông thường, chúng ta bind socket của server vào một well-known address — tức là, một địa chỉ cố định được các ứng dụng client cần giao tiếp với server đó biết trước.

> Có những khả năng khác ngoài việc bind socket của server vào một well-known address. Ví dụ, đối với Internet domain socket, server có thể bỏ qua lần gọi đến `bind()` và chỉ đơn giản gọi `listen()`, khiến kernel chọn một ephemeral port cho socket đó. (Chúng ta mô tả ephemeral port trong Mục 58.6.1.) Sau đó, server có thể sử dụng `getsockname()` (Mục 61.5) để lấy địa chỉ của socket của nó. Trong kịch bản này, server sau đó phải công bố địa chỉ đó để các client biết cách định vị socket của server. Việc công bố như vậy có thể được thực hiện bằng cách đăng ký địa chỉ của server với một ứng dụng dịch vụ thư mục tập trung mà các client sau đó liên hệ để lấy địa chỉ. (Ví dụ, Sun RPC giải quyết vấn đề này bằng server portmapper của nó.) Tất nhiên, socket của ứng dụng dịch vụ thư mục phải nằm tại một well-known address.

# **56.4 Cấu trúc Địa chỉ Socket Tổng quát: struct sockaddr**

Các đối số `addr` và `addrlen` của `bind()` cần được giải thích thêm. Nhìn vào Bảng 56-1, chúng ta thấy rằng mỗi socket domain sử dụng một định dạng địa chỉ khác nhau. Ví dụ, UNIX domain socket sử dụng pathname, trong khi Internet domain socket sử dụng sự kết hợp của địa chỉ IP cộng với số port. Đối với mỗi socket domain, một kiểu cấu trúc khác nhau được định nghĩa để lưu trữ địa chỉ socket. Tuy nhiên, vì các system call như `bind()` là chung cho tất cả các socket domain, chúng phải có khả năng chấp nhận cấu trúc địa chỉ của bất kỳ loại nào. Để cho phép điều này, sockets API định nghĩa một cấu trúc địa chỉ tổng quát, `struct sockaddr`. Mục đích duy nhất của kiểu này là để ép kiểu các cấu trúc địa chỉ dành riêng cho domain khác nhau thành một kiểu duy nhất để sử dụng làm đối số trong các system call socket. Cấu trúc `sockaddr` thường được định nghĩa như sau:

```
struct sockaddr {
 sa_family_t sa_family; /* Address family (AF_* constant) */
 char sa_data[14]; /* Socket address (size varies
 according to socket domain) */
};
```

Cấu trúc này đóng vai trò như một mẫu cho tất cả các cấu trúc địa chỉ dành riêng cho domain. Mỗi cấu trúc địa chỉ này bắt đầu với một trường family tương ứng với trường `sa_family` của cấu trúc `sockaddr`. (Kiểu dữ liệu `sa_family_t` là một kiểu số nguyên được chỉ định trong SUSv3.) Giá trị trong trường family đủ để xác định kích thước và định dạng của địa chỉ được lưu trong phần còn lại của cấu trúc.

> Một số triển khai UNIX cũng định nghĩa một trường bổ sung trong cấu trúc `sockaddr`, `sa_len`, chỉ định tổng kích thước của cấu trúc. SUSv3 không yêu cầu trường này, và nó không có mặt trong triển khai Linux của sockets API.

Nếu chúng ta định nghĩa macro kiểm tra tính năng `_GNU_SOURCE`, thì glibc sẽ khai báo nguyên mẫu các system call socket khác nhau trong `<sys/socket.h>` bằng cách sử dụng phần mở rộng gcc để loại bỏ nhu cầu về ép kiểu `(struct sockaddr *)`. Tuy nhiên, việc dựa vào tính năng này là không khả chuyển (nó sẽ tạo ra các cảnh báo biên dịch trên các hệ thống khác).

# **56.5 Stream Socket**

Hoạt động của stream socket có thể được giải thích bằng sự tương tự với hệ thống điện thoại:

1. System call `socket()`, tạo ra một socket, tương đương với việc lắp đặt một chiếc điện thoại. Để hai ứng dụng có thể giao tiếp, mỗi ứng dụng phải tạo một socket.
2. Giao tiếp qua stream socket tương tự như một cuộc gọi điện thoại. Một ứng dụng phải kết nối socket của nó với socket của ứng dụng khác trước khi có thể giao tiếp. Hai socket được kết nối như sau:
   - a) Một ứng dụng gọi `bind()` để bind socket vào một well-known address, sau đó gọi `listen()` để thông báo cho kernel về sự sẵn sàng chấp nhận các kết nối đến. Bước này tương tự như có một số điện thoại đã biết và đảm bảo điện thoại được bật để mọi người có thể gọi cho chúng ta.
   - b) Ứng dụng kia thiết lập kết nối bằng cách gọi `connect()`, chỉ định địa chỉ của socket mà kết nối sẽ được thực hiện. Điều này tương tự như quay số điện thoại của ai đó.
   - c) Ứng dụng đã gọi `listen()` sau đó chấp nhận kết nối bằng cách sử dụng `accept()`. Điều này tương tự như nhấc điện thoại khi nó reo. Nếu `accept()` được thực hiện trước khi ứng dụng peer gọi `connect()`, thì `accept()` sẽ block ("đang đợi bên điện thoại").
3. Sau khi kết nối được thiết lập, dữ liệu có thể được truyền theo cả hai hướng giữa các ứng dụng (tương tự như một cuộc trò chuyện điện thoại hai chiều) cho đến khi một trong số chúng đóng kết nối bằng `close()`. Giao tiếp được thực hiện bằng các system call `read()` và `write()` thông thường hoặc thông qua một số system call dành riêng cho socket (như `send()` và `recv()`) cung cấp thêm chức năng.

Hình 56-1 minh họa cách sử dụng các system call được dùng với stream socket.

#### **Socket chủ động và bị động**

Stream socket thường được phân biệt là chủ động (active) hoặc bị động (passive):

- Theo mặc định, một socket được tạo bằng `socket()` là chủ động. Một active socket có thể được sử dụng trong lần gọi `connect()` để thiết lập kết nối đến một passive socket. Điều này được gọi là thực hiện một active open.
- Một passive socket (còn gọi là listening socket) là socket đã được đánh dấu để cho phép các kết nối đến bằng cách gọi `listen()`. Chấp nhận kết nối đến được gọi là thực hiện một passive open.

Trong hầu hết các ứng dụng sử dụng stream socket, server thực hiện passive open, và client thực hiện active open. Chúng ta giả định kịch bản này trong các mục tiếp theo, để thay vì nói "ứng dụng thực hiện active socket open", chúng ta thường chỉ nói "client". Tương tự, chúng ta sẽ đồng nhất "server" với "ứng dụng thực hiện passive socket open".

**Hình 56-1:** Tổng quan về các system call được sử dụng với stream socket

# **56.5.1 Lắng nghe Kết nối Đến: listen()**

System call `listen()` đánh dấu stream socket được tham chiếu bởi file descriptor `sockfd` là bị động (passive). Socket sau đó sẽ được sử dụng để chấp nhận các kết nối từ các socket khác (chủ động).

```
#include <sys/socket.h>
int listen(int sockfd, int backlog);
                                             Returns 0 on success, or –1 on error
```

Chúng ta không thể áp dụng `listen()` cho một connected socket — tức là, socket mà `connect()` đã được thực hiện thành công hoặc socket được trả về bởi lần gọi đến `accept()`.

Để hiểu mục đích của đối số `backlog`, trước tiên chúng ta quan sát rằng client có thể gọi `connect()` trước khi server gọi `accept()`. Điều này có thể xảy ra, ví dụ, khi server đang bận xử lý một số client khác. Điều này dẫn đến một kết nối đang chờ, như được minh họa trong Hình 56-2.

**Hình 56-2:** Một socket connection đang chờ

Kernel phải ghi lại một số thông tin về mỗi yêu cầu kết nối đang chờ để `accept()` tiếp theo có thể được xử lý. Đối số `backlog` cho phép chúng ta giới hạn số lượng các kết nối đang chờ đó. Các yêu cầu kết nối trong giới hạn này sẽ thành công ngay lập tức. (Đối với TCP socket, câu chuyện phức tạp hơn một chút, như chúng ta sẽ thấy trong Mục 61.6.4.) Các yêu cầu kết nối tiếp theo sẽ block cho đến khi một kết nối đang chờ được chấp nhận (thông qua `accept()`), và do đó bị xóa khỏi hàng đợi các kết nối đang chờ.

SUSv3 cho phép một triển khai đặt giới hạn trên cho giá trị có thể được chỉ định cho `backlog`, và cho phép triển khai ngầm giảm các giá trị backlog xuống giới hạn này. SUSv3 chỉ định rằng triển khai nên quảng bá giới hạn này bằng cách định nghĩa hằng số `SOMAXCONN` trong `<sys/socket.h>`. Trên Linux, hằng số này được định nghĩa với giá trị 128. Tuy nhiên, từ kernel 2.4.25, Linux cho phép điều chỉnh giới hạn này trong thời gian chạy qua file `/proc/sys/net/core/somaxconn` dành riêng cho Linux. (Trong các phiên bản kernel trước đó, giới hạn `SOMAXCONN` là bất biến.)

> Trong triển khai socket BSD gốc, giới hạn trên cho `backlog` là 5, và chúng ta có thể thấy con số này được chỉ định trong code cũ hơn. Tất cả các triển khai hiện đại đều cho phép các giá trị `backlog` cao hơn, cần thiết cho các network server sử dụng TCP socket để phục vụ số lượng lớn client.

# **56.5.2 Chấp nhận Kết nối: accept()**

System call `accept()` chấp nhận một kết nối đến trên listening stream socket được tham chiếu bởi file descriptor `sockfd`. Nếu không có kết nối đang chờ khi `accept()` được gọi, lệnh gọi sẽ block cho đến khi có yêu cầu kết nối đến.

```
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
                               Returns file descriptor on success, or –1 on error
```

Điểm quan trọng cần hiểu về `accept()` là nó tạo ra một socket mới, và chính socket mới này được kết nối với peer socket đã thực hiện `connect()`. File descriptor cho connected socket được trả về là kết quả hàm của lần gọi `accept()`. Listening socket (`sockfd`) vẫn mở, và có thể được sử dụng để chấp nhận thêm các kết nối. Một ứng dụng server điển hình tạo một listening socket, bind nó vào một well-known address, và sau đó xử lý tất cả các yêu cầu client bằng cách chấp nhận kết nối qua socket đó.

Các đối số còn lại của `accept()` trả về địa chỉ của peer socket. Đối số `addr` trỏ đến một cấu trúc được sử dụng để trả về địa chỉ socket. Loại đối số này phụ thuộc vào socket domain (như với `bind()`).

Đối số `addrlen` là một đối số value-result. Nó trỏ đến một số nguyên mà, trước khi gọi, phải được khởi tạo bằng kích thước của buffer được trỏ đến bởi `addr`, để kernel biết có bao nhiêu không gian để trả về địa chỉ socket. Khi trả về từ `accept()`, số nguyên này được đặt để chỉ số byte dữ liệu thực sự được sao chép vào buffer.

Nếu chúng ta không quan tâm đến địa chỉ của peer socket, thì `addr` và `addrlen` nên được chỉ định là `NULL` và 0 tương ứng. (Nếu muốn, chúng ta có thể lấy địa chỉ của peer sau bằng cách sử dụng system call `getpeername()`, như được mô tả trong Mục 61.5.)

> Bắt đầu từ kernel 2.6.28, Linux hỗ trợ một system call mới, không chuẩn, `accept4()`. System call này thực hiện cùng nhiệm vụ như `accept()`, nhưng hỗ trợ đối số bổ sung, `flags`, có thể được sử dụng để sửa đổi hành vi của system call. Hai flag được hỗ trợ: `SOCK_CLOEXEC` và `SOCK_NONBLOCK`. Flag `SOCK_CLOEXEC` khiến kernel bật close-on-exec flag (`FD_CLOEXEC`) cho file descriptor mới được trả về bởi lần gọi. Flag này hữu ích vì những lý do tương tự như flag `O_CLOEXEC` của `open()` được mô tả trong Mục 4.3.1. Flag `SOCK_NONBLOCK` khiến kernel bật flag `O_NONBLOCK` trên open file description bên dưới, để các thao tác I/O trong tương lai trên socket sẽ là nonblocking. Điều này tiết kiệm các lần gọi thêm đến `fcntl()` để đạt được kết quả tương tự.

# **56.5.3 Kết nối đến Peer Socket: connect()**

System call `connect()` kết nối active socket được tham chiếu bởi file descriptor `sockfd` đến listening socket có địa chỉ được chỉ định bởi `addr` và `addrlen`.

```
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
                                            Returns 0 on success, or –1 on error
```

Các đối số `addr` và `addrlen` được chỉ định theo cách tương tự như các đối số tương ứng của `bind()`.

Nếu `connect()` thất bại và chúng ta muốn thử lại kết nối, thì SUSv3 chỉ định rằng phương pháp khả chuyển để làm điều này là đóng socket, tạo socket mới, và thử lại kết nối với socket mới.

# **56.5.4 I/O trên Stream Socket**

Một cặp connected stream socket cung cấp kênh giao tiếp hai chiều giữa hai đầu cuối. Hình 56-3 cho thấy điều này trông như thế nào trong UNIX domain.

**Hình 56-3:** UNIX domain stream socket cung cấp kênh giao tiếp hai chiều

Ngữ nghĩa của I/O trên connected stream socket tương tự như những ngữ nghĩa của pipe:

- Để thực hiện I/O, chúng ta sử dụng các system call `read()` và `write()` (hoặc `send()` và `recv()` dành riêng cho socket, mà chúng ta mô tả trong Mục 61.3). Vì socket là hai chiều, cả hai lần gọi có thể được sử dụng ở mỗi đầu kết nối.
- Một socket có thể được đóng bằng system call `close()` hoặc do ứng dụng kết thúc. Sau đó, khi ứng dụng peer cố gắng đọc từ đầu kia của kết nối, nó nhận được end-of-file (sau khi tất cả dữ liệu được buffer đã được đọc). Nếu ứng dụng peer cố gắng ghi vào socket của nó, nó nhận được signal `SIGPIPE`, và system call thất bại với lỗi `EPIPE`. Như chúng ta đã lưu ý trong Mục 44.2, cách thông thường để xử lý khả năng này là bỏ qua signal `SIGPIPE` và tìm hiểu về kết nối bị đóng thông qua lỗi `EPIPE`.

# **56.5.5 Kết thúc Kết nối: close()**

Cách thông thường để kết thúc kết nối stream socket là gọi `close()`. Nếu nhiều file descriptor tham chiếu đến cùng một socket, thì kết nối bị kết thúc khi tất cả các descriptor được đóng.

Giả sử rằng, sau khi chúng ta đóng kết nối, ứng dụng peer bị crash hoặc không thể đọc hoặc xử lý đúng dữ liệu mà chúng ta đã gửi đến nó trước đó. Trong trường hợp này, chúng ta không có cách nào biết rằng đã xảy ra lỗi. Nếu chúng ta cần đảm bảo rằng dữ liệu đã được đọc và xử lý thành công, thì chúng ta phải xây dựng một số loại giao thức xác nhận (acknowledgement protocol) vào ứng dụng của chúng ta. Điều này thường bao gồm một message xác nhận rõ ràng được truyền lại cho chúng ta từ peer.

Trong Mục 61.2, chúng ta mô tả system call `shutdown()`, cung cấp khả năng kiểm soát chi tiết hơn về cách đóng kết nối stream socket.

# **56.6 Datagram Socket**

Hoạt động của datagram socket có thể được giải thích bằng sự tương tự với hệ thống bưu chính:

1. System call `socket()` tương đương với việc thiết lập một hộp thư. (Ở đây, chúng ta giả định một hệ thống như dịch vụ bưu chính nông thôn ở một số quốc gia, vừa lấy thư từ hộp thư vừa giao thư đến hộp thư.) Mỗi ứng dụng muốn gửi hoặc nhận datagram tạo một datagram socket bằng `socket()`.
2. Để cho phép ứng dụng khác gửi datagram đến nó, một ứng dụng sử dụng `bind()` để bind socket của nó vào một well-known address. Thông thường, server bind socket của nó vào một well-known address, và client khởi tạo giao tiếp bằng cách gửi datagram đến địa chỉ đó. (Trong một số domain — đặc biệt là UNIX domain — client cũng có thể cần sử dụng `bind()` để gán địa chỉ cho socket của nó nếu nó muốn nhận datagram được gửi bởi server.)
3. Để gửi datagram, một ứng dụng gọi `sendto()`, lấy địa chỉ của socket mà datagram sẽ được gửi đến làm một trong các đối số của nó. Điều này tương tự như đặt địa chỉ của người nhận lên thư và gửi đi.
4. Để nhận datagram, một ứng dụng gọi `recvfrom()`, có thể block nếu chưa có datagram nào đến. Vì `recvfrom()` cho phép chúng ta lấy địa chỉ của người gửi, chúng ta có thể gửi phản hồi nếu muốn. (Điều này hữu ích nếu socket của người gửi được bind vào một địa chỉ không phải well-known, điều này là điển hình của client.) Ở đây, chúng ta mở rộng phép tương tự một chút, vì không có yêu cầu nào là thư đã gửi phải được đánh dấu với địa chỉ của người gửi.
5. Khi socket không còn cần thiết nữa, ứng dụng đóng nó bằng `close()`.

Cũng giống như hệ thống bưu chính, khi nhiều datagram (thư) được gửi từ một địa chỉ đến một địa chỉ khác, không có đảm bảo rằng chúng sẽ đến theo thứ tự chúng được gửi, hoặc thậm chí đến được. Datagram thêm một khả năng nữa không có trong hệ thống bưu chính: vì các giao thức mạng bên dưới đôi khi có thể truyền lại một data packet, cùng một datagram có thể đến nhiều hơn một lần.

Hình 56-4 minh họa cách sử dụng các system call được sử dụng với datagram socket.

**Hình 56-4:** Tổng quan về các system call được sử dụng với datagram socket

# **56.6.1 Trao đổi Datagram: recvfrom() và sendto()**

Các system call `recvfrom()` và `sendto()` nhận và gửi datagram trên datagram socket.

```
#include <sys/socket.h>
ssize_t recvfrom(int sockfd, void *buffer, size_t length, int flags,
 struct sockaddr *src_addr, socklen_t *addrlen);
                  Returns number of bytes received, 0 on EOF, or –1 on error
ssize_t sendto(int sockfd, const void *buffer, size_t length, int flags,
 const struct sockaddr *dest_addr, socklen_t addrlen);
                                 Returns number of bytes sent, or –1 on error
```

Giá trị trả về và ba đối số đầu tiên của các system call này giống như với `read()` và `write()`.

Đối số thứ tư, `flags`, là một bit mask kiểm soát các tính năng I/O dành riêng cho socket. Chúng ta đề cập đến các tính năng này khi mô tả các system call `recv()` và `send()` trong Mục 61.3. Nếu chúng ta không cần bất kỳ tính năng nào trong số này, chúng ta có thể chỉ định `flags` là 0.

Các đối số `src_addr` và `addrlen` được sử dụng để lấy hoặc chỉ định địa chỉ của peer socket mà chúng ta đang giao tiếp.

Đối với `recvfrom()`, các đối số `src_addr` và `addrlen` trả về địa chỉ của remote socket được sử dụng để gửi datagram. (Các đối số này tương tự như đối số `addr` và `addrlen` của `accept()`, trả về địa chỉ của peer socket đang kết nối.) Đối số `src_addr` là con trỏ đến cấu trúc địa chỉ phù hợp với communication domain. Như với `accept()`, `addrlen` là đối số value-result. Trước khi gọi, `addrlen` nên được khởi tạo bằng kích thước của cấu trúc được trỏ đến bởi `src_addr`; khi trả về, nó chứa số byte thực sự được ghi vào cấu trúc này.

Nếu chúng ta không quan tâm đến địa chỉ của người gửi, thì chúng ta chỉ định cả `src_addr` và `addrlen` là `NULL`. Trong trường hợp này, `recvfrom()` tương đương với việc sử dụng `recv()` để nhận datagram. Chúng ta cũng có thể sử dụng `read()` để đọc datagram, tương đương với việc sử dụng `recv()` với đối số `flags` là 0.

Bất kể giá trị được chỉ định cho `length`, `recvfrom()` lấy chính xác một message từ datagram socket. Nếu kích thước của message đó vượt quá `length` byte, message sẽ bị cắt bớt ngầm xuống còn `length` byte.

> Nếu chúng ta sử dụng system call `recvmsg()` (Mục 61.13.2), thì có thể tìm hiểu về datagram bị cắt bớt thông qua flag `MSG_TRUNC` được trả về trong trường `msg_flags` của cấu trúc `msghdr` được trả về. Xem trang man `recvmsg(2)` để biết chi tiết.

Đối với `sendto()`, các đối số `dest_addr` và `addrlen` chỉ định socket mà datagram sẽ được gửi đến. Các đối số này được sử dụng theo cách tương tự như các đối số tương ứng của `connect()`. Đối số `dest_addr` là cấu trúc địa chỉ phù hợp cho communication domain này. Nó được khởi tạo với địa chỉ của destination socket. Đối số `addrlen` chỉ định kích thước của `addr`.

> Trên Linux, có thể sử dụng `sendto()` để gửi datagram có độ dài 0. Tuy nhiên, không phải tất cả các triển khai UNIX đều cho phép điều này.

# **56.6.2 Sử dụng connect() với Datagram Socket**

Mặc dù datagram socket là connectionless, system call `connect()` có một mục đích khi được áp dụng cho datagram socket. Gọi `connect()` trên datagram socket khiến kernel ghi lại một địa chỉ cụ thể là peer của socket này. Thuật ngữ connected datagram socket được áp dụng cho socket như vậy. Thuật ngữ unconnected datagram socket được áp dụng cho datagram socket mà `connect()` chưa được gọi (tức là mặc định cho datagram socket mới).

Sau khi datagram socket được kết nối:

- Datagram có thể được gửi qua socket bằng cách sử dụng `write()` (hoặc `send()`) và tự động được gửi đến cùng peer socket. Cũng như `sendto()`, mỗi lần gọi `write()` tạo ra một datagram riêng biệt.
- Chỉ các datagram được gửi bởi peer socket mới có thể được đọc trên socket.

Lưu ý rằng hiệu ứng của `connect()` là bất đối xứng đối với datagram socket. Các phát biểu trên chỉ áp dụng cho socket mà `connect()` đã được gọi, không phải cho remote socket mà nó được kết nối đến (trừ khi ứng dụng peer cũng gọi `connect()` trên socket của nó).

Chúng ta có thể thay đổi peer của connected datagram socket bằng cách thực hiện lần gọi `connect()` tiếp theo. Cũng có thể giải thể quan hệ peer hoàn toàn bằng cách chỉ định một cấu trúc địa chỉ trong đó address family (ví dụ: trường `sun_family` trong UNIX domain) được chỉ định là `AF_UNSPEC`. Tuy nhiên, lưu ý rằng nhiều triển khai UNIX khác không hỗ trợ việc sử dụng `AF_UNSPEC` cho mục đích này.

> SUSv3 còn hơi mơ hồ về việc giải thể các quan hệ peer, nói rằng kết nối có thể được đặt lại bằng cách thực hiện lần gọi `connect()` chỉ định "địa chỉ null", mà không định nghĩa thuật ngữ đó. SUSv4 chỉ định rõ ràng việc sử dụng `AF_UNSPEC`.

Lợi thế rõ ràng của việc đặt peer cho datagram socket là chúng ta có thể sử dụng các system call I/O đơn giản hơn khi truyền dữ liệu trên socket. Chúng ta không còn cần sử dụng `sendto()` với các đối số `dest_addr` và `addrlen`, mà thay vào đó có thể sử dụng `write()`. Việc đặt peer hữu ích chủ yếu trong một ứng dụng cần gửi nhiều datagram đến một peer duy nhất (điều này là điển hình của một số datagram client).

> Trên một số triển khai TCP/IP, kết nối datagram socket với peer mang lại cải thiện hiệu suất ([Stevens et al., 2004]). Trên Linux, kết nối datagram socket ít ảnh hưởng đến hiệu suất.

# **56.7 Tóm tắt**

Socket cho phép giao tiếp giữa các ứng dụng trên cùng host hoặc trên các host khác nhau được kết nối qua mạng.

Một socket tồn tại trong một communication domain, xác định phạm vi giao tiếp và định dạng địa chỉ được sử dụng để nhận dạng socket. SUSv3 chỉ định các communication domain UNIX (`AF_UNIX`), IPv4 (`AF_INET`), và IPv6 (`AF_INET6`).

Hầu hết các ứng dụng sử dụng một trong hai loại socket: stream hoặc datagram. Stream socket (`SOCK_STREAM`) cung cấp kênh giao tiếp byte-stream đáng tin cậy, hai chiều giữa hai đầu cuối. Datagram socket (`SOCK_DGRAM`) cung cấp giao tiếp không đáng tin cậy, connectionless, hướng message.

Một stream socket server điển hình tạo socket của nó bằng `socket()`, sau đó bind socket vào một well-known address bằng `bind()`. Server sau đó gọi `listen()` để cho phép nhận các kết nối trên socket. Mỗi kết nối client sau đó được chấp nhận trên listening socket bằng `accept()`, trả về file descriptor cho socket mới được kết nối với socket của client. Một stream socket client điển hình tạo socket bằng `socket()`, sau đó thiết lập kết nối bằng cách gọi `connect()`, chỉ định well-known address của server. Sau khi hai stream socket được kết nối, dữ liệu có thể được truyền theo cả hai hướng bằng `read()` và `write()`. Sau khi tất cả các process có file descriptor tham chiếu đến đầu cuối của stream socket đã thực hiện đóng ngầm hoặc rõ ràng, kết nối bị kết thúc.

Một datagram socket server điển hình tạo socket bằng `socket()`, sau đó bind nó vào một well-known address bằng `bind()`. Vì datagram socket là connectionless, socket của server có thể được sử dụng để nhận datagram từ bất kỳ client nào. Datagram có thể được nhận bằng `read()` hoặc sử dụng system call `recvfrom()` dành riêng cho socket, trả về địa chỉ của socket gửi. Một datagram socket client tạo socket bằng `socket()`, sau đó sử dụng `sendto()` để gửi datagram đến một địa chỉ được chỉ định (tức là địa chỉ của server). System call `connect()` có thể được sử dụng với datagram socket để đặt peer address cho socket. Sau khi thực hiện điều này, không cần thiết phải chỉ định địa chỉ đích cho các datagram đi; một lần gọi `write()` có thể được sử dụng để gửi datagram.

#### **Thông tin thêm**

Tham khảo các nguồn thông tin thêm được liệt kê trong Mục 59.15.
