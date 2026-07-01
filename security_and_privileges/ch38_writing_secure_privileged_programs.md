## Chương 38
# Viết Chương Trình Có Đặc Quyền An Toàn

Các chương trình có đặc quyền (privileged programs) có quyền truy cập vào các tính năng và tài nguyên (file, thiết bị, v.v.) mà người dùng thông thường không có. Một chương trình có thể chạy với đặc quyền theo hai cách chung:

- Chương trình được khởi chạy dưới một user ID có đặc quyền. Nhiều daemon và network server — thường chạy với quyền root — thuộc loại này.
- Chương trình có bit permission set-user-ID hoặc set-group-ID được bật. Khi một chương trình setuser-ID (set-group-ID) được exec, nó thay đổi effective user (group) ID của process thành giá trị bằng với owner (group) của file chương trình. (Chúng ta đã mô tả các chương trình set-user-ID và set-group-ID lần đầu trong Mục 9.3.) Trong chương này, đôi khi chúng ta dùng thuật ngữ set-user-ID-root để phân biệt một chương trình set-user-ID cấp quyền superuser cho một process với một chương trình cấp cho process một danh tính effective khác.

Nếu một chương trình có đặc quyền chứa lỗi, hoặc có thể bị người dùng độc hại khai thác, thì tính bảo mật của hệ thống hoặc ứng dụng có thể bị xâm phạm. Từ góc độ bảo mật, chúng ta nên viết chương trình sao cho giảm thiểu cả khả năng bị xâm phạm lẫn thiệt hại có thể xảy ra nếu việc xâm phạm thực sự xảy ra. Các chủ đề này là nội dung của chương này, trình bày một tập hợp các thực hành được khuyến nghị cho lập trình bảo mật, và mô tả các bẫy cần tránh khi viết các chương trình có đặc quyền.

### **38.1 Có Cần Chương Trình Set-User-ID hay Set-Group-ID Không?**

Một trong những lời khuyên tốt nhất liên quan đến các chương trình set-user-ID và set-group-ID là hãy tránh viết chúng bất cứ khi nào có thể. Nếu có cách thay thế để thực hiện một tác vụ mà không cần cấp đặc quyền cho chương trình, chúng ta thường nên sử dụng cách đó, vì nó loại bỏ khả năng xảy ra vi phạm bảo mật.

Đôi khi, chúng ta có thể tách chức năng cần đặc quyền thành một chương trình riêng biệt thực hiện một tác vụ duy nhất, và exec chương trình đó trong một child process khi cần. Kỹ thuật này đặc biệt hữu ích cho các thư viện. Một ví dụ về cách dùng như vậy là chương trình `pt_chown` được mô tả trong Mục 64.2.2.

Ngay cả trong những trường hợp cần set-user-ID hoặc set-group-ID, không phải lúc nào một chương trình set-user-ID cũng cần cấp cho process quyền root. Nếu việc cấp cho process một số thông tin xác thực khác là đủ, thì tùy chọn đó nên được ưu tiên, vì chạy với quyền root mở ra cơ hội cho các vi phạm bảo mật tiềm năng.

Xét một chương trình set-user-ID cần cho phép người dùng cập nhật một file mà họ không có quyền ghi. Cách an toàn hơn là tạo một tài khoản group chuyên dụng (group ID) cho chương trình này, thay đổi group ownership của file sang group đó (và làm cho file có thể ghi được bởi group đó), và viết một chương trình setgroup-ID thiết lập effective group ID của process thành group ID chuyên dụng. Vì group ID chuyên dụng không có đặc quyền gì khác, điều này hạn chế đáng kể thiệt hại có thể xảy ra nếu chương trình chứa lỗi hoặc bị khai thác theo cách khác.

## **38.2 Hoạt Động với Ít Đặc Quyền Nhất**

Một chương trình set-user-ID (hoặc set-group-ID) thường chỉ cần đặc quyền để thực hiện một số thao tác nhất định. Trong khi chương trình (đặc biệt là chương trình giả định quyền superuser) đang thực hiện các công việc khác, nó nên vô hiệu hóa các đặc quyền này. Khi đặc quyền không còn cần thiết nữa, chúng nên được bỏ vĩnh viễn. Nói cách khác, chương trình luôn nên hoạt động với ít đặc quyền nhất cần thiết để thực hiện các tác vụ hiện tại của nó. Tính năng saved set-user-ID được thiết kế cho mục đích này (Mục 9.4).

### **Chỉ giữ đặc quyền khi cần thiết**

Trong một chương trình set-user-ID, chúng ta có thể sử dụng chuỗi các lời gọi `seteuid()` sau để tạm thời bỏ và sau đó lấy lại đặc quyền:

```
uid_t orig_euid;
orig_euid = geteuid();
if (seteuid(getuid()) == -1) /* Drop privileges */
 errExit("seteuid");
/* Do unprivileged work */
if (seteuid(orig_euid) == -1) /* Reacquire privileges */
 errExit("seteuid");
/* Do privileged work */
```

Lời gọi đầu tiên làm cho effective user ID của process gọi bằng với real ID của nó. Lời gọi thứ hai khôi phục effective user ID về giá trị được lưu trong saved set-user-ID.

Đối với các chương trình set-group-ID, saved set-group-ID lưu effective group ID ban đầu của chương trình, và `setegid()` được dùng để bỏ và lấy lại đặc quyền. Chúng ta mô tả `seteuid()`, `setegid()`, và các system call tương tự được đề cập trong các khuyến nghị tiếp theo trong Chương 9 và tóm tắt chúng trong Bảng 9-1 (trang 181).

