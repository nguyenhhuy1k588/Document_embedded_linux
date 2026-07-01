## Chương 46
# **MESSAGE QUEUE SYSTEM V**

Chương này mô tả message queue System V. Message queue cho phép các process trao đổi dữ liệu dưới dạng message. Mặc dù message queue có một số điểm tương đồng với pipe và FIFO, nhưng chúng cũng khác nhau ở một số điểm quan trọng:

- Handle dùng để tham chiếu đến một message queue là identifier được trả về bởi lời gọi `msgget()`. Các identifier này không giống với file descriptor được dùng cho hầu hết các hình thức I/O khác trên hệ thống UNIX.
- Giao tiếp qua message queue có tính chất hướng message (message-oriented); nghĩa là, reader nhận toàn bộ message như writer đã ghi. Không thể đọc một phần message rồi để phần còn lại trong queue, hay đọc nhiều message cùng một lúc. Điều này trái ngược với pipe, vốn cung cấp một luồng byte không phân biệt ranh giới (tức là với pipe, reader có thể đọc một số byte tùy ý tại một thời điểm, bất kể kích thước các khối dữ liệu mà writer đã ghi).
- Ngoài việc chứa dữ liệu, mỗi message còn có một kiểu nguyên (integer type). Các message có thể được lấy ra khỏi queue theo thứ tự vào trước ra trước (first-in, first-out), hoặc được lấy theo kiểu (type).

Ở cuối chương này (Mục 46.9), chúng ta sẽ tóm tắt một số hạn chế của message queue System V. Những hạn chế này dẫn đến kết luận rằng, nếu có thể, các ứng dụng mới nên tránh sử dụng message queue System V, thay vào đó hãy dùng các cơ chế IPC khác như FIFO, POSIX message queue và socket. Tuy nhiên, khi message queue lần đầu được thiết kế, những cơ chế thay thế này chưa có hoặc chưa phổ biến trên các hệ thống UNIX. Do đó, đã có nhiều ứng dụng hiện tại sử dụng message queue, và đây là một trong những lý do chính để chúng ta mô tả chúng.

## **46.1 Tạo hoặc Mở một Message Queue**

System call `msgget()` tạo một message queue mới hoặc lấy identifier của một queue đã tồn tại.

```
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
int msgget(key_t key, int msgflg);
                   Returns message queue identifier on success, or –1 on error
```

Đối số `key` là một key được tạo bằng một trong các phương pháp mô tả trong Mục 45.2 (tức là thường là giá trị `IPC_PRIVATE` hoặc một key được trả về bởi `ftok()`). Đối số `msgflg` là một bitmask xác định các quyền (Table 15-4, trang 295) sẽ được áp dụng lên một message queue mới hoặc được kiểm tra đối với một queue hiện có. Ngoài ra, không hoặc nhiều flag sau đây có thể được OR (`|`) vào `msgflg` để điều khiển hoạt động của `msgget()`:

`IPC_CREAT`

Nếu không có message queue nào với key đã chỉ định tồn tại, hãy tạo một queue mới.

`IPC_EXCL`

Nếu `IPC_CREAT` cũng được chỉ định, và một queue với key đã chỉ định đã tồn tại, hãy thất bại với lỗi `EEXIST`.

Các flag này được mô tả chi tiết hơn trong Mục 45.1.

System call `msgget()` bắt đầu bằng cách tìm kiếm trong tập hợp tất cả các message queue hiện có để tìm một queue có key đã chỉ định. Nếu tìm thấy một queue phù hợp, identifier của queue đó được trả về (trừ khi cả `IPC_CREAT` và `IPC_EXCL` đều được chỉ định trong `msgflg`, trong trường hợp đó một lỗi được trả về). Nếu không tìm thấy queue nào phù hợp và `IPC_CREAT` được chỉ định trong `msgflg`, một queue mới sẽ được tạo và identifier của nó được trả về.

Chương trình trong Listing 46-1 cung cấp một giao diện dòng lệnh cho system call `msgget()`. Chương trình cho phép sử dụng các tùy chọn và đối số dòng lệnh để chỉ định tất cả các khả năng cho đối số `key` và `msgflg` của `msgget()`. Chi tiết về định dạng lệnh được chấp nhận bởi chương trình này được hiển thị trong hàm `usageError()`. Khi tạo queue thành công, chương trình in ra identifier của queue. Chúng ta sẽ minh họa việc sử dụng chương trình này trong Mục 46.2.2.

```
Listing 46-1: Sử dụng msgget()
–––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_create.c
#include <sys/types.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <sys/stat.h>
#include "tlpi_hdr.h"
```

```
static void /* Print usage info, then exit */
usageError(const char *progName, const char *msg)
{
 if (msg != NULL)
 fprintf(stderr, "%s", msg);
 fprintf(stderr, "Usage: %s [-cx] {-f pathname | -k key | -p} "
 "[octal-perms]\n", progName);
 fprintf(stderr, " -c Use IPC_CREAT flag\n");
 fprintf(stderr, " -x Use IPC_EXCL flag\n");
 fprintf(stderr, " -f pathname Generate key using ftok()\n");
 fprintf(stderr, " -k key Use 'key' as key\n");
 fprintf(stderr, " -p Use IPC_PRIVATE key\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int numKeyFlags; /* Counts -f, -k, and -p options */
 int flags, msqid, opt;
 unsigned int perms;
 long lkey;
 key_t key;
 /* Parse command-line options and arguments */
 numKeyFlags = 0;
 flags = 0;
 while ((opt = getopt(argc, argv, "cf:k:px")) != -1) {
 switch (opt) {
 case 'c':
 flags |= IPC_CREAT;
 break;
 case 'f': /* -f pathname */
 key = ftok(optarg, 1);
 if (key == -1)
 errExit("ftok");
 numKeyFlags++;
 break;
 case 'k': /* -k key (octal, decimal or hexadecimal) */
 if (sscanf(optarg, "%li", &lkey) != 1)
 cmdLineErr("-k option requires a numeric argument\n");
 key = lkey;
 numKeyFlags++;
 break;
 case 'p':
 key = IPC_PRIVATE;
 numKeyFlags++;
 break;
```

```
 case 'x':
 flags |= IPC_EXCL;
 break;
 default:
 usageError(argv[0], "Bad option\n");
 }
 }
 if (numKeyFlags != 1)
 usageError(argv[0], "Exactly one of the options -f, -k, "
 "or -p must be supplied\n");
 perms = (optind == argc) ? (S_IRUSR | S_IWUSR) :
 getInt(argv[optind], GN_BASE_8, "octal-perms");
 msqid = msgget(key, flags | perms);
 if (msqid == -1)
 errExit("msgget");
 printf("%d\n", msqid);
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_create.c
```

# **46.2 Trao Đổi Message**

Các system call `msgsnd()` và `msgrcv()` thực hiện I/O trên message queue. Đối số đầu tiên của cả hai system call (`msqid`) là một message queue identifier. Đối số thứ hai, `msgp`, là một pointer đến một structure do lập trình viên định nghĩa, dùng để chứa message đang được gửi hoặc nhận. Structure này có dạng tổng quát như sau:

```
struct mymsg {
 long mtype; /* Message type */
 char mtext[]; /* Message body */
}
```

Định nghĩa này thực ra chỉ là cách viết tắt để nói rằng phần đầu của một message chứa kiểu message, được chỉ định dưới dạng một số nguyên `long`, trong khi phần còn lại của message là một structure do lập trình viên định nghĩa với độ dài và nội dung tùy ý; nó không nhất thiết phải là một mảng ký tự. Do đó, đối số `msgp` được gõ kiểu là `void *` để nó có thể là pointer đến bất kỳ loại structure nào.

Một trường `mtext` có độ dài bằng không là hợp lệ, và đôi khi hữu ích nếu thông tin cần truyền đạt có thể được mã hóa hoàn toàn trong kiểu message, hoặc nếu sự tồn tại của một message tự nó đã là đủ thông tin cho process nhận.

## **46.2.1 Gửi Message**

System call `msgsnd()` ghi một message vào một message queue.

```
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
int msgsnd(int msqid, const void *msgp, size_t msgsz, int msgflg);
                                           Returns 0 on success, or –1 on error
```

