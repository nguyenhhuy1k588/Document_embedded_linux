## Chương 33
# **THREADS: CHI TIẾT THÊM**

Chương này cung cấp thêm chi tiết về các khía cạnh khác nhau của POSIX threads. Chúng ta thảo luận về sự tương tác của thread với các khía cạnh của UNIX API truyền thống — cụ thể là signal và các nguyên hàm kiểm soát process (`fork()`, `exec()`, và `_exit()`). Chúng ta cũng cung cấp tổng quan về hai implementation POSIX threads có sẵn trên Linux — LinuxThreads và NPTL — và ghi chú nơi mỗi implementation này lệch khỏi đặc tả SUSv3 của Pthreads.

# **33.1 Stack của Thread**

Mỗi thread có stack riêng của nó với kích thước được cố định khi thread được tạo. Trên Linux/x86-32, đối với tất cả các thread ngoài thread chính, kích thước mặc định của per-thread stack là 2 MB. (Trên một số kiến trúc 64-bit, kích thước mặc định cao hơn; ví dụ, trên IA-64 là 32 MB.) Thread chính có không gian lớn hơn nhiều để stack phát triển (tham khảo Hình 29-1, trang 618).

Đôi khi, việc thay đổi kích thước stack của một thread là hữu ích. Hàm `pthread_attr_setstacksize()` thiết lập một thread attribute (Phần 29.8) xác định kích thước của stack trong các thread được tạo bằng đối tượng thread attributes. Hàm liên quan `pthread_attr_setstack()` có thể được dùng để kiểm soát cả kích thước và vị trí của stack, nhưng việc thiết lập vị trí của stack có thể giảm tính portable của ứng dụng. Các trang manual cung cấp chi tiết về các hàm này.

Một lý do để thay đổi kích thước của per-thread stack là cho phép stack lớn hơn cho các thread cấp phát các biến tự động lớn hoặc thực hiện các lời gọi hàm lồng nhau sâu (có thể do đệ quy). Ngoài ra, một ứng dụng có thể muốn giảm kích thước của per-thread stack để cho phép số lượng thread lớn hơn trong một process. Ví dụ, trên x86-32, nơi virtual address space có thể truy cập của người dùng là 3 GB, kích thước stack mặc định là 2 MB có nghĩa là chúng ta có thể tạo tối đa khoảng 1500 thread. (Tối đa chính xác phụ thuộc vào lượng virtual memory được tiêu thụ bởi các segment text và data, shared library, v.v.) Stack tối thiểu có thể được sử dụng trên một kiến trúc cụ thể có thể được xác định bằng cách gọi `sysconf(_SC_THREAD_STACK_MIN)`. Đối với implementation NPTL trên Linux/x86-32, lệnh gọi này trả về giá trị 16.384.

> Trong implementation threading NPTL, nếu giới hạn tài nguyên kích thước stack (`RLIMIT_STACK`) được đặt thành bất kỳ thứ gì khác với unlimited, thì nó được sử dụng làm kích thước stack mặc định khi tạo thread mới. Giới hạn này phải được đặt trước khi chương trình được thực thi, thông thường bằng cách sử dụng lệnh dựng sẵn `ulimit –s` của shell (hoặc `limit stacksize` trong C shell) trước khi thực thi chương trình. Việc sử dụng `setrlimit()` trong chương trình chính để đặt giới hạn là không đủ, vì NPTL thực hiện việc xác định kích thước stack mặc định trong quá trình khởi tạo run-time xảy ra trước khi `main()` được gọi.

# **33.2 Thread và Signal**

Mô hình signal UNIX được thiết kế với mô hình process UNIX trong đầu, và xuất hiện trước Pthreads vài thập kỷ. Do đó, có một số xung đột đáng kể giữa mô hình signal và thread. Các xung đột này nảy sinh chủ yếu từ nhu cầu duy trì ngữ nghĩa signal truyền thống cho các process single-threaded (tức là, ngữ nghĩa signal của các chương trình truyền thống không nên bị thay đổi bởi Pthreads), đồng thời phát triển một mô hình signal có thể sử dụng được trong một process đa luồng.

Sự khác biệt giữa mô hình signal và thread có nghĩa là việc kết hợp signal và thread là phức tạp, và nên tránh bất cứ khi nào có thể. Tuy nhiên, đôi khi chúng ta phải xử lý signal trong một chương trình threaded. Trong phần này, chúng ta thảo luận về sự tương tác giữa thread và signal, và mô tả các hàm hữu ích trong các chương trình threaded xử lý signal.

## **33.2.1 Cách Ánh Xạ Mô Hình Signal UNIX vào Thread**

Để hiểu cách signal UNIX ánh xạ vào mô hình Pthreads, chúng ta cần biết các khía cạnh nào của mô hình signal là process-wide (tức là, được chia sẻ bởi tất cả các thread trong process) so với các khía cạnh cụ thể của từng thread riêng lẻ trong process. Danh sách sau tóm tắt các điểm chính:

-  Các hành động signal là process-wide. Nếu bất kỳ signal nào chưa được xử lý có hành động mặc định là dừng hoặc kết thúc được gửi đến bất kỳ thread nào trong một process, thì tất cả các thread trong process đều bị dừng hoặc kết thúc.
-  Disposition của signal là process-wide; tất cả các thread trong một process chia sẻ cùng disposition cho mỗi signal. Nếu một thread sử dụng `sigaction()` để thiết lập handler cho, chẳng hạn, `SIGINT`, thì handler đó có thể được gọi từ bất kỳ thread nào mà `SIGINT` được gửi đến. Tương tự, nếu một thread đặt disposition của signal thành ignore, thì signal đó bị bỏ qua bởi tất cả các thread.

-  Một signal có thể được hướng đến toàn bộ process hoặc đến một thread cụ thể. Một signal là thread-directed nếu:
  - nó được tạo ra như kết quả trực tiếp của việc thực thi một lệnh phần cứng cụ thể trong context của thread (tức là, các ngoại lệ phần cứng được mô tả trong Phần 22.4: `SIGBUS`, `SIGFPE`, `SIGILL`, và `SIGSEGV`);
  - nó là signal `SIGPIPE` được tạo ra khi thread cố gắng ghi vào một pipe bị hỏng; hoặc
  - nó được gửi bằng `pthread_kill()` hoặc `pthread_sigqueue()`, là các hàm (được mô tả trong Phần 33.2.3) cho phép một thread gửi signal đến một thread khác trong cùng process.