Thực hành an toàn nhất là bỏ đặc quyền ngay khi khởi động chương trình, và sau đó tạm thời lấy lại chúng khi cần tại các điểm sau trong chương trình. Nếu tại một thời điểm nhất định, đặc quyền sẽ không bao giờ cần lại nữa, thì chương trình nên bỏ chúng vĩnh viễn, bằng cách đảm bảo rằng saved set-user-ID cũng được thay đổi. Điều này loại bỏ khả năng chương trình bị đánh lừa để lấy lại đặc quyền, có thể thông qua kỹ thuật stack crashing được mô tả trong Mục 38.9.

#### **Bỏ đặc quyền vĩnh viễn khi không còn cần thiết**

Nếu một chương trình set-user-ID hoặc set-group-ID đã hoàn thành tất cả các tác vụ cần đặc quyền, thì nó nên bỏ đặc quyền của mình vĩnh viễn để loại bỏ bất kỳ rủi ro bảo mật nào có thể xảy ra vì chương trình bị xâm phạm bởi lỗi hoặc hành vi bất ngờ khác. Bỏ đặc quyền vĩnh viễn được thực hiện bằng cách đặt lại tất cả các user (group) ID của process về cùng giá trị với real (group) ID.

Từ một chương trình set-user-ID-root có effective user ID hiện tại là 0, chúng ta có thể đặt lại tất cả user ID bằng đoạn mã sau:

```
if (setuid(getuid()) == -1)
 errExit("setuid");
```

Tuy nhiên, đoạn mã trên không đặt lại saved set-user-ID nếu effective user ID của process đang gọi hiện tại khác 0: khi được gọi từ một chương trình có effective user ID khác 0, `setuid()` chỉ thay đổi effective user ID (Mục 9.7.1). Nói cách khác, trong một chương trình set-user-ID-root, chuỗi lệnh sau không bỏ user ID 0 vĩnh viễn:

```
/* Initial UIDs: real=1000 effective=0 saved=0 */
/* 1. Usual call to temporarily drop privilege */
orig_euid = geteuid();
if (seteuid(getuid() == -1)
 errExit("seteuid");
/* UIDs changed to: real=1000 effective=1000 saved=0 */
/* 2. Looks like the right way to permanently drop privilege (WRONG!) */
if (setuid(getuid() == -1)
 errExit("setuid");
/* UIDs unchanged: real=1000 effective=1000 saved=0 */
```

Thay vào đó, chúng ta phải lấy lại đặc quyền trước khi bỏ nó vĩnh viễn, bằng cách chèn lời gọi sau đây giữa bước 1 và bước 2 ở trên:

```
if (seteuid(orig_euid) == -1)
 errExit("seteuid");
```

Mặt khác, nếu chúng ta có một chương trình set-user-ID thuộc sở hữu của một người dùng khác root, thì vì `setuid()` không đủ để thay đổi định danh set-user-ID, chúng ta phải dùng `setreuid()` hoặc `setresuid()` để bỏ vĩnh viễn định danh có đặc quyền. Ví dụ, chúng ta có thể đạt được kết quả mong muốn bằng cách dùng `setreuid()` như sau:

```
if (setreuid(getuid(), getuid()) == -1)
 errExit("setreuid");
```

Đoạn mã này dựa vào một tính năng của hiện thực Linux của `setreuid()`: nếu đối số đầu tiên (ruid) không phải –1, thì saved set-user-ID cũng được đặt về cùng giá trị với effective user ID (mới). SUSv3 không chỉ định tính năng này, nhưng nhiều hiện thực khác hoạt động giống Linux.

System call `setregid()` hoặc `setresgid()` cũng phải được sử dụng để bỏ vĩnh viễn một group ID có đặc quyền trong một chương trình set-group-ID, vì khi effective user ID của chương trình khác 0, `setgid()` chỉ thay đổi effective group ID của process đang gọi.

#### **Các lưu ý chung về thay đổi thông tin xác thực của process**

Trong các trang trước, chúng ta đã mô tả các kỹ thuật để tạm thời và vĩnh viễn bỏ đặc quyền. Bây giờ chúng ta bổ sung một số điểm chung liên quan đến việc sử dụng các kỹ thuật này:

- Ngữ nghĩa của một số system call thay đổi thông tin xác thực của process khác nhau giữa các hệ thống. Hơn nữa, ngữ nghĩa của một số system call này thay đổi tùy thuộc vào việc người gọi có hay không có đặc quyền (effective user ID bằng 0). Để biết chi tiết, xem Chương 9, đặc biệt là Mục 9.7.4. Do những khác biệt này, [Tsafrir et al., 2008] khuyến nghị rằng ứng dụng nên sử dụng các system call không chuẩn đặc thù của hệ thống để thay đổi thông tin xác thực của process, vì trong nhiều trường hợp, các system call không chuẩn này cung cấp ngữ nghĩa đơn giản và nhất quán hơn so với các đối tác chuẩn của chúng. Trên Linux, điều này có nghĩa là sử dụng `setresuid()` và `setresgid()` để thay đổi thông tin xác thực user và group. Mặc dù các system call này không có mặt trên tất cả các hệ thống, việc sử dụng chúng ít có khả năng gây ra lỗi hơn. ([Tsafrir et al., 2008] đề xuất một thư viện các hàm thực hiện thay đổi thông tin xác thực bằng cách sử dụng những gì họ cho là giao diện tốt nhất có sẵn trên mỗi nền tảng.)
- Trên Linux, ngay cả khi người gọi có effective user ID bằng 0, các system call để thay đổi thông tin xác thực có thể không hoạt động như mong đợi nếu chương trình đã thao tác rõ ràng các capabilities của nó. Ví dụ, nếu capability `CAP_SETUID` đã bị vô hiệu hóa, thì các lần thử thay đổi user ID của process sẽ thất bại hoặc, thậm chí tệ hơn, âm thầm chỉ thay đổi một số user ID được yêu cầu.
- Do các khả năng được liệt kê trong hai điểm trước, thực hành được khuyến nghị cao (xem, ví dụ, [Tsafrir et al., 2008]) là không chỉ kiểm tra rằng một system call thay đổi thông tin xác thực đã thành công, mà còn xác minh rằng sự thay đổi đã xảy ra như mong đợi. Ví dụ, nếu chúng ta đang tạm thời bỏ hoặc lấy lại một user ID có đặc quyền bằng cách dùng `seteuid()`, thì chúng ta nên theo sau lời gọi đó bằng một lời gọi `geteuid()` để xác minh rằng effective user ID là như mong đợi. Tương tự, nếu chúng ta đang bỏ vĩnh viễn một user ID có đặc quyền, thì chúng ta nên xác minh rằng real user ID, effective user ID, và saved set-user-ID đã được thay đổi thành công về user ID không có đặc quyền. Thật không may, trong khi có các system call chuẩn để lấy real và effective ID, không có các system call chuẩn để lấy saved set ID. Linux và một vài hệ thống khác cung cấp `getresuid()` và `getresgid()` cho mục đích này; trên một số hệ thống khác, chúng ta có thể cần sử dụng các kỹ thuật như phân tích thông tin trong các file /proc.
- Một số thay đổi thông tin xác thực chỉ có thể được thực hiện bởi các process có effective user ID bằng 0. Do đó, khi thay đổi nhiều ID — supplementary group ID, group ID, và user ID — chúng ta nên bỏ effective user ID có đặc quyền cuối cùng khi bỏ các ID có đặc quyền. Ngược lại, chúng ta nên nâng effective user ID có đặc quyền lên đầu tiên khi nâng các ID có đặc quyền.