Để gửi một message với `msgsnd()`, chúng ta phải đặt trường `mtype` của structure message thành một giá trị lớn hơn 0 (chúng ta sẽ thấy cách giá trị này được sử dụng khi thảo luận về `msgrcv()` trong phần tiếp theo) và sao chép thông tin mong muốn vào trường `mtext` do lập trình viên định nghĩa. Đối số `msgsz` chỉ định số byte chứa trong trường `mtext`.

> Khi gửi message với `msgsnd()`, không có khái niệm về việc ghi một phần như với `write()`. Đây là lý do tại sao một `msgsnd()` thành công chỉ cần trả về 0, thay vì số byte đã gửi.

Đối số cuối cùng, `msgflg`, là một bitmask các flag điều khiển hoạt động của `msgsnd()`. Chỉ có một flag như vậy được định nghĩa:

#### `IPC_NOWAIT`

Thực hiện gửi không chặn (nonblocking send). Thông thường, nếu một message queue đầy, `msgsnd()` sẽ block cho đến khi có đủ không gian để message được đặt vào queue. Tuy nhiên, nếu flag này được chỉ định, thì `msgsnd()` trả về ngay lập tức với lỗi `EAGAIN`.

Một lời gọi `msgsnd()` bị block do queue đầy có thể bị ngắt bởi một signal handler. Trong trường hợp đó, `msgsnd()` luôn thất bại với lỗi `EINTR`. (Như đã lưu ý trong Mục 21.5, `msgsnd()` nằm trong số những system call không bao giờ được tự động restart, bất kể cài đặt của flag `SA_RESTART` khi signal handler được thiết lập.)

Ghi một message vào một message queue yêu cầu quyền ghi (write permission) trên queue.

Chương trình trong Listing 46-2 cung cấp một giao diện dòng lệnh cho system call `msgsnd()`. Định dạng dòng lệnh được chấp nhận bởi chương trình này được hiển thị trong hàm `usageError()`. Lưu ý rằng chương trình này không sử dụng system call `msgget()`. (Chúng ta đã lưu ý rằng một process không cần sử dụng lời gọi get để truy cập một đối tượng IPC trong Mục 45.1.) Thay vào đó, chúng ta chỉ định message queue bằng cách cung cấp identifier của nó như một đối số dòng lệnh. Chúng ta sẽ minh họa việc sử dụng chương trình này trong Mục 46.2.2.

**Listing 46-2:** Sử dụng `msgsnd()` để gửi một message

––––––––––––––––––––––––––––––––––––––––––––––––––––––– **svmsg/svmsg\_send.c** #include <sys/types.h> #include <sys/msg.h> #include "tlpi\_hdr.h" #define MAX\_MTEXT 1024 struct mbuf { long mtype; /\* Message type \*/ char mtext[MAX\_MTEXT]; /\* Message body \*/ };

```
static void /* Print (optional) message, then usage description */
usageError(const char *progName, const char *msg)
{
 if (msg != NULL)
 fprintf(stderr, "%s", msg);
 fprintf(stderr, "Usage: %s [-n] msqid msg-type [msg-text]\n", progName);
 fprintf(stderr, " -n Use IPC_NOWAIT flag\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int msqid, flags, msgLen;
 struct mbuf msg; /* Message buffer for msgsnd() */
 int opt; /* Option character from getopt() */
 /* Parse command-line options and arguments */
 flags = 0;
 while ((opt = getopt(argc, argv, "n")) != -1) {
 if (opt == 'n')
 flags |= IPC_NOWAIT;
 else
 usageError(argv[0], NULL);
 }
 if (argc < optind + 2 || argc > optind + 3)
 usageError(argv[0], "Wrong number of arguments\n");
 msqid = getInt(argv[optind], 0, "msqid");
 msg.mtype = getInt(argv[optind + 1], 0, "msg-type");
 if (argc > optind + 2) { /* 'msg-text' was supplied */
 msgLen = strlen(argv[optind + 2]) + 1;
 if (msgLen > MAX_MTEXT)
 cmdLineErr("msg-text too long (max: %d characters)\n", MAX_MTEXT);
 memcpy(msg.mtext, argv[optind + 2], msgLen);
 } else { /* No 'msg-text' ==> zero-length msg */
 msgLen = 0;
 }
 /* Send message */
 if (msgsnd(msqid, &msg, msgLen, flags) == -1)
 errExit("msgsnd");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_send.c
```

## **46.2.2 Nhận Message**

System call `msgrcv()` đọc (và xóa) một message từ một message queue, và sao chép nội dung của nó vào buffer được trỏ bởi `msgp`.

```
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
ssize_t msgrcv(int msqid, void *msgp, size_t maxmsgsz, long msgtyp, int msgflg);
                Returns number of bytes copied into mtext field, or –1 on error
```

Không gian tối đa có sẵn trong trường `mtext` của buffer `msgp` được chỉ định bởi đối số `maxmsgsz`. Nếu phần thân của message cần lấy ra khỏi queue vượt quá `maxmsgsz` byte, thì không có message nào bị xóa khỏi queue, và `msgrcv()` thất bại với lỗi `E2BIG`. (Hành vi mặc định này có thể được thay đổi bằng flag `MSG_NOERROR` được mô tả ngay sau đây.)

Các message không nhất thiết phải được đọc theo thứ tự chúng được gửi. Thay vào đó, chúng ta có thể chọn message theo giá trị trong trường `mtype`. Sự lựa chọn này được điều khiển bởi đối số `msgtyp`, như sau:

- Nếu `msgtyp` bằng 0, message đầu tiên từ queue sẽ bị xóa và trả về cho process gọi.
- Nếu `msgtyp` lớn hơn 0, message đầu tiên trong queue có `mtype` bằng `msgtyp` sẽ bị xóa và trả về cho process gọi. Bằng cách chỉ định các giá trị khác nhau cho `msgtyp`, nhiều process có thể đọc từ một message queue mà không tranh giành để đọc cùng một message. Một kỹ thuật hữu ích là để mỗi process chọn những message có kiểu khớp với process ID của nó.
- Nếu `msgtyp` nhỏ hơn 0, hãy xem các message đang chờ như một priority queue. Message đầu tiên có `mtype` thấp nhất, nhỏ hơn hoặc bằng giá trị tuyệt đối của `msgtyp`, sẽ bị xóa và trả về cho process gọi.

Một ví dụ giúp làm rõ hành vi khi `msgtyp` nhỏ hơn 0. Giả sử chúng ta có một message queue chứa chuỗi message được hiển thị trong Hình 46-1 và chúng ta thực hiện một loạt các lời gọi `msgrcv()` có dạng sau:

```
msgrcv(id, &msg, maxmsgsz, -300, 0);
```

Các lời gọi `msgrcv()` này sẽ lấy các message theo thứ tự 2 (kiểu 100), 5 (kiểu 100), 3 (kiểu 200), và 1 (kiểu 300). Một lời gọi tiếp theo sẽ bị block, vì kiểu của message còn lại (400) vượt quá 300.

Đối số `msgflg` là một bitmask được tạo bằng cách OR không hoặc nhiều flag sau:

#### `IPC_NOWAIT`

Thực hiện nhận không chặn (nonblocking receive). Thông thường, nếu không có message nào khớp với `msgtyp` trong queue, `msgrcv()` block cho đến khi có một message như vậy. Chỉ định flag `IPC_NOWAIT` khiến `msgrcv()` thay vào đó trả về ngay lập tức với lỗi `ENOMSG`. (Lỗi `EAGAIN` sẽ nhất quán hơn, như xảy ra khi `msgsnd()` không chặn hoặc đọc không chặn từ một FIFO. Tuy nhiên, thất bại với `ENOMSG` là hành vi lịch sử và được SUSv3 yêu cầu.)

#### `MSG_EXCEPT`

Flag này chỉ có hiệu lực nếu `msgtyp` lớn hơn 0, trong trường hợp đó nó buộc thực hiện bổ sung của hoạt động thông thường; nghĩa là, message đầu tiên từ queue có `mtype` không bằng `msgtyp` sẽ bị xóa khỏi queue và trả về cho caller. Flag này đặc thù Linux, và chỉ được cung cấp từ `<sys/msg.h>` nếu `_GNU_SOURCE` được định nghĩa. Thực hiện một loạt các lời gọi có dạng `msgrcv(id, &msg, maxmsgsz, 100, MSG_EXCEPT)` trên message queue được hiển thị trong Hình 46-1 sẽ lấy các message theo thứ tự 1, 3, 4, và sau đó block.

