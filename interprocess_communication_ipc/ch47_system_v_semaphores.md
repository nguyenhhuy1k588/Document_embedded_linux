## Chương 47
# **SEMAPHORE SYSTEM V**

Chương này mô tả semaphore System V. Không giống như các cơ chế IPC được mô tả trong các chương trước, semaphore System V không được sử dụng để truyền dữ liệu giữa các process. Thay vào đó, chúng cho phép các process đồng bộ hóa các hành động của mình. Một cách sử dụng phổ biến của semaphore là đồng bộ hóa quyền truy cập vào một khối shared memory, để ngăn một process truy cập shared memory trong khi một process khác đang cập nhật nó.

Một semaphore là một số nguyên được kernel duy trì, có giá trị bị giới hạn lớn hơn hoặc bằng 0. Có thể thực hiện nhiều thao tác (tức là system call) trên một semaphore, bao gồm:

- đặt semaphore thành một giá trị tuyệt đối;
- thêm một số vào giá trị hiện tại của semaphore;
- trừ một số khỏi giá trị hiện tại của semaphore; và
- chờ cho giá trị semaphore bằng 0.

Hai thao tác cuối trong số này có thể khiến process gọi bị block. Khi giảm giá trị semaphore, kernel block bất kỳ nỗ lực nào giảm giá trị xuống dưới 0. Tương tự, chờ một semaphore bằng 0 sẽ block process gọi nếu giá trị semaphore hiện tại không phải là 0. Trong cả hai trường hợp, process gọi vẫn bị block cho đến khi một process khác thay đổi semaphore thành một giá trị cho phép thao tác tiếp tục, tại điểm đó kernel đánh thức process bị block. Hình 47-1 cho thấy việc sử dụng semaphore để đồng bộ hóa các hành động của hai process luân phiên di chuyển giá trị semaphore giữa 0 và 1.

![](_page_89_Figure_1.jpeg)

**Hình 47-1:** Sử dụng semaphore để đồng bộ hóa hai process

Về mặt kiểm soát các hành động của một process, một semaphore không có ý nghĩa gì trong bản thân nó. Ý nghĩa của nó chỉ được xác định bởi các liên kết được cung cấp cho nó bởi các process sử dụng semaphore. Thông thường, các process đồng ý về một quy ước liên kết semaphore với một tài nguyên chia sẻ, chẳng hạn như một vùng của shared memory. Các cách sử dụng semaphore khác cũng có thể, chẳng hạn như đồng bộ hóa giữa process cha và con sau `fork()`. (Trong Mục 24.5, chúng ta đã xem xét việc sử dụng signal để thực hiện cùng một nhiệm vụ.)

## **47.1 Tổng Quan**

Các bước chung để sử dụng một semaphore System V như sau:

- Tạo hoặc mở một semaphore set bằng `semget()`.
- Khởi tạo các semaphore trong set bằng thao tác `SETVAL` hoặc `SETALL` của `semctl()`. (Chỉ một process nên làm điều này.)
- Thực hiện các thao tác trên giá trị semaphore bằng `semop()`. Các process sử dụng semaphore thường sử dụng các thao tác này để chỉ ra việc thu nhận và giải phóng một tài nguyên chia sẻ.
- Khi tất cả các process đã sử dụng xong semaphore set, hãy xóa set bằng thao tác `IPC_RMID` của `semctl()`. (Chỉ một process nên làm điều này.)

Hầu hết các hệ điều hành cung cấp một loại nguyên tắc semaphore nào đó để sử dụng trong các chương trình ứng dụng. Tuy nhiên, semaphore System V được làm phức tạp một cách bất thường bởi thực tế là chúng được phân bổ theo nhóm gọi là semaphore set. Số lượng semaphore trong một set được chỉ định khi set được tạo bằng system call `semget()`. Mặc dù thông thường chỉ thao tác trên một semaphore tại một thời điểm, system call `semop()` cho phép chúng ta thực hiện nguyên tử một nhóm thao tác trên nhiều semaphore trong cùng một set.

Vì semaphore System V được tạo và khởi tạo trong các bước riêng biệt, race condition có thể xảy ra nếu hai process cố gắng thực hiện các bước này cùng lúc. Mô tả race condition này và cách tránh nó đòi hỏi phải mô tả `semctl()` trước khi mô tả `semop()`, điều này có nghĩa là có khá nhiều tài liệu cần đề cập trước khi chúng ta có tất cả các chi tiết cần thiết để hiểu đầy đủ về semaphore.

Trong thời gian này, chúng ta cung cấp Listing 47-1 như một ví dụ đơn giản về việc sử dụng các system call semaphore khác nhau. Chương trình này hoạt động theo hai chế độ:

- Được cho một đối số dòng lệnh số nguyên duy nhất, chương trình tạo một semaphore set mới chứa một semaphore duy nhất, và khởi tạo semaphore thành giá trị được cung cấp trong đối số dòng lệnh. Chương trình hiển thị identifier của semaphore set mới.
- Được cho hai đối số dòng lệnh, chương trình giải thích chúng là (theo thứ tự) identifier của một semaphore set hiện có và một giá trị cần thêm vào semaphore đầu tiên (được đánh số 0) trong set đó. Chương trình thực hiện thao tác đã chỉ định trên semaphore đó. Để cho phép chúng ta giám sát thao tác semaphore, chương trình in các message trước và sau thao tác. Mỗi message này bắt đầu bằng process ID, để chúng ta có thể phân biệt đầu ra của nhiều instance của chương trình.

Phiên shell log sau minh họa việc sử dụng chương trình trong Listing 47-1. Chúng ta bắt đầu bằng cách tạo một semaphore được khởi tạo thành 0:

```
$ ./svsem_demo 0
Semaphore ID = 98307 ID of new semaphore set
```

Sau đó chúng ta thực hiện một lệnh nền cố gắng giảm giá trị semaphore đi 2:

```
$ ./svsem_demo 98307 -2 &
23338: about to semop at 10:19:42
[1] 23338
```

Lệnh này bị block, vì giá trị của semaphore không thể giảm xuống dưới 0. Bây giờ, chúng ta thực hiện một lệnh thêm 3 vào giá trị semaphore:

```
$ ./svsem_demo 98307 +3
23339: about to semop at 10:19:55
23339: semop completed at 10:19:55
23338: semop completed at 10:19:55
[1]+ Done ./svsem_demo 98307 -2
```

Thao tác tăng semaphore thành công ngay lập tức, và khiến thao tác giảm semaphore trong lệnh nền tiếp tục, vì thao tác đó giờ đây có thể được thực hiện mà không để giá trị semaphore xuống dưới 0.

**Listing 47-1:** Tạo và thao tác trên semaphore System V

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_demo.c
#include <sys/types.h>
#include <sys/sem.h>
#include <sys/stat.h>
#include "curr_time.h" /* Declaration of currTime() */
#include "semun.h" /* Definition of semun union */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int semid;
 if (argc < 2 || argc > 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s init-value\n"
 " or: %s semid operation\n", argv[0], argv[0]);
 if (argc == 2) { /* Create and initialize semaphore */
 union semun arg;
 semid = semget(IPC_PRIVATE, 1, S_IRUSR | S_IWUSR);
 if (semid == -1)
 errExit("semid");
 arg.val = getInt(argv[1], 0, "init-value");
 if (semctl(semid, /* semnum= */ 0, SETVAL, arg) == -1)
 errExit("semctl");
 printf("Semaphore ID = %d\n", semid);
 } else { /* Perform an operation on first semaphore */
 struct sembuf sop; /* Structure defining operation */
 semid = getInt(argv[1], 0, "semid");
 sop.sem_num = 0; /* Specifies first semaphore in set */
 sop.sem_op = getInt(argv[2], 0, "operation");
 /* Add, subtract, or wait for 0 */
 sop.sem_flg = 0; /* No special options for operation */
 printf("%ld: about to semop at %s\n", (long) getpid(), currTime("%T"));
 if (semop(semid, &sop, 1) == -1)
 errExit("semop");
 printf("%ld: semop completed at %s\n", (long) getpid(), currTime("%T"));
 }
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_demo.c
```

## **47.2 Tạo hoặc Mở một Semaphore Set**

System call `semget()` tạo một semaphore set mới hoặc lấy identifier của một set hiện có.

```
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
int semget(key_t key, int nsems, int semflg);
                    Returns semaphore set identifier on success, or –1 on error
```

Đối số `key` là một key được tạo bằng một trong các phương pháp mô tả trong Mục 45.2 (tức là thường là giá trị `IPC_PRIVATE` hoặc một key được trả về bởi `ftok()`).

Nếu chúng ta đang sử dụng `semget()` để tạo một semaphore set mới, thì `nsems` chỉ định số lượng semaphore trong set đó, và phải lớn hơn 0. Nếu chúng ta đang sử dụng `semget()` để lấy identifier của một set hiện có, thì `nsems` phải nhỏ hơn hoặc bằng kích thước của set (nếu không thì lỗi `EINVAL` xảy ra). Không thể thay đổi số lượng semaphore trong một set hiện có.

Đối số `semflg` là một bitmask chỉ định các quyền sẽ được áp dụng lên một semaphore set mới hoặc được kiểm tra đối với một set hiện có. Các quyền này được chỉ định theo cách tương tự như đối với file (Table 15-4, trang 295). Ngoài ra, không hoặc nhiều flag sau đây có thể được OR (`|`) vào `semflg` để điều khiển hoạt động của `semget()`:

`IPC_CREAT`

Nếu không có semaphore set nào với key đã chỉ định tồn tại, hãy tạo một set mới.

`IPC_EXCL`

Nếu `IPC_CREAT` cũng được chỉ định, và một semaphore set với key đã chỉ định đã tồn tại, hãy thất bại với lỗi `EEXIST`.

Các flag này được mô tả chi tiết hơn trong Mục 45.1.

Khi thành công, `semget()` trả về identifier cho semaphore set mới hoặc hiện có. Các system call tiếp theo tham chiếu đến các semaphore riêng lẻ phải chỉ định cả identifier của semaphore set lẫn số của semaphore trong set đó. Các semaphore trong một set được đánh số bắt đầu từ 0.

## **47.3 Các Thao Tác Điều Khiển Semaphore**

System call `semctl()` thực hiện nhiều thao tác điều khiển khác nhau trên một semaphore set hoặc trên một semaphore riêng lẻ trong một set.

```
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
int semctl(int semid, int semnum, int cmd, ... /* union semun arg */);
         Returns nonnegative integer on success (see text); returns –1 on error
