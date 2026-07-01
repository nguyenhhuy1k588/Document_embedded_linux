## Chương 9
# Thông Tin Xác Thực Của Process

Mỗi process có một tập hợp các định danh số cho người dùng (UID) và nhóm (GID) được liên kết. Đôi khi, những định danh này được gọi là thông tin xác thực của process. Các định danh đó bao gồm:

- real user ID và group ID;
- effective user ID và group ID;
- saved set-user-ID và saved set-group-ID;
- file-system user ID và group ID (đặc thù của Linux); và
- supplementary group ID.

Trong chương này, chúng ta xem xét chi tiết mục đích của các định danh process này và mô tả các system call cùng library function có thể dùng để truy xuất và thay đổi chúng. Chúng ta cũng thảo luận về khái niệm process đặc quyền (privileged) và không đặc quyền (unprivileged), cũng như cơ chế set-user-ID và set-group-ID cho phép tạo ra các chương trình chạy với quyền của một người dùng hoặc nhóm cụ thể.

### **9.1 Real User ID và Real Group ID**

Real user ID và group ID xác định người dùng và nhóm mà process thuộc về. Trong quá trình đăng nhập, login shell nhận real user ID và group ID từ trường thứ ba và thứ tư trong bản ghi mật khẩu của người dùng trong file `/etc/passwd` (Mục 8.1). Khi một process mới được tạo (ví dụ khi shell thực thi một chương trình), nó kế thừa các định danh này từ process cha.

## **9.2 Effective User ID và Effective Group ID**

Trên hầu hết các cài đặt UNIX (Linux có chút khác biệt, như giải thích trong Mục 9.5), effective user ID và group ID, kết hợp với supplementary group ID, được dùng để xác định quyền được cấp cho process khi nó cố thực hiện các thao tác khác nhau (tức là các system call). Ví dụ, các định danh này xác định quyền được cấp cho process khi nó truy cập các tài nguyên như file và các đối tượng System V interprocess communication (IPC), vốn có user ID và group ID liên kết để xác định chúng thuộc về ai. Như chúng ta sẽ thấy ở Mục 20.5, effective user ID cũng được kernel dùng để xác định xem một process có thể gửi signal cho process khác hay không.

Một process có effective user ID là 0 (user ID của root) có toàn bộ quyền của superuser. Process đó được gọi là privileged process. Một số system call chỉ có thể được thực thi bởi privileged process.

> Trong Chương 39, chúng ta mô tả cài đặt capabilities của Linux, một cơ chế chia các đặc quyền được cấp cho superuser thành nhiều đơn vị riêng biệt có thể được bật hoặc tắt độc lập.

Thông thường, effective user ID và group ID có cùng giá trị với các real ID tương ứng, nhưng có hai cách để effective ID nhận các giá trị khác. Một cách là thông qua các system call mà chúng ta thảo luận ở Mục 9.7. Cách thứ hai là thông qua việc thực thi các chương trình set-user-ID và set-group-ID.

## **9.3 Chương Trình Set-User-ID và Set-Group-ID**

Chương trình set-user-ID cho phép process có được các đặc quyền mà bình thường nó không có, bằng cách đặt effective user ID của process bằng với user ID (chủ sở hữu) của file thực thi. Chương trình set-group-ID thực hiện thao tác tương tự cho effective group ID của process. (Các thuật ngữ set-user-ID program và set-group-ID program đôi khi được viết tắt là set-UID program và set-GID program.)

Giống như bất kỳ file nào khác, một file thực thi có user ID và group ID liên kết xác định quyền sở hữu file. Ngoài ra, file thực thi có hai bit quyền đặc biệt: set-user-ID bit và set-group-ID bit. (Thực ra, mọi file đều có hai bit quyền này, nhưng chúng ta quan tâm đến việc sử dụng chúng với file thực thi.) Các bit quyền này được đặt bằng lệnh `chmod`. Người dùng không có đặc quyền có thể đặt các bit này cho các file họ sở hữu. Người dùng có đặc quyền (`CAP_FOWNER`) có thể đặt các bit này cho bất kỳ file nào. Đây là ví dụ:

```
$ su
Password:
# ls -l prog
-rwxr-xr-x 1 root root 302585 Jun 26 15:05 prog
# chmod u+s prog Turn on set-user-ID permission bit
# chmod g+s prog Turn on set-group-ID permission bit
```

Như ví dụ trên cho thấy, một chương trình có thể có cả hai bit này được đặt, mặc dù điều này không phổ biến. Khi dùng `ls -l` để liệt kê quyền của chương trình có bit set-user-ID hoặc set-group-ID, chữ `x` thường dùng để chỉ rằng quyền thực thi được đặt sẽ được thay bằng chữ `s`:

```
# ls -l prog
-rwsr-sr-x 1 root root 302585 Jun 26 15:05 prog
```

Khi chương trình set-user-ID được chạy (tức là được nạp vào bộ nhớ của process bởi một `exec()`), kernel đặt effective user ID của process bằng với user ID của file thực thi. Chạy chương trình set-group-ID có tác động tương tự đối với effective group ID của process. Việc thay đổi effective user ID hoặc group ID theo cách này cấp cho process (hay nói cách khác, người dùng đang chạy chương trình) các đặc quyền mà thông thường nó không có. Ví dụ, nếu file thực thi thuộc sở hữu của root (superuser) và có set-user-ID bit được bật, thì process sẽ có được quyền superuser khi chương trình đó chạy.

Các chương trình set-user-ID và set-group-ID cũng có thể được thiết kế để thay đổi effective ID của process thành một giá trị khác root. Ví dụ, để cung cấp quyền truy cập vào file được bảo vệ (hoặc tài nguyên hệ thống khác), có thể chỉ cần tạo một user (group) ID chuyên dụng có các đặc quyền cần thiết để truy cập file đó, và tạo chương trình set-user-ID (set-group-ID) thay đổi effective user (group) ID của process thành ID đó. Điều này cho phép chương trình truy cập file mà không cấp cho nó toàn bộ quyền của superuser.

Đôi khi chúng ta dùng thuật ngữ set-user-ID-root để phân biệt chương trình set-user-ID thuộc sở hữu của root với chương trình thuộc sở hữu của người dùng khác, vốn chỉ cấp cho process các đặc quyền của người dùng đó.

