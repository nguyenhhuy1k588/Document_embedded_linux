## Chương 52
# POSIX MESSAGE QUEUE

Chương này mô tả POSIX message queue, cho phép các process trao đổi dữ liệu dưới dạng message. POSIX message queue tương tự như các System V counterpart của chúng, ở chỗ dữ liệu được trao đổi theo đơn vị toàn bộ message. Tuy nhiên, cũng có một số khác biệt đáng chú ý:

-  POSIX message queue được đếm tham chiếu. Một queue được đánh dấu để xóa chỉ bị xóa sau khi tất cả các process đang sử dụng nó đã đóng nó.
-  Mỗi System V message có một integer type, và các message có thể được chọn theo nhiều cách khác nhau bằng `msgrcv()`. Ngược lại, POSIX message có một độ ưu tiên liên kết, và các message luôn được xếp hàng (và do đó nhận) theo thứ tự độ ưu tiên nghiêm ngặt.
-  POSIX message queue cung cấp một tính năng cho phép một process được thông báo không đồng bộ khi một message có sẵn trong queue.

POSIX message queue là một bổ sung tương đối gần đây cho Linux. Hỗ trợ triển khai cần thiết đã được thêm vào trong kernel 2.6.6 (ngoài ra, cần có glibc 2.3.4 hoặc phiên bản mới hơn).

> Hỗ trợ POSIX message queue là một thành phần kernel tùy chọn được cấu hình thông qua tùy chọn `CONFIG_POSIX_MQUEUE`.

## **52.1 Tổng quan**

Các hàm chính trong POSIX message queue API là:

-  Hàm `mq_open()` tạo một message queue mới hoặc mở một queue hiện có, trả về một message queue descriptor để sử dụng trong các lời gọi sau.
-  Hàm `mq_send()` ghi một message vào queue.
-  Hàm `mq_receive()` đọc một message từ queue.
-  Hàm `mq_close()` đóng một message queue mà process đã mở trước đó.
-  Hàm `mq_unlink()` xóa tên message queue và đánh dấu queue để xóa khi tất cả các process đã đóng nó.

Các hàm trên đều phục vụ các mục đích khá rõ ràng. Ngoài ra, một vài tính năng đặc trưng cho POSIX message queue API:

-  Mỗi message queue có một tập hợp các thuộc tính liên kết. Một số thuộc tính này có thể được thiết lập khi queue được tạo hoặc mở bằng `mq_open()`. Hai hàm được cung cấp để lấy và thay đổi thuộc tính queue: `mq_getattr()` và `mq_setattr()`.
-  Hàm `mq_notify()` cho phép một process đăng ký để nhận thông báo message từ một queue. Sau khi đăng ký, process được thông báo về sự có sẵn của message bằng cách gửi signal hoặc bằng cách gọi một hàm trong một thread riêng biệt.

## **52.2 Mở, Đóng và Unlink một Message Queue**

Trong phần này, chúng ta xem xét các hàm được sử dụng để mở, đóng và xóa message queue.

#### **Mở một message queue**

Hàm `mq_open()` tạo một message queue mới hoặc mở một queue hiện có.

```
#include <fcntl.h> /* Defines O_* constants */
#include <sys/stat.h> /* Defines mode constants */
#include <mqueue.h>
mqd_t mq_open(const char *name, int oflag, ...
 /* mode_t mode, struct mq_attr *attr */);
         Returns a message queue descriptor on success, or (mqd_t) –1 on error
```

Đối số `name` xác định message queue, và được chỉ định theo các quy tắc được đưa ra trong Mục 51.1.

Đối số `oflag` là một bit mask kiểm soát các khía cạnh khác nhau của hoạt động của `mq_open()`. Các giá trị có thể được bao gồm trong mask này được tóm tắt trong Bảng 52-1.

**Bảng 52-1:** Giá trị bit cho đối số oflag của mq\_open()

| Flag       | Mô tả                                              |
|------------|----------------------------------------------------|
| `O_CREAT`  | Tạo queue nếu nó chưa tồn tại                      |
| `O_EXCL`   | Với `O_CREAT`, tạo queue độc quyền                 |
| `O_RDONLY` | Mở chỉ đọc                                         |
| `O_WRONLY` | Mở chỉ ghi                                         |
| `O_RDWR`   | Mở để đọc và ghi                                   |
| `O_NONBLOCK` | Mở ở chế độ nonblocking                          |

Một trong những mục đích của đối số `oflag` là xác định xem chúng ta đang mở một queue hiện có hay tạo và mở một queue mới. Nếu `oflag` không bao gồm `O_CREAT`, chúng ta đang mở một queue hiện có. Nếu `oflag` bao gồm `O_CREAT`, một queue mới, rỗng được tạo nếu chưa có queue nào với tên đã cho tồn tại. Nếu `oflag` chỉ định cả `O_CREAT` và `O_EXCL`, và một queue với tên đã cho đã tồn tại, thì `mq_open()` thất bại.

Đối số `oflag` cũng chỉ ra loại truy cập mà process đang gọi sẽ thực hiện đến message queue, bằng cách chỉ định chính xác một trong các giá trị `O_RDONLY`, `O_WRONLY` hoặc `O_RDWR`.

Giá trị flag còn lại, `O_NONBLOCK`, khiến queue được mở ở chế độ nonblocking. Nếu một lời gọi tiếp theo đến `mq_receive()` hoặc `mq_send()` không thể được thực hiện mà không blocking, lời gọi sẽ thất bại ngay lập tức với lỗi `EAGAIN`.

Nếu `mq_open()` đang được sử dụng để mở một message queue hiện có, lời gọi chỉ yêu cầu hai đối số. Tuy nhiên, nếu `O_CREAT` được chỉ định trong flags, hai đối số bổ sung là bắt buộc: `mode` và `attr`. (Nếu queue được chỉ định bởi `name` đã tồn tại, hai đối số này bị bỏ qua.) Các đối số này được sử dụng như sau:

-  Đối số `mode` là một bit mask chỉ định các quyền cần đặt trên message queue mới. Các giá trị bit có thể được chỉ định giống như đối với file (Bảng 15-4, trang 295), và, giống như `open()`, giá trị trong `mode` được AND với process umask (Mục 15.4.6). Để đọc từ queue (`mq_receive()`), quyền đọc phải được cấp cho class người dùng tương ứng; để ghi vào queue (`mq_send()`), cần có quyền ghi.
-  Đối số `attr` là một cấu trúc `mq_attr` chỉ định các thuộc tính cho message queue mới. Nếu `attr` là `NULL`, queue được tạo với các thuộc tính mặc định được định nghĩa bởi cài đặt. Chúng ta mô tả cấu trúc `mq_attr` trong Mục 52.4.

Khi hoàn thành thành công, `mq_open()` trả về một message queue descriptor, một giá trị kiểu `mqd_t`, được sử dụng trong các lời gọi tiếp theo để tham chiếu đến message queue đang mở này. Quy định duy nhất mà SUSv3 đưa ra về kiểu dữ liệu này là nó không thể là một mảng; nghĩa là nó được đảm bảo là một kiểu có thể được sử dụng trong một câu lệnh gán hoặc được truyền theo giá trị như một đối số hàm. (Trên Linux, `mqd_t` là một `int`, nhưng, ví dụ, trên Solaris nó được định nghĩa là `void *`.)

Một ví dụ về việc sử dụng `mq_open()` được cung cấp trong Listing 52-2.

## **Ảnh hưởng của fork(), exec() và kết thúc process đến message queue descriptor**

Trong quá trình `fork()`, process con nhận các bản sao của message queue descriptor của process cha, và các descriptor này tham chiếu đến cùng các open message queue description. (Chúng ta giải thích message queue description trong Mục 52.3.) Process con không kế thừa bất kỳ đăng ký message notification nào của process cha.