Tất cả các signal được tạo bởi các cơ chế khác đều là process-directed. Ví dụ là các signal được gửi từ process khác sử dụng `kill()` hoặc `sigqueue()`; các signal như `SIGINT` và `SIGTSTP`, được tạo ra khi người dùng gõ một trong các ký tự terminal đặc biệt tạo ra signal; và các signal được tạo ra cho các sự kiện phần mềm như việc thay đổi kích thước cửa sổ terminal (`SIGWINCH`) hoặc hết hạn của timer (ví dụ: `SIGALRM`).

-  Khi một signal được gửi đến một process đa luồng đã thiết lập signal handler, kernel chọn tùy ý một thread trong process để gửi signal và gọi handler trong thread đó. Hành vi này nhất quán với việc duy trì ngữ nghĩa signal truyền thống. Sẽ không có ý nghĩa gì khi một process thực hiện các hành động xử lý signal nhiều lần để phản hồi một signal duy nhất.
-  Signal mask là per-thread. (Không có khái niệm về process-wide signal mask quản lý tất cả các thread trong một process đa luồng.) Các thread có thể độc lập block hoặc unblock các signal khác nhau bằng cách sử dụng `pthread_sigmask()`, một hàm mới được định nghĩa bởi Pthreads API. Bằng cách thao tác với per-thread signal mask, một ứng dụng có thể kiểm soát (các) thread nào có thể xử lý một signal được hướng đến toàn bộ process.
-  Kernel duy trì một bản ghi các signal đang chờ cho toàn bộ process, cũng như một bản ghi các signal đang chờ cho mỗi thread. Một lệnh gọi `sigpending()` trả về hợp của tập các signal đang chờ cho process và các signal đang chờ cho thread đang gọi. Trong một thread mới được tạo, tập per-thread của các signal đang chờ ban đầu trống. Một thread-directed signal chỉ có thể được gửi đến thread đích. Nếu thread đang block signal, nó sẽ ở trạng thái chờ cho đến khi thread unblock signal (hoặc kết thúc).
-  Nếu một signal handler làm gián đoạn một lệnh gọi `pthread_mutex_lock()`, thì lệnh gọi luôn được tự động khởi động lại. Nếu một signal handler làm gián đoạn một lệnh gọi `pthread_cond_wait()`, thì lệnh gọi either được tự động khởi động lại (đây là điều Linux làm) hoặc trả về 0, cho thấy một spurious wake-up (trong trường hợp này, một ứng dụng được thiết kế tốt sẽ kiểm tra lại predicate tương ứng và khởi động lại lệnh gọi, như được mô tả trong Phần 30.2.3). SUSv3 yêu cầu hai hàm này hoạt động như được mô tả ở đây.
-  Alternate signal stack là per-thread (tham khảo mô tả về `sigaltstack()` trong Phần 21.3). Một thread mới được tạo không kế thừa alternate signal stack từ thread tạo ra nó.

Chính xác hơn, SUSv3 quy định rằng có một alternate signal stack riêng biệt cho mỗi kernel scheduling entity (KSE). Trên một hệ thống với implementation threading 1:1, như trên Linux, có một KSE cho mỗi thread (xem Phần 33.4).

## **33.2.2 Thao Tác với Thread Signal Mask**

Khi một thread mới được tạo, nó kế thừa một bản sao của signal mask của thread đã tạo ra nó. Một thread có thể sử dụng `pthread_sigmask()` để thay đổi signal mask của nó, để truy xuất mask hiện tại, hoặc cả hai.

```
#include <signal.h>
int pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset);
                       Trả về 0 nếu thành công, hoặc số lỗi dương nếu có lỗi
```

Ngoài thực tế là nó hoạt động trên thread signal mask, việc sử dụng `pthread_sigmask()` giống như việc sử dụng `sigprocmask()` (Phần 20.10).

> SUSv3 lưu ý rằng việc sử dụng `sigprocmask()` trong một chương trình đa luồng là không được xác định. Chúng ta không thể sử dụng `sigprocmask()` một cách portable trong một chương trình đa luồng. Trên thực tế, `sigprocmask()` và `pthread_sigmask()` giống hệt nhau trên nhiều implementation, bao gồm Linux.

## **33.2.3 Gửi Signal đến Thread**

Hàm `pthread_kill()` gửi signal `sig` đến một thread khác trong cùng process. Thread đích được xác định bởi đối số `thread`.

```
#include <signal.h>
int pthread_kill(pthread_t thread, int sig);
                        Trả về 0 nếu thành công, hoặc số lỗi dương nếu có lỗi
```

Vì một thread ID được đảm bảo là duy nhất chỉ trong một process (xem Phần 29.5), chúng ta không thể sử dụng `pthread_kill()` để gửi signal đến một thread trong process khác.

> Hàm `pthread_kill()` được implement bằng cách sử dụng system call đặc thù của Linux `tgkill(tgid, tid, sig)`, gửi signal `sig` đến thread được xác định bởi `tid` (một kernel thread ID thuộc kiểu được trả về bởi `gettid()`) trong thread group được xác định bởi `tgid`. Xem trang manual `tgkill(2)` để biết thêm chi tiết.

Hàm `pthread_sigqueue()` đặc thù của Linux kết hợp chức năng của `pthread_kill()` và `sigqueue()` (Phần 22.8.1): nó gửi một signal kèm theo dữ liệu đến một thread khác trong cùng process.

```
#define _GNU_SOURCE
#include <signal.h>
int pthread_sigqueue(pthread_t thread, int sig, const union sigval value);
                       Trả về 0 nếu thành công, hoặc số lỗi dương nếu có lỗi
```

Cũng như với `pthread_kill()`, `sig` chỉ định signal cần gửi, và `thread` xác định thread đích. Đối số `value` chỉ định dữ liệu đi kèm với signal, và được sử dụng theo cách tương tự như đối số tương đương của `sigqueue()`.

> Hàm `pthread_sigqueue()` được thêm vào glibc trong phiên bản 2.11 và cần sự hỗ trợ từ kernel. Sự hỗ trợ này được cung cấp bởi system call `rt_tgsigqueueinfo()`, được thêm vào trong Linux 2.6.31.

## **33.2.4 Xử Lý Signal Không Đồng Bộ Một Cách Hợp Lý**

