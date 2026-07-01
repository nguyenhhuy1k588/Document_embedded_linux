## Chương 44
# **PIPES VÀ FIFOS**

Chương này mô tả về pipe và FIFO. Pipe là phương thức IPC lâu đời nhất trên hệ thống UNIX, xuất hiện lần đầu trong Third Edition UNIX vào đầu những năm 1970. Pipe cung cấp một giải pháp thanh lịch cho một yêu cầu thường gặp: sau khi tạo ra hai process để chạy các chương trình (lệnh) khác nhau, làm thế nào để shell cho phép output của một process được dùng làm input cho process kia? Pipe có thể được dùng để truyền dữ liệu giữa các process có quan hệ với nhau (ý nghĩa của "có quan hệ" sẽ được làm rõ sau). FIFO là một biến thể của khái niệm pipe. Điểm khác biệt quan trọng là FIFO có thể được dùng để giao tiếp giữa bất kỳ process nào.

# **44.1 Tổng quan**

Mọi người dùng shell đều quen thuộc với việc sử dụng pipe trong các lệnh như sau, để đếm số lượng file trong một thư mục:

```
$ ls | wc -l
```

Để thực thi lệnh trên, shell tạo ra hai process, lần lượt thực thi `ls` và `wc`. (Việc này được thực hiện bằng `fork()` và `exec()`, được mô tả trong Chương 24 và 27.) Hình 44-1 cho thấy hai process sử dụng pipe như thế nào.

Trong số các nội dung khác, Hình 44-1 nhằm minh họa cách pipe có tên gọi đó. Chúng ta có thể nghĩ về pipe như một đoạn ống dẫn cho phép dữ liệu chảy từ process này sang process khác.

![](_page_13_Figure_0.jpeg)

<span id="page-13-0"></span>Hình 44-1: Sử dụng pipe để kết nối hai process

Một điểm cần lưu ý trong Hình 44-1 là hai process được kết nối với pipe sao cho process ghi (`ls`) có standard output (file descriptor 1) nối với đầu ghi của pipe, trong khi process đọc (`wc`) có standard input (file descriptor 0) nối với đầu đọc của pipe. Về thực chất, hai process này không biết về sự tồn tại của pipe; chúng chỉ đơn giản đọc từ và ghi vào các file descriptor tiêu chuẩn. Shell phải thực hiện một số công việc để thiết lập điều này, và chúng ta sẽ xem cách thực hiện trong Mục 44.4.

Trong các đoạn tiếp theo, chúng ta sẽ đề cập đến một số đặc điểm quan trọng của pipe.

#### Pipe là một byte stream

Khi nói rằng pipe là một byte stream, chúng ta có nghĩa là không có khái niệm về message hoặc ranh giới message khi sử dụng pipe. Process đọc từ pipe có thể đọc các khối dữ liệu có kích thước bất kỳ, bất kể kích thước các khối được ghi bởi process ghi. Hơn nữa, dữ liệu đi qua pipe theo thứ tự tuần tự — các byte được đọc từ pipe theo đúng thứ tự chúng được ghi. Không thể truy cập ngẫu nhiên vào dữ liệu trong pipe bằng `lseek()`.

Nếu chúng ta muốn thực hiện khái niệm các message rời rạc trong pipe, chúng ta phải tự làm điều đó trong ứng dụng. Mặc dù điều này là khả thi (tham khảo Mục 44.8), nhưng có thể tốt hơn khi sử dụng các cơ chế IPC thay thế, như message queue và datagram socket, mà chúng ta sẽ thảo luận trong các chương sau.

## Đọc từ pipe

Các lần thử đọc từ pipe đang rỗng sẽ block cho đến khi có ít nhất một byte được ghi vào pipe. Nếu đầu ghi của pipe bị đóng, thì process đọc từ pipe sẽ thấy end-of-file (tức là `read()` trả về 0) sau khi đã đọc hết toàn bộ dữ liệu còn lại trong pipe.

## Pipe là unidirectional (một chiều)

Dữ liệu chỉ có thể di chuyển theo một chiều qua pipe. Một đầu của pipe được dùng để ghi, đầu kia dùng để đọc.

Trên một số UNIX implementation khác — đáng chú ý là các hệ thống dẫn xuất từ System V Release 4 — pipe là bidirectional (hai chiều), được gọi là *stream pipe*. Pipe hai chiều không được chỉ định bởi bất kỳ chuẩn UNIX nào, vì vậy, ngay cả trên các implementation cung cấp chúng, tốt nhất là nên tránh phụ thuộc vào ngữ nghĩa của chúng. Thay thế, chúng ta có thể sử dụng các cặp UNIX domain stream socket (được tạo bằng system call `socketpair()` mô tả trong Mục 57.5), cung cấp cơ chế giao tiếp hai chiều được chuẩn hóa tương đương về mặt ngữ nghĩa với stream pipe.

## **Các lần ghi tối đa `PIPE_BUF` byte được đảm bảo là atomic**

Nếu nhiều process ghi vào một pipe duy nhất, thì dữ liệu của chúng sẽ không bị xen kẽ lẫn nhau nếu mỗi lần chúng ghi không quá `PIPE_BUF` byte.

SUSv3 yêu cầu `PIPE_BUF` phải có ít nhất `_POSIX_PIPE_BUF` (512). Một implementation nên định nghĩa `PIPE_BUF` (trong `<limits.h>`) và/hoặc cho phép lời gọi `fpathconf(fd, _PC_PIPE_BUF)` trả về giới hạn trên thực tế cho các lần ghi atomic. `PIPE_BUF` thay đổi tùy theo UNIX implementation; ví dụ, nó là 512 byte trên FreeBSD 6.0, 4096 byte trên Tru64 5.1, và 5120 byte trên Solaris 8. Trên Linux, `PIPE_BUF` có giá trị 4096.

Khi ghi các khối dữ liệu lớn hơn `PIPE_BUF` byte vào pipe, kernel có thể truyền dữ liệu theo nhiều phần nhỏ hơn, và nối thêm dữ liệu khi reader lấy các byte ra khỏi pipe. (Lời gọi `write()` sẽ block cho đến khi toàn bộ dữ liệu đã được ghi vào pipe.) Khi chỉ có một process ghi vào pipe (trường hợp thông thường), điều này không quan trọng. Tuy nhiên, nếu có nhiều process ghi, thì các lần ghi khối lớn có thể bị chia thành các đoạn có kích thước tùy ý (có thể nhỏ hơn `PIPE_BUF` byte) và xen kẽ với các lần ghi của process khác.

Giới hạn `PIPE_BUF` ảnh hưởng đến chính xác thời điểm dữ liệu được chuyển vào pipe. Khi ghi tối đa `PIPE_BUF` byte, `write()` sẽ block nếu cần thiết cho đến khi có đủ space trong pipe để có thể hoàn thành thao tác một cách atomic. Khi ghi nhiều hơn `PIPE_BUF` byte, `write()` chuyển dữ liệu càng nhiều càng tốt để lấp đầy pipe, rồi block cho đến khi dữ liệu được lấy ra bởi một process đọc. Nếu một `write()` đang block bị ngắt bởi một signal handler, thì lời gọi sẽ unblock và trả về số lượng byte đã truyền thành công, sẽ ít hơn số lượng được yêu cầu (gọi là *partial write*).

> Trên Linux 2.2, các lần ghi pipe với kích thước bất kỳ là atomic, trừ khi bị ngắt bởi signal handler. Trên Linux 2.4 và các phiên bản sau, bất kỳ lần ghi nào lớn hơn `PIPE_BUF` byte đều có thể bị xen kẽ với các lần ghi của process khác. (Code kernel thực thi pipe đã trải qua những thay đổi đáng kể giữa các phiên bản kernel 2.2 và 2.4.)

## **Pipe có capacity giới hạn**

Pipe đơn giản là một buffer được duy trì trong kernel memory. Buffer này có capacity tối đa. Khi pipe đầy, các lần ghi tiếp theo vào pipe sẽ block cho đến khi reader lấy bớt dữ liệu ra khỏi pipe.

SUSv3 không đặt yêu cầu về capacity của pipe. Trong các kernel Linux trước 2.6.11, capacity của pipe bằng với kích thước page của hệ thống (ví dụ: 4096 byte trên x86-32); kể từ Linux 2.6.11, capacity của pipe là 65,536 byte. Các UNIX implementation khác có capacity pipe khác nhau.

Nói chung, ứng dụng không bao giờ cần biết chính xác capacity của pipe. Nếu chúng ta muốn ngăn các process ghi bị block, các process đọc từ pipe nên được thiết kế để đọc dữ liệu ngay khi có.

> Về lý thuyết, không có lý do gì pipe không thể hoạt động với capacity nhỏ hơn, thậm chí với buffer một byte. Lý do sử dụng kích thước buffer lớn là hiệu quả: mỗi lần writer lấp đầy pipe, kernel phải thực hiện context switch để cho phép reader được lập lịch để có thể lấy bớt dữ liệu ra khỏi pipe. Sử dụng kích thước buffer lớn hơn có nghĩa là cần ít context switch hơn.

> Bắt đầu từ Linux 2.6.35, capacity của pipe có thể được thay đổi. Lời gọi đặc thù Linux `fcntl(fd, F_SETPIPE_SZ, size)` thay đổi capacity của pipe được tham chiếu bởi `fd` để ít nhất là `size` byte. Một process không có đặc quyền có thể thay đổi capacity của pipe sang bất kỳ giá trị nào trong phạm vi từ kích thước page hệ thống đến giá trị trong `/proc/sys/fs/pipe-max-size`. Giá trị mặc định cho `pipe-max-size` là 1,048,576 byte. Một process có đặc quyền (`CAP_SYS_RESOURCE`) có thể vượt qua giới hạn này. Khi cấp phát space cho pipe, kernel có thể làm tròn `size` lên một giá trị thuận tiện cho implementation. Lời gọi `fcntl(fd, F_GETPIPE_SZ)` trả về kích thước thực tế được cấp phát cho pipe.

# **44.2 Tạo và Sử dụng Pipe**

System call `pipe()` tạo một pipe mới.

```
#include <unistd.h>
int pipe(int filedes[2]);
                                              Returns 0 on success, or –1 on error
```

Một lời gọi `pipe()` thành công trả về hai file descriptor mở trong mảng `filedes`: một cho đầu đọc của pipe (`filedes[0]`) và một cho đầu ghi (`filedes[1]`).

Như với bất kỳ file descriptor nào, chúng ta có thể sử dụng các system call `read()` và `write()` để thực hiện I/O trên pipe. Sau khi được ghi vào đầu ghi của pipe, dữ liệu ngay lập tức có thể được đọc từ đầu đọc. Một `read()` từ pipe lấy giá trị nhỏ hơn trong số số byte được yêu cầu và số byte hiện đang có trong pipe (nhưng sẽ block nếu pipe rỗng).

Chúng ta cũng có thể sử dụng các hàm stdio (`printf()`, `scanf()`, v.v.) với pipe bằng cách trước tiên dùng `fdopen()` để lấy một file stream tương ứng với một trong các descriptor trong `filedes` (Mục 13.7). Tuy nhiên, khi làm điều này, chúng ta phải lưu ý đến các vấn đề về stdio buffering được mô tả trong Mục 44.6.