### **38.3 Cẩn Thận Khi Thực Thi Một Chương Trình**

Cần thận trọng khi một chương trình có đặc quyền thực thi một chương trình khác, trực tiếp qua `exec()`, hoặc gián tiếp qua `system()`, `popen()`, hoặc một library function tương tự.

### **Bỏ đặc quyền vĩnh viễn trước khi exec một chương trình khác**

Nếu một chương trình set-user-ID (hoặc set-group-ID) thực thi một chương trình khác, thì nó nên đảm bảo rằng tất cả user (group) ID của process được đặt lại về cùng giá trị với real user (group) ID, để chương trình mới không bắt đầu với đặc quyền và cũng không thể lấy lại chúng. Một cách để làm điều này là đặt lại tất cả các ID trước khi thực hiện `exec()`, sử dụng các kỹ thuật được mô tả trong Mục 38.2.

Kết quả tương tự có thể đạt được bằng cách gọi `setuid(getuid())` trước `exec()`. Mặc dù lời gọi `setuid()` này chỉ thay đổi effective user ID trong một process có effective user ID khác 0, đặc quyền vẫn bị bỏ vì (như mô tả trong Mục 9.4) một lần `exec()` thành công sẽ sao chép effective user ID vào saved set-user-ID. (Nếu `exec()` thất bại, thì saved set-user-ID không thay đổi. Điều này có thể hữu ích nếu chương trình sau đó cần thực hiện công việc có đặc quyền khác vì `exec()` thất bại.)

Một cách tiếp cận tương tự (tức là, `setgid(getgid())`) có thể được sử dụng với các chương trình set-group-ID, vì một lần `exec()` thành công cũng sao chép effective group ID vào saved set-group-ID.

Ví dụ, giả sử chúng ta có một chương trình set-user-ID thuộc sở hữu của user ID 200. Khi chương trình này được thực thi bởi một người dùng có ID là 1000, các user ID của process kết quả sẽ như sau:

```
real=1000 effective=200 saved=200
```

Nếu chương trình này sau đó thực thi lời gọi `setuid(getuid())`, thì các user ID của process được thay đổi thành:

```
real=1000 effective=1000 saved=200
```

Khi process thực thi một chương trình không có đặc quyền, effective user ID của process được sao chép vào saved set-user-ID, dẫn đến tập hợp các user ID của process sau:

```
real=1000 effective=1000 saved=1000
```

#### **Tránh thực thi shell (hoặc trình thông dịch khác) với đặc quyền**

Các chương trình có đặc quyền chạy dưới sự kiểm soát của người dùng không bao giờ được exec một shell, trực tiếp hoặc gián tiếp (qua `system()`, `popen()`, `execlp()`, `execvp()`, hoặc các library function tương tự khác). Sự phức tạp và sức mạnh của shell (và các trình thông dịch không bị ràng buộc khác như awk) có nghĩa là gần như không thể loại bỏ tất cả các lỗ hổng bảo mật, ngay cả khi shell được exec không cho phép truy cập tương tác. Rủi ro kéo theo là người dùng có thể thực thi các lệnh shell tùy ý dưới effective user ID của process. Nếu phải exec một shell, hãy đảm bảo đặc quyền đã được bỏ vĩnh viễn trước đó.

> Một ví dụ về loại lỗ hổng bảo mật có thể xảy ra khi exec một shell được ghi chú trong phần thảo luận về `system()` trong Mục 27.6.

Một vài hiện thực UNIX tôn trọng các bit permission set-user-ID và set-group-ID khi chúng được áp dụng cho các script của trình thông dịch (Mục 27.3), để khi script được chạy, process thực thi script giả định danh tính của một người dùng (có đặc quyền) khác. Do những rủi ro bảo mật vừa mô tả, Linux, giống như một số hiện thực UNIX khác, âm thầm bỏ qua các bit permission set-user-ID và set-group-ID khi exec một script. Ngay cả trên các hiện thực cho phép các script set-user-ID và set-group-ID, việc sử dụng chúng cũng nên được tránh.

### **Đóng tất cả các file descriptor không cần thiết trước khi exec()**