Trong các Chương 20 đến 22, chúng ta đã thảo luận về các yếu tố khác nhau — như vấn đề reentrancy, nhu cầu khởi động lại các system call bị gián đoạn, và tránh race condition — có thể làm phức tạp việc xử lý signal được tạo ra không đồng bộ thông qua signal handler. Hơn nữa, không có hàm nào trong Pthreads API nằm trong tập hợp các hàm async-signal-safe mà chúng ta có thể gọi an toàn từ trong một signal handler (Phần 21.1.2). Vì những lý do này, các chương trình đa luồng phải xử lý signal được tạo ra không đồng bộ thường không nên sử dụng signal handler như cơ chế để nhận thông báo về việc gửi signal. Thay vào đó, cách tiếp cận được ưa thích là như sau:

-  Tất cả các thread block tất cả các signal không đồng bộ mà process có thể nhận. Cách đơn giản nhất để làm điều này là block các signal trong thread chính trước khi bất kỳ thread nào khác được tạo. Mỗi thread được tạo sau đó sẽ kế thừa một bản sao của signal mask của thread chính.
-  Tạo một thread chuyên dụng duy nhất chấp nhận các signal đến bằng cách sử dụng `sigwaitinfo()`, `sigtimedwait()`, hoặc `sigwait()`. Chúng ta đã mô tả `sigwaitinfo()` và `sigtimedwait()` trong Phần 22.10. Chúng ta mô tả `sigwait()` bên dưới.

Ưu điểm của cách tiếp cận này là các signal được tạo ra không đồng bộ được nhận đồng bộ. Khi nhận các signal đến, thread chuyên dụng có thể an toàn sửa đổi các biến chia sẻ (dưới kiểm soát của mutex) và gọi các hàm không an toàn với async-signal. Nó cũng có thể báo hiệu condition variable, và sử dụng các cơ chế giao tiếp và đồng bộ hóa thread và process khác.

Hàm `sigwait()` chờ việc gửi một trong các signal trong tập signal được trỏ bởi `set`, chấp nhận signal đó, và trả về nó trong `sig`.

```
#include <signal.h>
int sigwait(const sigset_t *set, int *sig);
                      Trả về 0 nếu thành công, hoặc số lỗi dương nếu có lỗi
```

Hoạt động của `sigwait()` giống như `sigwaitinfo()`, ngoại trừ:

-  thay vì trả về một cấu trúc `siginfo_t` mô tả signal, `sigwait()` chỉ trả về số hiệu signal; và
-  giá trị trả về nhất quán với các hàm liên quan đến thread khác (thay vì 0 hoặc –1 được trả về bởi các system call UNIX truyền thống).

Nếu nhiều thread đang chờ cùng một signal với `sigwait()`, chỉ một trong các thread thực sự sẽ chấp nhận signal khi nó đến. Thread nào trong số này sẽ là không xác định.

# **33.3 Thread và Kiểm Soát Process**

Giống như cơ chế signal, `exec()`, `fork()`, và `exit()` xuất hiện trước Pthreads API. Trong các đoạn sau, chúng ta ghi chú một số chi tiết liên quan đến việc sử dụng các system call này trong các chương trình threaded.

### **Thread và exec()**

Khi bất kỳ thread nào gọi một trong các hàm `exec()`, chương trình đang gọi bị thay thế hoàn toàn. Tất cả các thread, ngoại trừ thread đã gọi `exec()`, biến mất ngay lập tức. Không có thread nào thực thi destructor cho thread-specific data hoặc gọi cleanup handler. Tất cả các mutex và condition variable (thuộc về private của process) thuộc về process cũng biến mất. Sau một `exec()`, thread ID của thread còn lại là không xác định.

## **Thread và fork()**

Khi một process đa luồng gọi `fork()`, chỉ có thread đang gọi được sao chép trong process con. (ID của thread trong con giống với ID của thread đã gọi `fork()` trong cha.) Tất cả các thread khác biến mất trong con; không có destructor thread-specific data hoặc cleanup handler nào được thực thi cho các thread đó. Điều này có thể dẫn đến các vấn đề khác nhau:

-  Mặc dù chỉ có thread đang gọi được sao chép trong con, trạng thái của các biến global, cũng như tất cả các đối tượng Pthreads như mutex và condition variable, được giữ nguyên trong con. (Điều này là bởi vì các đối tượng Pthreads này được cấp phát trong bộ nhớ của cha, và con nhận được bản sao của bộ nhớ đó.) Điều này có thể dẫn đến các tình huống phức tạp. Ví dụ, giả sử một thread khác đã khóa mutex tại thời điểm `fork()` và đang cập nhật một cấu trúc dữ liệu global ở giữa chừng. Trong trường hợp này, thread trong con sẽ không thể mở khóa mutex (vì nó không phải chủ sở hữu mutex) và sẽ bị block nếu nó cố gắng chiếm mutex. Hơn nữa, bản sao của cấu trúc dữ liệu global trong con có thể đang ở trạng thái không nhất quán, vì thread đang cập nhật nó đã biến mất giữa chừng.
-  Vì destructor cho thread-specific data và cleanup handler không được gọi, một `fork()` trong một chương trình đa luồng có thể gây ra memory leak trong con. Hơn nữa, các mục thread-specific data được tạo bởi các thread khác có thể không truy cập được với thread trong con mới, vì nó không có các con trỏ tham chiếu đến các mục này.

Vì những vấn đề này, khuyến nghị thông thường là việc sử dụng duy nhất `fork()` trong một process đa luồng nên là một lệnh gọi được theo sau ngay bởi một `exec()`. `exec()` khiến tất cả các đối tượng Pthreads trong process con biến mất khi chương trình mới ghi đè bộ nhớ của process.

Đối với các chương trình phải sử dụng `fork()` không được theo sau bởi `exec()`, Pthreads API cung cấp một cơ chế để định nghĩa fork handler. Fork handler được thiết lập bằng cách sử dụng lệnh gọi `pthread_atfork()` có dạng sau:

```
pthread_atfork(prepare_func, parent_func, child_func);
```

Mỗi lệnh gọi `pthread_atfork()` thêm `prepare_func` vào danh sách các hàm sẽ được tự động thực thi (theo thứ tự ngược của đăng ký) trước khi process con mới được tạo khi `fork()` được gọi. Tương tự, `parent_func` và `child_func` được thêm vào danh sách các hàm sẽ được gọi tự động (theo thứ tự đăng ký), lần lượt trong process cha và con, ngay trước khi `fork()` trả về.