Khi một process thực hiện `exec()` hoặc kết thúc, tất cả message queue descriptor đang mở của nó sẽ bị đóng. Là hệ quả của việc đóng message queue descriptor của nó, tất cả các đăng ký message notification của process trên các queue tương ứng sẽ bị hủy đăng ký.

#### **Đóng một message queue**

Hàm `mq_close()` đóng message queue descriptor `mqdes`.

```
#include <mqueue.h>
int mq_close(mqd_t mqdes);
                                             Returns 0 on success, or –1 on error
```

Nếu process đang gọi đã đăng ký thông qua `mqdes` để nhận message notification từ queue (Mục 52.6), thì đăng ký notification sẽ tự động bị xóa, và một process khác có thể đăng ký nhận message notification từ queue sau đó.

Một message queue descriptor tự động được đóng khi process kết thúc hoặc gọi `exec()`. Giống như các file descriptor, chúng ta nên đóng rõ ràng các message queue descriptor không còn cần thiết, để tránh process hết message queue descriptor.

Giống như `close()` đối với file, việc đóng một message queue không xóa nó. Để làm điều đó, chúng ta cần `mq_unlink()`, là message queue analog của `unlink()`.

#### **Xóa một message queue**

Hàm `mq_unlink()` xóa message queue được xác định bởi `name`, và đánh dấu queue để bị hủy một khi tất cả các process ngừng sử dụng nó (điều này có thể có nghĩa là ngay lập tức, nếu tất cả các process đã có queue mở đã đóng nó rồi).

```
#include <mqueue.h>
int mq_unlink(const char *name);
                                            Returns 0 on success, or –1 on error
```

Listing 52-1 minh họa việc sử dụng `mq_unlink()`.