> Chúng ta đã bắt đầu dùng thuật ngữ "privileged" (đặc quyền) theo hai nghĩa khác nhau. Một nghĩa là như đã định nghĩa trước: process có effective user ID là 0, có toàn bộ các đặc quyền dành cho root. Tuy nhiên, khi nói về chương trình set-user-ID thuộc sở hữu của người dùng không phải root, đôi khi chúng ta gọi một process là đang có được các đặc quyền tương ứng với user ID của chương trình set-user-ID. Nghĩa nào của thuật ngữ "privileged" trong mỗi trường hợp sẽ rõ ràng từ ngữ cảnh.

> Vì lý do chúng ta giải thích trong Mục 38.3, các bit quyền set-user-ID và set-group-ID không có tác dụng đối với shell script trên Linux.

Các ví dụ về chương trình set-user-ID thường dùng trên Linux bao gồm: `passwd(1)`, thay đổi mật khẩu của người dùng; `mount(8)` và `umount(8)`, mount và umount các file system; và `su(1)`, cho phép người dùng chạy shell với user ID khác. Ví dụ về chương trình set-group-ID là `wall(1)`, ghi thông điệp đến tất cả terminal thuộc nhóm `tty` (thông thường, mọi terminal đều thuộc nhóm này).

Trong Mục 8.5, chúng ta đã ghi nhận rằng chương trình trong Listing 8-2 cần được chạy từ phiên đăng nhập root để có thể truy cập file `/etc/shadow`. Chúng ta có thể làm cho chương trình này chạy được bởi bất kỳ người dùng nào bằng cách biến nó thành chương trình set-user-ID-root như sau:

```
$ su
Password:
# chown root check_password Make this program owned by root
# chmod u+s check_password With the set-user-ID bit enabled
```

```
# ls -l check_password
-rwsr-xr-x 1 root users 18150 Oct 28 10:49 check_password
# exit
$ whoami This is an unprivileged login
mtk
$ ./check_password But we can now access the shadow
Username: avr password file using this program
Password:
Successfully authenticated: UID=1001
```

Kỹ thuật set-user-ID/set-group-ID là một công cụ hữu ích và mạnh mẽ, nhưng có thể dẫn đến các lỗ hổng bảo mật trong các ứng dụng được thiết kế kém. Trong Chương 38, chúng ta liệt kê một tập hợp các thực hành tốt cần tuân theo khi viết chương trình set-user-ID và set-group-ID.

## **9.4 Saved Set-User-ID và Saved Set-Group-ID**

Saved set-user-ID và saved set-group-ID được thiết kế để dùng với các chương trình set-user-ID và set-group-ID. Khi một chương trình được thực thi, các bước sau (trong số nhiều bước khác) xảy ra:

- 1. Nếu bit quyền set-user-ID (set-group-ID) được bật trên file thực thi, thì effective user (group) ID của process được đặt bằng với chủ sở hữu của file thực thi. Nếu bit set-user-ID (set-group-ID) không được đặt, thì không có thay đổi nào được thực hiện đối với effective user (group) ID của process.
- 2. Các giá trị của saved set-user-ID và saved set-group-ID được sao chép từ các effective ID tương ứng. Việc sao chép này xảy ra bất kể bit set-user-ID hoặc set-group-ID có được đặt hay không trên file đang được thực thi.

Ví dụ về tác động của các bước trên: giả sử một process có real user ID, effective user ID và saved set-user-ID đều là 1000 thực thi chương trình set-user-ID thuộc sở hữu của root (user ID 0). Sau khi `exec`, các user ID của process sẽ thay đổi như sau:

```
real=1000 effective=0 saved=0
```

Nhiều system call cho phép chương trình set-user-ID chuyển đổi effective user ID của nó giữa giá trị real user ID và saved set-user-ID. Các system call tương tự cho phép chương trình set-group-ID thay đổi effective group ID của nó. Bằng cách này, chương trình có thể tạm thời hủy bỏ và lấy lại bất kỳ đặc quyền nào liên quan đến user (group) ID của file đã được exec. (Nói cách khác, chương trình có thể chuyển đổi giữa trạng thái có thể có đặc quyền và thực sự hoạt động với đặc quyền.) Như chúng ta sẽ trình bày trong Mục 38.2, thực hành lập trình an toàn là các chương trình set-user-ID và set-group-ID nên hoạt động dưới ID không đặc quyền (tức là real ID) bất cứ khi nào chương trình không thực sự cần thực hiện các thao tác liên quan đến ID đặc quyền (tức là saved set ID).

> Saved set-user-ID và saved set-group-ID đôi khi còn được gọi là saved user ID và saved group ID.

Saved set ID là phát minh của System V được POSIX chấp nhận. Chúng không có trên các bản phát hành BSD trước 4.4. Tiêu chuẩn POSIX.1 ban đầu đặt hỗ trợ cho các ID này là tùy chọn, nhưng các tiêu chuẩn sau đó (bắt đầu từ FIPS 151-1 năm 1988) đã bắt buộc hỗ trợ.

## **9.5 File-System User ID và File-System Group ID**

Trên Linux, chính file-system user ID và group ID, thay vì effective user ID và group ID, được dùng (kết hợp với supplementary group ID) để xác định quyền khi thực hiện các thao tác file-system như mở file, thay đổi quyền sở hữu file và sửa đổi quyền file. (Các effective ID vẫn được dùng, như trên các cài đặt UNIX khác, cho các mục đích khác đã mô tả trước đó.)

Thông thường, file-system user ID và group ID có cùng giá trị với các effective ID tương ứng (và do đó thường giống với các real ID tương ứng). Hơn nữa, bất cứ khi nào effective user ID hoặc group ID thay đổi, hoặc bởi system call hoặc bởi việc thực thi chương trình set-user-ID hoặc set-group-ID, file-system ID tương ứng cũng được thay đổi thành cùng giá trị đó. Vì file-system ID theo sau effective ID theo cách này, Linux thực tế hoạt động giống như bất kỳ cài đặt UNIX nào khác khi kiểm tra đặc quyền và quyền. File-system ID khác với effective ID tương ứng, và do đó Linux khác với các cài đặt UNIX khác, chỉ khi chúng ta sử dụng hai system call đặc thù của Linux là `setfsuid()` và `setfsgid()` để làm chúng khác nhau một cách rõ ràng.