> Lời gọi `ioctl(fd, FIONREAD, &cnt)` trả về số byte chưa được đọc trong pipe hoặc FIFO được tham chiếu bởi file descriptor `fd`. Tính năng này cũng có sẵn trên một số implementation khác, nhưng không được chỉ định trong SUSv3.

Hình 44-2 cho thấy tình huống sau khi một pipe đã được tạo bởi `pipe()`, với process gọi có các file descriptor tham chiếu đến mỗi đầu.

![](_page_15_Figure_9.jpeg)

<span id="page-15-0"></span>**Hình 44-2:** Các file descriptor của process sau khi tạo pipe

Pipe ít có dụng nếu chỉ trong một process (chúng ta xem xét một trường hợp trong Mục 63.5.2). Thông thường, chúng ta sử dụng pipe để cho phép giao tiếp giữa hai process. Để kết nối hai process bằng pipe, chúng ta thực hiện lời gọi `pipe()` sau đó là lời gọi `fork()`. Trong một `fork()`, child process kế thừa các bản sao của file descriptor của process cha (Mục 24.2.1), tạo ra tình huống được hiển thị ở phía bên trái của Hình 44-3.

![](_page_16_Figure_0.jpeg)

<span id="page-16-0"></span>**Hình 44-3:** Thiết lập pipe để truyền dữ liệu từ parent đến child

Mặc dù cả parent và child đều có thể đọc từ và ghi vào pipe, nhưng điều này không phổ biến. Do đó, ngay sau `fork()`, một process đóng descriptor cho đầu ghi của pipe, và process kia đóng descriptor cho đầu đọc. Ví dụ: nếu parent gửi dữ liệu cho child, thì parent sẽ đóng read descriptor của pipe, `filedes[0]`, trong khi child đóng write descriptor của pipe, `filedes[1]`, tạo ra tình huống được hiển thị ở phía bên phải của Hình 44-3. Code để tạo thiết lập này được hiển thị trong Listing 44-1.

**Listing 44-1:** Các bước tạo pipe để truyền dữ liệu từ parent đến child

```
 int filedes[2];
 if (pipe(filedes) == -1) /* Create the pipe */
 errExit("pipe");
 switch (fork()) { /* Create a child process */
 case -1:
 errExit("fork");
 case 0: /* Child */
 if (close(filedes[1]) == -1) /* Close unused write end */
 errExit("close");
 /* Child now reads from pipe */
 break;
 default: /* Parent */
 if (close(filedes[0]) == -1) /* Close unused read end */
 errExit("close");
 /* Parent now writes to pipe */
 break;
 }
```

Một lý do khiến cả parent và child không thường xuyên đọc từ một pipe duy nhất là nếu hai process cùng cố đọc từ pipe đồng thời, chúng ta không thể chắc chắn process nào sẽ thành công trước — hai process tranh nhau dữ liệu. Việc ngăn chặn các cuộc đua như vậy sẽ cần sử dụng một cơ chế đồng bộ hóa nào đó. Tuy nhiên, nếu chúng ta cần giao tiếp hai chiều, có một cách đơn giản hơn: chỉ cần tạo hai pipe, mỗi pipe để gửi dữ liệu theo một chiều giữa hai process. (Nếu sử dụng kỹ thuật này, chúng ta cần cẩn thận với các deadlock có thể xảy ra nếu cả hai process block trong khi cố đọc từ pipe rỗng hoặc khi cố ghi vào pipe đã đầy.)

Mặc dù có thể có nhiều process ghi vào một pipe, nhưng thông thường chỉ có một writer. (Chúng ta sẽ thấy một ví dụ về việc có nhiều writer trên một pipe trong Mục 44.3.) Ngược lại, có những tình huống có thể hữu ích khi có nhiều writer trên một FIFO, và chúng ta thấy ví dụ về điều này trong Mục 44.8.

> Bắt đầu từ kernel 2.6.27, Linux hỗ trợ một system call mới, không chuẩn, là `pipe2()`. System call này thực hiện cùng nhiệm vụ như `pipe()`, nhưng hỗ trợ một đối số bổ sung, `flags`, có thể được dùng để thay đổi hành vi của system call. Hai flag được hỗ trợ. Flag `O_CLOEXEC` khiến kernel bật close-on-exec flag (`FD_CLOEXEC`) cho hai file descriptor mới. Flag này hữu ích vì những lý do tương tự như flag `O_CLOEXEC` của `open()` được mô tả trong Mục 4.3.1. Flag `O_NONBLOCK` khiến kernel đánh dấu cả hai open file description bên dưới là nonblocking, để các thao tác I/O sau này sẽ là nonblocking. Điều này tiết kiệm các lời gọi `fcntl()` bổ sung để đạt được kết quả tương tự.

### **Pipe cho phép giao tiếp giữa các process có quan hệ**

Trong thảo luận cho đến nay, chúng ta đã nói về việc sử dụng pipe để giao tiếp giữa parent và child process. Tuy nhiên, pipe có thể được dùng để giao tiếp giữa bất kỳ hai (hoặc nhiều) process có quan hệ, miễn là pipe được tạo bởi một tổ tiên chung trước chuỗi các lời gọi `fork()` dẫn đến sự tồn tại của các process đó. (Đây là ý nghĩa khi chúng ta đề cập đến các process "có quan hệ" ở đầu chương này.) Ví dụ: một pipe có thể được dùng để giao tiếp giữa một process và cháu nội của nó. Process đầu tiên tạo pipe, sau đó fork một child process lại tiếp tục fork để tạo ra cháu nội. Một kịch bản phổ biến là pipe được dùng để giao tiếp giữa hai sibling — parent tạo pipe, sau đó tạo ra hai child. Đây là những gì shell thực hiện khi xây dựng một pipeline.

> Có một ngoại lệ đối với phát biểu rằng pipe chỉ có thể được dùng để giao tiếp giữa các process có quan hệ. Việc truyền file descriptor qua UNIX domain socket (một kỹ thuật mà chúng ta mô tả ngắn gọn trong Mục 61.13.3) cho phép truyền file descriptor cho một pipe đến một process không có quan hệ.

## **Đóng các file descriptor pipe không dùng đến**

Đóng các file descriptor pipe không dùng không chỉ là vấn đề đảm bảo rằng process không làm cạn kiệt bộ file descriptor hạn chế của nó — mà còn thiết yếu cho việc sử dụng pipe đúng cách. Chúng ta sẽ xem xét tại sao các file descriptor không dùng cho cả đầu đọc và đầu ghi của pipe phải được đóng.

Process đọc từ pipe đóng write descriptor của mình cho pipe, để khi process kia hoàn thành output và đóng write descriptor, reader thấy end-of-file (sau khi đã đọc hết dữ liệu còn lại trong pipe).

Nếu process đọc không đóng đầu ghi của pipe, thì sau khi process kia đóng write descriptor, reader sẽ không thấy end-of-file, ngay cả sau khi đã đọc hết tất cả dữ liệu từ pipe. Thay vào đó, `read()` sẽ block chờ dữ liệu, vì kernel biết rằng vẫn còn ít nhất một write descriptor mở cho pipe. Thực tế là descriptor này được giữ mở bởi chính process đọc là không liên quan; về lý thuyết, process đó vẫn có thể ghi vào pipe, ngay cả khi nó đang block để thử đọc. Ví dụ: `read()` có thể bị ngắt bởi signal handler ghi dữ liệu vào pipe. (Đây là một kịch bản thực tế, như chúng ta sẽ thấy trong Mục 63.5.2.)

Process ghi đóng read descriptor của mình cho pipe vì một lý do khác. Khi process cố ghi vào pipe mà không có process nào có read descriptor mở, kernel gửi signal `SIGPIPE` cho process ghi. Theo mặc định, signal này sẽ giết process đó. Process có thể sắp xếp để bắt hoặc ignore signal này, trong trường hợp đó `write()` trên pipe thất bại với lỗi `EPIPE` (broken pipe). Nhận signal `SIGPIPE` hoặc nhận lỗi `EPIPE` là một dấu hiệu hữu ích về trạng thái của pipe, và đó là lý do tại sao các read descriptor không dùng cho pipe phải được đóng.

> Lưu ý rằng việc xử lý một `write()` bị ngắt bởi SIGPIPE handler có điểm đặc biệt. Thông thường, khi `write()` (hoặc system call "chậm" khác) bị ngắt bởi signal handler, lời gọi có thể tự động khởi động lại hoặc thất bại với lỗi `EINTR`, tùy thuộc vào việc handler được cài đặt với flag `SA_RESTART` của `sigaction()` (Mục 21.5). Hành vi trong trường hợp `SIGPIPE` khác biệt vì không có ý nghĩa gì khi tự động khởi động lại `write()` hay chỉ đơn giản cho biết rằng `write()` bị ngắt bởi handler (như vậy ngụ ý rằng `write()` có thể được thử lại thủ công một cách hữu ích). Trong cả hai trường hợp, một lần thử `write()` tiếp theo đều không thể thành công, vì pipe vẫn sẽ bị vỡ.

Nếu process ghi không đóng đầu đọc của pipe, thì ngay cả sau khi process kia đóng đầu đọc của pipe, process ghi vẫn có thể ghi vào pipe. Cuối cùng, process ghi sẽ lấp đầy pipe, và một lần thử ghi tiếp theo sẽ block vô thời hạn.

Một lý do cuối cùng để đóng các file descriptor không dùng là chỉ sau khi tất cả file descriptor trong tất cả các process tham chiếu đến một pipe đều được đóng thì pipe mới bị hủy và tài nguyên của nó được giải phóng để các process khác tái sử dụng. Tại điểm này, mọi dữ liệu chưa đọc trong pipe đều bị mất.

## **Chương trình ví dụ**

Chương trình trong Listing 44-2 minh họa việc sử dụng pipe để giao tiếp giữa các process parent và child. Ví dụ này minh họa tính chất byte stream của pipe được đề cập trước đó — parent ghi dữ liệu của mình trong một thao tác duy nhất, trong khi child đọc dữ liệu từ pipe theo từng khối nhỏ.

Chương trình chính gọi `pipe()` để tạo pipe, sau đó gọi `fork()` để tạo một child. Sau `fork()`, process parent đóng file descriptor đầu đọc của pipe, và ghi chuỗi được cung cấp làm đối số dòng lệnh vào đầu ghi của pipe. Parent sau đó đóng đầu ghi của pipe, và gọi `wait()` để chờ child kết thúc. Sau khi đóng file descriptor đầu ghi của pipe, child process vào một vòng lặp đọc các khối dữ liệu (tối đa `BUF_SIZE` byte) từ pipe và ghi chúng ra standard output. Khi child gặp end-of-file trên pipe, nó thoát vòng lặp, ghi ký tự newline kết thúc, đóng descriptor đầu đọc của pipe, và kết thúc.

Đây là ví dụ về những gì chúng ta có thể thấy khi chạy chương trình trong Listing 44-2:

```
$ ./simple_pipe 'It was a bright cold day in April, '\
'and the clocks were striking thirteen.'
It was a bright cold day in April, and the clocks were striking thirteen.
```

