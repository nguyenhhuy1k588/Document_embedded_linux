## Chương 27
# Thực Thi Chương Trình

Chương này tiếp nối phần thảo luận về tạo process và kết thúc process trong các chương trước. Bây giờ chúng ta xem xét cách một process có thể sử dụng system call `execve()` để thay thế chương trình đang chạy bằng một chương trình hoàn toàn mới. Sau đó chúng ta trình bày cách cài đặt hàm `system()`, cho phép người gọi thực thi một lệnh shell tùy ý.

## **27.1 Thực Thi Chương Trình Mới: execve()**

System call `execve()` tải một chương trình mới vào bộ nhớ của một process. Trong quá trình này, chương trình cũ bị loại bỏ, và stack, data, và heap của process được thay thế bằng những phần tương ứng của chương trình mới. Sau khi thực thi các đoạn code khởi tạo runtime của thư viện C và code khởi tạo chương trình (ví dụ: các static constructor của C++ hoặc các hàm C được khai báo với thuộc tính constructor của gcc mô tả trong Mục 42.4), chương trình mới bắt đầu thực thi tại hàm `main()` của nó.

Cách sử dụng phổ biến nhất của `execve()` là trong process con được tạo bởi `fork()`, mặc dù đôi khi nó cũng được dùng trong các ứng dụng mà không có lời gọi `fork()` trước đó.

Nhiều hàm thư viện, tất cả đều có tên bắt đầu bằng exec, được xây dựng trên đỉnh của system call `execve()`. Mỗi hàm trong số này cung cấp một interface khác nhau cho cùng một chức năng. Việc tải một chương trình mới bởi bất kỳ lời gọi nào trong số này thường được gọi là thao tác exec, hoặc đơn giản là ký hiệu `exec()`. Chúng ta bắt đầu với mô tả về `execve()` rồi sau đó mô tả các hàm thư viện.

```c
#include <unistd.h>

int execve(const char *pathname, char *const argv[], char *const envp[]);
```

*Không bao giờ trả về khi thành công; trả về –1 khi có lỗi*

Đối số `pathname` chứa đường dẫn của chương trình mới được tải vào bộ nhớ của process. Đường dẫn này có thể là tuyệt đối (bắt đầu bằng /) hoặc tương đối so với thư mục làm việc hiện tại của process gọi.

Đối số `argv` chỉ định các đối số dòng lệnh được truyền cho chương trình mới. Mảng này tương ứng với, và có cùng dạng với, đối số thứ hai (`argv`) của hàm `main()` trong C; đây là danh sách các con trỏ đến các chuỗi ký tự được kết thúc bằng NULL. Giá trị cung cấp cho `argv[0]` tương ứng với tên lệnh. Thông thường giá trị này giống với basename (tức là thành phần cuối cùng) của `pathname`.

Đối số cuối cùng, `envp`, chỉ định danh sách environment cho chương trình mới. Đối số `envp` tương ứng với mảng `environ` của chương trình mới; đây là danh sách các con trỏ đến các chuỗi ký tự có dạng `name=value` được kết thúc bằng NULL (Mục 6.7).

> File `/proc/PID/exe` đặc thù của Linux là một symbolic link chứa đường dẫn tuyệt đối của file thực thi đang được chạy bởi process tương ứng.

Sau một lệnh `execve()`, process ID của process vẫn không thay đổi, vì cùng một process tiếp tục tồn tại. Một số thuộc tính khác của process cũng không thay đổi, như mô tả trong Mục 28.4.

Nếu bit quyền set-user-ID (set-group-ID) của file chương trình được chỉ định bởi `pathname` được đặt, thì khi file được exec, effective user (group) ID của process được thay đổi để giống với owner (group) của file chương trình. Đây là cơ chế để tạm thời cấp quyền cho người dùng khi chạy một chương trình cụ thể (xem Mục 9.3).

Sau khi tùy chọn thay đổi các effective ID, và bất kể chúng có được thay đổi hay không, `execve()` sao chép giá trị effective user ID của process vào saved set-user-ID, và sao chép giá trị effective group ID của process vào saved set-group-ID.

Vì nó thay thế chương trình đã gọi nó, một lệnh `execve()` thành công sẽ không bao giờ trả về. Chúng ta không bao giờ cần kiểm tra giá trị trả về từ `execve()`; nó sẽ luôn là –1. Chính việc nó trả về cho chúng ta biết đã xảy ra lỗi, và như thường lệ, chúng ta có thể dùng `errno` để xác định nguyên nhân. Trong số các lỗi có thể được trả về trong `errno` có những lỗi sau:

#### EACCES

Đối số `pathname` không tham chiếu đến một file thông thường, file không có quyền thực thi được bật, hoặc một trong các thành phần thư mục của `pathname` không thể tìm kiếm (tức là quyền thực thi bị từ chối trên thư mục). Ngoài ra, file nằm trên một file system được mount với cờ `MS_NOEXEC` (Mục 14.8.1).

ENOENT

File được tham chiếu bởi `pathname` không tồn tại.

ENOEXEC

File được tham chiếu bởi `pathname` được đánh dấu là có thể thực thi, nhưng nó không ở định dạng thực thi có thể nhận biết được. Có thể đây là một script không bắt đầu bằng một dòng (bắt đầu với các ký tự `#!`) chỉ định trình thông dịch script.

ETXTBSY

File được tham chiếu bởi `pathname` đang được mở để ghi bởi một process khác (Mục 4.3.2).

E2BIG

Tổng không gian cần thiết cho danh sách đối số và danh sách environment vượt quá giới hạn tối đa được phép.

Các lỗi liệt kê ở trên cũng có thể được tạo ra nếu bất kỳ điều kiện nào trong số này áp dụng cho file trình thông dịch được định nghĩa để thực thi một script (tham khảo Mục 27.3) hoặc cho trình thông dịch ELF đang được sử dụng để thực thi chương trình.

> Định dạng Executable and Linking Format (ELF) là một đặc tả được triển khai rộng rãi mô tả bố cục của các file thực thi. Thông thường, trong quá trình exec, một process image được xây dựng bằng cách sử dụng các segment của file thực thi (Mục 6.3). Tuy nhiên, đặc tả ELF cũng cho phép một file thực thi định nghĩa một trình thông dịch (phần tử program header ELF `PT_INTERP`) để được sử dụng để thực thi chương trình. Nếu một trình thông dịch được định nghĩa, kernel xây dựng process image từ các segment của file thực thi trình thông dịch được chỉ định. Sau đó, trình thông dịch có trách nhiệm tải và thực thi chương trình. Chúng ta nói thêm một chút về trình thông dịch ELF trong Chương 41 và cung cấp một số con trỏ đến thông tin thêm trong chương đó.

### **Chương trình ví dụ**

Listing 27-1 minh họa cách sử dụng `execve()`. Chương trình này tạo một danh sách đối số và một environment cho một chương trình mới, và sau đó gọi `execve()`, sử dụng đối số dòng lệnh của nó (`argv[1]`) làm pathname để thực thi.

Listing 27-2 hiển thị một chương trình được thiết kế để được thực thi bởi chương trình trong Listing 27-1. Chương trình này chỉ đơn giản hiển thị các đối số dòng lệnh và danh sách environment của nó (danh sách sau được truy cập bằng biến toàn cục `environ`, như mô tả trong Mục 6.7).

Phiên shell sau đây minh họa việc sử dụng các chương trình trong Listing 27-1 và Listing 27-2 (trong ví dụ này, một đường dẫn tương đối được sử dụng để chỉ định chương trình cần được exec):

```
$ ./t_execve ./envargs
argv[0] = envargs    Tất cả output được in bởi envargs
argv[1] = hello world
argv[2] = goodbye
environ: GREET=salut
environ: BYE=adieu
```

