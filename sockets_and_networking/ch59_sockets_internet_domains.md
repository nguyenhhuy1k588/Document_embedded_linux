## Chương 59
# SOCKETS: INTERNET DOMAIN

Sau khi đã tìm hiểu các khái niệm socket chung và bộ giao thức TCP/IP trong các chương trước, giờ đây chúng ta đã sẵn sàng để xem xét lập trình với socket trong các domain IPv4 (`AF_INET`) và IPv6 (`AF_INET6`).

Như đã đề cập trong Chương 58, địa chỉ Internet domain socket bao gồm địa chỉ IP và port number. Mặc dù máy tính sử dụng biểu diễn nhị phân của địa chỉ IP và port number, con người xử lý tên tốt hơn nhiều so với số. Do đó, chúng ta mô tả các kỹ thuật được dùng để nhận dạng máy tính host và các port bằng tên. Chúng ta cũng kiểm tra việc sử dụng các library function để lấy địa chỉ IP cho một hostname cụ thể và port number tương ứng với một tên dịch vụ cụ thể. Cuộc thảo luận về hostname của chúng ta bao gồm mô tả về Domain Name System (DNS), hệ thống này implement một cơ sở dữ liệu phân tán ánh xạ hostname sang địa chỉ IP và ngược lại.

# **59.1 Internet Domain Socket**

Internet domain stream socket được implement trên TCP. Chúng cung cấp kênh giao tiếp byte-stream, bidirectional, đáng tin cậy.

Internet domain datagram socket được implement trên UDP. UDP socket tương tự như các UNIX domain counterpart của chúng, nhưng lưu ý các điểm khác biệt sau:

- UNIX domain datagram socket là đáng tin cậy, nhưng UDP socket thì không—datagram có thể bị mất, bị trùng lặp, hoặc đến theo thứ tự khác với thứ tự chúng được gửi.
- Việc gửi trên UNIX domain datagram socket sẽ block nếu hàng đợi dữ liệu cho socket nhận đầy. Ngược lại, với UDP, nếu datagram đến sẽ làm tràn hàng đợi của receiver, thì datagram sẽ bị loại bỏ im lặng.

# **59.2 Network Byte Order**