<span id="page-19-0"></span>**Listing 44-2:** Sử dụng pipe để giao tiếp giữa parent và child process

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/simple_pipe.c
  #include <sys/wait.h>
  #include "tlpi_hdr.h"
  #define BUF_SIZE 10
  int
  main(int argc, char *argv[])
  {
   int pfd[2]; /* Pipe file descriptors */
   char buf[BUF_SIZE];
   ssize_t numRead;
   if (argc != 2 || strcmp(argv[1], "--help") == 0)
   usageErr("%s string\n", argv[0]);
q if (pipe(pfd) == -1) /* Create the pipe */
   errExit("pipe");
w switch (fork()) {
   case -1:
   errExit("fork");
   case 0: /* Child - reads from pipe */
e if (close(pfd[1]) == -1) /* Write end is unused */
   errExit("close - child");
   for (;;) { /* Read data from pipe, echo on stdout */
r numRead = read(pfd[0], buf, BUF_SIZE);
   if (numRead == -1)
   errExit("read");
t if (numRead == 0)
   break; /* End-of-file */
y if (write(STDOUT_FILENO, buf, numRead) != numRead)
   fatal("child - partial/failed write");
   }
u write(STDOUT_FILENO, "\n", 1);
   if (close(pfd[0]) == -1)
   errExit("close");
   _exit(EXIT_SUCCESS);
   default: /* Parent - writes to pipe */
i if (close(pfd[0]) == -1) /* Read end is unused */
   errExit("close - parent");
o if (write(pfd[1], argv[1], strlen(argv[1])) != strlen(argv[1]))
   fatal("parent - partial/failed write");
a if (close(pfd[1]) == -1) /* Child will see EOF */
   errExit("close");
s wait(NULL); /* Wait for child to finish */
   exit(EXIT_SUCCESS);
   }
  }
  –––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/simple_pipe.c
```

# **44.3 Pipe như Phương thức Đồng bộ hóa Process**

Trong Mục 24.5, chúng ta đã xem xét cách signal có thể được dùng để đồng bộ hóa các hành động của process parent và child nhằm tránh race condition. Pipe có thể được dùng để đạt được kết quả tương tự, như được hiển thị bởi chương trình khung trong Listing 44-3. Chương trình này tạo nhiều child process (một cho mỗi đối số dòng lệnh), mỗi cái có nhiệm vụ thực hiện một hành động nào đó, được mô phỏng trong chương trình ví dụ bằng cách sleep trong một khoảng thời gian. Parent chờ cho đến khi tất cả các child hoàn thành hành động của chúng.

Để thực hiện đồng bộ hóa, parent xây dựng một pipe trước khi tạo các child process. Mỗi child kế thừa một file descriptor cho đầu ghi của pipe và đóng descriptor này sau khi hoàn thành hành động của nó. Sau khi tất cả các child đã đóng file descriptor của chúng cho đầu ghi của pipe, `read()` của parent từ pipe sẽ hoàn thành, trả về end-of-file (0). Tại thời điểm này, parent tự do để tiếp tục làm việc khác. (Lưu ý rằng việc đóng đầu ghi không dùng của pipe trong parent là thiết yếu cho hoạt động đúng đắn của kỹ thuật này; nếu không, parent sẽ block mãi khi cố đọc từ pipe.)

Sau đây là ví dụ về những gì chúng ta thấy khi dùng chương trình trong Listing 44-3 để tạo ba child sleep 4, 2, và 6 giây:

```
$ ./pipe_sync 4 2 6
08:22:16 Parent started
08:22:18 Child 2 (PID=2445) closing pipe
08:22:20 Child 1 (PID=2444) closing pipe
08:22:22 Child 3 (PID=2446) closing pipe
08:22:22 Parent ready to go
```

<span id="page-20-2"></span>**Listing 44-3:** Sử dụng pipe để đồng bộ hóa nhiều process

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/pipe_sync.c
#include "curr_time.h" /* Declaration of currTime() */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int pfd[2]; /* Process synchronization pipe */
 int j, dummy;
 if (argc < 2 || strcmp(argv[1], "--help") == 0)
   usageErr("%s sleep-time...\n", argv[0]);
   setbuf(stdout, NULL); /* Make stdout unbuffered, since we
   terminate child with _exit() */
   printf("%s Parent started\n", currTime("%T"));
q if (pipe(pfd) == -1)
   errExit("pipe");
   for (j = 1; j < argc; j++) {
w switch (fork()) {
   case -1:
   errExit("fork %d", j);
   case 0: /* Child */
   if (close(pfd[0]) == -1) /* Read end is unused */
   errExit("close");
   /* Child does some work, and lets parent know it's done */
   sleep(getInt(argv[j], GN_NONNEG, "sleep-time"));
   /* Simulate processing */
   printf("%s Child %d (PID=%ld) closing pipe\n",
   currTime("%T"), j, (long) getpid());
e if (close(pfd[1]) == -1)
   errExit("close");
   /* Child now carries on to do other things... */
   _exit(EXIT_SUCCESS);
   default: /* Parent loops to create next child */
   break;
   }
   }
   /* Parent comes here; close write end of pipe so we can see EOF */
r if (close(pfd[1]) == -1) /* Write end is unused */
   errExit("close");
   /* Parent may do other work, then synchronizes with children */
t if (read(pfd[0], &dummy, 1) != 0)
   fatal("parent didn't get EOF");
   printf("%s Parent ready to go\n", currTime("%T"));
   /* Parent can now carry on to do other things... */
   exit(EXIT_SUCCESS);
  }
  –––––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/pipe_sync.c
```

Đồng bộ hóa bằng pipe có ưu điểm so với ví dụ trước về đồng bộ hóa bằng signal: nó có thể được dùng để phối hợp hành động của một process với nhiều process (có quan hệ) khác. Thực tế là nhiều signal (tiêu chuẩn) không thể được xếp hàng làm cho signal không phù hợp trong trường hợp này. (Ngược lại, signal có ưu điểm là chúng có thể được broadcast bởi một process đến tất cả các thành viên của một process group.)

Các cấu trúc liên kết đồng bộ hóa khác cũng có thể (ví dụ: sử dụng nhiều pipe). Hơn nữa, kỹ thuật này có thể được mở rộng để thay vì đóng pipe, mỗi child ghi một message vào pipe chứa process ID và một số thông tin trạng thái của nó. Ngoài ra, mỗi child có thể ghi một byte duy nhất vào pipe. Process parent sau đó có thể đếm và phân tích các message này. Cách tiếp cận này bảo vệ trước khả năng child vô tình kết thúc thay vì đóng pipe một cách rõ ràng.

# **44.4 Sử dụng Pipe để Kết nối các Filter**

Khi một pipe được tạo, các file descriptor được dùng cho hai đầu của pipe là các descriptor có số thứ tự thấp nhất hiện có. Vì trong hoàn cảnh bình thường, descriptor 0, 1, và 2 đã được sử dụng cho một process, một số descriptor có số thứ tự cao hơn sẽ được cấp phát cho pipe. Vậy làm thế nào để tạo ra tình huống được hiển thị trong Hình 44-1, nơi hai filter (tức là các chương trình đọc từ stdin và ghi vào stdout) được kết nối bằng pipe, sao cho standard output của một chương trình được chuyển hướng vào pipe và standard input của chương trình kia được lấy từ pipe? Và đặc biệt, làm thế nào chúng ta có thể làm điều này mà không sửa đổi code của chính các filter?

Câu trả lời là sử dụng các kỹ thuật được mô tả trong Mục 5.5 để nhân đôi file descriptor. Theo truyền thống, chuỗi lời gọi sau được dùng để đạt được kết quả mong muốn:

```
int pfd[2];
pipe(pfd); /* Allocates (say) file descriptors 3 and 4 for pipe */
/* Other steps here, e.g., fork() */
close(STDOUT_FILENO); /* Free file descriptor 1 */
dup(pfd[1]); /* Duplication uses lowest free file
 descriptor, i.e., fd 1 */
```

Kết quả cuối cùng của các bước trên là standard output của process được gắn với đầu ghi của pipe. Một tập hợp lời gọi tương ứng có thể được dùng để gắn standard input của process với đầu đọc của pipe.

Lưu ý rằng các bước này phụ thuộc vào giả định rằng file descriptor 0, 1, và 2 cho một process đã được mở. (Shell thường đảm bảo điều này cho mỗi chương trình nó thực thi.) Nếu file descriptor 0 đã bị đóng trước các bước trên, thì chúng ta sẽ vô tình gắn standard input của process với đầu ghi của pipe. Để tránh khả năng này, chúng ta có thể thay thế các lời gọi `close()` và `dup()` bằng lời gọi `dup2()` sau, cho phép chúng ta chỉ định rõ ràng descriptor được gắn với đầu pipe:

```
dup2(pfd[1], STDOUT_FILENO); /* Close descriptor 1, and reopen bound
 to write end of pipe */
```

Sau khi nhân đôi `pfd[1]`, chúng ta có hai file descriptor tham chiếu đến đầu ghi của pipe: descriptor 1 và `pfd[1]`. Vì các file descriptor pipe không dùng phải được đóng, sau lời gọi `dup2()`, chúng ta đóng descriptor thừa:

```
close(pfd[1]);
```

Code chúng ta đã trình bày đến nay phụ thuộc vào việc standard output đã được mở trước đó. Giả sử rằng, trước lời gọi `pipe()`, cả standard input lẫn standard output đều bị đóng. Trong trường hợp này, `pipe()` sẽ cấp phát hai descriptor này cho pipe, có thể với `pfd[0]` có giá trị 0 và `pfd[1]` có giá trị 1. Kết quả là các lời gọi `dup2()` và `close()` trước đó sẽ tương đương với:

```
dup2(1, 1); /* Does nothing */
close(1); /* Closes sole descriptor for write end of pipe */
```

Do đó, là một thực hành lập trình phòng thủ tốt khi bao bọc các lời gọi này bằng câu lệnh `if` có dạng sau:

```
if (pfd[1] != STDOUT_FILENO) {
 dup2(pfd[1], STDOUT_FILENO);
 close(pfd[1]);
}
```

### **Chương trình ví dụ**

Chương trình trong Listing 44-4 sử dụng các kỹ thuật được mô tả trong mục này để tạo ra thiết lập được hiển thị trong Hình 44-1. Sau khi xây dựng pipe, chương trình này tạo hai child process. Child đầu tiên gắn standard output của nó với đầu ghi của pipe rồi exec `ls`. Child thứ hai gắn standard input của nó với đầu đọc của pipe rồi exec `wc`.