Fork handler đôi khi hữu ích cho code thư viện sử dụng thread. Nếu không có fork handler, sẽ không có cách nào để thư viện xử lý các ứng dụng ngây thơ sử dụng thư viện và gọi `fork()`, không biết rằng thư viện đã tạo ra một số thread.

Process con được tạo bởi `fork()` kế thừa fork handler từ thread đã gọi `fork()`. Trong quá trình `exec()`, fork handler không được giữ lại (chúng không thể được, vì code của các handler bị ghi đè trong quá trình `exec()`).

Thêm chi tiết về fork handler, và ví dụ về việc sử dụng chúng, có thể được tìm thấy trong [Butenhof, 1996].

> Trên Linux, fork handler không được gọi nếu một chương trình sử dụng thư viện threading NPTL gọi `vfork()`. Tuy nhiên, trong một chương trình sử dụng LinuxThreads, fork handler được gọi trong trường hợp này.

#### **Thread và exit()**

Nếu bất kỳ thread nào gọi `exit()` hoặc, tương đương, thread chính thực hiện một `return`, tất cả các thread biến mất ngay lập tức; không có destructor thread-specific data hoặc cleanup handler nào được thực thi.

## **33.4 Mô Hình Triển Khai Thread**

Trong phần này, chúng ta đi vào một số lý thuyết, xem xét ngắn gọn ba mô hình khác nhau để triển khai một threading API. Điều này cung cấp nền tảng hữu ích cho Phần 33.5, nơi chúng ta xem xét các implementation threading của Linux. Sự khác biệt giữa các mô hình triển khai này xoay quanh cách các thread được ánh xạ vào kernel scheduling entity (KSE), là các đơn vị mà kernel phân bổ CPU và các tài nguyên hệ thống khác. (Trong các implementation UNIX truyền thống trước thread, thuật ngữ kernel scheduling entity đồng nghĩa với thuật ngữ process.)

#### **Triển khai Many-to-one (M:1) (user-level thread)**

Trong các implementation threading M:1, tất cả các chi tiết về tạo thread, lập lịch, và đồng bộ hóa (khóa mutex, chờ trên condition variable, v.v.) được xử lý hoàn toàn trong process bởi một thư viện threading user-space. Kernel không biết gì về sự tồn tại của nhiều thread trong process.

Các implementation M:1 có một số ưu điểm. Ưu điểm lớn nhất là nhiều hoạt động threading — ví dụ: tạo và kết thúc thread, chuyển ngữ cảnh giữa các thread, và các hoạt động mutex và condition variable — nhanh chóng, vì không cần chuyển sang kernel mode. Hơn nữa, vì sự hỗ trợ kernel cho thư viện threading không cần thiết, một implementation M:1 có thể được port tương đối dễ dàng từ hệ thống này sang hệ thống khác.

Tuy nhiên, các implementation M:1 có một số nhược điểm nghiêm trọng:

-  Khi một thread thực hiện một system call như `read()`, quyền kiểm soát chuyển từ thư viện threading user-space sang kernel. Điều này có nghĩa là nếu lệnh gọi `read()` bị block, thì tất cả các thread trong process đều bị block.
-  Kernel không thể lập lịch các thread của một process. Vì kernel không biết về sự tồn tại của nhiều thread trong process, nó không thể lập lịch các thread riêng biệt vào các bộ xử lý khác nhau trên phần cứng đa bộ xử lý. Cũng không thể gán cho một thread trong một process ưu tiên cao hơn một thread trong process khác một cách có ý nghĩa, vì việc lập lịch các thread được xử lý hoàn toàn trong process.

## **Triển khai One-to-one (1:1) (kernel-level thread)**

Trong một implementation threading 1:1, mỗi thread ánh xạ vào một KSE riêng biệt. Kernel xử lý lập lịch của mỗi thread riêng lẻ. Các hoạt động đồng bộ hóa thread được implement bằng cách sử dụng system call vào kernel.

Các implementation 1:1 loại bỏ các nhược điểm của các implementation M:1. Một system call bị block không khiến tất cả các thread trong process bị block, và kernel có thể lập lịch các thread của một process vào các CPU khác nhau trên phần cứng đa bộ xử lý.

Tuy nhiên, các hoạt động như tạo thread, chuyển ngữ cảnh, và đồng bộ hóa chậm hơn trên các implementation 1:1, vì cần phải chuyển sang kernel mode. Hơn nữa, chi phí cần thiết để duy trì một KSE riêng biệt cho mỗi thread trong một ứng dụng chứa nhiều thread có thể đặt ra tải đáng kể lên kernel scheduler, làm giảm hiệu suất tổng thể của hệ thống.

Mặc dù có những nhược điểm này, một implementation 1:1 thường được ưa thích hơn một implementation M:1. Cả hai implementation threading của Linux — LinuxThreads và NPTL — đều sử dụng mô hình 1:1.

> Trong quá trình phát triển NPTL, nỗ lực đáng kể đã được dành để viết lại kernel scheduler và xây dựng một implementation threading cho phép thực thi hiệu quả các process đa luồng chứa hàng nghìn thread. Các thử nghiệm sau đó cho thấy mục tiêu này đã đạt được.

## **Triển khai Many-to-many (M:N) (mô hình hai cấp)**

Các implementation M:N nhằm kết hợp các ưu điểm của mô hình 1:1 và M:1, đồng thời loại bỏ các nhược điểm của chúng.

Trong mô hình M:N, mỗi process có thể có nhiều KSE liên quan, và nhiều thread có thể ánh xạ vào mỗi KSE. Thiết kế này cho phép kernel phân phối các thread của ứng dụng trên nhiều CPU, đồng thời loại bỏ các vấn đề scalability có thể xảy ra với các ứng dụng sử dụng số lượng thread lớn.

Nhược điểm quan trọng nhất của mô hình M:N là sự phức tạp. Nhiệm vụ lập lịch thread được chia sẻ giữa kernel và thư viện threading user-space, phải hợp tác và trao đổi thông tin với nhau. Quản lý signal theo các yêu cầu của SUSv3 cũng phức tạp trong một implementation M:N.

> Một implementation M:N ban đầu đã được xem xét cho implementation threading NPTL, nhưng bị từ chối vì đòi hỏi các thay đổi kernel quá rộng và có thể không cần thiết, vì khả năng của Linux scheduler mở rộng tốt, ngay cả khi xử lý số lượng lớn KSE.