Tại sao Linux cung cấp file-system ID và trong hoàn cảnh nào chúng ta muốn effective ID và file-system ID khác nhau? Lý do chủ yếu là lịch sử. File-system ID xuất hiện lần đầu trong Linux 1.2. Trong phiên bản kernel đó, một process có thể gửi signal đến process khác nếu effective user ID của bên gửi khớp với real hoặc effective user ID của process đích. Điều này ảnh hưởng đến một số chương trình như chương trình máy chủ Linux NFS (Network File System), cần có khả năng truy cập file như thể nó có effective ID của process client tương ứng. Tuy nhiên, nếu máy chủ NFS thay đổi effective user ID của nó, nó sẽ dễ bị tổn thương trước các signal từ các process người dùng không có đặc quyền. Để ngăn chặn khả năng này, các file-system user ID và group ID riêng biệt đã được phát minh. Bằng cách để nguyên effective ID của mình, nhưng thay đổi file-system ID, máy chủ NFS có thể giả vờ là người dùng khác để truy cập file mà không dễ bị tổn thương trước signal từ các process người dùng.

Từ kernel 2.0 trở đi, Linux áp dụng các quy tắc do SUSv3 yêu cầu về quyền gửi signal, và các quy tắc này không liên quan đến effective user ID của process đích (xem Mục 20.5). Do đó, tính năng file-system ID không còn thực sự cần thiết nữa (một process ngày nay có thể đạt được kết quả mong muốn bằng cách sử dụng hợp lý các system call được mô tả ở phần sau của chương này để thay đổi giá trị effective user ID về và từ một giá trị không có đặc quyền, khi cần thiết), nhưng nó vẫn còn đó để tương thích với phần mềm hiện có.

Vì file-system ID có phần kỳ lạ, và thường có cùng giá trị với effective ID tương ứng, trong phần còn lại của cuốn sách này, chúng ta thường mô tả các kiểm tra quyền file khác nhau, cũng như việc đặt quyền sở hữu file mới, theo nghĩa effective ID của process. Mặc dù file-system ID của process thực sự được dùng cho các mục đích này trên Linux, nhưng trên thực tế, sự hiện diện của chúng hiếm khi tạo ra sự khác biệt thực chất.

## **9.6 Supplementary Group ID**

Supplementary group ID là tập hợp các nhóm bổ sung mà process thuộc về. Process mới kế thừa các ID này từ process cha của nó. Login shell nhận supplementary group ID từ file nhóm hệ thống. Như đã ghi nhận ở trên, các ID này được dùng kết hợp với effective ID và file-system ID để xác định quyền truy cập file, các đối tượng System V IPC và các tài nguyên hệ thống khác.

## **9.7 Truy Xuất và Sửa Đổi Thông Tin Xác Thực Của Process**

Linux cung cấp nhiều system call và library function để truy xuất và thay đổi các user ID và group ID khác nhau mà chúng ta đã mô tả trong chương này. Chỉ một số API này được SUSv3 quy định. Trong số còn lại, nhiều API phổ biến trên các cài đặt UNIX khác và một số là đặc thù của Linux. Chúng ta sẽ ghi nhận các vấn đề về tính di động khi mô tả từng giao diện. Về cuối chương này, Bảng 9-1 tóm tắt hoạt động của tất cả các giao diện dùng để thay đổi thông tin xác thực của process.

Thay vì dùng các system call được mô tả ở các trang sau, thông tin xác thực của bất kỳ process nào có thể được tìm thấy bằng cách kiểm tra các dòng `Uid`, `Gid` và `Groups` trong file `/proc/PID/status` đặc thù của Linux. Các dòng `Uid` và `Gid` liệt kê các định danh theo thứ tự: real, effective, saved set và file system.

Trong các mục tiếp theo, chúng ta sử dụng định nghĩa truyền thống về privileged process là process có effective user ID là 0. Tuy nhiên, Linux chia khái niệm đặc quyền superuser thành các capabilities riêng biệt, như mô tả trong Chương 39. Hai capabilities liên quan đến thảo luận của chúng ta về tất cả các system call dùng để thay đổi user ID và group ID của process:

- Capability `CAP_SETUID` cho phép process thực hiện các thay đổi tùy ý đối với user ID của nó.
- Capability `CAP_SETGID` cho phép process thực hiện các thay đổi tùy ý đối với group ID của nó.

## **9.7.1 Truy Xuất và Sửa Đổi Real, Effective và Saved Set ID**

Trong các đoạn sau, chúng ta mô tả các system call truy xuất và sửa đổi real, effective và saved set ID. Có nhiều system call thực hiện các tác vụ này, và trong một số trường hợp chức năng của chúng trùng nhau, phản ánh thực tế là các system call khác nhau có nguồn gốc từ các cài đặt UNIX khác nhau.

#### **Truy xuất real và effective ID**

Các system call `getuid()` và `getgid()` trả về lần lượt real user ID và real group ID của process đang gọi. Các system call `geteuid()` và `getegid()` thực hiện các tác vụ tương ứng cho effective ID. Các system call này luôn thành công.

```
#include <unistd.h>
uid_t getuid(void);
                                           Returns real user ID of calling process
uid_t geteuid(void);
                                      Returns effective user ID of calling process
gid_t getgid(void);
                                         Returns real group ID of calling process
gid_t getegid(void);
                                     Returns effective group ID of calling process
```

#### **Sửa đổi effective ID**

System call `setuid()` thay đổi effective user ID—và có thể cả real user ID lẫn saved set-user-ID—của process đang gọi thành giá trị được cung cấp bởi tham số `uid`. System call `setgid()` thực hiện tác vụ tương tự cho các group ID tương ứng.

```
#include <unistd.h>
int setuid(uid_t uid);
int setgid(gid_t gid);
                                         Both return 0 on success, or –1 on error
```

Các quy tắc về những thay đổi mà process có thể thực hiện đối với thông tin xác thực của nó bằng `setuid()` và `setgid()` phụ thuộc vào việc process có đặc quyền hay không (tức là có effective user ID bằng 0 hay không). Các quy tắc sau áp dụng cho `setuid()`:

- 1. Khi một process không có đặc quyền gọi `setuid()`, chỉ effective user ID của process bị thay đổi. Hơn nữa, nó chỉ có thể được thay đổi thành cùng giá trị với real user ID hoặc saved set-user-ID. (Các nỗ lực vi phạm ràng buộc này sẽ sinh ra lỗi `EPERM`.) Điều này có nghĩa là đối với người dùng không có đặc quyền, cuộc gọi này chỉ hữu ích khi thực thi chương trình set-user-ID, vì với việc thực thi các chương trình thông thường, real user ID, effective user ID và saved set-user-ID của process đều có cùng giá trị. Trên một số cài đặt dựa trên BSD, các cuộc gọi đến `setuid()` hoặc `setgid()` bởi process không có đặc quyền có ngữ nghĩa khác với các cài đặt UNIX khác: các cuộc gọi thay đổi real, effective và saved set ID (thành giá trị của real hoặc effective ID hiện tại).
- 2. Khi process có đặc quyền thực thi `setuid()` với tham số khác 0, thì real user ID, effective user ID và saved set-user-ID đều được đặt thành giá trị được chỉ định trong tham số `uid`. Đây là một chiều duy nhất—một khi process có đặc quyền đã thay đổi các định danh của nó theo cách này, nó mất tất cả đặc quyền và do đó không thể dùng `setuid()` để đặt lại các định danh về 0. Nếu điều này không mong muốn, thì nên dùng `seteuid()` hoặc `setreuid()` thay vì `setuid()`.