Địa chỉ IP và port number là các giá trị nguyên. Một vấn đề chúng ta gặp phải khi truyền các giá trị này qua mạng là các kiến trúc phần cứng khác nhau lưu trữ các byte của số nguyên đa byte theo các thứ tự khác nhau. Như được hiển thị trong Hình 59-1, các kiến trúc lưu trữ số nguyên với byte quan trọng nhất (MSB) trước (tức là tại địa chỉ bộ nhớ thấp nhất) được gọi là big endian; những kiến trúc lưu trữ byte ít quan trọng nhất trước được gọi là little endian. (Các thuật ngữ này bắt nguồn từ cuốn tiểu thuyết châm biếm năm 1726 Gulliver's Travels của Jonathan Swift.) Ví dụ đáng chú ý nhất về kiến trúc little-endian là x86. Hầu hết các kiến trúc khác đều là big endian. Một số kiến trúc phần cứng có thể chuyển đổi giữa hai định dạng. Thứ tự byte được sử dụng trên một máy cụ thể được gọi là host byte order.

|                             | 2-byte integer |                  | 4-byte integer |              |                  |                  |                  |
|-----------------------------|----------------|------------------|----------------|--------------|------------------|------------------|------------------|
|                             | address<br>N   | address<br>N + 1 |                | address<br>N | address<br>N + 1 | address<br>N + 2 | address<br>N + 3 |
| Big-endian<br>byte order    | 1<br>(MSB)     | 0<br>(LSB)       |                | 3<br>(MSB)   | 2                | 1                | 0<br>(LSB)       |
|                             | address<br>N   | address<br>N + 1 |                | address<br>N | address<br>N + 1 | address<br>N + 2 | address<br>N + 3 |
| Little-endian<br>byte order | 0<br>(LSB)     | 1<br>(MSB)       |                | 0<br>(LSB)   | 1                | 2                | 3<br>(MSB)       |

MSB = Most Significant Byte, LSB = Least Significant Byte

**Hình 59-1:** Thứ tự byte big-endian và little-endian cho số nguyên 2-byte và 4-byte

Vì port number và địa chỉ IP phải được truyền giữa, và được hiểu bởi, tất cả các host trên mạng, một thứ tự chuẩn phải được sử dụng. Thứ tự này được gọi là network byte order, và tình cờ là big endian.

Sau trong chương này, chúng ta xem xét các hàm khác nhau chuyển đổi hostname (ví dụ: www.kernel.org) và tên dịch vụ (ví dụ: http) thành các dạng số tương ứng. Các hàm này thường trả về số nguyên theo network byte order, và các số nguyên này có thể được sao chép trực tiếp vào các trường liên quan của cấu trúc địa chỉ socket.

Tuy nhiên, đôi khi chúng ta sử dụng trực tiếp các hằng số nguyên cho địa chỉ IP và port number. Ví dụ, chúng ta có thể chọn hard-code một port number vào chương trình, chỉ định port number như đối số dòng lệnh cho chương trình, hoặc sử dụng các hằng số như `INADDR_ANY` và `INADDR_LOOPBACK` khi chỉ định địa chỉ IPv4. Các giá trị này được biểu diễn trong C theo các quy ước của máy host, vì vậy chúng theo host byte order. Chúng ta phải chuyển đổi các giá trị này sang network byte order trước khi lưu chúng vào các cấu trúc địa chỉ socket.

Các hàm `htons()`, `htonl()`, `ntohs()` và `ntohl()` được định nghĩa (thường là macro) để chuyển đổi số nguyên theo một trong hai hướng giữa host byte order và network byte order.

```
#include <arpa/inet.h>
uint16_t htons(uint16_t host_uint16);
                           Returns host_uint16 converted to network byte order
uint32_t htonl(uint32_t host_uint32);
                           Returns host_uint32 converted to network byte order
uint16_t ntohs(uint16_t net_uint16);
                                Returns net_uint16 converted to host byte order
uint32_t ntohl(uint32_t net_uint32);
                                Returns net_uint32 converted to host byte order
```

Nói một cách chặt chẽ, việc sử dụng bốn hàm này chỉ cần thiết trên các hệ thống nơi host byte order khác với network byte order. Tuy nhiên, các hàm này phải luôn luôn được sử dụng, để các chương trình có thể portable sang các kiến trúc phần cứng khác nhau. Trên các hệ thống nơi host byte order giống với network byte order, các hàm này chỉ đơn giản trả về đối số của chúng mà không thay đổi.

# **59.3 Biểu Diễn Dữ Liệu**

Khi viết các chương trình mạng, chúng ta cần biết rằng các kiến trúc máy tính khác nhau sử dụng các quy ước khác nhau để biểu diễn các kiểu dữ liệu khác nhau. Chúng ta đã lưu ý rằng các kiểu nguyên có thể được lưu trữ theo dạng big-endian hoặc little-endian. Ngoài ra cũng có các điểm khác biệt có thể. Ví dụ, kiểu `long` trong C có thể là 32 bit trên một số hệ thống và 64 bit trên các hệ thống khác. Khi chúng ta xem xét các cấu trúc, vấn đề càng phức tạp hơn vì các implementation khác nhau sử dụng các quy tắc khác nhau để căn chỉnh các trường của cấu trúc với các ranh giới địa chỉ trên hệ thống host, để lại số byte padding khác nhau giữa các trường.

Do những khác biệt này trong biểu diễn dữ liệu, các ứng dụng trao đổi dữ liệu giữa các hệ thống không đồng nhất qua mạng phải áp dụng một số quy ước chung để mã hóa dữ liệu đó. Quá trình đặt dữ liệu vào định dạng chuẩn để truyền qua mạng được gọi là marshalling. Nhiều tiêu chuẩn marshalling tồn tại, chẳng hạn như XDR (External Data Representation, được mô tả trong RFC 1014), ASN.1-BER, CORBA và XML.

Tuy nhiên, một phương pháp đơn giản hơn marshalling thường được sử dụng: mã hóa tất cả dữ liệu được truyền dưới dạng văn bản, với các mục dữ liệu riêng biệt được phân tách bằng một ký tự được chỉ định, thường là ký tự newline. Một ưu điểm của phương pháp này là chúng ta có thể dùng telnet để debug một ứng dụng. Để làm điều này, chúng ta dùng lệnh sau:

#### $ **telnet** *host port*

Sau đó chúng ta có thể gõ các dòng văn bản để truyền đến ứng dụng, và xem các phản hồi được gửi bởi ứng dụng. Chúng ta minh họa kỹ thuật này trong Mục 59.11.

Nếu chúng ta mã hóa dữ liệu được truyền trên stream socket dưới dạng văn bản được phân tách bằng newline, thì sẽ thuận tiện khi định nghĩa một hàm như `readLine()`, được hiển thị trong Listing 59-1.

```
#include "read_line.h"
ssize_t readLine(int fd, void *buffer, size_t n);
                           Returns number of bytes copied into buffer (excluding
                        terminating null byte), or 0 on end-of-file, or –1 on error
```

Hàm `readLine()` đọc byte từ file được tham chiếu bởi đối số file descriptor `fd` cho đến khi gặp newline. Chuỗi byte đầu vào được trả về tại vị trí được trỏ bởi `buffer`, phải trỏ đến vùng bộ nhớ ít nhất `n` byte. Chuỗi trả về luôn kết thúc bằng null; do đó, tối đa (`n – 1`) byte dữ liệu thực sự sẽ được trả về. Khi thành công, `readLine()` trả về số byte dữ liệu được đặt trong `buffer`; byte null kết thúc không được tính trong số này.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/read_line.c
#include <unistd.h>
#include <errno.h>
#include "read_line.h" /* Declaration of readLine() */
ssize_t
readLine(int fd, void *buffer, size_t n)
{
 ssize_t numRead; /* # of bytes fetched by last read() */
 size_t totRead; /* Total bytes read so far */
 char *buf;
 char ch;
 if (n <= 0 || buffer == NULL) {
 errno = EINVAL;
 return -1;
 }
 buf = buffer; /* No pointer arithmetic on "void *" */
 totRead = 0;
 for (;;) {
 numRead = read(fd, &ch, 1);
 if (numRead == -1) {
 if (errno == EINTR) /* Interrupted --> restart read() */
 continue;
 else
 return -1; /* Some other error */
 } else if (numRead == 0) { /* EOF */
 if (totRead == 0) /* No bytes read; return 0 */
 return 0;
 else /* Some bytes read; add '\0' */
 break;
 } else { /* 'numRead' must be 1 if we get here */
 if (totRead < n - 1) { /* Discard > (n - 1) bytes */
 totRead++;
 *buf++ = ch;
 }
 if (ch == '\n')
 break;
 }
 }
 *buf = '\0';
 return totRead;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/read_line.c
```

Chúng ta sử dụng hàm `readLine()` trong các chương trình ví dụ được trình bày trong Mục 59.11.

# **59.4 Địa Chỉ Internet Socket**

Có hai loại địa chỉ Internet domain socket: IPv4 và IPv6.

#### **Địa chỉ IPv4 socket: struct sockaddr\_in**

Địa chỉ IPv4 socket được lưu trữ trong cấu trúc `sockaddr_in`, được định nghĩa trong `<netinet/in.h>` như sau:

```
struct in_addr { /* IPv4 4-byte address */
 in_addr_t s_addr; /* Unsigned 32-bit integer */
};
struct sockaddr_in { /* IPv4 socket address */
 sa_family_t sin_family; /* Address family (AF_INET) */
 in_port_t sin_port; /* Port number */
 struct in_addr sin_addr; /* IPv4 address */
 unsigned char __pad[X]; /* Pad to size of 'sockaddr'
 structure (16 bytes) */
};
```

Trong Mục 56.4, chúng ta đã thấy rằng cấu trúc `sockaddr` chung bắt đầu với một trường nhận dạng socket domain. Điều này tương ứng với trường `sin_family` trong cấu trúc `sockaddr_in`, luôn được đặt thành `AF_INET`. Các trường `sin_port` và `sin_addr` là port number và địa chỉ IP, cả hai đều ở network byte order. Các kiểu dữ liệu `in_port_t` và `in_addr_t` là các kiểu số nguyên không dấu, lần lượt là 16 và 32 bit.

#### **Địa chỉ IPv6 socket: struct sockaddr\_in6**

Giống như địa chỉ IPv4, địa chỉ IPv6 socket bao gồm địa chỉ IP và port number. Sự khác biệt là địa chỉ IPv6 là 128 bit thay vì 32 bit. Địa chỉ IPv6 socket được lưu trữ trong cấu trúc `sockaddr_in6`, được định nghĩa trong `<netinet/in.h>` như sau:

```
struct in6_addr { /* IPv6 address structure */
 uint8_t s6_addr[16]; /* 16 bytes == 128 bits */
};
```

```
struct sockaddr_in6 { /* IPv6 socket address */
 sa_family_t sin6_family; /* Address family (AF_INET6) */
 in_port_t sin6_port; /* Port number */
 uint32_t sin6_flowinfo; /* IPv6 flow information */
 struct in6_addr sin6_addr; /* IPv6 address */
 uint32_t sin6_scope_id; /* Scope ID (new in kernel 2.4) */
};
```

Trường `sin_family` được đặt thành `AF_INET6`. Các trường `sin6_port` và `sin6_addr` là port number và địa chỉ IP. Các trường còn lại, `sin6_flowinfo` và `sin6_scope_id`, vượt quá phạm vi của cuốn sách này; với mục đích của chúng ta, chúng luôn được đặt thành 0. Tất cả các trường trong cấu trúc `sockaddr_in6` đều ở network byte order.

IPv6 có các tương đương của địa chỉ wildcard và loopback IPv4. Hằng số `IN6ADDR_ANY_INIT` được định nghĩa cho địa chỉ wildcard IPv6 (0::0):

```
#define IN6ADDR_ANY_INIT { { 0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0 } }
```

Chúng ta có thể dùng hằng số `IN6ADDR_ANY_INIT` trong bộ khởi tạo đi kèm với khai báo biến, nhưng không thể dùng nó ở phía phải của câu lệnh gán, vì cú pháp C không cho phép dùng hằng số có cấu trúc trong phép gán. Thay vào đó, chúng ta phải dùng một biến được định nghĩa trước, `in6addr_any`, được thư viện C khởi tạo như sau:

```
const struct in6_addr in6addr_any = IN6ADDR_ANY_INIT;
```

Do đó, chúng ta có thể khởi tạo cấu trúc địa chỉ IPv6 socket bằng địa chỉ wildcard như sau:

```
struct sockaddr_in6 addr;
memset(&addr, 0, sizeof(struct sockaddr_in6));
addr.sin6_family = AF_INET6;
addr.sin6_addr = in6addr_any;
addr.sin6_port = htons(SOME_PORT_NUM);
```

Hằng số và biến tương ứng cho địa chỉ loopback IPv6 (::1) là `IN6ADDR_LOOPBACK_INIT` và `in6addr_loopback`.

Nếu IPv4 và IPv6 cùng tồn tại trên một host, chúng chia sẻ cùng không gian port-number. Điều này có nghĩa là nếu, ví dụ, một ứng dụng bind một IPv6 socket vào TCP port 2000 (sử dụng địa chỉ wildcard IPv6), thì một IPv4 TCP socket không thể được bind vào cùng port đó.

#### **Cấu trúc sockaddr\_storage**

Với IPv6 sockets API, cấu trúc chung mới `sockaddr_storage` đã được giới thiệu. Cấu trúc này được định nghĩa đủ lớn để chứa bất kỳ loại địa chỉ socket nào. Cấu trúc `sockaddr_storage` được định nghĩa trên Linux như sau:

```
#define __ss_aligntype uint32_t /* On 32-bit architectures */
struct sockaddr_storage {
 sa_family_t ss_family;
 __ss_aligntype __ss_align; /* Force alignment */
 char __ss_padding[SS_PADSIZE]; /* Pad to 128 bytes */
};
```

# **59.5 Tổng Quan về Các Hàm Chuyển Đổi Host và Dịch Vụ**

Máy tính biểu diễn địa chỉ IP và port number ở dạng nhị phân. Tuy nhiên, con người thấy tên dễ nhớ hơn số. Việc dùng tên tượng trưng cũng cung cấp một mức độ định hướng gián tiếp hữu ích; người dùng và chương trình có thể tiếp tục dùng cùng tên ngay cả khi giá trị số bên dưới thay đổi.

Một hostname là định danh tượng trưng cho một hệ thống được kết nối với mạng (có thể có nhiều địa chỉ IP). Tên dịch vụ là biểu diễn tượng trưng của port number.

Các phương pháp sau có thể dùng để biểu diễn địa chỉ host và port:

- Địa chỉ host có thể được biểu diễn dưới dạng giá trị nhị phân, hostname tượng trưng, hoặc dạng presentation (dotted-decimal cho IPv4 hoặc hex-string cho IPv6).
- Một port có thể được biểu diễn dưới dạng giá trị nhị phân hoặc tên dịch vụ tượng trưng.

Nhiều library function được cung cấp để chuyển đổi giữa các định dạng này. Phần này tóm tắt ngắn gọn các hàm này. Các phần tiếp theo mô tả các API hiện đại (`inet_ntop()`, `inet_pton()`, `getaddrinfo()`, `getnameinfo()`, và v.v.) chi tiết.

## **Chuyển đổi địa chỉ IPv4 giữa dạng nhị phân và dạng người đọc được**

Các hàm `inet_aton()` và `inet_ntoa()` chuyển đổi địa chỉ IPv4 theo ký hiệu dotted-decimal sang nhị phân và ngược lại. Chúng ta mô tả các hàm này chủ yếu vì chúng xuất hiện trong code lịch sử. Ngày nay, chúng đã lỗi thời. Các chương trình hiện đại cần thực hiện các chuyển đổi như vậy nên dùng các hàm mà chúng ta mô tả tiếp theo.

## **Chuyển đổi địa chỉ IPv4 và IPv6 giữa dạng nhị phân và dạng người đọc được**

Các hàm `inet_pton()` và `inet_ntop()` giống như `inet_aton()` và `inet_ntoa()`, nhưng khác ở chỗ chúng cũng xử lý địa chỉ IPv6. Chúng chuyển đổi địa chỉ IPv4 và IPv6 nhị phân sang và từ dạng presentation—tức là, ký hiệu dotted-decimal hoặc hex-string.

Vì con người xử lý tên tốt hơn số, chúng ta thường chỉ dùng các hàm này đôi khi trong chương trình.

## **Chuyển đổi tên host và dịch vụ sang và từ dạng nhị phân (lỗi thời)**

Hàm `gethostbyname()` trả về các địa chỉ IP nhị phân tương ứng với hostname và hàm `getservbyname()` trả về port number tương ứng với tên dịch vụ. Các chuyển đổi ngược lại được thực hiện bởi `gethostbyaddr()` và `getservbyport()`. Chúng ta mô tả các hàm này vì chúng được sử dụng rộng rãi trong code hiện có. Tuy nhiên, chúng hiện đã lỗi thời. Code mới nên dùng các hàm `getaddrinfo()` và `getnameinfo()` (được mô tả tiếp theo) cho các chuyển đổi như vậy.

## **Chuyển đổi tên host và dịch vụ sang và từ dạng nhị phân (hiện đại)**

Hàm `getaddrinfo()` là người kế thừa hiện đại cho cả `gethostbyname()` và `getservbyname()`. Cho tên host và tên dịch vụ, `getaddrinfo()` trả về một tập hợp các cấu trúc chứa địa chỉ IP nhị phân và port number tương ứng. Không giống như `gethostbyname()`, `getaddrinfo()` xử lý trong suốt cả địa chỉ IPv4 và IPv6. Do đó, chúng ta có thể dùng nó để viết các chương trình không chứa các dependency về phiên bản IP đang được sử dụng. Tất cả code mới nên dùng `getaddrinfo()` để chuyển đổi hostname và tên dịch vụ sang biểu diễn nhị phân.

Hàm `getnameinfo()` thực hiện dịch ngược lại, chuyển đổi địa chỉ IP và port number thành hostname và tên dịch vụ tương ứng.

# **59.6 Các Hàm inet\_pton() và inet\_ntop()**

Các hàm `inet_pton()` và `inet_ntop()` cho phép chuyển đổi cả địa chỉ IPv4 và IPv6 giữa dạng nhị phân và ký hiệu dotted-decimal hoặc hex-string.

```
#include <arpa/inet.h>
int inet_pton(int domain, const char *src_str, void *addrptr);
                          Returns 1 on successful conversion, 0 if src_str is not in
                                              presentation format, or –1 on error
const char *inet_ntop(int domain, const void *addrptr, char *dst_str, size_t len);
                           Returns pointer to dst_str on success, or NULL on error
```

Chữ `p` trong tên của các hàm này là viết tắt cho "presentation", và `n` là viết tắt cho "network". Dạng presentation là một chuỗi người đọc được, chẳng hạn như:

- 204.152.189.116 (địa chỉ IPv4 dotted-decimal);
- ::1 (địa chỉ IPv6 colon-separated hexadecimal); hoặc
- ::FFFF:204.152.189.116 (địa chỉ IPv4-mapped IPv6).

Hàm `inet_pton()` chuyển đổi chuỗi presentation chứa trong `src_str` thành địa chỉ IP nhị phân theo network byte order. Đối số `domain` phải được chỉ định là `AF_INET` hoặc `AF_INET6`. Địa chỉ được chuyển đổi được đặt trong cấu trúc được trỏ bởi `addrptr`, phải trỏ đến cấu trúc `in_addr` hoặc `in6_addr`, tùy theo giá trị được chỉ định trong `domain`.

Hàm `inet_ntop()` thực hiện chuyển đổi ngược lại. Một lần nữa, `domain` phải được chỉ định là `AF_INET` hoặc `AF_INET6`, và `addrptr` phải trỏ đến cấu trúc `in_addr` hoặc `in6_addr` mà chúng ta muốn chuyển đổi. Chuỗi kết thúc bằng null kết quả được đặt trong buffer được trỏ bởi `dst_str`. Đối số `len` phải chỉ định kích thước của buffer này.

Để định kích thước chính xác cho buffer được trỏ bởi `dst_str`, chúng ta có thể sử dụng hai hằng số được định nghĩa trong `<netinet/in.h>`:

```
#define INET_ADDRSTRLEN 16 /* Maximum IPv4 dotted-decimal string */
#define INET6_ADDRSTRLEN 46 /* Maximum IPv6 hexadecimal string */
```

# **59.7 Ví Dụ Client-Server (Datagram Socket)**

Trong phần này, chúng ta lấy các chương trình server và client chuyển đổi chữ hoa được hiển thị trong Mục 57.3 và sửa đổi chúng để dùng datagram socket trong domain `AF_INET6`. Chúng ta trình bày các chương trình này với ít chú thích tối thiểu, vì cấu trúc của chúng tương tự như các chương trình trước. Các điểm khác biệt chính trong các chương trình mới nằm ở việc khai báo và khởi tạo cấu trúc địa chỉ IPv6 socket.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/i6d_ucase.h
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <ctype.h>
#include "tlpi_hdr.h"
#define BUF_SIZE 10 /* Maximum size of messages exchanged
 between client and server */
#define PORT_NUM 50002 /* Server port number */
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/i6d_ucase.h
```

```
–––––––––––––––––––––––––––––––––––––––– sockets/i6d_ucase_sv.c
#include "i6d_ucase.h"
int
main(int argc, char *argv[])
{
 struct sockaddr_in6 svaddr, claddr;
 int sfd, j;
 ssize_t numBytes;
 socklen_t len;
 char buf[BUF_SIZE];
 char claddrStr[INET6_ADDRSTRLEN];
 sfd = socket(AF_INET6, SOCK_DGRAM, 0);
 if (sfd == -1)
 errExit("socket");
 memset(&svaddr, 0, sizeof(struct sockaddr_in6));
 svaddr.sin6_family = AF_INET6;
 svaddr.sin6_addr = in6addr_any; /* Wildcard address */
 svaddr.sin6_port = htons(PORT_NUM);
 if (bind(sfd, (struct sockaddr *) &svaddr,
 sizeof(struct sockaddr_in6)) == -1)
 errExit("bind");
 /* Receive messages, convert to uppercase, and return to client */
 for (;;) {
 len = sizeof(struct sockaddr_in6);
 numBytes = recvfrom(sfd, buf, BUF_SIZE, 0,
 (struct sockaddr *) &claddr, &len);
 if (numBytes == -1)
 errExit("recvfrom");
 if (inet_ntop(AF_INET6, &claddr.sin6_addr, claddrStr,
 INET6_ADDRSTRLEN) == NULL)
 printf("Couldn't convert client address to string\n");
 else
 printf("Server received %ld bytes from (%s, %u)\n",
 (long) numBytes, claddrStr, ntohs(claddr.sin6_port));
 for (j = 0; j < numBytes; j++)
 buf[j] = toupper((unsigned char) buf[j]);
 if (sendto(sfd, buf, numBytes, 0, (struct sockaddr *) &claddr, len) !=
 numBytes)
 fatal("sendto");
 }
}
–––––––––––––––––––––––––––––––––––––––– sockets/i6d_ucase_sv.c
```

```
–––––––––––––––––––––––––––––––––––––––– sockets/i6d_ucase_cl.c
#include "i6d_ucase.h"
int
main(int argc, char *argv[])
{
 struct sockaddr_in6 svaddr;
 int sfd, j;
 size_t msgLen;
 ssize_t numBytes;
 char resp[BUF_SIZE];
 if (argc < 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s host-address msg...\n", argv[0]);
 sfd = socket(AF_INET6, SOCK_DGRAM, 0); /* Create client socket */
 if (sfd == -1)
 errExit("socket");
 memset(&svaddr, 0, sizeof(struct sockaddr_in6));
 svaddr.sin6_family = AF_INET6;
 svaddr.sin6_port = htons(PORT_NUM);
 if (inet_pton(AF_INET6, argv[1], &svaddr.sin6_addr) <= 0)
 fatal("inet_pton failed for address '%s'", argv[1]);
 /* Send messages to server; echo responses on stdout */
 for (j = 2; j < argc; j++) {
 msgLen = strlen(argv[j]);
 if (sendto(sfd, argv[j], msgLen, 0, (struct sockaddr *) &svaddr,
 sizeof(struct sockaddr_in6)) != msgLen)
 fatal("sendto");
 numBytes = recvfrom(sfd, resp, BUF_SIZE, 0, NULL, NULL);
 if (numBytes == -1)
 errExit("recvfrom");
 printf("Response %d: %.*s\n", j - 1, (int) numBytes, resp);
 }
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––– sockets/i6d_ucase_cl.c
```

# **59.8 Domain Name System (DNS)**

Trong Mục 59.10, chúng ta mô tả `getaddrinfo()`, lấy địa chỉ IP tương ứng với hostname, và `getnameinfo()`, thực hiện nhiệm vụ ngược lại. Tuy nhiên, trước khi xem xét các hàm này, chúng ta giải thích cách DNS được dùng để duy trì ánh xạ giữa hostname và địa chỉ IP.

Trước khi có DNS, ánh xạ giữa hostname và địa chỉ IP được định nghĩa trong file cục bộ được duy trì thủ công, `/etc/hosts`, chứa các bản ghi có dạng sau:

```
# IP-address canonical hostname [aliases]
127.0.0.1 localhost
```

Hàm `gethostbyname()` (người tiền nhiệm của `getaddrinfo()`) lấy địa chỉ IP bằng cách tìm kiếm file này, tìm kiếm kết quả phù hợp trên canonical hostname (tức là tên chính thức hoặc tên chính của host) hoặc một trong các alias (tùy chọn, phân tách bằng dấu cách).

Tuy nhiên, sơ đồ `/etc/hosts` mở rộng kém, và sau đó trở nên không thể, khi số lượng host trong mạng tăng lên (ví dụ: Internet, với hàng triệu host). DNS được thiết kế để giải quyết vấn đề này. Các ý tưởng chính của DNS là:

- Hostname được tổ chức thành một namespace phân cấp (Hình 59-2). Mỗi nút trong phân cấp DNS có một nhãn (tên), có thể dài đến 63 ký tự. Ở gốc của phân cấp là một nút không có tên, "anonymous root".
- Domain name của một nút bao gồm tất cả các tên từ nút đó lên đến root được nối với nhau, với mỗi tên được phân tách bằng dấu chấm (.). Ví dụ, `google.com` là domain name cho nút google.
- Một fully qualified domain name (FQDN), chẳng hạn như `www.kernel.org.`, nhận dạng một host trong phân cấp. FQDN được phân biệt bằng dấu chấm ở cuối, mặc dù trong nhiều ngữ cảnh dấu chấm có thể bị bỏ qua.
- Không có một tổ chức hoặc hệ thống đơn nào quản lý toàn bộ phân cấp. Thay vào đó, có một phân cấp các DNS server, mỗi server quản lý một nhánh (zone) của cây. Thông thường, mỗi zone có một primary master name server, và một hoặc nhiều slave name server, cung cấp backup trong trường hợp primary master name server bị crash.

Implementation DNS server được sử dụng trên Linux là implementation Berkeley Internet Name Domain (BIND) được sử dụng rộng rãi, `named(8)`, được duy trì bởi Internet Systems Consortium (http://www.isc.org/).

**Hình 59-2:** Một tập con của phân cấp DNS

#### **Yêu cầu phân giải đệ quy và lặp**

Các yêu cầu phân giải DNS thuộc hai loại: đệ quy và lặp. Trong một yêu cầu đệ quy, người yêu cầu nhờ server xử lý toàn bộ nhiệm vụ phân giải, bao gồm nhiệm vụ giao tiếp với bất kỳ DNS server nào khác, nếu cần thiết. Khi một ứng dụng trên host cục bộ gọi `getaddrinfo()`, hàm đó thực hiện một yêu cầu đệ quy đến DNS server cục bộ. Nếu DNS server cục bộ không có thông tin để thực hiện phân giải, nó phân giải domain name theo cách lặp.

#### **Top-level domain**

Các nút ngay bên dưới anonymous root tạo thành các top-level domain (TLD). TLD thuộc hai loại: generic và country.

Lịch sử, có bảy generic TLD, hầu hết có thể được coi là quốc tế. Mỗi quốc gia có một country TLD tương ứng (được chuẩn hóa là ISO 3166-1), với tên 2 ký tự. Ví dụ: de (Germany), eu (European Union), nz (New Zealand), us (United States of America).

# **59.9 File /etc/services**

Như đã lưu ý trong Mục 58.6.1, các port number well-known được IANA đăng ký tập trung. Mỗi port này có một tên dịch vụ tương ứng. Vì số dịch vụ được quản lý tập trung và ít biến động hơn địa chỉ IP, một tương đương của DNS server thường không cần thiết. Thay vào đó, port number và tên dịch vụ được ghi trong file `/etc/services`. Các hàm `getaddrinfo()` và `getnameinfo()` sử dụng thông tin trong file này để chuyển đổi tên dịch vụ sang port number và ngược lại.

File `/etc/services` bao gồm các dòng chứa ba cột:

|        | # Service name port/protocol [aliases] |                                 |
|--------|----------------------------------------|---------------------------------|
| echo   | 7/tcp                                  | Echo<br># echo service          |
| echo   | 7/udp                                  | Echo                            |
| ssh    | 22/tcp                                 | # Secure Shell                  |
| ssh    | 22/udp                                 |                                 |
| telnet | 23/tcp                                 | # Telnet                        |
| telnet | 23/udp                                 |                                 |
| smtp   | 25/tcp                                 | # Simple Mail Transfer Protocol |
| smtp   | 25/udp                                 |                                 |
| domain | 53/tcp                                 | # Domain Name Server            |
| domain | 53/udp                                 |                                 |
| http   | 80/tcp                                 | # Hypertext Transfer Protocol   |
| http   | 80/udp                                 |                                 |
| ntp    | 123/tcp                                | # Network Time Protocol         |
| ntp    | 123/udp                                |                                 |
| login  | 513/tcp                                | # rlogin(1)                     |
| who    | 513/udp                                | # rwho(1)                       |
| shell  | 514/tcp                                | # rsh(1)                        |
| syslog | 514/udp                                | # syslog                        |

> File `/etc/services` chỉ là một bản ghi các ánh xạ tên-sang-số. Nó không phải là cơ chế đặt trước: sự xuất hiện của một port number trong `/etc/services` không đảm bảo rằng nó thực sự sẽ khả dụng để binding bởi một dịch vụ cụ thể.

# **59.10 Chuyển Đổi Host và Dịch Vụ Không Phụ Thuộc Giao Thức**

Hàm `getaddrinfo()` chuyển đổi tên host và dịch vụ thành địa chỉ IP và port number. Nó được định nghĩa trong POSIX.1g là người kế thừa (reentrant) của các hàm lỗi thời `gethostbyname()` và `getservbyname()`.

Hàm `getnameinfo()` là nghịch đảo của `getaddrinfo()`. Nó dịch cấu trúc địa chỉ socket (IPv4 hoặc IPv6) thành các chuỗi chứa tên host và tên dịch vụ tương ứng. Hàm này là tương đương (reentrant) của các hàm lỗi thời `gethostbyaddr()` và `getservbyport()`.

# **59.10.1 Hàm getaddrinfo()**

Cho tên host và tên dịch vụ, `getaddrinfo()` trả về một danh sách các cấu trúc địa chỉ socket, mỗi cấu trúc chứa địa chỉ IP và port number.

```
#include <sys/socket.h>
#include <netdb.h>
int getaddrinfo(const char *host, const char *service,
 const struct addrinfo *hints, struct addrinfo **result);
                                    Returns 0 on success, or nonzero on error
```

Là đầu vào, `getaddrinfo()` nhận các đối số `host`, `service` và `hints`. Đối số `host` chứa hostname hoặc chuỗi địa chỉ số, được biểu diễn theo ký hiệu IPv4 dotted-decimal hoặc IPv6 hex-string. Đối số `service` chứa tên dịch vụ hoặc port number thập phân. Đối số `hints` trỏ đến cấu trúc `addrinfo` chỉ định các tiêu chí bổ sung để chọn các cấu trúc địa chỉ socket được trả về qua `result`.

Là đầu ra, `getaddrinfo()` cấp phát động một danh sách liên kết các cấu trúc `addrinfo` và đặt `result` trỏ đến đầu danh sách này. Cấu trúc `addrinfo` có dạng sau:

```
struct addrinfo {
 int ai_flags; /* Input flags (AI_* constants) */
 int ai_family; /* Address family */
 int ai_socktype; /* Type: SOCK_STREAM, SOCK_DGRAM */
 int ai_protocol; /* Socket protocol */
 size_t ai_addrlen; /* Size of structure pointed to by ai_addr */
 char *ai_canonname; /* Canonical name of host */
 struct sockaddr *ai_addr; /* Pointer to socket address structure */
 struct addrinfo *ai_next; /* Next structure in linked list */
};
```

## **Đối số hints**

Đối số `hints` chỉ định các tiêu chí bổ sung để chọn các cấu trúc địa chỉ socket được trả về bởi `getaddrinfo()`. Khi được dùng làm đối số `hints`, chỉ các trường `ai_flags`, `ai_family`, `ai_socktype`, và `ai_protocol` của cấu trúc `addrinfo` có thể được đặt. Các trường khác không được sử dụng và phải được khởi tạo thành 0 hoặc NULL.

Trường `hints.ai_flags` là bit mask sửa đổi hành vi của `getaddrinfo()`. Trường này được tạo thành bằng cách OR không hoặc nhiều giá trị sau:

**AI\_ADDRCONFIG**: Chỉ trả về địa chỉ IPv4 nếu có ít nhất một địa chỉ IPv4 được cấu hình cho hệ thống cục bộ, và chỉ trả về địa chỉ IPv6 nếu có ít nhất một địa chỉ IPv6 được cấu hình.

**AI\_ALL**: Xem mô tả của `AI_V4MAPPED` bên dưới.

**AI\_CANONNAME**: Nếu `host` không phải NULL, trả về con trỏ đến chuỗi kết thúc bằng null chứa canonical name của host.

**AI\_NUMERICHOST**: Buộc diễn giải `host` như chuỗi địa chỉ số.

**AI\_NUMERICSERV**: Diễn giải `service` như số port thập phân.

**AI\_PASSIVE**: Trả về các cấu trúc địa chỉ socket phù hợp cho passive open (tức là listening socket).

**AI\_V4MAPPED**: Nếu `AF_INET6` được chỉ định trong trường `ai_family` của `hints`, thì các cấu trúc địa chỉ IPv4-mapped IPv6 nên được trả về trong `result` nếu không tìm thấy địa chỉ IPv6 phù hợp nào.

## **59.10.2 Giải phóng danh sách addrinfo: freeaddrinfo()**

Hàm `getaddrinfo()` cấp phát động bộ nhớ cho tất cả các cấu trúc được tham chiếu bởi `result`. Do đó, caller phải giải phóng các cấu trúc này khi chúng không còn cần thiết. Hàm `freeaddrinfo()` được cung cấp để thực hiện thuận tiện việc giải phóng này trong một bước duy nhất.

```
#include <sys/socket.h>
#include <netdb.h>
void freeaddrinfo(struct addrinfo *result);
```

# **59.10.3 Chẩn đoán lỗi: gai\_strerror()**

Khi có lỗi, `getaddrinfo()` trả về một trong các mã lỗi khác không được hiển thị trong Bảng 59-1.

| Error constant | Mô tả                                                                        |
|----------------|------------------------------------------------------------------------------|
| EAI_ADDRFAMILY | Không có địa chỉ nào cho host trong hints.ai_family                          |
| EAI_AGAIN      | Lỗi tạm thời trong phân giải tên (thử lại sau)                               |
| EAI_BADFLAGS   | Một flag không hợp lệ được chỉ định trong hints.ai_flags                    |
| EAI_FAIL       | Lỗi không thể phục hồi khi truy cập name server                              |
| EAI_FAMILY     | Address family được chỉ định trong hints.ai_family không được hỗ trợ        |
| EAI_MEMORY     | Lỗi cấp phát bộ nhớ                                                          |
| EAI_NODATA     | Không có địa chỉ nào liên kết với host                                       |
| EAI_NONAME     | Host hoặc dịch vụ không xác định                                             |
| EAI_OVERFLOW   | Tràn buffer đối số                                                            |
| EAI_SERVICE    | Dịch vụ được chỉ định không được hỗ trợ cho hints.ai_socktype                |
| EAI_SOCKTYPE   | hints.ai_socktype được chỉ định không được hỗ trợ                            |
| EAI_SYSTEM     | Lỗi hệ thống trả về trong errno                                              |

```
#include <netdb.h>
const char *gai_strerror(int errcode);
                             Returns pointer to string containing error message
```

## **59.10.4 Hàm getnameinfo()**

Hàm `getnameinfo()` là nghịch đảo của `getaddrinfo()`. Cho một cấu trúc địa chỉ socket (IPv4 hoặc IPv6), nó trả về các chuỗi chứa tên host và tên dịch vụ tương ứng, hoặc các tương đương số nếu không thể phân giải tên.

```
#include <sys/socket.h>
#include <netdb.h>
int getnameinfo(const struct sockaddr *addr, socklen_t addrlen, char *host,
 size_t hostlen, char *service, size_t servlen, int flags);
                                    Returns 0 on success, or nonzero on error
```

Đối số `flags` là bit mask điều khiển hành vi của `getnameinfo()`. Các hằng số sau có thể được OR với nhau để tạo thành bit mask này:

**NI\_DGRAM**: Theo mặc định, `getnameinfo()` trả về tên tương ứng với dịch vụ stream socket (tức là TCP). Flag `NI_DGRAM` buộc trả về tên dịch vụ datagram socket (tức là UDP).

**NI\_NAMEREQD**: Theo mặc định, nếu hostname không thể phân giải, một chuỗi địa chỉ số được trả về trong `host`. Nếu flag `NI_NAMEREQD` được chỉ định, thay vào đó trả về lỗi (`EAI_NONAME`).

**NI\_NOFQDN**: Theo mặc định, fully qualified domain name của host được trả về. Chỉ định flag `NI_NOFQDN` khiến chỉ phần đầu tiên (tức là phần hostname) của tên được trả về, nếu đây là host trên mạng cục bộ.

**NI\_NUMERICHOST**: Buộc trả về chuỗi địa chỉ số trong `host`.

**NI\_NUMERICSERV**: Buộc trả về chuỗi port number thập phân trong `service`.

# **59.11 Ví Dụ Client-Server (Stream Socket)**

Bây giờ chúng ta có đủ thông tin để xem xét một ứng dụng client-server đơn giản sử dụng TCP socket. Nhiệm vụ được thực hiện bởi ứng dụng này giống như được thực hiện bởi ứng dụng FIFO client-server được trình bày trong Mục 44.8: phân bổ số thứ tự duy nhất (hoặc dải số thứ tự) cho client.

#### **File header chung**

Cả server và client đều bao gồm file header được hiển thị trong Listing 59-5.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_seqnum.h
#include <netinet/in.h>
#include <sys/socket.h>
#include <signal.h>
#include "read_line.h" /* Declaration of readLine() */
#include "tlpi_hdr.h"
#define PORT_NUM "50000" /* Port number for server */
#define INT_LEN 30 /* Size of string able to hold largest
 integer (including terminating '\n') */
–––––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_seqnum.h
```

**Chương trình server**

Chương trình server được hiển thị trong Listing 59-6. Server thực hiện các bước sau:

- Khởi tạo sequence number của server về 1 hoặc về giá trị được cung cấp trong đối số dòng lệnh tùy chọn.
- Bỏ qua signal `SIGPIPE`. Điều này ngăn server nhận signal `SIGPIPE` nếu nó cố gắng ghi vào socket mà peer của nó đã đóng; thay vào đó, `write()` thất bại với lỗi `EPIPE`.
- Gọi `getaddrinfo()` để lấy một tập hợp các cấu trúc địa chỉ socket cho TCP socket sử dụng port number `PORT_NUM`.
- Đặt tùy chọn `SO_REUSEADDR` cho socket. Chúng ta hoãn thảo luận về tùy chọn này đến Mục 61.10.
- Đánh dấu socket là listening socket.
- Bắt đầu một vòng lặp `for` vô hạn phục vụ client theo cách lặp.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_seqnum_sv.c
  #define _BSD_SOURCE
  #include <netdb.h>
  #include "is_seqnum.h"
  #define BACKLOG 50
  int
  main(int argc, char *argv[])
  {
   uint32_t seqNum;
   char reqLenStr[INT_LEN];
   char seqNumStr[INT_LEN];
   struct sockaddr_storage claddr;
   int lfd, cfd, optval, reqLen;
   socklen_t addrlen;
   struct addrinfo hints;
   struct addrinfo *result, *rp;
  #define ADDRSTRLEN (NI_MAXHOST + NI_MAXSERV + 10)
   char addrStr[ADDRSTRLEN];
   char host[NI_MAXHOST];
   char service[NI_MAXSERV];
   if (argc > 1 && strcmp(argv[1], "--help") == 0)
   usageErr("%s [init-seq-num]\n", argv[0]);
   seqNum = (argc > 1) ? getInt(argv[1], 0, "init-seq-num") : 0;
   if (signal(SIGPIPE, SIG_IGN) == SIG_ERR)
   errExit("signal");
   memset(&hints, 0, sizeof(struct addrinfo));
   hints.ai_canonname = NULL;
   hints.ai_addr = NULL;
   hints.ai_next = NULL;
   hints.ai_socktype = SOCK_STREAM;
   hints.ai_family = AF_UNSPEC;
   hints.ai_flags = AI_PASSIVE | AI_NUMERICSERV;
   if (getaddrinfo(NULL, PORT_NUM, &hints, &result) != 0)
   errExit("getaddrinfo");
   optval = 1;
   for (rp = result; rp != NULL; rp = rp->ai_next) {
   lfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
   if (lfd == -1)
   continue;
   if (setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval))
   == -1)
   errExit("setsockopt");
   if (bind(lfd, rp->ai_addr, rp->ai_addrlen) == 0)
   break;
   close(lfd);
   }
   if (rp == NULL)
   fatal("Could not bind socket to any address");
   if (listen(lfd, BACKLOG) == -1)
   errExit("listen");
   freeaddrinfo(result);
   for (;;) {
   addrlen = sizeof(struct sockaddr_storage);
   cfd = accept(lfd, (struct sockaddr *) &claddr, &addrlen);
   if (cfd == -1) {
   errMsg("accept");
   continue;
   }
   if (getnameinfo((struct sockaddr *) &claddr, addrlen,
   host, NI_MAXHOST, service, NI_MAXSERV, 0) == 0)
   snprintf(addrStr, ADDRSTRLEN, "(%s, %s)", host, service);
   else
   snprintf(addrStr, ADDRSTRLEN, "(?UNKNOWN?)");
   printf("Connection from %s\n", addrStr);
   if (readLine(cfd, reqLenStr, INT_LEN) <= 0) {
   close(cfd);
   continue;
   }
   reqLen = atoi(reqLenStr);
   if (reqLen <= 0) {
   close(cfd);
   continue;
   }
   snprintf(seqNumStr, INT_LEN, "%d\n", seqNum);
   if (write(cfd, &seqNumStr, strlen(seqNumStr)) != strlen(seqNumStr))
   fprintf(stderr, "Error on write");
   seqNum += reqLen;
   if (close(cfd) == -1)
   errMsg("close");
   }
  }
  –––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_seqnum_sv.c
```

**Chương trình client**

Chương trình client được hiển thị trong Listing 59-7. Chương trình này nhận hai đối số. Đối số đầu tiên, là tên host mà server đang chạy, là bắt buộc. Đối số tùy chọn thứ hai là độ dài của chuỗi mong muốn bởi client. Độ dài mặc định là 1.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_seqnum_cl.c
  #include <netdb.h>
  #include "is_seqnum.h"
  int
  main(int argc, char *argv[])
  {
   char *reqLenStr;
   char seqNumStr[INT_LEN];
   int cfd;
   ssize_t numRead;
   struct addrinfo hints;
   struct addrinfo *result, *rp;
   if (argc < 2 || strcmp(argv[1], "--help") == 0)
   usageErr("%s server-host [sequence-len]\n", argv[0]);
   memset(&hints, 0, sizeof(struct addrinfo));
   hints.ai_canonname = NULL;
   hints.ai_addr = NULL;
   hints.ai_next = NULL;
   hints.ai_family = AF_UNSPEC;
   hints.ai_socktype = SOCK_STREAM;
   hints.ai_flags = AI_NUMERICSERV;
   if (getaddrinfo(argv[1], PORT_NUM, &hints, &result) != 0)
   errExit("getaddrinfo");
   for (rp = result; rp != NULL; rp = rp->ai_next) {
   cfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
   if (cfd == -1)
   continue;
   if (connect(cfd, rp->ai_addr, rp->ai_addrlen) != -1)
   break;
   close(cfd);
   }
   if (rp == NULL)
   fatal("Could not connect socket to any address");
   freeaddrinfo(result);
   reqLenStr = (argc > 2) ? argv[2] : "1";
   if (write(cfd, reqLenStr, strlen(reqLenStr)) != strlen(reqLenStr))
   fatal("Partial/failed write (reqLenStr)");
   if (write(cfd, "\n", 1) != 1)
   fatal("Partial/failed write (newline)");
   numRead = readLine(cfd, seqNumStr, INT_LEN);
   if (numRead == -1)
   errExit("readLine");
   if (numRead == 0)
   fatal("Unexpected EOF from server");
   printf("Sequence number: %s", seqNumStr);
   exit(EXIT_SUCCESS);
  }
  –––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/is_seqnum_cl.c
```

# **59.12 Thư Viện Internet Domain Socket**

Trong phần này, chúng ta sử dụng các hàm được trình bày trong Mục 59.10 để implement một thư viện các hàm thực hiện các nhiệm vụ thường xuyên cần thiết cho Internet domain socket. Vì các hàm này sử dụng `getaddrinfo()` và `getnameinfo()` không phụ thuộc giao thức, chúng có thể được dùng với cả IPv4 và IPv6.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/inet_sockets.h
#ifndef INET_SOCKETS_H
#define INET_SOCKETS_H
#include <sys/socket.h>
#include <netdb.h>
int inetConnect(const char *host, const char *service, int type);
int inetListen(const char *service, int backlog, socklen_t *addrlen);
int inetBind(const char *service, int type, socklen_t *addrlen);
char *inetAddressStr(const struct sockaddr *addr, socklen_t addrlen,
 char *addrStr, int addrStrLen);
#define IS_ADDR_STR_LEN 4096
#endif
–––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/inet_sockets.h
```

Hàm `inetConnect()` tạo socket với kiểu socket đã cho, và kết nối nó với địa chỉ được chỉ định bởi `host` và `service`. Hàm này được thiết kế cho TCP hoặc UDP client cần kết nối socket của họ vào server socket.

Hàm `inetListen()` tạo listening stream (`SOCK_STREAM`) socket được bind vào địa chỉ wildcard IP trên TCP port được chỉ định bởi `service`. Hàm này được thiết kế để sử dụng bởi TCP server.

Hàm `inetBind()` tạo socket của kiểu đã cho, được bind vào địa chỉ wildcard IP trên port được chỉ định bởi `service` và `type`. Hàm này được thiết kế (chủ yếu) cho UDP server và client để tạo socket được bind vào một địa chỉ cụ thể.

Hàm `inetAddressStr()` chuyển đổi địa chỉ Internet socket sang dạng có thể in.

```
–––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/inet_sockets.c
#define _BSD_SOURCE
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include "inet_sockets.h"
#include "tlpi_hdr.h"
int
inetConnect(const char *host, const char *service, int type)
{
 struct addrinfo hints;
 struct addrinfo *result, *rp;
 int sfd, s;
 memset(&hints, 0, sizeof(struct addrinfo));
 hints.ai_canonname = NULL;
 hints.ai_addr = NULL;
 hints.ai_next = NULL;
 hints.ai_family = AF_UNSPEC;
 hints.ai_socktype = type;
 s = getaddrinfo(host, service, &hints, &result);
 if (s != 0) {
 errno = ENOSYS;
 return -1;
 }
 for (rp = result; rp != NULL; rp = rp->ai_next) {
 sfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
 if (sfd == -1)
 continue;
 if (connect(sfd, rp->ai_addr, rp->ai_addrlen) != -1)
 break;
 close(sfd);
 }
 freeaddrinfo(result);
 return (rp == NULL) ? -1 : sfd;
}
```

```
static int
inetPassiveSocket(const char *service, int type, socklen_t *addrlen,
 Boolean doListen, int backlog)
{
 struct addrinfo hints;
 struct addrinfo *result, *rp;
 int sfd, optval, s;
 memset(&hints, 0, sizeof(struct addrinfo));
 hints.ai_canonname = NULL;
 hints.ai_addr = NULL;
 hints.ai_next = NULL;
 hints.ai_socktype = type;
 hints.ai_family = AF_UNSPEC;
 hints.ai_flags = AI_PASSIVE;
 s = getaddrinfo(NULL, service, &hints, &result);
 if (s != 0)
 return -1;
 optval = 1;
 for (rp = result; rp != NULL; rp = rp->ai_next) {
 sfd = socket(rp->ai_family, rp->ai_socktype, rp->ai_protocol);
 if (sfd == -1)
 continue;
 if (doListen) {
 if (setsockopt(sfd, SOL_SOCKET, SO_REUSEADDR, &optval,
 sizeof(optval)) == -1) {
 close(sfd);
 freeaddrinfo(result);
 return -1;
 }
 }
 if (bind(sfd, rp->ai_addr, rp->ai_addrlen) == 0)
 break;
 close(sfd);
 }
 if (rp != NULL && doListen) {
 if (listen(sfd, backlog) == -1) {
 freeaddrinfo(result);
 return -1;
 }
 }
 if (rp != NULL && addrlen != NULL)
 *addrlen = rp->ai_addrlen;
 freeaddrinfo(result);
 return (rp == NULL) ? -1 : sfd;
}
int
inetListen(const char *service, int backlog, socklen_t *addrlen)
{
 return inetPassiveSocket(service, SOCK_STREAM, addrlen, TRUE, backlog);
}
int
inetBind(const char *service, int type, socklen_t *addrlen)
{
 return inetPassiveSocket(service, type, addrlen, FALSE, 0);
}
char *
inetAddressStr(const struct sockaddr *addr, socklen_t addrlen,
 char *addrStr, int addrStrLen)
{
 char host[NI_MAXHOST], service[NI_MAXSERV];
 if (getnameinfo(addr, addrlen, host, NI_MAXHOST,
 service, NI_MAXSERV, NI_NUMERICSERV) == 0)
 snprintf(addrStr, addrStrLen, "(%s, %s)", host, service);
 else
 snprintf(addrStr, addrStrLen, "(?UNKNOWN?)");
 addrStr[addrStrLen - 1] = '\0';
 return addrStr;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– sockets/inet_sockets.c
```

# **59.13 Các API Lỗi Thời cho Chuyển Đổi Host và Dịch Vụ**

Trong các phần sau, chúng ta mô tả các hàm cũ, hiện đã lỗi thời để chuyển đổi tên host và tên dịch vụ sang và từ các dạng nhị phân và presentation. Mặc dù các chương trình mới nên thực hiện các chuyển đổi này bằng các hàm hiện đại được mô tả trước đó trong chương này, nhưng kiến thức về các hàm lỗi thời là hữu ích vì chúng ta có thể gặp chúng trong code cũ hơn.

# **59.13.1 Các Hàm inet\_aton() và inet\_ntoa()**

Các hàm `inet_aton()` và `inet_ntoa()` chuyển đổi địa chỉ IPv4 giữa ký hiệu dotted-decimal và dạng nhị phân (theo network byte order). Các hàm này hiện đã bị lỗi thời bởi `inet_pton()` và `inet_ntop()`.

```
#include <arpa/inet.h>
int inet_aton(const char *str, struct in_addr *addr);
     Returns 1 (true) if str is a valid dotted-decimal address, or 0 (false) on error
```

```
#include <arpa/inet.h>
char *inet_ntoa(struct in_addr addr);
                                           Returns pointer to (statically allocated)
                                             dotted-decimal string version of addr
```

Vì chuỗi được trả về bởi `inet_ntoa()` được cấp phát tĩnh, nó bị ghi đè bởi các lời gọi liên tiếp.

# **59.13.2 Các Hàm gethostbyname() và gethostbyaddr()**

Các hàm `gethostbyname()` và `gethostbyaddr()` cho phép chuyển đổi giữa hostname và địa chỉ IP. Các hàm này hiện đã bị lỗi thời bởi `getaddrinfo()` và `getnameinfo()`.

```
#include <netdb.h>
extern int h_errno;
struct hostent *gethostbyname(const char *name);
struct hostent *gethostbyaddr(const char *addr, socklen_t len, int type);
                     Both return pointer to (statically allocated) hostent structure
                                                       on success, or NULL on error
```

Cấu trúc `hostent` có dạng sau:

```
struct hostent {
 char *h_name; /* Official (canonical) name of host */
 char **h_aliases; /* NULL-terminated array of pointers
 to alias strings */
 int h_addrtype; /* Address type (AF_INET or AF_INET6) */
 int h_length; /* Length (in bytes) of addresses pointed
 to by h_addr_list (4 bytes for AF_INET,
 16 bytes for AF_INET6) */
 char **h_addr_list; /* NULL-terminated array of pointers to
 host IP addresses (in_addr or in6_addr
 structures) in network byte order */
};
#define h_addr h_addr_list[0]
```

```
–––––––––––––––––––––––––––––––––––––––––– sockets/t_gethostbyname.c
#define _BSD_SOURCE
#include <netdb.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 struct hostent *h;
 char **pp;
 char str[INET6_ADDRSTRLEN];
 for (argv++; *argv != NULL; argv++) {
 h = gethostbyname(*argv);
 if (h == NULL) {
 fprintf(stderr, "gethostbyname() failed for '%s': %s\n",
 *argv, hstrerror(h_errno));
 continue;
 }
 printf("Canonical name: %s\n", h->h_name);
 printf(" alias(es): ");
 for (pp = h->h_aliases; *pp != NULL; pp++)
 printf(" %s", *pp);
 printf("\n");
 printf(" address type: %s\n",
 (h->h_addrtype == AF_INET) ? "AF_INET" :
 (h->h_addrtype == AF_INET6) ? "AF_INET6" : "???");
 if (h->h_addrtype == AF_INET || h->h_addrtype == AF_INET6) {
 printf(" address(es): ");
 for (pp = h->h_addr_list; *pp != NULL; pp++)
 printf(" %s", inet_ntop(h->h_addrtype, *pp,
 str, INET6_ADDRSTRLEN));
 printf("\n");
 }
 }
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––– sockets/t_gethostbyname.c
```

## **59.13.3 Các Hàm getservbyname() và getservbyport()**

Các hàm `getservbyname()` và `getservbyport()` lấy các bản ghi từ file `/etc/services`. Các hàm này hiện đã bị lỗi thời bởi `getaddrinfo()` và `getnameinfo()`.

```
#include <netdb.h>
struct servent *getservbyname(const char *name, const char *proto);
struct servent *getservbyport(int port, const char *proto);
                   Both return pointer to a (statically allocated) servent structure
                                        on success, or NULL on not found or error
```

```
struct servent {
 char *s_name; /* Official service name */
 char **s_aliases; /* Pointers to aliases (NULL-terminated) */
 int s_port; /* Port number (in network byte order) */
 char *s_proto; /* Protocol */
};
```

# **59.14 UNIX Versus Internet Domain Socket**

Khi viết các ứng dụng giao tiếp qua mạng, chúng ta nhất thiết phải dùng Internet domain socket. Tuy nhiên, khi dùng socket để giao tiếp giữa các ứng dụng trên cùng một hệ thống, chúng ta có thể chọn dùng Internet hoặc UNIX domain socket.

Viết ứng dụng chỉ sử dụng Internet domain socket thường là cách đơn giản nhất, vì nó sẽ hoạt động trên cả một host đơn lẻ và qua mạng. Tuy nhiên, có một số lý do tại sao chúng ta có thể chọn dùng UNIX domain socket:

- Trên một số implementation, UNIX domain socket nhanh hơn Internet domain socket.
- Chúng ta có thể dùng quyền thư mục (và, trên Linux, quyền file) để kiểm soát truy cập vào UNIX domain socket, để chỉ các ứng dụng có user ID hoặc group ID được chỉ định mới có thể kết nối đến listening stream socket hoặc gửi datagram đến datagram socket. Điều này cung cấp một phương pháp đơn giản để xác thực client.
- Sử dụng UNIX domain socket, chúng ta có thể truyền open file descriptor và thông tin xác thực sender, như được tóm tắt trong Mục 61.13.3.

# **59.15 Tài Liệu Thêm**

Có nhiều tài nguyên in và trực tuyến về TCP/IP và sockets API:

- Cuốn sách chính về lập trình mạng với sockets API là [Stevens et al., 2004].
- [Stevens, 1994] và [Wright & Stevens, 1995] mô tả TCP/IP chi tiết.
- [Tanenbaum, 2002] cung cấp nền tảng chung về mạng máy tính.
- [Herbert, 2004] mô tả các chi tiết về TCP/IP stack của Linux 2.6.
- Tài liệu thư viện GNU C (trực tuyến tại http://www.gnu.org/) có thảo luận rộng rãi về sockets API.
- Linux-specific information có thể tìm thấy trong các trang manual sau: `socket(7)`, `ip(7)`, `raw(7)`, `tcp(7)`, `udp(7)`, và `packet(7)`.

# **59.16 Tóm Tắt**

Internet domain socket cho phép các ứng dụng trên các host khác nhau giao tiếp qua mạng TCP/IP. Địa chỉ Internet domain socket bao gồm địa chỉ IP và port number. Trong IPv4, địa chỉ IP là số 32-bit; trong IPv6, là số 128-bit. Internet domain datagram socket hoạt động trên UDP. Internet domain stream socket hoạt động trên TCP.

Các kiến trúc máy tính khác nhau sử dụng các quy ước khác nhau để biểu diễn các kiểu dữ liệu. Điều này có nghĩa là chúng ta cần sử dụng một số biểu diễn độc lập kiến trúc khi truyền dữ liệu giữa các máy không đồng nhất được kết nối qua mạng.

Chúng ta đã xem xét một loạt các hàm có thể được dùng để chuyển đổi giữa các biểu diễn chuỗi (số) của địa chỉ IP và các tương đương nhị phân của chúng. Tuy nhiên, nói chung là tốt hơn khi dùng tên host và dịch vụ thay vì số. Hàm hiện đại để dịch tên host và dịch vụ thành địa chỉ socket là `getaddrinfo()`, nhưng thường thấy các hàm lịch sử `gethostbyname()` và `getservbyname()` trong code hiện có.

Việc xem xét các chuyển đổi hostname dẫn chúng ta vào thảo luận về DNS, hệ thống implement cơ sở dữ liệu phân tán cho dịch vụ thư mục phân cấp.

# **59.17 Bài Tập**

**59-1.** Khi đọc lượng lớn dữ liệu, hàm `readLine()` được hiển thị trong Listing 59-1 không hiệu quả, vì một system call được yêu cầu để đọc mỗi ký tự. Một interface hiệu quả hơn sẽ đọc một khối ký tự vào buffer và trích xuất từng dòng một từ buffer này. Implement hai hàm như vậy. Sửa đổi các chương trình trong Listing 59-6 và Listing 59-7 để sử dụng các hàm này.

- **59-2.** Sửa đổi các chương trình trong Listing 59-6 và Listing 59-7 để sử dụng các hàm `inetListen()` và `inetConnect()` được cung cấp trong Listing 59-9.
- **59-3.** Viết một thư viện UNIX domain socket với API tương tự như thư viện Internet domain socket được hiển thị trong Mục 59.12. Viết lại các chương trình trong Listing 57-3 và Listing 57-4 để sử dụng thư viện này.
- **59-4.** Viết một network server lưu trữ các cặp name-value. Server nên cho phép client thêm, xóa, sửa đổi và truy xuất tên. Viết một hoặc nhiều chương trình client để kiểm tra server. Tùy chọn, implement một số loại cơ chế bảo mật cho phép chỉ client tạo tên mới có thể xóa nó hoặc sửa đổi giá trị liên quan đến nó.
- **59-5.** Giả sử chúng ta tạo hai Internet domain datagram socket, được bind vào các địa chỉ cụ thể, và kết nối socket đầu tiên với socket thứ hai. Điều gì xảy ra nếu chúng ta tạo socket datagram thứ ba và cố gắng gửi (`sendto()`) một datagram qua socket đó đến socket đầu tiên? Viết một chương trình để xác định câu trả lời.