## **33.5 Các Implementation Linux của POSIX Threads**

Linux có hai implementation chính của Pthreads API:

-  LinuxThreads: Đây là implementation threading Linux gốc, được phát triển bởi Xavier Leroy.
-  NPTL (Native POSIX Threads Library): Đây là implementation threading Linux hiện đại, được phát triển bởi Ulrich Drepper và Ingo Molnar như là người kế nhiệm LinuxThreads. NPTL cung cấp hiệu suất vượt trội so với LinuxThreads, và tuân thủ chặt chẽ hơn với đặc tả SUSv3 cho Pthreads. Hỗ trợ cho NPTL đòi hỏi các thay đổi trong kernel, và những thay đổi này xuất hiện trong Linux 2.6.

Trong một thời gian, có vẻ như người kế nhiệm LinuxThreads sẽ là một implementation khác, được gọi là Next Generation POSIX Threads (NGPT), một implementation threading được phát triển tại IBM. NGPT sử dụng thiết kế M:N và hoạt động tốt hơn đáng kể so với LinuxThreads. Tuy nhiên, các nhà phát triển NPTL quyết định theo đuổi một implementation mới. Cách tiếp cận này đã được chứng minh — NPTL với thiết kế 1:1 cho thấy hiệu suất tốt hơn NGPT. Sau khi phát hành NPTL, việc phát triển NGPT đã bị ngừng lại.

Trong các phần tiếp theo, chúng ta xem xét thêm chi tiết về hai implementation này, và ghi chú các điểm mà chúng lệch khỏi các yêu cầu SUSv3 cho Pthreads.

Tại thời điểm này, đáng nhấn mạnh rằng implementation LinuxThreads hiện đã lỗi thời; nó không được hỗ trợ trong glibc 2.4 và các phiên bản sau. Tất cả các phát triển thư viện thread mới chỉ diễn ra trong NPTL.

## **33.5.1 LinuxThreads**

Trong nhiều năm, LinuxThreads là implementation threading chính trên Linux, và nó đủ để triển khai nhiều ứng dụng threaded. Các yếu tố cơ bản của implementation LinuxThreads như sau:

 Các thread được tạo bằng cách sử dụng lệnh gọi `clone()` chỉ định các flag sau:

```
CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND
```

Điều này có nghĩa là các thread LinuxThreads chia sẻ virtual memory, file descriptor, thông tin liên quan đến file system (umask, thư mục root, và thư mục làm việc hiện tại), và disposition của signal. Tuy nhiên, các thread không chia sẻ process ID và parent process ID.

-  Ngoài các thread được tạo bởi ứng dụng, LinuxThreads tạo ra một "manager" thread bổ sung xử lý việc tạo và kết thúc thread.
-  Implementation sử dụng signal cho hoạt động nội bộ của nó. Với các kernel hỗ trợ realtime signal (Linux 2.2 và sau), ba realtime signal đầu tiên được sử dụng. Với các kernel cũ hơn, `SIGUSR1` và `SIGUSR2` được sử dụng. Các ứng dụng không thể sử dụng các signal này. (Việc sử dụng signal dẫn đến độ trễ cao cho các hoạt động đồng bộ hóa thread khác nhau.)

## **Các lệch lạc của LinuxThreads so với hành vi được chỉ định**

LinuxThreads không tuân thủ đặc tả SUSv3 cho Pthreads trên một số điểm. (Implementation LinuxThreads bị ràng buộc bởi các tính năng kernel có sẵn tại thời điểm nó được phát triển; nó tuân thủ tối đa trong các ràng buộc đó.) Danh sách sau tóm tắt các điểm không tuân thủ:

-  Các lệnh gọi `getpid()` trả về một giá trị khác nhau trong mỗi thread của process. Các lệnh gọi `getppid()` phản ánh thực tế là mỗi thread ngoài thread chính được tạo bởi manager thread của process (tức là, `getppid()` trả về process ID của manager thread). Các lệnh gọi `getppid()` trong các thread khác phải trả về cùng giá trị với một lệnh gọi `getppid()` trong thread chính.
-  Nếu một thread tạo ra một tiến trình con bằng cách sử dụng `fork()`, thì bất kỳ thread nào khác phải có thể lấy trạng thái kết thúc của tiến trình con đó bằng cách sử dụng `wait()` (hoặc tương tự). Tuy nhiên, điều này không đúng; chỉ có thread đã tạo process con mới có thể `wait()` cho nó.
-  Nếu một thread gọi `exec()`, thì, như SUSv3 yêu cầu, tất cả các thread khác bị kết thúc. Tuy nhiên, nếu `exec()` được thực hiện từ bất kỳ thread nào khác ngoài thread chính, thì process kết quả sẽ có cùng process ID với thread đang gọi — tức là, một process ID khác với process ID của thread chính. Theo SUSv3, process ID phải giống với process ID của thread chính.
-  Các thread không chia sẻ credentials (user và group ID). Khi một process đa luồng đang thực thi một chương trình set-user-ID, điều này có thể dẫn đến các tình huống trong đó một thread không thể gửi signal đến thread khác bằng cách sử dụng `pthread_kill()`, vì credentials của hai thread đã được thay đổi theo cách mà thread gửi không còn có quyền báo hiệu thread đích (tham khảo Hình 20-2, trang 403). Hơn nữa, vì implementation LinuxThreads sử dụng signal nội bộ, nhiều hoạt động Pthreads có thể thất bại hoặc treo nếu một thread thay đổi credentials của nó.
-  Nhiều khía cạnh của đặc tả SUSv3 về sự tương tác giữa thread và signal không được tuân thủ:
  - Một signal được gửi đến một process bằng cách sử dụng `kill()` hoặc `sigqueue()` phải được gửi đến và xử lý bởi, một thread tùy ý trong process đích không block signal. Tuy nhiên, vì các thread LinuxThreads có process ID khác nhau, một signal chỉ có thể được nhắm mục tiêu đến một thread cụ thể. Nếu thread đó đang block signal, nó vẫn ở trạng thái chờ, ngay cả khi có các thread khác không block signal.