Các quy tắc chi phối các thay đổi có thể thực hiện đối với group ID bằng `setgid()` tương tự, nhưng thay `setgid()` cho `setuid()` và group cho user. Với những thay đổi này, quy tắc 1 áp dụng đúng như đã nêu. Trong quy tắc 2, vì việc thay đổi group ID không khiến process mất đặc quyền (đặc quyền được xác định bởi effective user ID), các chương trình có đặc quyền có thể dùng `setgid()` để tự do thay đổi group ID thành bất kỳ giá trị mong muốn nào.

Cuộc gọi sau là phương pháp ưa thích cho chương trình set-user-ID-root có effective user ID hiện tại là 0 để hủy bỏ vĩnh viễn tất cả đặc quyền (bằng cách đặt cả effective user ID và saved set-user-ID thành cùng giá trị với real user ID):

```
if (setuid(getuid()) == -1)
 errExit("setuid");
```

Chương trình set-user-ID thuộc sở hữu của người dùng không phải root có thể dùng `setuid()` để chuyển đổi effective user ID giữa giá trị của real user ID và saved set-user-ID vì lý do bảo mật được mô tả trong Mục 9.4. Tuy nhiên, `seteuid()` thích hợp hơn cho mục đích này, vì nó có tác dụng như nhau, bất kể chương trình set-user-ID có thuộc sở hữu của root hay không.

Một process có thể dùng `seteuid()` để thay đổi effective user ID của nó (thành giá trị được chỉ định bởi `euid`), và `setegid()` để thay đổi effective group ID của nó (thành giá trị được chỉ định bởi `egid`).

```
#include <unistd.h>
int seteuid(uid_t euid);
int setegid(gid_t egid);
                                         Both return 0 on success, or –1 on error
```

Các quy tắc sau chi phối các thay đổi mà process có thể thực hiện đối với effective ID của nó bằng `seteuid()` và `setegid()`:

- 1. Process không có đặc quyền chỉ có thể thay đổi effective ID thành cùng giá trị với real hoặc saved set ID tương ứng. (Nói cách khác, đối với process không có đặc quyền, `seteuid()` và `setegid()` có tác dụng giống như `setuid()` và `setgid()` tương ứng, ngoại trừ các vấn đề di động BSD đã đề cập trước đó.)
- 2. Process có đặc quyền có thể thay đổi effective ID thành bất kỳ giá trị nào. Nếu process có đặc quyền dùng `seteuid()` để thay đổi effective user ID thành giá trị khác 0, thì nó không còn được đặc quyền nữa (nhưng có thể lấy lại đặc quyền thông qua quy tắc trước).

Dùng `seteuid()` là phương pháp ưa thích cho các chương trình set-user-ID và set-group-ID để tạm thời hủy bỏ và sau đó lấy lại đặc quyền. Đây là ví dụ:

```
euid = geteuid(); /* Save initial effective user ID (which
 is same as saved set-user-ID) */
if (seteuid(getuid()) == -1) /* Drop privileges */
 errExit("seteuid");
if (seteuid(euid) == -1) /* Regain privileges */
 errExit("seteuid");
```

Có nguồn gốc từ BSD, `seteuid()` và `setegid()` hiện đã được quy định trong SUSv3 và xuất hiện trên hầu hết các cài đặt UNIX.

Trong các phiên bản cũ hơn của GNU C library (glibc 2.0 và trước đó), `seteuid(euid)` được cài đặt như `setreuid(–1, euid)`. Trong các phiên bản hiện đại của glibc, `seteuid(euid)` được cài đặt như `setresuid(–1, euid, –1)`. (Chúng ta mô tả `setreuid()`, `setresuid()` và các hàm tương tự cho group ngay sau đây.) Cả hai cách cài đặt đều cho phép chúng ta chỉ định `euid` là cùng giá trị với effective user ID hiện tại (tức là không thay đổi). Tuy nhiên, SUSv3 không quy định hành vi này cho `seteuid()`, và điều đó không thể thực hiện được trên một số cài đặt UNIX khác. Nhìn chung, sự biến đổi tiềm ẩn này giữa các cài đặt không rõ ràng, vì trong các tình huống bình thường, effective user ID có cùng giá trị với real user ID hoặc saved set-user-ID. (Cách duy nhất để làm effective user ID khác với cả real user ID và saved set-user-ID trên Linux là thông qua việc sử dụng system call `setresuid()` không chuẩn.)

Trong tất cả các phiên bản glibc (kể cả phiên bản hiện đại), `setegid(egid)` được cài đặt như `setregid(–1, egid)`. Cũng như `seteuid()`, điều này có nghĩa là chúng ta có thể chỉ định `egid` là cùng giá trị với effective group ID hiện tại, mặc dù hành vi này không được quy định trong SUSv3. Nó cũng có nghĩa là `setegid()` thay đổi saved set-group-ID nếu effective group ID được đặt thành giá trị khác với real group ID hiện tại. (Nhận xét tương tự áp dụng cho cài đặt cũ hơn của `seteuid()` dùng `setreuid()`.) Một lần nữa, hành vi này không được quy định trong SUSv3.

#### **Sửa đổi real và effective ID**

System call `setreuid()` cho phép process gọi thay đổi độc lập các giá trị real và effective user ID của nó. System call `setregid()` thực hiện tác vụ tương tự cho real và effective group ID.

```
#include <unistd.h>
int setreuid(uid_t ruid, uid_t euid);
int setregid(gid_t rgid, gid_t egid);
                                         Both return 0 on success, or –1 on error
```

Tham số đầu tiên của mỗi system call này là real ID mới. Tham số thứ hai là effective ID mới. Nếu chúng ta chỉ muốn thay đổi một trong các định danh, thì có thể chỉ định –1 cho tham số kia.

Có nguồn gốc từ BSD, `setreuid()` và `setregid()` hiện được quy định trong SUSv3 và có sẵn trên hầu hết các cài đặt UNIX.