```
Listing 27-1: Sử dụng execve() để thực thi một chương trình mới
```

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execve.c
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    char *argVec[10]; /* Lớn hơn cần thiết */
    char *envVec[] = { "GREET=salut", "BYE=adieu", NULL };
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s pathname\n", argv[0]);
    argVec[0] = strrchr(argv[1], '/'); /* Lấy basename từ argv[1] */
    if (argVec[0] != NULL)
        argVec[0]++;
    else
        argVec[0] = argv[1];
    argVec[1] = "hello world";
    argVec[2] = "goodbye";
    argVec[3] = NULL; /* Danh sách phải kết thúc bằng NULL */
    execve(argv[1], argVec, envVec);
    errExit("execve"); /* Nếu đến đây, có gì đó đã sai */
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execve.c
```

```c
Listing 27-2: Hiển thị danh sách đối số và environment
––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/envargs.c
#include "tlpi_hdr.h"
extern char **environ;
int
main(int argc, char *argv[])
{
    int j;
    char **ep;
    for (j = 0; j < argc; j++)
        printf("argv[%d] = %s\n", j, argv[j]);
    for (ep = environ; *ep != NULL; ep++)
        printf("environ: %s\n", *ep);
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/envargs.c
```

## **27.2 Các Hàm Thư Viện exec()**

Các hàm thư viện được mô tả trong phần này cung cấp các API thay thế để thực hiện một `exec()`. Tất cả các hàm này được xây dựng trên đỉnh của `execve()`, và chúng chỉ khác nhau với nhau và với `execve()` ở cách chỉ định tên chương trình, danh sách đối số, và environment của chương trình mới.

```c
#include <unistd.h>
int execle(const char *pathname, const char *arg, ...
            /* , (char *) NULL, char *const envp[] */ );
int execlp(const char *filename, const char *arg, ...
            /* , (char *) NULL */);
int execvp(const char *filename, char *const argv[]);
int execv(const char *pathname, char *const argv[]);
int execl(const char *pathname, const char *arg, ...
           /* , (char *) NULL */);
          Không hàm nào trả về khi thành công; tất cả trả về –1 khi có lỗi
```

Các chữ cái cuối trong tên của các hàm này cung cấp gợi ý về sự khác biệt giữa chúng. Những sự khác biệt này được tóm tắt trong Bảng 27-1 và được trình bày chi tiết trong danh sách sau:

- Hầu hết các hàm `exec()` kỳ vọng một pathname là đặc tả của chương trình mới cần tải. Tuy nhiên, `execlp()` và `execvp()` cho phép chỉ định chương trình chỉ bằng tên file. Tên file được tìm kiếm trong danh sách các thư mục được chỉ định trong biến environment `PATH` (giải thích chi tiết hơn bên dưới). Đây là loại tìm kiếm mà shell thực hiện khi được cung cấp một tên lệnh. Để chỉ ra sự khác biệt này trong hoạt động, tên của các hàm này chứa chữ cái p (cho PATH). Biến `PATH` không được sử dụng nếu tên file chứa dấu gạch chéo (/), trong trường hợp đó nó được coi là đường dẫn tương đối hoặc tuyệt đối.

- Thay vì sử dụng một mảng để chỉ định danh sách `argv` cho chương trình mới, `execle()`, `execlp()`, và `execl()` yêu cầu lập trình viên chỉ định các đối số như một danh sách các chuỗi trong lời gọi hàm. Đối số đầu tiên trong số này tương ứng với `argv[0]` trong hàm main của chương trình mới, và vì vậy thông thường giống với đối số `filename` hoặc thành phần basename của đối số `pathname`. Một con trỏ NULL phải kết thúc danh sách đối số, để các lời gọi này có thể xác định phần cuối của danh sách. (Yêu cầu này được chỉ ra bởi comment `(char *) NULL` trong các prototype ở trên; để thảo luận về lý do tại sao cần ép kiểu trước NULL, xem Phụ lục C.) Tên của các hàm chứa chữ cái l (cho list) để phân biệt chúng với các hàm yêu cầu danh sách đối số như một mảng kết thúc bằng NULL. Tên của các hàm yêu cầu danh sách đối số như một mảng (`execve()`, `execvp()`, và `execv()`) chứa chữ cái v (cho vector).

  Các hàm `execve()` và `execle()` cho phép lập trình viên chỉ định tường minh environment cho chương trình mới bằng `envp`, một mảng con trỏ đến các chuỗi ký tự kết thúc bằng NULL. Tên của các hàm này kết thúc bằng chữ cái e (cho environment) để chỉ ra điều này. Tất cả các hàm `exec()` khác sử dụng environment hiện có của người gọi (tức là nội dung của `environ`) làm environment cho chương trình mới.

> Phiên bản 2.11 của glibc đã thêm một hàm không chuẩn, `execvpe(file, argv, envp)`. Hàm này giống như `execvp()`, nhưng thay vì lấy environment cho chương trình mới từ `environ`, người gọi chỉ định environment mới thông qua đối số `envp` (giống như `execve()` và `execle()`).

Trong vài trang tiếp theo, chúng ta minh họa việc sử dụng một số biến thể `exec()` này.

| Hàm | Chỉ định file chương trình (–, p) | Chỉ định đối số (v, l) | Nguồn environment (e, –) |
|-----|------------------------------------|------------------------|--------------------------|
| `execve()` | pathname | mảng | đối số envp |
| `execle()` | pathname | danh sách | đối số envp |
| `execlp()` | filename + PATH | danh sách | environ của người gọi |
| `execvp()` | filename + PATH | mảng | environ của người gọi |
| `execv()` | pathname | mảng | environ của người gọi |
| `execl()` | pathname | danh sách | environ của người gọi |

**Bảng 27-1:** Tóm tắt sự khác biệt giữa các hàm `exec()`

## **27.2.1 Biến Environment PATH**

Các hàm `execvp()` và `execlp()` cho phép chúng ta chỉ định tên file cần thực thi. Các hàm này sử dụng biến environment `PATH` để tìm kiếm file. Giá trị của `PATH` là một chuỗi gồm các tên thư mục được phân cách bằng dấu hai chấm, được gọi là các path prefix. Ví dụ, giá trị `PATH` sau đây chỉ định năm thư mục:

#### $ **echo $PATH**
/home/mtk/bin:/usr/local/bin:/usr/bin:/bin:.

Giá trị `PATH` cho một login shell được đặt bởi các script khởi động shell toàn hệ thống và theo người dùng cụ thể. Vì một process con kế thừa một bản sao của các biến environment của process cha, mỗi process mà shell tạo ra để thực thi một lệnh kế thừa một bản sao `PATH` của shell.

Các đường dẫn thư mục được chỉ định trong `PATH` có thể là tuyệt đối (bắt đầu bằng /) hoặc tương đối. Một đường dẫn tương đối được giải thích liên quan đến thư mục làm việc hiện tại của process gọi. Thư mục làm việc hiện tại có thể được chỉ định bằng cách dùng `.` (chấm), như trong ví dụ ở trên.

> Cũng có thể chỉ định thư mục làm việc hiện tại bằng cách bao gồm một prefix có độ dài bằng không trong `PATH`, bằng cách sử dụng các dấu hai chấm liên tiếp, một dấu hai chấm đầu, hoặc một dấu hai chấm cuối (ví dụ: `/usr/bin:/bin:`). SUSv3 tuyên bố kỹ thuật này là lỗi thời; thư mục làm việc hiện tại nên được chỉ định tường minh bằng cách dùng `.` (chấm).

Nếu biến `PATH` không được định nghĩa, thì `execvp()` và `execlp()` giả định một danh sách path mặc định là `.:/usr/bin:/bin`.

Là một biện pháp bảo mật, tài khoản superuser (root) thường được thiết lập sao cho thư mục làm việc hiện tại bị loại khỏi `PATH`. Điều này ngăn root vô tình thực thi một file từ thư mục làm việc hiện tại (có thể đã được cố tình đặt ở đó bởi một người dùng ác ý) với cùng tên như một lệnh chuẩn hoặc với tên là sai chính tả của một lệnh phổ biến (ví dụ: `sl` thay vì `ls`). Trong một số bản phân phối Linux, giá trị mặc định cho `PATH` cũng loại trừ thư mục làm việc hiện tại cho các người dùng không có đặc quyền. Chúng ta giả định định nghĩa `PATH` như vậy trong tất cả các nhật ký phiên shell được hiển thị trong cuốn sách này, đó là lý do tại sao chúng ta luôn thêm tiền tố `./` vào tên của các chương trình được thực thi từ thư mục làm việc hiện tại.

Các hàm `execvp()` và `execlp()` tìm kiếm tên file trong mỗi thư mục được đặt tên trong `PATH`, bắt đầu từ đầu danh sách và tiếp tục cho đến khi một file với tên đã cho được exec thành công. Việc sử dụng biến environment `PATH` theo cách này hữu ích nếu chúng ta không biết vị trí runtime của một file thực thi hoặc không muốn tạo sự phụ thuộc hardcode vào vị trí đó.

Việc sử dụng `execvp()` và `execlp()` trong các chương trình set-user-ID hoặc set-group-ID nên tránh, hoặc ít nhất phải tiếp cận với sự cẩn trọng lớn. Đặc biệt, biến environment `PATH` nên được kiểm soát cẩn thận để ngăn việc exec một chương trình độc hại.

Listing 27-3 cung cấp một ví dụ về cách sử dụng `execlp()`. Nhật ký phiên shell sau đây minh họa việc sử dụng chương trình này để gọi lệnh echo (`/bin/echo`):

```
$ which echo
/bin/echo
$ ls -l /bin/echo
-rwxr-xr-x 1 root 15428 Mar 19 21:28 /bin/echo
$ echo $PATH    Hiển thị nội dung của biến environment PATH
/home/mtk/bin:/usr/local/bin:/usr/bin:/bin   /bin có trong PATH
$ ./t_execlp echo    execlp() sử dụng PATH để tìm thành công echo
hello world
```

Chuỗi hello world xuất hiện ở trên được cung cấp như đối số thứ ba của lời gọi đến `execlp()` trong chương trình Listing 27-3.

Chúng ta tiếp tục bằng cách tái định nghĩa `PATH` để bỏ qua `/bin`, là thư mục chứa chương trình echo:

```
$ PATH=/home/mtk/bin:/usr/local/bin:/usr/bin
$ ./t_execlp echo
ERROR [ENOENT No such file or directory] execlp
$ ./t_execlp /bin/echo
hello world
```

Như có thể thấy, khi chúng ta cung cấp một tên file (tức là một chuỗi không chứa dấu gạch chéo) cho `execlp()`, lời gọi thất bại, vì một file có tên echo không được tìm thấy trong bất kỳ thư mục nào được liệt kê trong `PATH`. Mặt khác, khi chúng ta cung cấp một đường dẫn chứa một hoặc nhiều dấu gạch chéo, `execlp()` bỏ qua nội dung của `PATH`.

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execlp.c
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s pathname\n", argv[0]);
    execlp(argv[1], argv[1], "hello world", (char *) NULL);
    errExit("execlp"); /* Nếu đến đây, có gì đó đã sai */
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execlp.c
```

## **27.2.2 Chỉ Định Đối Số Chương Trình Như Một Danh Sách**

Khi chúng ta biết số lượng đối số cho một `exec()` tại thời điểm viết chương trình, chúng ta có thể sử dụng `execle()`, `execlp()`, hoặc `execl()` để chỉ định các đối số như một danh sách trong lời gọi hàm. Điều này có thể thuận tiện, vì nó đòi hỏi ít code hơn so với việc lắp ráp các đối số trong một vector `argv`. Chương trình trong Listing 27-4 đạt được cùng kết quả như chương trình trong Listing 27-1 nhưng sử dụng `execle()` thay vì `execve()`.

**Listing 27-4:** Sử dụng `execle()` để chỉ định đối số chương trình như một danh sách

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execle.c
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    char *envVec[] = { "GREET=salut", "BYE=adieu", NULL };
    char *filename;
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s pathname\n", argv[0]);
    filename = strrchr(argv[1], '/'); /* Lấy basename từ argv[1] */
    if (filename != NULL)
        filename++;
    else
        filename = argv[1];
    execle(argv[1], filename, "hello world", (char *) NULL, envVec);
    errExit("execle"); /* Nếu đến đây, có gì đó đã sai */
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execle.c
```

## **27.2.3 Truyền Environment Của Người Gọi Cho Chương Trình Mới**

Các hàm `execlp()`, `execvp()`, `execl()`, và `execv()` không cho phép lập trình viên chỉ định tường minh một danh sách environment; thay vào đó, chương trình mới kế thừa environment của nó từ process gọi (Mục 6.7). Điều này có thể hoặc không thể là mong muốn. Vì lý do bảo mật, đôi khi tốt hơn là đảm bảo rằng một chương trình được exec với một danh sách environment đã biết. Chúng ta xem xét điểm này thêm trong Mục 38.8.

Listing 27-5 minh họa rằng chương trình mới kế thừa environment của nó từ người gọi trong một lời gọi `execl()`. Chương trình này trước tiên sử dụng `putenv()` để thay đổi environment mà nó kế thừa từ shell như là kết quả của `fork()`. Sau đó chương trình `printenv` được exec để hiển thị giá trị của các biến environment `USER` và `SHELL`. Khi chúng ta chạy chương trình này, chúng ta thấy như sau:

```
$ echo $USER $SHELL    Hiển thị một số biến environment của shell
blv /bin/bash
$ ./t_execl
Initial value of USER: blv    Bản sao environment được kế thừa từ shell
britta                         Hai dòng này được hiển thị bởi printenv được exec
/bin/bash
```

**Listing 27-5:** Truyền environment của người gọi cho chương trình mới bằng cách sử dụng `execl()`

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execl.c
#include <stdlib.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    printf("Initial value of USER: %s\n", getenv("USER"));
    if (putenv("USER=britta") != 0)
        errExit("putenv");
    execl("/usr/bin/printenv", "printenv", "USER", "SHELL", (char *) NULL);
    errExit("execl"); /* Nếu đến đây, có gì đó đã sai */
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_execl.c
```

## **27.2.4 Thực Thi File Được Tham Chiếu Bởi Một Descriptor: fexecve()**

Kể từ phiên bản 2.3.2, glibc cung cấp `fexecve()`, hoạt động giống như `execve()`, nhưng chỉ định file cần được exec thông qua file descriptor mở `fd`, thay vì như một pathname. Việc sử dụng `fexecve()` hữu ích cho các ứng dụng muốn mở một file, xác minh nội dung của nó bằng cách thực hiện checksum, và sau đó thực thi file.

```c
#define _GNU_SOURCE
#include <unistd.h>
int fexecve(int fd, char *const argv[], char *const envp[]);
                              Không trả về khi thành công; trả về –1 khi có lỗi
```

Nếu không có `fexecve()`, chúng ta có thể `open()` và `read()` file để xác minh nội dung của nó, và sau đó exec nó. Tuy nhiên, điều này sẽ cho phép khả năng rằng, giữa thời điểm mở file và exec nó, file đã bị thay thế (giữ một file descriptor mở không ngăn một file mới với cùng tên được tạo ra), do đó nội dung được exec sẽ khác với nội dung đã được kiểm tra.

## **27.3 Script Trình Thông Dịch**

Một trình thông dịch là một chương trình đọc các lệnh ở dạng văn bản và thực thi chúng. (Điều này tương phản với một compiler, dịch source code đầu vào thành ngôn ngữ máy có thể sau đó được thực thi trên một máy thực hoặc ảo.) Ví dụ về trình thông dịch bao gồm các UNIX shell khác nhau và các chương trình như awk, sed, perl, python, và ruby. Ngoài khả năng đọc và thực thi các lệnh tương tác, các trình thông dịch thường cung cấp một cơ sở để đọc và thực thi các lệnh từ một file văn bản, được gọi là script.

Các kernel UNIX cho phép các script trình thông dịch được chạy theo cùng cách như một file chương trình nhị phân, miễn là hai yêu cầu được đáp ứng. Đầu tiên, quyền thực thi phải được bật cho file script. Thứ hai, file phải chứa một dòng đầu tiên chỉ định đường dẫn của trình thông dịch sẽ được sử dụng để chạy script. Dòng này có dạng sau:

#### #! interpreter-path [ optional-arg ]

Các ký tự `#!` phải được đặt ở đầu dòng; tùy chọn, một khoảng trắng có thể ngăn cách các ký tự này với đường dẫn trình thông dịch. Biến environment `PATH` không được sử dụng trong việc giải thích đường dẫn này, do đó một đường dẫn tuyệt đối thường nên được chỉ định. Một đường dẫn tương đối cũng có thể, mặc dù không phổ biến; nó được giải thích liên quan đến thư mục làm việc hiện tại của process bắt đầu trình thông dịch. Khoảng trắng phân cách đường dẫn trình thông dịch khỏi một đối số tùy chọn, mà mục đích của nó chúng ta giải thích ngay dưới đây. Đối số tùy chọn không nên chứa các ký tự khoảng trắng.

Ví dụ, các UNIX shell script thường bắt đầu bằng dòng sau, chỉ định rằng shell được sử dụng để thực thi script:

```
#!/bin/sh
```

Đối số tùy chọn trong dòng đầu tiên của file script trình thông dịch không nên chứa khoảng trắng vì hành vi trong trường hợp này rất phụ thuộc vào từng cài đặt. Trên Linux, khoảng trắng trong optional-arg không được diễn giải đặc biệt — tất cả văn bản từ đầu đối số đến cuối dòng được diễn giải như một từ duy nhất (được cung cấp như một đối số cho script, như chúng ta mô tả bên dưới).

Linux kernel đặt giới hạn 127 ký tự cho độ dài của dòng `#!` của một script (không bao gồm ký tự dòng mới ở cuối dòng). Các ký tự bổ sung bị bỏ qua một cách âm thầm.

Kỹ thuật `#!` cho script trình thông dịch không được chỉ định trong SUSv3, nhưng có sẵn trên hầu hết các cài đặt UNIX.

#### **Thực thi script trình thông dịch**

Vì một script không chứa mã máy nhị phân, khi `execve()` được sử dụng để chạy script, rõ ràng có điều gì đó khác với thông thường xảy ra khi script được thực thi. Nếu `execve()` phát hiện rằng file mà nó được cấp bắt đầu với chuỗi 2 byte `#!`, thì nó trích xuất phần còn lại của dòng (pathname và đối số), và exec file trình thông dịch với danh sách đối số sau:

```
interpreter-path [ optional-arg ] script-path arg...
```

Ở đây, `interpreter-path` và `optional-arg` được lấy từ dòng `#!` của script, `script-path` là pathname được cung cấp cho `execve()`, và `arg...` là danh sách bất kỳ đối số thêm nào được chỉ định thông qua đối số `argv` cho `execve()` (nhưng không bao gồm `argv[0]`). Nguồn gốc của mỗi đối số script được tóm tắt trong Hình 27-1.

**Hình 27-1:** Danh sách đối số được cung cấp cho một script được exec

Chúng ta có thể minh họa nguồn gốc của các đối số trình thông dịch bằng cách viết một script sử dụng chương trình trong Listing 6-2 (necho.c) như một trình thông dịch. Chương trình này chỉ đơn giản echo tất cả các đối số dòng lệnh của nó. Sau đó chúng ta sử dụng chương trình trong Listing 27-1 để exec script:

```
$ cat > necho.script    Tạo script
#!/home/mtk/bin/necho some argument
Some junk
Nhấn Control-D
$ chmod +x necho.script    Làm script có thể thực thi
$ ./t_execve necho.script    Và exec script
argv[0] = /home/mtk/bin/necho    Ba đối số đầu tiên được tạo bởi kernel
argv[1] = some argument          Đối số script được coi như một từ duy nhất
argv[2] = necho.script           Đây là đường dẫn script
argv[3] = hello world            Đây là argVec[1] được cung cấp cho execve()
argv[4] = goodbye                Và đây là argVec[2]
```

Trong ví dụ này, "trình thông dịch" của chúng ta (necho) bỏ qua nội dung của file script của nó (necho.script), và dòng thứ hai của script (Some junk) không có hiệu ứng trên việc thực thi nó.

> Kernel Linux 2.2 chỉ truyền phần basename của interpreter-path như đối số đầu tiên khi gọi một script. Do đó, trên Linux 2.2, dòng hiển thị `argv[0]` sẽ chỉ hiển thị giá trị echo.

Hầu hết các UNIX shell và trình thông dịch coi ký tự `#` là bắt đầu của một comment. Do đó, các trình thông dịch này bỏ qua dòng `#!` đầu tiên khi diễn giải script.

#### **Sử dụng optional-arg của script**

Một cách sử dụng của optional-arg trong dòng `#!` đầu tiên của script là chỉ định các tùy chọn dòng lệnh cho trình thông dịch. Tính năng này hữu ích với một số trình thông dịch nhất định, chẳng hạn như awk.

> Trình thông dịch awk đã là một phần của hệ thống UNIX từ cuối những năm 1970. Ngôn ngữ awk được mô tả trong một số cuốn sách, bao gồm một cuốn của các tác giả tạo ra nó [Aho et al., 1988], có chữ cái đầu tên họ đã đặt tên cho ngôn ngữ. Điểm mạnh của nó là tạo mẫu nhanh của các ứng dụng xử lý văn bản.

Một script có thể được cung cấp cho awk theo hai cách khác nhau. Mặc định là cung cấp script như đối số dòng lệnh đầu tiên cho awk:

```
$ awk 'script' input-file...
```

Ngoài ra, một awk script có thể nằm bên trong một file, như trong awk script sau, in ra độ dài của dòng dài nhất của đầu vào:

```
$ cat longest_line.awk
#!/usr/bin/awk
length > max { max = length; }
END { print max; }
```

Giả sử chúng ta thử exec script này bằng cách sử dụng đoạn code C sau:

```c
execl("longest_line.awk", "longest_line.awk", "input.txt", (char *) NULL);
```

Lời gọi `execl()` này lần lượt sử dụng `execve()` với danh sách đối số sau để gọi awk:

```
/usr/bin/awk longest_line.awk input.txt
```

Lời gọi `execve()` này thất bại, vì awk diễn giải chuỗi `longest_line.awk` như một script chứa một lệnh awk không hợp lệ. Chúng ta cần một cách để thông báo cho awk rằng đối số này thực sự là tên của một file chứa script. Chúng ta có thể làm điều này bằng cách thêm tùy chọn `-f` như đối số tùy chọn trong dòng `#!` của script. Điều này nói với awk rằng đối số tiếp theo là một file script:

```
#!/usr/bin/awk -f
length > max { max = length; }
END { print max; }
```

Bây giờ, lời gọi `execl()` của chúng ta tạo ra danh sách đối số sau:

```
/usr/bin/awk -f longest_line.awk input.txt
```

Điều này thành công gọi awk sử dụng script `longest_line.awk` để xử lý file `input.txt`.

#### **Thực thi script với execlp() và execvp()**

Thông thường, việc thiếu dòng `#!` ở đầu của một script khiến các hàm `exec()` thất bại. Tuy nhiên, `execlp()` và `execvp()` làm mọi thứ có phần khác. Nhớ lại rằng đây là các hàm sử dụng biến environment `PATH` để lấy danh sách các thư mục để tìm kiếm một file cần thực thi. Nếu một trong hai hàm này tìm thấy một file có quyền thực thi được bật, nhưng không phải là một file thực thi nhị phân và không bắt đầu bằng dòng `#!`, thì chúng exec shell để diễn giải file. Trên Linux, điều này có nghĩa là các file như vậy được coi như thể chúng bắt đầu bằng một dòng chứa chuỗi `#!/bin/sh`.

## **27.4 File Descriptor và exec()**

Theo mặc định, tất cả các file descriptor được mở bởi một chương trình gọi `exec()` vẫn mở qua `exec()` và có sẵn để sử dụng bởi chương trình mới. Điều này thường hữu ích, vì chương trình gọi có thể mở các file trên các descriptor cụ thể, và các file này tự động có sẵn cho chương trình mới, không cần biết tên hoặc mở các file.

Shell tận dụng tính năng này để xử lý I/O redirection cho các chương trình mà nó thực thi. Ví dụ, giả sử chúng ta nhập lệnh shell sau:

#### $ **ls /tmp > dir.txt**

Shell thực hiện các bước sau để thực thi lệnh này:

- 1. Một `fork()` được thực hiện để tạo một process con cũng đang chạy một bản sao của shell (và do đó có một bản sao của lệnh).
- 2. Shell con mở `dir.txt` để output sử dụng file descriptor 1 (standard output). Điều này có thể được thực hiện theo một trong hai cách sau:
  - a) Shell con đóng descriptor 1 (`STDOUT_FILENO`) và sau đó mở file `dir.txt`. Vì `open()` luôn sử dụng file descriptor có sẵn thấp nhất, và standard input (descriptor 0) vẫn mở, file sẽ được mở trên descriptor 1.
  - b) Shell mở `dir.txt`, lấy một file descriptor mới. Sau đó, nếu file descriptor đó không phải là standard output, shell sử dụng `dup2()` để buộc standard output là một bản sao của descriptor mới và đóng descriptor mới, vì nó không còn cần thiết nữa. (Phương pháp này an toàn hơn phương pháp trước, vì nó không phụ thuộc vào các descriptor có số thấp hơn đang mở.) Chuỗi code trông giống như sau:

```c
fd = open("dir.txt", O_WRONLY | O_CREAT,
          S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
          /* rw-rw-rw- */
if (fd != STDOUT_FILENO) {
    dup2(fd, STDOUT_FILENO);
    close(fd);
}
```

3. Shell con exec chương trình `ls`. Chương trình `ls` ghi output của nó ra standard output, là file `dir.txt`.

> Giải thích được đưa ra ở đây về cách shell thực hiện I/O redirection đơn giản hóa một số điểm. Đặc biệt, một số lệnh—gọi là các lệnh shell built-in—được thực thi trực tiếp bởi shell, mà không thực hiện `fork()` hoặc `exec()`. Các lệnh như vậy phải được xử lý có phần khác nhau cho mục đích I/O redirection.

## **Cờ close-on-exec (FD_CLOEXEC)**

Đôi khi, có thể mong muốn đảm bảo rằng một số file descriptor nhất định được đóng trước một `exec()`. Đặc biệt, nếu chúng ta `exec()` một chương trình không rõ (tức là một chương trình mà chúng ta không viết) từ một process có đặc quyền, hoặc một chương trình không cần descriptor cho các file chúng ta đã mở, thì đây là thực hành lập trình an toàn để đảm bảo tất cả các file descriptor không cần thiết được đóng trước khi chương trình mới được tải. Chúng ta có thể làm điều này bằng cách gọi `close()` trên tất cả các descriptor như vậy, nhưng điều này có các hạn chế sau:

- File descriptor có thể đã được mở bởi một hàm thư viện. Hàm này không có cơ chế để buộc chương trình chính đóng file descriptor trước khi `exec()` được thực hiện. (Như một nguyên tắc chung, các hàm thư viện nên luôn đặt cờ close-on-exec, sử dụng kỹ thuật được mô tả bên dưới, cho bất kỳ file nào mà chúng mở.)

- Nếu lời gọi `exec()` thất bại vì một lý do nào đó, chúng ta có thể muốn giữ các file descriptor mở. Nếu chúng đã đóng, có thể khó khăn hoặc không thể mở lại chúng để chúng tham chiếu đến cùng các file.

Vì những lý do này, kernel cung cấp một cờ close-on-exec cho mỗi file descriptor. Nếu cờ này được đặt, thì file descriptor được tự động đóng trong một `exec()` thành công, nhưng để mở nếu `exec()` thất bại. Cờ close-on-exec cho một file descriptor có thể được truy cập bằng cách sử dụng system call `fcntl()` (Mục 5.2). Thao tác `F_GETFD` của `fcntl()` truy xuất một bản sao của các cờ file descriptor:

```c
int flags;
flags = fcntl(fd, F_GETFD);
if (flags == -1)
    errExit("fcntl");
```

Sau khi truy xuất các cờ này, chúng ta có thể sửa đổi bit `FD_CLOEXEC` và sử dụng lời gọi `fcntl()` thứ hai chỉ định `F_SETFD` để cập nhật các cờ:

```c
flags |= FD_CLOEXEC;
if (fcntl(fd, F_SETFD, flags) == -1)
    errExit("fcntl");
```

`FD_CLOEXEC` thực sự là bit duy nhất được sử dụng trong các cờ file descriptor. Bit này tương ứng với giá trị 1. Trong các chương trình cũ hơn, đôi khi chúng ta có thể thấy cờ close-on-exec được đặt chỉ bằng cách sử dụng lời gọi `fcntl(fd, F_SETFD, 1)`. Theo lý thuyết, điều này có thể không luôn như vậy (trong tương lai, một số hệ thống UNIX có thể triển khai các bit cờ bổ sung), do đó chúng ta nên sử dụng kỹ thuật được hiển thị trong phần chính.

Khi `dup()`, `dup2()`, hoặc `fcntl()` được sử dụng để tạo một bản sao của file descriptor, cờ close-on-exec luôn bị xóa cho descriptor bản sao. (Hành vi này là lịch sử và là yêu cầu của SUSv3.)

Listing 27-6 minh họa việc thao tác cờ close-on-exec. Tùy thuộc vào sự hiện diện của một đối số dòng lệnh (bất kỳ chuỗi nào), chương trình này trước tiên đặt cờ close-on-exec cho standard output và sau đó exec chương trình `ls`. Đây là những gì chúng ta thấy khi chạy chương trình:

```
$ ./closeonexec       Exec ls mà không đóng standard output
-rwxr-xr-x 1 mtk users 28098 Jun 15 13:59 closeonexec
$ ./closeonexec n     Đặt cờ close-on-exec cho standard output
ls: write error: Bad file descriptor
```