- LinuxThreads không hỗ trợ khái niệm signal đang chờ cho toàn bộ process; chỉ hỗ trợ per-thread pending signal.
- Nếu một signal được hướng đến một process group chứa một ứng dụng đa luồng, thì signal sẽ được xử lý bởi tất cả các thread trong ứng dụng (tức là, tất cả các thread đã thiết lập signal handler), thay vì bởi một thread đơn lẻ (tùy ý). Một signal như vậy có thể, ví dụ, được tạo ra bằng cách gõ một trong các ký tự terminal tạo ra job-control signal cho foreground process group.
- Cài đặt alternate signal stack (được thiết lập bởi `sigaltstack()`) là per-thread. Tuy nhiên, vì một thread mới không đúng kế thừa cài đặt alternate signal stack của nó từ người gọi `pthread_create()`, hai thread chia sẻ alternate signal stack. SUSv3 yêu cầu rằng một thread mới phải bắt đầu mà không có alternate signal stack được định nghĩa. Hậu quả của sự không tuân thủ LinuxThreads này là nếu hai thread xử lý đồng thời các signal khác nhau trên các alternate signal stack chia sẻ của chúng cùng một lúc, tình trạng hỗn loạn có thể xảy ra (ví dụ: crash chương trình). Vấn đề này có thể rất khó tái tạo và debug, vì sự xuất hiện của nó phụ thuộc vào sự kiện có thể hiếm xảy ra là hai signal được xử lý cùng một lúc.

Trong một chương trình sử dụng LinuxThreads, một thread mới có thể thực hiện một lệnh gọi `sigaltstack()` để đảm bảo rằng nó sử dụng alternate signal stack khác với thread đã tạo ra nó (hoặc không có stack). Tuy nhiên, các chương trình portable (và các hàm thư viện tạo thread) sẽ không biết để làm điều này, vì đây không phải là yêu cầu trên các implementation khác. Hơn nữa, ngay cả khi chúng ta sử dụng kỹ thuật này, vẫn có race condition có thể xảy ra: thread mới có thể nhận và xử lý một signal trên alternate stack trước khi nó có cơ hội gọi `sigaltstack()`.

-  Các thread không chia sẻ session ID và process group ID chung. Các system call `setsid()` và `setpgid()` không thể được sử dụng để thay đổi session hoặc tư cách thành viên process group của một process đa luồng.
-  Các record lock được thiết lập bằng cách sử dụng `fcntl()` không được chia sẻ. Các yêu cầu khóa chồng chéo cùng loại không được hợp nhất.
-  Các thread không chia sẻ giới hạn tài nguyên. SUSv3 quy định rằng giới hạn tài nguyên là các thuộc tính process-wide.
-  Thời gian CPU được trả về bởi `times()` và thông tin sử dụng tài nguyên được trả về bởi `getrusage()` là per-thread. Các system call này phải trả về tổng process-wide.
-  Một số phiên bản `ps(1)` hiển thị tất cả các thread trong một process (bao gồm manager thread) như các mục riêng biệt với process ID khác nhau.
-  Các thread không chia sẻ nice value được đặt bởi `setpriority()`.
-  Interval timer được tạo bằng cách sử dụng `setitimer()` không được chia sẻ giữa các thread.
-  Các thread không chia sẻ giá trị undo semaphore System V (`semadj`).

### **Các vấn đề khác với LinuxThreads**

Ngoài các lệch lạc so với SUSv3 ở trên, implementation LinuxThreads có các vấn đề sau:

-  Nếu manager thread bị kill, thì các thread còn lại phải được dọn dẹp thủ công.
-  Một core dump của một chương trình đa luồng có thể không bao gồm tất cả các thread của process (hoặc thậm chí là thread đã kích hoạt core dump).
-  Hoạt động `ioctl()` không chuẩn `TIOCNOTTY` chỉ có thể xóa liên kết của process với controlling terminal khi được gọi từ thread chính.

## **33.5.2 NPTL**

NPTL được thiết kế để giải quyết hầu hết các thiếu sót của LinuxThreads. Cụ thể:

-  NPTL cung cấp sự tuân thủ gần hơn nhiều với đặc tả SUSv3 cho Pthreads.
-  Các ứng dụng sử dụng số lượng thread lớn mở rộng tốt hơn nhiều dưới NPTL so với LinuxThreads.

NPTL cho phép một ứng dụng tạo số lượng thread lớn. Các nhà triển khai NPTL có thể chạy các chương trình thử nghiệm tạo 100.000 thread. Với LinuxThreads, giới hạn thực tế về số lượng thread là vài nghìn. (Thật ra, rất ít ứng dụng cần số lượng thread lớn như vậy.)

Công việc triển khai NPTL bắt đầu vào năm 2002 và tiến triển trong năm tiếp theo hoặc hơn. Song song đó, nhiều thay đổi đã được thực hiện trong kernel Linux để đáp ứng các yêu cầu của NPTL. Các thay đổi xuất hiện trong kernel Linux 2.6 để hỗ trợ NPTL bao gồm:

-  tinh chỉnh việc triển khai thread group (Phần 28.2.1);
-  bổ sung futex như một cơ chế đồng bộ hóa (futex là một cơ chế chung không chỉ được thiết kế cho NPTL);
-  bổ sung các system call mới (`get_thread_area()` và `set_thread_area()`) để hỗ trợ thread-local storage;
-  hỗ trợ threaded core dump và debugging của các process đa luồng;
-  sửa đổi để hỗ trợ quản lý signal theo cách nhất quán với mô hình Pthreads;
-  bổ sung system call `exit_group()` mới để kết thúc tất cả các thread trong một process (bắt đầu từ glibc 2.3, `_exit()` — và do đó cũng là hàm thư viện `exit()` — được đặt bí danh là wrapper gọi `exit_group()`, trong khi một lệnh gọi `pthread_exit()` gọi system call `_exit()` thực sự trong kernel, chỉ kết thúc thread đang gọi);
-  viết lại kernel scheduler để cho phép lập lịch hiệu quả số lượng rất lớn (tức là, hàng nghìn) KSE;

-  cải thiện hiệu suất cho code kết thúc process của kernel; và
-  mở rộng system call `clone()` (Phần 28.2).

Các yếu tố cơ bản của implementation NPTL như sau:

 Các thread được tạo bằng cách sử dụng lệnh gọi `clone()` chỉ định các flag sau:

```
CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND |
CLONE_THREAD | CLONE_SETTLS | CLONE_PARENT_SETTID |
CLONE_CHILD_CLEARTID | CLONE_SYSVSEM
```