Cũng như các system call khác được mô tả trong mục này, có các quy tắc chi phối các thay đổi có thể thực hiện bằng `setreuid()` và `setregid()`. Chúng ta mô tả các quy tắc này từ góc độ của `setreuid()`, với hiểu biết rằng `setregid()` tương tự, ngoại trừ khi có ghi chú:

1. Process không có đặc quyền chỉ có thể đặt real user ID thành giá trị hiện tại của real (tức là không thay đổi) hoặc effective user ID. Effective user ID chỉ có thể được đặt thành giá trị hiện tại của real user ID, effective user ID (tức là không thay đổi) hoặc saved set-user-ID.

SUSv3 nói rằng chưa được quy định rõ ràng liệu process không có đặc quyền có thể dùng `setreuid()` để thay đổi giá trị real user ID thành giá trị hiện tại của real user ID, effective user ID hoặc saved set-user-ID hay không, và chi tiết về chính xác những thay đổi nào có thể thực hiện đối với real user ID khác nhau giữa các cài đặt.

SUSv3 mô tả hành vi hơi khác cho `setregid()`: process không có đặc quyền có thể đặt real group ID thành giá trị hiện tại của saved set-group-ID hoặc đặt effective group ID thành giá trị hiện tại của real group ID hoặc saved set-group-ID. Một lần nữa, chi tiết về chính xác những thay đổi nào có thể thực hiện khác nhau giữa các cài đặt.

- 2. Process có đặc quyền có thể thực hiện bất kỳ thay đổi nào đối với các ID.
- 3. Đối với cả process có đặc quyền và không có đặc quyền, saved set-user-ID cũng được đặt thành cùng giá trị với effective user ID (mới) nếu một trong các điều kiện sau là đúng:
  - a) `ruid` không phải –1 (tức là real user ID đang được đặt, ngay cả với cùng giá trị mà nó đã có), hoặc
  - b) effective user ID đang được đặt thành giá trị khác với giá trị real user ID trước khi gọi.

Nói ngược lại, nếu một process dùng `setreuid()` chỉ để thay đổi effective user ID thành cùng giá trị với real user ID hiện tại, thì saved set-user-ID không thay đổi, và cuộc gọi sau đến `setreuid()` (hoặc `seteuid()`) có thể khôi phục effective user ID về giá trị saved set-user-ID. (SUSv3 không quy định tác động của `setreuid()` và `setregid()` lên saved set ID, nhưng SUSv4 quy định hành vi được mô tả ở đây.)

Quy tắc thứ ba cung cấp một cách để chương trình set-user-ID hủy bỏ vĩnh viễn đặc quyền của nó, bằng cách gọi như sau:

```
setreuid(getuid(), getuid());
```

Process set-user-ID-root muốn thay đổi cả user lẫn group credentials thành các giá trị tùy ý nên gọi `setregid()` trước rồi gọi `setreuid()` sau. Nếu các cuộc gọi được thực hiện theo thứ tự ngược lại, thì cuộc gọi `setregid()` sẽ thất bại, vì chương trình sẽ không còn được đặc quyền sau khi gọi `setreuid()`. Nhận xét tương tự áp dụng nếu chúng ta đang dùng `setresuid()` và `setresgid()` (mô tả bên dưới) cho mục đích này.

> Các bản phát hành BSD lên đến 4.3BSD không có saved set-user-ID và saved set-group-ID (hiện được bắt buộc bởi SUSv3). Thay vào đó, trên BSD, `setreuid()` và `setregid()` cho phép process hủy bỏ và lấy lại đặc quyền bằng cách hoán đổi các giá trị của real và effective ID qua lại. Điều này có tác dụng phụ không mong muốn là thay đổi real user ID để thay đổi effective user ID.

#### **Truy xuất real, effective và saved set ID**

Trên hầu hết các cài đặt UNIX, process không thể trực tiếp truy xuất (hoặc cập nhật) saved set-user-ID và saved set-group-ID của nó. Tuy nhiên, Linux cung cấp hai system call (không chuẩn) cho phép làm điều đó: `getresuid()` và `getresgid()`.

```
#define _GNU_SOURCE
#include <unistd.h>
int getresuid(uid_t *ruid, uid_t *euid, uid_t *suid);
int getresgid(gid_t *rgid, gid_t *egid, gid_t *sgid);
                                         Both return 0 on success, or –1 on error
```

System call `getresuid()` trả về các giá trị hiện tại của real user ID, effective user ID và saved set-user-ID của process gọi vào các vị trí được trỏ bởi ba tham số của nó. System call `getresgid()` làm tương tự cho các group ID tương ứng.

#### **Sửa đổi real, effective và saved set ID**

System call `setresuid()` cho phép process gọi thay đổi độc lập các giá trị của cả ba user ID của nó. Các giá trị mới cho mỗi user ID được chỉ định bởi ba tham số của system call. System call `setresgid()` thực hiện tác vụ tương tự cho group ID.

```
#define _GNU_SOURCE
#include <unistd.h>
int setresuid(uid_t ruid, uid_t euid, uid_t suid);
int setresgid(gid_t rgid, gid_t egid, gid_t sgid);
                                         Both return 0 on success, or –1 on error
```

Nếu chúng ta không muốn thay đổi tất cả các định danh, thì chỉ định –1 cho tham số sẽ giữ nguyên định danh tương ứng. Ví dụ, cuộc gọi sau tương đương với `seteuid(x)`:

```
setresuid(-1, x, -1);
```

Các quy tắc về những thay đổi có thể thực hiện bởi `setresuid()` (với `setresgid()` tương tự) như sau:

- 1. Process không có đặc quyền có thể đặt bất kỳ trong số real user ID, effective user ID và saved set-user-ID của nó thành bất kỳ giá trị nào hiện có trong real user ID, effective user ID hoặc saved set-user-ID hiện tại của nó.
- 2. Process có đặc quyền có thể thực hiện các thay đổi tùy ý đối với real user ID, effective user ID và saved set-user-ID của nó.
- 3. Bất kể cuộc gọi có thực hiện bất kỳ thay đổi nào đối với các ID khác hay không, file-system user ID luôn được đặt thành cùng giá trị với effective user ID (có thể mới).

Các cuộc gọi đến `setresuid()` và `setresgid()` có tác dụng tất-cả-hoặc-không-gì. Hoặc tất cả các định danh được yêu cầu được thay đổi thành công hoặc không có định danh nào được thay đổi. (Nhận xét tương tự áp dụng đối với các system call khác được mô tả trong chương này có thay đổi nhiều định danh.)