Trong lần chạy thứ hai hiển thị ở trên, `ls` phát hiện rằng standard output của nó đã đóng và in một thông báo lỗi trên standard error.

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/closeonexec.c
#include <fcntl.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    int flags;
    if (argc > 1) {
        flags = fcntl(STDOUT_FILENO, F_GETFD); /* Lấy flags */
        if (flags == -1)
            errExit("fcntl - F_GETFD");
        flags |= FD_CLOEXEC; /* Bật FD_CLOEXEC */
        if (fcntl(STDOUT_FILENO, F_SETFD, flags) == -1) /* Cập nhật flags */
            errExit("fcntl - F_SETFD");
    }
    execlp("ls", "ls", "-l", argv[0], (char *) NULL);
    errExit("execlp");
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/closeonexec.c
```

## **27.5 Signal và exec()**

Trong quá trình `exec()`, văn bản của process hiện tại bị loại bỏ. Văn bản này có thể bao gồm các signal handler được thiết lập bởi chương trình gọi. Vì các handler biến mất, kernel đặt lại các disposition của tất cả các signal đã xử lý về `SIG_DFL`. Các disposition của tất cả các signal khác (tức là những signal có disposition là `SIG_IGN` hoặc `SIG_DFL`) không bị thay đổi bởi một `exec()`. Hành vi này được yêu cầu bởi SUSv3.

SUSv3 có một trường hợp đặc biệt cho signal `SIGCHLD` bị bỏ qua. SUSv3 để không xác định liệu một `SIGCHLD` bị bỏ qua có vẫn bị bỏ qua qua một `exec()` hay disposition của nó được đặt lại về `SIG_DFL`. Linux làm theo cách trước, nhưng một số cài đặt UNIX khác (ví dụ: Solaris) làm theo cách sau. Điều này có nghĩa là, trong các chương trình bỏ qua `SIGCHLD`, để tối đa tính di động, chúng ta nên thực hiện lời gọi `signal(SIGCHLD, SIG_DFL)` trước một `exec()`.

Việc phá hủy data, heap, và stack của chương trình cũ cũng có nghĩa là bất kỳ alternate signal stack nào được thiết lập bởi lời gọi đến `sigaltstack()` (Mục 21.3) bị mất. Vì một alternate signal stack không được bảo tồn qua một `exec()`, bit `SA_ONSTACK` cũng bị xóa cho tất cả các signal.

Trong quá trình `exec()`, signal mask và tập hợp các signal đang chờ xử lý của process đều được bảo tồn. Tính năng này cho phép chúng ta block và xếp hàng các signal cho chương trình được exec mới. Tuy nhiên, SUSv3 lưu ý rằng nhiều ứng dụng hiện có sai lầm giả định rằng chúng được bắt đầu với disposition của một số signal nhất định được đặt là `SIG_DFL` hoặc rằng các signal này không bị block. Vì lý do này, SUSv3 khuyến nghị rằng các signal không nên bị block hoặc bỏ qua qua một `exec()` của một chương trình tùy ý.

## **27.6 Thực Thi Một Lệnh Shell: system()**

Hàm `system()` cho phép chương trình gọi thực thi một lệnh shell tùy ý. Trong phần này, chúng ta mô tả hoạt động của `system()`, và trong phần tiếp theo chúng ta hiển thị cách `system()` có thể được triển khai bằng cách sử dụng `fork()`, `exec()`, `wait()`, và `exit()`.

> Trong Mục 44.5, chúng ta xem xét các hàm `popen()` và `pclose()`, cũng có thể được sử dụng để thực thi một lệnh shell, nhưng cho phép chương trình gọi đọc output của lệnh hoặc gửi input cho lệnh.

```c
#include <stdlib.h>
int system(const char *command);
                       Xem phần chính để biết mô tả về giá trị trả về
```

Hàm `system()` tạo một process con gọi một shell để thực thi `command`. Đây là một ví dụ về lời gọi đến `system()`:

```c
system("ls | wc");
```

Các ưu điểm chính của `system()` là sự đơn giản và tiện lợi:

- Chúng ta không cần xử lý chi tiết của việc gọi `fork()`, `exec()`, `wait()`, và `exit()`.
- Xử lý lỗi và signal được thực hiện bởi `system()` thay mặt chúng ta.
- Vì `system()` sử dụng shell để thực thi `command`, tất cả các xử lý shell thông thường, thay thế, và redirection được thực hiện trên `command` trước khi nó được thực thi. Điều này giúp dễ dàng thêm tính năng "thực thi một lệnh shell" vào một ứng dụng. (Nhiều ứng dụng tương tác cung cấp tính năng như vậy dưới dạng lệnh `!`.)

Chi phí chính của `system()` là kém hiệu quả. Thực thi một lệnh bằng cách sử dụng `system()` yêu cầu tạo ít nhất hai process—một cho shell và một hoặc nhiều hơn cho các lệnh nó thực thi—mỗi process thực hiện một `exec()`. Nếu hiệu quả hoặc tốc độ là yêu cầu, tốt hơn nên sử dụng các lời gọi `fork()` và `exec()` rõ ràng để thực thi chương trình mong muốn.

Giá trị trả về của `system()` như sau:

- Nếu `command` là một con trỏ NULL, thì `system()` trả về một giá trị khác không nếu có shell, và 0 nếu không có shell. Trường hợp này phát sinh từ các tiêu chuẩn ngôn ngữ lập trình C, không gắn với bất kỳ hệ điều hành nào, do đó shell có thể không có sẵn nếu `system()` đang chạy trên một hệ thống không phải UNIX. Hơn nữa, ngay cả khi tất cả các cài đặt UNIX đều có shell, shell này có thể không có sẵn nếu chương trình đã gọi `chroot()` trước khi gọi `system()`. Nếu `command` không phải NULL, thì giá trị trả về cho `system()` được xác định theo các quy tắc còn lại trong danh sách này.

- Nếu một process con không thể được tạo hoặc trạng thái kết thúc của nó không thể được truy xuất, thì `system()` trả về –1.
- Nếu một shell không thể được exec trong process con, thì `system()` trả về một giá trị như thể shell con đã kết thúc với lời gọi `_exit(127)`.
- Nếu tất cả các system call thành công, thì `system()` trả về trạng thái kết thúc của shell con được sử dụng để thực thi `command`. (Trạng thái kết thúc của một shell là trạng thái kết thúc của lệnh cuối cùng nó thực thi.)

Không thể phân biệt (sử dụng giá trị được trả về bởi `system()`) trường hợp `system()` không thể exec shell với trường hợp shell thoát với trạng thái 127.

Trong hai trường hợp cuối, giá trị được trả về bởi `system()` là một trạng thái wait có cùng dạng được trả về bởi `waitpid()`. Điều này có nghĩa là chúng ta nên sử dụng các hàm được mô tả trong Mục 26.1.3 để phân tích giá trị này.

#### **Chương trình ví dụ**

Listing 27-7 minh họa việc sử dụng `system()`. Chương trình này thực thi một vòng lặp đọc một chuỗi lệnh, thực thi nó bằng cách sử dụng `system()`, và sau đó phân tích và hiển thị giá trị được trả về bởi `system()`. Đây là một ví dụ chạy:

```
$ ./t_system
Command: whoami
mtk
system() returned: status=0x0000 (0,0)
child exited, status=0
Command: ls | grep XYZ    Shell kết thúc với trạng thái của...
system() returned: status=0x0100 (1,0)    lệnh cuối cùng (grep), không
child exited, status=1                    tìm thấy kết quả, và do đó exit(1)
Command: exit 127
system() returned: status=0x7f00 (127,0)
(Probably) could not invoke shell    Thực ra, không đúng trong trường hợp này
Command: sleep 100
Nhấn Control-Z để suspend foreground process group
[1]+ Stopped ./t_system
$ ps | grep sleep    Tìm PID của sleep
29361 pts/6    00:00:00 sleep
$ kill 29361    Và gửi signal để kết thúc nó
$ fg    Đưa t_system về foreground
./t_system
system() returned: status=0x000f (0,15)
child killed by signal 15 (Terminated)
Command: ^D$    Nhấn Control-D để kết thúc chương trình
```

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_system.c
#include <sys/wait.h>
#include "print_wait_status.h"
#include "tlpi_hdr.h"
#define MAX_CMD_LEN 200
int
main(int argc, char *argv[])
{
    char str[MAX_CMD_LEN]; /* Lệnh cần được thực thi bởi system() */
    int status;            /* Trạng thái trả về từ system() */
    for (;;) { /* Đọc và thực thi một lệnh shell */
        printf("Command: ");
        fflush(stdout);
        if (fgets(str, MAX_CMD_LEN, stdin) == NULL)
            break; /* end-of-file */
        status = system(str);
        printf("system() returned: status=0x%04x (%d,%d)\n",
               (unsigned int) status, status >> 8, status & 0xff);
        if (status == -1) {
            errExit("system");
        } else {
            if (WIFEXITED(status) && WEXITSTATUS(status) == 127)
                printf("(Probably) could not invoke shell\n");
            else /* Shell thực thi thành công lệnh */
                printWaitStatus(NULL, status);
        }
    }
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/t_system.c
```