**Listing 52-1:** Sử dụng mq\_unlink() để unlink một POSIX message queue

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_unlink.c
#include <mqueue.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
```

```
 if (argc != 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s mq-name\n", argv[0]);
 if (mq_unlink(argv[1]) == -1)
 errExit("mq_unlink");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_unlink.c
```

## **52.3 Mối quan hệ giữa Descriptor và Message Queue**

Mối quan hệ giữa một message queue descriptor và một open message queue tương tự như mối quan hệ giữa một file descriptor và một open file (Hình 5-2, trang 95). Một message queue descriptor là một per-process handle tham chiếu đến một mục trong bảng system-wide của các open message queue description, và mục này lần lượt tham chiếu đến một đối tượng message queue. Mối quan hệ này được minh họa trong Hình 52-1.

> Trên Linux, POSIX message queue được triển khai như các i-node trong một file system ảo, và message queue descriptor cũng như open message queue description được triển khai tương ứng như file descriptor và open file description. Tuy nhiên, đây là các chi tiết triển khai không được SUSv3 yêu cầu và không đúng trên một số cài đặt UNIX khác. Dù vậy, chúng ta sẽ quay lại điểm này trong Mục 52.7, vì Linux cung cấp một số tính năng không chuẩn được tạo ra có thể bởi việc triển khai này.

![](_page_10_Figure_4.jpeg)

**Hình 52-1:** Mối quan hệ giữa các cấu trúc dữ liệu kernel cho POSIX message queue

Hình 52-1 giúp làm rõ một số chi tiết về việc sử dụng message queue descriptor (tất cả đều tương tự như việc sử dụng file descriptor):

-  Một open message queue description có một tập hợp các flag liên kết. SUSv3 chỉ chỉ định một flag như vậy, `O_NONBLOCK`, xác định xem I/O có phải nonblocking không.
-  Hai process có thể giữ message queue descriptor (descriptor x trong sơ đồ) tham chiếu đến cùng open message queue description. Điều này có thể xảy ra vì một process mở một message queue và sau đó gọi `fork()`. Các descriptor này chia sẻ trạng thái của flag `O_NONBLOCK`.
-  Hai process có thể giữ các open message queue descriptor tham chiếu đến các message queue description khác nhau tham chiếu đến cùng một message queue (ví dụ: descriptor z trong process A và descriptor y trong process B đều tham chiếu đến `/mq-r`). Điều này xảy ra vì hai process mỗi process đã sử dụng `mq_open()` để mở cùng một queue.

## **52.4 Thuộc tính Message Queue**

Các hàm `mq_open()`, `mq_getattr()` và `mq_setattr()` đều cho phép một đối số là pointer đến cấu trúc `mq_attr`. Cấu trúc này được định nghĩa trong `<mqueue.h>` như sau:

```
struct mq_attr {
 long mq_flags; /* Message queue description flags: 0 or
 O_NONBLOCK [mq_getattr(), mq_setattr()] */
 long mq_maxmsg; /* Maximum number of messages on queue
 [mq_open(), mq_getattr()] */
 long mq_msgsize; /* Maximum message size (in bytes)
 [mq_open(), mq_getattr()] */
 long mq_curmsgs; /* Number of messages currently in queue
 [mq_getattr()] */
};
```

Trước khi xem xét cấu trúc `mq_attr` chi tiết, cần lưu ý những điểm sau:

-  Chỉ một số trường được sử dụng bởi mỗi hàm trong số ba hàm. Các trường được sử dụng bởi mỗi hàm được chỉ ra trong các comment đi kèm với định nghĩa cấu trúc ở trên.
-  Cấu trúc chứa thông tin về open message queue description (`mq_flags`) liên kết với message descriptor và thông tin về queue được tham chiếu bởi descriptor đó (`mq_maxmsg`, `mq_msgsize`, `mq_curmsgs`).
-  Một số trường chứa thông tin được cố định tại thời điểm queue được tạo bằng `mq_open()` (`mq_maxmsg` và `mq_msgsize`); các trường khác trả về thông tin về trạng thái hiện tại của message queue description (`mq_flags`) hoặc message queue (`mq_curmsgs`).

#### **Thiết lập thuộc tính message queue khi tạo queue**

Khi chúng ta tạo một message queue bằng `mq_open()`, các trường `mq_attr` sau xác định các thuộc tính của queue:

 Trường `mq_maxmsg` xác định giới hạn về số lượng message có thể được đặt vào queue bằng `mq_send()`. Giá trị này phải lớn hơn 0.

 Trường `mq_msgsize` xác định giới hạn trên về kích thước của mỗi message có thể được đặt vào queue. Giá trị này phải lớn hơn 0.

Cùng nhau, hai giá trị này cho phép kernel xác định lượng bộ nhớ tối đa mà message queue này có thể yêu cầu.

Các thuộc tính `mq_maxmsg` và `mq_msgsize` được cố định khi một message queue được tạo; chúng không thể thay đổi sau đó. Trong Mục 52.8, chúng ta mô tả hai file `/proc` đặt giới hạn trên toàn hệ thống đối với các giá trị có thể được chỉ định cho thuộc tính `mq_maxmsg` và `mq_msgsize`.

Chương trình trong Listing 52-2 cung cấp một giao diện dòng lệnh cho hàm `mq_open()` và cho thấy cách cấu trúc `mq_attr` được sử dụng với `mq_open()`.

Hai tùy chọn dòng lệnh cho phép thuộc tính message queue được chỉ định: `-m` cho `mq_maxmsg` và `-s` cho `mq_msgsize`. Nếu một trong các tùy chọn này được cung cấp, một đối số `attrp` không phải `NULL` được truyền cho `mq_open()`. Một số giá trị mặc định được gán cho các trường của cấu trúc `mq_attr` mà `attrp` trỏ đến, trong trường hợp chỉ một trong các tùy chọn `-m` và `-s` được chỉ định trên dòng lệnh. Nếu không có tùy chọn nào trong số này được cung cấp, `attrp` được chỉ định là `NULL` khi gọi `mq_open()`, khiến queue được tạo với các giá trị mặc định được định nghĩa bởi cài đặt cho các thuộc tính queue.

**Listing 52-2:** Tạo một POSIX message queue

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_create.c
#include <mqueue.h>
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
static void
usageError(const char *progName)
{
 fprintf(stderr, "Usage: %s [-cx] [-m maxmsg] [-s msgsize] mq-name "
 "[octal-perms]\n", progName);
 fprintf(stderr, " -c Create queue (O_CREAT)\n");
 fprintf(stderr, " -m maxmsg Set maximum # of messages\n");
 fprintf(stderr, " -s msgsize Set maximum message size\n");
 fprintf(stderr, " -x Create exclusively (O_EXCL)\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int flags, opt;
 mode_t perms;
 mqd_t mqd;
 struct mq_attr attr, *attrp;
 attrp = NULL;
 attr.mq_maxmsg = 50;
 attr.mq_msgsize = 2048;
 flags = O_RDWR;
```

```
 /* Parse command-line options */
 while ((opt = getopt(argc, argv, "cm:s:x")) != -1) {
 switch (opt) {
 case 'c':
 flags |= O_CREAT;
 break;
 case 'm':
 attr.mq_maxmsg = atoi(optarg);
 attrp = &attr;
 break;
 case 's':
 attr.mq_msgsize = atoi(optarg);
 attrp = &attr;
 break;
 case 'x':
 flags |= O_EXCL;
 break;
 default: 
         usageError(argv[0]);
 }
 }
 if (optind >= argc)
 usageError(argv[0]);
 perms = (argc <= optind + 1) ? (S_IRUSR | S_IWUSR) :
 getInt(argv[optind + 1], GN_BASE_8, "octal-perms");
 mqd = mq_open(argv[optind], flags, perms, attrp);
 if (mqd == (mqd_t) -1)
 errExit("mq_open");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_create.c
```

## **Lấy thuộc tính message queue**

Hàm `mq_getattr()` trả về một cấu trúc `mq_attr` chứa thông tin về message queue description và message queue liên kết với descriptor `mqdes`.

```
#include <mqueue.h>
int mq_getattr(mqd_t mqdes, struct mq_attr *attr);
                                             Returns 0 on success, or –1 on error
```

Ngoài các trường `mq_maxmsg` và `mq_msgsize` mà chúng ta đã mô tả, các trường sau được trả về trong cấu trúc được trỏ đến bởi `attr`:

#### mq\_flags

Đây là các flag cho open message queue description liên kết với descriptor `mqdes`. Chỉ có một flag như vậy được chỉ định: `O_NONBLOCK`. Flag này được khởi tạo từ đối số `oflag` của `mq_open()`, và có thể được thay đổi bằng `mq_setattr()`.

#### mq\_curmsgs

Đây là số lượng message hiện đang có trong queue. Thông tin này có thể đã thay đổi vào thời điểm `mq_getattr()` trả về, nếu các process khác đang đọc message từ queue hoặc ghi message vào nó.

Chương trình trong Listing 52-3 sử dụng `mq_getattr()` để lấy thuộc tính cho message queue được chỉ định trong đối số dòng lệnh của nó, và sau đó hiển thị các thuộc tính đó trên standard output.

**Listing 52-3:** Lấy thuộc tính POSIX message queue

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_getattr.c
#include <mqueue.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 mqd_t mqd;
 struct mq_attr attr;
 if (argc != 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s mq-name\n", argv[0]);
 mqd = mq_open(argv[1], O_RDONLY);
 if (mqd == (mqd_t) -1)
 errExit("mq_open");
 if (mq_getattr(mqd, &attr) == -1)
 errExit("mq_getattr");
 printf("Maximum # of messages on queue: %ld\n", attr.mq_maxmsg);
 printf("Maximum message size: %ld\n", attr.mq_msgsize);
 printf("# of messages currently on queue: %ld\n", attr.mq_curmsgs);
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_getattr.c
```

Trong shell session sau đây, chúng ta sử dụng chương trình trong Listing 52-2 để tạo một message queue với các thuộc tính mặc định được định nghĩa bởi cài đặt (nghĩa là đối số cuối cùng cho `mq_open()` là `NULL`), và sau đó sử dụng chương trình trong Listing 52-3 để hiển thị các thuộc tính queue để chúng ta có thể thấy các cài đặt mặc định trên Linux.

```
$ ./pmsg_create -cx /mq
$ ./pmsg_getattr /mq
Maximum # of messages on queue: 10
Maximum message size: 8192
# of messages currently on queue: 0
$ ./pmsg_unlink /mq Remove message queue
```

Từ đầu ra ở trên, chúng ta thấy rằng các giá trị mặc định Linux cho `mq_maxmsg` và `mq_msgsize` lần lượt là 10 và 8192.

Có sự biến đổi lớn về các giá trị mặc định được định nghĩa bởi cài đặt cho `mq_maxmsg` và `mq_msgsize`. Các ứng dụng portable thường cần chọn các giá trị rõ ràng cho các thuộc tính này, thay vì dựa vào các giá trị mặc định.

#### **Sửa đổi thuộc tính message queue**

Hàm `mq_setattr()` thiết lập các thuộc tính của message queue description liên kết với message queue descriptor `mqdes`, và tùy chọn trả về thông tin về message queue.

```
#include <mqueue.h>
int mq_setattr(mqd_t mqdes, const struct mq_attr *newattr,
 struct mq_attr *oldattr);
                                         Returns 0 on success, or –1 on error
```

Hàm `mq_setattr()` thực hiện các nhiệm vụ sau:

-  Nó sử dụng trường `mq_flags` trong cấu trúc `mq_attr` được trỏ đến bởi `newattr` để thay đổi các flag của message queue description liên kết với descriptor `mqdes`.
-  Nếu `oldattr` không phải `NULL`, nó trả về một cấu trúc `mq_attr` chứa các flag message queue description trước đó và các thuộc tính message queue (nghĩa là cùng nhiệm vụ được thực hiện bởi `mq_getattr()`).

Thuộc tính duy nhất mà SUSv3 chỉ định có thể được thay đổi bằng `mq_setattr()` là trạng thái của flag `O_NONBLOCK`.

Để xử lý khả năng một cài đặt cụ thể có thể định nghĩa các flag có thể sửa đổi khác, hoặc SUSv3 có thể thêm các flag mới trong tương lai, một ứng dụng portable nên thay đổi trạng thái của flag `O_NONBLOCK` bằng cách sử dụng `mq_getattr()` để lấy giá trị `mq_flags`, sửa đổi bit `O_NONBLOCK`, và gọi `mq_setattr()` để thay đổi các cài đặt `mq_flags`. Ví dụ, để bật `O_NONBLOCK`, chúng ta sẽ làm như sau:

```
if (mq_getattr(mqd, &attr) == -1)
 errExit("mq_getattr");
attr.mq_flags |= O_NONBLOCK;
if (mq_setattr(mqd, &attr, NULL) == -1)
 errExit("mq_getattr");
```

## **52.5 Trao đổi Message**

Trong phần này, chúng ta xem xét các hàm được sử dụng để gửi message đến và nhận message từ một queue.

### **52.5.1 Gửi Message**

Hàm `mq_send()` thêm message trong buffer được trỏ đến bởi `msg_ptr` vào message queue được tham chiếu bởi descriptor `mqdes`.

```
#include <mqueue.h>
int mq_send(mqd_t mqdes, const char *msg_ptr, size_t msg_len,
 unsigned int msg_prio);
                                         Returns 0 on success, or –1 on error
```

Đối số `msg_len` chỉ định độ dài của message được trỏ đến bởi `msg_ptr`. Giá trị này phải nhỏ hơn hoặc bằng thuộc tính `mq_msgsize` của queue; nếu không, `mq_send()` thất bại với lỗi `EMSGSIZE`. Các message có độ dài 0 được phép.

Mỗi message có một integer priority không âm, được chỉ định bởi đối số `msg_prio`. Các message được sắp xếp trong queue theo thứ tự giảm dần của độ ưu tiên (nghĩa là 0 là độ ưu tiên thấp nhất). Khi một message mới được thêm vào queue, nó được đặt sau bất kỳ message nào khác có cùng độ ưu tiên. Nếu một ứng dụng không cần sử dụng độ ưu tiên message, thì chỉ cần luôn chỉ định `msg_prio` là 0.

> Như đã lưu ý ở đầu chương này, thuộc tính type của System V message cung cấp chức năng khác. System V message luôn được xếp hàng theo thứ tự FIFO, nhưng `msgrcv()` cho phép chúng ta chọn message theo nhiều cách khác nhau: theo thứ tự FIFO, theo type chính xác, hoặc theo type cao nhất nhỏ hơn hoặc bằng một giá trị nào đó.

SUSv3 cho phép một cài đặt quảng cáo giới hạn trên của nó cho các độ ưu tiên message, bằng cách định nghĩa hằng số `MQ_PRIO_MAX` hoặc thông qua giá trị trả về từ `sysconf(_SC_MQ_PRIO_MAX)`. SUSv3 yêu cầu giới hạn này phải ít nhất là 32 (`_POSIX_MQ_PRIO_MAX`); nghĩa là, các độ ưu tiên ít nhất trong phạm vi 0 đến 31 có sẵn. Tuy nhiên, phạm vi thực tế trên các cài đặt là rất biến đổi. Ví dụ, trên Linux, hằng số này có giá trị 32,768; trên Solaris, nó là 32; và trên Tru64, nó là 256.

Nếu message queue đã đầy (nghĩa là giới hạn `mq_maxmsg` cho queue đã đạt đến), thì một `mq_send()` thêm sẽ block cho đến khi có không gian trong queue, hoặc, nếu flag `O_NONBLOCK` có hiệu lực, sẽ thất bại ngay lập tức với lỗi `EAGAIN`.

Chương trình trong Listing 52-4 cung cấp một giao diện dòng lệnh cho hàm `mq_send()`. Chúng ta minh họa việc sử dụng chương trình này trong phần tiếp theo.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_send.c
#include <mqueue.h>
#include <fcntl.h> /* For definition of O_NONBLOCK */
#include "tlpi_hdr.h"
static void
usageError(const char *progName)
{
 fprintf(stderr, "Usage: %s [-n] name msg [prio]\n", progName);
 fprintf(stderr, " -n Use O_NONBLOCK flag\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int flags, opt;
 mqd_t mqd;
 unsigned int prio;
 flags = O_WRONLY;
 while ((opt = getopt(argc, argv, "n")) != -1) {
 switch (opt) {
 case 'n': flags |= O_NONBLOCK; break;
 default: usageError(argv[0]);
 }
 }
 if (optind + 1 >= argc)
 usageError(argv[0]);
 mqd = mq_open(argv[optind], flags);
 if (mqd == (mqd_t) -1)
 errExit("mq_open");
 prio = (argc > optind + 2) ? atoi(argv[optind + 2]) : 0;
 if (mq_send(mqd, argv[optind + 1], strlen(argv[optind + 1]), prio) == -1)
 errExit("mq_send");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_send.c
```

## **52.5.2 Nhận Message**

Hàm `mq_receive()` xóa message cũ nhất với độ ưu tiên cao nhất từ message queue được tham chiếu bởi `mqdes` và trả về message đó trong buffer được trỏ đến bởi `msg_ptr`.

```
#include <mqueue.h>
ssize_t mq_receive(mqd_t mqdes, char *msg_ptr, size_t msg_len,
 unsigned int *msg_prio);
       Returns number of bytes in received message on success, or –1 on error
```

Đối số `msg_len` được caller sử dụng để chỉ định số byte không gian có sẵn trong buffer được trỏ đến bởi `msg_ptr`.

Bất kể kích thước thực tế của message là gì, `msg_len` (và do đó kích thước của buffer được trỏ đến bởi `msg_ptr`) phải lớn hơn hoặc bằng thuộc tính `mq_msgsize` của queue; nếu không, `mq_receive()` thất bại với lỗi `EMSGSIZE`. Nếu chúng ta không biết giá trị của thuộc tính `mq_msgsize` của một queue, chúng ta có thể lấy nó bằng `mq_getattr()`. (Trong một ứng dụng bao gồm các process hợp tác, việc sử dụng `mq_getattr()` thường có thể được bỏ qua, vì ứng dụng thường có thể quyết định cài đặt `mq_msgsize` của queue trước.)

Nếu `msg_prio` không phải `NULL`, thì độ ưu tiên của message nhận được được sao chép vào vị trí được trỏ đến bởi `msg_prio`.

Nếu message queue hiện tại rỗng, thì `mq_receive()` sẽ block cho đến khi có message, hoặc, nếu flag `O_NONBLOCK` có hiệu lực, sẽ thất bại ngay lập tức với lỗi `EAGAIN`. (Không có tương đương với hành vi pipe nơi reader thấy end-of-file nếu không có writer nào.)

Chương trình trong Listing 52-5 cung cấp một giao diện dòng lệnh cho hàm `mq_receive()`. Định dạng lệnh cho chương trình này được hiển thị trong hàm `usageError()`.

Shell session sau đây minh họa việc sử dụng các chương trình trong Listing 52-4 và Listing 52-5. Chúng ta bắt đầu bằng cách tạo một message queue và gửi một vài message với các độ ưu tiên khác nhau:

```
$ ./pmsg_create -cx /mq
$ ./pmsg_send /mq msg-a 5
$ ./pmsg_send /mq msg-b 0
$ ./pmsg_send /mq msg-c 10
```

Sau đó chúng ta thực hiện một loạt các lệnh để lấy message từ queue:

```
$ ./pmsg_receive /mq
Read 5 bytes; priority = 10
msg-c
$ ./pmsg_receive /mq
Read 5 bytes; priority = 5
msg-a
$ ./pmsg_receive /mq
Read 5 bytes; priority = 0
msg-b
```

Như chúng ta có thể thấy từ đầu ra ở trên, các message được lấy theo thứ tự độ ưu tiên.

Tại thời điểm này, queue hiện tại đã rỗng. Khi chúng ta thực hiện một blocking receive khác, thao tác sẽ block:

```
$ ./pmsg_receive /mq
Blocks; we type Control-C to terminate the program
```

Mặt khác, nếu chúng ta thực hiện một nonblocking receive, lời gọi trả về ngay lập tức với trạng thái thất bại:

\$ **./pmsg\_receive -n /mq** ERROR [EAGAIN/EWOULDBLOCK Resource temporarily unavailable] mq\_receive

**Listing 52-5:** Đọc một message từ POSIX message queue

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_receive.c
#include <mqueue.h>
#include <fcntl.h> /* For definition of O_NONBLOCK */
#include "tlpi_hdr.h"
static void
usageError(const char *progName)
{
 fprintf(stderr, "Usage: %s [-n] name\n", progName);
 fprintf(stderr, " -n Use O_NONBLOCK flag\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int flags, opt;
 mqd_t mqd;
 unsigned int prio;
 void *buffer;
 struct mq_attr attr;
 ssize_t numRead;
 flags = O_RDONLY;
 while ((opt = getopt(argc, argv, "n")) != -1) {
 switch (opt) {
 case 'n': flags |= O_NONBLOCK; break;
 default: usageError(argv[0]);
 }
 }
 if (optind >= argc)
 usageError(argv[0]);
 mqd = mq_open(argv[optind], flags);
 if (mqd == (mqd_t) -1)
 errExit("mq_open");
 if (mq_getattr(mqd, &attr) == -1)
 errExit("mq_getattr");
 buffer = malloc(attr.mq_msgsize);
 if (buffer == NULL)
 errExit("malloc");
```

```
 numRead = mq_receive(mqd, buffer, attr.mq_msgsize, &prio);
 if (numRead == -1)
 errExit("mq_receive");
 printf("Read %ld bytes; priority = %u\n", (long) numRead, prio);
 if (write(STDOUT_FILENO, buffer, numRead) == -1)
 errExit("write");
 write(STDOUT_FILENO, "\n", 1);
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/pmsg_receive.c
```

## **52.5.3 Gửi và Nhận Message với Timeout**

Các hàm `mq_timedsend()` và `mq_timedreceive()` hoàn toàn giống như `mq_send()` và `mq_receive()`, ngoại trừ nếu thao tác không thể được thực hiện ngay lập tức, và flag `O_NONBLOCK` không có hiệu lực cho message queue description, thì đối số `abs_timeout` chỉ định giới hạn về thời gian lời gọi sẽ block.

```
#define _XOPEN_SOURCE 600
#include <mqueue.h>
#include <time.h>
int mq_timedsend(mqd_t mqdes, const char *msg_ptr, size_t msg_len,
 unsigned int msg_prio, const struct timespec *abs_timeout);
                                         Returns 0 on success, or –1 on error
ssize_t mq_timedreceive(mqd_t mqdes, char *msg_ptr, size_t msg_len,
 unsigned int *msg_prio, const struct timespec *abs_timeout);
       Returns number of bytes in received message on success, or –1 on error
```

Đối số `abs_timeout` là một cấu trúc `timespec` (Mục 23.4.2) chỉ định timeout như một giá trị tuyệt đối tính bằng giây và nanosecond kể từ Epoch. Để thực hiện relative timeout, chúng ta có thể lấy giá trị hiện tại của đồng hồ `CLOCK_REALTIME` bằng `clock_gettime()` và thêm lượng cần thiết vào giá trị đó để tạo ra một cấu trúc `timespec` được khởi tạo phù hợp.

Nếu một lời gọi đến `mq_timedsend()` hoặc `mq_timedreceive()` hết thời gian chờ mà không thể hoàn thành thao tác của nó, thì lời gọi thất bại với lỗi `ETIMEDOUT`.

Trên Linux, chỉ định `abs_timeout` là `NULL` có nghĩa là timeout vô hạn. Tuy nhiên, hành vi này không được chỉ định trong SUSv3, và các ứng dụng portable không thể dựa vào nó.

Các hàm `mq_timedsend()` và `mq_timedreceive()` ban đầu bắt nguồn từ POSIX.1d (1999) và không có sẵn trên tất cả các cài đặt UNIX.

## **52.6 Thông báo Message**

Một tính năng phân biệt POSIX message queue với các System V counterpart là khả năng nhận thông báo không đồng bộ về sự có sẵn của message trên một queue trước đó rỗng (nghĩa là khi queue chuyển từ rỗng sang không rỗng). Tính năng này có nghĩa là thay vì thực hiện một lời gọi `mq_receive()` blocking hoặc đánh dấu message queue descriptor là nonblocking và thực hiện các lời gọi `mq_receive()` định kỳ ("poll") trên queue, một process có thể yêu cầu thông báo về sự đến của message và sau đó thực hiện các nhiệm vụ khác cho đến khi được thông báo. Một process có thể chọn được thông báo qua signal hoặc qua việc gọi một hàm trong một thread riêng biệt.

> Tính năng thông báo của POSIX message queue tương tự như cơ sở thông báo mà chúng ta mô tả cho POSIX timer trong Mục 23.6. (Cả hai API này đều bắt nguồn từ POSIX.1b.)

Hàm `mq_notify()` đăng ký process đang gọi để nhận thông báo khi một message đến trên queue rỗng được tham chiếu bởi descriptor `mqdes`.

```
#include <mqueue.h>
int mq_notify(mqd_t mqdes, const struct sigevent *notification);
                                             Returns 0 on success, or –1 on error
```

Đối số `notification` chỉ định cơ chế mà qua đó process được thông báo. Trước khi đi vào chi tiết của đối số `notification`, chúng ta lưu ý một vài điểm về message notification:

-  Tại bất kỳ thời điểm nào, chỉ có một process ("process đã đăng ký") có thể được đăng ký để nhận thông báo từ một message queue cụ thể. Nếu đã có một process được đăng ký cho một message queue, các lần thử đăng ký thêm cho queue đó sẽ thất bại (`mq_notify()` thất bại với lỗi `EBUSY`).
-  Process đã đăng ký chỉ được thông báo khi một message mới đến trên một queue trước đó rỗng. Nếu queue đã chứa message tại thời điểm đăng ký, một thông báo sẽ chỉ xảy ra sau khi queue được làm rỗng và một message mới đến.
-  Sau khi một thông báo được gửi đến process đã đăng ký, đăng ký bị xóa, và bất kỳ process nào sau đó có thể đăng ký nhận thông báo. Nói cách khác, miễn là một process muốn tiếp tục nhận thông báo, nó phải đăng ký lại sau mỗi thông báo bằng cách một lần nữa gọi `mq_notify()`.
-  Process đã đăng ký chỉ được thông báo nếu không có process nào khác hiện đang bị block trong lời gọi `mq_receive()` cho queue. Nếu một process khác bị block trong `mq_receive()`, process đó sẽ đọc message, và process đã đăng ký sẽ vẫn được đăng ký.
-  Một process có thể hủy đăng ký rõ ràng khỏi mục tiêu nhận message notification bằng cách gọi `mq_notify()` với đối số `notification` là `NULL`.

Chúng ta đã hiển thị cấu trúc `sigevent` được sử dụng để gõ đối số `notification` trong Mục 23.6.1. Ở đây, chúng ta trình bày cấu trúc ở dạng đơn giản, chỉ hiển thị các trường liên quan đến việc thảo luận về `mq_notify()`:

```
union sigval {
 int sival_int; /* Integer value for accompanying data */
 void *sival_ptr; /* Pointer value for accompanying data */
};
```

```
struct sigevent {
 int sigev_notify; /* Notification method */
 int sigev_signo; /* Notification signal for SIGEV_SIGNAL */
 union sigval sigev_value; /* Value passed to signal handler or
 thread function */
 void (*sigev_notify_function) (union sigval);
 /* Thread notification function */
 void *sigev_notify_attributes; /* Really 'pthread_attr_t' */
};
```

Trường `sigev_notify` của cấu trúc này được thiết lập theo một trong các giá trị sau:

#### `SIGEV_NONE`

Đăng ký process này để nhận thông báo, nhưng khi một message đến trên queue trước đó rỗng, không thực sự thông báo cho process. Như thường lệ, đăng ký bị xóa khi một message mới đến trên một queue rỗng.

#### `SIGEV_SIGNAL`

Thông báo cho process bằng cách tạo signal được chỉ định trong trường `sigev_signo`. Nếu `sigev_signo` là một realtime signal, thì trường `sigev_value` chỉ định dữ liệu đi kèm với signal (Mục 22.8.1). Dữ liệu này có thể được lấy thông qua trường `si_value` của cấu trúc `siginfo_t` được truyền cho signal handler hoặc được trả về bởi lời gọi `sigwaitinfo()` hoặc `sigtimedwait()`. Các trường sau trong cấu trúc `siginfo_t` cũng được điền: `si_code`, với giá trị `SI_MESGQ`; `si_signo`, với số signal; `si_pid`, với process ID của process gửi message; và `si_uid`, với real user ID của process gửi message. (Các trường `si_pid` và `si_uid` không được thiết lập trên hầu hết các cài đặt khác.)

#### `SIGEV_THREAD`

Thông báo cho process bằng cách gọi hàm được chỉ định trong `sigev_notify_function` như thể nó là hàm start trong một thread mới. Trường `sigev_notify_attributes` có thể được chỉ định là `NULL` hoặc như một pointer đến cấu trúc `pthread_attr_t` định nghĩa các thuộc tính cho thread (Mục 29.8). Giá trị `union sigval` được chỉ định trong `sigev_value` được truyền như đối số của hàm này.

## **52.6.1 Nhận Thông báo qua Signal**

Listing 52-6 cung cấp một ví dụ về message notification sử dụng signal. Chương trình này thực hiện các bước sau:

- 1. Mở message queue được đặt tên trên dòng lệnh ở chế độ nonblocking q, xác định thuộc tính `mq_msgsize` cho queue w, và phân bổ một buffer có kích thước đó để nhận message e.
- 2. Block notification signal (`SIGUSR1`) và thiết lập một handler cho nó r.
- 3. Thực hiện một lời gọi ban đầu đến `mq_notify()` để đăng ký process nhận message notification t.
- 4. Thực hiện một vòng lặp vô hạn thực hiện các bước sau:
  - a) Gọi `sigsuspend()`, bỏ block notification signal và chờ cho đến khi signal được bắt y. Trở về từ system call này chỉ ra rằng một message notification đã xảy ra. Tại thời điểm này, process sẽ đã bị hủy đăng ký nhận message notification.
  - b) Gọi `mq_notify()` để đăng ký lại process này để nhận message notification u.
  - c) Thực hiện một vòng lặp while làm cạn queue bằng cách đọc càng nhiều message càng tốt i.

**Listing 52-6:** Nhận message notification qua signal

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/mq_notify_sig.c
  #include <signal.h>
  #include <mqueue.h>
  #include <fcntl.h> /* For definition of O_NONBLOCK */
  #include "tlpi_hdr.h"
  #define NOTIFY_SIG SIGUSR1
  static void
  handler(int sig)
  {
   /* Just interrupt sigsuspend() */
  }
  int
  main(int argc, char *argv[])
  {
   struct sigevent sev;
   mqd_t mqd;
   struct mq_attr attr;
   void *buffer;
   ssize_t numRead;
   sigset_t blockMask, emptyMask;
   struct sigaction sa;
   if (argc != 2 || strcmp(argv[1], "--help") == 0)
   usageErr("%s mq-name\n", argv[0]);
q mqd = mq_open(argv[1], O_RDONLY | O_NONBLOCK);
   if (mqd == (mqd_t) -1)
   errExit("mq_open");
w if (mq_getattr(mqd, &attr) == -1)
   errExit("mq_getattr");
e buffer = malloc(attr.mq_msgsize);
   if (buffer == NULL)
   errExit("malloc");
r sigemptyset(&blockMask);
   sigaddset(&blockMask, NOTIFY_SIG);
   if (sigprocmask(SIG_BLOCK, &blockMask, NULL) == -1)
   errExit("sigprocmask");
```

```
 sigemptyset(&sa.sa_mask);
   sa.sa_flags = 0;
   sa.sa_handler = handler;
   if (sigaction(NOTIFY_SIG, &sa, NULL) == -1)
   errExit("sigaction");
t sev.sigev_notify = SIGEV_SIGNAL;
   sev.sigev_signo = NOTIFY_SIG;
   if (mq_notify(mqd, &sev) == -1)
   errExit("mq_notify");
   sigemptyset(&emptyMask);
   for (;;) {
y sigsuspend(&emptyMask); /* Wait for notification signal */
u if (mq_notify(mqd, &sev) == -1)
   errExit("mq_notify");
i while ((numRead = mq_receive(mqd, buffer, attr.mq_msgsize, NULL)) >= 0)
   printf("Read %ld bytes\n", (long) numRead);
   if (errno != EAGAIN) /* Unexpected error */
   errExit("mq_receive");
   }
  }
  –––––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/mq_notify_sig.c
```

Các khía cạnh khác nhau của chương trình trong Listing 52-6 cần được bình luận thêm:

-  Chúng ta block notification signal và sử dụng `sigsuspend()` để chờ nó, thay vì `pause()`, để ngăn chặn khả năng bỏ lỡ một signal được gửi trong khi chương trình đang thực thi ở nơi khác (nghĩa là không bị block chờ signal) trong vòng lặp `for`. Nếu điều này xảy ra, và chúng ta đang sử dụng `pause()` để chờ signal, thì lời gọi `pause()` tiếp theo sẽ block, mặc dù một signal đã được gửi rồi.
-  Chúng ta mở queue ở chế độ nonblocking, và, bất cứ khi nào một notification xảy ra, chúng ta sử dụng vòng lặp while để đọc tất cả message từ queue. Làm cạn queue theo cách này đảm bảo rằng một notification tiếp theo được tạo ra khi một message mới đến. Sử dụng chế độ nonblocking có nghĩa là vòng lặp while sẽ kết thúc (`mq_receive()` sẽ thất bại với lỗi `EAGAIN`) khi chúng ta đã làm cạn queue. (Phương pháp này tương tự như việc sử dụng nonblocking I/O với edge-triggered I/O notification, mà chúng ta mô tả trong Mục 63.1.1, và được sử dụng vì các lý do tương tự.)
-  Trong vòng lặp `for`, điều quan trọng là chúng ta phải đăng ký lại nhận message notification trước khi đọc tất cả message từ queue. Nếu chúng ta đảo ngược các bước này, chuỗi sau có thể xảy ra: tất cả message được đọc từ queue, và vòng lặp while kết thúc; một message khác được đặt vào queue; `mq_notify()` được gọi để đăng ký lại nhận message notification. Tại thời điểm này, không có notification signal nào nữa sẽ được tạo ra, vì queue đã không rỗng. Do đó, chương trình sẽ vẫn bị block vĩnh viễn trong lời gọi tiếp theo của nó đến `sigsuspend()`.

## **52.6.2 Nhận Thông báo qua Thread**

Listing 52-7 cung cấp một ví dụ về message notification sử dụng thread. Chương trình này chia sẻ một số tính năng thiết kế với chương trình trong Listing 52-6:

-  Khi message notification xảy ra, chương trình bật lại notification trước khi làm cạn queue w.
-  Chế độ nonblocking được sử dụng để, sau khi nhận được notification, chúng ta có thể hoàn toàn làm cạn queue mà không bị block t.

**Listing 52-7:** Nhận message notification qua thread

```
––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/mq_notify_thread.c
  #include <pthread.h>
  #include <mqueue.h>
  #include <fcntl.h> /* For definition of O_NONBLOCK */
  #include "tlpi_hdr.h"
  static void notifySetup(mqd_t *mqdp);
  static void /* Thread notification function */
q threadFunc(union sigval sv)
  {
   ssize_t numRead;
   mqd_t *mqdp;
   void *buffer;
   struct mq_attr attr;
   mqdp = sv.sival_ptr;
   if (mq_getattr(*mqdp, &attr) == -1)
   errExit("mq_getattr");
   buffer = malloc(attr.mq_msgsize);
   if (buffer == NULL)
   errExit("malloc");
w notifySetup(mqdp);
   while ((numRead = mq_receive(*mqdp, buffer, attr.mq_msgsize, NULL)) >= 0)
   printf("Read %ld bytes\n", (long) numRead);
   if (errno != EAGAIN) /* Unexpected error */
   errExit("mq_receive");
   free(buffer);
   pthread_exit(NULL);
  }
  static void
  notifySetup(mqd_t *mqdp)
  {
   struct sigevent sev;
```

```
e sev.sigev_notify = SIGEV_THREAD; /* Notify via thread */
   sev.sigev_notify_function = threadFunc;
   sev.sigev_notify_attributes = NULL;
   /* Could be pointer to pthread_attr_t structure */
r sev.sigev_value.sival_ptr = mqdp; /* Argument to threadFunc() */
   if (mq_notify(*mqdp, &sev) == -1)
   errExit("mq_notify");
  }
  int
  main(int argc, char *argv[])
  {
   mqd_t mqd;
   if (argc != 2 || strcmp(argv[1], "--help") == 0)
   usageErr("%s mq-name\n", argv[0]);
t mqd = mq_open(argv[1], O_RDONLY | O_NONBLOCK);
   if (mqd == (mqd_t) -1)
   errExit("mq_open");
y notifySetup(&mqd);
   pause(); /* Wait for notifications via thread function */
  }
  ––––––––––––––––––––––––––––––––––––––––––––––––––– pmsg/mq_notify_thread.c
```

Lưu ý các điểm tiếp theo sau đây về thiết kế của chương trình trong Listing 52-7:

-  Chương trình yêu cầu notification qua một thread, bằng cách chỉ định `SIGEV_THREAD` trong trường `sigev_notify` của cấu trúc `sigevent` được truyền cho `mq_notify()`. Hàm start của thread, `threadFunc()`, được chỉ định trong trường `sigev_notify_function` e.
-  Sau khi bật message notification, chương trình chính tạm dừng vô thời hạn y; timer notification được gửi bởi các lần gọi `threadFunc()` trong một thread riêng biệt q.
-  Chúng ta có thể đã làm cho message queue descriptor, `mqd`, có thể nhìn thấy trong `threadFunc()` bằng cách biến nó thành một biến toàn cục. Tuy nhiên, chúng ta đã áp dụng một phương pháp khác để minh họa giải pháp thay thế: chúng ta đặt địa chỉ của message queue descriptor vào trường `sigev_value.sival_ptr` được truyền cho `mq_notify()` r. Khi `threadFunc()` được gọi sau đó, địa chỉ này được truyền như đối số của nó.

Chúng ta phải gán một pointer đến message queue descriptor cho `sigev_value.sival_ptr`, thay vì (phiên bản cast nào đó của) chính descriptor, vì, ngoài việc quy định rằng nó không phải là kiểu mảng, SUSv3 không đảm bảo về bản chất hoặc kích thước của kiểu được sử dụng để biểu diễn kiểu dữ liệu `mqd_t`.

## **52.7 Các Tính năng Dành riêng cho Linux**

Việc triển khai Linux của POSIX message queue cung cấp một số tính năng không chuẩn nhưng vẫn hữu ích.

#### **Hiển thị và xóa các đối tượng message queue qua dòng lệnh**

Trong Chương 51, chúng ta đã đề cập rằng các đối tượng POSIX IPC được triển khai như các file trong các file system ảo, và các file này có thể được liệt kê và xóa bằng `ls` và `rm`. Để làm điều này với POSIX message queue, chúng ta phải mount file system message queue bằng lệnh có dạng sau:

```
# mount -t mqueue source target
```

`source` có thể là bất kỳ tên nào (thông thường dùng chuỗi `none`). Tầm quan trọng duy nhất của nó là nó xuất hiện trong `/proc/mounts` và được hiển thị bởi các lệnh `mount` và `df`. `target` là điểm mount cho file system message queue.

Shell session sau đây cho thấy cách mount file system message queue và hiển thị nội dung của nó. Chúng ta bắt đầu bằng cách tạo một mount point cho file system và mount nó:

```
$ su Privilege is required for mount
Password:
# mkdir /dev/mqueue
# mount -t mqueue none /dev/mqueue
$ exit Terminate root shell session
```

Tiếp theo, chúng ta hiển thị bản ghi trong `/proc/mounts` cho mount mới, và sau đó hiển thị các quyền cho thư mục mount:

```
$ cat /proc/mounts | grep mqueue
none /dev/mqueue mqueue rw 0 0
$ ls -ld /dev/mqueue
drwxrwxrwt 2 root root 40 Jul 26 12:09 /dev/mqueue
```

Một điểm cần lưu ý từ đầu ra của lệnh `ls` là file system message queue tự động được mount với sticky bit được thiết lập cho thư mục mount. (Chúng ta thấy điều này từ thực tế là có `t` trong trường other-execute permission được hiển thị bởi `ls`.) Điều này có nghĩa là một process không có đặc quyền chỉ có thể unlink các message queue mà nó sở hữu.

Tiếp theo, chúng ta tạo một message queue, sử dụng `ls` để cho thấy nó hiển thị trong file system, và sau đó xóa message queue:

```
$ ./pmsg_create -c /newq
$ ls /dev/mqueue
newq
$ rm /dev/mqueue/newq
```

#### **Lấy thông tin về một message queue**

Chúng ta có thể hiển thị nội dung của các file trong file system message queue. Mỗi file ảo này chứa thông tin về message queue liên kết:

```
$ ./pmsg_create -c /mq Create a queue
$ ./pmsg_send /mq abcdefg Write 7 bytes to the queue
$ cat /dev/mqueue/mq
QSIZE:7 NOTIFY:0 SIGNO:0 NOTIFY_PID:0
```

Trường `QSIZE` là số đếm tổng số byte dữ liệu trong queue. Các trường còn lại liên quan đến message notification. Nếu `NOTIFY_PID` khác 0, thì process có process ID được chỉ định đã đăng ký nhận message notification từ queue này, và các trường còn lại cung cấp thông tin về loại notification:

-  `NOTIFY` là một giá trị tương ứng với một trong các hằng số `sigev_notify`: 0 cho `SIGEV_SIGNAL`, 1 cho `SIGEV_NONE`, hoặc 2 cho `SIGEV_THREAD`.
-  Nếu phương thức notification là `SIGEV_SIGNAL`, trường `SIGNO` chỉ ra signal nào được gửi cho message notification.

Shell session sau đây minh họa thông tin xuất hiện trong các trường này:

```
$ ./mq_notify_sig /mq & Notify using SIGUSR1 (signal 10 on x86)
[1] 18158
$ cat /dev/mqueue/mq
QSIZE:7 NOTIFY:0 SIGNO:10 NOTIFY_PID:18158
$ kill %1
[1] Terminated ./mq_notify_sig /mq
$ ./mq_notify_thread /mq & Notify using a thread
[2] 18160
$ cat /dev/mqueue/mq
QSIZE:7 NOTIFY:2 SIGNO:0 NOTIFY_PID:18160
```

#### **Sử dụng message queue với các I/O model thay thế**

Trong cài đặt Linux, một message queue descriptor thực sự là một file descriptor. Chúng ta có thể monitor file descriptor này bằng các system call I/O multiplexing (`select()` và `poll()`) hoặc epoll API. (Xem Chương 63 để biết thêm chi tiết về các API này.) Điều này cho phép chúng ta tránh khó khăn mà chúng ta gặp phải với System V message queue khi cố gắng chờ đầu vào trên cả message queue và file descriptor (xem Mục 46.9). Tuy nhiên, tính năng này không chuẩn; SUSv3 không yêu cầu rằng message queue descriptor được triển khai như file descriptor.

## **52.8 Giới hạn Message Queue**

SUSv3 định nghĩa hai giới hạn cho POSIX message queue:

`MQ_PRIO_MAX`

Chúng ta đã mô tả giới hạn này, xác định độ ưu tiên tối đa cho một message, trong Mục 52.5.1.

`MQ_OPEN_MAX`

Một cài đặt có thể định nghĩa giới hạn này để chỉ ra số lượng message queue tối đa mà một process có thể giữ mở. SUSv3 yêu cầu giới hạn này phải ít nhất là `_POSIX_MQ_OPEN_MAX` (8). Linux không định nghĩa giới hạn này. Thay vào đó, vì Linux triển khai message queue descriptor như file descriptor (Mục 52.7), các giới hạn áp dụng là những giới hạn áp dụng cho file descriptor. (Nói cách khác, trên Linux, các giới hạn per-process và system-wide về số lượng file descriptor thực sự áp dụng cho tổng số file descriptor và message queue descriptor.) Để biết thêm chi tiết về các giới hạn áp dụng, xem thảo luận về resource limit `RLIMIT_NOFILE` trong Mục 36.3.

Ngoài các giới hạn được chỉ định bởi SUSv3 ở trên, Linux cung cấp một số file `/proc` để xem và (với đặc quyền) thay đổi các giới hạn kiểm soát việc sử dụng POSIX message queue. Ba file sau nằm trong thư mục `/proc/sys/fs/mqueue`:

`msg_max`

Giới hạn này chỉ định một trần cho thuộc tính `mq_maxmsg` của các message queue mới (nghĩa là trần cho `attr.mq_maxmsg` khi tạo queue bằng `mq_open()`). Giá trị mặc định cho giới hạn này là 10. Giá trị tối thiểu là 1 (10 trong các kernel trước Linux 2.6.28). Giá trị tối đa được định nghĩa bởi hằng số kernel `HARD_MSGMAX`. Giá trị cho hằng số này được tính là `(131,072 / sizeof(void *))`, đánh giá là 32,768 trên Linux/x86-32. Khi một process có đặc quyền (`CAP_SYS_RESOURCE`) gọi `mq_open()`, giới hạn `msg_max` bị bỏ qua, nhưng `HARD_MSGMAX` vẫn hoạt động như một trần cho `attr.mq_maxmsg`.

`msgsize_max`

Giới hạn này chỉ định một trần cho thuộc tính `mq_msgsize` của các message queue mới được tạo bởi các process không có đặc quyền (nghĩa là trần cho `attr.mq_msgsize` khi tạo queue bằng `mq_open()`). Giá trị mặc định cho giới hạn này là 8192. Giá trị tối thiểu là 128 (8192 trong các kernel trước Linux 2.6.28). Giá trị tối đa là 1,048,576 (`INT_MAX` trong các kernel trước 2.6.28). Giới hạn này bị bỏ qua khi một process có đặc quyền (`CAP_SYS_RESOURCE`) gọi `mq_open()`.

`queues_max`

Đây là giới hạn trên toàn hệ thống về số lượng message queue có thể được tạo. Một khi đạt đến giới hạn này, chỉ một process có đặc quyền (`CAP_SYS_RESOURCE`) mới có thể tạo các queue mới. Giá trị mặc định cho giới hạn này là 256. Nó có thể được thay đổi thành bất kỳ giá trị nào trong phạm vi 0 đến `INT_MAX`.

Linux cũng cung cấp resource limit `RLIMIT_MSGQUEUE`, có thể được sử dụng để đặt một trần trên lượng không gian có thể được tiêu thụ bởi tất cả các message queue thuộc về real user ID của process đang gọi. Xem Mục 36.3 để biết chi tiết.

## **52.9 So sánh POSIX và System V Message Queue**

Mục 51.2 đã liệt kê các ưu điểm khác nhau của giao diện POSIX IPC so với giao diện IPC System V: giao diện POSIX IPC đơn giản hơn và nhất quán hơn với mô hình file UNIX truyền thống, và các đối tượng POSIX IPC được đếm tham chiếu, đơn giản hóa nhiệm vụ xác định khi nào cần xóa một đối tượng. Các ưu điểm chung này cũng áp dụng cho POSIX message queue.

POSIX message queue cũng có các ưu điểm cụ thể sau so với System V message queue:

-  Tính năng message notification cho phép một (duy nhất) process được thông báo không đồng bộ qua signal hoặc việc khởi tạo một thread khi message đến trên một queue trước đó rỗng.
-  Trên Linux (nhưng không phải các cài đặt UNIX khác), POSIX message queue có thể được monitor bằng `poll()`, `select()` và `epoll`. System V message queue không cung cấp tính năng này.

Tuy nhiên, POSIX message queue cũng có một số nhược điểm so với System V message queue:

-  POSIX message queue ít portable hơn. Vấn đề này ngay cả áp dụng giữa các hệ thống Linux, vì hỗ trợ message queue chỉ có sẵn kể từ kernel 2.6.6.
-  Cơ sở để chọn System V message theo type cung cấp tính linh hoạt hơn một chút so với thứ tự ưu tiên nghiêm ngặt của POSIX message.

Có sự biến đổi lớn về cách POSIX message queue được triển khai trên các hệ thống UNIX. Một số hệ thống cung cấp các cài đặt trong user space, và trên ít nhất một cài đặt như vậy (Solaris 10), trang man `mq_open()` ghi chú rõ ràng rằng cài đặt không thể được coi là an toàn. Trên Linux, một trong những động lực để chọn cài đặt kernel của message queue là không thể cung cấp một cài đặt user space an toàn.

## **52.10 Tóm tắt**

POSIX message queue cho phép các process trao đổi dữ liệu dưới dạng message. Mỗi message có một integer priority liên kết, và các message được xếp hàng (và do đó nhận) theo thứ tự độ ưu tiên.

POSIX message queue có một số ưu điểm so với System V message queue, đáng chú ý là chúng được đếm tham chiếu và một process có thể được thông báo không đồng bộ về sự đến của message trên một queue rỗng. Tuy nhiên, POSIX message queue ít portable hơn System V message queue.

#### **Thông tin thêm**

[Stevens, 1999] cung cấp một trình bày thay thế về POSIX message queue và cho thấy một cài đặt user-space sử dụng memory-mapped file. POSIX message queue cũng được mô tả chi tiết trong [Gallmeister, 1995].

## **52.11 Bài tập**

- **52-1.** Sửa đổi chương trình trong Listing 52-5 (pmsg\_receive.c) để chấp nhận timeout (số giây tương đối) trên dòng lệnh, và sử dụng `mq_timedreceive()` thay vì `mq_receive()`.
- **52-2.** Viết lại ứng dụng client-server sequence-number của Mục 44.8 để sử dụng POSIX message queue.
- **52-3.** Viết lại ứng dụng file-server của Mục 46.8 để sử dụng POSIX message queue thay vì System V message queue.
- **52-4.** Viết một chương trình chat đơn giản (tương tự như `talk(1)`, nhưng không có giao diện curses) sử dụng POSIX message queue.
- **52-5.** Sửa đổi chương trình trong Listing 52-6 (mq\_notify\_sig.c) để chứng minh rằng message notification được thiết lập bởi `mq_notify()` chỉ xảy ra một lần. Điều này có thể được thực hiện bằng cách xóa lời gọi `mq_notify()` bên trong vòng lặp `for`.

- **52-6.** Thay thế việc sử dụng signal handler trong Listing 52-6 (mq\_notify\_sig.c) bằng việc sử dụng `sigwaitinfo()`. Khi trở về từ `sigwaitinfo()`, hiển thị các giá trị trong cấu trúc `siginfo_t` được trả về. Làm thế nào chương trình có thể lấy message queue descriptor trong cấu trúc `siginfo_t` được trả về bởi `sigwaitinfo()`?
- **52-7.** Trong Listing 52-7, liệu `buffer` có thể được tạo thành một biến toàn cục và bộ nhớ của nó được phân bổ chỉ một lần (trong chương trình chính) không? Giải thích câu trả lời của bạn.