Mặc dù `setresuid()` và `setresgid()` cung cấp API đơn giản nhất để thay đổi thông tin xác thực của process, chúng ta không thể dùng chúng một cách di động trong các ứng dụng; chúng không được quy định trong SUSv3 và chỉ có sẵn trên một vài cài đặt UNIX khác.

#### **9.7.2 Truy Xuất và Sửa Đổi File-System ID**

Tất cả các system call đã mô tả trước đó thay đổi effective user ID hoặc group ID của process cũng luôn thay đổi file-system ID tương ứng. Để thay đổi file-system ID độc lập với effective ID, chúng ta phải dùng hai system call đặc thù của Linux: `setfsuid()` và `setfsgid()`.

```
#include <sys/fsuid.h>
int setfsuid(uid_t fsuid);
                                  Always returns the previous file-system user ID
int setfsgid(gid_t fsgid);
                                Always returns the previous file-system group ID
```

System call `setfsuid()` thay đổi file-system user ID của process thành giá trị được chỉ định bởi `fsuid`. System call `setfsgid()` thay đổi file-system group ID thành giá trị được chỉ định bởi `fsgid`.

Cũng có các quy tắc về loại thay đổi có thể thực hiện. Các quy tắc cho `setfsgid()` tương tự với các quy tắc cho `setfsuid()`, như sau:

- 1. Process không có đặc quyền có thể đặt file-system user ID thành giá trị hiện tại của real user ID, effective user ID, file-system user ID (tức là không thay đổi) hoặc saved set-user-ID.
- 2. Process có đặc quyền có thể đặt file-system user ID thành bất kỳ giá trị nào.

Cài đặt của các cuộc gọi này có phần thô sơ. Trước tiên, không có các system call tương ứng để truy xuất giá trị hiện tại của file-system ID. Ngoài ra, các system call không thực hiện kiểm tra lỗi; nếu process không có đặc quyền cố đặt file-system ID thành giá trị không được chấp nhận, nỗ lực đó bị bỏ qua một cách im lặng. Giá trị trả về từ mỗi system call này là giá trị trước đó của file-system ID tương ứng, bất kể cuộc gọi có thành công hay không. Do đó, chúng ta có một cách để tìm ra giá trị hiện tại của file-system ID, nhưng chỉ cùng lúc với khi chúng ta cố (thành công hoặc không thành công) thay đổi chúng.

Việc sử dụng các system call `setfsuid()` và `setfsgid()` không còn cần thiết trên Linux và nên tránh trong các ứng dụng được thiết kế để chuyển sang các cài đặt UNIX khác.

#### **9.7.3 Truy Xuất và Sửa Đổi Supplementary Group ID**

System call `getgroups()` trả về tập hợp các nhóm mà process đang gọi hiện là thành viên, trong mảng được trỏ bởi `grouplist`.

```
#include <unistd.h>
int getgroups(int gidsetsize, gid_t grouplist[]);
     Returns number of group IDs placed in grouplist on success, or –1 on error
```

Trên Linux, cũng như trên hầu hết các cài đặt UNIX, `getgroups()` đơn giản trả về supplementary group ID của process đang gọi. Tuy nhiên, SUSv3 cũng cho phép cài đặt có thể bao gồm effective group ID của process đang gọi trong `grouplist` được trả về.

Chương trình gọi phải cấp phát mảng `grouplist` và chỉ định độ dài của nó trong tham số `gidsetsize`. Khi hoàn thành thành công, `getgroups()` trả về số lượng group ID được đặt trong `grouplist`.

Nếu số lượng nhóm mà process là thành viên vượt quá `gidsetsize`, `getgroups()` trả về lỗi (`EINVAL`). Để tránh khả năng này, chúng ta có thể khai báo mảng `grouplist` lớn hơn một phần (để cho phép di động bao gồm có thể có của effective group ID) so với hằng số `NGROUPS_MAX` (định nghĩa trong `<limits.h>`), xác định số lượng tối đa của supplementary group mà process có thể là thành viên. Do đó, chúng ta có thể khai báo `grouplist` như sau:

```
gid_t grouplist[NGROUPS_MAX + 1];
```

Trong các kernel Linux trước 2.6.4, `NGROUPS_MAX` có giá trị 32. Từ kernel 2.6.4 trở đi, `NGROUPS_MAX` có giá trị 65.536.

Ứng dụng cũng có thể xác định giới hạn `NGROUPS_MAX` tại thời gian chạy theo các cách sau:

- Gọi `sysconf(_SC_NGROUPS_MAX)`. (Chúng ta giải thích việc sử dụng `sysconf()` trong Mục 11.2.)
- Đọc giới hạn từ file `/proc/sys/kernel/ngroups_max` chỉ đọc, đặc thù của Linux. File này được cung cấp từ kernel 2.6.4.

Ngoài ra, ứng dụng có thể thực hiện cuộc gọi đến `getgroups()` chỉ định `gidsetsize` là 0. Trong trường hợp này, `grouplist` không được sửa đổi, nhưng giá trị trả về của cuộc gọi cho biết số lượng nhóm mà process là thành viên.

Giá trị thu được bởi bất kỳ kỹ thuật thời gian chạy nào trong số này sau đó có thể được dùng để cấp phát động mảng `grouplist` cho cuộc gọi `getgroups()` trong tương lai.

Process có đặc quyền có thể dùng `setgroups()` và `initgroups()` để thay đổi tập hợp supplementary group ID của nó.

```
#define _BSD_SOURCE
#include <grp.h>
int setgroups(size_t gidsetsize, const gid_t *grouplist);
int initgroups(const char *user, gid_t group);
                                         Both return 0 on success, or –1 on error
```

System call `setgroups()` thay thế supplementary group ID của process đang gọi bằng tập hợp được cung cấp trong mảng `grouplist`. Tham số `gidsetsize` chỉ định số lượng group ID trong tham số mảng `grouplist`.

Hàm `initgroups()` khởi tạo supplementary group ID của process đang gọi bằng cách quét `/etc/groups` và xây dựng danh sách tất cả các nhóm mà người dùng được đặt tên là thành viên. Ngoài ra, group ID được chỉ định trong `group` cũng được thêm vào tập hợp supplementary group ID của process.

Người dùng chính của `initgroups()` là các chương trình tạo phiên đăng nhập—ví dụ, `login(1)`, thiết lập các thuộc tính process khác nhau trước khi thực thi login shell của người dùng. Các chương trình như vậy thường thu được giá trị được dùng cho tham số `group` bằng cách đọc trường group ID từ bản ghi của người dùng trong file mật khẩu. Điều này có phần dễ nhầm lẫn, vì group ID từ file mật khẩu thực ra không phải là supplementary group. Thay vào đó, nó xác định real user ID, effective user ID và saved set-user-ID ban đầu của login shell. Tuy nhiên, đây là cách `initgroups()` thường được dùng.

