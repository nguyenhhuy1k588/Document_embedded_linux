## Chương 21
# **SIGNALS: SIGNAL HANDLERS**

Chương này tiếp nối phần mô tả về signal đã bắt đầu ở chương trước. Nội dung tập trung vào signal handler, mở rộng thảo luận đã bắt đầu từ Mục 20.4. Các chủ đề được đề cập bao gồm:

-  cách thiết kế một signal handler, đòi hỏi phải thảo luận về tính reentrant và các async-signal-safe function;
-  các phương án thay thế cho việc thực hiện lệnh return thông thường từ signal handler, đặc biệt là sử dụng nonlocal goto cho mục đích này;
-  xử lý signal trên một alternate stack;
-  sử dụng flag `SA_SIGINFO` của `sigaction()` để cho phép signal handler lấy thêm thông tin chi tiết về signal đã kích hoạt nó; và
-  cách một blocking system call có thể bị gián đoạn bởi signal handler, và cách restart lại lệnh gọi đó nếu cần.

## **21.1 Thiết kế Signal Handler**

Nói chung, nên viết các signal handler đơn giản. Một lý do quan trọng là để giảm nguy cơ tạo ra race condition. Hai thiết kế phổ biến cho signal handler là:

-  Signal handler đặt một global flag và thoát. Chương trình chính định kỳ kiểm tra flag này, và nếu được bật, thực hiện hành động thích hợp. (Nếu chương trình chính không thể thực hiện các kiểm tra định kỳ như vậy vì cần giám sát một hoặc nhiều file descriptor để xem liệu có I/O nào khả dụng không, thì signal handler cũng có thể ghi một byte duy nhất vào một pipe chuyên dụng mà đầu đọc của nó được đưa vào danh sách file descriptor được giám sát bởi chương trình chính. Chúng ta sẽ thấy ví dụ về kỹ thuật này ở Mục 63.5.2.)
-  Signal handler thực hiện một số dọn dẹp rồi hoặc kết thúc process, hoặc sử dụng nonlocal goto (Mục [21.2.1](#page-8-0)) để unwind stack và trả quyền điều khiển về một vị trí xác định trước trong chương trình chính.

Trong các phần sau, chúng ta sẽ khám phá các ý tưởng này cũng như các khái niệm khác quan trọng trong việc thiết kế signal handler.

## **21.1.1 Signal Không Được Xếp Hàng (Nhắc Lại)**

Ở Mục 20.10, chúng ta đã lưu ý rằng việc gửi signal bị block trong quá trình thực thi handler của nó (trừ khi chúng ta chỉ định flag `SA_NODEFER` cho `sigaction()`). Nếu signal được tạo ra (một lần nữa) trong khi handler đang thực thi, thì nó được đánh dấu là pending và được gửi sau khi handler trả về. Chúng ta cũng đã lưu ý rằng signal không được xếp hàng. Nếu signal được tạo ra nhiều hơn một lần trong khi handler đang thực thi, thì nó vẫn chỉ được đánh dấu là pending, và sau đó sẽ chỉ được gửi một lần duy nhất.

Việc signal có thể "biến mất" theo cách này có ảnh hưởng đến cách thiết kế signal handler. Trước tiên, chúng ta không thể đếm chính xác số lần một signal được tạo ra. Hơn nữa, chúng ta có thể cần lập trình signal handler để xử lý trường hợp nhiều sự kiện thuộc loại tương ứng với signal đó đã xảy ra. Chúng ta sẽ thấy ví dụ về điều này khi xem xét việc sử dụng signal `SIGCHLD` ở Mục 26.3.1.

## <span id="page-1-0"></span>**21.1.2 Reentrant và Async-Signal-Safe Function**

<span id="page-1-1"></span>Không phải tất cả system call và library function đều có thể được gọi một cách an toàn từ signal handler. Để hiểu tại sao cần giải thích hai khái niệm: reentrant function và async-signal-safe function.

#### **Reentrant và nonreentrant function**

Để giải thích reentrant function là gì, trước tiên chúng ta cần phân biệt giữa chương trình single-threaded và multithreaded. Các chương trình UNIX cổ điển có một thread thực thi duy nhất: CPU xử lý các lệnh theo một luồng thực thi logic duy nhất qua chương trình. Trong một chương trình multithreaded, có nhiều luồng thực thi logic độc lập, đồng thời trong cùng một process.

Trong Chương 29, chúng ta sẽ thấy cách tạo rõ ràng các chương trình chứa nhiều thread thực thi. Tuy nhiên, khái niệm về nhiều thread thực thi cũng liên quan đến các chương trình sử dụng signal handler. Vì signal handler có thể ngắt không đồng bộ quá trình thực thi của chương trình tại bất kỳ thời điểm nào, chương trình chính và signal handler thực chất tạo thành hai thread thực thi độc lập (dù không đồng thời) trong cùng một process.

Một function được gọi là reentrant nếu nó có thể được thực thi đồng thời một cách an toàn bởi nhiều thread thực thi trong cùng một process. Trong ngữ cảnh này, "an toàn" có nghĩa là function đạt được kết quả mong đợi, bất kể trạng thái thực thi của bất kỳ thread thực thi nào khác.

> Định nghĩa của SUSv3 về reentrant function là một function "mà hiệu ứng của nó, khi được gọi bởi hai hay nhiều thread, được đảm bảo là như thể các thread mỗi cái đã thực thi function theo thứ tự lần lượt không xác định, ngay cả khi thực tế thực thi là xen kẽ nhau."

Một function có thể là nonreentrant nếu nó cập nhật global hoặc static data structure. (Một function chỉ sử dụng biến cục bộ được đảm bảo là reentrant.) Nếu hai lần gọi (tức là hai thread thực thi) function đồng thời cố gắng cập nhật cùng một biến global hoặc data structure, thì những cập nhật này có thể gây xung đột và tạo ra kết quả không chính xác. Ví dụ, giả sử một thread đang trong quá trình cập nhật một linked list để thêm một phần tử mới khi một thread khác cũng cố gắng cập nhật cùng linked list đó. Vì thêm một phần tử mới vào danh sách đòi hỏi cập nhật nhiều pointer, nếu thread khác ngắt các bước này và cập nhật cùng các pointer đó, kết quả sẽ là hỗn loạn.

Những khả năng như vậy thực sự rất phổ biến trong thư viện C chuẩn. Ví dụ, chúng ta đã lưu ý ở Mục 7.1.3 rằng `malloc()` và `free()` duy trì một linked list các khối bộ nhớ đã giải phóng sẵn sàng để tái phân bổ từ heap. Nếu một lệnh gọi `malloc()` trong chương trình chính bị ngắt bởi signal handler mà cũng gọi `malloc()`, thì linked list này có thể bị hỏng. Vì lý do này, họ hàm `malloc()`, và các library function khác sử dụng chúng, là nonreentrant.

Các library function khác là nonreentrant vì chúng trả về thông tin bằng cách sử dụng bộ nhớ được cấp phát tĩnh. Ví dụ về các function như vậy (được mô tả ở chỗ khác trong cuốn sách này) bao gồm `crypt()`, `getpwnam()`, `gethostbyname()`, và `getservbyname()`. Nếu signal handler cũng sử dụng một trong các function này, thì nó sẽ ghi đè thông tin được trả về bởi bất kỳ lời gọi trước đó nào đến cùng function từ bên trong chương trình chính (hoặc ngược lại).

Các function cũng có thể là nonreentrant nếu chúng sử dụng static data structure cho việc bookkeeping nội bộ. Các ví dụ rõ ràng nhất về các function như vậy là các thành viên của thư viện stdio (`printf()`, `scanf()`, v.v.), những thứ cập nhật data structure nội bộ cho buffered I/O. Do đó, khi sử dụng `printf()` từ bên trong signal handler, đôi khi chúng ta có thể thấy đầu ra kỳ lạ — hoặc thậm chí crash chương trình hoặc data corruption nếu handler ngắt chương trình chính ở giữa quá trình thực thi lời gọi `printf()` hoặc một stdio function khác.

Ngay cả khi chúng ta không sử dụng các nonreentrant library function, các vấn đề về reentrancy vẫn có thể liên quan. Nếu signal handler cập nhật các global data structure do lập trình viên định nghĩa mà cũng được cập nhật trong chương trình chính, thì chúng ta có thể nói signal handler là nonreentrant so với chương trình chính.

Nếu một function là nonreentrant, trang manual của nó thường sẽ cung cấp chỉ dẫn rõ ràng hoặc ngầm định về điều này. Đặc biệt, hãy chú ý đến các phát biểu nói rằng function sử dụng hoặc trả về thông tin trong các biến được cấp phát tĩnh.

#### **Chương trình ví dụ**

[Listing 21-1](#page-3-0) minh họa tính nonreentrant của function `crypt()` (Mục 8.5). Như các đối số dòng lệnh, chương trình này nhận hai chuỗi. Chương trình thực hiện các bước sau:

- 1. Gọi `crypt()` để mã hóa chuỗi trong đối số dòng lệnh đầu tiên, và sao chép chuỗi này vào một buffer riêng bằng cách sử dụng `strdup()`.
- 2. Thiết lập một handler cho `SIGINT` (được tạo bằng cách nhấn Control-C). Handler gọi `crypt()` để mã hóa chuỗi được cung cấp trong đối số dòng lệnh thứ hai.
- 3. Bước vào vòng lặp for vô hạn sử dụng `crypt()` để mã hóa chuỗi trong đối số dòng lệnh đầu tiên và kiểm tra xem chuỗi được trả về có giống với chuỗi đã lưu ở bước 1 không.

Khi không có signal, các chuỗi sẽ luôn khớp nhau ở bước 3. Tuy nhiên, nếu signal `SIGINT` đến và quá trình thực thi signal handler ngắt chương trình chính ngay sau khi thực thi lời gọi `crypt()` trong vòng lặp for, nhưng trước khi kiểm tra xem các chuỗi có khớp nhau không, thì chương trình chính sẽ báo cáo không khớp. Khi chạy chương trình, đây là những gì chúng ta thấy:

#### \$ **./non\_reentrant abc def**

```
Repeatedly type Control-C to generate SIGINT
Mismatch on call 109871 (mismatch=1 handled=1)
Mismatch on call 128061 (mismatch=2 handled=2)
Many lines of output removed
Mismatch on call 727935 (mismatch=149 handled=156)
Mismatch on call 729547 (mismatch=150 handled=157)
Type Control-\ to generate SIGQUIT
Quit (core dumped)
```

So sánh các giá trị mismatch và handled trong đầu ra trên, chúng ta thấy rằng trong phần lớn các trường hợp signal handler được gọi, nó ghi đè buffer được cấp phát tĩnh giữa lệnh gọi `crypt()` và so sánh chuỗi trong `main()`.

<span id="page-3-0"></span>**Listing 21-1:** Gọi một nonreentrant function từ cả `main()` và signal handler

–––––––––––––––––––––––––––––––––––––––––––––––––––– **signals/nonreentrant.c** #define \_XOPEN\_SOURCE 600 #include <unistd.h> #include <signal.h> #include <string.h> #include "tlpi\_hdr.h" static char \*str2; /\* Set from argv[2] \*/ static int handled = 0; /\* Counts number of calls to handler \*/ static void handler(int sig) { crypt(str2, "xx"); handled++; }

```
int
main(int argc, char *argv[])
{
 char *cr1;
 int callNum, mismatch;
 struct sigaction sa;
 if (argc != 3)
 usageErr("%s str1 str2\n", argv[0]);
 str2 = argv[2]; /* Make argv[2] available to handler */
 cr1 = strdup(crypt(argv[1], "xx")); /* Copy statically allocated string
 to another buffer */
 if (cr1 == NULL)
 errExit("strdup");
 sigemptyset(&sa.sa_mask);
 sa.sa_flags = 0;
 sa.sa_handler = handler;
 if (sigaction(SIGINT, &sa, NULL) == -1)
 errExit("sigaction");
 /* Repeatedly call crypt() using argv[1]. If interrupted by a
 signal handler, then the static storage returned by crypt()
 will be overwritten by the results of encrypting argv[2], and
 strcmp() will detect a mismatch with the value in 'cr1'. */
 for (callNum = 1, mismatch = 0; ; callNum++) {
 if (strcmp(crypt(argv[1], "xx"), cr1) != 0) {
 mismatch++;
 printf("Mismatch on call %d (mismatch=%d handled=%d)\n",
 callNum, mismatch, handled);
 }
 }
}
```

–––––––––––––––––––––––––––––––––––––––––––––––––––– **signals/nonreentrant.c**

#### **Các async-signal-safe function chuẩn**

Một async-signal-safe function là một function mà implementation đảm bảo là an toàn khi được gọi từ signal handler. Một function là async-signal-safe hoặc vì nó là reentrant, hoặc vì nó không thể bị ngắt bởi signal handler.

Bảng 21-1 liệt kê các function mà các tiêu chuẩn khác nhau yêu cầu phải là async-signal-safe. Trong bảng này, các function có tên không theo sau bởi v2 hoặc v3 đã được chỉ định là async-signal-safe trong POSIX.1-1990. SUSv2 đã thêm các function được đánh dấu v2 vào danh sách, và những function được đánh dấu v3 đã được SUSv3 thêm vào. Các implementation UNIX riêng lẻ có thể làm cho các function khác là async-signal-safe, nhưng tất cả các implementation UNIX tuân theo chuẩn phải đảm bảo rằng ít nhất các function này là async-signal-safe (nếu chúng được cung cấp bởi implementation; không phải tất cả các function này đều được cung cấp trên Linux).

SUSv4 thực hiện các thay đổi sau đối với Bảng 21-1:

 Các function sau đây được loại bỏ: `fpathconf()`, `pathconf()`, và `sysconf()`.

 Các function sau đây được thêm vào: `execl()`, `execv()`, `faccessat()`, `fchmodat()`, `fchownat()`, `fexecve()`, `fstatat()`, `futimens()`, `linkat()`, `mkdirat()`, `mkfifoat()`, `mknod()`, `mknodat()`, `openat()`, `readlinkat()`, `renameat()`, `symlinkat()`, `unlinkat()`, `utimensat()`, và `utimes()`.

**Bảng 21-1:** Các function được yêu cầu là async-signal-safe theo POSIX.1-1990, SUSv2 và SUSv3

| _Exit()<br>(v3)<br>_exit()<br>abort() (v3)<br>accept() (v3)<br>access()<br>aio_error() (v2)<br>aio_return() (v2) | getpid()<br>getppid()<br>getsockname() (v3)<br>getsockopt() (v3)<br>getuid()<br>kill()<br>link()<br>listen() (v3) | sigdelset()<br>sigemptyset()<br>sigfillset()<br>sigismember()<br>signal() (v2)<br>sigpause() (v2)<br>sigpending() |
|------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
|                                                                                                                  |                                                                                                                   |                                                                                                                   |
| aio_suspend() (v2)                                                                                               |                                                                                                                   | sigprocmask()                                                                                                     |
| alarm()                                                                                                          | lseek()                                                                                                           | sigqueue() (v2)                                                                                                   |
| bind()<br>(v3)                                                                                                   | lstat() (v3)                                                                                                      | sigset() (v2)                                                                                                     |
| cfgetispeed()                                                                                                    | mkdir()                                                                                                           | sigsuspend()                                                                                                      |
| cfgetospeed()                                                                                                    | mkfifo()                                                                                                          | sleep()                                                                                                           |
| cfsetispeed()                                                                                                    | open()                                                                                                            | socket() (v3)                                                                                                     |
| cfsetospeed()                                                                                                    | pathconf()                                                                                                        | sockatmark() (v3)                                                                                                 |
| chdir()                                                                                                          | pause()                                                                                                           | socketpair() (v3)                                                                                                 |
| chmod()                                                                                                          | pipe()                                                                                                            | stat()                                                                                                            |
| chown()                                                                                                          | poll() (v3)                                                                                                       | symlink() (v3)                                                                                                    |
| clock_gettime() (v2)                                                                                             | posix_trace_event() (v3)                                                                                          | sysconf()                                                                                                         |
| close()                                                                                                          | pselect() (v3)                                                                                                    | tcdrain()                                                                                                         |
| connect() (v3)                                                                                                   | raise() (v2)                                                                                                      | tcflow()                                                                                                          |
| creat()                                                                                                          | read()                                                                                                            | tcflush()                                                                                                         |
| dup()                                                                                                            | readlink() (v3)                                                                                                   | tcgetattr()                                                                                                       |
| dup2()                                                                                                           | recv() (v3)                                                                                                       | tcgetpgrp()                                                                                                       |
| execle()                                                                                                         | recvfrom() (v3)                                                                                                   | tcsendbreak()                                                                                                     |
| execve()                                                                                                         | recvmsg() (v3)                                                                                                    | tcsetattr()                                                                                                       |
| fchmod() (v3)                                                                                                    | rename()                                                                                                          | tcsetpgrp()                                                                                                       |
| fchown() (v3)                                                                                                    | rmdir()                                                                                                           | time()                                                                                                            |
| fcntl()                                                                                                          | select() (v3)                                                                                                     | timer_getoverrun() (v2)                                                                                           |
| fdatasync() (v2)                                                                                                 | sem_post() (v2)                                                                                                   | timer_gettime() (v2)                                                                                              |
| fork()                                                                                                           | send()<br>(v3)                                                                                                    | timer_settime() (v2)                                                                                              |
| fpathconf() (v2)                                                                                                 | sendmsg()<br>(v3)                                                                                                 | times()                                                                                                           |
| fstat()                                                                                                          | sendto() (v3)                                                                                                     | umask()                                                                                                           |
| fsync() (v2)                                                                                                     | setgid()                                                                                                          | uname()                                                                                                           |
| ftruncate() (v3)                                                                                                 | setpgid()                                                                                                         | unlink()                                                                                                          |
| getegid()                                                                                                        | setsid()                                                                                                          | utime()                                                                                                           |
| geteuid()                                                                                                        | setsockopt() (v3)                                                                                                 | wait()                                                                                                            |
| getgid()                                                                                                         | setuid()                                                                                                          | waitpid()                                                                                                         |
| getgroups()                                                                                                      | shutdown() (v3)                                                                                                   | write()                                                                                                           |
| getpeername() (v3)                                                                                               | sigaction()                                                                                                       |                                                                                                                   |
| getpgrp()                                                                                                        | sigaddset()                                                                                                       |                                                                                                                   |

SUSv3 lưu ý rằng tất cả các function không được liệt kê trong Bảng 21-1 được coi là không an toàn đối với signal, nhưng chỉ ra rằng một function chỉ không an toàn khi việc gọi signal handler làm gián đoạn quá trình thực thi của một unsafe function, và bản thân handler cũng gọi một unsafe function. Nói cách khác, khi viết signal handler, chúng ta có hai lựa chọn:

-  Đảm bảo rằng code của bản thân signal handler là reentrant và chỉ gọi các async-signal-safe function.
-  Block việc gửi signal trong khi thực thi code trong chương trình chính gọi các unsafe function hoặc làm việc với global data structure cũng được signal handler cập nhật.

Vấn đề với phương án thứ hai là trong một chương trình phức tạp, có thể khó đảm bảo rằng signal handler sẽ không bao giờ ngắt chương trình chính trong khi nó đang gọi một unsafe function. Vì lý do này, các quy tắc trên thường được đơn giản hóa thành câu phát biểu rằng chúng ta không được gọi các unsafe function từ bên trong signal handler.

> Nếu chúng ta thiết lập cùng một handler function để xử lý một số signal khác nhau, hoặc sử dụng flag `SA_NODEFER` cho `sigaction()`, thì một handler có thể tự ngắt chính nó. Hệ quả là handler có thể là nonreentrant nếu nó cập nhật các biến global (hoặc static), ngay cả khi chúng không được sử dụng bởi chương trình chính.

### **Sử dụng errno bên trong signal handler**

Vì chúng có thể cập nhật `errno`, việc sử dụng các function được liệt kê trong Bảng 21-1 vẫn có thể làm signal handler không reentrant, vì chúng có thể ghi đè giá trị `errno` đã được đặt bởi một function được gọi từ chương trình chính. Cách giải quyết là lưu giá trị của `errno` khi vào signal handler sử dụng bất kỳ function nào trong Bảng 21-1, và khôi phục giá trị `errno` khi thoát khỏi handler, như trong ví dụ sau:

```
void
handler(int sig)
{
 int savedErrno;
 savedErrno = errno;
 /* Now we can execute a function that might modify errno */
 errno = savedErrno;
}
```

## **Sử dụng các unsafe function trong các chương trình ví dụ trong cuốn sách này**

Mặc dù `printf()` không phải là async-signal-safe, chúng ta sử dụng nó trong signal handler trong nhiều chương trình ví dụ trong cuốn sách này. Chúng ta làm vậy vì `printf()` cung cấp một cách dễ dàng và ngắn gọn để minh họa rằng signal handler đã được gọi, và để hiển thị nội dung của các biến liên quan trong handler. Vì các lý do tương tự, đôi khi chúng ta sử dụng một vài unsafe function khác trong signal handler, bao gồm các stdio function khác và `strsignal()`.

Các ứng dụng thực tế nên tránh gọi các hàm không phải async-signal-safe từ signal handler. Để làm rõ điều này, mỗi signal handler trong các chương trình ví dụ sử dụng một trong các function này được đánh dấu bằng một comment chỉ ra rằng việc sử dụng là không an toàn:

```
printf("Some message\n"); /* UNSAFE */
```

## **21.1.3 Biến Global và Kiểu Dữ Liệu sig\_atomic\_t**

Bất chấp các vấn đề về reentrancy, việc chia sẻ biến global giữa chương trình chính và signal handler có thể hữu ích. Điều này có thể an toàn miễn là chương trình chính xử lý đúng khả năng signal handler có thể thay đổi biến global bất cứ lúc nào. Ví dụ, một thiết kế phổ biến là làm cho hành động duy nhất của signal handler là đặt một global flag. Flag này được kiểm tra định kỳ bởi chương trình chính, sau đó thực hiện hành động thích hợp để đáp lại việc gửi signal (và xóa flag). Khi các biến global được truy cập theo cách này từ signal handler, chúng ta nên khai báo chúng bằng thuộc tính `volatile` (xem Mục 6.8) để ngăn compiler thực hiện các tối ưu hóa dẫn đến việc biến được lưu trong register.

Việc đọc và ghi biến global có thể liên quan đến nhiều hơn một lệnh machine-language, và signal handler có thể ngắt chương trình chính ở giữa một chuỗi lệnh như vậy. (Chúng ta nói rằng việc truy cập vào biến là nonatomic.) Vì lý do này, các tiêu chuẩn ngôn ngữ C và SUSv3 chỉ định một kiểu dữ liệu số nguyên, `sig_atomic_t`, mà đọc và ghi được đảm bảo là atomic. Do đó, một biến flag global được chia sẻ giữa chương trình chính và signal handler nên được khai báo như sau:

```
volatile sig_atomic_t flag;
```

Chúng ta sẽ thấy ví dụ về việc sử dụng kiểu dữ liệu `sig_atomic_t` trong [Listing 22-5,](#page-45-0) ở trang [466](#page-45-0).

Lưu ý rằng các toán tử tăng (++) và giảm (--) của C không nằm trong đảm bảo được cung cấp cho `sig_atomic_t`. Trên một số kiến trúc phần cứng, các thao tác này có thể không phải là atomic (tham khảo Mục 30.1 để biết thêm chi tiết). Tất cả những gì chúng ta được đảm bảo là có thể làm an toàn với biến `sig_atomic_t` là đặt nó bên trong signal handler, và kiểm tra nó trong chương trình chính (hoặc ngược lại).

C99 và SUSv3 chỉ định rằng một implementation nên định nghĩa hai hằng số (trong `<stdint.h>`), `SIG_ATOMIC_MIN` và `SIG_ATOMIC_MAX`, xác định phạm vi các giá trị có thể được gán cho các biến kiểu `sig_atomic_t`. Các tiêu chuẩn yêu cầu phạm vi này ít nhất là –127 đến 127 nếu `sig_atomic_t` được biểu diễn là giá trị có dấu, hoặc 0 đến 255 nếu nó được biểu diễn là giá trị không dấu. Trên Linux, hai hằng số này tương đương với giới hạn âm và dương cho số nguyên có dấu 32 bit.

## <span id="page-7-0"></span>**21.2 Các Phương Pháp Khác Để Kết Thúc Signal Handler**

Tất cả các signal handler mà chúng ta đã xem xét cho đến nay đều hoàn thành bằng cách trả về chương trình chính. Tuy nhiên, đơn giản là return từ signal handler đôi khi không mong muốn, hoặc trong một số trường hợp, thậm chí không hữu ích. (Chúng ta sẽ thấy ví dụ về nơi return từ signal handler không hữu ích khi chúng ta thảo luận về hardware-generated signal trong Mục [22.4](#page-31-0).)

Có nhiều cách khác để kết thúc signal handler:

-  Sử dụng `_exit()` để kết thúc process. Trước đó, handler có thể thực hiện một số hành động dọn dẹp. Lưu ý rằng chúng ta không thể sử dụng `exit()` để kết thúc signal handler, vì nó không phải là một trong các safe function được liệt kê trong Bảng 21-1. Nó không an toàn vì nó flush stdio buffer trước khi gọi `_exit()`, như mô tả trong Mục 25.1.
-  Sử dụng `kill()` hoặc `raise()` để gửi signal kết thúc process (tức là signal có hành động mặc định là kết thúc process).
-  Thực hiện nonlocal goto từ signal handler.
-  Sử dụng function `abort()` để kết thúc process với core dump.

<span id="page-8-1"></span>Hai tùy chọn cuối cùng trong số này được mô tả chi tiết hơn trong các phần sau.

## <span id="page-8-0"></span>**21.2.1 Thực Hiện Nonlocal Goto Từ Signal Handler**

Mục 6.8 đã mô tả việc sử dụng `setjmp()` và `longjmp()` để thực hiện nonlocal goto từ một function đến một trong các caller của nó. Chúng ta cũng có thể sử dụng kỹ thuật này từ signal handler. Điều này cung cấp một cách để phục hồi sau khi gửi signal do hardware exception (ví dụ: lỗi truy cập bộ nhớ), và cũng cho phép chúng ta bắt signal và trả quyền điều khiển về một điểm cụ thể trong chương trình. Ví dụ, khi nhận được signal `SIGINT` (thường được tạo bằng cách nhấn Control-C), shell thực hiện nonlocal goto để trả quyền điều khiển về vòng lặp đầu vào chính của nó (và do đó đọc một lệnh mới).

Tuy nhiên, có một vấn đề khi sử dụng function `longjmp()` chuẩn để thoát khỏi signal handler. Chúng ta đã lưu ý trước đó rằng, khi vào signal handler, kernel tự động thêm signal đang gọi, cũng như bất kỳ signal nào được chỉ định trong trường `act.sa_mask`, vào process signal mask, và sau đó xóa các signal này khỏi mask khi handler thực hiện return thông thường.

Điều gì xảy ra với signal mask nếu chúng ta thoát khỏi signal handler bằng cách sử dụng `longjmp()`? Câu trả lời phụ thuộc vào dòng dõi của implementation UNIX cụ thể. Theo System V, `longjmp()` không khôi phục signal mask, vì vậy các signal bị block không được unblock khi rời khỏi handler. Linux theo hành vi System V. (Điều này thường không phải là điều chúng ta muốn, vì nó để lại signal đã kích hoạt handler bị block.) Theo các implementation dựa trên BSD, `setjmp()` lưu signal mask trong đối số `env` của nó, và signal mask đã lưu được khôi phục bởi `longjmp()`. (Các implementation dựa trên BSD cũng cung cấp hai function khác, `_setjmp()` và `_longjmp()`, có ngữ nghĩa System V.) Nói cách khác, chúng ta không thể sử dụng `longjmp()` một cách di động để thoát khỏi signal handler.

> Nếu chúng ta định nghĩa feature test macro `_BSD_SOURCE` khi biên dịch chương trình, thì `setjmp()` (của glibc) theo ngữ nghĩa BSD.

Vì sự khác biệt này trong hai biến thể UNIX chính, POSIX.1-1990 đã chọn không chỉ định việc xử lý signal mask bởi `setjmp()` và `longjmp()`. Thay vào đó, nó định nghĩa một cặp function mới, `sigsetjmp()` và `siglongjmp()`, cung cấp kiểm soát rõ ràng về signal mask khi thực hiện nonlocal goto.

```
#include <setjmp.h>
int sigsetjmp(sigjmp_buf env, int savesigs);
                      Returns 0 on initial call, nonzero on return via siglongjmp()
void siglongjmp(sigjmp_buf env, int val);
```

Các function `sigsetjmp()` và `siglongjmp()` hoạt động tương tự như `setjmp()` và `longjmp()`. Sự khác biệt duy nhất là kiểu của đối số `env` (`sigjmp_buf` thay vì `jmp_buf`) và đối số `savesigs` bổ sung cho `sigsetjmp()`. Nếu `savesigs` khác không, thì process signal mask hiện tại tại thời điểm gọi `sigsetjmp()` được lưu trong `env` và được khôi phục bởi lời gọi `siglongjmp()` sau đó với cùng đối số `env`. Nếu `savesigs` là 0, thì process signal mask không được lưu và khôi phục.

Các function `longjmp()` và `siglongjmp()` không được liệt kê trong số các async-signal-safe function trong Bảng 21-1. Điều này là vì gọi bất kỳ non-async-signal-safe function nào sau khi thực hiện nonlocal goto mang những rủi ro tương tự như gọi function đó từ bên trong signal handler. Hơn nữa, nếu signal handler ngắt chương trình chính khi nó đang trong quá trình cập nhật data structure, và handler thoát bằng cách thực hiện nonlocal goto, thì việc cập nhật chưa hoàn thành có thể để data structure đó ở trạng thái không nhất quán. Một kỹ thuật có thể giúp tránh các vấn đề là sử dụng `sigprocmask()` để tạm thời block signal trong khi các cập nhật nhạy cảm đang được thực hiện.

### **Chương trình ví dụ**

[Listing 21-2](#page-11-0) minh họa sự khác biệt trong việc xử lý signal mask đối với hai loại nonlocal goto. Chương trình này thiết lập handler cho `SIGINT`. Chương trình được thiết kế để cho phép sử dụng `setjmp()` cộng với `longjmp()` hoặc `sigsetjmp()` cộng với `siglongjmp()` để thoát khỏi signal handler, tùy thuộc vào việc chương trình được biên dịch với macro `USE_SIGSETJMP` được định nghĩa hay không. Chương trình hiển thị cài đặt hiện tại của signal mask cả khi vào signal handler và sau khi nonlocal goto đã chuyển quyền điều khiển từ handler trở lại chương trình chính.

Khi chúng ta xây dựng chương trình để sử dụng `longjmp()` để thoát khỏi signal handler, đây là những gì chúng ta thấy khi chạy chương trình:

```
$ make -s sigmask_longjmp Default compilation causes setjmp() to be used
$ ./sigmask_longjmp
Signal mask at startup:
 <empty signal set>
Calling setjmp()
Type Control-C to generate SIGINT
Received signal 2 (Interrupt), signal mask is:
 2 (Interrupt)
After jump from handler, signal mask is:
 2 (Interrupt)
(At this point, typing Control-C again has no effect, since SIGINT is blocked)
Type Control-\ to kill the program
Quit
```

Từ đầu ra của chương trình, chúng ta có thể thấy rằng, sau `longjmp()` từ signal handler, signal mask vẫn được đặt thành giá trị mà nó được đặt khi vào signal handler.

> Trong shell session trên, chúng ta đã xây dựng chương trình bằng makefile được cung cấp cùng với bản phân phối mã nguồn của cuốn sách này. Tùy chọn `-s` yêu cầu make không echo các lệnh đang thực thi. Chúng ta sử dụng tùy chọn này để tránh làm lộn xộn nhật ký session. ([Mecklenburg, 2005] cung cấp mô tả về chương trình GNU make.)

Khi chúng ta biên dịch cùng file nguồn để xây dựng một file thực thi sử dụng `siglongjmp()` để thoát khỏi handler, chúng ta thấy điều sau:

```
$ make -s sigmask_siglongjmp Compiles using cc –DUSE_SIGSETJMP
$ ./sigmask_siglongjmp x
Signal mask at startup:
 <empty signal set>
Calling sigsetjmp()
Type Control-C
Received signal 2 (Interrupt), signal mask is:
 2 (Interrupt)
After jump from handler, signal mask is:
 <empty signal set>
```

Tại thời điểm này, `SIGINT` không bị block, vì `siglongjmp()` đã khôi phục signal mask về trạng thái ban đầu. Tiếp theo, chúng ta nhấn Control-C lần nữa, để handler được gọi lại một lần nữa:

```
Type Control-C
Received signal 2 (Interrupt), signal mask is:
 2 (Interrupt)
After jump from handler, signal mask is:
 <empty signal set>
Type Control-\ to kill the program
Quit
```

Từ đầu ra trên, chúng ta có thể thấy rằng `siglongjmp()` khôi phục signal mask về giá trị mà nó có tại thời điểm gọi `sigsetjmp()` (tức là một signal set rỗng).

[Listing 21-2](#page-11-0) cũng minh họa một kỹ thuật hữu ích để sử dụng với signal handler thực hiện nonlocal goto. Vì signal có thể được tạo ra bất cứ lúc nào, nó thực sự có thể xảy ra trước khi đích của goto đã được thiết lập bởi `sigsetjmp()` (hoặc `setjmp()`). Để ngăn chặn khả năng này (điều này sẽ khiến handler thực hiện nonlocal goto sử dụng buffer `env` chưa được khởi tạo), chúng ta sử dụng một guard variable, `canJump`, để chỉ ra liệu buffer `env` đã được khởi tạo hay chưa. Nếu `canJump` là false, thì thay vì thực hiện nonlocal goto, handler chỉ đơn giản là return. Một cách tiếp cận thay thế là sắp xếp code chương trình sao cho lời gọi `sigsetjmp()` (hoặc `setjmp()`) xảy ra trước khi signal handler được thiết lập. Tuy nhiên, trong các chương trình phức tạp, có thể khó đảm bảo rằng hai bước này được thực hiện theo thứ tự đó, và việc sử dụng guard variable có thể đơn giản hơn.

Lưu ý rằng sử dụng `#ifdef` là cách đơn giản nhất để viết chương trình trong [Listing 21-2](#page-11-0) theo cách tuân theo tiêu chuẩn. Đặc biệt, chúng ta không thể thay thế `#ifdef` bằng kiểm tra runtime sau:

```
if (useSiglongjmp)
 s = sigsetjmp(senv, 1);
else
 s = setjmp(env);
if (s == 0)
 ...
```

Điều này không được phép vì SUSv3 không cho phép `setjmp()` và `sigsetjmp()` được sử dụng trong một câu lệnh gán (xem Mục 6.8).

<span id="page-11-0"></span>**Listing 21-2:** Thực hiện nonlocal goto từ signal handler

```
–––––––––––––––––––––––––––––––––––––––––––––––––– signals/sigmask_longjmp.c
#define _GNU_SOURCE /* Get strsignal() declaration from <string.h> */
#include <string.h>
#include <setjmp.h>
#include <signal.h>
#include "signal_functions.h" /* Declaration of printSigMask() */
#include "tlpi_hdr.h"
static volatile sig_atomic_t canJump = 0;
 /* Set to 1 once "env" buffer has been
 initialized by [sig]setjmp() */
#ifdef USE_SIGSETJMP
static sigjmp_buf senv;
#else
static jmp_buf env;
#endif
static void
handler(int sig)
{
 /* UNSAFE: This handler uses non-async-signal-safe functions
 (printf(), strsignal(), printSigMask(); see Section 21.1.2) */
 printf("Received signal %d (%s), signal mask is:\n", sig,
 strsignal(sig));
 printSigMask(stdout, NULL);
 if (!canJump) {
 printf("'env' buffer not yet set, doing a simple return\n");
 return;
 }
#ifdef USE_SIGSETJMP
 siglongjmp(senv, 1);
#else
 longjmp(env, 1);
#endif
}
```

```
int
main(int argc, char *argv[])
{
 struct sigaction sa;
 printSigMask(stdout, "Signal mask at startup:\n");
 sigemptyset(&sa.sa_mask);
 sa.sa_flags = 0;
 sa.sa_handler = handler;
 if (sigaction(SIGINT, &sa, NULL) == -1)
 errExit("sigaction");
#ifdef USE_SIGSETJMP
 printf("Calling sigsetjmp()\n");
 if (sigsetjmp(senv, 1) == 0)
#else
 printf("Calling setjmp()\n");
 if (setjmp(env) == 0)
#endif
 canJump = 1; /* Executed after [sig]setjmp() */
 else /* Executed after [sig]longjmp() */
 printSigMask(stdout, "After jump from handler, signal mask is:\n" );
 for (;;) /* Wait for signals until killed */
 pause();
}
–––––––––––––––––––––––––––––––––––––––––––––––––– signals/sigmask_longjmp.c
```

## **21.2.2 Kết Thúc Process Bất Thường: abort()**

Function `abort()` kết thúc process đang gọi và khiến nó tạo ra core dump.

```
#include <stdlib.h>
void abort(void);
```

Function `abort()` kết thúc process đang gọi bằng cách tạo ra signal `SIGABRT`. Hành động mặc định của `SIGABRT` là tạo ra file core dump và kết thúc process. File core dump sau đó có thể được sử dụng trong debugger để kiểm tra trạng thái của chương trình tại thời điểm gọi `abort()`.

SUSv3 yêu cầu `abort()` ghi đè hiệu ứng của việc block hoặc ignore `SIGABRT`. Hơn nữa, SUSv3 chỉ định rằng `abort()` phải kết thúc process trừ khi process bắt signal với handler không return. Câu phát biểu cuối cùng này đòi hỏi một chút suy nghĩ. Trong số các phương pháp kết thúc signal handler được mô tả ở Mục [21.2,](#page-7-0) phương pháp liên quan ở đây là sử dụng nonlocal goto để thoát khỏi handler. Nếu điều này được thực hiện, thì hiệu ứng của `abort()` sẽ bị vô hiệu hóa; nếu không, `abort()` luôn kết thúc process. Trong hầu hết các implementation, việc kết thúc được đảm bảo như sau: nếu process vẫn chưa kết thúc sau khi tạo ra `SIGABRT` một lần (tức là một handler bắt signal và return, để quá trình thực thi `abort()` được tiếp tục), `abort()` đặt lại việc xử lý `SIGABRT` thành `SIG_DFL` và tạo ra `SIGABRT` lần thứ hai, điều này được đảm bảo sẽ kill process.

Nếu `abort()` kết thúc process thành công, thì nó cũng flush và đóng các luồng stdio.

Ví dụ về việc sử dụng `abort()` được cung cấp trong các error-handling function của Listing 3-3, ở trang 54.

## **21.3 Xử Lý Signal Trên Alternate Stack: sigaltstack()**

Thông thường, khi signal handler được gọi, kernel tạo một frame cho nó trên process stack. Tuy nhiên, điều này có thể không thể thực hiện được nếu process cố gắng mở rộng stack vượt quá kích thước tối đa có thể. Ví dụ, điều này có thể xảy ra vì stack phát triển quá lớn đến mức gặp một vùng mapped memory (Mục 48.5) hoặc heap đang phát triển hướng lên, hoặc đạt đến giới hạn tài nguyên `RLIMIT_STACK` (Mục 36.3).

Khi process cố gắng phát triển stack của nó vượt quá kích thước tối đa có thể, kernel tạo ra signal `SIGSEGV` cho process. Tuy nhiên, vì space stack đã cạn kiệt, kernel không thể tạo frame cho bất kỳ `SIGSEGV` handler nào mà chương trình có thể đã thiết lập. Do đó, handler không được gọi, và process bị kết thúc (hành động mặc định cho `SIGSEGV`).

Nếu thay vào đó chúng ta cần đảm bảo rằng signal `SIGSEGV` được xử lý trong các trường hợp này, chúng ta có thể làm như sau:

- 1. Cấp phát một vùng bộ nhớ, gọi là alternate signal stack, để sử dụng cho stack frame của signal handler.
- 2. Sử dụng system call `sigaltstack()` để thông báo cho kernel về sự tồn tại của alternate signal stack.
- 3. Khi thiết lập signal handler, chỉ định flag `SA_ONSTACK`, để nói với kernel rằng frame cho handler này nên được tạo trên alternate stack.

System call `sigaltstack()` vừa thiết lập alternate signal stack vừa trả về thông tin về bất kỳ alternate signal stack nào đã được thiết lập.

```
#include <signal.h>
int sigaltstack(const stack_t *sigstack, stack_t *old_sigstack);
                                             Returns 0 on success, or –1 on error
```

Đối số `sigstack` trỏ đến một structure chỉ định vị trí và thuộc tính của alternate signal stack mới. Đối số `old_sigstack` trỏ đến một structure được sử dụng để trả về thông tin về alternate signal stack đã được thiết lập trước đó (nếu có). Một trong hai đối số này có thể được chỉ định là `NULL`. Ví dụ, chúng ta có thể tìm hiểu về alternate signal stack hiện có mà không thay đổi nó bằng cách chỉ định `NULL` cho đối số `sigstack`. Nếu không, mỗi đối số này trỏ đến một structure có kiểu sau:

```
typedef struct {
 void *ss_sp; /* Starting address of alternate stack */
 int ss_flags; /* Flags: SS_ONSTACK, SS_DISABLE */
 size_t ss_size; /* Size of alternate stack */
} stack_t;
```

Các trường `ss_sp` và `ss_size` chỉ định kích thước và vị trí của alternate signal stack. Khi thực sự sử dụng alternate signal stack, kernel tự động căn chỉnh giá trị được cung cấp trong `ss_sp` đến một ranh giới địa chỉ phù hợp với kiến trúc phần cứng.

Thông thường, alternate signal stack được cấp phát tĩnh hoặc được cấp phát động trên heap. SUSv3 chỉ định hằng số `SIGSTKSZ` để sử dụng như một giá trị điển hình khi xác định kích thước alternate stack, và `MINSIGSTKSZ` là kích thước tối thiểu cần thiết để gọi signal handler. Trên Linux/x86-32, các hằng số này được định nghĩa với các giá trị lần lượt là 8192 và 2048.

Kernel không thay đổi kích thước alternate signal stack. Nếu stack tràn không gian chúng ta đã cấp phát cho nó, thì kết quả là hỗn loạn (ví dụ: ghi đè các biến ngoài giới hạn của stack). Điều này thường không phải là vấn đề — vì thường chúng ta sử dụng alternate signal stack để xử lý trường hợp đặc biệt là standard stack bị tràn, thường chỉ có một hoặc vài frame được cấp phát trên stack. Công việc của `SIGSEGV` handler là thực hiện một số dọn dẹp và kết thúc process hoặc unwind standard stack bằng cách sử dụng nonlocal goto.

Trường `ss_flags` chứa một trong các giá trị sau:

#### SS\_ONSTACK

Nếu flag này được đặt khi truy xuất thông tin về alternate signal stack hiện đang được thiết lập (`old_sigstack`), nó chỉ ra rằng process đang thực thi trên alternate signal stack. Các nỗ lực thiết lập alternate signal stack mới trong khi process đã đang chạy trên alternate signal stack sẽ dẫn đến lỗi (`EPERM`) từ `sigaltstack()`.

#### SS\_DISABLE

Được trả về trong `old_sigstack`, flag này chỉ ra rằng không có alternate signal stack nào hiện được thiết lập. Khi được chỉ định trong `sigstack`, nó vô hiệu hóa alternate signal stack đã được thiết lập.

[Listing 21-3](#page-15-0) minh họa việc thiết lập và sử dụng alternate signal stack. Sau khi thiết lập alternate signal stack và handler cho `SIGSEGV`, chương trình này gọi một function đệ quy vô hạn, do đó stack bị tràn và process nhận được signal `SIGSEGV`. Khi chạy chương trình, đây là những gì chúng ta thấy:

```
$ ulimit -s unlimited
$ ./t_sigaltstack
Top of standard stack is near 0xbffff6b8
Alternate stack is at 0x804a948-0x804cfff
Call 1 - top of stack near 0xbff0b3ac
Call 2 - top of stack near 0xbfe1714c
Many intervening lines of output removed
Call 2144 - top of stack near 0x4034120c
```

```
Call 2145 - top of stack near 0x4024cfac
Caught signal 11 (Segmentation fault)
Top of handler stack near 0x804c860
```

Trong shell session này, lệnh `ulimit` được sử dụng để xóa bất kỳ giới hạn tài nguyên `RLIMIT_STACK` nào có thể đã được đặt trong shell. Chúng ta giải thích giới hạn tài nguyên này trong Mục 36.3.

<span id="page-15-0"></span>**Listing 21-3:** Sử dụng `sigaltstack()`

```
––––––––––––––––––––––––––––––––––––––––––––––––––––signals/t_sigaltstack.c
#define _GNU_SOURCE /* Get strsignal() declaration from <string.h> */
#include <string.h>
#include <signal.h>
#include "tlpi_hdr.h"
static void
sigsegvHandler(int sig)
{
 int x;
 /* UNSAFE: This handler uses non-async-signal-safe functions
 (printf(), strsignal(), fflush(); see Section 21.1.2) */
 printf("Caught signal %d (%s)\n", sig, strsignal(sig));
 printf("Top of handler stack near %10p\n", (void *) &x);
 fflush(NULL);
 _exit(EXIT_FAILURE); /* Can't return after SIGSEGV */
}
static void /* A recursive function that overflows the stack */
overflowStack(int callNum)
{
 char a[100000]; /* Make this stack frame large */
 printf("Call %4d - top of stack near %10p\n", callNum, &a[0]);
 overflowStack(callNum+1);
}
int
main(int argc, char *argv[])
{
 stack_t sigstack;
 struct sigaction sa;
 int j;
 printf("Top of standard stack is near %10p\n", (void *) &j);
 /* Allocate alternate stack and inform kernel of its existence */
 sigstack.ss_sp = malloc(SIGSTKSZ);
 if (sigstack.ss_sp == NULL)
 errExit("malloc");
```

```
 sigstack.ss_size = SIGSTKSZ;
 sigstack.ss_flags = 0;
 if (sigaltstack(&sigstack, NULL) == -1)
 errExit("sigaltstack");
 printf("Alternate stack is at %10p-%p\n",
 sigstack.ss_sp, (char *) sbrk(0) - 1);
 sa.sa_handler = sigsegvHandler; /* Establish handler for SIGSEGV */
 sigemptyset(&sa.sa_mask);
 sa.sa_flags = SA_ONSTACK; /* Handler uses alternate stack */
 if (sigaction(SIGSEGV, &sa, NULL) == -1)
 errExit("sigaction");
 overflowStack(1);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––signals/t_sigaltstack.c
```

# **21.4 Flag SA\_SIGINFO**

<span id="page-16-0"></span>Đặt flag `SA_SIGINFO` khi thiết lập handler với `sigaction()` cho phép handler lấy thêm thông tin về signal khi nó được gửi. Để lấy thông tin này, chúng ta phải khai báo handler như sau:

```
void handler(int sig, siginfo_t *siginfo, void *ucontext);
```

Đối số đầu tiên, `sig`, là số hiệu signal, như với standard signal handler. Đối số thứ hai, `siginfo`, là một structure được sử dụng để cung cấp thông tin bổ sung về signal. Chúng ta mô tả structure này dưới đây. Đối số cuối cùng, `ucontext`, cũng được mô tả dưới đây.

Vì signal handler trên có prototype khác với standard signal handler, các quy tắc kiểu dữ liệu C có nghĩa là chúng ta không thể sử dụng trường `sa_handler` của structure `sigaction` để chỉ định địa chỉ của handler. Thay vào đó, chúng ta phải sử dụng một trường thay thế: `sa_sigaction`. Nói cách khác, định nghĩa của structure `sigaction` phức tạp hơn một chút so với những gì được hiển thị trong Mục 20.13. Toàn bộ structure được định nghĩa như sau:

```
struct sigaction {
 union {
 void (*sa_handler)(int);
 void (*sa_sigaction)(int, siginfo_t *, void *);
 } __sigaction_handler;
 sigset_t sa_mask;
 int sa_flags;
 void (*sa_restorer)(void);
};
/* Following defines make the union fields look like simple fields
 in the parent structure */
#define sa_handler __sigaction_handler.sa_handler
#define sa_sigaction __sigaction_handler.sa_sigaction
```

Structure `sigaction` sử dụng union để kết hợp các trường `sa_sigaction` và `sa_handler`. (Hầu hết các implementation UNIX khác cũng sử dụng union cho mục đích này.) Sử dụng union là có thể vì chỉ một trong các trường này được yêu cầu trong một lần gọi `sigaction()` cụ thể. (Tuy nhiên, điều này có thể dẫn đến các lỗi kỳ lạ nếu chúng ta ngây thơ mong đợi có thể đặt các trường `sa_handler` và `sa_sigaction` độc lập với nhau, có thể vì chúng ta tái sử dụng một structure `sigaction` duy nhất trong nhiều lời gọi `sigaction()` để thiết lập handler cho các signal khác nhau.)

Đây là ví dụ về việc sử dụng `SA_SIGINFO` để thiết lập signal handler:

```
struct sigaction act;
sigemptyset(&act.sa_mask);
act.sa_sigaction = handler;
act.sa_flags = SA_SIGINFO;
if (sigaction(SIGINT, &act, NULL) == -1)
 errExit("sigaction");
```

Để xem ví dụ đầy đủ về việc sử dụng flag `SA_SIGINFO`, xem [Listing 22-3](#page-41-0) (trang [462](#page-41-0)) và Listing 23-5 (trang 500).

#### **Structure siginfo\_t**

Structure `siginfo_t` được truyền như đối số thứ hai cho signal handler được thiết lập với `SA_SIGINFO` có dạng sau:

```
typedef struct {
 int si_signo; /* Signal number */
 int si_code; /* Signal code */
 int si_trapno; /* Trap number for hardware-generated signal
 (unused on most architectures) */
 union sigval si_value; /* Accompanying data from sigqueue() */
 pid_t si_pid; /* Process ID of sending process */
 uid_t si_uid; /* Real user ID of sender */
 int si_errno; /* Error number (generally unused) */
 void *si_addr; /* Address that generated signal
 (hardware-generated signals only) */
 int si_overrun; /* Overrun count (Linux 2.6, POSIX timers) */
 int si_timerid; /* (Kernel-internal) Timer ID
 (Linux 2.6, POSIX timers) */
 long si_band; /* Band event (SIGPOLL/SIGIO) */
 int si_fd; /* File descriptor (SIGPOLL/SIGIO) */
 int si_status; /* Exit status or signal (SIGCHLD) */
 clock_t si_utime; /* User CPU time (SIGCHLD) */
 clock_t si_stime; /* System CPU time (SIGCHLD) */
} siginfo_t;
```

Feature test macro `_POSIX_C_SOURCE` phải được định nghĩa với giá trị lớn hơn hoặc bằng 199309 để hiển thị khai báo của structure `siginfo_t` từ `<signal.h>`.

Trên Linux, cũng như trên hầu hết các implementation UNIX, nhiều trường trong structure `siginfo_t` được kết hợp thành union, vì không phải tất cả các trường đều cần thiết cho mỗi signal. (Xem `<bits/siginfo.h>` để biết chi tiết.)

Khi vào signal handler, các trường của structure `siginfo_t` được đặt như sau:

si\_signo

Trường này được đặt cho tất cả signal. Nó chứa số hiệu của signal gây ra việc gọi handler — tức là cùng giá trị với đối số `sig` của handler.

si\_code

Trường này được đặt cho tất cả signal. Nó chứa một mã cung cấp thông tin bổ sung về nguồn gốc của signal, như được hiển thị trong Bảng 21-1.

si\_value

Trường này chứa dữ liệu đi kèm cho signal được gửi qua `sigqueue()`. Chúng ta mô tả `sigqueue()` trong Mục [22.8.1](#page-37-0).

si\_pid

Đối với signal được gửi qua `kill()` hoặc `sigqueue()`, trường này được đặt thành process ID của process gửi.

si\_uid

Đối với signal được gửi qua `kill()` hoặc `sigqueue()`, trường này được đặt thành real user ID của process gửi. Hệ thống cung cấp real user ID của process gửi vì điều đó có nhiều thông tin hơn là cung cấp effective user ID. Hãy xem xét các quy tắc quyền để gửi signal được mô tả trong Mục 20.5: nếu effective user ID cấp cho người gửi quyền gửi signal, thì user ID đó phải là 0 (tức là process có đặc quyền), hoặc giống với real user ID hoặc saved set-user-ID của process nhận. Trong trường hợp này, có thể hữu ích cho bên nhận khi biết real user ID của bên gửi, có thể khác với effective user ID (ví dụ: nếu bên gửi là chương trình set-user-ID).

si\_errno

Nếu trường này được đặt thành giá trị khác không, thì nó chứa một số lỗi (như `errno`) xác định nguyên nhân của signal. Trường này thường không được sử dụng trên Linux.

si\_addr

Trường này chỉ được đặt cho các signal `SIGBUS`, `SIGSEGV`, `SIGILL` và `SIGFPE` do phần cứng tạo ra. Đối với các signal `SIGBUS` và `SIGSEGV`, trường này chứa địa chỉ gây ra tham chiếu bộ nhớ không hợp lệ. Đối với các signal `SIGILL` và `SIGFPE`, trường này chứa địa chỉ của lệnh chương trình đã tạo ra signal.

Các trường sau, là phần mở rộng không chuẩn của Linux, chỉ được đặt khi gửi signal được tạo ra khi hết hạn POSIX timer (xem Mục 23.6):

si\_timerid

Trường này chứa ID mà kernel sử dụng nội bộ để xác định timer.

si\_overrun

Trường này được đặt thành số overrun count của timer.

Hai trường sau chỉ được đặt khi gửi signal `SIGIO` (Mục 63.3):

si\_band

Trường này chứa giá trị "band event" liên quan đến sự kiện I/O. (Trong các phiên bản glibc trước 2.3.2, `si_band` có kiểu `int`.)

si\_fd

Trường này chứa số của file descriptor liên quan đến sự kiện I/O. Trường này không được chỉ định trong SUSv3, nhưng có mặt trên nhiều implementation khác.

Các trường sau chỉ được đặt khi gửi signal `SIGCHLD` (Mục 26.3):

si\_status

Trường này chứa exit status của child (nếu `si_code` là `CLD_EXITED`) hoặc số hiệu của signal được gửi đến child (tức là số hiệu của signal đã kết thúc hoặc dừng child, như mô tả trong Mục 26.1.3).

si\_utime

Trường này chứa user CPU time được sử dụng bởi child process. Trong các kernel trước 2.6 và từ 2.6.27 trở đi, điều này được đo bằng system clock tick (chia cho `sysconf(_SC_CLK_TCK)`). Trong các kernel 2.6 trước 2.6.27, một lỗi có nghĩa là trường này báo cáo thời gian được đo bằng jiffy có thể cấu hình bởi người dùng (xem Mục 10.6). Trường này không được chỉ định trong SUSv3, nhưng có mặt trên nhiều implementation khác.

si\_stime

Trường này chứa system CPU time được sử dụng bởi child process. Xem mô tả trường `si_utime`. Trường này không được chỉ định trong SUSv3, nhưng có mặt trên nhiều implementation khác.

Trường `si_code` cung cấp thêm thông tin về nguồn gốc của signal, sử dụng các giá trị được hiển thị trong Bảng 21-2. Không phải tất cả các giá trị dành riêng cho signal được hiển thị trong cột thứ hai của bảng này đều xuất hiện trên tất cả các implementation UNIX và kiến trúc phần cứng (đặc biệt là trong trường hợp bốn hardware-generated signal `SIGBUS`, `SIGSEGV`, `SIGILL` và `SIGFPE`), mặc dù tất cả các hằng số này được định nghĩa trên Linux và hầu hết đều xuất hiện trong SUSv3.

Lưu ý các điểm bổ sung sau về các giá trị được hiển thị trong Bảng 21-2:

-  Các giá trị `SI_KERNEL` và `SI_SIGIO` là đặc thù của Linux. Chúng không được chỉ định trong SUSv3 và không xuất hiện trên các implementation UNIX khác.
-  `SI_SIGIO` chỉ được sử dụng trong Linux 2.2. Từ kernel 2.4 trở đi, Linux sử dụng các hằng số `POLL_*` được hiển thị trong bảng thay thế.

SUSv4 chỉ định function `psiginfo()`, mục đích tương tự như `psignal()` (Mục 20.8). Function `psiginfo()` nhận hai đối số: một pointer đến structure `siginfo_t` và một message string. Nó in message string ra stderr, theo sau là thông tin về signal được mô tả trong structure `siginfo_t`. Function `psiginfo()` được cung cấp bởi glibc từ phiên bản 2.10. Implementation của glibc in mô tả signal, nguồn gốc của signal (được chỉ ra bởi trường `si_code`), và đối với một số signal, các trường khác từ structure `siginfo_t`. Function `psiginfo()` là mới trong SUSv4 và không có sẵn trên tất cả các hệ thống.

<span id="page-20-0"></span>**Bảng 21-2:** Các giá trị được trả về trong trường `si_code` của structure `siginfo_t`

| Signal   | Giá trị si_code | Nguồn gốc của signal                                        |
|----------|---------------|-------------------------------------------------------------|
| Bất kỳ   | SI_ASYNCIO    | Hoàn thành một thao tác asynchronous I/O (AIO)              |
|          | SI_KERNEL     | Được gửi bởi kernel (ví dụ: signal từ terminal driver)      |
|          | SI_MESGQ      | Tin nhắn đến trên POSIX message queue (từ Linux 2.6.6)      |
|          | SI_QUEUE      | Realtime signal từ user process qua `sigqueue()`            |
|          | SI_SIGIO      | Signal SIGIO (chỉ Linux 2.2)                                |
|          | SI_TIMER      | Hết hạn POSIX (realtime) timer                              |
|          | SI_TKILL      | User process qua `tkill()` hoặc `tgkill()` (từ Linux 2.4.19) |
|          | SI_USER       | User process qua `kill()` hoặc `raise()`                   |
| SIGBUS   | BUS_ADRALN    | Căn chỉnh địa chỉ không hợp lệ                              |
|          | BUS_ADRERR    | Địa chỉ vật lý không tồn tại                                |
|          | BUS_MCEERR_AO | Lỗi bộ nhớ phần cứng; hành động tùy chọn (từ Linux 2.6.32) |
|          | BUS_MCEERR_AR | Lỗi bộ nhớ phần cứng; hành động bắt buộc (từ Linux 2.6.32) |
|          | BUS_OBJERR    | Lỗi phần cứng đặc thù cho đối tượng                        |
| SIGCHLD  | CLD_CONTINUED | Child được tiếp tục bởi SIGCONT (từ Linux 2.6.9)            |
|          | CLD_DUMPED    | Child kết thúc bất thường, có core dump                     |
|          | CLD_EXITED    | Child đã thoát                                              |
|          | CLD_KILLED    | Child kết thúc bất thường, không có core dump               |
|          | CLD_STOPPED   | Child bị dừng                                               |
|          | CLD_TRAPPED   | Traced child đã dừng                                        |
| SIGFPE   | FPE_FLTDIV    | Chia số thực dấu phẩy động cho không                        |
|          | FPE_FLTINV    | Thao tác dấu phẩy động không hợp lệ                         |
|          | FPE_FLTOVF    | Tràn dấu phẩy động                                          |
|          | FPE_FLTRES    | Kết quả dấu phẩy động không chính xác                       |
|          | FPE_FLTUND    | Underflow dấu phẩy động                                     |
|          | FPE_INTDIV    | Chia số nguyên cho không                                     |
|          | FPE_INTOVF    | Tràn số nguyên                                              |
|          | FPE_SUB       | Chỉ số ngoài phạm vi                                        |
| SIGILL   | ILL_BADSTK    | Lỗi stack nội bộ                                            |
|          | ILL_COPROC    | Lỗi coprocessor                                             |
|          | ILL_ILLADR    | Chế độ địa chỉ không hợp lệ                                 |
|          | ILL_ILLOPC    | Opcode không hợp lệ                                         |
|          | ILL_ILLOPN    | Toán hạng không hợp lệ                                      |
|          | ILL_ILLTRP    | Bẫy không hợp lệ                                            |
|          | ILL_PRVOPC    | Opcode có đặc quyền                                         |
|          | ILL_PRVREG    | Register có đặc quyền                                       |
| SIGPOLL/ | POLL_ERR      | Lỗi I/O                                                     |
| SIGIO    | POLL_HUP      | Thiết bị ngắt kết nối                                       |
|          | POLL_IN       | Dữ liệu đầu vào sẵn có                                      |
|          | POLL_MSG      | Tin nhắn đầu vào sẵn có                                     |
|          | POLL_OUT      | Buffer đầu ra sẵn có                                        |
|          | POLL_PRI      | Đầu vào độ ưu tiên cao sẵn có                               |
| SIGSEGV  | SEGV_ACCERR   | Quyền không hợp lệ cho mapped object                        |
|          |               |                                                             |

*(còn tiếp)*

**Bảng 21-2:** Các giá trị được trả về trong trường `si_code` của structure `siginfo_t` (tiếp theo)

| Signal  | Giá trị si_code | Nguồn gốc của signal           |
|---------|---------------|--------------------------------|
| SIGTRAP | TRAP_BRANCH   | Bẫy nhánh của process          |
|         | TRAP_BRKPT    | Điểm dừng của process          |
|         | TRAP_HWBKPT   | Điểm dừng/watchpoint phần cứng |
|         | TRAP_TRACE    | Bẫy trace của process          |

#### **Đối số ucontext**

Đối số cuối cùng được truyền cho handler được thiết lập với flag `SA_SIGINFO`, `ucontext`, là pointer đến structure kiểu `ucontext_t` (được định nghĩa trong `<ucontext.h>`). (SUSv3 sử dụng void pointer cho đối số này vì nó không chỉ định bất kỳ chi tiết nào của đối số.) Structure này cung cấp thông tin user-context mô tả trạng thái của process trước khi gọi signal handler, bao gồm process signal mask trước đó và các giá trị register đã lưu (ví dụ: program counter và stack pointer). Thông tin này hiếm khi được sử dụng trong signal handler, vì vậy chúng ta không đi vào chi tiết hơn.

> Một cách sử dụng khác của structure `ucontext_t` là với các function `getcontext()`, `makecontext()`, `setcontext()`, và `swapcontext()`, cho phép process truy xuất, tạo, thay đổi và chuyển đổi execution context, tương ứng. (Các thao tác này có phần giống như `setjmp()` và `longjmp()`, nhưng tổng quát hơn.) Các function này có thể được sử dụng để triển khai coroutine, nơi thread thực thi của process luân phiên giữa hai (hoặc nhiều) function. SUSv3 chỉ định các function này, nhưng đánh dấu chúng là lỗi thời. SUSv4 xóa các đặc tả và đề nghị rằng các ứng dụng nên được viết lại để sử dụng POSIX thread thay thế. Tài liệu hướng dẫn glibc cung cấp thêm thông tin về các function này.

# **21.5 Gián Đoạn và Khởi Động Lại System Call**

<span id="page-21-0"></span>Hãy xem xét kịch bản sau:

- 1. Chúng ta thiết lập handler cho một signal nào đó.
- 2. Chúng ta thực hiện một blocking system call, ví dụ `read()` từ terminal, blocking cho đến khi đầu vào được cung cấp.
- 3. Trong khi system call đang bị block, signal mà chúng ta đã thiết lập handler được gửi và signal handler của nó được gọi.

Điều gì xảy ra sau khi signal handler return? Theo mặc định, system call thất bại với lỗi `EINTR` ("Interrupted function"). Đây có thể là tính năng hữu ích. Trong Mục 23.3, chúng ta sẽ thấy cách sử dụng timer (dẫn đến việc gửi signal `SIGALRM`) để đặt timeout cho blocking system call chẳng hạn như `read()`.

Tuy nhiên, thường chúng ta muốn tiếp tục thực thi một system call bị gián đoạn. Để làm điều này, chúng ta có thể sử dụng code như sau để khởi động lại thủ công một system call trong trường hợp nó bị ngắt bởi signal handler:

```
while ((cnt = read(fd, buf, BUF_SIZE)) == -1 && errno == EINTR)
 continue; /* Do nothing loop body */
if (cnt == -1) /* read() failed with other than EINTR */
 errExit("read");
```

Nếu chúng ta thường xuyên viết code như trên, có thể hữu ích khi định nghĩa một macro như sau:

```
#define NO_EINTR(stmt) while ((stmt) == -1 && errno == EINTR);
```

Sử dụng macro này, chúng ta có thể viết lại lời gọi `read()` trước đó như sau:

```
NO_EINTR(cnt = read(fd, buf, BUF_SIZE));
if (cnt == -1) /* read() failed with other than EINTR */
 errExit("read");
```

Thư viện GNU C cung cấp một macro (không chuẩn) có cùng mục đích như macro `NO_EINTR()` của chúng ta trong `<unistd.h>`. Macro này được gọi là `TEMP_FAILURE_RETRY()` và được cung cấp nếu feature test macro `_GNU_SOURCE` được định nghĩa.

Ngay cả khi chúng ta sử dụng macro như `NO_EINTR()`, việc các signal handler gián đoạn system call có thể bất tiện, vì chúng ta phải thêm code vào mỗi blocking system call (giả sử rằng chúng ta muốn khởi động lại lời gọi trong mỗi trường hợp). Thay vào đó, chúng ta có thể chỉ định flag `SA_RESTART` khi thiết lập signal handler với `sigaction()`, để system call được kernel tự động khởi động lại thay mặt cho process. Điều này có nghĩa là chúng ta không cần xử lý lỗi `EINTR` có thể trả về cho các system call này.

Flag `SA_RESTART` là cài đặt theo từng signal. Nói cách khác, chúng ta có thể cho phép handler của một số signal ngắt blocking system call, trong khi những signal khác cho phép tự động khởi động lại system call.

## **System call (và library function) mà SA\_RESTART có hiệu lực**

Thật không may, không phải tất cả các blocking system call đều tự động khởi động lại do chỉ định `SA_RESTART`. Lý do cho điều này một phần mang tính lịch sử:

-  Khởi động lại system call được giới thiệu trong 4.2BSD, và bao gồm các lời gọi bị ngắt tới `wait()` và `waitpid()`, cũng như các I/O system call sau: `read()`, `readv()`, `write()`, `writev()`, và các thao tác `ioctl()` blocking. Các I/O system call có thể bị ngắt, và do đó tự động được khởi động lại bởi `SA_RESTART` chỉ khi hoạt động trên thiết bị "chậm". Các thiết bị chậm bao gồm terminal, pipe, FIFO và socket. Trên các loại file này, các thao tác I/O khác nhau có thể block. (Ngược lại, file đĩa không thuộc loại thiết bị chậm, vì các thao tác I/O đĩa thường có thể được đáp ứng ngay lập tức thông qua buffer cache. Nếu cần I/O đĩa, kernel đưa process vào trạng thái ngủ cho đến khi I/O hoàn thành.)
-  Một số blocking system call khác được lấy từ System V, vốn ban đầu không cung cấp khởi động lại system call.

Trên Linux, các blocking system call (và library function được xây dựng trên system call) sau đây được tự động khởi động lại nếu bị ngắt bởi signal handler được thiết lập bằng flag `SA_RESTART`:

-  Các system call được sử dụng để chờ child process (Mục 26.1): `wait()`, `waitpid()`, `wait3()`, `wait4()`, và `waitid()`.
-  Các I/O system call `read()`, `readv()`, `write()`, `writev()`, và `ioctl()` khi áp dụng cho các thiết bị "chậm". Trong các trường hợp dữ liệu đã được chuyển một phần tại thời điểm gửi signal, các I/O system call sẽ bị ngắt, nhưng trả về trạng thái thành công: một số nguyên chỉ ra số byte đã được chuyển thành công.

-  System call `open()`, trong các trường hợp nó có thể block (ví dụ: khi mở FIFO, như mô tả trong Mục 44.7).
-  Nhiều system call được sử dụng với socket: `accept()`, `accept4()`, `connect()`, `send()`, `sendmsg()`, `sendto()`, `recv()`, `recvfrom()`, và `recvmsg()`. (Trên Linux, các system call này không tự động khởi động lại nếu timeout đã được đặt trên socket bằng cách sử dụng `setsockopt()`. Xem trang manual `signal(7)` để biết chi tiết.)
-  Các system call được sử dụng cho I/O trên POSIX message queue: `mq_receive()`, `mq_timedreceive()`, `mq_send()`, và `mq_timedsend()`.
-  Các system call và library function được sử dụng để đặt file lock: `flock()`, `fcntl()`, và `lockf()`.
-  Thao tác `FUTEX_WAIT` của Linux-specific system call `futex()`.
-  Các function `sem_wait()` và `sem_timedwait()` được sử dụng để giảm POSIX semaphore. (Trên một số implementation UNIX, `sem_wait()` được khởi động lại nếu flag `SA_RESTART` được chỉ định.)
-  Các function được sử dụng để đồng bộ hóa POSIX thread: `pthread_mutex_lock()`, `pthread_mutex_trylock()`, `pthread_mutex_timedlock()`, `pthread_cond_wait()`, và `pthread_cond_timedwait()`.

Trong các kernel trước 2.6.22, `futex()`, `sem_wait()`, và `sem_timedwait()` luôn thất bại với lỗi `EINTR` khi bị ngắt, bất kể cài đặt của flag `SA_RESTART`.

Các blocking system call (và library function) sau đây không bao giờ được tự động khởi động lại (ngay cả khi `SA_RESTART` được chỉ định):

-  Các lời gọi I/O multiplexing `poll()`, `ppoll()`, `select()`, và `pselect()`. (SUSv3 nêu rõ rằng hành vi của `select()` và `pselect()` khi bị ngắt bởi signal handler là không được chỉ định, bất kể cài đặt của `SA_RESTART`.)
-  Các Linux-specific system call `epoll_wait()` và `epoll_pwait()`.
-  Linux-specific system call `io_getevents()`.
-  Các blocking system call được sử dụng với System V message queue và semaphore: `semop()`, `semtimedop()`, `msgrcv()`, và `msgsnd()`. (Mặc dù System V ban đầu không cung cấp tự động khởi động lại system call, trên một số implementation UNIX, các system call này được khởi động lại nếu flag `SA_RESTART` được chỉ định.)
-  `read()` từ inotify file descriptor.
-  Các system call và library function được thiết kế để tạm dừng thực thi chương trình trong một khoảng thời gian nhất định: `sleep()`, `nanosleep()`, và `clock_nanosleep()`.
-  Các system call được thiết kế đặc biệt để chờ cho đến khi signal được gửi: `pause()`, `sigsuspend()`, `sigtimedwait()`, và `sigwaitinfo()`.

## **Thay đổi flag SA\_RESTART cho signal**

Function `siginterrupt()` thay đổi cài đặt `SA_RESTART` liên quan đến một signal.

```
#include <signal.h>
int siginterrupt(int sig, int flag);
                                             Returns 0 on success, or –1 on error
```

Nếu `flag` là true (1), thì handler cho signal `sig` sẽ ngắt các blocking system call. Nếu `flag` là false (0), thì blocking system call sẽ được khởi động lại sau khi thực thi handler cho `sig`.

Function `siginterrupt()` hoạt động bằng cách sử dụng `sigaction()` để lấy bản sao về disposition hiện tại của signal, chỉnh sửa flag `SA_RESTART` trong structure `oldact` được trả về, và sau đó gọi `sigaction()` một lần nữa để cập nhật disposition của signal.

SUSv4 đánh dấu `siginterrupt()` là lỗi thời, khuyến nghị sử dụng `sigaction()` thay thế cho mục đích này.

### **Stop signal chưa được xử lý có thể tạo ra EINTR cho một số Linux system call**

Trên Linux, một số blocking system call có thể trả về `EINTR` ngay cả khi không có signal handler. Điều này có thể xảy ra nếu system call bị block và process bị dừng bởi signal (`SIGSTOP`, `SIGTSTP`, `SIGTTIN`, hoặc `SIGTTOU`), và sau đó được tiếp tục bởi việc gửi signal `SIGCONT`.

Các system call và function sau đây thể hiện hành vi này: `epoll_pwait()`, `epoll_wait()`, `read()` từ inotify file descriptor, `semop()`, `semtimedop()`, `sigtimedwait()`, và `sigwaitinfo()`.

Trong các kernel trước 2.6.24, `poll()` cũng thể hiện hành vi này, cũng như `sem_wait()`, `sem_timedwait()`, `futex(FUTEX_WAIT)`, trong các kernel trước 2.6.22, `msgrcv()` và `msgsnd()` trong các kernel trước 2.6.9, và `nanosleep()` trong Linux 2.4 và trước đó.

Trong Linux 2.4 và trước đó, `sleep()` cũng có thể bị ngắt theo cách này, nhưng thay vì trả về lỗi, nó trả về số giây chưa ngủ còn lại.

Hệ quả của hành vi này là nếu có khả năng chương trình của chúng ta có thể bị dừng và khởi động lại bởi signal, thì chúng ta có thể cần thêm code để khởi động lại các system call này, ngay cả trong chương trình không cài đặt handler cho các stop signal.

## **21.6 Tóm Tắt**

Trong chương này, chúng ta đã xem xét một loạt các yếu tố ảnh hưởng đến hoạt động và thiết kế của signal handler.

Vì signal không được xếp hàng, signal handler đôi khi phải được lập trình để xử lý khả năng nhiều sự kiện của một loại cụ thể đã xảy ra, ngay cả khi chỉ một signal được gửi. Vấn đề reentrancy ảnh hưởng đến cách chúng ta có thể cập nhật biến global và giới hạn tập hợp các function mà chúng ta có thể gọi an toàn từ signal handler.

Thay vì return, signal handler có thể kết thúc theo nhiều cách khác, bao gồm gọi `_exit()`, kết thúc process bằng cách gửi signal (`kill()`, `raise()`, hoặc `abort()`), hoặc thực hiện nonlocal goto. Sử dụng `sigsetjmp()` và `siglongjmp()` cung cấp cho chương trình khả năng kiểm soát rõ ràng về cách xử lý process signal mask khi thực hiện nonlocal goto.

Chúng ta có thể sử dụng `sigaltstack()` để định nghĩa alternate signal stack cho process. Đây là vùng bộ nhớ được sử dụng thay vì standard process stack khi gọi signal handler. Alternate signal stack hữu ích trong các trường hợp standard stack đã cạn kiệt do phát triển quá lớn (lúc đó kernel gửi signal `SIGSEGV` cho process).

Flag `SA_SIGINFO` của `sigaction()` cho phép chúng ta thiết lập signal handler nhận thêm thông tin về signal. Thông tin này được cung cấp thông qua structure `siginfo_t` mà địa chỉ của nó được truyền như đối số cho signal handler.

Khi signal handler ngắt một blocked system call, system call thất bại với lỗi `EINTR`. Chúng ta có thể tận dụng hành vi này để, ví dụ, đặt timer trên blocking system call. Các system call bị ngắt có thể được khởi động lại thủ công nếu muốn. Thay vào đó, thiết lập signal handler với flag `SA_RESTART` của `sigaction()` khiến nhiều (nhưng không phải tất cả) system call được tự động khởi động lại.

#### **Tài liệu tham khảo thêm**

Xem các nguồn được liệt kê trong Mục 20.15.

## **21.7 Bài Tập**

**21-1.** Hãy tự cài đặt `abort()`.