#### `MSG_NOERROR`

Theo mặc định, nếu kích thước của trường `mtext` của message vượt quá không gian có sẵn (theo định nghĩa bởi đối số `maxmsgsz`), `msgrcv()` thất bại. Nếu flag `MSG_NOERROR` được chỉ định, thì `msgrcv()` thay vào đó sẽ xóa message khỏi queue, cắt ngắn trường `mtext` xuống còn `maxmsgsz` byte, và trả về cho caller. Dữ liệu bị cắt ngắn sẽ bị mất.

Khi hoàn thành thành công, `msgrcv()` trả về kích thước của trường `mtext` của message được nhận; khi có lỗi, –1 được trả về.

| Vị trí trong queue | Kiểu message (mtype) | Phần thân message (mtext) |
|--------------------|----------------------|---------------------------|
| 1                  | 300                  |                           |
| 2                  | 100                  |                           |
| 3                  | 200                  |                           |
| 4                  | 400                  |                           |
| 5                  | 100                  |                           |

**Hình 46-1:** Ví dụ về một message queue chứa các message thuộc các kiểu khác nhau

Giống như `msgsnd()`, nếu một lời gọi `msgrcv()` đang bị block bị ngắt bởi một signal handler, thì lời gọi thất bại với lỗi `EINTR`, bất kể cài đặt của flag `SA_RESTART` khi signal handler được thiết lập.

Đọc một message từ một message queue yêu cầu quyền đọc (read permission) trên queue.

### **Chương trình ví dụ**

Chương trình trong Listing 46-3 cung cấp một giao diện dòng lệnh cho system call `msgrcv()`. Định dạng dòng lệnh được chấp nhận bởi chương trình này được hiển thị trong hàm `usageError()`. Giống như chương trình trong Listing 46-2, đã minh họa việc sử dụng `msgsnd()`, chương trình này không sử dụng system call `msgget()`, mà thay vào đó mong đợi một message queue identifier làm đối số dòng lệnh của nó.

Phiên shell sau minh họa việc sử dụng các chương trình trong Listing 46-1, Listing 46-2 và Listing 46-3. Chúng ta bắt đầu bằng cách tạo một message queue sử dụng key `IPC_PRIVATE`, sau đó ghi ba message với các kiểu khác nhau vào queue:

```
$ ./svmsg_create -p
32769 ID of message queue
$ ./svmsg_send 32769 20 "I hear and I forget."
$ ./svmsg_send 32769 10 "I see and I remember."
$ ./svmsg_send 32769 30 "I do and I understand."
```

Sau đó chúng ta sử dụng chương trình trong Listing 46-3 để đọc các message có kiểu nhỏ hơn hoặc bằng 20 từ queue:

```
$ ./svmsg_receive -t -20 32769
Received: type=10; length=22; body=I see and I remember.
$ ./svmsg_receive -t -20 32769
Received: type=20; length=21; body=I hear and I forget.
$ ./svmsg_receive -t -20 32769
```

Lệnh cuối cùng trong số các lệnh trên đã bị block, vì không có message nào trong queue có kiểu nhỏ hơn hoặc bằng 20. Vì vậy, chúng ta tiếp tục bằng cách gõ Control-C để kết thúc lệnh, sau đó thực hiện một lệnh đọc message thuộc bất kỳ kiểu nào từ queue:

```
Type Control-C to terminate program
$ ./svmsg_receive 32769
Received: type=30; length=23; body=I do and I understand.
```