Mặc dù không phải là một phần của SUSv3, `setgroups()` và `initgroups()` có sẵn trên tất cả các cài đặt UNIX.

## **9.7.4 Tóm Tắt Các Cuộc Gọi Để Sửa Đổi Thông Tin Xác Thực Của Process**

Bảng 9-1 tóm tắt tác động của các system call và library function khác nhau được dùng để thay đổi thông tin xác thực của process.

Hình 9-1 cung cấp cái nhìn tổng quan về cùng thông tin trong Bảng 9-1. Sơ đồ này cho thấy mọi thứ từ góc độ của các cuộc gọi thay đổi user ID, nhưng các quy tắc thay đổi group ID cũng tương tự.

```mermaid
flowchart TD
    setuid["setuid(u)"]
    seteuid["seteuid(e)"]
    setreuid["setreuid(r, e)"]
    setresuid["setresuid(r, e, s)"]

    real["Real\nuser ID"]
    effective["Effective\nuser ID"]
    saved["Saved\nset-user-ID"]

    setuid -.->|"r, s"| real
    setuid -.->|"r, s"| effective
    setuid -.->|"r, s"| saved

    seteuid -->|"e"| effective
    seteuid -.->|"e"| effective

    setreuid -->|"r, e"| real
    setreuid -->|"r, e"| effective
    setreuid -.->|"r, e"| real
    setreuid -.->|"r, e"| effective

    setresuid -->|"r, e, s"| real
    setresuid -->|"r, e, s"| effective
    setresuid -->|"r, e, s"| saved
    setresuid -.->|"r, e, s"| real
    setresuid -.->|"r, e, s"| effective
    setresuid -.->|"r, e, s"| saved
``

**Hình 9-1:** Tác động của các hàm thay đổi thông tin xác thực lên user ID của process

**Bảng 9-1:** Tóm tắt các giao diện dùng để thay đổi thông tin xác thực của process

| Giao diện                                | Mục đích và tác động trong:                                                                                                                                              | Tính di động                                                                         |                                                                          |
|------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------|--------------------------------------------------------------------------|
|                                          | process không đặc quyền                                                                                                                                                    | process đặc quyền                                                                  |                                                                          |
| setuid(u)<br>setgid(g)                   | Thay đổi effective ID thành<br>cùng giá trị với real hoặc<br>saved set ID hiện tại                                                                                             | Thay đổi real,<br>effective và<br>saved set ID thành<br>bất kỳ giá trị (duy nhất)            | Được quy định trong SUSv3;<br>các dẫn xuất BSD<br>có ngữ nghĩa khác    |
| seteuid(e)<br>setegid(e)                 | Thay đổi effective ID thành<br>cùng giá trị với real hoặc<br>saved set ID hiện tại                                                                                             | Thay đổi effective<br>ID thành bất kỳ giá trị nào                                                 | Được quy định trong SUSv3                                                       |
| setreuid(r, e)<br>setregid(r, e)         | (Độc lập) thay đổi<br>real ID thành cùng giá trị với<br>real hoặc effective ID hiện tại,<br>và effective ID thành<br>cùng giá trị với real, effective<br>hoặc saved set ID hiện tại | (Độc lập)<br>thay đổi real và<br>effective ID thành<br>bất kỳ giá trị nào                | Được quy định trong SUSv3,<br>nhưng hoạt động<br>khác nhau giữa<br>các cài đặt |
| setresuid(r, e, s)<br>setresgid(r, e, s) | (Độc lập) thay đổi<br>real, effective và saved<br>set ID thành cùng giá trị với<br>real, effective hoặc<br>saved set ID hiện tại                                                         | (Độc lập)<br>thay đổi real,<br>effective và<br>saved set ID thành<br>bất kỳ giá trị nào | Không có trong SUSv3 và<br>chỉ có trên một vài<br>cài đặt UNIX khác      |
| setfsuid(u)<br>setfsgid(u)               | Thay đổi file-system ID thành<br>cùng giá trị với real, effective,<br>file-system hoặc saved set ID hiện tại                                                                   | Thay đổi file-system<br>ID thành bất kỳ giá trị nào                                               | Đặc thù của Linux                                                           |
| setgroups(n, l)                          | Không thể gọi từ<br>process không đặc quyền                                                                                                                                | Đặt supplementary<br>group ID thành bất kỳ<br>giá trị nào                                     | Không có trong SUSv3, nhưng<br>có sẵn trên tất cả cài đặt UNIX            |

Lưu ý thông tin bổ sung sau đây cho Bảng 9-1:

- Các cài đặt glibc của `seteuid()` (như `setresuid(–1, e, –1)`) và `setegid()` (như `setregid(–1, e)`) cũng cho phép đặt effective ID thành cùng giá trị mà nó đã có, nhưng điều này không được quy định trong SUSv3. Cài đặt `setegid()` cũng thay đổi saved set-group-ID nếu effective user ID được đặt thành giá trị khác với real user ID hiện tại. (SUSv3 không quy định rằng `setegid()` thực hiện thay đổi đối với saved set-group-ID.)
- Đối với các cuộc gọi đến `setreuid()` và `setregid()` bởi cả process có đặc quyền và không có đặc quyền, nếu `r` không phải –1, hoặc `e` được chỉ định là giá trị khác với real ID trước khi gọi, thì saved set-user-ID hoặc saved set-group-ID cũng được đặt thành cùng giá trị với effective ID (mới). (SUSv3 không quy định rằng `setreuid()` và `setregid()` thực hiện thay đổi đối với saved set ID.)
- Bất cứ khi nào effective user (group) ID thay đổi, file-system user (group) ID đặc thù của Linux cũng được thay đổi thành cùng giá trị.
- Các cuộc gọi đến `setresuid()` luôn sửa đổi file-system user ID để có cùng giá trị với effective user ID, bất kể effective user ID có được thay đổi bởi cuộc gọi hay không. Các cuộc gọi đến `setresgid()` có tác động tương tự đối với file-system group ID.

### **9.7.5 Ví Dụ: Hiển Thị Thông Tin Xác Thực Của Process**

Chương trình trong Listing 9-1 dùng các system call và library function được mô tả ở các trang trước để truy xuất tất cả user ID và group ID của process, rồi hiển thị chúng.

**Listing 9-1:** Hiển thị tất cả user ID và group ID của process

––––––––––––––––––––––––––––––––––––––––––––––––––––––––– **proccred/idshow.c** #define \_GNU\_SOURCE #include <unistd.h> #include <sys/fsuid.h> #include <limits.h> #include "ugid\_functions.h" /\* userNameFromId() & groupNameFromId() \*/ #include "tlpi\_hdr.h" #define SG\_SIZE (NGROUPS\_MAX + 1) int main(int argc, char \*argv[]) { uid\_t ruid, euid, suid, fsuid; gid\_t rgid, egid, sgid, fsgid; gid\_t suppGroups[SG\_SIZE]; int numGroups, j; char \*p; if (getresuid(&ruid, &euid, &suid) == -1) errExit("getresuid"); if (getresgid(&rgid, &egid, &sgid) == -1) errExit("getresgid"); /\* Attempts to change the file-system IDs are always ignored for unprivileged processes, but even so, the following calls return the current file-system IDs \*/ fsuid = setfsuid(0); fsgid = setfsgid(0); printf("UID: "); p = userNameFromId(ruid); printf("real=%s (%ld); ", (p == NULL) ? "???" : p, (long) ruid); p = userNameFromId(euid); printf("eff=%s (%ld); ", (p == NULL) ? "???" : p, (long) euid); p = userNameFromId(suid); printf("saved=%s (%ld); ", (p == NULL) ? "???" : p, (long) suid); p = userNameFromId(fsuid); printf("fs=%s (%ld); ", (p == NULL) ? "???" : p, (long) fsuid); printf("\n"); printf("GID: "); p = groupNameFromId(rgid);

printf("real=%s (%ld); ", (p == NULL) ? "???" : p, (long) rgid);

```
 p = groupNameFromId(egid);
 printf("eff=%s (%ld); ", (p == NULL) ? "???" : p, (long) egid);
 p = groupNameFromId(sgid);
 printf("saved=%s (%ld); ", (p == NULL) ? "???" : p, (long) sgid);
 p = groupNameFromId(fsgid);
 printf("fs=%s (%ld); ", (p == NULL) ? "???" : p, (long) fsgid);
 printf("\n");
 numGroups = getgroups(SG_SIZE, suppGroups);
 if (numGroups == -1)
 errExit("getgroups");
 printf("Supplementary groups (%d): ", numGroups);
 for (j = 0; j < numGroups; j++) {
 p = groupNameFromId(suppGroups[j]);
 printf("%s (%ld) ", (p == NULL) ? "???" : p, (long) suppGroups[j]);
 }
 printf("\n");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– proccred/idshow.c