Các thread NPTL chia sẻ tất cả thông tin mà các thread LinuxThreads chia sẻ, và nhiều hơn nữa. Flag `CLONE_THREAD` có nghĩa là một thread được đặt trong cùng thread group với người tạo ra nó và chia sẻ cùng process ID và parent process ID. Flag `CLONE_SYSVSEM` có nghĩa là một thread chia sẻ giá trị undo semaphore System V với người tạo ra nó.

> Khi chúng ta sử dụng `ps(1)` để liệt kê một process đa luồng đang chạy dưới NPTL, chúng ta chỉ thấy một dòng đầu ra. Để xem thông tin về các thread trong một process, chúng ta có thể sử dụng tùy chọn `ps –L`.

 Implementation sử dụng nội bộ hai realtime signal đầu tiên. Các ứng dụng không thể sử dụng các signal này.

> Một trong những signal này được sử dụng để triển khai thread cancellation. Signal kia được sử dụng như một phần của kỹ thuật đảm bảo rằng tất cả các thread trong một process có cùng user và group ID. Kỹ thuật này là cần thiết vì, ở cấp độ kernel, các thread có user và group credential riêng biệt. Do đó, implementation NPTL thực hiện một số công việc trong wrapper function cho mỗi system call thay đổi user và group ID (`setuid()`, `setresuid()`, v.v., và các tương tự group của chúng) khiến ID được thay đổi trong tất cả các thread của process.

 Không giống LinuxThreads, NPTL không sử dụng manager thread.

#### **Sự tuân thủ tiêu chuẩn của NPTL**

Những thay đổi này có nghĩa là NPTL đạt được sự tuân thủ SUSv3 gần hơn nhiều so với LinuxThreads. Tại thời điểm viết, sự không tuân thủ sau đây vẫn còn:

 Các thread không chia sẻ nice value.

Có một số điểm không tuân thủ NPTL bổ sung trong các kernel 2.6.x sớm hơn:

-  Trong các kernel trước 2.6.16, alternate signal stack là per-thread, nhưng một thread mới không đúng kế thừa cài đặt alternate signal stack (được thiết lập bởi `sigaltstack()`) từ người gọi `pthread_create()`, với hậu quả là hai thread chia sẻ alternate signal stack.
-  Trong các kernel trước 2.6.16, chỉ có thread group leader (tức là, thread chính) mới có thể bắt đầu một session mới bằng cách gọi `setsid()`.
-  Trong các kernel trước 2.6.16, chỉ có thread group leader mới có thể sử dụng `setpgid()` để làm cho host process trở thành process group leader.

-  Trong các kernel trước 2.6.12, interval timer được tạo bằng cách sử dụng `setitimer()` không được chia sẻ giữa các thread của một process.
-  Trong các kernel trước 2.6.10, cài đặt giới hạn tài nguyên không được chia sẻ giữa các thread của một process.
-  Trong các kernel trước 2.6.9, thời gian CPU được trả về bởi `times()` và thông tin sử dụng tài nguyên được trả về bởi `getrusage()` là per-thread.

NPTL được thiết kế để tương thích ABI với LinuxThreads. Điều này có nghĩa là các chương trình được liên kết với thư viện GNU C cung cấp LinuxThreads không cần được liên kết lại để sử dụng NPTL. Tuy nhiên, một số hành vi có thể thay đổi khi chương trình được chạy với NPTL, chủ yếu là vì NPTL tuân thủ chặt chẽ hơn với đặc tả SUSv3 cho Pthreads.

## **33.5.3 Implementation Threading Nào?**

Một số phân phối Linux được cung cấp với thư viện GNU C cung cấp cả LinuxThreads và NPTL, với mặc định được xác định bởi dynamic linker dựa trên kernel nền tảng mà hệ thống đang chạy. (Các phân phối này hiện đã là lịch sử vì, kể từ phiên bản 2.4, glibc không còn cung cấp LinuxThreads.) Do đó, đôi khi chúng ta có thể cần trả lời các câu hỏi sau:

-  Implementation threading nào có sẵn trong một phân phối Linux cụ thể?
-  Trên một phân phối Linux cung cấp cả LinuxThreads và NPTL, implementation nào được sử dụng mặc định, và làm thế nào chúng ta có thể chọn rõ ràng implementation được sử dụng bởi một chương trình?

## **Khám phá implementation threading**

Chúng ta có thể sử dụng một vài kỹ thuật để khám phá implementation threading có sẵn trên một hệ thống cụ thể, hoặc để khám phá implementation mặc định sẽ được sử dụng khi một chương trình được chạy trên một hệ thống cung cấp cả hai implementation threading.

Trên một hệ thống cung cấp glibc phiên bản 2.3.2 hoặc sau, chúng ta có thể sử dụng lệnh sau để khám phá implementation threading mà hệ thống cung cấp, hoặc, nếu nó cung cấp cả hai implementation, thì cái nào được sử dụng mặc định:

#### \$ **getconf GNU\_LIBPTHREAD\_VERSION**

Trên một hệ thống nơi NPTL là implementation duy nhất hoặc mặc định, điều này sẽ hiển thị một chuỗi như sau:

NPTL 2.3.4

Kể từ glibc 2.3.2, một chương trình có thể lấy thông tin tương tự bằng cách sử dụng `confstr(3)` để truy xuất giá trị của biến cấu hình `_CS_GNU_LIBPTHREAD_VERSION` đặc thù của glibc.

Trên các hệ thống có thư viện GNU C cũ hơn, chúng ta phải làm thêm một ít việc. Đầu tiên, lệnh sau có thể được sử dụng để hiển thị pathname của thư viện GNU C được sử dụng khi chúng ta chạy một chương trình (ở đây, chúng ta sử dụng ví dụ về chương trình `ls` tiêu chuẩn, nằm tại `/bin/ls`):

```
$ ldd /bin/ls | grep libc.so
 libc.so.6 => /lib/tls/libc.so.6 (0x40050000)
```

Chúng ta sẽ nói thêm về chương trình `ldd` (liệt kê các phụ thuộc động) trong Phần 41.5.

Pathname của thư viện GNU C được hiển thị sau `=>`. Nếu chúng ta thực thi pathname này như một lệnh, thì glibc hiển thị một loạt thông tin về chính nó. Chúng ta có thể grep qua thông tin này để chọn dòng hiển thị implementation threading:

```
$ /lib/tls/libc.so.6 | egrep -i 'threads|nptl'
 Native POSIX Threads Library by Ulrich Drepper et al
```

