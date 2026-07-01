## Chương 32
# HỦY BỎ THREAD

Thông thường, nhiều thread thực thi song song, mỗi thread thực hiện nhiệm vụ của mình cho đến khi nó quyết định kết thúc bằng cách gọi `pthread_exit()` hoặc trả về từ hàm khởi động của thread.

Đôi khi, việc hủy bỏ một thread là hữu ích; tức là gửi cho nó một yêu cầu đề nghị nó kết thúc ngay lập tức. Điều này có thể hữu ích, ví dụ khi một nhóm thread đang thực hiện phép tính, và một thread phát hiện ra điều kiện lỗi yêu cầu các thread khác phải kết thúc. Ngoài ra, một ứng dụng có giao diện đồ họa (GUI) có thể cung cấp nút hủy để cho phép người dùng dừng một tác vụ đang được thực hiện bởi một thread nền; trong trường hợp này, thread chính (điều khiển GUI) cần thông báo cho thread nền kết thúc.

Trong chương này, chúng ta mô tả cơ chế hủy bỏ POSIX threads.

## **32.1 Hủy bỏ một Thread**

Hàm `pthread_cancel()` gửi yêu cầu hủy bỏ đến thread được chỉ định.

```
#include <pthread.h>

int pthread_cancel(pthread_t thread);

Trả về 0 nếu thành công, hoặc số lỗi dương nếu có lỗi
```

Sau khi gửi yêu cầu hủy bỏ, `pthread_cancel()` trả về ngay lập tức; tức là, nó không chờ thread đích kết thúc.

Chính xác điều gì xảy ra với thread đích, và khi nào nó xảy ra, phụ thuộc vào trạng thái và kiểu hủy bỏ của thread đó, như được mô tả trong phần tiếp theo.

## **32.2 Trạng Thái và Kiểu Hủy Bỏ**

Các hàm `pthread_setcancelstate()` và `pthread_setcanceltype()` thiết lập các cờ cho phép một thread kiểm soát cách nó phản hồi với yêu cầu hủy bỏ.

```
#include <pthread.h>
int pthread_setcancelstate(int state, int *oldstate);
int pthread_setcanceltype(int type, int *oldtype);
                   Cả hai trả về 0 nếu thành công, hoặc số lỗi dương nếu có lỗi
```

Hàm `pthread_setcancelstate()` thiết lập trạng thái khả năng hủy bỏ của thread đang gọi thành giá trị được cho trong `state`. Đối số này có một trong các giá trị sau:

#### PTHREAD\_CANCEL\_DISABLE

Thread không thể bị hủy bỏ. Nếu nhận được yêu cầu hủy bỏ, nó sẽ ở trạng thái chờ cho đến khi khả năng hủy bỏ được bật.

#### PTHREAD\_CANCEL\_ENABLE

Thread có thể bị hủy bỏ. Đây là trạng thái khả năng hủy bỏ mặc định trong các thread mới được tạo ra.

Trạng thái khả năng hủy bỏ trước đó của thread được trả về tại vị trí được trỏ bởi `oldstate`.

> Nếu chúng ta không quan tâm đến trạng thái khả năng hủy bỏ trước đó, Linux cho phép chỉ định `oldstate` là `NULL`. Điều này cũng đúng với nhiều implementation khác; tuy nhiên, SUSv3 không quy định tính năng này, vì vậy các ứng dụng portable không thể dựa vào nó. Chúng ta luôn nên chỉ định một giá trị khác `NULL` cho `oldstate`.

Việc tạm thời vô hiệu hóa khả năng hủy bỏ (`PTHREAD_CANCEL_DISABLE`) hữu ích khi một thread đang thực thi một đoạn code mà tất cả các bước phải được hoàn thành.

Nếu một thread có thể bị hủy bỏ (`PTHREAD_CANCEL_ENABLE`), thì cách xử lý yêu cầu hủy bỏ được xác định bởi kiểu khả năng hủy bỏ của thread, được chỉ định bởi đối số `type` trong lệnh gọi `pthread_setcanceltype()`. Đối số này có một trong các giá trị sau:

#### PTHREAD\_CANCEL\_ASYNCHRONOUS

Thread có thể bị hủy bỏ bất kỳ lúc nào (có thể, nhưng không nhất thiết, ngay lập tức). Hủy bỏ không đồng bộ hiếm khi hữu ích, và chúng ta hoãn thảo luận về nó đến Phần 32.6.