<span id="page-23-0"></span>**Listing 44-4:** Sử dụng pipe để kết nối `ls` và `wc`

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/pipe_ls_wc.c
#include <sys/wait.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int pfd[2]; /* Pipe file descriptors */
 if (pipe(pfd) == -1) /* Create pipe */
 errExit("pipe");
 switch (fork()) {
 case -1:
 errExit("fork");
 case 0: /* First child: exec 'ls' to write to pipe */
 if (close(pfd[0]) == -1) /* Read end is unused */
 errExit("close 1");
 /* Duplicate stdout on write end of pipe; close duplicated descriptor */
 if (pfd[1] != STDOUT_FILENO) { /* Defensive check */
 if (dup2(pfd[1], STDOUT_FILENO) == -1)
 errExit("dup2 1");
 if (close(pfd[1]) == -1)
 errExit("close 2");
 }
 execlp("ls", "ls", (char *) NULL); /* Writes to pipe */
 errExit("execlp ls");
 default: /* Parent falls through to create next child */
 break;
 }
 switch (fork()) {
 case -1:
 errExit("fork");
 case 0: /* Second child: exec 'wc' to read from pipe */
 if (close(pfd[1]) == -1) /* Write end is unused */
 errExit("close 3");
 /* Duplicate stdin on read end of pipe; close duplicated descriptor */
 if (pfd[0] != STDIN_FILENO) { /* Defensive check */
 if (dup2(pfd[0], STDIN_FILENO) == -1)
 errExit("dup2 2");
 if (close(pfd[0]) == -1)
 errExit("close 4");
 }
 execlp("wc", "wc", "-l", (char *) NULL); /* Reads from pipe */
 errExit("execlp wc");
 default: /* Parent falls through */
 break;
 }
 /* Parent closes unused file descriptors for pipe, and waits for children */
 if (close(pfd[0]) == -1)
 errExit("close 5");
 if (close(pfd[1]) == -1)
 errExit("close 6");
 if (wait(NULL) == -1)
 errExit("wait 1");
 if (wait(NULL) == -1)
 errExit("wait 2");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/pipe_ls_wc.c
```

# 44.5 Giao tiếp với Lệnh Shell qua Pipe: `popen()`

Một ứng dụng phổ biến của pipe là thực thi một lệnh shell và đọc output của nó hoặc gửi input vào nó. Các hàm `popen()` và `pclose()` được cung cấp để đơn giản hóa nhiệm vụ này.

```
#include <stdio.h>
FILE *popen(const char *command, const char *mode);
```

Hàm `popen()` tạo một pipe, sau đó fork một child process thực thi shell, shell lại tạo thêm một child process để thực thi chuỗi được cung cấp trong `command`. Đối số `mode` là một chuỗi xác định liệu process gọi sẽ đọc từ pipe (`mode` là `r`) hay ghi vào nó (`mode` là `w`). (Vì pipe là unidirectional, giao tiếp hai chiều với `command` được thực thi là không thể.) Giá trị của `mode` xác định liệu standard output của lệnh được thực thi có được kết nối với đầu ghi của pipe hay standard input của nó được kết nối với đầu đọc của pipe, như được hiển thị trong Hình 44-4.

![](_page_25_Figure_6.jpeg)

<span id="page-25-0"></span>**Hình 44-4:** Tổng quan về mối quan hệ process và cách sử dụng pipe cho `popen()`

Khi thành công, `popen()` trả về một con trỏ file stream có thể được sử dụng với các hàm thư viện `stdio`. Nếu xảy ra lỗi (ví dụ: `mode` không phải là `r` hay `w`, tạo pipe thất bại, hoặc `fork()` để tạo child thất bại), thì `popen()` trả về `NULL` và thiết lập `errno` để chỉ ra nguyên nhân lỗi.

Sau lời gọi `popen()`, process gọi sử dụng pipe để đọc output của `command` hoặc gửi input vào nó. Giống như với pipe được tạo bằng `pipe()`, khi đọc từ pipe, process gọi gặp end-of-file khi `command` đóng đầu ghi của pipe; khi ghi vào pipe, nó nhận signal `SIGPIPE` và nhận lỗi `EPIPE`, nếu `command` đã đóng đầu đọc của pipe.

Sau khi I/O hoàn tất, hàm `pclose()` được dùng để đóng pipe và chờ child shell kết thúc. (Hàm `fclose()` không nên được dùng, vì nó không chờ child.) Khi thành công, `pclose()` trả về trạng thái kết thúc (Mục 26.1.3) của child shell (là trạng thái kết thúc của lệnh cuối cùng mà shell thực thi, trừ khi shell bị giết bởi signal). Như với `system()` (Mục 27.6), nếu một shell không thể được exec, thì `pclose()` trả về một giá trị như thể child shell đã kết thúc bằng lời gọi `_exit(127)`. Nếu có lỗi khác, `pclose()` trả về –1. Một lỗi có thể là không thể lấy được trạng thái kết thúc. Chúng ta sẽ giải thích cách điều này có thể xảy ra.

Khi thực hiện wait để lấy trạng thái của child shell, SUSv3 yêu cầu `pclose()`, như `system()`, phải tự động khởi động lại lời gọi nội bộ mà nó thực hiện đến `waitpid()` nếu lời gọi đó bị ngắt bởi signal handler.

Nhìn chung, chúng ta có thể đưa ra những nhận xét tương tự cho `popen()` như đã đưa ra trong Mục 27.6 cho `system()`. Sử dụng `popen()` mang lại sự tiện lợi. Nó xây dựng pipe, thực hiện descriptor duplication, đóng các descriptor không dùng, và xử lý tất cả các chi tiết của `fork()` và `exec()` thay cho chúng ta. Ngoài ra, shell xử lý lệnh được thực hiện. Sự tiện lợi này đi kèm với chi phí về hiệu suất. Ít nhất hai process bổ sung phải được tạo: một cho shell và một hoặc nhiều hơn cho các lệnh được thực thi bởi shell. Như với `system()`, `popen()` không bao giờ nên được sử dụng từ các chương trình có đặc quyền.

Mặc dù có một số điểm tương đồng giữa `system()` và `popen()` cộng với `pclose()`, cũng có những khác biệt đáng kể. Những khác biệt này xuất phát từ thực tế rằng, với `system()`, việc thực thi lệnh shell được đóng gói trong một lời gọi hàm duy nhất, trong khi với `popen()`, process gọi chạy song song với lệnh shell và sau đó gọi `pclose()`. Các khác biệt như sau:

-  Vì process gọi và lệnh được thực thi hoạt động song song, SUSv3 yêu cầu `popen()` không được ignore `SIGINT` và `SIGQUIT`. Nếu được tạo từ bàn phím, các signal này được gửi đến cả process gọi và lệnh được thực thi. Điều này xảy ra vì cả hai process đều nằm trong cùng process group, và các signal được tạo bởi terminal được gửi đến tất cả các thành viên của process group (foreground), như được mô tả trong Mục 34.5.
-  Vì process gọi có thể tạo các child process khác giữa lần thực thi `popen()` và `pclose()`, SUSv3 yêu cầu `popen()` không được block `SIGCHLD`. Điều này có nghĩa là nếu process gọi thực hiện thao tác wait trước lời gọi `pclose()`, nó có thể lấy được trạng thái của child được tạo bởi `popen()`. Trong trường hợp này, khi `pclose()` được gọi sau đó, nó sẽ trả về –1, với `errno` được đặt thành `ECHILD`, cho biết `pclose()` không thể lấy được trạng thái của child.

## **Chương trình ví dụ**

Listing 44-5 minh họa việc sử dụng `popen()` và `pclose()`. Chương trình này liên tục đọc một pattern wildcard tên file, sau đó dùng `popen()` để lấy kết quả từ việc truyền pattern này cho lệnh `ls`. (Các kỹ thuật tương tự như thế này đã được sử dụng trên các UNIX implementation cũ hơn để thực hiện tạo tên file, còn được biết là globbing, trước khi có hàm thư viện `glob()`.)

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/popen_glob.c
  #include <ctype.h>
  #include <limits.h>
  #include "print_wait_status.h" /* For printWaitStatus() */
  #include "tlpi_hdr.h"
q #define POPEN_FMT "/bin/ls -d %s 2> /dev/null"
  #define PAT_SIZE 50
  #define PCMD_BUF_SIZE (sizeof(POPEN_FMT) + PAT_SIZE)
  int
  main(int argc, char *argv[])
  {
   char pat[PAT_SIZE]; /* Pattern for globbing */
   char popenCmd[PCMD_BUF_SIZE];
   FILE *fp; /* File stream returned by popen() */
   Boolean badPattern; /* Invalid characters in 'pat'? */
   int len, status, fileCnt, j;
   char pathname[PATH_MAX];
   for (;;) { /* Read pattern, display results of globbing */
   printf("pattern: ");
   fflush(stdout);
w if (fgets(pat, PAT_SIZE, stdin) == NULL)
   break; /* EOF */
   len = strlen(pat);
   if (len <= 1) /* Empty line */
   continue;
   if (pat[len - 1] == '\n') /* Strip trailing newline */
   pat[len - 1] = '\0';
   /* Ensure that the pattern contains only valid characters,
   i.e., letters, digits, underscore, dot, and the shell
   globbing characters. (Our definition of valid is more
   restrictive than the shell, which permits other characters
   to be included in a filename if they are quoted.) */
e for (j = 0, badPattern = FALSE; j < len && !badPattern; j++)
   if (!isalnum((unsigned char) pat[j]) &&
   strchr("_*?[^-].", pat[j]) == NULL)
   badPattern = TRUE;
   if (badPattern) {
   printf("Bad pattern character: %c\n", pat[j - 1]);
   continue;
   }
   /* Build and execute command to glob 'pat' */
r snprintf(popenCmd, PCMD_BUF_SIZE, POPEN_FMT, pat);
   popenCmd[PCMD_BUF_SIZE - 1] = '\0'; /* Ensure string is
   null-terminated */
t fp = popen(popenCmd, "r");
   if (fp == NULL) {
   printf("popen() failed\n");
   continue;
   }
   /* Read resulting list of pathnames until EOF */
   fileCnt = 0;
   while (fgets(pathname, PATH_MAX, fp) != NULL) {
   printf("%s", pathname);
   fileCnt++;
   }
   /* Close pipe, fetch and display termination status */
   status = pclose(fp);
   printf(" %d matching file%s\n", fileCnt, (fileCnt != 1) ? "s" : "");
   printf(" pclose() status == %#x\n", (unsigned int) status);
   if (status != -1)
   printWaitStatus("\t", status);
   }
   exit(EXIT_SUCCESS);
  }
  ––––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/popen_glob.c
```

Phiên shell sau minh họa việc sử dụng chương trình trong Listing 44-5. Trong ví dụ này, chúng ta trước tiên cung cấp một pattern khớp với hai tên file, sau đó một pattern không khớp với tên file nào:

```
$ ./popen_glob
pattern: popen_glob* Matches two filenames
popen_glob
popen_glob.c
 2 matching files
 pclose() status = 0
 child exited, status=0
pattern: x* Matches no filename
 0 matching files
 pclose() status = 0x100 ls(1) exits with status 1
 child exited, status=1
pattern: ^D$ Type Control-D to terminate
```

Việc xây dựng lệnh để globbing trong Listing 44-5 cần được giải thích. Việc globbing thực tế của pattern được thực hiện bởi shell. Lệnh `ls` chỉ đơn giản được dùng để liệt kê các tên file khớp, mỗi file một dòng. Chúng ta có thể đã thử dùng lệnh `echo` thay thế, nhưng điều này sẽ có kết quả không mong muốn là nếu pattern không khớp với tên file nào, thì shell sẽ để nguyên pattern, và `echo` sẽ đơn giản hiển thị pattern. Ngược lại, nếu `ls` được cung cấp tên của một file không tồn tại, nó in thông báo lỗi ra stderr (chúng ta bỏ đi bằng cách chuyển hướng stderr đến `/dev/null`), không in gì ra stdout, và thoát với trạng thái 1.