```

Đối số `semid` là identifier của semaphore set mà thao tác cần được thực hiện. Đối với những thao tác được thực hiện trên một semaphore duy nhất, đối số `semnum` xác định một semaphore cụ thể trong set. Đối với các thao tác khác, đối số này bị bỏ qua, và chúng ta có thể chỉ định nó là 0. Đối số `cmd` chỉ định thao tác cần thực hiện.

Một số thao tác nhất định đòi hỏi một đối số thứ tư cho `semctl()`, mà chúng ta gọi là `arg` trong phần còn lại của mục này. Đối số này là một union được định nghĩa như được hiển thị trong Listing 47-2. Chúng ta phải định nghĩa union này một cách tường minh trong các chương trình của mình. Chúng ta thực hiện điều này trong các chương trình ví dụ bằng cách include file header trong Listing 47-2.

> Mặc dù việc đặt định nghĩa của `semun` union trong một file header chuẩn sẽ hợp lý, SUSv3 yêu cầu lập trình viên định nghĩa nó một cách tường minh. Tuy nhiên, một số hệ thống UNIX cung cấp định nghĩa này trong `<sys/sem.h>`. Các phiên bản cũ hơn của glibc (lên đến và bao gồm phiên bản 2.0) cũng cung cấp định nghĩa này. Theo SUSv3, các phiên bản glibc gần đây hơn thì không, và macro `_SEM_SEMUN_UNDEFINED` được định nghĩa với giá trị 1 trong `<sys/sem.h>` để chỉ ra điều này (tức là một ứng dụng được biên dịch với glibc có thể kiểm tra macro này để xác định liệu chương trình có phải tự định nghĩa `semun` union hay không).

**Listing 47-2:** Định nghĩa của `semun` union

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––––svsem/semun.h
#ifndef SEMUN_H
#define SEMUN_H /* Prevent accidental double inclusion */
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
union semun { /* Used in calls to semctl() */
 int val;
 struct semid_ds * buf;
 unsigned short * array;
#if defined(__linux__)
 struct seminfo * __buf;
#endif
};
#endif
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––––svsem/semun.h
```

SUSv2 và SUSv3 chỉ định rằng đối số cuối cùng của `semctl()` là tùy chọn. Tuy nhiên, một vài hệ thống UNIX (chủ yếu là cũ hơn) (và các phiên bản cũ hơn của glibc) đã khai báo nguyên mẫu `semctl()` như sau:

```
int semctl(int semid, int semnum, int cmd, union semun arg);
```

Điều này có nghĩa là đối số thứ tư là bắt buộc ngay cả trong những trường hợp nó không thực sự được sử dụng (ví dụ: các thao tác `IPC_RMID` và `GETVAL` được mô tả bên dưới). Để có tính di động đầy đủ, chúng ta chỉ định một đối số giả cuối cùng cho `semctl()` trong những lời gọi không cần thiết.

Trong phần còn lại của mục này, chúng ta xem xét các thao tác điều khiển khác nhau có thể được chỉ định cho `cmd`.

### **Các thao tác điều khiển tổng quát**

Các thao tác sau là những thao tác tương tự có thể được áp dụng cho các loại đối tượng IPC System V khác. Trong mỗi trường hợp, đối số `semnum` bị bỏ qua. Chi tiết thêm về các thao tác này, bao gồm các đặc quyền và quyền hạn được yêu cầu bởi process gọi, được cung cấp trong Mục 45.3.

`IPC_RMID`

Ngay lập tức xóa semaphore set và cấu trúc dữ liệu `semid_ds` liên kết. Bất kỳ process nào đang bị block trong các lời gọi `semop()` đang chờ trên các semaphore trong set này sẽ được đánh thức ngay lập tức, với `semop()` báo cáo lỗi `EIDRM`. Đối số `arg` không bắt buộc.

`IPC_STAT`

Đặt một bản sao của cấu trúc dữ liệu `semid_ds` liên kết với semaphore set này vào buffer được trỏ bởi `arg.buf`. Chúng ta mô tả cấu trúc `semid_ds` trong Mục 47.4.

`IPC_SET`

Cập nhật các trường được chọn của cấu trúc dữ liệu `semid_ds` liên kết với semaphore set này bằng cách sử dụng các giá trị trong buffer được trỏ bởi `arg.buf`.

### **Lấy và khởi tạo giá trị semaphore**

Các thao tác sau lấy hoặc khởi tạo giá trị của một semaphore riêng lẻ hoặc tất cả semaphore trong một set. Việc lấy giá trị semaphore đòi hỏi quyền đọc trên semaphore, trong khi khởi tạo giá trị đòi hỏi quyền thay đổi (ghi).

`GETVAL`

Là kết quả hàm của nó, `semctl()` trả về giá trị của semaphore thứ `semnum` trong semaphore set được chỉ định bởi `semid`. Đối số `arg` không bắt buộc.

`SETVAL`

Giá trị của semaphore thứ `semnum` trong set được tham chiếu bởi `semid` được khởi tạo thành giá trị được chỉ định trong `arg.val`.

`GETALL`

Lấy các giá trị của tất cả các semaphore trong set được tham chiếu bởi `semid`, đặt chúng vào mảng được trỏ bởi `arg.array`. Lập trình viên phải đảm bảo rằng mảng này có kích thước đủ. (Số lượng semaphore trong một set có thể lấy được từ trường `sem_nsems` của cấu trúc dữ liệu `semid_ds` được lấy bởi thao tác `IPC_STAT`.) Đối số `semnum` bị bỏ qua. Một ví dụ về việc sử dụng thao tác `GETALL` được cung cấp trong Listing 47-3.

`SETALL`

Khởi tạo tất cả các semaphore trong set được tham chiếu bởi `semid`, sử dụng các giá trị được cung cấp trong mảng được trỏ bởi `arg.array`. Đối số `semnum` bị bỏ qua. Listing 47-4 minh họa việc sử dụng thao tác `SETALL`.

Nếu một process khác đang chờ thực hiện một thao tác trên các semaphore được sửa đổi bởi các thao tác `SETVAL` hoặc `SETALL`, và các thay đổi được thực hiện sẽ cho phép thao tác đó tiếp tục, thì kernel sẽ đánh thức process đó.

Việc thay đổi giá trị của một semaphore với `SETVAL` hoặc `SETALL` sẽ xóa các undo entry cho semaphore đó trong tất cả các process. Chúng ta mô tả semaphore undo entry trong Mục 47.8.

Lưu ý rằng thông tin được trả về bởi `GETVAL` và `GETALL` có thể đã lỗi thời vào thời điểm process gọi đến sử dụng nó. Bất kỳ chương trình nào phụ thuộc vào thông tin được trả về bởi các thao tác này không thay đổi có thể chịu tác động của race condition time-of-check, time-of-use (Mục 38.6).

## **Lấy thông tin per-semaphore**

Các thao tác sau trả về (qua giá trị kết quả hàm) thông tin về semaphore thứ `semnum` của set được tham chiếu bởi `semid`. Đối với tất cả các thao tác này, cần có quyền đọc trên semaphore set, và đối số `arg` không bắt buộc.

`GETPID`

Trả về process ID của process cuối cùng thực hiện một `semop()` trên semaphore này; đây được gọi là giá trị `sempid`. Nếu không có process nào đã thực hiện `semop()` trên semaphore này, 0 được trả về.

`GETNCNT`

Trả về số process hiện đang chờ giá trị của semaphore này tăng lên; đây được gọi là giá trị `semncnt`.

`GETZCNT`

Trả về số process hiện đang chờ giá trị của semaphore này trở thành 0; đây được gọi là giá trị `semzcnt`.

Như với các thao tác `GETVAL` và `GETALL` được mô tả ở trên, thông tin được trả về bởi các thao tác `GETPID`, `GETNCNT` và `GETZCNT` có thể đã lỗi thời vào thời điểm process gọi đến sử dụng nó.

Listing 47-3 minh họa việc sử dụng ba thao tác này.

# **47.4 Cấu Trúc Dữ Liệu Liên Kết Với Semaphore**

Mỗi semaphore set có một cấu trúc dữ liệu `semid_ds` liên kết với dạng sau:

```
struct semid_ds {
 struct ipc_perm sem_perm; /* Ownership and permissions */
 time_t sem_otime; /* Time of last semop() */
 time_t sem_ctime; /* Time of last change */
 unsigned long sem_nsems; /* Number of semaphores in set */
};
```

SUSv3 yêu cầu tất cả các trường mà chúng ta hiển thị trong cấu trúc `semid_ds`. Một số hệ thống UNIX khác bao gồm các trường không chuẩn bổ sung. Trên Linux 2.4 và mới hơn, trường `sem_nsems` được gõ kiểu là `unsigned long`. SUSv3 chỉ định kiểu của trường này là `unsigned short`, và nó được định nghĩa như vậy trong Linux 2.2 và trên hầu hết các hệ thống UNIX khác.

Các trường của cấu trúc `semid_ds` được cập nhật ngầm định bởi các system call semaphore khác nhau, và một số subfield của trường `sem_perm` có thể được cập nhật tường minh bằng thao tác `IPC_SET` của `semctl()`. Chi tiết như sau:

#### `sem_perm`

Khi semaphore set được tạo, các trường của substructure này được khởi tạo như mô tả trong Mục 45.3. Các subfield `uid`, `gid` và `mode` có thể được cập nhật thông qua `IPC_SET`.

`sem_otime`