#### PTHREAD\_CANCEL\_DEFERRED

Việc hủy bỏ vẫn ở trạng thái chờ cho đến khi một cancellation point (xem phần tiếp theo) được đạt tới. Đây là kiểu khả năng hủy bỏ mặc định trong các thread mới được tạo ra. Chúng ta sẽ nói thêm về hủy bỏ trì hoãn trong các phần tiếp theo.

Kiểu khả năng hủy bỏ trước đó của thread được trả về tại vị trí được trỏ bởi `oldtype`.

> Cũng như với đối số `oldstate` của `pthread_setcancelstate()`, nhiều implementation, bao gồm Linux, cho phép chỉ định `oldtype` là `NULL` nếu chúng ta không quan tâm đến kiểu khả năng hủy bỏ trước đó. Một lần nữa, SUSv3 không quy định tính năng này, và các ứng dụng portable không thể dựa vào nó. Chúng ta luôn nên chỉ định một giá trị khác `NULL` cho `oldtype`.

Khi một thread gọi `fork()`, tiến trình con kế thừa kiểu và trạng thái khả năng hủy bỏ của thread đang gọi. Khi một thread gọi `exec()`, kiểu và trạng thái khả năng hủy bỏ của thread chính của chương trình mới được đặt lại thành `PTHREAD_CANCEL_ENABLE` và `PTHREAD_CANCEL_DEFERRED`, tương ứng.

# **32.3 Cancellation Point**

Khi khả năng hủy bỏ được bật và ở chế độ trì hoãn, một yêu cầu hủy bỏ chỉ được thực thi khi một thread đến cancellation point tiếp theo. Một cancellation point là một lệnh gọi đến một trong số các hàm được xác định bởi implementation.

SUSv3 quy định rằng các hàm được liệt kê trong Bảng 32-1 phải là cancellation point nếu chúng được cung cấp bởi một implementation. Hầu hết trong số này là các hàm có khả năng chặn thread trong một khoảng thời gian không xác định.

**Bảng 32-1:** Các hàm được SUSv3 yêu cầu phải là cancellation point

| accept()          | nanosleep()              | sem_timedwait() |
|-------------------|--------------------------|-----------------|
| aio_suspend()     | open()                   | sem_wait()      |
| clock_nanosleep() | pause()                  | send()          |
| close()           | poll()                   | sendmsg()       |
| connect()         | pread()                  | sendto()        |
| creat()           | pselect()                | sigpause()      |
| fcntl(F_SETLKW)   | pthread_cond_timedwait() | sigsuspend()    |
| fsync()           | pthread_cond_wait()      | sigtimedwait()  |
| fdatasync()       | pthread_join()           | sigwait()       |
| getmsg()          | pthread_testcancel()     | sigwaitinfo()   |
| getpmsg()         | putmsg()                 | sleep()         |
| lockf(F_LOCK)     | putpmsg()                | system()        |
| mq_receive()      | pwrite()                 | tcdrain()       |
| mq_send()         | read()                   | usleep()        |
| mq_timedreceive() | readv()                  | wait()          |
| mq_timedsend()    | recv()                   | waitid()        |
| msgrcv()          | recvfrom()               | waitpid()       |
| msgsnd()          | recvmsg()                | write()         |
| msync()           | select()                 | writev()        |

Ngoài các hàm trong Bảng 32-1, SUSv3 còn quy định một nhóm hàm lớn hơn mà một implementation có thể định nghĩa là cancellation point. Chúng bao gồm các hàm stdio, dlopen API, syslog API, `nftw()`, `popen()`, `semop()`, `unlink()`, và nhiều hàm khác truy xuất thông tin từ các file hệ thống như file utmp. Một chương trình portable phải xử lý đúng khả năng rằng một thread có thể bị hủy bỏ khi gọi các hàm này.

SUSv3 quy định rằng ngoài hai danh sách các hàm phải và có thể là cancellation point, không có hàm nào khác trong tiêu chuẩn có thể hoạt động như cancellation point (tức là, một chương trình portable không cần xử lý khả năng rằng việc gọi các hàm khác này có thể gây ra việc hủy bỏ thread).

SUSv4 thêm `openat()` vào danh sách các hàm phải là cancellation point, và loại bỏ `sigpause()` (chuyển sang danh sách các hàm có thể là cancellation point) và `usleep()` (bị loại bỏ khỏi tiêu chuẩn).