Cũng lưu ý việc kiểm tra input được thực hiện trong Listing 44-5. Điều này được thực hiện để ngăn input không hợp lệ khiến `popen()` thực thi một lệnh shell không mong muốn. Giả sử rằng các kiểm tra này bị bỏ qua và người dùng nhập input sau:

```
pattern: ; rm *
```

Chương trình sau đó sẽ truyền lệnh sau cho `popen()`, với kết quả tai hại:

```
/bin/ls -d ; rm * 2> /dev/null
```

Việc kiểm tra input như vậy luôn được yêu cầu trong các chương trình sử dụng `popen()` (hoặc `system()`) để thực thi một lệnh shell được xây dựng từ input của người dùng. (Một giải pháp thay thế là ứng dụng quote bất kỳ ký tự nào khác ngoài các ký tự đang được kiểm tra, để những ký tự đó không được xử lý đặc biệt bởi shell.)

# **44.6 Pipe và stdio Buffering**

Vì con trỏ file stream được trả về bởi lời gọi `popen()` không tham chiếu đến terminal, thư viện stdio áp dụng block buffering cho file stream (Mục 13.2). Điều này có nghĩa là khi chúng ta gọi `popen()` với `mode` là `w`, thì theo mặc định, output được gửi đến child process ở đầu kia của pipe chỉ khi stdio buffer đầy hoặc chúng ta đóng pipe bằng `pclose()`. Trong nhiều trường hợp, điều này không gây ra vấn đề gì. Tuy nhiên, nếu chúng ta cần đảm bảo rằng child process nhận dữ liệu trên pipe ngay lập tức, thì chúng ta có thể sử dụng các lời gọi `fflush()` định kỳ hoặc vô hiệu hóa stdio buffering bằng lời gọi `setbuf(fp, NULL)`. Kỹ thuật này cũng có thể được sử dụng nếu chúng ta tạo pipe bằng system call `pipe()` và sau đó dùng `fdopen()` để lấy một stdio stream tương ứng với đầu ghi của pipe.

Nếu process gọi `popen()` đang đọc từ pipe (tức là `mode` là `r`), mọi thứ có thể không đơn giản như vậy. Trong trường hợp này, nếu child process đang sử dụng thư viện stdio, thì — trừ khi nó có các lời gọi rõ ràng đến `fflush()` hoặc `setbuf()` — output của nó sẽ chỉ có sẵn cho process gọi khi child lấp đầy stdio buffer hoặc gọi `fclose()`. (Tuyên bố tương tự áp dụng nếu chúng ta đang đọc từ pipe được tạo bằng `pipe()` và process ghi ở đầu kia đang sử dụng thư viện stdio.) Nếu đây là vấn đề, không có nhiều cách chúng ta có thể làm trừ khi chúng ta có thể sửa đổi source code của chương trình đang chạy trong child process để thêm các lời gọi `setbuf()` hoặc `fflush()`.

Nếu sửa đổi source code không phải là lựa chọn, thì thay vì sử dụng pipe, chúng ta có thể sử dụng pseudoterminal. Pseudoterminal là một kênh IPC xuất hiện với process ở một đầu như thể đó là terminal. Do đó, thư viện stdio line buffer output. Chúng ta mô tả pseudoterminal trong Chương 64.

# **44.7 FIFO**

Về mặt ngữ nghĩa, FIFO tương tự như pipe. Điểm khác biệt chính là FIFO có tên trong file system và được mở theo cùng cách như một file thông thường. Điều này cho phép FIFO được dùng để giao tiếp giữa các process không có quan hệ (ví dụ: client và server).

Sau khi FIFO được mở, chúng ta sử dụng cùng các system call I/O như được dùng với pipe và các file khác (tức là `read()`, `write()`, và `close()`). Giống như với pipe, FIFO có đầu ghi và đầu đọc, và dữ liệu được đọc từ pipe theo cùng thứ tự như được ghi. Thực tế này đặt tên cho FIFO: first in, first out (vào trước, ra trước). FIFO cũng đôi khi được gọi là named pipe.

Như với pipe, khi tất cả các descriptor tham chiếu đến FIFO đều được đóng, bất kỳ dữ liệu chưa được đọc nào đều bị loại bỏ.

Chúng ta có thể tạo FIFO từ shell bằng lệnh `mkfifo`:

```
$ mkfifo [ -m mode ] pathname
```

`pathname` là tên của FIFO cần tạo, và tùy chọn `–m` được dùng để chỉ định permission mode theo cùng cách như lệnh `chmod`.

Khi được áp dụng cho FIFO (hoặc pipe), `fstat()` và `stat()` trả về file type là `S_IFIFO` trong trường `st_mode` của cấu trúc `stat` (Mục 15.1). Khi được liệt kê bằng `ls –l`, FIFO được hiển thị với type `p` trong cột đầu tiên, và `ls –F` thêm ký hiệu pipe (`|`) vào pathname của FIFO.

Hàm `mkfifo()` tạo một FIFO mới với pathname đã cho.

```
#include <sys/stat.h>
int mkfifo(const char *pathname, mode_t mode);
                                            Returns 0 on success, or –1 on error
```

Đối số `mode` chỉ định các permission cho FIFO mới. Các permission này được chỉ định bằng cách OR các kết hợp mong muốn của các hằng số từ Bảng 15-4, trang 295. Như thường lệ, các permission này được masked với giá trị process `umask` (Mục 15.4.6).

> Theo lịch sử, FIFO được tạo bằng system call `mknod(pathname, S_IFIFO, 0)`. POSIX.1-1990 đã chỉ định `mkfifo()` như một API đơn giản hơn tránh tính tổng quát của `mknod()`, cho phép tạo nhiều loại file khác nhau, bao gồm file thiết bị. (SUSv3 chỉ định `mknod()`, nhưng yếu, chỉ định nghĩa việc sử dụng nó để tạo FIFO.) Hầu hết các UNIX implementation cung cấp `mkfifo()` như một hàm thư viện được xây dựng trên `mknod()`.

Sau khi FIFO được tạo, bất kỳ process nào cũng có thể mở nó, tùy thuộc vào các kiểm tra permission file thông thường (Mục 15.4.3).

Việc mở FIFO có ngữ nghĩa hơi bất thường. Nhìn chung, việc sử dụng FIFO hợp lý duy nhất là có một process đọc và một process ghi ở mỗi đầu. Do đó, theo mặc định, việc mở FIFO để đọc (flag `O_RDONLY` của `open()`) sẽ block cho đến khi một process khác mở FIFO để ghi (flag `O_WRONLY` của `open()`). Ngược lại, việc mở FIFO để ghi sẽ block cho đến khi một process khác mở FIFO để đọc. Nói cách khác, việc mở FIFO đồng bộ hóa các process đọc và ghi. Nếu đầu kia của FIFO đã được mở (có thể vì một cặp process đã mở mỗi đầu của FIFO), thì `open()` thành công ngay lập tức.

Trong hầu hết các UNIX implementation (bao gồm Linux), có thể tránh hành vi blocking khi mở FIFO bằng cách chỉ định flag `O_RDWR` khi mở FIFO. Trong trường hợp này, `open()` trả về ngay lập tức với file descriptor có thể được dùng để đọc và ghi trên FIFO. Làm điều này khá phá vỡ mô hình I/O cho FIFO, và SUSv3 ghi rõ rằng việc mở FIFO với flag `O_RDWR` là không được chỉ định; do đó, vì lý do portable, kỹ thuật này nên được tránh. Trong hoàn cảnh mà chúng ta cần ngăn blocking khi mở FIFO, flag `O_NONBLOCK` của `open()` cung cấp một phương pháp chuẩn hóa để làm điều đó (tham khảo Mục 44.9).

> Tránh sử dụng flag `O_RDWR` khi mở FIFO cũng có thể mong muốn vì một lý do khác. Sau một `open()` như vậy, process gọi sẽ không bao giờ thấy end-of-file khi đọc từ file descriptor kết quả, vì luôn có ít nhất một descriptor mở để ghi vào FIFO — cùng descriptor mà process đang đọc.

## **Sử dụng FIFO và `tee(1)` để tạo dual pipeline**

Một trong những đặc điểm của shell pipeline là chúng là tuyến tính; mỗi process trong pipeline đọc dữ liệu được tạo ra bởi tiền nhiệm của nó và gửi dữ liệu đến người kế nhiệm của nó. Sử dụng FIFO, có thể tạo một nhánh trong pipeline, để một bản sao của output của process được gửi đến process khác ngoài người kế nhiệm của nó trong pipeline. Để làm điều này, chúng ta cần sử dụng lệnh `tee`, ghi hai bản sao của những gì nó đọc từ standard input: một đến standard output và bản kia đến file được đặt tên trong đối số dòng lệnh của nó.

Đặt đối số file cho `tee` là một FIFO cho phép chúng ta có hai process đồng thời đọc output được nhân đôi được tạo ra bởi `tee`. Chúng ta minh họa điều này trong phiên shell sau, tạo một FIFO có tên `myfifo`, khởi động một lệnh `wc` nền mở FIFO để đọc (điều này sẽ block cho đến khi FIFO được mở để ghi), và sau đó thực thi một pipeline gửi output của `ls` đến `tee`, vừa truyền output tiếp theo trong pipeline đến `sort` vừa gửi nó đến FIFO `myfifo`. (Tùy chọn `–k5n` cho `sort` khiến output của `ls` được sắp xếp theo thứ tự số tăng dần trên trường phân cách bằng khoảng trắng thứ năm.)

```
$ mkfifo myfifo
$ wc -l < myfifo &
$ ls -l | tee myfifo | sort -k5n
(Resulting output not shown)
```

Chương trình `tee` được đặt tên như vậy vì hình dạng của nó. Chúng ta có thể coi `tee` hoạt động tương tự như một pipe, nhưng có thêm một nhánh gửi output được nhân đôi. Về mặt sơ đồ, hình dạng này giống chữ T viết hoa (xem Hình 44-5). Ngoài mục đích được mô tả ở đây, `tee` cũng hữu ích để debug pipeline và lưu kết quả được tạo ra tại một điểm trung gian trong một pipeline phức tạp.

![](_page_31_Figure_8.jpeg)

<span id="page-31-0"></span>**Hình 44-5:** Sử dụng FIFO và `tee(1)` để tạo dual pipeline

# **44.8 Ứng dụng Client-Server Sử dụng FIFO**

Trong mục này, chúng ta trình bày một ứng dụng client-server đơn giản sử dụng FIFO cho IPC. Server cung cấp dịch vụ (tầm thường) là cấp phát các số tuần tự duy nhất cho mỗi client yêu cầu chúng. Trong quá trình thảo luận về ứng dụng này, chúng ta giới thiệu một vài khái niệm và kỹ thuật trong thiết kế server.

### **Tổng quan về ứng dụng**

Trong ứng dụng ví dụ, tất cả client gửi yêu cầu của chúng đến server bằng cách sử dụng một server FIFO duy nhất. File header (Listing 44-6) định nghĩa tên well-known (`/tmp/seqnum_sv`) mà server sử dụng cho FIFO của nó. Tên này là cố định, để tất cả client biết cách liên hệ với server. (Trong ứng dụng ví dụ này, chúng ta tạo các FIFO trong thư mục `/tmp`, vì điều này cho phép chúng ta chạy các chương trình một cách thuận tiện mà không cần thay đổi trên hầu hết các hệ thống. Tuy nhiên, như đã lưu ý trong Mục 38.7, việc tạo file trong các thư mục có thể ghi công khai như `/tmp` có thể dẫn đến nhiều lỗ hổng bảo mật và nên tránh trong các ứng dụng thực tế.)