**Listing 46-3:** Sử dụng `msgrcv()` để đọc một message

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_receive.c
#define _GNU_SOURCE /* Get definition of MSG_EXCEPT */
#include <sys/types.h>
#include <sys/msg.h>
#include "tlpi_hdr.h"
#define MAX_MTEXT 1024
struct mbuf {
 long mtype; /* Message type */
 char mtext[MAX_MTEXT]; /* Message body */
};
static void
usageError(const char *progName, const char *msg)
{
 if (msg != NULL)
 fprintf(stderr, "%s", msg);
 fprintf(stderr, "Usage: %s [options] msqid [max-bytes]\n", progName);
 fprintf(stderr, "Permitted options are:\n");
 fprintf(stderr, " -e Use MSG_NOERROR flag\n");
 fprintf(stderr, " -t type Select message of given type\n");
 fprintf(stderr, " -n Use IPC_NOWAIT flag\n");
```

```
#ifdef MSG_EXCEPT
 fprintf(stderr, " -x Use MSG_EXCEPT flag\n");
#endif
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int msqid, flags, type;
 ssize_t msgLen;
 size_t maxBytes;
 struct mbuf msg; /* Message buffer for msgrcv() */
 int opt; /* Option character from getopt() */
 /* Parse command-line options and arguments */
 flags = 0;
 type = 0;
 while ((opt = getopt(argc, argv, "ent:x")) != -1) {
 switch (opt) {
 case 'e': flags |= MSG_NOERROR; break;
 case 'n': flags |= IPC_NOWAIT; break;
 case 't': type = atoi(optarg); break;
#ifdef MSG_EXCEPT
 case 'x': flags |= MSG_EXCEPT; break;
#endif
 default: usageError(argv[0], NULL);
 }
 }
 if (argc < optind + 1 || argc > optind + 2)
 usageError(argv[0], "Wrong number of arguments\n");
 msqid = getInt(argv[optind], 0, "msqid");
 maxBytes = (argc > optind + 1) ?
 getInt(argv[optind + 1], 0, "max-bytes") : MAX_MTEXT;
 /* Get message and display on stdout */
 msgLen = msgrcv(msqid, &msg, maxBytes, type, flags);
 if (msgLen == -1)
 errExit("msgrcv");
 printf("Received: type=%ld; length=%ld", msg.mtype, (long) msgLen);
 if (msgLen > 0)
 printf("; body=%s", msg.mtext);
 printf("\n");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_receive.c
```

# **46.3 Các Thao Tác Điều Khiển Message Queue**

System call `msgctl()` thực hiện các thao tác điều khiển trên message queue được xác định bởi `msqid`.

```
#include <sys/types.h> /* For portability */
#include <sys/msg.h>
int msgctl(int msqid, int cmd, struct msqid_ds *buf);
                                           Returns 0 on success, or –1 on error
```

Đối số `cmd` chỉ định thao tác cần thực hiện trên queue. Nó có thể là một trong những giá trị sau:

#### `IPC_RMID`

Ngay lập tức xóa đối tượng message queue và cấu trúc dữ liệu `msqid_ds` liên quan. Tất cả các message còn lại trong queue sẽ bị mất, và mọi process reader hoặc writer đang bị block sẽ được đánh thức ngay lập tức, với `msgsnd()` hoặc `msgrcv()` thất bại với lỗi `EIDRM`. Đối số thứ ba của `msgctl()` bị bỏ qua cho thao tác này.

#### `IPC_STAT`

Đặt một bản sao của cấu trúc dữ liệu `msqid_ds` liên kết với message queue này vào buffer được trỏ bởi `buf`. Chúng ta mô tả cấu trúc `msqid_ds` trong Mục 46.4.

#### `IPC_SET`

Cập nhật các trường được chọn của cấu trúc dữ liệu `msqid_ds` liên kết với message queue này bằng cách sử dụng các giá trị được cung cấp trong buffer được trỏ bởi `buf`.

Chi tiết thêm về các thao tác này, bao gồm các đặc quyền và quyền hạn được yêu cầu bởi process gọi, được mô tả trong Mục 45.3. Chúng ta mô tả một số giá trị khác cho `cmd` trong Mục 46.6.

Chương trình trong Listing 46-4 minh họa việc sử dụng `msgctl()` để xóa một message queue.

**Listing 46-4:** Xóa các message queue System V

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_rm.c
#include <sys/types.h>
#include <sys/msg.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int j;
```

```
 if (argc > 1 && strcmp(argv[1], "--help") == 0)
 usageErr("%s [msqid...]\n", argv[0]);
 for (j = 1; j < argc; j++)
 if (msgctl(getInt(argv[j], 0, "msqid"), IPC_RMID, NULL) == -1)
 errExit("msgctl %s", argv[j]);
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_rm.c
```

# **46.4 Cấu Trúc Dữ Liệu Liên Kết Với Message Queue**

Mỗi message queue có một cấu trúc dữ liệu `msqid_ds` liên kết với dạng sau:

```
struct msqid_ds {
 struct ipc_perm msg_perm; /* Ownership and permissions */
 time_t msg_stime; /* Time of last msgsnd() */
 time_t msg_rtime; /* Time of last msgrcv() */
 time_t msg_ctime; /* Time of last change */
 unsigned long __msg_cbytes; /* Number of bytes in queue */
 msgqnum_t msg_qnum; /* Number of messages in queue */
 msglen_t msg_qbytes; /* Maximum bytes in queue */
 pid_t msg_lspid; /* PID of last msgsnd() */
 pid_t msg_lrpid; /* PID of last msgrcv() */
};
```

Mục đích của cách viết tắt `msq` trong tên `msqid_ds` là để gây nhầm lẫn cho lập trình viên. Đây là giao diện message queue duy nhất sử dụng cách viết tắt này.

Các kiểu dữ liệu `msgqnum_t` và `msglen_t` — được dùng để gõ kiểu cho các trường `msg_qnum` và `msg_qbytes` — là các kiểu số nguyên không dấu được chỉ định trong SUSv3.

Các trường của cấu trúc `msqid_ds` được cập nhật ngầm định bởi các system call message queue khác nhau, và một số trường nhất định có thể được cập nhật tường minh bằng thao tác `IPC_SET` của `msgctl()`. Chi tiết như sau:

```
msg_perm
```

Khi message queue được tạo, các trường của substructure này được khởi tạo như mô tả trong Mục 45.3. Các subfield `uid`, `gid` và `mode` có thể được cập nhật thông qua `IPC_SET`.

`msg_stime`

Khi queue được tạo, trường này được đặt thành 0; mỗi `msgsnd()` thành công sau đó đặt trường này thành thời gian hiện tại. Trường này và các trường timestamp khác trong cấu trúc `msqid_ds` được gõ kiểu là `time_t`; chúng lưu thời gian tính bằng giây kể từ Epoch.

`msg_rtime`

Trường này được đặt thành 0 khi message queue được tạo, và sau đó được đặt thành thời gian hiện tại sau mỗi `msgrcv()` thành công.

#### `msg_ctime`

Trường này được đặt thành thời gian hiện tại khi message queue được tạo và bất cứ khi nào thao tác `IPC_SET` được thực hiện thành công.

#### `__msg_cbytes`

Trường này được đặt thành 0 khi message queue được tạo, và sau đó được điều chỉnh trong mỗi `msgsnd()` và `msgrcv()` thành công để phản ánh tổng số byte chứa trong các trường `mtext` của tất cả các message trong queue.

#### `msg_qnum`

Khi message queue được tạo, trường này được đặt thành 0. Sau đó nó được tăng lên bởi mỗi `msgsnd()` thành công và được giảm bởi mỗi `msgrcv()` thành công để phản ánh tổng số message trong queue.

#### `msg_qbytes`

Giá trị trong trường này xác định giới hạn trên số byte trong các trường `mtext` của tất cả các message trong message queue. Trường này được khởi tạo theo giá trị của giới hạn `MSGMNB` khi queue được tạo. Một process có đặc quyền (`CAP_SYS_RESOURCE`) có thể sử dụng thao tác `IPC_SET` để điều chỉnh `msg_qbytes` đến bất kỳ giá trị nào trong khoảng từ 0 đến `INT_MAX` (2,147,483,647 trên các nền tảng 32-bit) byte. Một process không có đặc quyền có thể điều chỉnh `msg_qbytes` đến bất kỳ giá trị nào trong khoảng từ 0 đến `MSGMNB`. Một user có đặc quyền có thể sửa đổi giá trị chứa trong file `/proc/sys/kernel/msgmnb` đặc thù Linux để thay đổi cài đặt `msg_qbytes` ban đầu cho tất cả các message queue được tạo sau đó, cũng như giới hạn trên cho các thay đổi tiếp theo đối với `msg_qbytes` bởi các process không có đặc quyền. Chúng ta nói thêm về các giới hạn message queue trong Mục 46.5.

#### `msg_lspid`

Trường này được đặt thành 0 khi queue được tạo, và sau đó được đặt thành process ID của process đang gọi sau mỗi `msgsnd()` thành công.

#### `msg_lrpid`

Trường này được đặt thành 0 khi message queue được tạo, và sau đó được đặt thành process ID của process đang gọi sau mỗi `msgrcv()` thành công.

Tất cả các trường trên được chỉ định bởi SUSv3, ngoại trừ `__msg_cbytes`. Tuy nhiên, hầu hết các hệ thống UNIX đều cung cấp một trường tương đương với trường `__msg_cbytes`.

Chương trình trong Listing 46-5 minh họa việc sử dụng các thao tác `IPC_STAT` và `IPC_SET` để sửa đổi cài đặt `msg_qbytes` của một message queue.

**Listing 46-5:** Thay đổi cài đặt `msg_qbytes` của một message queue System V

–––––––––––––––––––––––––––––––––––––––––––––––––––– **svmsg/svmsg\_chqbytes.c** #include <sys/types.h> #include <sys/msg.h> #include "tlpi\_hdr.h" int main(int argc, char \*argv[]) { struct msqid\_ds ds; int msqid;

```
 if (argc != 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s msqid max-bytes\n", argv[0]);
 /* Retrieve copy of associated data structure from kernel */
 msqid = getInt(argv[1], 0, "msqid");
 if (msgctl(msqid, IPC_STAT, &ds) == -1)
 errExit("msgctl");
 ds.msg_qbytes = getInt(argv[2], 0, "max-bytes");
 /* Update associated data structure in kernel */
 if (msgctl(msqid, IPC_SET, &ds) == -1)
 errExit("msgctl");
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_chqbytes.c
```

# **46.5 Các Giới Hạn Message Queue**

Hầu hết các hệ thống UNIX đặt ra nhiều giới hạn khác nhau đối với hoạt động của message queue System V. Ở đây, chúng ta mô tả các giới hạn trên Linux và lưu ý một vài điểm khác biệt so với các hệ thống UNIX khác.

Các giới hạn sau được thực thi trên Linux. System call bị ảnh hưởng bởi giới hạn và lỗi xảy ra khi giới hạn đạt được được ghi chú trong ngoặc đơn.

`MSGMNI`

Đây là giới hạn toàn hệ thống về số lượng message queue identifier (nói cách khác, message queue) có thể được tạo. (`msgget()`, `ENOSPC`)

`MSGMAX`

Đây là giới hạn toàn hệ thống chỉ định số byte (`mtext`) tối đa có thể được ghi trong một message duy nhất. (`msgsnd()`, `EINVAL`)

`MSGMNB`

Đây là số byte (`mtext`) tối đa có thể được chứa trong một message queue tại một thời điểm. Giới hạn này là một thông số toàn hệ thống được sử dụng để khởi tạo trường `msg_qbytes` của cấu trúc dữ liệu `msqid_ds` liên kết với message queue này. Sau đó, giá trị `msg_qbytes` có thể được sửa đổi trên cơ sở từng queue, như mô tả trong Mục 46.4. Nếu giới hạn `msg_qbytes` của queue đạt đến, thì `msgsnd()` block, hoặc thất bại với lỗi `EAGAIN` nếu `IPC_NOWAIT` được đặt.

Một số hệ thống UNIX cũng định nghĩa các giới hạn thêm sau:

`MSGTQL`

Đây là giới hạn toàn hệ thống về số lượng message có thể được đặt trên tất cả các message queue trong hệ thống.

`MSGPOOL`

Đây là giới hạn toàn hệ thống về kích thước của buffer pool được sử dụng để chứa dữ liệu trong tất cả các message queue trong hệ thống.

Mặc dù Linux không áp đặt bất kỳ giới hạn nào trong hai giới hạn trên, nhưng nó giới hạn số lượng message trên một queue riêng lẻ theo giá trị được chỉ định bởi cài đặt `msg_qbytes` của queue. Giới hạn này chỉ có liên quan nếu chúng ta đang ghi các message có độ dài bằng không vào một queue. Nó có tác dụng là giới hạn về số lượng message có độ dài bằng không bằng với giới hạn về số lượng message 1 byte có thể được ghi vào queue. Điều này là cần thiết để ngăn chặn việc ghi vô hạn các message có độ dài bằng không vào queue. Mặc dù chúng không chứa dữ liệu, nhưng mỗi message có độ dài bằng không tiêu thụ một lượng bộ nhớ nhỏ cho chi phí bookkeeping của hệ thống.

Khi khởi động hệ thống, các giới hạn message queue được đặt thành các giá trị mặc định. Các mặc định này đã thay đổi phần nào qua các phiên bản kernel. (Một số kernel của nhà phân phối đặt các mặc định khác so với các kernel vanilla.) Trên Linux, các giới hạn có thể được xem hoặc thay đổi thông qua các file trong hệ thống file `/proc`. Bảng 46-1 hiển thị file `/proc` tương ứng với mỗi giới hạn. Ví dụ, đây là các giới hạn mặc định mà chúng ta thấy cho Linux 2.6.31 trên một hệ thống x86-32:

```
$ cd /proc/sys/kernel
$ cat msgmni
748
$ cat msgmax
8192
$ cat msgmnb
16384
```

**Bảng 46-1:** Các giới hạn message queue System V

| Giới hạn | Giá trị trần (x86-32)       | File tương ứng trong /proc/sys/kernel |
|----------|-----------------------------|---------------------------------------|
| MSGMNI   | 32768 (IPCMNI)              | msgmni                                |
| MSGMAX   | Phụ thuộc vào bộ nhớ có sẵn | msgmax                                |
| MSGMNB   | 2147483647 (INT_MAX)        | msgmnb                                |

Cột giá trị trần của Bảng 46-1 hiển thị giá trị tối đa mà mỗi giới hạn có thể được nâng lên trên kiến trúc x86-32. Lưu ý rằng mặc dù giới hạn `MSGMNB` có thể được nâng lên đến giá trị `INT_MAX`, nhưng một số giới hạn khác (ví dụ: thiếu bộ nhớ) sẽ đạt đến trước khi một message queue có thể được tải đầy nhiều dữ liệu như vậy.

Thao tác `IPC_INFO` của `msgctl()` đặc thù Linux lấy một cấu trúc kiểu `msginfo`, chứa các giá trị của các giới hạn message queue khác nhau:

```
struct msginfo buf;
msgctl(0, IPC_INFO, (struct msqid_ds *) &buf);
```

Chi tiết về `IPC_INFO` và cấu trúc `msginfo` có thể tìm thấy trong trang hướng dẫn `msgctl(2)`.

# **46.6 Hiển Thị Tất Cả Message Queue Trong Hệ Thống**

Trong Mục 45.7, chúng ta đã xem xét một cách để lấy danh sách tất cả các đối tượng IPC trong hệ thống: thông qua một tập hợp các file trong hệ thống file `/proc`. Bây giờ chúng ta xem xét một phương pháp thứ hai để lấy thông tin tương tự: thông qua một tập hợp các thao tác IPC ctl đặc thù Linux (`msgctl()`, `semctl()` và `shmctl()`). (Chương trình `ipcs` sử dụng các thao tác này.) Các thao tác này như sau:

`MSG_INFO`, `SEM_INFO` và `SHM_INFO`: Thao tác `MSG_INFO` phục vụ hai mục đích. Thứ nhất, nó trả về một cấu trúc chi tiết về tài nguyên được tiêu thụ bởi tất cả các message queue trong hệ thống. Thứ hai, là kết quả hàm của lời gọi ctl, nó trả về chỉ số của mục tối đa trong mảng entries trỏ đến các cấu trúc dữ liệu cho các đối tượng message queue (xem Hình 45-1, trang 932). Các thao tác `SEM_INFO` và `SHM_INFO` thực hiện nhiệm vụ tương tự cho các semaphore set và shared memory segment, tương ứng. Chúng ta phải định nghĩa macro feature test `_GNU_SOURCE` để lấy định nghĩa của ba hằng số này từ các file header IPC System V tương ứng.

> Một ví dụ về việc sử dụng `MSG_INFO` để lấy một cấu trúc `msginfo` chứa thông tin về các tài nguyên được sử dụng bởi tất cả các đối tượng message queue được cung cấp trong file `svmsg/svmsg_info.c` trong bản phân phối mã nguồn của cuốn sách này.

`MSG_STAT`, `SEM_STAT` và `SHM_STAT`: Giống như thao tác `IPC_STAT`, các thao tác này lấy cấu trúc dữ liệu liên kết với một đối tượng IPC. Tuy nhiên, chúng khác nhau ở hai điểm. Thứ nhất, thay vì mong đợi một IPC identifier làm đối số đầu tiên của lời gọi ctl, các thao tác này mong đợi một chỉ số vào mảng entries. Thứ hai, nếu thao tác thành công, thì, là kết quả hàm của nó, lời gọi ctl trả về identifier của đối tượng IPC tương ứng với chỉ số đó. Chúng ta phải định nghĩa macro feature test `_GNU_SOURCE` để lấy định nghĩa của ba hằng số này từ các file header IPC System V tương ứng.

Để liệt kê tất cả các message queue trong hệ thống, chúng ta có thể làm như sau:

1. Sử dụng thao tác `MSG_INFO` để tìm ra chỉ số tối đa (`maxind`) của mảng entries cho message queue.
2. Thực hiện một vòng lặp cho tất cả các giá trị từ 0 đến `maxind` bao gồm, sử dụng thao tác `MSG_STAT` cho mỗi giá trị. Trong vòng lặp này, chúng ta bỏ qua các lỗi có thể xảy ra nếu một mục của mảng entries trống (`EINVAL`) hoặc nếu chúng ta không có quyền trên đối tượng mà nó tham chiếu đến (`EACCES`).

Listing 46-6 cung cấp một cài đặt của các bước trên cho message queue. Phiên shell log sau minh họa việc sử dụng chương trình này:

## $ **./svmsg\_ls** maxind: 4 index ID key messages 2 98306 0x00000000 0 4 163844 0x000004d2 2 \$ **ipcs -q** Check above against output of ipcs ------ Message Queues ------- key msqid owner perms used-bytes messages 0x00000000 98306 mtk 600 0 0 0x000004d2 163844 mtk 600 12 2

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_ls.c
#define _GNU_SOURCE
#include <sys/types.h>
#include <sys/msg.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int maxind, ind, msqid;
 struct msqid_ds ds;
 struct msginfo msginfo;
 /* Obtain size of kernel 'entries' array */
 maxind = msgctl(0, MSG_INFO, (struct msqid_ds *) &msginfo);
 if (maxind == -1)
 errExit("msgctl-MSG_INFO");
 printf("maxind: %d\n\n", maxind);
 printf("index id key messages\n");
 /* Retrieve and display information from each element of 'entries' array */
 for (ind = 0; ind <= maxind; ind++) {
 msqid = msgctl(ind, MSG_STAT, &ds);
 if (msqid == -1) {
 if (errno != EINVAL && errno != EACCES)
 errMsg("msgctl-MSG_STAT"); /* Unexpected error */
 continue; /* Ignore this item */
 }
 printf("%4d %8d 0x%08lx %7ld\n", ind, msqid,
 (unsigned long) ds.msg_perm.__key, (long) ds.msg_qnum);
 }
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_ls.c
```

## **46.7 Lập Trình Client-Server với Message Queue**

Trong phần này, chúng ta xem xét hai trong số các thiết kế khả thi cho ứng dụng client-server sử dụng message queue System V:

- Sử dụng một message queue duy nhất để trao đổi message theo cả hai chiều giữa server và client.
- Sử dụng các message queue riêng biệt cho server và cho từng client. Queue của server được sử dụng để nhận các yêu cầu client đến, và các phản hồi được gửi đến client thông qua các queue client riêng lẻ.

Cách tiếp cận nào chúng ta chọn phụ thuộc vào các yêu cầu của ứng dụng. Tiếp theo chúng ta xem xét một số yếu tố có thể ảnh hưởng đến lựa chọn của chúng ta.

## **Sử dụng một message queue duy nhất cho server và client**

Sử dụng một message queue duy nhất có thể phù hợp khi các message được trao đổi giữa server và client có kích thước nhỏ. Tuy nhiên, lưu ý những điểm sau:

- Vì nhiều process có thể cố gắng đọc message cùng lúc, chúng ta phải sử dụng trường kiểu message (`mtype`) để cho phép mỗi process chỉ chọn những message dành riêng cho nó. Một cách để thực hiện điều này là sử dụng process ID của client làm kiểu message cho các message được gửi từ server đến client. Client có thể gửi process ID của nó như một phần trong message của nó đến server. Hơn nữa, các message đến server cũng phải được phân biệt bởi một kiểu message duy nhất. Cho mục đích này, chúng ta có thể sử dụng số 1, là process ID của process `init` chạy liên tục, không bao giờ có thể là process ID của một client process. (Một phương án thay thế sẽ là sử dụng process ID của server làm kiểu message; tuy nhiên, thật khó để client lấy thông tin này.) Sơ đồ đánh số này được hiển thị trong Hình 46-2.
- Message queue có dung lượng hạn chế. Điều này có khả năng gây ra một số vấn đề. Một trong số đó là nhiều client đồng thời có thể lấp đầy message queue, dẫn đến tình huống deadlock, trong đó không có yêu cầu client mới nào có thể được gửi và server bị block khỏi việc ghi bất kỳ phản hồi nào. Vấn đề khác là một client hoạt động không tốt hoặc có ý định độc hại có thể không đọc các phản hồi từ server. Điều này có thể dẫn đến việc queue bị tắc nghẽn bởi các message chưa đọc, ngăn bất kỳ giao tiếp nào giữa client và server. (Sử dụng hai queue — một cho các message từ client đến server, và một cho các message từ server đến client — sẽ giải quyết vấn đề đầu tiên trong số này, nhưng không phải vấn đề thứ hai.)

![](_page_77_Figure_5.jpeg)

**Hình 46-2:** Sử dụng một message queue duy nhất cho IPC client-server

### **Sử dụng một message queue mỗi client**

Sử dụng một message queue mỗi client (cũng như một cho server) là tốt hơn khi cần trao đổi các message lớn, hoặc khi có khả năng xảy ra các vấn đề được liệt kê ở trên khi sử dụng một message queue duy nhất. Lưu ý những điểm sau liên quan đến cách tiếp cận này:

- Mỗi client phải tạo queue message của riêng nó (thường sử dụng key `IPC_PRIVATE`) và thông báo cho server về identifier của queue, thường bằng cách truyền identifier như một phần của message của client đến server.
- Có một giới hạn toàn hệ thống (`MSGMNI`) về số lượng message queue, và giá trị mặc định cho giới hạn này khá thấp trên một số hệ thống. Nếu chúng ta mong đợi có một số lượng lớn client đồng thời, chúng ta có thể cần phải nâng giới hạn này.
- Server nên cho phép khả năng rằng queue message của client có thể không còn tồn tại nữa (có thể vì client đã xóa nó sớm).

Chúng ta sẽ nói thêm về việc sử dụng một message queue mỗi client trong phần tiếp theo.

## 46.8 Ứng Dụng File-Server Sử Dụng Message Queue

Trong phần này, chúng ta mô tả một ứng dụng client-server sử dụng một message queue mỗi client. Ứng dụng là một file server đơn giản. Client gửi một message yêu cầu đến message queue của server, yêu cầu nội dung của một file có tên. Server phản hồi bằng cách trả về nội dung file dưới dạng một loạt message đến message queue riêng tư của client. Hình 46-3 cung cấp tổng quan về ứng dụng.

Vì server không thực hiện xác thực client, bất kỳ user nào có thể chạy client đều có thể lấy bất kỳ file nào mà server có thể truy cập. Một server phức tạp hơn sẽ yêu cầu một số loại xác thực từ client trước khi phục vụ file được yêu cầu.

![](_page_78_Figure_8.jpeg)

**Hình 46-3:** IPC client-server sử dụng một message queue mỗi client

#### File header chung

Listing 46-7 là file header được include bởi cả server và client. File header này định nghĩa key đã biết sẽ được sử dụng cho message queue của server (`SERVER_KEY`), và định nghĩa định dạng của các message được truyền giữa client và server.

Cấu trúc `requestMsg` định nghĩa định dạng của yêu cầu được gửi từ client đến server. Trong cấu trúc này, thành phần `mtext` bao gồm hai trường: identifier của message queue của client và tên đường dẫn của file được client yêu cầu. Hằng số `REQ_MSG_SIZE` tương đương với kích thước kết hợp của hai trường này và được sử dụng như đối số `msgsz` trong các lời gọi `msgsnd()` sử dụng cấu trúc này.

Cấu trúc `responseMsg` định nghĩa định dạng của các message phản hồi được gửi từ server trở lại client. Trường `mtype` được sử dụng trong các message phản hồi để cung cấp thông tin về nội dung message, như được định nghĩa bởi các hằng số `RESP_MT_*`.

**Listing 46-7:** File header cho `svmsg_file_server.c` và `svmsg_file_client.c`

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_file.h
#include <sys/types.h>
#include <sys/msg.h>
#include <sys/stat.h>
#include <stddef.h> /* For definition of offsetof() */
#include <limits.h>
#include <fcntl.h>
#include <signal.h>
#include <sys/wait.h>
#include "tlpi_hdr.h"
#define SERVER_KEY 0x1aaaaaa1 /* Key for server's message queue */
struct requestMsg { /* Requests (client to server) */
 long mtype; /* Unused */
 int clientId; /* ID of client's message queue */
 char pathname[PATH_MAX]; /* File to be returned */
};
/* REQ_MSG_SIZE computes size of 'mtext' part of 'requestMsg' structure.
 We use offsetof() to handle the possibility that there are padding
 bytes between the 'clientId' and 'pathname' fields. */
#define REQ_MSG_SIZE (offsetof(struct requestMsg, pathname) - \
 offsetof(struct requestMsg, clientId) + PATH_MAX)
#define RESP_MSG_SIZE 8192
struct responseMsg { /* Responses (server to client) */
 long mtype; /* One of RESP_MT_* values below */
 char data[RESP_MSG_SIZE]; /* File content / response message */
};
/* Types for response messages sent from server to client */
#define RESP_MT_FAILURE 1 /* File couldn't be opened */
#define RESP_MT_DATA 2 /* Message contains file data */
#define RESP_MT_END 3 /* File data complete */
––––––––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_file.h
```

### **Chương trình server**

Listing 46-8 là chương trình server cho ứng dụng. Lưu ý những điểm sau về server:

- Server được thiết kế để xử lý các yêu cầu đồng thời. Thiết kế server đồng thời tốt hơn so với thiết kế tuần tự (iterative) được sử dụng trong Listing 44-7 (trang 912), vì chúng ta muốn tránh khả năng yêu cầu client cho một file lớn sẽ khiến tất cả các yêu cầu client khác phải chờ.
- Mỗi yêu cầu client được xử lý bằng cách tạo một child process phục vụ file được yêu cầu. Trong thời gian đó, process server chính chờ các yêu cầu client tiếp theo. Lưu ý những điểm sau về server child:
  - Vì child được tạo thông qua `fork()` kế thừa một bản sao của stack của parent, nó do đó lấy được một bản sao của message yêu cầu được đọc bởi process server chính.
  - Server child kết thúc sau khi xử lý yêu cầu client liên kết của nó.
- Để tránh tạo ra zombie process (Mục 26.2), server thiết lập một handler cho `SIGCHLD` và gọi `waitpid()` trong handler này.
- Lời gọi `msgrcv()` trong process server cha có thể bị block, và do đó bị ngắt bởi `SIGCHLD` handler. Để xử lý khả năng này, một vòng lặp được sử dụng để restart lời gọi nếu nó thất bại với lỗi `EINTR`.
- Server child thực thi hàm `serveRequest()`, gửi ba kiểu message trở lại cho client. Một yêu cầu có `mtype` là `RESP_MT_FAILURE` chỉ ra rằng server không thể mở file được yêu cầu; `RESP_MT_DATA` được sử dụng cho một loạt message chứa dữ liệu file; và `RESP_MT_END` (với trường dữ liệu có độ dài bằng không) được sử dụng để chỉ ra rằng việc truyền dữ liệu file đã hoàn tất.

Chúng ta xem xét một số cách để cải thiện và mở rộng chương trình server trong Bài tập 46-4.

**Listing 46-8:** Một file server sử dụng message queue System V

```
––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_file_server.c
  #include "svmsg_file.h"
  static void /* SIGCHLD handler */
  grimReaper(int sig)
  {
   int savedErrno;
   savedErrno = errno; /* waitpid() might change 'errno' */
q while (waitpid(-1, NULL, WNOHANG) > 0)
   continue;
   errno = savedErrno;
  }
```

```
static void /* Executed in child process: serve a single client */
w serveRequest(const struct requestMsg *req)
  {
   int fd;
   ssize_t numRead;
   struct responseMsg resp;
   fd = open(req->pathname, O_RDONLY);
   if (fd == -1) { /* Open failed: send error text */
e resp.mtype = RESP_MT_FAILURE;
   snprintf(resp.data, sizeof(resp.data), "%s", "Couldn't open");
   msgsnd(req->clientId, &resp, strlen(resp.data) + 1, 0);
   exit(EXIT_FAILURE); /* and terminate */
   }
   /* Transmit file contents in messages with type RESP_MT_DATA. We don't
   diagnose read() and msgsnd() errors since we can't notify client. */
r resp.mtype = RESP_MT_DATA;
   while ((numRead = read(fd, resp.data, RESP_MSG_SIZE)) > 0)
   if (msgsnd(req->clientId, &resp, numRead, 0) == -1)
   break;
   /* Send a message of type RESP_MT_END to signify end-of-file */
t resp.mtype = RESP_MT_END;
   msgsnd(req->clientId, &resp, 0, 0); /* Zero-length mtext */
  }
  int
  main(int argc, char *argv[])
  {
   struct requestMsg req;
   pid_t pid;
   ssize_t msgLen;
   int serverId;
   struct sigaction sa;
   /* Create server message queue */
   serverId = msgget(SERVER_KEY, IPC_CREAT | IPC_EXCL |
   S_IRUSR | S_IWUSR | S_IWGRP);
   if (serverId == -1)
   errExit("msgget");
   /* Establish SIGCHLD handler to reap terminated children */
   sigemptyset(&sa.sa_mask);
   sa.sa_flags = SA_RESTART;
   sa.sa_handler = grimReaper;
y if (sigaction(SIGCHLD, &sa, NULL) == -1)
   errExit("sigaction");
```

```
 /* Read requests, handle each in a separate child process */
   for (;;) {
   msgLen = msgrcv(serverId, &req, REQ_MSG_SIZE, 0, 0);
   if (msgLen == -1) {
u if (errno == EINTR) /* Interrupted by SIGCHLD handler? */
   continue; /* ... then restart msgrcv() */
   errMsg("msgrcv"); /* Some other error */
   break; /* ... so terminate loop */
   }
i pid = fork(); /* Create child process */
   if (pid == -1) {
   errMsg("fork");
   break;
   }
   if (pid == 0) { /* Child handles request */
   serveRequest(&req);
o _exit(EXIT_SUCCESS);
   }
   /* Parent loops to receive next client request */
   }
   /* If msgrcv() or fork() fails, remove server MQ and exit */
   if (msgctl(serverId, IPC_RMID, NULL) == -1)
   errExit("msgctl");
   exit(EXIT_SUCCESS);
  }
  ––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_file_server.c
```

## **Chương trình client**

Listing 46-9 là chương trình client cho ứng dụng. Lưu ý những điểm sau:

- Client tạo một message queue với key `IPC_PRIVATE` và sử dụng `atexit()` để thiết lập một exit handler nhằm đảm bảo rằng queue được xóa khi client thoát.
- Client truyền identifier cho queue của nó, cũng như tên đường dẫn của file cần phục vụ, trong một yêu cầu đến server.
- Client xử lý khả năng rằng message phản hồi đầu tiên được gửi bởi server có thể là thông báo lỗi (`mtype` bằng `RESP_MT_FAILURE`) bằng cách in văn bản của thông báo lỗi được trả về bởi server và thoát.
- Nếu file được mở thành công, thì client vòng lặp, nhận một loạt message chứa nội dung file (`mtype` bằng `RESP_MT_DATA`). Vòng lặp kết thúc khi nhận được message end-of-file (`mtype` bằng `RESP_MT_END`).

Client đơn giản này không xử lý các khả năng khác nhau do lỗi trong server. Chúng ta xem xét một số cải tiến trong Bài tập 46-5.

```
––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_file_client.c
  #include "svmsg_file.h"
  static int clientId;
  static void
  removeQueue(void)
  {
   if (msgctl(clientId, IPC_RMID, NULL) == -1)
q errExit("msgctl");
  }
  int
  main(int argc, char *argv[])
  {
   struct requestMsg req;
   struct responseMsg resp;
   int serverId, numMsgs;
   ssize_t msgLen, totBytes;
   if (argc != 2 || strcmp(argv[1], "--help") == 0)
   usageErr("%s pathname\n", argv[0]);
   if (strlen(argv[1]) > sizeof(req.pathname) - 1)
   cmdLineErr("pathname too long (max: %ld bytes)\n",
   (long) sizeof(req.pathname) - 1);
   /* Get server's queue identifier; create queue for response */
   serverId = msgget(SERVER_KEY, S_IWUSR);
   if (serverId == -1)
   errExit("msgget - server message queue");
w clientId = msgget(IPC_PRIVATE, S_IRUSR | S_IWUSR | S_IWGRP);
   if (clientId == -1)
   errExit("msgget - client message queue");
e if (atexit(removeQueue) != 0)
   errExit("atexit");
   /* Send message asking for file named in argv[1] */
   req.mtype = 1; /* Any type will do */
   req.clientId = clientId;
   strncpy(req.pathname, argv[1], sizeof(req.pathname) - 1);
   req.pathname[sizeof(req.pathname) - 1] = '\0';
   /* Ensure string is terminated */
r if (msgsnd(serverId, &req, REQ_MSG_SIZE, 0) == -1)
   errExit("msgsnd");
```

```
 /* Get first response, which may be failure notification */
   msgLen = msgrcv(clientId, &resp, RESP_MSG_SIZE, 0, 0);
   if (msgLen == -1)
   errExit("msgrcv");
t if (resp.mtype == RESP_MT_FAILURE) {
   printf("%s\n", resp.data); /* Display msg from server */
   if (msgctl(clientId, IPC_RMID, NULL) == -1)
   errExit("msgctl");
   exit(EXIT_FAILURE);
   }
   /* File was opened successfully by server; process messages
   (including the one already received) containing file data */
   totBytes = msgLen; /* Count first message */
y for (numMsgs = 1; resp.mtype == RESP_MT_DATA; numMsgs++) {
   msgLen = msgrcv(clientId, &resp, RESP_MSG_SIZE, 0, 0);
   if (msgLen == -1)
   errExit("msgrcv");
   totBytes += msgLen;
   }
   printf("Received %ld bytes (%d messages)\n", (long) totBytes, numMsgs);
   exit(EXIT_SUCCESS);
  }
  ––––––––––––––––––––––––––––––––––––––––––––––––– svmsg/svmsg_file_client.c
```

Phiên shell sau minh họa việc sử dụng các chương trình trong Listing 46-8 và Listing 46-9:

```
$ ./svmsg_file_server & Run server in background
[1] 9149
$ wc -c /etc/services Show size of file that client will request
764360 /etc/services
$ ./svmsg_file_client /etc/services 
Received 764360 bytes (95 messages) Bytes received matches size above
$ kill %1 Terminate server
[1]+ Terminated ./svmsg_file_server
```

# **46.9 Những Nhược Điểm của Message Queue System V**

Các hệ thống UNIX cung cấp một số cơ chế để truyền dữ liệu từ một process này sang process khác trên cùng một hệ thống, dưới dạng luồng byte không có ranh giới (pipe, FIFO và UNIX domain stream socket) hoặc dưới dạng message có ranh giới (message queue System V, POSIX message queue và UNIX domain datagram socket).

Một tính năng đặc biệt của message queue System V là khả năng gắn một kiểu số vào mỗi message. Điều này cho phép hai khả năng có thể hữu ích cho các ứng dụng: các process đọc có thể chọn message theo kiểu, hoặc chúng có thể sử dụng chiến lược priority-queue để các message có độ ưu tiên cao hơn (tức là những message có giá trị kiểu message thấp hơn) được đọc trước.

Tuy nhiên, message queue System V có một số nhược điểm:

- Message queue được tham chiếu bởi identifier, thay vì file descriptor được sử dụng bởi hầu hết các cơ chế I/O UNIX khác. Điều này có nghĩa là nhiều kỹ thuật I/O dựa trên file descriptor được mô tả trong Chương 63 (ví dụ: `select()`, `poll()` và `epoll`) không thể được áp dụng cho message queue. Hơn nữa, việc viết các chương trình đồng thời xử lý cả đầu vào từ message queue và cơ chế I/O dựa trên file descriptor đòi hỏi code phức tạp hơn so với code chỉ xử lý file descriptor. (Chúng ta xem xét một cách kết hợp hai mô hình I/O trong Bài tập 63-3.)
- Việc sử dụng key thay vì tên file để xác định message queue dẫn đến sự phức tạp lập trình bổ sung và cũng đòi hỏi sử dụng `ipcs` và `ipcrm` thay vì `ls` và `rm`. Hàm `ftok()` thường tạo ra một key duy nhất, nhưng không được đảm bảo làm vậy. Sử dụng key `IPC_PRIVATE` đảm bảo một queue identifier duy nhất, nhưng để lại cho chúng ta nhiệm vụ làm cho identifier đó hiển thị cho các process khác cần nó.
- Message queue không có kết nối, và kernel không duy trì một đếm số lượng process tham chiếu đến queue như được thực hiện với pipe, FIFO và socket. Do đó, có thể khó trả lời các câu hỏi sau:
  - Khi nào thì an toàn để ứng dụng xóa một message queue? (Xóa queue sớm dẫn đến mất dữ liệu ngay lập tức, bất kể bất kỳ process nào có thể quan tâm đến việc đọc từ queue sau này.)
  - Làm thế nào để ứng dụng đảm bảo rằng một queue không được sử dụng sẽ bị xóa?
- Có các giới hạn về tổng số message queue, kích thước của message và dung lượng của các queue riêng lẻ. Các giới hạn này có thể cấu hình được, nhưng nếu ứng dụng hoạt động ngoài phạm vi các giới hạn mặc định, điều này đòi hỏi thêm công việc khi cài đặt ứng dụng.

Tóm lại, message queue System V thường nên tránh. Trong các tình huống yêu cầu khả năng chọn message theo kiểu, chúng ta nên xem xét các phương án thay thế. POSIX message queue (Chương 52) là một phương án thay thế như vậy. Như một phương án thay thế tiếp theo, các giải pháp liên quan đến nhiều kênh giao tiếp dựa trên file descriptor có thể cung cấp chức năng tương tự như chọn message theo kiểu, đồng thời cho phép sử dụng các mô hình I/O thay thế được mô tả trong Chương 63. Ví dụ, nếu chúng ta cần truyền message "bình thường" và "ưu tiên", chúng ta có thể sử dụng một cặp FIFO hoặc UNIX domain socket cho hai kiểu message, và sau đó sử dụng `select()` hoặc `poll()` để giám sát các file descriptor cho cả hai kênh.

# **46.10 Tóm Tắt**

Message queue System V cho phép các process giao tiếp bằng cách trao đổi các message bao gồm một kiểu số cộng với một phần thân chứa dữ liệu tùy ý. Các tính năng đặc biệt của message queue là ranh giới message được bảo toàn và các process nhận có thể chọn message theo kiểu, thay vì đọc message theo thứ tự vào trước ra trước.

Nhiều yếu tố dẫn chúng ta đến kết luận rằng các cơ chế IPC khác thường tốt hơn so với message queue System V. Một khó khăn lớn là message queue không được tham chiếu bằng file descriptor. Điều này có nghĩa là chúng ta không thể sử dụng các mô hình I/O thay thế khác nhau với message queue; cụ thể, việc đồng thời giám sát cả message queue và file descriptor để xem liệu I/O có khả thi không là phức tạp. Hơn nữa, thực tế là message queue không có kết nối (tức là không tính tham chiếu) làm cho ứng dụng khó biết khi nào một queue có thể được xóa một cách an toàn.

## **46.11 Bài Tập**

- **46-1.** Thử nghiệm với các chương trình trong Listing 46-1 (svmsg\_create.c), Listing 46-2 (svmsg\_send.c) và Listing 46-3 (svmsg\_receive.c) để xác nhận hiểu biết của bạn về các system call `msgget()`, `msgsnd()` và `msgrcv()`.
- **46-2.** Viết lại ứng dụng client-server số thứ tự của Mục 44.8 để sử dụng message queue System V. Sử dụng một message queue duy nhất để truyền message từ cả client đến server và server đến client. Sử dụng các quy ước về kiểu message được mô tả trong Mục 46.8.
- **46-3.** Trong ứng dụng client-server của Mục 46.8, tại sao client truyền identifier của message queue của nó trong phần thân message (trong trường `clientId`), thay vì trong kiểu message (`mtype`)?
- **46-4.** Thực hiện các thay đổi sau cho ứng dụng client-server của Mục 46.8:
  - a) Thay thế việc sử dụng một message queue key được mã hóa cứng bằng code trong server sử dụng `IPC_PRIVATE` để tạo một identifier duy nhất, và sau đó ghi identifier này vào một file đã biết. Client phải đọc identifier từ file này. Server nên xóa file này nếu nó kết thúc.
  - b) Trong hàm `serveRequest()` của chương trình server, các lỗi system call không được chẩn đoán. Thêm code ghi nhật ký lỗi bằng `syslog()` (Mục 37.5).
  - c) Thêm code vào server để nó trở thành một daemon khi khởi động (Mục 37.2).
  - d) Trong server, thêm một handler cho `SIGTERM` và `SIGINT` thực hiện thoát gọn gàng. Handler nên xóa message queue và (nếu phần trước của bài tập này đã được thực hiện) file được tạo để chứa identifier message queue của server. Bao gồm code trong handler để kết thúc server bằng cách hủy thiết lập handler, và sau đó một lần nữa raise signal tương tự đã gọi handler (xem Mục 26.1.4 để biết lý do và các bước cần thiết cho nhiệm vụ này).
  - e) Server child không xử lý khả năng client có thể kết thúc sớm, trong trường hợp đó server child sẽ lấp đầy message queue của client, và sau đó block vô hạn. Sửa đổi server để xử lý khả năng này bằng cách thiết lập một timeout khi gọi `msgsnd()`, như mô tả trong Mục 23.3. Nếu server child nhận thấy rằng client đã biến mất, nó nên cố gắng xóa message queue của client, và sau đó thoát (có thể sau khi ghi nhật ký một message qua `syslog()`).

- **46-5.** Client được hiển thị trong Listing 46-9 (svmsg\_file\_client.c) không xử lý các khả năng khác nhau cho lỗi trong server. Cụ thể, nếu message queue của server đầy (có thể vì server đã kết thúc và queue đã được lấp đầy bởi các client khác), thì lời gọi `msgsnd()` sẽ bị block vô hạn. Tương tự, nếu server không gửi phản hồi cho client, thì lời gọi `msgrcv()` sẽ bị block vô hạn. Thêm code vào client để đặt timeout (Mục 23.3) trên các lời gọi này. Nếu một trong hai lời gọi timeout, thì chương trình nên báo cáo lỗi cho user và kết thúc.
- **46-6.** Viết một ứng dụng chat đơn giản (tương tự như `talk(1)`, nhưng không có giao diện curses) sử dụng message queue System V. Sử dụng một message queue duy nhất cho mỗi client.