> Một implementation có quyền đánh dấu thêm các hàm không được quy định trong tiêu chuẩn là cancellation point. Bất kỳ hàm nào có thể bị chặn (có thể vì nó có thể truy cập file) đều là ứng cử viên tiềm năng để trở thành cancellation point. Trong glibc, nhiều hàm không chuẩn được đánh dấu là cancellation point vì lý do này.

Khi nhận được yêu cầu hủy bỏ, một thread có khả năng hủy bỏ được bật và ở chế độ trì hoãn sẽ kết thúc khi nó đến cancellation point tiếp theo. Nếu thread không được detach, thì một thread khác trong process phải join với nó, để ngăn nó trở thành zombie thread. Khi một thread bị hủy bỏ được join, giá trị được trả về trong đối số thứ hai của `pthread_join()` là một giá trị trả về đặc biệt của thread: `PTHREAD_CANCELED`.

### **Chương trình ví dụ**

Listing 32-1 cho thấy một ví dụ đơn giản về việc sử dụng `pthread_cancel()`. Chương trình chính tạo ra một thread thực thi một vòng lặp vô hạn, ngủ một giây và in giá trị của biến đếm vòng lặp. (Thread này sẽ chỉ kết thúc nếu nó nhận được yêu cầu hủy bỏ hoặc nếu process thoát.) Trong khi đó, chương trình chính ngủ 3 giây, sau đó gửi yêu cầu hủy bỏ đến thread mà nó đã tạo. Khi chúng ta chạy chương trình này, chúng ta thấy như sau:

```
$ ./t_pthread_cancel
New thread started
Loop 1
Loop 2
Loop 3
Thread was canceled
```

**Listing 32-1:** Hủy bỏ một thread với `pthread_cancel()`

```
––––––––––––––––––––––––––––––––––––––––––––––––––– threads/thread_cancel.c
#include <pthread.h>
#include "tlpi_hdr.h"
static void *
threadFunc(void *arg)
{
 int j;
```

```
 printf("New thread started\n"); /* May be a cancellation point */
 for (j = 1; ; j++) {
 printf("Loop %d\n", j); /* May be a cancellation point */
 sleep(1); /* A cancellation point */
 }
 /* NOTREACHED */
 return NULL;
}
int
main(int argc, char *argv[])
{
 pthread_t thr;
 int s;
 void *res;
 s = pthread_create(&thr, NULL, threadFunc, NULL);
 if (s != 0)
 errExitEN(s, "pthread_create");
 sleep(3); /* Allow new thread to run a while */
 s = pthread_cancel(thr);
 if (s != 0)
 errExitEN(s, "pthread_cancel");
 s = pthread_join(thr, &res);
 if (s != 0)
 errExitEN(s, "pthread_join");
 if (res == PTHREAD_CANCELED)
 printf("Thread was canceled\n");
 else
 printf("Thread was not canceled (should not happen!)\n");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––– threads/thread_cancel.c
```

## **32.4 Kiểm Tra Hủy Bỏ Thread**

Trong Listing 32-1, thread được tạo bởi `main()` đã chấp nhận yêu cầu hủy bỏ vì nó thực thi một hàm là cancellation point (`sleep()` là cancellation point; `printf()` có thể là một). Tuy nhiên, giả sử một thread thực thi một vòng lặp không chứa cancellation point nào (ví dụ: một vòng lặp compute-bound). Trong trường hợp này, thread sẽ không bao giờ thực hiện yêu cầu hủy bỏ.

Mục đích của `pthread_testcancel()` đơn giản chỉ là để trở thành một cancellation point. Nếu có một yêu cầu hủy bỏ đang chờ khi hàm này được gọi, thread đang gọi sẽ bị kết thúc.

```
#include <pthread.h>
void pthread_testcancel(void);
```

Một thread đang thực thi code mà không chứa cancellation point nào có thể định kỳ gọi `pthread_testcancel()` để đảm bảo rằng nó phản hồi kịp thời với yêu cầu hủy bỏ được gửi bởi thread khác.

## **32.5 Cleanup Handler**