## **Tránh sử dụng system() trong các chương trình set-user-ID và set-group-ID**

Các chương trình set-user-ID và set-group-ID không bao giờ nên sử dụng `system()` trong khi hoạt động dưới identifier có đặc quyền của chương trình. Ngay cả khi các chương trình như vậy không cho phép người dùng chỉ định văn bản của lệnh cần thực thi, sự phụ thuộc của shell vào các biến environment khác nhau để kiểm soát hoạt động của nó có nghĩa là việc sử dụng `system()` không tránh khỏi mở ra cánh cửa cho một vi phạm bảo mật hệ thống.

Các chương trình an toàn cần sinh ra một chương trình khác nên sử dụng `fork()` và một trong các hàm `exec()`—ngoại trừ `execlp()` hoặc `execvp()`—trực tiếp.

## **27.7 Triển Khai system()**

Trong phần này, chúng ta giải thích cách triển khai `system()`. Chúng ta bắt đầu với một triển khai đơn giản, giải thích những gì còn thiếu trong triển khai đó, và sau đó trình bày một triển khai hoàn chỉnh.

#### **Triển khai đơn giản của system()**

Tùy chọn `-c` của lệnh sh cung cấp một cách dễ dàng để thực thi một chuỗi chứa các lệnh shell tùy ý:

```
$ sh -c "ls | wc"
  38 38 444
```

Do đó, để triển khai `system()`, chúng ta cần sử dụng `fork()` để tạo một child sau đó thực hiện `execl()` với các đối số tương ứng với lệnh sh ở trên:

```c
execl("/bin/sh", "sh", "-c", command, (char *) NULL);
```

Để thu thập trạng thái của child được tạo bởi `system()`, chúng ta sử dụng lời gọi `waitpid()` chỉ định process ID của child. (Việc sử dụng `wait()` sẽ không đủ, vì `wait()` chờ bất kỳ child nào, có thể vô tình thu thập trạng thái của một child nào đó khác được tạo bởi process gọi.) Một triển khai đơn giản, và chưa hoàn chỉnh, của `system()` được hiển thị trong Listing 27-8.

**Listing 27-8:** Một triển khai của `system()` không bao gồm xử lý signal

```c
––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/simple_system.c
#include <unistd.h>
#include <sys/wait.h>
#include <sys/types.h>
int
system(char *command)
{
    int status;
    pid_t childPid;
    switch (childPid = fork()) {
    case -1: /* Lỗi */
        return -1;
    case 0: /* Child */
        execl("/bin/sh", "sh", "-c", command, (char *) NULL);
        _exit(127); /* exec thất bại */
    default: /* Parent */
        if (waitpid(childPid, &status, 0) == -1)
            return -1;
        else
            return status;
    }
}
––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/simple_system.c
```

#### **Xử lý signal đúng cách bên trong system()**

Điều làm phức tạp thêm triển khai của `system()` là việc xử lý đúng các signal.

Signal đầu tiên cần xem xét là `SIGCHLD`. Giả sử rằng chương trình gọi `system()` cũng đang trực tiếp tạo các child, và đã thiết lập một handler cho `SIGCHLD` thực hiện `wait()` riêng của nó. Trong tình huống này, khi một signal `SIGCHLD` được tạo ra bởi sự kết thúc của child được tạo bởi `system()`, có thể rằng handler signal của chương trình chính sẽ được gọi—và thu thập trạng thái của child—trước khi `system()` có cơ hội gọi `waitpid()`. (Đây là một ví dụ về race condition.) Điều này có hai hậu quả không mong muốn:

- Chương trình gọi sẽ bị lừa rằng một trong các child mà nó tạo ra đã kết thúc.
- Hàm `system()` sẽ không thể lấy trạng thái kết thúc của child mà nó tạo ra.

Do đó, `system()` phải block việc gửi `SIGCHLD` trong khi nó đang thực thi.

Các signal khác cần xem xét là những signal được tạo ra bởi ký tự interrupt terminal (thường là Control-C) và quit (thường là Control-\), `SIGINT` và `SIGQUIT`, tương ứng. Xem xét những gì đang xảy ra khi chúng ta thực thi lời gọi sau:

```c
system("sleep 20");
```

Tại thời điểm này, ba process đang chạy: process thực thi chương trình gọi, một shell, và sleep, như được hiển thị trong Hình 27-2.

Tất cả các process được hiển thị trong Hình 27-2 tạo thành một phần của foreground process group cho terminal. Do đó, khi chúng ta gõ ký tự interrupt hoặc quit, tất cả ba process được gửi signal tương ứng. Shell bỏ qua `SIGINT` và `SIGQUIT` trong khi chờ child của nó. Tuy nhiên, cả chương trình gọi lẫn process sleep đều mặc định bị giết bởi các signal này.

SUSv3 chỉ định như sau:

- `SIGINT` và `SIGQUIT` nên bị bỏ qua trong process gọi trong khi lệnh đang được thực thi.
- Trong child, `SIGINT` và `SIGQUIT` nên được coi như chúng sẽ được nếu process gọi thực hiện `fork()` và `exec()`; tức là, disposition của các signal đã xử lý được đặt lại về mặc định, và disposition của các signal khác vẫn không thay đổi.

#### **Triển khai system() được cải thiện**

Listing 27-9 hiển thị một triển khai của `system()` tuân thủ các quy tắc được mô tả ở trên. Lưu ý các điểm sau về triển khai này:

- Như đã lưu ý trước đó, nếu `command` là một con trỏ NULL, thì `system()` nên trả về khác không nếu có shell hoặc 0 nếu không có shell. Cách duy nhất để xác định thông tin này một cách đáng tin cậy là thử thực thi một shell. Chúng ta làm điều này bằng cách gọi đệ quy `system()` để thực thi lệnh shell `:` và kiểm tra trạng thái trả về 0 từ lời gọi đệ quy. Lệnh `:` là một lệnh shell built-in không làm gì, nhưng luôn trả về trạng thái thành công.

- Chỉ trong process cha (người gọi của `system()`) mà `SIGCHLD` cần được block, và `SIGINT` và `SIGQUIT` cần bị bỏ qua. Tuy nhiên, chúng ta phải thực hiện các hành động này trước lời gọi `fork()`, vì, nếu chúng được thực hiện trong cha sau `fork()`, chúng ta sẽ tạo ra một race condition.

- Trong cha, chúng ta bỏ qua lỗi từ các lời gọi `sigaction()` và `sigprocmask()` được sử dụng để thao tác các disposition signal và signal mask. Chúng ta làm điều này vì hai lý do. Đầu tiên, các lời gọi này rất khó thất bại. Thứ hai, chúng ta giả định rằng người gọi quan tâm hơn đến việc biết nếu `fork()` hoặc `waitpid()` thất bại hơn là biết nếu các lời gọi thao tác signal này thất bại.

- Nếu lời gọi `execl()` trong child thất bại, thì chúng ta sử dụng `_exit()` để kết thúc process, thay vì `exit()`, để ngăn việc xả bất kỳ dữ liệu chưa được ghi nào trong bản sao stdio buffers của child.

- Trong cha, chúng ta phải sử dụng `waitpid()` để chờ cụ thể cho child mà chúng ta tạo ra. Nếu chúng ta dùng `wait()`, thì chúng ta có thể vô tình lấy trạng thái của một child khác được tạo bởi chương trình gọi.