Chúng ta bao gồm `nptl` trong biểu thức chính quy `egrep` vì một số bản phát hành glibc chứa NPTL hiển thị chuỗi như sau:

```
 NPTL 0.61 by Ulrich Drepper
```

Vì pathname glibc có thể thay đổi từ phân phối Linux này sang phân phối khác, chúng ta có thể sử dụng shell command substitution để tạo ra một dòng lệnh sẽ hiển thị implementation threading đang sử dụng trên bất kỳ hệ thống Linux nào, như sau:

```
$ $(ldd /bin/ls | grep libc.so | awk '{print $3}') | egrep -i 'threads|nptl'
 Native POSIX Threads Library by Ulrich Drepper et al
```

#### **Chọn implementation threading được sử dụng bởi một chương trình**

Trên một hệ thống Linux cung cấp cả NPTL và LinuxThreads, đôi khi hữu ích khi có thể kiểm soát rõ ràng implementation threading nào được sử dụng. Ví dụ phổ biến nhất về yêu cầu này là khi chúng ta có một chương trình cũ hơn phụ thuộc vào một số hành vi (có thể không chuẩn) của LinuxThreads, vì vậy chúng ta muốn buộc chương trình chạy với implementation threading đó, thay vì NPTL mặc định.

Vì mục đích này, chúng ta có thể sử dụng một biến môi trường đặc biệt được hiểu bởi dynamic linker: `LD_ASSUME_KERNEL`. Như tên gợi ý, biến môi trường này cho dynamic linker biết để hoạt động như thể nó đang chạy trên đỉnh của một phiên bản kernel Linux cụ thể. Bằng cách chỉ định một phiên bản kernel không cung cấp hỗ trợ cho NPTL (ví dụ: 2.2.5), chúng ta có thể đảm bảo rằng LinuxThreads được sử dụng. Do đó, chúng ta có thể chạy một chương trình đa luồng với LinuxThreads bằng lệnh sau:

```
$ LD_ASSUME_KERNEL=2.2.5 ./prog
```

Khi chúng ta kết hợp cài đặt biến môi trường này với lệnh chúng ta đã mô tả trước đó để hiển thị implementation threading đang được sử dụng, chúng ta thấy điều gì đó như sau:

```
$ export LD_ASSUME_KERNEL=2.2.5
$ $(ldd /bin/ls | grep libc.so | awk '{print $3}') | egrep -i 'threads|nptl'
 linuxthreads-0.10 by Xavier Leroy
```

Phạm vi của các số phiên bản kernel có thể được chỉ định trong `LD_ASSUME_KERNEL` phải tuân theo một số giới hạn. Trong một số phân phối phổ biến cung cấp cả NPTL và LinuxThreads, việc chỉ định số phiên bản là 2.2.5 là đủ để đảm bảo việc sử dụng LinuxThreads. Để có mô tả đầy đủ hơn về việc sử dụng biến môi trường này, xem http://people.redhat.com/drepper/assumekernel.html.

## **33.6 Các Tính Năng Nâng Cao của Pthreads API**

Một số tính năng nâng cao của Pthreads API bao gồm:

-  Lập lịch realtime: Chúng ta có thể đặt chính sách và ưu tiên lập lịch realtime cho các thread. Điều này tương tự với các system call lập lịch realtime cho process được mô tả trong Phần 35.3.
-  Mutex và condition variable được chia sẻ giữa process: SUSv3 quy định một tùy chọn để cho phép mutex và condition variable được chia sẻ giữa các process (thay vì chỉ giữa các thread của một process duy nhất). Trong trường hợp này, condition variable hoặc mutex phải được đặt trong một vùng bộ nhớ được chia sẻ giữa các process. NPTL hỗ trợ tính năng này.
-  Các nguyên hàm đồng bộ hóa thread nâng cao: Các tiện ích này bao gồm barrier, read-write lock, và spin lock.

Thêm chi tiết về tất cả các tính năng này có thể được tìm thấy trong [Butenhof, 1996].

## **33.7 Tổng Kết**

Thread không kết hợp tốt với signal; các thiết kế ứng dụng đa luồng nên tránh sử dụng signal bất cứ khi nào có thể. Nếu một ứng dụng đa luồng phải xử lý signal không đồng bộ, thường cách sạch nhất để làm vậy là block signal trong tất cả các thread, và có một thread chuyên dụng duy nhất chấp nhận signal đến bằng cách sử dụng `sigwait()` (hoặc tương tự). Thread này sau đó có thể an toàn thực hiện các tác vụ như sửa đổi các biến chia sẻ (dưới kiểm soát mutex) và gọi các hàm không an toàn với async-signal.

Hai implementation threading thường có sẵn trên Linux: LinuxThreads và NPTL. LinuxThreads đã có sẵn trên Linux nhiều năm, nhưng có một số điểm không tuân thủ các yêu cầu SUSv3 và nó hiện đã lỗi thời. Implementation NPTL mới hơn cung cấp sự tuân thủ SUSv3 gần hơn và hiệu suất vượt trội, và là implementation được cung cấp trong các phân phối Linux hiện đại.

#### **Thông tin thêm**

Tham khảo các nguồn thông tin thêm được liệt kê trong Phần 29.10.

Tác giả của LinuxThreads đã ghi lại implementation trong một trang web có thể tìm thấy tại http://pauillac.inria.fr/~xleroy/linuxthreads/. Implementation NPTL được mô tả bởi các nhà triển khai trong một bài báo (hiện đã có phần lỗi thời) có sẵn trực tuyến tại http://people.redhat.com/drepper/nptl-design.pdf.

## **33.8 Bài Tập**

- **33-1.** Viết một chương trình để chứng minh rằng các thread khác nhau trong cùng một process có thể có các tập pending signal khác nhau, như được trả về bởi `sigpending()`. Bạn có thể làm điều này bằng cách sử dụng `pthread_kill()` để gửi các signal khác nhau đến hai thread khác nhau đã block các signal này, và sau đó để mỗi thread gọi `sigpending()` và hiển thị thông tin về pending signal. (Bạn có thể thấy các hàm trong Listing 20-4 hữu ích.)
- **33-2.** Giả sử một thread tạo ra một tiến trình con bằng cách sử dụng `fork()`. Khi tiến trình con kết thúc, liệu signal `SIGCHLD` kết quả có được đảm bảo gửi đến thread đã gọi `fork()` không (so với một thread khác trong process)?