Trong Mục 27.4, chúng ta đã lưu ý rằng, theo mặc định, các file descriptor vẫn mở qua một `exec()`. Một chương trình có đặc quyền có thể mở một file mà các process thông thường không thể truy cập. File descriptor đang mở kết quả đại diện cho một tài nguyên có đặc quyền. File descriptor nên được đóng trước một `exec()`, để chương trình được exec không thể truy cập file liên kết. Chúng ta có thể làm điều này bằng cách đóng rõ ràng file descriptor hoặc bằng cách đặt flag close-on-exec của nó (Mục 27.4).

## **38.4 Tránh Tiết Lộ Thông Tin Nhạy Cảm**

Khi một chương trình đọc password hoặc thông tin nhạy cảm khác, nó nên thực hiện bất kỳ xử lý nào cần thiết, và sau đó xóa ngay thông tin đó khỏi bộ nhớ. (Chúng ta trình bày một ví dụ về điều này trong Mục 8.5.) Để thông tin như vậy trong bộ nhớ là một rủi ro bảo mật, vì những lý do sau:

- Trang virtual memory chứa dữ liệu có thể bị swap ra (trừ khi nó được khóa trong bộ nhớ bằng `mlock()` hoặc tương tự), và sau đó có thể được đọc từ vùng swap bởi một chương trình có đặc quyền.
- Nếu process nhận được một signal khiến nó tạo ra một file core dump, thì file đó có thể được đọc để lấy thông tin.

Tiếp theo điểm cuối cùng, với tư cách là nguyên tắc chung, một chương trình an toàn nên ngăn các core dump, để một file core dump không thể được kiểm tra để lấy thông tin nhạy cảm. Một chương trình có thể đảm bảo rằng file core dump không được tạo bằng cách sử dụng `setrlimit()` để đặt giới hạn tài nguyên `RLIMIT_CORE` về 0 (xem Mục 36.3).

> Theo mặc định, Linux không cho phép một chương trình set-user-ID thực hiện core dump khi phản hồi một signal (Mục 22.1), ngay cả khi chương trình đã bỏ tất cả đặc quyền. Tuy nhiên, các hiện thực UNIX khác có thể không cung cấp tính năng bảo mật này.

### **38.5 Cô Lập Process**

Trong phần này, chúng ta xem xét các cách có thể cô lập một chương trình để hạn chế thiệt hại được thực hiện nếu chương trình bị xâm phạm.

### **Cân nhắc sử dụng capabilities**

Cơ chế Linux capabilities chia cơ chế đặc quyền tất cả hoặc không có gì của UNIX truyền thống thành các đơn vị riêng biệt được gọi là capabilities. Một process có thể độc lập bật hoặc tắt từng capability riêng lẻ. Bằng cách chỉ bật những capabilities mà nó cần, một chương trình hoạt động với ít đặc quyền hơn so với khi chạy với quyền root đầy đủ. Điều này giảm tiềm năng gây thiệt hại nếu chương trình bị xâm phạm.

Hơn nữa, sử dụng capabilities và các flag securebits, chúng ta có thể tạo một process có một tập hợp hạn chế các capabilities nhưng không thuộc sở hữu của root (tức là, tất cả user ID của nó đều khác 0). Một process như vậy không thể còn dùng `exec()` để lấy lại một tập đầy đủ các capabilities. Chúng ta mô tả capabilities và các flag securebits trong Chương 39.

### **Cân nhắc sử dụng chroot jail**

Một kỹ thuật bảo mật hữu ích trong một số trường hợp là thiết lập một chroot jail để hạn chế tập hợp các thư mục và file mà một chương trình có thể truy cập. (Hãy chắc chắn cũng gọi `chdir()` để thay đổi thư mục làm việc hiện tại của process sang một vị trí trong jail.) Tuy nhiên, lưu ý rằng chroot jail không đủ để cô lập một chương trình set-user-ID-root (xem Mục 18.12).

> Một lựa chọn thay thế cho việc sử dụng chroot jail là virtual server, là một server được triển khai trên một virtual kernel. Vì mỗi virtual kernel được cô lập khỏi các virtual kernel khác có thể chạy trên cùng phần cứng, một virtual server an toàn và linh hoạt hơn chroot jail. (Một số hệ điều hành hiện đại khác cũng cung cấp các hiện thực riêng của virtual server.) Hiện thực ảo hóa lâu đời nhất trên Linux là User-Mode Linux (UML), là một phần chuẩn của Linux 2.6 kernel. Thông tin thêm về UML có thể được tìm thấy tại http://user-mode-linux.sourceforge.net/. Các dự án virtual kernel gần đây hơn bao gồm Xen (http://www.cl.cam.ac.uk/Research/SRG/netos/xen/) và KVM (http://kvm.qumranet.com/).

### **38.6 Chú Ý Đến Signals và Race Conditions**

Người dùng có thể gửi các signal tùy ý đến một chương trình set-user-ID mà họ đã khởi động. Các signal như vậy có thể đến bất kỳ lúc nào và với bất kỳ tần suất nào. Chúng ta cần xem xét các race condition có thể xảy ra nếu một signal được phân phối tại bất kỳ điểm nào trong quá trình thực thi chương trình. Khi thích hợp, các signal nên được bắt, chặn, hoặc bỏ qua để ngăn các vấn đề bảo mật có thể xảy ra. Hơn nữa, thiết kế của các signal handler nên càng đơn giản càng tốt, để giảm nguy cơ vô tình tạo ra race condition.

Vấn đề này đặc biệt liên quan đến các signal dừng một process (ví dụ: `SIGTSTP` và `SIGSTOP`). Tình huống có vấn đề như sau:

1. Một chương trình set-user-ID xác định một số thông tin về môi trường thời gian chạy của nó.
2. Người dùng quản lý để dừng process đang chạy chương trình và thay đổi chi tiết của môi trường thời gian chạy. Những thay đổi như vậy có thể bao gồm sửa đổi quyền trên một file, thay đổi mục tiêu của một symbolic link, hoặc xóa một file mà chương trình phụ thuộc vào.
3. Người dùng tiếp tục process với signal `SIGCONT`. Tại điểm này, chương trình sẽ tiếp tục thực thi dựa trên các giả định hiện giờ sai về môi trường thời gian chạy của nó, và những giả định này có thể dẫn đến vi phạm bảo mật.

Tình huống được mô tả ở đây thực sự chỉ là một trường hợp đặc biệt của race condition time-of-check, time-of-use. Một process có đặc quyền nên tránh thực hiện các thao tác dựa trên các xác minh trước đó có thể không còn giữ nữa (tham khảo cuộc thảo luận về system call `access()` trong Mục 15.4.4 để biết ví dụ cụ thể). Hướng dẫn này áp dụng ngay cả khi người dùng không thể gửi signal đến process. Khả năng dừng một process chỉ đơn giản cho phép người dùng mở rộng khoảng thời gian giữa thời điểm kiểm tra và thời điểm sử dụng.

> Mặc dù có thể khó dừng một process giữa thời điểm kiểm tra và thời điểm sử dụng trong một lần thử duy nhất, người dùng độc hại có thể thực thi một chương trình set-user-ID nhiều lần, và sử dụng một chương trình khác hoặc một shell script để liên tục gửi signal dừng đến chương trình set-user-ID và thay đổi môi trường thời gian chạy. Điều này cải thiện đáng kể khả năng phá hoại chương trình set-user-ID.

## **38.7 Các Bẫy Khi Thực Hiện Các Thao Tác File và File I/O**

Nếu một process có đặc quyền cần tạo một file, thì chúng ta phải chú ý đến ownership và permissions của file đó để đảm bảo rằng không bao giờ có một thời điểm, dù ngắn đến đâu, khi file dễ bị thao tác độc hại. Các hướng dẫn sau áp dụng:

- Process umask (Mục 15.4.6) nên được đặt thành một giá trị đảm bảo rằng process không bao giờ tạo các file có thể ghi công khai, vì những file này có thể bị sửa đổi bởi người dùng độc hại.
- Vì ownership của file được lấy từ effective user ID của process tạo ra nó, việc sử dụng cẩn thận `seteuid()` hoặc `setreuid()` để tạm thời thay đổi thông tin xác thực của process có thể cần thiết để đảm bảo rằng một file mới tạo không thuộc về người dùng sai. Vì group ownership của file có thể được lấy

từ effective group ID của process (xem Mục 15.3.1), một tuyên bố tương tự áp dụng với các chương trình set-group-ID, và các lời gọi group ID tương ứng có thể được sử dụng để tránh các vấn đề như vậy. (Để chính xác nghiêm ngặt, trên Linux, owner của một file mới được xác định bởi file-system user ID của process, thường có cùng giá trị với effective user ID của process; tham khảo Mục 9.5.)

- Nếu một chương trình set-user-ID-root phải tạo một file mà ban đầu nó phải sở hữu, nhưng cuối cùng sẽ thuộc sở hữu của người dùng khác, thì file nên được tạo ra để ban đầu không thể ghi được bởi những người dùng khác, bằng cách sử dụng đối số mode phù hợp cho `open()` hoặc bằng cách đặt process umask trước khi gọi `open()`. Sau đó, chương trình có thể thay đổi ownership của nó bằng `fchown()`, và sau đó thay đổi permissions của nó, nếu cần, bằng `fchmod()`. Điểm mấu chốt là một chương trình set-user-ID nên đảm bảo rằng nó không bao giờ tạo ra một file thuộc sở hữu của owner của chương trình và có thể ghi được bởi những người dùng khác dù chỉ trong một khoảnh khắc.
- Kiểm tra các thuộc tính file nên được thực hiện trên các file descriptor đang mở (ví dụ: `open()` theo sau bởi `fstat()`), thay vì kiểm tra các thuộc tính liên quan đến một pathname và sau đó mở file (ví dụ: `stat()` theo sau bởi `open()`). Phương pháp sau tạo ra vấn đề time-of-use, time-of-check.
- Nếu một chương trình phải đảm bảo rằng nó là người tạo ra một file, thì flag `O_EXCL` nên được sử dụng khi gọi `open()`.
- Một chương trình có đặc quyền nên tránh tạo hoặc dựa vào các file trong các thư mục có thể ghi công khai như `/tmp`, vì điều này khiến chương trình dễ bị tấn công độc hại để tạo các file không được phép với tên mà chương trình có đặc quyền mong đợi. Một chương trình tuyệt đối phải tạo một file trong một thư mục có thể ghi công khai ít nhất nên đảm bảo rằng file có tên không thể đoán trước, bằng cách sử dụng một hàm như `mkstemp()` (Mục 5.12).

### **38.8 Đừng Tin Tưởng Các Input hoặc Môi Trường**

Các chương trình có đặc quyền nên tránh đưa ra các giả định về input chúng nhận được, hoặc môi trường mà chúng đang chạy.

#### **Đừng tin tưởng danh sách môi trường**

Các chương trình set-user-ID và set-group-ID không nên giả định rằng các giá trị của các biến môi trường là đáng tin cậy. Hai biến đặc biệt liên quan là `PATH` và `IFS`.

`PATH` xác định nơi shell (và do đó `system()` và `popen()`), cũng như `execlp()` và `execvp()`, tìm kiếm các chương trình. Người dùng độc hại có thể đặt `PATH` thành một giá trị có thể đánh lừa một chương trình set-user-ID sử dụng một trong các hàm này để thực thi một chương trình tùy ý với đặc quyền. Nếu các hàm này được sử dụng, `PATH` nên được đặt thành một danh sách các thư mục đáng tin cậy (nhưng tốt hơn là nên chỉ định pathname tuyệt đối khi exec các chương trình). Tuy nhiên, như đã lưu ý, tốt nhất là bỏ đặc quyền trước khi exec một shell hoặc sử dụng một trong các hàm đã đề cập.