**Listing 27-9:** Triển khai `system()`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/system.c
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/types.h>
#include <errno.h>
int
system(const char *command)
{
    sigset_t blockMask, origMask;
    struct sigaction saIgnore, saOrigQuit, saOrigInt, saDefault;
    pid_t childPid;
    int status, savedErrno;

    if (command == NULL) /* Có shell không? */
        return system(":") == 0;

    sigemptyset(&blockMask); /* Block SIGCHLD */
    sigaddset(&blockMask, SIGCHLD);
    sigprocmask(SIG_BLOCK, &blockMask, &origMask);
    saIgnore.sa_handler = SIG_IGN; /* Bỏ qua SIGINT và SIGQUIT */
    saIgnore.sa_flags = 0;
    sigemptyset(&saIgnore.sa_mask);
    sigaction(SIGINT, &saIgnore, &saOrigInt);
    sigaction(SIGQUIT, &saIgnore, &saOrigQuit);

    switch (childPid = fork()) {
    case -1: /* fork() thất bại */
        status = -1;
        break; /* Tiếp tục để đặt lại thuộc tính signal */
    case 0: /* Child: exec lệnh */
        saDefault.sa_handler = SIG_DFL;
        saDefault.sa_flags = 0;
        sigemptyset(&saDefault.sa_mask);
        if (saOrigInt.sa_handler != SIG_IGN)
            sigaction(SIGINT, &saDefault, NULL);
        if (saOrigQuit.sa_handler != SIG_IGN)
            sigaction(SIGQUIT, &saDefault, NULL);
        sigprocmask(SIG_SETMASK, &origMask, NULL);
        execl("/bin/sh", "sh", "-c", command, (char *) NULL);
        _exit(127); /* Chúng ta không thể exec shell */
    default: /* Parent: chờ child của chúng ta kết thúc */
        while (waitpid(childPid, &status, 0) == -1) {
            if (errno != EINTR) { /* Lỗi khác EINTR */
                status = -1;
                break; /* Thoát vòng lặp */
            }
        }
        break;
    }

    /* Unblock SIGCHLD, khôi phục disposition của SIGINT và SIGQUIT */
    savedErrno = errno; /* Phần sau có thể thay đổi 'errno' */
    sigprocmask(SIG_SETMASK, &origMask, NULL);
    sigaction(SIGINT, &saOrigInt, NULL);
    sigaction(SIGQUIT, &saOrigQuit, NULL);
    errno = savedErrno;
    return status;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– procexec/system.c
```

#### **Chi tiết thêm về system()**

Các ứng dụng di động nên đảm bảo rằng `system()` không được gọi khi disposition của `SIGCHLD` được đặt là `SIG_IGN`, vì không thể để lời gọi `waitpid()` lấy trạng thái của child trong trường hợp này. (Bỏ qua `SIGCHLD` khiến trạng thái của một process con bị loại bỏ ngay lập tức, như mô tả trong Mục 26.3.3.)

Trên một số cài đặt UNIX, `system()` xử lý trường hợp nó được gọi với disposition của `SIGCHLD` được đặt là `SIG_IGN` bằng cách tạm thời đặt disposition của `SIGCHLD` thành `SIG_DFL`. Điều này có thể thực hiện được, miễn là cài đặt UNIX là một trong những cài đặt (không giống Linux) thu thập các zombie child hiện có khi disposition của `SIGCHLD` được đặt lại là `SIG_IGN`.

Trên một số cài đặt UNIX (đặc biệt là Solaris), `/bin/sh` không phải là một POSIX shell chuẩn. Nếu chúng ta muốn đảm bảo rằng chúng ta exec một shell chuẩn, thì chúng ta phải sử dụng hàm thư viện `confstr()` để lấy giá trị của biến cấu hình `_CS_PATH`. Giá trị này là một danh sách kiểu PATH của các thư mục chứa các tiện ích hệ thống chuẩn.

## **27.8 Tóm Tắt**

Sử dụng `execve()`, một process có thể thay thế chương trình mà nó đang chạy bằng một chương trình mới. Các đối số cho lời gọi `execve()` cho phép chỉ định danh sách đối số (argv) và danh sách environment cho chương trình mới. Các hàm thư viện có tên tương tự khác nhau được xây dựng trên đỉnh của `execve()` và cung cấp các interface khác nhau cho cùng một chức năng.

Tất cả các hàm `exec()` có thể được sử dụng để tải một file thực thi nhị phân hoặc để thực thi một script trình thông dịch. Khi một process exec một script, chương trình trình thông dịch của script thay thế chương trình hiện đang được thực thi bởi process. Trình thông dịch của script thường được xác định bởi một dòng đầu tiên (bắt đầu bằng các ký tự `#!`) trong script chỉ định đường dẫn của trình thông dịch. Nếu không có dòng như vậy, thì script chỉ có thể thực thi được qua `execlp()` hoặc `execvp()`, và các hàm này exec shell như trình thông dịch script.

Chúng ta đã hiển thị cách `fork()`, `exec()`, `exit()`, và `wait()` có thể được kết hợp để triển khai hàm `system()`, có thể được sử dụng để thực thi một lệnh shell tùy ý.

## **27.9 Bài Tập**

**27-1.** Lệnh cuối cùng trong phiên shell sau sử dụng chương trình trong Listing 27-3 để exec chương trình xyz. Điều gì xảy ra?

```
$ echo $PATH
/usr/local/bin:/usr/bin:/bin:./dir1:./dir2
$ ls -l dir1
total 8
-rw-r--r-- 1 mtk users 7860 Jun 13 11:55 xyz
$ ls -l dir2
total 28
-rwxr-xr-x 1 mtk users 27452 Jun 13 11:55 xyz
$ ./t_execlp xyz
```

**27-2.** Sử dụng `execve()` để triển khai `execlp()`. Bạn sẽ cần sử dụng API `stdarg(3)` để xử lý danh sách đối số có độ dài thay đổi được cung cấp cho `execlp()`. Bạn cũng cần sử dụng các hàm trong gói malloc để cấp phát không gian cho các vector đối số và environment.

**27-3.** Chúng ta sẽ thấy output gì nếu chúng ta làm script sau có thể thực thi và `exec()` nó?

```
#!/bin/cat -n
Hello world
```

**27-4.** Hiệu ứng của đoạn code sau là gì? Trong trường hợp nào nó có thể hữu ích?

```c
childPid = fork();
if (childPid == -1)
    errExit("fork1");
if (childPid == 0) { /* Child */
    switch (fork()) {
    case -1: errExit("fork2");
    case 0: /* Grandchild */
        /* ----- Thực hiện công việc thực sự ở đây ----- */
        exit(EXIT_SUCCESS); /* Sau khi làm xong công việc thực sự */
    default:
        exit(EXIT_SUCCESS); /* Làm grandchild thành orphan */
    }
}
/* Parent tiếp tục đến đây */
if (waitpid(childPid, &status, 0) == -1)
    errExit("waitpid");
/* Parent tiếp tục làm những thứ khác */
```

**27-5.** Khi chúng ta chạy chương trình sau, chúng ta thấy nó không tạo ra output. Tại sao vậy?

```c
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    printf("Hello world");
    execlp("sleep", "sleep", "0", (char *) NULL);
}
```

**27-6.** Giả sử một process cha đã thiết lập một handler cho `SIGCHLD` và cũng đã block signal này. Sau đó, một trong các child của nó thoát, và process cha thực hiện `wait()` để thu thập trạng thái của child. Điều gì xảy ra khi process cha unblock `SIGCHLD`? Viết một chương trình để xác minh câu trả lời của bạn. Kết quả này có liên quan gì đến một chương trình gọi hàm `system()`?