Nếu một thread có yêu cầu hủy bỏ đang chờ đơn giản bị kết thúc khi nó đến cancellation point, thì các biến chia sẻ và các đối tượng Pthreads (ví dụ: mutex) có thể bị để lại ở trạng thái không nhất quán, có thể khiến các thread còn lại trong process tạo ra kết quả không chính xác, deadlock, hoặc crash. Để giải quyết vấn đề này, một thread có thể thiết lập một hoặc nhiều cleanup handler — các hàm được tự động thực thi nếu thread bị hủy bỏ. Một cleanup handler có thể thực hiện các tác vụ như sửa đổi giá trị của các biến global và giải phóng mutex trước khi thread bị kết thúc.

Mỗi thread có thể có một stack các cleanup handler. Khi một thread bị hủy bỏ, các cleanup handler được thực thi từ trên xuống của stack; tức là, handler được thiết lập gần đây nhất được gọi trước, sau đó đến handler được thiết lập gần đây nhất tiếp theo, v.v. Khi tất cả các cleanup handler đã được thực thi, thread kết thúc.

Các hàm `pthread_cleanup_push()` và `pthread_cleanup_pop()` lần lượt thêm và xóa các handler trên stack cleanup handler của thread đang gọi.

```
#include <pthread.h>
void pthread_cleanup_push(void (*routine)(void*), void *arg);
void pthread_cleanup_pop(int execute);
```

Hàm `pthread_cleanup_push()` thêm hàm có địa chỉ được chỉ định trong `routine` lên đầu stack cleanup handler của thread đang gọi. Đối số `routine` là một con trỏ đến một hàm có dạng sau:

```
void
routine(void *arg)
{
 /* Code to perform cleanup */
}
```

Giá trị `arg` được truyền cho `pthread_cleanup_push()` được truyền làm đối số của cleanup handler khi nó được gọi. Đối số này được khai báo là `void *`, nhưng, thông qua việc ép kiểu thích hợp, có thể truyền các kiểu dữ liệu khác trong đối số này.

Thông thường, một hành động cleanup chỉ cần thiết nếu một thread bị hủy bỏ trong quá trình thực thi một đoạn code cụ thể. Nếu thread đến cuối đoạn code đó mà không bị hủy bỏ, thì hành động cleanup không còn cần thiết nữa. Do đó, mỗi lệnh gọi `pthread_cleanup_push()` đều có một lệnh gọi `pthread_cleanup_pop()` tương ứng. Hàm này xóa hàm trên cùng khỏi stack cleanup handler. Nếu đối số `execute` khác không, handler cũng được thực thi. Điều này thuận tiện nếu chúng ta muốn thực hiện hành động cleanup ngay cả khi thread không bị hủy bỏ.

Mặc dù chúng ta đã mô tả `pthread_cleanup_push()` và `pthread_cleanup_pop()` là các hàm, SUSv3 cho phép chúng được implement như các macro mở rộng thành các chuỗi câu lệnh bao gồm dấu ngoặc mở (`{`) và đóng (`}`), tương ứng. Không phải tất cả các implementation UNIX đều làm theo cách này, nhưng Linux và nhiều hệ thống khác thì có. Điều này có nghĩa là mỗi lần sử dụng `pthread_cleanup_push()` phải được ghép đôi với đúng một lệnh `pthread_cleanup_pop()` tương ứng trong cùng một khối lexical. (Trên các implementation làm theo cách này, các biến được khai báo giữa `pthread_cleanup_push()` và `pthread_cleanup_pop()` sẽ bị giới hạn trong phạm vi lexical đó.) Ví dụ, không đúng khi viết code như sau:

```
pthread_cleanup_push(func, arg);
...
if (cond) {
 pthread_cleanup_pop(0);
}
```

Như một tiện ích trong lập trình, bất kỳ cleanup handler nào chưa được pop ra cũng được thực thi tự động nếu một thread kết thúc bằng cách gọi `pthread_exit()` (nhưng không nếu nó thực hiện một lệnh `return` đơn giản).

#### **Chương trình ví dụ**

Chương trình trong Listing 32-2 cung cấp một ví dụ đơn giản về việc sử dụng cleanup handler. Chương trình chính tạo ra một thread mà các hành động đầu tiên là cấp phát một khối bộ nhớ có địa chỉ được lưu trong `buf`, và sau đó khóa mutex `mtx`. Vì thread có thể bị hủy bỏ, nó sử dụng `pthread_cleanup_push()` để cài đặt một cleanup handler được gọi với địa chỉ được lưu trong `buf`. Nếu được gọi, cleanup handler sẽ giải phóng bộ nhớ và mở khóa mutex.