> Trong các ứng dụng client-server, chúng ta sẽ liên tục gặp khái niệm địa chỉ well-known hoặc tên được server sử dụng để làm cho dịch vụ của nó hiển thị với client. Sử dụng địa chỉ well-known là một giải pháp cho bài toán làm thế nào client biết cách liên hệ với server. Một giải pháp khả thi khác là cung cấp một loại name server mà với đó các server có thể đăng ký tên dịch vụ của chúng. Mỗi client sau đó liên hệ với name server để lấy vị trí của dịch vụ mà nó mong muốn. Giải pháp này cho phép vị trí của server linh hoạt, với chi phí lập trình bổ sung. Tất nhiên, client và server sau đó cần biết cách liên hệ với name server; thông thường, nó nằm ở một địa chỉ well-known.

Tuy nhiên, không thể sử dụng một FIFO duy nhất để gửi phản hồi cho tất cả client, vì nhiều client sẽ tranh nhau đọc từ FIFO và có thể đọc nhầm message phản hồi của client khác. Do đó, mỗi client tạo một FIFO duy nhất mà server sử dụng để giao hàng phản hồi cho client đó, và server cần biết cách tìm FIFO của mỗi client. Một cách có thể là client tạo tên FIFO của mình, sau đó truyền tên đó như một phần của message yêu cầu của nó. Ngoài ra, client và server có thể đồng ý về một quy ước để xây dựng tên FIFO của client, và như một phần của yêu cầu của nó, client có thể truyền cho server thông tin cần thiết để xây dựng tên đường dẫn cụ thể cho client này. Giải pháp sau này được sử dụng trong ví dụ của chúng ta. Tên FIFO của mỗi client được xây dựng từ một template (`CLIENT_FIFO_TEMPLATE`) bao gồm một pathname chứa process ID của client. Việc bao gồm process ID cung cấp một cách dễ dàng để tạo ra một tên duy nhất cho client này.

Hình 44-6 cho thấy cách ứng dụng này sử dụng FIFO để giao tiếp giữa các process client và server.

File header (Listing 44-6) định nghĩa các định dạng cho các message yêu cầu được gửi từ client đến server, và cho các message phản hồi được gửi từ server đến client.

![](_page_33_Figure_0.jpeg)

<span id="page-33-0"></span>Hình 44-6: Sử dụng FIFO trong ứng dụng một server, nhiều client

Hãy nhớ rằng dữ liệu trong pipe và FIFO là một byte stream; ranh giới giữa nhiều message không được bảo toàn. Điều này có nghĩa là khi nhiều message đang được giao đến một process duy nhất, chẳng hạn như server trong ví dụ của chúng ta, thì người gửi và người nhận phải đồng ý về một số quy ước để phân tách các message. Có nhiều cách tiếp cận:

- Kết thúc mỗi message bằng một *ký tự delimiter*, chẳng hạn như ký tự newline. (Ví dụ về kỹ thuật này, xem hàm `readLine()` trong Listing 59-1, trang 1201.) Trong trường hợp này, ký tự delimiter phải là ký tự không bao giờ xuất hiện như một phần của message, hoặc chúng ta phải áp dụng một quy ước để escape delimiter nếu nó xuất hiện trong message. Ví dụ: nếu chúng ta sử dụng delimiter newline, thì các ký tự `\` cộng với newline có thể được dùng để đại diện cho ký tự newline thực sự trong message, trong khi `\\` có thể đại diện cho `\` thực sự. Một nhược điểm của cách tiếp cận này là process đọc message phải quét dữ liệu từ FIFO từng byte một cho đến khi tìm thấy ký tự delimiter.
- Bao gồm *header có kích thước cố định với trường độ dài* trong mỗi message chỉ định số byte trong phần thay đổi còn lại của message. Trong trường hợp này, process đọc trước tiên đọc header từ FIFO, sau đó sử dụng trường độ dài của header để xác định số byte cần đọc cho phần còn lại của message. Cách tiếp cận này có ưu điểm là cho phép hiệu quả các message có kích thước tùy ý, nhưng có thể dẫn đến vấn đề nếu một message bị định dạng sai (ví dụ: trường *length* sai) được ghi vào pipe.
- Sử dụng *message có kích thước cố định*, và yêu cầu server luôn đọc các message có kích thước cố định này. Điều này có ưu điểm là đơn giản để lập trình. Tuy nhiên, nó đặt giới hạn trên cho kích thước message và có nghĩa là một số capacity kênh bị lãng phí (vì các message ngắn phải được đệm đến độ dài cố định). Hơn nữa, nếu một trong các client vô tình hoặc cố ý gửi một message có độ dài không đúng, thì tất cả các message tiếp theo sẽ bị lệch; trong tình huống này, server không thể dễ dàng phục hồi.

Ba kỹ thuật này được minh họa trong Hình 44-7. Lưu ý rằng đối với mỗi kỹ thuật này, tổng độ dài của mỗi message phải nhỏ hơn `PIPE_BUF` byte để tránh khả năng message bị kernel chia nhỏ và xen kẽ với message từ các writer khác.

Trong ba kỹ thuật được mô tả trong văn bản chính, một kênh duy nhất (FIFO) được sử dụng cho tất cả message từ tất cả client. Một giải pháp thay thế là sử dụng *một kết nối cho mỗi message*. Người gửi mở kênh giao tiếp, gửi message của mình, rồi đóng kênh. Process đọc biết rằng message hoàn chỉnh khi nó gặp end-of-file. Nếu nhiều writer giữ FIFO mở, thì cách tiếp cận này không khả thi, vì reader sẽ không thấy end-of-file khi một trong các writer đóng FIFO. Tuy nhiên, cách tiếp cận này khả thi khi sử dụng stream socket, nơi một server process tạo một kênh giao tiếp duy nhất cho mỗi kết nối client đến.

![](_page_34_Figure_1.jpeg)

<span id="page-34-1"></span>Hình 44-7: Phân tách message trong byte stream

Trong ứng dụng ví dụ của chúng ta, chúng ta sử dụng kỹ thuật thứ ba được mô tả ở trên, với mỗi client gửi message có kích thước cố định đến server. Message này được định nghĩa bởi cấu trúc `request` được định nghĩa trong Listing 44-6. Mỗi yêu cầu đến server bao gồm process ID của client, điều này cho phép server xây dựng tên của FIFO được client sử dụng để nhận phản hồi. Yêu cầu cũng chứa một trường (`seqLen`) chỉ định có bao nhiêu số tuần tự nên được cấp phát cho client này. Message phản hồi được gửi từ server đến client bao gồm một trường duy nhất, `seqNum`, là giá trị bắt đầu của phạm vi số tuần tự được cấp phát cho client này.

<span id="page-34-0"></span>**Listing 44-6:** File header cho `fifo_seqnum_server.c` và `fifo_seqnum_client.c`

```
pipes/fifo_seqnum.h
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
#define SERVER_FIFO "/tmp/seqnum_sv"
                                /* Well-known name for server's FIFO */
#define CLIENT_FIFO_TEMPLATE "/tmp/seqnum_cl.%ld"
                                /* Template for building client FIFO name */
#define CLIENT_FIFO_NAME_LEN (sizeof(CLIENT_FIFO_TEMPLATE) + 20)
                                /* Space required for client FIFO pathname
                                   (+20 as a generous allowance for the PID) */
struct request {
                                /* Request (client --> server) */
   pid_t pid;                  /* PID of client */
   int seqLen;                 /* Length of desired sequence */
};
struct response { /* Response (server --> client) */
 int seqNum; /* Start of sequence */
};
–––––––––––––––––––––––––––––––––––––––––––––––––––––– pipes/fifo_seqnum.h
```

### **Chương trình Server**

Listing 44-7 là code cho server. Server thực hiện các bước sau:

-  Tạo FIFO well-known của server và mở FIFO để đọc. Server phải được chạy trước bất kỳ client nào, để server FIFO tồn tại vào thời điểm client cố mở nó. `open()` của server sẽ block cho đến khi client đầu tiên mở đầu kia của server FIFO để ghi.
-  Mở server FIFO một lần nữa, lần này để ghi. Điều này sẽ không bao giờ block, vì FIFO đã được mở để đọc. Lần mở thứ hai này là một tiện ích để đảm bảo server không thấy end-of-file nếu tất cả client đóng đầu ghi của FIFO.
-  Ignore signal `SIGPIPE`, để nếu server cố ghi vào client FIFO không có reader, thì thay vì nhận signal `SIGPIPE` (mặc định giết process), nó nhận lỗi `EPIPE` từ system call `write()`.
-  Vào một vòng lặp đọc và phản hồi từng yêu cầu client đến. Để gửi phản hồi, server xây dựng tên của client FIFO và sau đó mở FIFO đó.
-  Nếu server gặp lỗi khi mở client FIFO, nó bỏ qua yêu cầu của client đó.

Đây là ví dụ về một iterative server, trong đó server đọc và xử lý từng yêu cầu client trước khi tiếp tục xử lý yêu cầu client tiếp theo. Thiết kế iterative server phù hợp khi mỗi yêu cầu client có thể được xử lý nhanh chóng và phản hồi, để các yêu cầu client khác không bị trì hoãn. Một thiết kế thay thế là concurrent server, trong đó process server chính sử dụng một child process (hoặc thread) riêng biệt để xử lý từng yêu cầu client. Chúng ta thảo luận thêm về thiết kế server trong Chương 60.

<span id="page-35-0"></span>**Listing 44-7:** Iterative server sử dụng FIFO

```
–––––––––––––––––––––––––––––––––––––––––––––––– pipes/fifo_seqnum_server.c
#include <signal.h>
#include "fifo_seqnum.h"
int
main(int argc, char *argv[])
{
 int serverFd, dummyFd, clientFd;
 char clientFifo[CLIENT_FIFO_NAME_LEN];
 struct request req;
 struct response resp;
 int seqNum = 0; /* This is our "service" */
 /* Create well-known FIFO, and open it for reading */
   umask(0); /* So we get the permissions we want */
q if (mkfifo(SERVER_FIFO, S_IRUSR | S_IWUSR | S_IWGRP) == -1
   && errno != EEXIST)
   errExit("mkfifo %s", SERVER_FIFO);
w serverFd = open(SERVER_FIFO, O_RDONLY);
   if (serverFd == -1)
   errExit("open %s", SERVER_FIFO);
   /* Open an extra write descriptor, so that we never see EOF */
e dummyFd = open(SERVER_FIFO, O_WRONLY);
   if (dummyFd == -1)
   errExit("open %s", SERVER_FIFO);
r if (signal(SIGPIPE, SIG_IGN) == SIG_ERR)
   errExit("signal");
t for (;;) { /* Read requests and send responses */
   if (read(serverFd, &req, sizeof(struct request))
   != sizeof(struct request)) {
   fprintf(stderr, "Error reading request; discarding\n");
   continue; /* Either partial read or error */
   }
   /* Open client FIFO (previously created by client) */
y snprintf(clientFifo, CLIENT_FIFO_NAME_LEN, CLIENT_FIFO_TEMPLATE,
   (long) req.pid);
u clientFd = open(clientFifo, O_WRONLY);
   if (clientFd == -1) { /* Open failed, give up on client */
   errMsg("open %s", clientFifo);
i continue;
   }
   /* Send response and close FIFO */
   resp.seqNum = seqNum;
   if (write(clientFd, &resp, sizeof(struct response))
   != sizeof(struct response))
   fprintf(stderr, "Error writing to FIFO %s\n", clientFifo);
   if (close(clientFd) == -1)
   errMsg("close");
   seqNum += req.seqLen; /* Update our sequence number */
   }
  }
  –––––––––––––––––––––––––––––––––––––––––––––––– pipes/fifo_seqnum_server.c