`IFS` chỉ định các ký tự phân cách mà shell hiểu là phân tách các từ của một dòng lệnh. Biến này nên được đặt thành một chuỗi rỗng, có nghĩa là chỉ các ký tự khoảng trắng được shell hiểu là dấu phân cách từ. Một số shell luôn đặt `IFS` theo cách này khi khởi động. (Mục 27.6 mô tả một lỗ hổng liên quan đến `IFS` xuất hiện trong các phiên bản Bourne shell cũ hơn.)

Trong một số trường hợp, cách an toàn nhất có thể là xóa toàn bộ danh sách môi trường (Mục 6.7), và sau đó khôi phục các biến môi trường được chọn với các giá trị đã biết là an toàn, đặc biệt khi thực thi các chương trình khác hoặc gọi các thư viện có thể bị ảnh hưởng bởi các cài đặt biến môi trường.

### **Xử lý input của người dùng không tin cậy một cách phòng thủ**

Một chương trình có đặc quyền nên kiểm tra cẩn thận tất cả các input từ các nguồn không tin cậy trước khi thực hiện hành động dựa trên các input đó. Việc kiểm tra như vậy có thể bao gồm xác minh rằng các số nằm trong giới hạn chấp nhận được, và rằng các chuỗi có độ dài chấp nhận được và bao gồm các ký tự chấp nhận được. Trong số các input có thể cần được kiểm tra theo cách này là những input đến từ các file do người dùng tạo, đối số dòng lệnh, input tương tác, input CGI, thư email, biến môi trường, kênh interprocess communication (FIFO, shared memory, v.v.) có thể truy cập bởi người dùng không tin cậy, và các gói mạng.

#### **Tránh các giả định không đáng tin cậy về môi trường thời gian chạy của process**

Một chương trình set-user-ID nên tránh đưa ra các giả định không đáng tin cậy về môi trường thời gian chạy ban đầu của nó. Ví dụ, standard input, output, hoặc error có thể đã bị đóng. (Các descriptor này có thể đã bị đóng trong chương trình exec chương trình set-user-ID.) Trong trường hợp này, việc mở một file có thể vô tình tái sử dụng descriptor 1 (ví dụ), do đó, trong khi chương trình nghĩ rằng nó đang ghi vào standard output, nó thực sự đang ghi vào file mà nó mở.

Có nhiều khả năng khác cần xem xét. Ví dụ, một process có thể cạn kiệt các giới hạn tài nguyên khác nhau, chẳng hạn như giới hạn về số lượng process có thể được tạo, giới hạn tài nguyên thời gian CPU, hoặc giới hạn tài nguyên kích thước file, với kết quả là các system call khác nhau có thể thất bại hoặc các signal khác nhau có thể được tạo ra. Người dùng độc hại có thể cố tình tạo ra việc cạn kiệt tài nguyên trong một nỗ lực để phá hoại một chương trình.

## **38.9 Chú Ý Đến Buffer Overrun**

Hãy chú ý đến buffer overrun (overflow), nơi một giá trị input hoặc chuỗi được sao chép vượt quá không gian buffer được phân bổ. Không bao giờ sử dụng `gets()`, và sử dụng các hàm như `scanf()`, `sprintf()`, `strcpy()`, và `strcat()` một cách thận trọng (ví dụ: bảo vệ việc sử dụng của chúng bằng các câu lệnh if ngăn chặn buffer overrun).