Thread sau đó vào một vòng lặp chờ condition variable `cond` được báo hiệu. Vòng lặp này sẽ kết thúc theo một trong hai cách, tùy thuộc vào việc chương trình có được cung cấp đối số dòng lệnh hay không:

-  Nếu không có đối số dòng lệnh nào được cung cấp, thread bị hủy bỏ bởi `main()`. Trong trường hợp này, việc hủy bỏ sẽ xảy ra tại lệnh gọi `pthread_cond_wait()`, là một trong các cancellation point được liệt kê trong Bảng 32-1. Là một phần của việc hủy bỏ, cleanup handler được thiết lập bằng `pthread_cleanup_push()` được gọi tự động.
-  Nếu có đối số dòng lệnh được cung cấp, condition variable được báo hiệu sau khi biến global `glob` được đặt thành một giá trị khác không. Trong trường hợp này, thread tiếp tục thực thi `pthread_cleanup_pop()`, với đối số khác không, cũng khiến cleanup handler được gọi.

Chương trình chính join với thread đã kết thúc và báo cáo liệu thread có bị hủy bỏ hay kết thúc bình thường.

```
–––––––––––––––––––––––––––––––––––––––––––––––––– threads/thread_cleanup.c
  #include <pthread.h>
  #include "tlpi_hdr.h"
  static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
  static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
  static int glob = 0; /* Predicate variable */
  static void /* Free memory pointed to by 'arg' and unlock mutex */
  cleanupHandler(void *arg)
  {
   int s;
   printf("cleanup: freeing block at %p\n", arg);
q free(arg);
   printf("cleanup: unlocking mutex\n");
w s = pthread_mutex_unlock(&mtx);
   if (s != 0)
   errExitEN(s, "pthread_mutex_unlock");
  }
  static void *
  threadFunc(void *arg)
  {
   int s;
   void *buf = NULL; /* Buffer allocated by thread */
e buf = malloc(0x10000); /* Not a cancellation point */
   printf("thread: allocated memory at %p\n", buf);
r s = pthread_mutex_lock(&mtx); /* Not a cancellation point */
   if (s != 0)
   errExitEN(s, "pthread_mutex_lock");
t pthread_cleanup_push(cleanupHandler, buf);
   while (glob == 0) {
y s = pthread_cond_wait(&cond, &mtx); /* A cancellation point */
   if (s != 0)
   errExitEN(s, "pthread_cond_wait");
   }
   printf("thread: condition wait loop completed\n");
u pthread_cleanup_pop(1); /* Executes cleanup handler */
   return NULL;
  }
  int
  main(int argc, char *argv[])
  {
   pthread_t thr;
   void *res;
   int s;
```

```
i s = pthread_create(&thr, NULL, threadFunc, NULL);
   if (s != 0)
   errExitEN(s, "pthread_create");
   sleep(2); /* Give thread a chance to get started */
   if (argc == 1) { /* Cancel thread */
   printf("main: about to cancel thread\n");
o s = pthread_cancel(thr);
   if (s != 0)
   errExitEN(s, "pthread_cancel");
   } else { /* Signal condition variable */
   printf("main: about to signal condition variable\n");
   glob = 1;
a s = pthread_cond_signal(&cond);
   if (s != 0)
   errExitEN(s, "pthread_cond_signal");
   }
s s = pthread_join(thr, &res);
   if (s != 0)
   errExitEN(s, "pthread_join");
   if (res == PTHREAD_CANCELED)
   printf("main: thread was canceled\n");
   else
   printf("main: thread terminated normally\n");
   exit(EXIT_SUCCESS);
  }
  –––––––––––––––––––––––––––––––––––––––––––––––––– threads/thread_cleanup.c
```

Nếu chúng ta gọi chương trình trong Listing 32-2 mà không có đối số dòng lệnh nào, thì `main()` gọi `pthread_cancel()`, cleanup handler được gọi tự động, và chúng ta thấy như sau:

#### \$ **./thread\_cleanup**

thread: allocated memory at 0x804b050 main: about to cancel thread cleanup: freeing block at 0x804b050 cleanup: unlocking mutex main: thread was canceled

Nếu chúng ta gọi chương trình với một đối số dòng lệnh, thì `main()` đặt `glob` thành 1 và báo hiệu condition variable, cleanup handler được gọi bởi `pthread_cleanup_pop()`, và chúng ta thấy như sau:

#### \$ **./thread\_cleanup s**

thread: allocated memory at 0x804b050 main: about to signal condition variable thread: condition wait loop completed cleanup: freeing block at 0x804b050 cleanup: unlocking mutex main: thread terminated normally

## **32.6 Hủy Bỏ Không Đồng Bộ**

Khi một thread được đặt thành có thể hủy bỏ không đồng bộ (kiểu khả năng hủy bỏ `PTHREAD_CANCEL_ASYNCHRONOUS`), nó có thể bị hủy bỏ bất kỳ lúc nào (tức là, tại bất kỳ lệnh ngôn ngữ máy nào); việc thực hiện hủy bỏ không bị trì hoãn cho đến khi thread đến cancellation point tiếp theo.

Vấn đề với hủy bỏ không đồng bộ là, mặc dù các cleanup handler vẫn được gọi, các handler không có cách nào để xác định trạng thái của một thread. Trong chương trình trong Listing 32-2, sử dụng kiểu khả năng hủy bỏ trì hoãn, thread chỉ có thể bị hủy bỏ khi nó thực thi lệnh gọi `pthread_cond_wait()`, là cancellation point duy nhất. Vào thời điểm này, chúng ta biết rằng `buf` đã được khởi tạo để trỏ đến một khối bộ nhớ được cấp phát và mutex `mtx` đã được khóa. Tuy nhiên, với hủy bỏ không đồng bộ, thread có thể bị hủy bỏ tại bất kỳ điểm nào; ví dụ, trước lệnh gọi `malloc()`, giữa lệnh gọi `malloc()` và việc khóa mutex, hoặc sau khi khóa mutex. Cleanup handler không có cách nào biết việc hủy bỏ đã xảy ra ở đâu, hoặc chính xác cần thực hiện các bước cleanup nào. Hơn nữa, thread thậm chí có thể bị hủy bỏ trong quá trình gọi `malloc()`, sau đó tình trạng hỗn loạn có thể xảy ra (Phần 7.1.3).

Như một nguyên tắc chung, một thread có thể bị hủy bỏ không đồng bộ không thể cấp phát bất kỳ tài nguyên nào hoặc chiếm bất kỳ mutex, semaphore, hay lock nào. Điều này loại trừ việc sử dụng nhiều hàm thư viện, bao gồm hầu hết các hàm Pthreads. (SUSv3 đưa ra ngoại lệ cho `pthread_cancel()`, `pthread_setcancelstate()`, và `pthread_setcanceltype()`, được yêu cầu rõ ràng phải an toàn với hủy bỏ không đồng bộ; tức là, một implementation phải làm cho chúng an toàn để gọi từ một thread có thể bị hủy bỏ không đồng bộ.) Nói cách khác, có rất ít trường hợp mà hủy bỏ không đồng bộ là hữu ích. Một trường hợp như vậy là hủy bỏ một thread đang trong một vòng lặp compute-bound.

## **32.7 Tổng Kết**

Hàm `pthread_cancel()` cho phép một thread gửi cho một thread khác yêu cầu hủy bỏ, là một yêu cầu rằng thread đích nên kết thúc.

Cách thread đích phản ứng với yêu cầu này được xác định bởi trạng thái và kiểu khả năng hủy bỏ của nó. Nếu trạng thái khả năng hủy bỏ hiện tại được đặt thành vô hiệu hóa, yêu cầu sẽ ở trạng thái chờ cho đến khi trạng thái khả năng hủy bỏ được đặt thành bật. Nếu khả năng hủy bỏ được bật, kiểu khả năng hủy bỏ xác định khi nào thread đích phản ứng với yêu cầu. Nếu kiểu là trì hoãn, việc hủy bỏ xảy ra khi thread tiếp theo gọi một trong số các hàm được SUSv3 quy định là cancellation point. Nếu kiểu là không đồng bộ, hủy bỏ có thể xảy ra bất kỳ lúc nào (điều này hiếm khi hữu ích).

Một thread có thể thiết lập một stack các cleanup handler, là các hàm do lập trình viên định nghĩa được tự động gọi để thực hiện cleanup (ví dụ: khôi phục trạng thái của các biến chia sẻ, hoặc giải phóng mutex) nếu thread bị hủy bỏ.

#### **Thông tin thêm**

Tham khảo các nguồn thông tin thêm được liệt kê trong Phần 29.10.