```

### **Chương trình Client**

Listing 44-8 là code cho client. Client thực hiện các bước sau:

-  Tạo một FIFO để nhận phản hồi từ server. Điều này được thực hiện trước khi gửi yêu cầu, để đảm bảo rằng FIFO tồn tại vào thời điểm server cố mở nó và gửi message phản hồi.
-  Xây dựng một message cho server chứa process ID của client và một số (lấy từ đối số dòng lệnh tùy chọn) chỉ định độ dài của chuỗi mà client muốn server cấp phát cho nó. (Nếu không có đối số dòng lệnh nào được cung cấp, độ dài chuỗi mặc định là 1.)
-  Mở server FIFO và gửi message đến server.
-  Mở client FIFO, đọc và in phản hồi của server.

Chi tiết đáng chú ý duy nhất khác là exit handler, được thiết lập với `atexit()`, đảm bảo rằng FIFO của client bị xóa khi process thoát. Ngoài ra, chúng ta có thể đã chỉ cần đặt lời gọi `unlink()` ngay sau `open()` của client FIFO. Điều này sẽ hoạt động vì tại thời điểm đó, sau khi cả hai đã thực hiện các lời gọi `open()` blocking, server và client đều giữ các file descriptor mở cho FIFO, và việc xóa tên FIFO khỏi file system không ảnh hưởng đến các descriptor này hoặc các open file description mà chúng tham chiếu.

Đây là ví dụ về những gì chúng ta thấy khi chạy các chương trình client và server:

```
$ ./fifo_seqnum_server &
[1] 5066
$ ./fifo_seqnum_client 3 Request a sequence of three numbers
0 Assigned sequence begins at 0
$ ./fifo_seqnum_client 2 Request a sequence of two numbers
3 Assigned sequence begins at 3
$ ./fifo_seqnum_client Request a single number
5
```

<span id="page-37-0"></span>**Listing 44-8:** Client cho sequence-number server

```
–––––––––––––––––––––––––––––––––––––––––––––––– pipes/fifo_seqnum_client.c
  #include "fifo_seqnum.h"
  static char clientFifo[CLIENT_FIFO_NAME_LEN];
  static void /* Invoked on exit to delete client FIFO */
q removeFifo(void)
  {
   unlink(clientFifo);
  }
  int
  main(int argc, char *argv[])
  {
   int serverFd, clientFd;
   struct request req;
   struct response resp;
 if (argc > 1 && strcmp(argv[1], "--help") == 0)
   usageErr("%s [seq-len...]\n", argv[0]);
   /* Create our FIFO (before sending request, to avoid a race) */
   umask(0); /* So we get the permissions we want */
w snprintf(clientFifo, CLIENT_FIFO_NAME_LEN, CLIENT_FIFO_TEMPLATE,
   (long) getpid());
   if (mkfifo(clientFifo, S_IRUSR | S_IWUSR | S_IWGRP) == -1
   && errno != EEXIST)
   errExit("mkfifo %s", clientFifo);
e if (atexit(removeFifo) != 0)
   errExit("atexit");
   /* Construct request message, open server FIFO, and send request */
r req.pid = getpid();
   req.seqLen = (argc > 1) ? getInt(argv[1], GN_GT_0, "seq-len") : 1;
t serverFd = open(SERVER_FIFO, O_WRONLY);
   if (serverFd == -1)
   errExit("open %s", SERVER_FIFO);
y if (write(serverFd, &req, sizeof(struct request)) !=
   sizeof(struct request))
   fatal("Can't write to server");
   /* Open our FIFO, read and display response */
u clientFd = open(clientFifo, O_RDONLY);
   if (clientFd == -1)
   errExit("open %s", clientFifo);
i if (read(clientFd, &resp, sizeof(struct response))
   != sizeof(struct response))
   fatal("Can't read response from server");
   printf("%d\n", resp.seqNum);
   exit(EXIT_SUCCESS);
  }
  –––––––––––––––––––––––––––––––––––––––––––––––– pipes/fifo_seqnum_client.c
```

# **44.9 I/O Nonblocking**

Như đã lưu ý trước đó, khi một process mở một đầu của FIFO, nó block nếu đầu kia của FIFO chưa được mở. Đôi khi, không mong muốn block, và vì mục đích này, flag `O_NONBLOCK` có thể được chỉ định khi gọi `open()`:

```
fd = open("fifopath", O_RDONLY | O_NONBLOCK);
if (fd == -1)
 errExit("open");
```

Nếu đầu kia của FIFO đã được mở, thì flag `O_NONBLOCK` không có tác dụng gì đến lời gọi `open()` — nó mở FIFO thành công ngay lập tức như thường. Flag `O_NONBLOCK` chỉ thay đổi mọi thứ khi đầu kia của FIFO chưa được mở, và tác dụng phụ thuộc vào việc chúng ta đang mở FIFO để đọc hay ghi:

-  Nếu FIFO đang được mở để đọc, và không có process nào hiện có đầu ghi của FIFO mở, thì lời gọi `open()` thành công ngay lập tức (giống như đầu kia của FIFO đã được mở).
-  Nếu FIFO đang được mở để ghi, và đầu kia của FIFO chưa được mở để đọc, thì `open()` thất bại, thiết lập `errno` thành `ENXIO`.

Sự bất đối xứng của flag `O_NONBLOCK` tùy thuộc vào việc FIFO đang được mở để đọc hay ghi có thể được giải thích như sau. Chấp nhận mở FIFO để đọc khi không có writer ở đầu kia của FIFO, vì bất kỳ lần thử đọc nào từ FIFO chỉ đơn giản không trả về dữ liệu. Tuy nhiên, cố ghi vào FIFO không có reader sẽ dẫn đến việc tạo ra signal `SIGPIPE` và lỗi `EPIPE` từ `write()`.

Bảng 44-1 tóm tắt ngữ nghĩa của việc mở FIFO, bao gồm các tác dụng của `O_NONBLOCK` được mô tả ở trên.

|          | Kiểu `open()`    |                              | Kết quả `open()`         |
|----------|------------------|------------------------------|--------------------------|
| mở cho   | flag bổ sung     | đầu kia của FIFO đang mở     | đầu kia của FIFO đóng    |
|          | không (blocking) | thành công ngay lập tức      | blocks                   |
| đọc      | O_NONBLOCK       | thành công ngay lập tức      | thành công ngay lập tức  |
|          | không (blocking) | thành công ngay lập tức      | blocks                   |
| ghi      | O_NONBLOCK       | thành công ngay lập tức      | thất bại (ENXIO)         |

**Bảng 44-1:** Ngữ nghĩa của `open()` cho FIFO

Sử dụng flag `O_NONBLOCK` khi mở FIFO phục vụ hai mục đích chính:

-  Nó cho phép một process duy nhất mở cả hai đầu của FIFO. Process trước tiên mở FIFO để đọc chỉ định `O_NONBLOCK`, sau đó mở FIFO để ghi.
-  Nó ngăn deadlock giữa các process mở hai FIFO.

Deadlock là tình huống hai hoặc nhiều process bị block vì mỗi process đang chờ process kia hoàn thành một hành động nào đó. Hai process được hiển thị trong Hình 44-8 đang trong tình trạng deadlock. Mỗi process bị block chờ mở một FIFO để đọc. Việc block này sẽ không xảy ra nếu mỗi process có thể thực hiện bước thứ hai của nó (mở FIFO kia để ghi). Vấn đề deadlock cụ thể này có thể được giải quyết bằng cách đảo ngược thứ tự các bước 1 và 2 trong process Y, trong khi để nguyên thứ tự trong process X, hoặc ngược lại. Tuy nhiên, sắp xếp các bước như vậy có thể không dễ thực hiện trong một số ứng dụng. Thay vào đó, chúng ta có thể giải quyết vấn đề bằng cách yêu cầu một trong hai process, hoặc cả hai, chỉ định flag `O_NONBLOCK` khi mở các FIFO để đọc.

| Process X                                   | Process Y                                   |
|---------------------------------------------|---------------------------------------------|
| 1. Mở FIFO A để đọc<br>blocks              | 1. Mở FIFO B để đọc<br>blocks              |
| 2. Mở FIFO B để ghi                         | 2. Mở FIFO A để ghi                         |

<span id="page-40-0"></span>**Hình 44-8:** Deadlock giữa các process mở hai FIFO

## **`read()` và `write()` Nonblocking**

Flag `O_NONBLOCK` không chỉ ảnh hưởng đến ngữ nghĩa của `open()` mà còn — vì flag sau đó vẫn được đặt cho open file description — ngữ nghĩa của các lời gọi `read()` và `write()` tiếp theo. Chúng ta mô tả các tác dụng này trong mục tiếp theo.

Đôi khi, chúng ta cần thay đổi trạng thái của flag `O_NONBLOCK` cho một FIFO (hoặc loại file khác) đã được mở. Các kịch bản có thể phát sinh bao gồm:

-  Chúng ta mở FIFO bằng `O_NONBLOCK`, nhưng muốn các lời gọi `read()` và `write()` tiếp theo hoạt động trong chế độ blocking.
-  Chúng ta muốn bật nonblocking mode cho file descriptor được trả về bởi `pipe()`. Tổng quát hơn, chúng ta có thể muốn thay đổi trạng thái nonblocking của bất kỳ file descriptor nào được lấy theo cách khác ngoài lời gọi `open()` — ví dụ: một trong ba descriptor tiêu chuẩn được tự động mở cho mỗi chương trình mới được shell chạy hoặc file descriptor được trả về bởi `socket()`.
-  Vì một mục đích cụ thể của ứng dụng, chúng ta cần bật tắt cài đặt `O_NONBLOCK` của file descriptor.

Vì các mục đích này, chúng ta có thể sử dụng `fcntl()` để bật hoặc tắt flag trạng thái file mở `O_NONBLOCK`. Để bật flag, chúng ta viết như sau (bỏ qua kiểm tra lỗi):

```
int flags;
   flags = fcntl(fd, F_GETFL); /* Fetch open files status flags */
   flags |= O_NONBLOCK; /* Enable O_NONBLOCK bit */
   fcntl(fd, F_SETFL, flags); /* Update open files status flags */
Và để tắt, chúng ta viết như sau:
   flags = fcntl(fd, F_GETFL);
   flags &= ~O_NONBLOCK; /* Disable O_NONBLOCK bit */
   fcntl(fd, F_SETFL, flags);