Buffer overrun cho phép các kỹ thuật như stack crashing (còn gọi là stack smashing), theo đó người dùng độc hại sử dụng buffer overrun để đặt các byte được mã hóa cẩn thận vào một stack frame để buộc chương trình có đặc quyền thực thi mã tùy ý. (Một số nguồn trực tuyến giải thích chi tiết về stack crashing; xem thêm [Erickson, 2008] và [Anley, 2007].) Buffer overrun có lẽ là nguồn vi phạm bảo mật phổ biến nhất trên các hệ thống máy tính, như được thể hiện qua tần suất các cảnh báo được đăng bởi CERT (http://www.cert.org/) và trên Bugtraq (http://www.securityfocus.com/). Buffer overrun đặc biệt nguy hiểm trong các network server, vì chúng khiến hệ thống dễ bị tấn công từ xa từ bất kỳ đâu trên mạng. Để làm cho stack crashing khó hơn — cụ thể, để làm cho các cuộc tấn công như vậy tốn nhiều thời gian hơn khi tiến hành từ xa đối với các network server — từ kernel 2.6.12 trở đi, Linux thực hiện address-space randomization. Kỹ thuật này ngẫu nhiên hóa vị trí của stack trên phạm vi 8 MB ở đầu virtual memory. Ngoài ra, vị trí của các memory mapping cũng có thể được ngẫu nhiên hóa, nếu giới hạn mềm `RLIMIT_STACK` không phải là vô hạn và file Linux-specific `/proc/sys/vm/legacy_va_layout` chứa giá trị 0.

Các kiến trúc x86-32 gần đây hơn cung cấp hỗ trợ phần cứng để đánh dấu page table là NX ("no execute"). Tính năng này được sử dụng để ngăn chặn việc thực thi mã chương trình trên stack, do đó làm cho stack crashing khó hơn.

Có các lựa chọn thay thế an toàn cho nhiều hàm được đề cập ở trên — ví dụ: `snprintf()`, `strncpy()`, và `strncat()` — cho phép người gọi chỉ định số ký tự tối đa được sao chép. Các hàm này tính đến giới hạn tối đa được chỉ định để tránh tràn qua buffer mục tiêu. Nhìn chung, các lựa chọn thay thế này được ưa thích, nhưng vẫn phải được xử lý cẩn thận. Cụ thể, lưu ý các điểm sau:

- Với hầu hết các hàm này, nếu giới hạn tối đa được chỉ định đạt đến, thì một phiên bản bị cắt ngắn của chuỗi nguồn được đặt vào buffer mục tiêu. Vì chuỗi bị cắt ngắn như vậy có thể vô nghĩa về mặt ngữ nghĩa của chương trình, người gọi phải kiểm tra xem việc cắt ngắn có xảy ra không (ví dụ: sử dụng giá trị trả về từ `snprintf()`), và thực hiện hành động thích hợp nếu có.
- Sử dụng `strncpy()` có thể gây ra ảnh hưởng hiệu suất. Nếu trong lời gọi `strncpy(s1, s2, n)`, chuỗi được trỏ bởi s2 ngắn hơn n byte, thì các byte null đệm được ghi vào s1 để đảm bảo rằng tổng cộng n byte được ghi.
- Nếu giá trị kích thước tối đa được cho cho `strncpy()` không đủ dài để cho phép bao gồm ký tự null kết thúc, thì chuỗi mục tiêu không được kết thúc bằng null.

Một số hiện thực UNIX cung cấp hàm `strlcpy()`, khi được cho đối số độ dài n, sao chép nhiều nhất n – 1 byte vào buffer đích và luôn thêm ký tự null vào cuối buffer. Tuy nhiên, hàm này không được chỉ định trong SUSv3 và không được triển khai trong glibc. Hơn nữa, trong những trường hợp người gọi không kiểm tra cẩn thận độ dài chuỗi, hàm này chỉ thay thế một vấn đề (buffer overflow) bằng một vấn đề khác (âm thầm loại bỏ dữ liệu).

## **38.10 Chú Ý Đến Các Cuộc Tấn Công Từ Chối Dịch Vụ**

Với sự gia tăng của các dịch vụ dựa trên Internet đã đến tương ứng với sự gia tăng các cơ hội cho các cuộc tấn công từ chối dịch vụ từ xa. Các cuộc tấn công này cố gắng làm cho dịch vụ không khả dụng cho các client hợp lệ, bằng cách gửi cho server dữ liệu bị lỗi khiến nó bị crash hoặc bằng cách tràn ngập nó với các yêu cầu giả mạo.

> Các cuộc tấn công từ chối dịch vụ cục bộ cũng có thể xảy ra. Ví dụ nổi tiếng nhất là khi người dùng chạy một fork bomb đơn giản (một chương trình liên tục fork, do đó chiếm dụng tất cả các slot process trên hệ thống). Tuy nhiên, nguồn gốc của các cuộc tấn công từ chối dịch vụ cục bộ dễ xác định hơn nhiều, và chúng thường có thể được ngăn chặn bằng các biện pháp bảo mật vật lý và password thích hợp.

Xử lý các yêu cầu bị lỗi rất đơn giản — một server nên được lập trình để kiểm tra nghiêm ngặt các input của nó và tránh buffer overrun, như mô tả ở trên.

Các cuộc tấn công tải quá mức khó xử lý hơn. Vì server không thể kiểm soát hành vi của các client từ xa hoặc tốc độ mà chúng gửi yêu cầu, các cuộc tấn công như vậy không thể ngăn chặn. (Server thậm chí có thể không xác định được nguồn gốc thực sự của cuộc tấn công, vì địa chỉ IP nguồn của một gói mạng có thể bị giả mạo. Ngoài ra, các cuộc tấn công phân tán có thể tuyển dụng các host trung gian không biết gì để hướng cuộc tấn công vào hệ thống mục tiêu.) Tuy nhiên, có thể thực hiện các biện pháp khác nhau để giảm thiểu rủi ro và hậu quả của một cuộc tấn công tải quá mức:

- Server nên thực hiện load throttling, loại bỏ các yêu cầu khi tải vượt quá một giới hạn được định sẵn. Điều này sẽ có hậu quả là loại bỏ các yêu cầu hợp lệ, nhưng ngăn server và máy host bị quá tải. Việc sử dụng các giới hạn tài nguyên và disk quota cũng có thể hữu ích trong việc hạn chế tải quá mức. (Tham khảo http://sourceforge.net/projects/linuxquota/ để biết thêm thông tin về disk quota.)
- Một server nên sử dụng timeout cho giao tiếp với client, để nếu client (có thể cố ý) không phản hồi, server không bị ràng buộc vô thời hạn chờ đợi client.
- Trong trường hợp quá tải, server nên ghi lại các thông báo phù hợp để quản trị viên hệ thống được thông báo về vấn đề. (Tuy nhiên, việc ghi log cũng nên được điều tiết, để bản thân logging không làm quá tải hệ thống.)
- Server nên được lập trình để không bị crash trước tải bất ngờ. Ví dụ, bounds checking nên được thực hiện nghiêm ngặt để đảm bảo rằng các yêu cầu quá mức không làm tràn một cấu trúc dữ liệu.
- Các cấu trúc dữ liệu nên được thiết kế để tránh các cuộc tấn công độ phức tạp thuật toán. Ví dụ, một binary tree có thể được cân bằng và cung cấp hiệu suất chấp nhận được dưới các tải thông thường. Tuy nhiên, kẻ tấn công có thể xây dựng một chuỗi các input dẫn đến một cây không cân bằng (tương đương với một linked list trong trường hợp tệ nhất), điều này có thể làm tê liệt hiệu suất. [Crosby & Wallach, 2003] mô tả chi tiết bản chất của các cuộc tấn công như vậy và thảo luận về các kỹ thuật cấu trúc dữ liệu có thể được sử dụng để tránh chúng.

## **38.11 Kiểm Tra Trạng Thái Trả Về và Thất Bại An Toàn**

Một chương trình có đặc quyền luôn nên kiểm tra xem các system call và library function có thành công không, và liệu chúng có trả về các giá trị mong đợi không. (Điều này đúng với tất cả các chương trình, tất nhiên, nhưng đặc biệt quan trọng đối với các chương trình có đặc quyền.) Các system call khác nhau có thể thất bại, ngay cả đối với một chương trình đang chạy với quyền root. Ví dụ, `fork()` có thể thất bại nếu giới hạn toàn hệ thống về số lượng process được gặp, một `open()` để ghi có thể thất bại trên một file system chỉ đọc, hoặc `chdir()` có thể thất bại nếu thư mục mục tiêu không tồn tại.

Ngay cả khi một system call thành công, có thể cần kiểm tra kết quả của nó. Ví dụ, khi quan trọng, một chương trình có đặc quyền nên kiểm tra rằng một `open()` thành công không trả về một trong ba file descriptor chuẩn: 0, 1, hoặc 2.

Cuối cùng, nếu một chương trình có đặc quyền gặp một tình huống bất ngờ, thì hành vi thích hợp thường là kết thúc hoặc, trong trường hợp một server, loại bỏ yêu cầu client. Cố gắng sửa các vấn đề bất ngờ thường đòi hỏi phải đưa ra các giả định có thể không đúng trong mọi hoàn cảnh và có thể dẫn đến việc tạo ra các lỗ hổng bảo mật. Trong các tình huống như vậy, sẽ an toàn hơn khi để chương trình kết thúc, hoặc để server ghi lại thông báo và loại bỏ yêu cầu của client.

## **38.12 Tóm Tắt**

Các chương trình có đặc quyền có quyền truy cập vào các tài nguyên hệ thống không có sẵn cho người dùng thông thường. Nếu các chương trình như vậy có thể bị phá hoại, thì tính bảo mật của hệ thống có thể bị xâm phạm. Trong chương này, chúng ta đã trình bày một tập hợp các hướng dẫn để viết các chương trình có đặc quyền. Mục tiêu của những hướng dẫn này là hai mặt: giảm thiểu khả năng một chương trình có đặc quyền bị phá hoại, và giảm thiểu thiệt hại có thể được thực hiện trong trường hợp một chương trình có đặc quyền bị phá hoại.

#### **Thông tin thêm**

[Viega & McGraw, 2002] bao gồm một loạt các chủ đề liên quan đến thiết kế và triển khai phần mềm an toàn. Thông tin chung về bảo mật trên các hệ thống UNIX, cũng như một chương về các kỹ thuật lập trình an toàn có thể được tìm thấy trong [Garfinkel et al., 2003]. Bảo mật máy tính được đề cập ở một mức độ nào đó trong [Bishop, 2005], và ở mức độ lớn hơn nữa bởi cùng tác giả trong [Bishop, 2003]. [Peikari & Chuvakin, 2004] mô tả bảo mật máy tính với trọng tâm là các phương tiện khác nhau mà hệ thống có thể bị tấn công. [Erickson, 2008] và [Anley, 2007] đều cung cấp thảo luận kỹ lưỡng về các khai thác bảo mật khác nhau, cung cấp đủ chi tiết cho các lập trình viên khôn ngoan để tránh những khai thác này. [Chen et al., 2002] là một bài báo mô tả và phân tích mô hình set-user-ID UNIX. [Tsafrir et al., 2008] sửa đổi và cải thiện thảo luận về các điểm khác nhau trong [Chen et al., 2002]. [Drepper, 2009] cung cấp rất nhiều mẹo về lập trình an toàn và phòng thủ trên Linux.

Một số nguồn thông tin về viết chương trình an toàn có sẵn trực tuyến, bao gồm:

- Matt Bishop đã viết một loạt các bài báo liên quan đến bảo mật, có sẵn trực tuyến tại http://nob.cs.ucdavis.edu/~bishop/secprog. Thú vị nhất trong số này là "How to Write a Setuid Program," (ban đầu được xuất bản trong ;login: 12(1) Jan/Feb 1986). Mặc dù có phần lỗi thời, bài báo này chứa rất nhiều mẹo hữu ích.
- Secure Programming for Linux and Unix HOWTO, được viết bởi David Wheeler, có sẵn tại http://www.dwheeler.com/secure-programs/.
- Danh sách kiểm tra hữu ích để viết các chương trình set-user-ID có sẵn trực tuyến tại http://www.homeport.org/~adam/setuid.7.html.

### **38.13 Bài Tập**

- **38-1.** Đăng nhập với tư cách người dùng thông thường không có đặc quyền, tạo một file có thể thực thi (hoặc sao chép một file hiện có như `/bin/sleep`), và bật bit permission set-user-ID trên file đó (`chmod u+s`). Thử sửa đổi file (ví dụ: `cat >> file`). Điều gì xảy ra với các file permissions kết quả (`ls -l`)? Tại sao điều này xảy ra?
- **38-2.** Viết một chương trình set-user-ID-root tương tự như chương trình `sudo(8)`. Chương trình này nên nhận các tùy chọn và đối số dòng lệnh như sau:
  - `$ ./douser [-u user] program-file arg1 arg2 ...`

  Chương trình douser thực thi program-file, với các đối số đã cho, như thể nó được chạy bởi user. (Nếu tùy chọn -u bị bỏ qua, thì user nên mặc định là root.) Trước khi thực thi program-file, douser nên yêu cầu password cho user, xác thực nó với file password chuẩn (xem Listing 8-2, trang 164), và sau đó đặt tất cả các user và group ID của process về các giá trị chính xác cho user đó.