Trường này được đặt thành 0 khi semaphore set được tạo, và sau đó được đặt thành thời gian hiện tại sau mỗi `semop()` thành công, hoặc khi giá trị semaphore được sửa đổi do thao tác `SEM_UNDO` (Mục 47.8). Trường này và `sem_ctime` được gõ kiểu là `time_t`, và lưu thời gian tính bằng giây kể từ Epoch.

`sem_ctime`

Trường này được đặt thành thời gian hiện tại khi semaphore set được tạo và sau mỗi thao tác `IPC_SET`, `SETALL` hoặc `SETVAL` thành công. (Trên một số hệ thống UNIX, các thao tác `SETALL` và `SETVAL` không sửa đổi `sem_ctime`.)

`sem_nsems`

Khi set được tạo, trường này được khởi tạo thành số lượng semaphore trong set.

Trong phần còn lại của mục này, chúng ta hiển thị hai chương trình ví dụ sử dụng cấu trúc dữ liệu `semid_ds` và một số thao tác `semctl()` được mô tả trong Mục 47.3. Chúng ta minh họa việc sử dụng cả hai chương trình này trong Mục 47.6.

## **Giám sát một semaphore set**

Chương trình trong Listing 47-3 sử dụng nhiều thao tác `semctl()` khác nhau để hiển thị thông tin về semaphore set hiện có có identifier được cung cấp làm đối số dòng lệnh của nó. Chương trình đầu tiên hiển thị các trường thời gian từ cấu trúc dữ liệu `semid_ds`. Sau đó, đối với mỗi semaphore trong set, chương trình hiển thị giá trị hiện tại của semaphore, cũng như các giá trị `sempid`, `semncnt` và `semzcnt` của nó.

**Listing 47-3:** Một chương trình giám sát semaphore

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_mon.c
#include <sys/types.h>
#include <sys/sem.h>
#include <time.h>
#include "semun.h" /* Definition of semun union */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 struct semid_ds ds;
 union semun arg, dummy; /* Fourth argument for semctl() */
 int semid, j;