```

## **9.8 Tóm Tắt**

Mỗi process có một số user ID và group ID liên kết (thông tin xác thực). Real ID xác định quyền sở hữu của process. Trên hầu hết các cài đặt UNIX, effective ID được dùng để xác định quyền của process khi truy cập tài nguyên như file. Tuy nhiên trên Linux, file-system ID được dùng để xác định quyền truy cập file, trong khi effective ID được dùng cho các kiểm tra quyền khác. (Vì file-system ID thường có cùng giá trị với effective ID tương ứng, Linux hoạt động giống như các cài đặt UNIX khác khi kiểm tra quyền file.) Supplementary group ID của process là tập hợp bổ sung các nhóm mà process được coi là thành viên cho mục đích kiểm tra quyền. Nhiều system call và library function cho phép process truy xuất và thay đổi user ID và group ID của nó.

Khi chương trình set-user-ID được chạy, effective user ID của process được đặt thành của chủ sở hữu file. Cơ chế này cho phép người dùng giả định danh tính, và do đó đặc quyền, của người dùng khác khi chạy một chương trình cụ thể. Tương ứng, các chương trình set-group-ID thay đổi effective group ID của process đang chạy một chương trình. Saved set-user-ID và saved set-group-ID cho phép các chương trình set-user-ID và set-group-ID tạm thời hủy bỏ và sau đó lấy lại đặc quyền.

User ID 0 là đặc biệt. Thông thường, một tài khoản người dùng duy nhất, có tên root, có user ID này. Các process có effective user ID là 0 được đặc quyền—tức là chúng được miễn nhiều kiểm tra quyền thường được thực hiện khi process thực hiện các system call khác nhau (như những system call dùng để tùy ý thay đổi user ID và group ID khác nhau của process).

## **9.9 Bài Tập**

- **9-1.** Giả sử trong mỗi trường hợp sau, tập hợp ban đầu của user ID của process là real=1000 effective=0 saved=0 file-system=0. Trạng thái của user ID sẽ ra sao sau các cuộc gọi sau?
  - a) `setuid(2000)`;
  - b) `setreuid(–1, 2000)`;
  - c) `seteuid(2000)`;
  - d) `setfsuid(2000)`;
  - e) `setresuid(–1, 2000, 3000)`;
- **9-2.** Process có các user ID sau có được đặc quyền không? Giải thích câu trả lời của bạn.

real=0 effective=1000 saved=1000 file-system=1000

- **9-3.** Cài đặt `initgroups()` sử dụng `setgroups()` và các library function để truy xuất thông tin từ file mật khẩu và nhóm (Mục 8.4). Nhớ rằng process phải có đặc quyền để có thể gọi `setgroups()`.
- **9-4.** Nếu một process có tất cả user ID bằng X thực thi chương trình set-user-ID có user ID là Y (khác 0), thì thông tin xác thực của process được đặt như sau:

real=X effective=Y saved=Y

(Chúng ta bỏ qua file-system user ID vì nó theo effective user ID.) Hãy cho thấy các cuộc gọi `setuid()`, `seteuid()`, `setreuid()` và `setresuid()` tương ứng sẽ được dùng để thực hiện các thao tác sau:

- a) Tạm dừng và tiếp tục danh tính set-user-ID (tức là chuyển effective user ID thành giá trị của real user ID và sau đó trở lại saved set-user-ID).
- b) Hủy bỏ vĩnh viễn danh tính set-user-ID (tức là đảm bảo rằng effective user ID và saved set-user-ID được đặt thành giá trị của real user ID).

(Bài tập này cũng yêu cầu sử dụng `getuid()` và `geteuid()` để truy xuất real và effective user ID của process.) Lưu ý rằng đối với một số system call được liệt kê ở trên, một số trong các thao tác này không thể thực hiện được.

**9-5.** Lặp lại bài tập trước cho process thực thi chương trình set-user-ID-root, có tập hợp ban đầu thông tin xác thực như sau:

real=X effective=0 saved=0