```

# **44.10 Ngữ nghĩa của `read()` và `write()` trên Pipe và FIFO**

Bảng 44-2 tóm tắt hoạt động của `read()` cho pipe và FIFO, và bao gồm tác dụng của flag `O_NONBLOCK`.

Sự khác biệt duy nhất giữa blocking và nonblocking reads xảy ra khi không có dữ liệu nào và đầu ghi đang mở. Trong trường hợp này, `read()` thông thường sẽ block, trong khi `read()` nonblocking thất bại với lỗi `EAGAIN`.

**Bảng 44-2:** Ngữ nghĩa của việc đọc n byte từ pipe hoặc FIFO chứa p byte

| O_NONBLOCK | Byte dữ liệu có trong pipe hoặc FIFO (p) |                            |              |              |
|------------|------------------------------------------|----------------------------|--------------|--------------|
| bật?       | p = 0, đầu ghi mở                        | p = 0, đầu ghi đóng        | p < n        | p >= n       |
| Không      | block                                    | trả về 0 (EOF)             | đọc p byte   | đọc n byte   |
| Có         | thất bại (EAGAIN)                        | trả về 0 (EOF)             | đọc p byte   | đọc n byte   |

Tác động của flag `O_NONBLOCK` khi ghi vào pipe hoặc FIFO trở nên phức tạp do tương tác với giới hạn `PIPE_BUF`. Hành vi của `write()` được tóm tắt trong Bảng 44-3.

**Bảng 44-3:** Ngữ nghĩa của việc ghi n byte vào pipe hoặc FIFO

| O_NONBLOCK | Đầu đọc mở                                                                                                                                         |                                                                                                                                                                                                                           | Đầu đọc     |
|------------|----------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------|
| bật?       | n <= PIPE_BUF                                                                                                                                      | n > PIPE_BUF                                                                                                                                                                                                              | đóng        |
| Không      | Ghi atomic n byte;<br>có thể block cho đến khi đủ<br>dữ liệu được đọc để `write()`<br>được thực hiện                                              | Ghi n byte; có thể block cho đến khi<br>đủ dữ liệu được đọc để `write()` hoàn thành;<br>dữ liệu có thể bị xen kẽ với các lần ghi<br>của các process khác                                                               |             |
| Có         | Nếu có đủ space để ghi ngay lập<br>tức n byte, thì `write()` thành<br>công atomic; nếu không, thất bại<br>(EAGAIN)                                 | Nếu có đủ space để ghi ngay lập tức<br>một số byte, thì ghi từ 1 đến n byte<br>(có thể bị xen kẽ với dữ liệu được ghi<br>bởi các process khác); nếu không,<br>`write()` thất bại (EAGAIN)                               | SIGPIPE<br>+<br>EPIPE |

Flag `O_NONBLOCK` khiến `write()` trên pipe hoặc FIFO thất bại (với lỗi `EAGAIN`) trong bất kỳ trường hợp nào mà dữ liệu không thể được chuyển ngay lập tức. Điều này có nghĩa là nếu chúng ta đang ghi tối đa `PIPE_BUF` byte, thì `write()` sẽ thất bại nếu không có đủ space trong pipe hoặc FIFO, vì kernel không thể hoàn thành thao tác ngay lập tức và không thể thực hiện partial write, vì điều đó sẽ vi phạm yêu cầu rằng các lần ghi tối đa `PIPE_BUF` byte là atomic.

Khi ghi nhiều hơn `PIPE_BUF` byte mỗi lần, việc ghi không bắt buộc phải atomic. Vì lý do này, `write()` chuyển càng nhiều byte càng tốt (partial write) để lấp đầy pipe hoặc FIFO. Trong trường hợp này, giá trị trả về từ `write()` là số byte thực sự được chuyển, và người gọi phải thử lại sau để ghi các byte còn lại. Tuy nhiên, nếu pipe hoặc FIFO đầy, không thể chuyển được dù chỉ một byte, thì `write()` thất bại với lỗi `EAGAIN`.

## **44.11 Tóm tắt**

Pipe là phương thức IPC đầu tiên trên hệ thống UNIX, và chúng được sử dụng thường xuyên bởi shell, cũng như trong các ứng dụng khác. Pipe là unidirectional, limited-capacity byte stream có thể được dùng để giao tiếp giữa các process có quan hệ. Mặc dù các khối dữ liệu có kích thước bất kỳ có thể được ghi vào pipe, nhưng chỉ các lần ghi không vượt quá `PIPE_BUF` byte mới được đảm bảo là atomic. Ngoài việc được sử dụng như một phương thức IPC, pipe cũng có thể được dùng để đồng bộ hóa process.

Khi sử dụng pipe, chúng ta phải cẩn thận để đóng các descriptor không dùng nhằm đảm bảo rằng các process đọc phát hiện end-of-file và các process ghi nhận signal `SIGPIPE` hoặc lỗi `EPIPE`. (Thường thì cách đơn giản nhất là để ứng dụng ghi vào pipe ignore `SIGPIPE` và phát hiện pipe "bị vỡ" thông qua lỗi `EPIPE`.)

Các hàm `popen()` và `pclose()` cho phép chương trình truyền dữ liệu đến hoặc từ một lệnh shell tiêu chuẩn, mà không cần xử lý các chi tiết về tạo pipe, exec shell, và đóng các file descriptor không dùng.

FIFO hoạt động theo cùng cách như pipe, ngoại trừ việc chúng được tạo bằng `mkfifo()`, có tên trong file system, và có thể được mở bởi bất kỳ process nào có quyền thích hợp. Theo mặc định, việc mở FIFO để đọc sẽ block cho đến khi một process khác mở FIFO để ghi, và ngược lại.

Trong quá trình chương này, chúng ta đã xem xét một số chủ đề liên quan. Trước tiên, chúng ta thấy cách nhân đôi file descriptor sao cho standard input hoặc output của một filter có thể được gắn với pipe. Trong khi trình bày một ví dụ client-server sử dụng FIFO, chúng ta đề cập đến một số chủ đề trong thiết kế client-server, bao gồm việc sử dụng địa chỉ well-known cho server và thiết kế iterative so với concurrent server. Trong quá trình phát triển ứng dụng FIFO ví dụ, chúng ta lưu ý rằng mặc dù dữ liệu được truyền qua pipe là byte stream, nhưng đôi khi hữu ích cho các process giao tiếp để đóng gói dữ liệu thành message, và chúng ta đã xem xét các cách khác nhau để thực hiện điều này.

Cuối cùng, chúng ta lưu ý tác dụng của flag `O_NONBLOCK` (I/O nonblocking) khi mở và thực hiện I/O trên FIFO. Flag `O_NONBLOCK` hữu ích nếu chúng ta không muốn block khi mở FIFO. Nó cũng hữu ích nếu chúng ta không muốn đọc block nếu không có dữ liệu, hoặc ghi block nếu không có đủ space trong pipe hoặc FIFO.

## **Thông tin thêm**

Việc thực thi pipe được thảo luận trong [Bach, 1986] và [Bovet & Cesati, 2005]. Các chi tiết hữu ích về pipe và FIFO cũng có thể được tìm thấy trong [Vahalia, 1996].

# **44.12 Bài tập**

- **44-1.** Viết một chương trình sử dụng hai pipe để cho phép giao tiếp hai chiều giữa parent và child process. Process parent nên lặp đọc một khối văn bản từ standard input và sử dụng một trong các pipe để gửi văn bản đó đến child, child chuyển đổi nó thành chữ hoa và gửi lại cho parent qua pipe kia. Parent đọc dữ liệu từ child và echo nó ra standard output trước khi tiếp tục vòng lặp.
- **44-2.** Triển khai `popen()` và `pclose()`. Mặc dù các hàm này được đơn giản hóa bằng cách không yêu cầu signal handling như trong triển khai của `system()` (Mục 27.7), bạn sẽ cần cẩn thận để gắn đúng các đầu pipe với file stream trong mỗi process, và đảm bảo rằng tất cả các descriptor không dùng tham chiếu đến các đầu pipe đều được đóng. Vì các child được tạo bởi nhiều lời gọi `popen()` có thể đang chạy cùng một lúc, bạn sẽ cần duy trì một cấu trúc dữ liệu kết hợp các con trỏ file stream được cấp phát bởi `popen()` với các process ID của child tương ứng. (Nếu sử dụng mảng cho mục đích này, giá trị được trả về bởi hàm `fileno()`, lấy file descriptor tương ứng với file stream, có thể được dùng để lập chỉ mục mảng.) Lấy đúng process ID từ cấu trúc này sẽ cho phép `pclose()` chọn child để wait. Cấu trúc này cũng sẽ hỗ trợ với yêu cầu SUSv3 rằng bất kỳ file stream nào vẫn còn mở được tạo bởi các lời gọi `popen()` trước đó phải được đóng trong child process mới.
- **44-3.** Server trong Listing 44-7 (`fifo_seqnum_server.c`) luôn bắt đầu cấp phát số tuần tự từ 0 mỗi khi khởi động. Sửa đổi chương trình để sử dụng file backup được cập nhật mỗi khi số tuần tự được cấp phát. (Flag `O_SYNC` của `open()`, được mô tả trong Mục 4.3.1, có thể hữu ích.) Khi khởi động, chương trình nên kiểm tra sự tồn tại của file này, và nếu có, sử dụng giá trị trong đó để khởi tạo số tuần tự. Nếu không thể tìm thấy file backup khi khởi động, chương trình nên tạo file mới và bắt đầu cấp phát số tuần tự bắt đầu từ 0. (Một kỹ thuật thay thế cho cách này là sử dụng memory-mapped file, được mô tả trong Chương 49.)
- **44-4.** Thêm code vào server trong Listing 44-7 (`fifo_seqnum_server.c`) để nếu chương trình nhận các signal `SIGINT` hoặc `SIGTERM`, nó xóa server FIFO và kết thúc.
- **44-5.** Server trong Listing 44-7 (`fifo_seqnum_server.c`) thực hiện lần mở `O_WRONLY` thứ hai của FIFO để nó không bao giờ thấy end-of-file khi đọc từ read descriptor (`serverFd`) của FIFO. Thay vì làm điều này, có thể thử một cách tiếp cận thay thế: bất cứ khi nào server thấy end-of-file trên read descriptor, nó đóng descriptor, và sau đó một lần nữa mở FIFO để đọc. (Lần mở này sẽ block cho đến khi client tiếp theo mở FIFO để ghi.) Điều gì sai với cách tiếp cận này?
- **44-6.** Server trong Listing 44-7 (`fifo_seqnum_server.c`) giả định rằng client process hoạt động đúng. Nếu một client hành xử sai tạo một client FIFO và gửi yêu cầu đến server, nhưng không mở FIFO của nó, thì lần thử của server để mở client FIFO sẽ block, và các yêu cầu của client khác sẽ bị trì hoãn vô hạn. (Nếu được thực hiện cố ý, đây sẽ là một cuộc tấn công từ chối dịch vụ.) Thiết kế một phương án để đối phó với vấn đề này. Mở rộng server (và có thể cả client trong Listing 44-8) theo đó.
- **44-7.** Viết các chương trình để xác minh hoạt động của nonblocking open và nonblocking I/O trên FIFO (xem Mục 44.9).