```

```
 if (argc != 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s semid\n", argv[0]);
 semid = getInt(argv[1], 0, "semid");
 arg.buf = &ds;
 if (semctl(semid, 0, IPC_STAT, arg) == -1)
 errExit("semctl");
 printf("Semaphore changed: %s", ctime(&ds.sem_ctime));
 printf("Last semop(): %s", ctime(&ds.sem_otime));
 /* Display per-semaphore information */
 arg.array = calloc(ds.sem_nsems, sizeof(arg.array[0]));
 if (arg.array == NULL)
 errExit("calloc");
 if (semctl(semid, 0, GETALL, arg) == -1)
 errExit("semctl-GETALL");
 printf("Sem # Value SEMPID SEMNCNT SEMZCNT\n");
 for (j = 0; j < ds.sem_nsems; j++)
 printf("%3d %5d %5d %5d %5d\n", j, arg.array[j],
 semctl(semid, j, GETPID, dummy),
 semctl(semid, j, GETNCNT, dummy),
 semctl(semid, j, GETZCNT, dummy));
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_mon.c
```

### **Khởi tạo tất cả semaphore trong một set**

Chương trình trong Listing 47-4 cung cấp một giao diện dòng lệnh để khởi tạo tất cả các semaphore trong một set hiện có. Đối số dòng lệnh đầu tiên là identifier của semaphore set cần khởi tạo. Các đối số dòng lệnh còn lại chỉ định các giá trị mà semaphore sẽ được khởi tạo (phải có nhiều đối số này bằng số lượng semaphore trong set).

**Listing 47-4:** Sử dụng thao tác `SETALL` để khởi tạo một semaphore set System V

```
––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_setall.c
#include <sys/types.h>
#include <sys/sem.h>
#include "semun.h" /* Definition of semun union */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 struct semid_ds ds;
 union semun arg; /* Fourth argument for semctl() */
 int j, semid;
```

```
 if (argc < 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s semid val...\n", argv[0]);
 semid = getInt(argv[1], 0, "semid");
   /* Obtain size of semaphore set */
 arg.buf = &ds;
 if (semctl(semid, 0, IPC_STAT, arg) == -1)
 errExit("semctl");
 if (ds.sem_nsems != argc - 2)
 cmdLineErr("Set contains %ld semaphores, but %d values were supplied\n",
 (long) ds.sem_nsems, argc - 2);
 /* Set up array of values; perform semaphore initialization */
 arg.array = calloc(ds.sem_nsems, sizeof(arg.array[0]));
 if (arg.array == NULL)
 errExit("calloc");
 for (j = 2; j < argc; j++)
 arg.array[j - 2] = getInt(argv[j], 0, "val");
 if (semctl(semid, 0, SETALL, arg) == -1)
 errExit("semctl-SETALL");
 printf("Semaphore values changed (PID=%ld)\n", (long) getpid());
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_setall.c
```

# **47.5 Khởi Tạo Semaphore**

Theo SUSv3, một hệ thống không bắt buộc phải khởi tạo các giá trị của các semaphore trong một set được tạo bởi `semget()`. Thay vào đó, lập trình viên phải khởi tạo rõ ràng semaphore bằng system call `semctl()`. (Trên Linux, các semaphore được trả về bởi `semget()` thực ra được khởi tạo thành 0, nhưng chúng ta không thể dựa vào điều này một cách di động.) Như đã nêu trước đó, thực tế là việc tạo và khởi tạo semaphore phải được thực hiện bởi các system call riêng biệt, thay vì trong một bước nguyên tử duy nhất, dẫn đến các race condition có thể xảy ra khi khởi tạo một semaphore. Trong phần này, chúng ta trình bày chi tiết về bản chất của race và xem xét một phương pháp tránh nó dựa trên một ý tưởng được mô tả trong [Stevens, 1999].

Giả sử chúng ta có một ứng dụng bao gồm nhiều peer process sử dụng một semaphore để phối hợp các hành động của họ. Vì không có process nào được đảm bảo là đầu tiên sử dụng semaphore (đây là ý nghĩa của thuật ngữ peer), mỗi process phải sẵn sàng tạo và khởi tạo semaphore nếu nó chưa tồn tại. Cho mục đích này, chúng ta có thể cân nhắc sử dụng code được hiển thị trong Listing 47-5.

```
–––––––––––––––––––––––––––––––––––––––––––––––––from svsem/svsem_bad_init.c
   /* Create a set containing 1 semaphore */
 semid = semget(key, 1, IPC_CREAT | IPC_EXCL | perms);
 if (semid != -1) { /* Successfully created the semaphore */
 union semun arg;
 /* XXXX */
 arg.val = 0; /* Initialize semaphore */
 if (semctl(semid, 0, SETVAL, arg) == -1)
 errExit("semctl");
   } else { /* We didn't create the semaphore */
 if (errno != EEXIST) { /* Unexpected error from semget() */
 errExit("semget");
 semid = semget(key, 1, perms); /* Retrieve ID of existing set */
 if (semid == -1)
 errExit("semget");
 }
 /* Now perform some operation on the semaphore */
 sops[0].sem_op = 1; /* Add 1... */
 sops[0].sem_num = 0; /* to semaphore 0 */
 sops[0].sem_flg = 0;
 if (semop(semid, sops, 1) == -1)
 errExit("semop");
–––––––––––––––––––––––––––––––––––––––––––––––––from svsem/svsem_bad_init.c
```

Vấn đề với code trong Listing 47-5 là nếu hai process thực thi nó cùng lúc, thì chuỗi được hiển thị trong Hình 47-2 có thể xảy ra, nếu khoảng thời gian chạy của process đầu tiên xảy ra kết thúc tại điểm được đánh dấu XXXX trong code. Chuỗi này có vấn đề vì hai lý do. Thứ nhất, process B thực hiện `semop()` trên một semaphore chưa được khởi tạo (tức là giá trị của nó là tùy ý). Thứ hai, lời gọi `semctl()` trong process A ghi đè lên các thay đổi được thực hiện bởi process B.

Giải pháp cho vấn đề này dựa vào một tính năng lịch sử, và hiện tại đã được chuẩn hóa, của việc khởi tạo trường `sem_otime` trong cấu trúc dữ liệu `semid_ds` liên kết với semaphore set. Khi một semaphore set được tạo lần đầu tiên, trường `sem_otime` được khởi tạo thành 0, và nó chỉ được thay đổi bởi một lời gọi `semop()` tiếp theo. Chúng ta có thể khai thác tính năng này để loại bỏ race condition được mô tả ở trên. Chúng ta làm điều này bằng cách chèn code bổ sung để buộc process thứ hai (tức là process không tạo semaphore) chờ cho đến khi process đầu tiên đã cả khởi tạo semaphore lẫn thực hiện một lời gọi `semop()` cập nhật trường `sem_otime`, nhưng không sửa đổi giá trị của semaphore. Code đã sửa đổi được hiển thị trong Listing 47-6.

> Thật không may, giải pháp cho vấn đề khởi tạo được mô tả trong văn bản chính không hoạt động trên tất cả các hệ thống UNIX. Trong một số phiên bản của các dẫn xuất BSD hiện đại, `semop()` không cập nhật trường `sem_otime`.

```
–––––––––––––––––––––––––––––––––––––––––––––––– from svsem/svsem_good_init.c
   semid = semget(key, 1, IPC_CREAT | IPC_EXCL | perms);
 if (semid != -1) { /* Successfully created the semaphore */
 union semun arg;
 struct sembuf sop;
 arg.val = 0; /* So initialize it to 0 */
 if (semctl(semid, 0, SETVAL, arg) == -1)
 errExit("semctl");
 /* Perform a "no-op" semaphore operation - changes sem_otime
 so other processes can see we've initialized the set. */
 sop.sem_num = 0; /* Operate on semaphore 0 */
 sop.sem_op = 0; /* Wait for value to equal 0 */
 sop.sem_flg = 0;
 if (semop(semid, &sop, 1) == -1)
 errExit("semop");
   } else { /* We didn't create the semaphore set */
 const int MAX_TRIES = 10;
 int j;
 union semun arg;
 struct semid_ds ds;
 if (errno != EEXIST) { /* Unexpected error from semget() */
 errExit("semget");
 semid = semget(key, 1, perms); /* Retrieve ID of existing set */
 if (semid == -1)
 errExit("semget");
 /* Wait until another process has called semop() */
 arg.buf = &ds;
 for (j = 0; j < MAX_TRIES; j++) {
 if (semctl(semid, 0, IPC_STAT, arg) == -1)
 errExit("semctl");
 if (ds.sem_otime != 0) /* semop() performed? */
 break; /* Yes, quit loop */
 sleep(1); /* If not, wait and retry */
 }
 if (ds.sem_otime == 0) /* Loop ran to completion! */
 fatal("Existing semaphore not initialized");
 }
 /* Now perform some operation on the semaphore */
–––––––––––––––––––––––––––––––––––––––––––––––– from svsem/svsem_good_init.c
```

Chúng ta có thể sử dụng các biến thể của kỹ thuật được hiển thị trong Listing 47-6 để đảm bảo rằng nhiều semaphore trong một set được khởi tạo đúng cách, hoặc semaphore được khởi tạo thành một giá trị khác không.

Giải pháp phức tạp này cho vấn đề race không bắt buộc trong tất cả các ứng dụng. Chúng ta không cần nó nếu một process được đảm bảo có thể tạo và khởi tạo semaphore trước khi bất kỳ process nào khác cố gắng sử dụng nó. Điều này sẽ là trường hợp, ví dụ, nếu một process cha tạo và khởi tạo semaphore trước khi tạo các child process mà nó chia sẻ semaphore. Trong những trường hợp như vậy, đủ để process đầu tiên theo lời gọi `semget()` của nó bằng thao tác `semctl()` `SETVAL` hoặc `SETALL`.

![](_page_101_Figure_1.jpeg)

**Hình 47-2:** Hai process đua nhau để khởi tạo cùng một semaphore

# **47.6 Các Thao Tác Semaphore**

System call `semop()` thực hiện một hoặc nhiều thao tác trên các semaphore trong semaphore set được xác định bởi `semid`.

```
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
int semop(int semid, struct sembuf *sops, unsigned int nsops);
                                           Returns 0 on success, or –1 on error
```

Đối số `sops` là một pointer đến một mảng chứa các thao tác cần thực hiện, và `nsops` cho biết kích thước của mảng này (phải chứa ít nhất một phần tử). Các thao tác được thực hiện một cách nguyên tử và theo thứ tự mảng. Các phần tử của mảng `sops` là các structure có dạng sau:

```
struct sembuf {
 unsigned short sem_num; /* Semaphore number */
 short sem_op; /* Operation to be performed */
 short sem_flg; /* Operation flags (IPC_NOWAIT and SEM_UNDO) */
};
```

Trường `sem_num` xác định semaphore trong set mà thao tác sẽ được thực hiện. Trường `sem_op` chỉ định thao tác cần thực hiện:

- Nếu `sem_op` lớn hơn 0, giá trị của `sem_op` được thêm vào giá trị semaphore. Kết quả là, các process khác đang chờ giảm giá trị semaphore có thể được đánh thức và thực hiện các thao tác của họ. Process gọi phải có quyền thay đổi (ghi) trên semaphore.
- Nếu `sem_op` bằng 0, giá trị của semaphore được kiểm tra xem nó có hiện tại bằng 0 không. Nếu có, thao tác hoàn thành ngay lập tức; nếu không, `semop()` block cho đến khi giá trị semaphore trở thành 0. Process gọi phải có quyền đọc trên semaphore.
- Nếu `sem_op` nhỏ hơn 0, hãy giảm giá trị của semaphore theo lượng được chỉ định trong `sem_op`. Nếu giá trị hiện tại của semaphore lớn hơn hoặc bằng giá trị tuyệt đối của `sem_op`, thao tác hoàn thành ngay lập tức. Nếu không, `semop()` block cho đến khi giá trị semaphore đã được tăng lên đến mức cho phép thực hiện thao tác mà không dẫn đến giá trị âm. Process gọi phải có quyền thay đổi trên semaphore.

Về mặt ngữ nghĩa, việc tăng giá trị của một semaphore tương ứng với việc làm cho một tài nguyên có sẵn để người khác có thể sử dụng, trong khi việc giảm giá trị của một semaphore tương ứng với việc đặt trước một tài nguyên để (độc quyền) sử dụng bởi process này. Khi giảm giá trị của một semaphore, thao tác bị block nếu giá trị semaphore quá thấp — tức là nếu một process khác đã đặt trước tài nguyên.

Khi một lời gọi `semop()` bị block, process vẫn bị block cho đến khi một trong những điều sau xảy ra:

- Một process khác sửa đổi giá trị của semaphore sao cho thao tác được yêu cầu có thể tiếp tục.
- Một signal ngắt lời gọi `semop()`. Trong trường hợp này, lỗi `EINTR` xảy ra. (Như đã lưu ý trong Mục 21.5, `semop()` không bao giờ được tự động restart sau khi bị ngắt bởi một signal handler.)
- Một process khác xóa semaphore được tham chiếu bởi `semid`. Trong trường hợp này, `semop()` thất bại với lỗi `EIDRM`.

Chúng ta có thể ngăn `semop()` block khi thực hiện một thao tác trên một semaphore cụ thể bằng cách chỉ định flag `IPC_NOWAIT` trong trường `sem_flg` tương ứng. Trong trường hợp đó, nếu `semop()` đáng lẽ bị block, nó thay vào đó thất bại với lỗi `EAGAIN`.

Mặc dù thông thường chỉ thao tác trên một semaphore tại một thời điểm, nhưng có thể thực hiện một lời gọi `semop()` thực hiện các thao tác trên nhiều semaphore trong một set. Điểm chính cần lưu ý là nhóm thao tác này được thực hiện một cách nguyên tử; tức là, `semop()` thực hiện tất cả các thao tác ngay lập tức, nếu có thể, hoặc block cho đến khi có thể thực hiện tất cả các thao tác đồng thời.

> Rất ít hệ thống ghi lại thực tế là `semop()` thực hiện các thao tác theo thứ tự mảng, mặc dù tất cả các hệ thống mà tác giả biết đều làm vậy, và một vài ứng dụng phụ thuộc vào hành vi này. SUSv4 thêm văn bản yêu cầu rõ ràng hành vi này.

Listing 47-7 minh họa việc sử dụng `semop()` để thực hiện các thao tác trên ba semaphore trong một set. Các thao tác trên semaphore 0 và 2 có thể không thực hiện được ngay lập tức, tùy thuộc vào các giá trị hiện tại của các semaphore. Nếu thao tác trên semaphore 0 không thể thực hiện ngay lập tức, thì không có thao tác nào được yêu cầu được thực hiện, và `semop()` block. Mặt khác, nếu thao tác trên semaphore 0 có thể thực hiện ngay lập tức, nhưng thao tác trên semaphore 2 không thể, thì — vì flag `IPC_NOWAIT` được chỉ định — không có thao tác nào được yêu cầu được thực hiện, và `semop()` trả về ngay lập tức với lỗi `EAGAIN`.

System call `semtimedop()` thực hiện cùng nhiệm vụ như `semop()`, ngoại trừ đối số `timeout` chỉ định giới hạn trên về thời gian lời gọi sẽ block.

```
#define _GNU_SOURCE
#include <sys/types.h> /* For portability */
#include <sys/sem.h>
int semtimedop(int semid, struct sembuf *sops, unsigned int nsops,
 struct timespec *timeout);
                                         Returns 0 on success, or –1 on error
```

Đối số `timeout` là một pointer đến một cấu trúc `timespec` (Mục 23.4.2), cho phép một khoảng thời gian được biểu thị bằng số giây và nanosecond. Nếu khoảng thời gian đã chỉ định hết hạn trước khi có thể hoàn thành thao tác semaphore, `semtimedop()` thất bại với lỗi `EAGAIN`. Nếu `timeout` được chỉ định là `NULL`, `semtimedop()` hoàn toàn giống với `semop()`.

System call `semtimedop()` được cung cấp như một phương pháp hiệu quả hơn để đặt timeout trên thao tác semaphore so với việc sử dụng `setitimer()` cộng với `semop()`. Lợi ích hiệu suất nhỏ mà điều này mang lại có ý nghĩa đối với một số ứng dụng nhất định (đặc biệt là một số hệ thống cơ sở dữ liệu) cần thực hiện thường xuyên các thao tác như vậy. Tuy nhiên, `semtimedop()` không được chỉ định trong SUSv3 và chỉ có mặt trên một vài hệ thống UNIX khác.

> System call `semtimedop()` xuất hiện như một tính năng mới trong Linux 2.6 và sau đó được back-port vào Linux 2.4, bắt đầu từ kernel 2.4.22.

```
 struct sembuf sops[3];
 sops[0].sem_num = 0; /* Subtract 1 from semaphore 0 */
 sops[0].sem_op = -1;
 sops[0].sem_flg = 0;
 sops[1].sem_num = 1; /* Add 2 to semaphore 1 */
 sops[1].sem_op = 2;
 sops[1].sem_flg = 0;
 sops[2].sem_num = 2; /* Wait for semaphore 2 to equal 0 */
 sops[2].sem_op = 0;
 sops[2].sem_flg = IPC_NOWAIT; /* But don't block if operation
 can't be performed immediately */
 if (semop(semid, sops, 3) == -1) {
 if (errno == EAGAIN) /* Semaphore 2 would have blocked */
 printf("Operation would have blocked\n");
 else
 errExit("semop"); /* Some other error */
 }
```

## **Chương trình ví dụ**

Chương trình trong Listing 47-8 cung cấp một giao diện dòng lệnh cho system call `semop()`. Đối số đầu tiên của chương trình này là identifier của semaphore set mà các thao tác sẽ được thực hiện.

Mỗi đối số dòng lệnh còn lại chỉ định một nhóm thao tác semaphore cần thực hiện trong một lời gọi `semop()` duy nhất. Các thao tác trong một đối số dòng lệnh duy nhất được phân cách bằng dấu phẩy. Mỗi thao tác có một trong các dạng sau:

- `semnum+value`: thêm `value` vào semaphore `semnum`.
- `semnum-value`: trừ `value` khỏi semaphore `semnum`.
- `semnum=0`: kiểm tra semaphore `semnum` xem nó có bằng 0 không.

Ở cuối mỗi thao tác, chúng ta có thể tùy chọn bao gồm một `n`, một `u`, hoặc cả hai. Chữ `n` có nghĩa là bao gồm `IPC_NOWAIT` trong giá trị `sem_flg` cho thao tác này. Chữ `u` có nghĩa là bao gồm `SEM_UNDO` trong `sem_flg`. (Chúng ta mô tả flag `SEM_UNDO` trong Mục 47.8.)

Dòng lệnh sau chỉ định hai lời gọi `semop()` trên semaphore set có identifier là 0:

#### $ **./svsem\_op 0 0=0 0-1,1-2n**

Đối số dòng lệnh đầu tiên chỉ định một lời gọi `semop()` chờ cho semaphore 0 bằng 0. Đối số thứ hai chỉ định một lời gọi `semop()` trừ 1 khỏi semaphore 0, và trừ 2 khỏi semaphore 1. Đối với thao tác trên semaphore 0, `sem_flg` là 0; đối với thao tác trên semaphore 1, `sem_flg` là `IPC_NOWAIT`.

**Listing 47-8:** Thực hiện các thao tác semaphore System V với `semop()`

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_op.c
#include <sys/types.h>
#include <sys/sem.h>
#include <ctype.h>
#include "curr_time.h" /* Declaration of currTime() */
#include "tlpi_hdr.h"
#define MAX_SEMOPS 1000 /* Maximum operations that we permit for
 a single semop() */
static void
usageError(const char *progName)
{
 fprintf(stderr, "Usage: %s semid op[,op...] ...\n\n", progName);
 fprintf(stderr, "'op' is either: <sem#>{+|-}<value>[n][u]\n");
 fprintf(stderr, " or: <sem#>=0[n]\n");
 fprintf(stderr, " \"n\" means include IPC_NOWAIT in 'op'\n");
 fprintf(stderr, " \"u\" means include SEM_UNDO in 'op'\n\n");
 fprintf(stderr, "The operations in each argument are "
 "performed in a single semop() call\n\n");
 fprintf(stderr, "e.g.: %s 12345 0+1,1-2un\n", progName);
 fprintf(stderr, " %s 12345 0=0n 1+1,2-1u 1=0\n", progName);
 exit(EXIT_FAILURE);
}
/* Parse comma-delimited operations in 'arg', returning them in the
 array 'sops'. Return number of operations as function result. */
static int
parseOps(char *arg, struct sembuf sops[])
{
 char *comma, *sign, *remaining, *flags;
 int numOps; /* Number of operations in 'arg' */
 for (numOps = 0, remaining = arg; ; numOps++) {
 if (numOps >= MAX_SEMOPS)
 cmdLineErr("Too many operations (maximum=%d): \"%s\"\n",
 MAX_SEMOPS, arg);
 if (*remaining == '\0')
 fatal("Trailing comma or empty argument: \"%s\"", arg);
 if (!isdigit((unsigned char) *remaining))
 cmdLineErr("Expected initial digit: \"%s\"\n", arg);
 sops[numOps].sem_num = strtol(remaining, &sign, 10);
 if (*sign == '\0' || strchr("+-=", *sign) == NULL)
 cmdLineErr("Expected '+', '-', or '=' in \"%s\"\n", arg);
 if (!isdigit((unsigned char) *(sign + 1)))
 cmdLineErr("Expected digit after '%c' in \"%s\"\n", *sign, arg);
 sops[numOps].sem_op = strtol(sign + 1, &flags, 10);
```

```
 if (*sign == '-') /* Reverse sign of operation */
 sops[numOps].sem_op = - sops[numOps].sem_op;
 else if (*sign == '=') /* Should be '=0' */
 if (sops[numOps].sem_op != 0)
 cmdLineErr("Expected \"=0\" in \"%s\"\n", arg);
 sops[numOps].sem_flg = 0;
 for (;; flags++) {
 if (*flags == 'n')
 sops[numOps].sem_flg |= IPC_NOWAIT;
 else if (*flags == 'u')
 sops[numOps].sem_flg |= SEM_UNDO;
 else
 break;
 }
 if (*flags != ',' && *flags != '\0')
 cmdLineErr("Bad trailing character (%c) in \"%s\"\n", *flags, arg);
 comma = strchr(remaining, ',');
 if (comma == NULL)
 break; /* No comma --> no more ops */
 else
 remaining = comma + 1;
 }
 return numOps + 1;
}
int
main(int argc, char *argv[])
{
 struct sembuf sops[MAX_SEMOPS];
 int ind, nsops;
 if (argc < 2 || strcmp(argv[1], "--help") == 0)
 usageError(argv[0]);
 for (ind = 2; argv[ind] != NULL; ind++) {
 nsops = parseOps(argv[ind], sops);
 printf("%5ld, %s: about to semop() [%s]\n", (long) getpid(),
 currTime("%T"), argv[ind]);
 if (semop(getInt(argv[1], 0, "semid"), sops, nsops) == -1)
 errExit("semop (PID=%ld)", (long) getpid());
 printf("%5ld, %s: semop() completed [%s]\n", (long) getpid(),
 currTime("%T"), argv[ind]);
 }
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/svsem_op.c
```

Sử dụng chương trình trong Listing 47-8, cùng với nhiều chương trình khác được hiển thị trong chương này, chúng ta có thể nghiên cứu hoạt động của semaphore System V, như được minh họa trong phiên shell sau. Chúng ta bắt đầu bằng cách sử dụng một chương trình tạo một semaphore set chứa hai semaphore, mà chúng ta khởi tạo thành 1 và 0:

```
$ ./svsem_create -p 2
32769 ID of semaphore set
$ ./svsem_setall 32769 1 0
Semaphore values changed (PID=3658)
```

Chúng ta không hiển thị code của chương trình `svsem/svsem_create.c` trong chương này, nhưng nó được cung cấp trong bản phân phối mã nguồn của cuốn sách này. Chương trình này thực hiện cùng chức năng cho semaphore như chương trình trong Listing 46-1 (trang 938) thực hiện cho message queue; tức là, nó tạo một semaphore set. Sự khác biệt đáng chú ý duy nhất là `svsem_create.c` nhận một đối số bổ sung chỉ định kích thước của semaphore set cần tạo.

Tiếp theo, chúng ta khởi động ba instance nền của chương trình trong Listing 47-8 để thực hiện các thao tác `semop()` trên semaphore set. Chương trình in các message trước và sau mỗi thao tác semaphore. Các message này bao gồm thời gian, để chúng ta có thể thấy khi nào mỗi thao tác bắt đầu và khi nào nó hoàn thành, và process ID, để chúng ta có thể theo dõi hoạt động của nhiều instance của chương trình. Lệnh đầu tiên gửi yêu cầu giảm cả hai semaphore đi 1:

```
$ ./svsem_op 32769 0-1,1-1 & Operation 1
 3659, 16:02:05: about to semop() [0-1,1-1]
[1] 3659
```

Trong đầu ra trên, chúng ta thấy rằng chương trình in một message nói rằng thao tác `semop()` sắp được thực hiện, nhưng không in thêm message nào, vì lời gọi `semop()` bị block. Lời gọi bị block vì semaphore 1 có giá trị 0.

Tiếp theo, chúng ta thực hiện một lệnh gửi yêu cầu giảm semaphore 1 đi 1:

```
$ ./svsem_op 32769 1-1 & Operation 2
 3660, 16:02:22: about to semop() [1-1]
[2] 3660
```

Lệnh này cũng bị block. Tiếp theo, chúng ta thực hiện một lệnh chờ giá trị của semaphore 0 bằng 0:

```
$ ./svsem_op 32769 0=0 & Operation 3
 3661, 16:02:27: about to semop() [0=0]
[3] 3661
```

Một lần nữa, lệnh này bị block, trong trường hợp này vì giá trị của semaphore 0 hiện tại là 1.

Bây giờ, chúng ta sử dụng chương trình trong Listing 47-3 để kiểm tra semaphore set:

```
$ ./svsem_mon 32769
Semaphore changed: Sun Jul 25 16:01:53 2010
Last semop(): Thu Jan 1 01:00:00 1970
Sem # Value SEMPID SEMNCNT SEMZCNT
 0 1 0 1 1
 1 0 0 2 0
```

Khi một semaphore set được tạo, trường `sem_otime` của cấu trúc `semid_ds` liên kết được khởi tạo thành 0. Một giá trị thời gian lịch là 0 tương ứng với Epoch (Mục 10.1), và `ctime()` hiển thị đây là 1 giờ sáng, ngày 1 tháng 1 năm 1970, vì múi giờ địa phương là Trung Âu, sớm hơn một giờ so với UTC.

Kiểm tra đầu ra thêm, chúng ta có thể thấy rằng, đối với semaphore 0, giá trị `semncnt` là 1 vì operation 1 đang chờ giảm giá trị semaphore, và `semzcnt` là 1 vì operation 3 đang chờ semaphore này bằng 0. Đối với semaphore 1, giá trị `semncnt` là 2 phản ánh thực tế là operation 1 và operation 2 đang chờ giảm giá trị semaphore.

Tiếp theo, chúng ta thử một thao tác không chặn trên semaphore set. Thao tác này chờ semaphore 0 bằng 0. Vì thao tác này không thể thực hiện ngay lập tức, `semop()` thất bại với lỗi `EAGAIN`:

```
$ ./svsem_op 32769 0=0n Operation 4
 3673, 16:03:13: about to semop() [0=0n]
ERROR [EAGAIN/EWOULDBLOCK Resource temporarily unavailable] semop (PID=3673)
```

Bây giờ chúng ta thêm 1 vào semaphore 1. Điều này khiến hai thao tác bị block trước đó (1 và 3) được mở khóa:

```
$ ./svsem_op 32769 1+1 Operation 5
 3674, 16:03:29: about to semop() [1+1]
 3659, 16:03:29: semop() completed [0-1,1-1] Operation 1 completes
 3661, 16:03:29: semop() completed [0=0] Operation 3 completes
 3674, 16:03:29: semop() completed [1+1] Operation 5 completes
[1] Done ./svsem_op 32769 0-1,1-1
[3]+ Done ./svsem_op 32769 0=0
```

Khi chúng ta sử dụng chương trình giám sát để kiểm tra trạng thái của semaphore set, chúng ta thấy rằng trường `sem_otime` của cấu trúc `semid_ds` liên kết đã được cập nhật, và các giá trị `sempid` của cả hai semaphore đã được cập nhật. Chúng ta cũng thấy rằng giá trị `semncnt` cho semaphore 1 là 1, vì operation 2 vẫn đang bị block, chờ giảm giá trị của semaphore này:

#### $ **./svsem\_mon 32769** Semaphore changed: Sun Jul 25 16:01:53 2010 Last semop(): Sun Jul 25 16:03:29 2010 Sem # Value SEMPID SEMNCNT SEMZCNT 0 0 3661 0 0 1 0 3659 1 0

Từ đầu ra trên, chúng ta thấy rằng giá trị `sem_otime` đã được cập nhật. Chúng ta cũng thấy rằng semaphore 0 được thao tác cuối cùng bởi process ID 3661 (operation 3) và semaphore 1 được thao tác cuối cùng bởi process ID 3659 (operation 1).

Cuối cùng, chúng ta xóa semaphore set. Điều này khiến operation 2 vẫn đang bị block thất bại với lỗi `EIDRM`:

```
$ ./svsem_rm 32769
ERROR [EIDRM Identifier removed] semop (PID=3660)
```

Chúng ta không hiển thị mã nguồn cho chương trình `svsem/svsem_rm.c` trong chương này, nhưng nó được cung cấp trong bản phân phối mã nguồn của cuốn sách này. Chương trình này xóa semaphore set được xác định bởi đối số dòng lệnh của nó.

# **47.7 Xử Lý Nhiều Thao Tác Semaphore Bị Block**

Nếu nhiều process đang bị block cố gắng giảm giá trị của một semaphore theo cùng một lượng, thì không thể xác định process nào sẽ được phép thực hiện thao tác trước khi điều đó trở nên khả thi (tức là process nào có thể thực hiện thao tác sẽ phụ thuộc vào các biến động của thuật toán lập lịch process của kernel).

Mặt khác, nếu các process đang bị block cố gắng giảm giá trị semaphore theo các lượng khác nhau, thì các yêu cầu được phục vụ theo thứ tự chúng trở nên khả thi. Giả sử rằng một semaphore hiện có giá trị 0, và process A yêu cầu giảm giá trị semaphore đi 2, và sau đó process B yêu cầu giảm giá trị đi 1. Nếu một process thứ ba sau đó thêm 1 vào semaphore, process B sẽ là process đầu tiên được mở khóa và thực hiện thao tác của nó, mặc dù process A đã là người đầu tiên yêu cầu một thao tác đối với semaphore. Trong các ứng dụng được thiết kế kém, các tình huống như vậy có thể dẫn đến starvation, trong đó một process vẫn bị block mãi mãi vì trạng thái của semaphore không bao giờ đạt đến mức cho phép thao tác của process tiếp tục. Tiếp tục ví dụ của chúng ta, chúng ta có thể hình dung các tình huống trong đó nhiều process điều chỉnh semaphore sao cho giá trị của nó không bao giờ lớn hơn 1, với kết quả là process A vẫn bị block mãi mãi.

Starvation cũng có thể xảy ra nếu một process đang bị block cố gắng thực hiện các thao tác trên nhiều semaphore. Hãy xem xét tình huống sau, được thực hiện trên một cặp semaphore, cả hai ban đầu có giá trị 0:

1. Process A gửi yêu cầu trừ 1 khỏi semaphore 0 và 1 (blocks).
2. Process B gửi yêu cầu trừ 1 khỏi semaphore 0 (blocks).
3. Process C thêm 1 vào semaphore 0.

Tại điểm này, process B được mở khóa và hoàn thành yêu cầu của nó, mặc dù nó đặt yêu cầu sau process A. Một lần nữa, có thể thiết kế các tình huống trong đó process A bị starvation trong khi các process khác điều chỉnh và block trên các giá trị của các semaphore riêng lẻ.

# **47.8 Giá Trị Undo Semaphore**

Giả sử rằng, sau khi điều chỉnh giá trị của một semaphore (ví dụ: giảm giá trị semaphore để nó bây giờ là 0), một process sau đó kết thúc, dù có chủ ý hay vô tình. Theo mặc định, giá trị của semaphore không thay đổi. Điều này có thể là vấn đề đối với các process khác sử dụng semaphore, vì họ có thể đang bị block chờ semaphore đó — tức là chờ process vừa kết thúc hoàn tác thay đổi mà nó đã thực hiện.

Để tránh những vấn đề như vậy, chúng ta có thể sử dụng flag `SEM_UNDO` khi thay đổi giá trị của một semaphore qua `semop()`. Khi flag này được chỉ định, kernel ghi lại tác động của thao tác semaphore, và sau đó hoàn tác thao tác nếu process kết thúc. Việc hoàn tác xảy ra bất kể process kết thúc bình thường hay bất thường.

Kernel không cần lưu bản ghi tất cả các thao tác được thực hiện bằng `SEM_UNDO`. Chỉ cần ghi lại tổng của tất cả các điều chỉnh semaphore được thực hiện bằng `SEM_UNDO` trong một tổng số nguyên per-semaphore, per-process gọi là giá trị `semadj` (semaphore adjustment). Khi process kết thúc, tất cả những gì cần thiết là trừ tổng này khỏi giá trị hiện tại của semaphore.

> Kể từ Linux 2.6, các process (thread) được tạo bằng `clone()` chia sẻ các giá trị `semadj` nếu flag `CLONE_SYSVSEM` được sử dụng. Việc chia sẻ này được yêu cầu cho một hệ thống tuân thủ của POSIX thread. Hệ thống NPTL threading sử dụng `CLONE_SYSVSEM` để triển khai `pthread_create()`.

Khi giá trị semaphore được đặt bằng thao tác `SETVAL` hoặc `SETALL` của `semctl()`, các giá trị `semadj` tương ứng sẽ bị xóa (tức là đặt thành 0) trong tất cả các process sử dụng semaphore. Điều này có ý nghĩa, vì việc đặt tuyệt đối giá trị của một semaphore phá hủy giá trị liên kết với bản ghi lịch sử được duy trì trong tổng `semadj`.

Một child được tạo qua `fork()` không kế thừa các giá trị `semadj` của cha; sẽ không có ý nghĩa gì khi child hoàn tác các thao tác semaphore của cha. Mặt khác, các giá trị `semadj` được bảo toàn qua một `exec()`. Điều này cho phép chúng ta điều chỉnh giá trị semaphore bằng `SEM_UNDO`, và sau đó `exec()` một chương trình không thực hiện thao tác nào trên semaphore, nhưng tự động điều chỉnh semaphore khi process kết thúc. (Điều này có thể được sử dụng như một kỹ thuật cho phép một process khác khám phá khi process này kết thúc.)

## **Ví dụ về tác động của SEM_UNDO**

Phiên shell log sau cho thấy tác động của việc thực hiện các thao tác trên hai semaphore: một thao tác với flag `SEM_UNDO` và một thao tác không có. Chúng ta bắt đầu bằng cách tạo một set chứa hai semaphore:

```
$ ./svsem_create -p 2
131073
```

Tiếp theo, chúng ta thực hiện một lệnh thêm 1 vào cả hai semaphore và sau đó kết thúc. Thao tác trên semaphore 0 chỉ định flag `SEM_UNDO`:

```
$ ./svsem_op 131073 0+1u 1+1
 2248, 06:41:56: about to semop()
 2248, 06:41:56: semop() completed
```

Bây giờ, chúng ta sử dụng chương trình trong Listing 47-3 để kiểm tra trạng thái của các semaphore:

```
$ ./svsem_mon 131073
Semaphore changed: Sun Jul 25 06:41:34 2010
Last semop(): Sun Jul 25 06:41:56 2010
Sem # Value SEMPID SEMNCNT SEMZCNT
 0 0 2248 0 0
 1 1 2248 0 0
```

Nhìn vào các giá trị semaphore trong hai dòng cuối của đầu ra trên, chúng ta có thể thấy rằng thao tác trên semaphore 0 đã được hoàn tác, nhưng thao tác trên semaphore 1 thì không.

## **Hạn chế của SEM_UNDO**

Chúng ta kết luận bằng cách lưu ý rằng flag `SEM_UNDO` kém hữu ích hơn so với vẻ ngoài của nó, vì hai lý do. Một là vì việc sửa đổi một semaphore thường tương ứng với việc thu nhận hoặc giải phóng một tài nguyên chia sẻ, nên việc sử dụng `SEM_UNDO` một mình có thể không đủ để cho phép một ứng dụng đa process phục hồi trong trường hợp một process kết thúc bất ngờ. Trừ khi việc kết thúc process cũng tự động trả trạng thái tài nguyên chia sẻ về một trạng thái nhất quán (không có khả năng trong nhiều tình huống), việc hoàn tác một thao tác semaphore có lẽ không đủ để cho phép ứng dụng phục hồi.

Yếu tố thứ hai hạn chế tính hữu dụng của `SEM_UNDO` là trong một số trường hợp, không thể thực hiện các điều chỉnh semaphore khi một process kết thúc. Hãy xem xét tình huống sau, được áp dụng cho một semaphore có giá trị ban đầu là 0:

1. Process A tăng giá trị của một semaphore lên 2, chỉ định flag `SEM_UNDO` cho thao tác.
2. Process B giảm giá trị của semaphore đi 1, vì vậy nó có giá trị 1.
3. Process A kết thúc.

Tại điểm này, không thể hoàn toàn hoàn tác tác động của thao tác của process A trong bước 1, vì giá trị của semaphore quá thấp. Có ba cách có thể để giải quyết tình huống này:

- Buộc process phải block cho đến khi điều chỉnh semaphore có thể thực hiện được.
- Giảm giá trị semaphore càng nhiều càng tốt (tức là xuống 0) và thoát.
- Thoát mà không thực hiện bất kỳ điều chỉnh semaphore nào.

Giải pháp đầu tiên không khả thi vì nó có thể buộc một process đang kết thúc phải block mãi mãi. Linux áp dụng giải pháp thứ hai. Một số hệ thống UNIX khác áp dụng giải pháp thứ ba. SUSv3 không nói rõ một hệ thống nên làm gì trong tình huống này.

> Một thao tác undo cố gắng nâng giá trị semaphore lên trên giá trị tối đa được phép là 32,767 (giới hạn `SEMVMX`, được mô tả Mục 47.10) cũng gây ra hành vi bất thường. Trong trường hợp này, kernel luôn thực hiện điều chỉnh, do đó (bất hợp lệ) nâng giá trị semaphore lên trên `SEMVMX`.

## **47.9 Triển Khai Giao Thức Binary Semaphore**

API cho semaphore System V phức tạp, cả vì các giá trị semaphore có thể được điều chỉnh theo các lượng tùy ý, và vì các semaphore được phân bổ và thao tác trong các set. Cả hai tính năng này cung cấp nhiều chức năng hơn mức thường cần thiết trong các ứng dụng, vì vậy việc triển khai một số giao thức (API) đơn giản hơn trên cơ sở semaphore System V là hữu ích.

Một giao thức thường được sử dụng là binary semaphore. Một binary semaphore có hai giá trị: có sẵn (free) và đã đặt trước (in use). Hai thao tác được định nghĩa cho binary semaphore:

Reserve: Cố gắng đặt trước semaphore này để sử dụng độc quyền. Nếu semaphore đã được đặt trước bởi một process khác, thì block cho đến khi semaphore được giải phóng.

Release: Giải phóng một semaphore hiện đang được đặt trước, để nó có thể được đặt trước bởi một process khác.

> Trong khoa học máy tính học thuật, hai thao tác này thường được gọi bằng tên P và V, các chữ cái đầu của các thuật ngữ tiếng Hà Lan cho các thao tác này. Thuật ngữ này được đặt ra bởi nhà khoa học máy tính người Hà Lan quá cố Edsger Dijkstra, người đã tạo ra phần lớn công trình lý thuyết ban đầu về semaphore. Các thuật ngữ down (decrement semaphore) và up (increment semaphore) cũng được sử dụng. POSIX gọi hai thao tác là wait và post.

Một thao tác thứ ba đôi khi cũng được định nghĩa:

Reserve conditionally: Thực hiện một nỗ lực không chặn (nonblocking) để đặt trước semaphore này để sử dụng độc quyền. Nếu semaphore đã được đặt trước, thì ngay lập tức trả về một trạng thái chỉ ra rằng semaphore không có sẵn.

Khi triển khai binary semaphore, chúng ta phải chọn cách biểu diễn các trạng thái có sẵn và đã đặt trước, và cách triển khai các thao tác trên. Một khoảnh khắc suy nghĩ dẫn chúng ta đến nhận ra rằng cách tốt nhất để biểu diễn các trạng thái là sử dụng giá trị 1 cho free và giá trị 0 cho reserved, với các thao tác reserve và release giảm và tăng giá trị semaphore lên một.

Listing 47-9 và Listing 47-10 cung cấp một cài đặt binary semaphore bằng semaphore System V. Cũng như cung cấp các nguyên mẫu của các hàm trong cài đặt, file header trong Listing 47-9 khai báo hai biến Boolean toàn cục được expose bởi cài đặt. Biến `bsUseSemUndo` kiểm soát liệu cài đặt có sử dụng flag `SEM_UNDO` trong các lời gọi `semop()` không. Biến `bsRetryOnEintr` kiểm soát liệu cài đặt có restart các lời gọi `semop()` bị ngắt bởi signal không.

**Listing 47-9:** File header cho `binary_sems.c`

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/binary_sems.h
#ifndef BINARY_SEMS_H /* Prevent accidental double inclusion */
#define BINARY_SEMS_H
#include "tlpi_hdr.h"
/* Variables controlling operation of functions below */
extern Boolean bsUseSemUndo; /* Use SEM_UNDO during semop()? */
extern Boolean bsRetryOnEintr; /* Retry if semop() interrupted by
 signal handler? */
int initSemAvailable(int semId, int semNum);
int initSemInUse(int semId, int semNum);
int reserveSem(int semId, int semNum);
int releaseSem(int semId, int semNum);
#endif
–––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/binary_sems.h
```

Listing 47-10 hiển thị cài đặt của các hàm binary semaphore. Mỗi hàm trong cài đặt này nhận hai đối số, xác định một semaphore set và số của một semaphore trong set đó. (Các hàm này không xử lý việc tạo và xóa các semaphore set; và chúng cũng không xử lý race condition được mô tả trong Mục 47.5.) Chúng ta sử dụng các hàm này trong các chương trình ví dụ được hiển thị trong Mục 48.4.

**Listing 47-10:** Triển khai binary semaphore bằng semaphore System V

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/binary_sems.c
#include <sys/types.h>
#include <sys/sem.h>
#include "semun.h" /* Definition of semun union */
#include "binary_sems.h"
Boolean bsUseSemUndo = FALSE;
Boolean bsRetryOnEintr = TRUE;
int /* Initialize semaphore to 1 (i.e., "available") */
initSemAvailable(int semId, int semNum)
{
 union semun arg;
 arg.val = 1;
 return semctl(semId, semNum, SETVAL, arg);
}
int /* Initialize semaphore to 0 (i.e., "in use") */
initSemInUse(int semId, int semNum)
{
 union semun arg;
 arg.val = 0;
 return semctl(semId, semNum, SETVAL, arg);
}
/* Reserve semaphore (blocking), return 0 on success, or -1 with 'errno'
 set to EINTR if operation was interrupted by a signal handler */
int /* Reserve semaphore - decrement it by 1 */
reserveSem(int semId, int semNum)
{
 struct sembuf sops;
 sops.sem_num = semNum;
 sops.sem_op = -1;
 sops.sem_flg = bsUseSemUndo ? SEM_UNDO : 0;
 while (semop(semId, &sops, 1) == -1)
 if (errno != EINTR || !bsRetryOnEintr)
 return -1;
 return 0;
}
```

```
int /* Release semaphore - increment it by 1 */
releaseSem(int semId, int semNum)
{
 struct sembuf sops;
 sops.sem_num = semNum;
 sops.sem_op = 1;
 sops.sem_flg = bsUseSemUndo ? SEM_UNDO : 0;
 return semop(semId, &sops, 1);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– svsem/binary_sems.c
```

# **47.10 Các Giới Hạn Semaphore**

Hầu hết các hệ thống UNIX áp đặt các giới hạn khác nhau đối với hoạt động của semaphore System V. Sau đây là danh sách các giới hạn semaphore Linux. System call bị ảnh hưởng bởi giới hạn và lỗi xảy ra nếu giới hạn đạt được được ghi chú trong ngoặc đơn.

`SEMAEM`

Đây là giá trị tối đa có thể được ghi trong tổng `semadj`. `SEMAEM` được định nghĩa có cùng giá trị với `SEMVMX` (được mô tả bên dưới). (`semop()`, `ERANGE`)

`SEMMNI`

Đây là giới hạn toàn hệ thống về số lượng semaphore identifier (nói cách khác, semaphore set) có thể được tạo. (`semget()`, `ENOSPC`)

`SEMMSL`

Đây là số lượng semaphore tối đa có thể được phân bổ trong một semaphore set. (`semget()`, `EINVAL`)

`SEMMNS`

Đây là giới hạn toàn hệ thống về số lượng semaphore trong tất cả các semaphore set. Số lượng semaphore trên hệ thống cũng bị giới hạn bởi `SEMMNI` và `SEMMSL`; trên thực tế, giá trị mặc định cho `SEMMNS` là tích của các mặc định cho hai giới hạn này. (`semget()`, `ENOSPC`)

`SEMOPM`

Đây là số lượng thao tác tối đa mỗi lần gọi `semop()`. (`semop()`, `E2BIG`)

`SEMVMX`

Đây là giá trị tối đa cho một semaphore. (`semop()`, `ERANGE`)

Các giới hạn trên xuất hiện trên hầu hết các hệ thống UNIX. Một số hệ thống UNIX (nhưng không phải Linux) áp đặt các giới hạn bổ sung sau liên quan đến các thao tác semaphore undo (Mục 47.8):

`SEMMNU`

Đây là giới hạn toàn hệ thống về tổng số cấu trúc undo semaphore. Các cấu trúc undo được phân bổ để lưu trữ các giá trị `semadj`.

`SEMUME`

Đây là số lượng undo entry tối đa mỗi cấu trúc undo semaphore.

Khi khởi động hệ thống, các giới hạn semaphore được đặt thành các giá trị mặc định. Các mặc định này có thể thay đổi qua các phiên bản kernel. (Một số kernel của nhà phân phối đặt các mặc định khác so với các kernel vanilla.) Một số giới hạn này có thể được sửa đổi bằng cách thay đổi các giá trị được lưu trong file `/proc/sys/kernel/sem` đặc thù Linux. File này chứa bốn số phân cách bằng dấu cách định nghĩa, theo thứ tự, các giới hạn `SEMMSL`, `SEMMNS`, `SEMOPM` và `SEMMNI`. (Các giới hạn `SEMVMX` và `SEMAEM` không thể thay đổi; cả hai đều được cố định ở 32,767.) Ví dụ, đây là các giới hạn mặc định mà chúng ta thấy cho Linux 2.6.31 trên một hệ thống x86-32:

```
$ cd /proc/sys/kernel
$ cat sem
250 32000 32 128
```

Các định dạng được sử dụng trong hệ thống file Linux `/proc` không nhất quán cho ba cơ chế IPC System V. Đối với message queue và shared memory, mỗi giới hạn có thể cấu hình được kiểm soát bởi một file riêng. Đối với semaphore, một file chứa tất cả các giới hạn có thể cấu hình. Đây là một sự cố lịch sử xảy ra trong quá trình phát triển các API này và khó có thể sửa chữa vì lý do tương thích.

Bảng 47-1 hiển thị giá trị tối đa mà mỗi giới hạn có thể được nâng lên trên kiến trúc x86-32. Lưu ý thông tin bổ sung sau vào bảng này:

Có thể nâng `SEMMSL` lên các giá trị lớn hơn 65.536, và tạo semaphore set lên kích thước lớn hơn đó. Tuy nhiên, không thể sử dụng `semop()` để điều chỉnh các semaphore trong set vượt quá phần tử thứ 65.536.

> Vì một số hạn chế trong cài đặt hiện tại, giới hạn trên được khuyến nghị thực tế về kích thước của một semaphore set là khoảng 8000.

- Giá trị trần thực tế cho giới hạn `SEMMNS` được quy định bởi lượng RAM có sẵn trên hệ thống.
- Giá trị trần cho giới hạn `SEMOPM` được xác định bởi các nguyên tắc phân bổ bộ nhớ được sử dụng trong kernel. Giá trị tối đa được khuyến nghị là 1000. Trong thực tế, hiếm khi hữu ích khi thực hiện nhiều hơn vài thao tác trong một lần gọi `semop()` duy nhất.

**Bảng 47-1:** Các giới hạn semaphore System V

| Giới hạn | Giá trị trần (x86-32) |  |
|----------|------------------------|--|
| SEMMNI   | 32768 (IPCMNI)         |  |
| SEMMSL   | 65536                  |  |
| SEMMNS   | 2147483647 (INT_MAX)   |  |
| SEMOPM   | Xem văn bản            |  |

Thao tác `IPC_INFO` của `semctl()` đặc thù Linux lấy một cấu trúc kiểu `seminfo`, chứa các giá trị của các giới hạn semaphore khác nhau:

```
union semun arg;
struct seminfo buf;
arg.__buf = &buf;
semctl(0, 0, IPC_INFO, arg);
```

Một thao tác đặc thù Linux liên quan, `SEM_INFO`, lấy một cấu trúc `seminfo` chứa thông tin về các tài nguyên thực tế được sử dụng cho các đối tượng semaphore. Một ví dụ về việc sử dụng `SEM_INFO` được cung cấp trong file `svsem/svsem_info.c` trong bản phân phối mã nguồn của cuốn sách này.

Chi tiết về `IPC_INFO`, `SEM_INFO` và cấu trúc `seminfo` có thể tìm thấy trong trang hướng dẫn `semctl(2)`.

# **47.11 Những Nhược Điểm của Semaphore System V**

Semaphore System V có nhiều nhược điểm giống như message queue (Mục 46.9), bao gồm:

- Semaphore được tham chiếu bởi identifier, thay vì file descriptor được sử dụng bởi hầu hết các cơ chế I/O và IPC UNIX khác. Điều này làm cho khó thực hiện các thao tác như đồng thời chờ cả trên một semaphore và trên đầu vào từ một file descriptor. (Có thể giải quyết khó khăn này bằng cách tạo một child process hoặc thread thao tác trên semaphore và ghi message vào một pipe được giám sát, cùng với các file descriptor khác, bằng một trong các phương pháp được mô tả trong Chương 63.)
- Việc sử dụng key thay vì tên file để xác định semaphore dẫn đến sự phức tạp lập trình bổ sung.
- Việc sử dụng các system call riêng biệt để tạo và khởi tạo semaphore có nghĩa là trong một số trường hợp, chúng ta phải thực hiện thêm công việc lập trình để tránh race condition khi khởi tạo một semaphore.
- Kernel không duy trì một đếm số lượng process tham chiếu đến một semaphore set. Điều này làm phức tạp quyết định khi nào là thích hợp để xóa một semaphore set và làm cho việc đảm bảo rằng một set không được sử dụng sẽ bị xóa trở nên khó khăn.
- Giao diện lập trình được cung cấp bởi System V là quá phức tạp. Trong trường hợp thông thường, một chương trình thao tác trên một semaphore duy nhất. Khả năng đồng thời thao tác trên nhiều semaphore trong một set là không cần thiết.
- Có các giới hạn khác nhau đối với hoạt động của semaphore. Các giới hạn này có thể cấu hình được, nhưng nếu ứng dụng hoạt động ngoài phạm vi các giới hạn mặc định, điều này đòi hỏi thêm công việc khi cài đặt ứng dụng.

Tuy nhiên, không giống như tình huống với message queue, có ít sự thay thế hơn cho semaphore System V, và do đó có nhiều tình huống hơn mà chúng ta có thể chọn sử dụng chúng. Một sự thay thế cho việc sử dụng semaphore là record locking, mà chúng ta mô tả trong Chương 55. Ngoài ra, từ kernel 2.6 trở đi, Linux hỗ trợ việc sử dụng POSIX semaphore để đồng bộ hóa process. Chúng ta mô tả POSIX semaphore trong Chương 53.

# **47.12 Tóm Tắt**

Semaphore System V cho phép các process đồng bộ hóa các hành động của họ. Điều này hữu ích khi một process phải có quyền truy cập độc quyền vào một tài nguyên chia sẻ nào đó, chẳng hạn như một vùng của shared memory.

Semaphore được tạo và thao tác trong các set chứa một hoặc nhiều semaphore. Mỗi semaphore trong một set là một số nguyên có giá trị luôn lớn hơn hoặc bằng 0. System call `semop()` cho phép caller thêm một số nguyên vào một semaphore, trừ một số nguyên khỏi một semaphore, hoặc chờ một semaphore bằng 0. Hai thao tác cuối trong số này có thể khiến caller bị block.

Một hệ thống triển khai không bắt buộc phải khởi tạo các thành viên của một semaphore set mới, vì vậy một ứng dụng phải khởi tạo set sau khi tạo nó. Khi bất kỳ số process peer nào có thể cố gắng tạo và khởi tạo semaphore, cần phải cẩn thận đặc biệt để tránh race condition xảy ra do thực tế là hai bước này được thực hiện qua các system call riêng biệt.

Khi nhiều process đang cố gắng giảm một semaphore theo cùng một lượng, không thể xác định process nào thực sự được phép thực hiện thao tác trước. Tuy nhiên, khi các process khác nhau đang cố gắng giảm một semaphore theo các lượng khác nhau, các thao tác hoàn thành theo thứ tự mà chúng trở nên khả thi, và chúng ta có thể cần cẩn thận để tránh các tình huống trong đó một process bị starvation vì giá trị semaphore không bao giờ đạt đến mức cho phép thao tác của process tiếp tục.

Flag `SEM_UNDO` cho phép các thao tác semaphore của một process được tự động hoàn tác khi process kết thúc. Điều này có thể hữu ích để tránh các tình huống trong đó một process kết thúc vô tình, để lại một semaphore ở trạng thái khiến các process khác block vô hạn chờ giá trị semaphore được thay đổi bởi process đã kết thúc.

Semaphore System V được phân bổ và thao tác trong các set, và có thể được tăng và giảm theo các lượng tùy ý. Điều này cung cấp nhiều chức năng hơn mức cần thiết bởi hầu hết các ứng dụng. Một yêu cầu phổ biến là cho các binary semaphore riêng lẻ, chỉ nhận các giá trị 0 và 1. Chúng ta đã chỉ ra cách triển khai binary semaphore trên cơ sở semaphore System V.

## **Tài liệu tham khảo thêm**

[Bovet & Cesati, 2005] và [Maxwell, 1999] cung cấp một số thông tin cơ bản về việc triển khai semaphore trên Linux. [Dijkstra, 1968] là một bài báo kinh điển ban đầu về lý thuyết semaphore.

# **47.13 Bài Tập**

- **47-1.** Thử nghiệm với chương trình trong Listing 47-8 (svsem\_op.c) để xác nhận hiểu biết của bạn về system call `semop()`.
- **47-2.** Sửa đổi chương trình trong Listing 24-6 (fork\_sig\_sync.c, trang 528) để sử dụng semaphore thay vì signal để đồng bộ hóa các process cha và con.
- **47-3.** Thử nghiệm với chương trình trong Listing 47-8 (svsem\_op.c) và các chương trình semaphore khác được cung cấp trong chương này để xem điều gì xảy ra với giá trị `sempid` nếu một process đang thoát thực hiện điều chỉnh `SEM_UNDO` cho một semaphore.
- **47-4.** Thêm hàm `reserveSemNB()` vào code trong Listing 47-10 (binary\_sems.c) để triển khai thao tác reserve conditionally, sử dụng flag `IPC_NOWAIT`.

- **47-5.** Đối với hệ điều hành VMS, Digital đã cung cấp một phương pháp đồng bộ hóa tương tự như binary semaphore, gọi là event flag. Một event flag có hai giá trị có thể, clear và set, và bốn thao tác sau có thể được thực hiện: `setEventFlag`, để đặt flag; `clearEventFlag`, để xóa flag; `waitForEventFlag`, để block cho đến khi flag được đặt; và `getFlagState`, để lấy trạng thái hiện tại của flag. Đề xuất một cài đặt event flag bằng semaphore System V. Cài đặt này sẽ yêu cầu hai đối số cho mỗi hàm trên: một semaphore identifier và một semaphore number. (Việc xem xét thao tác `waitForEventFlag` sẽ dẫn bạn đến nhận ra rằng các giá trị được chọn cho clear và set không phải là các lựa chọn rõ ràng.)
- **47-6.** Triển khai một giao thức binary semaphore bằng named pipe. Cung cấp các hàm để reserve, release và conditionally reserve semaphore.
- **47-7.** Viết một chương trình, tương tự như chương trình trong Listing 46-6 (svmsg\_ls.c, trang 953), sử dụng các thao tác `SEM_INFO` và `SEM_STAT` của `semctl()` để lấy và hiển thị danh sách tất cả các semaphore set trong hệ thống.
