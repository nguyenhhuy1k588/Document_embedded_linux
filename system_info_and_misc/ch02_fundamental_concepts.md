## Chương 2
# **CÁC KHÁI NIỆM CƠ BẢN**

Chương này giới thiệu nhiều khái niệm liên quan đến lập trình hệ thống Linux. Chương dành cho các độc giả đã làm việc chủ yếu với các hệ điều hành khác, hoặc chỉ có kinh nghiệm hạn chế với Linux hay một phiên bản UNIX khác.

## **2.1 Hệ Điều Hành Lõi: Kernel**

Thuật ngữ hệ điều hành thường được dùng với hai nghĩa khác nhau:

- Chỉ toàn bộ gói bao gồm phần mềm trung tâm quản lý tài nguyên máy tính và tất cả các công cụ phần mềm chuẩn đi kèm, như command-line interpreter, giao diện đồ họa, tiện ích file và trình soạn thảo.
- Theo nghĩa hẹp hơn, chỉ phần mềm trung tâm quản lý và phân bổ tài nguyên máy tính (tức là CPU, RAM và các thiết bị).

Thuật ngữ kernel thường được dùng như từ đồng nghĩa với nghĩa thứ hai, và đây là nghĩa của hệ điều hành mà chúng ta quan tâm trong cuốn sách này.

Mặc dù có thể chạy chương trình trên máy tính mà không cần kernel, sự hiện diện của kernel giúp đơn giản hóa đáng kể việc viết và sử dụng các chương trình khác, đồng thời tăng sức mạnh và tính linh hoạt cho lập trình viên. Kernel làm điều này bằng cách cung cấp một lớp phần mềm để quản lý các tài nguyên hạn chế của máy tính.

Linux kernel executable thường nằm tại đường dẫn `/boot/vmlinuz`, hoặc gì đó tương tự. Tên file này có nguồn gốc lịch sử. Trên các phiên bản UNIX đời đầu, kernel được gọi là `unix`. Các phiên bản UNIX sau này, có cài đặt virtual memory, đổi tên kernel thành `vmunix`. Trên Linux, tên file phản ánh tên hệ thống, với chữ `z` thay thế chữ `x` cuối cùng để biểu thị kernel là một executable được nén.

## **Các tác vụ kernel thực hiện**

Trong số những thứ khác, kernel thực hiện các tác vụ sau:

- **Process scheduling:** Máy tính có một hoặc nhiều central processing unit (CPU), thực thi các lệnh của chương trình. Giống như các hệ thống UNIX khác, Linux là hệ điều hành preemptive multitasking. Multitasking có nghĩa là nhiều process (tức là chương trình đang chạy) có thể đồng thời nằm trong bộ nhớ và mỗi process có thể nhận được quyền sử dụng CPU. Preemptive có nghĩa là các quy tắc quản lý process nào nhận được quyền sử dụng CPU và trong bao lâu được xác định bởi kernel process scheduler (chứ không phải bởi chính các process).

- **Memory management:** Trong khi bộ nhớ máy tính khổng lồ so với tiêu chuẩn một hay hai thập kỷ trước, kích thước phần mềm cũng đã tăng tương ứng, do đó physical memory (RAM) vẫn là tài nguyên hạn chế mà kernel phải chia sẻ giữa các process một cách công bằng và hiệu quả. Giống như hầu hết các hệ điều hành hiện đại, Linux sử dụng virtual memory management (Mục 6.4), một kỹ thuật mang lại hai lợi thế chính:
  - Các process được cô lập với nhau và với kernel, sao cho một process không thể đọc hay sửa đổi bộ nhớ của process khác hoặc của kernel.
  - Chỉ một phần của process cần được giữ trong bộ nhớ, từ đó giảm yêu cầu bộ nhớ của mỗi process và cho phép nhiều process được giữ trong RAM đồng thời.

- **Cung cấp file system:** Kernel cung cấp file system trên disk, cho phép file được tạo, truy xuất, cập nhật, xóa, v.v.

- **Tạo và kết thúc process:** Kernel có thể nạp một chương trình mới vào bộ nhớ, cung cấp cho nó các tài nguyên (ví dụ: CPU, bộ nhớ và quyền truy cập file) cần thiết để chạy. Một instance như vậy của một chương trình đang chạy được gọi là process. Khi process hoàn thành việc thực thi, kernel đảm bảo rằng các tài nguyên nó sử dụng được giải phóng để tái sử dụng sau này.

- **Truy cập thiết bị:** Các thiết bị (chuột, màn hình, bàn phím, ổ đĩa, v.v.) được kết nối với máy tính cho phép giao tiếp thông tin giữa máy tính và thế giới bên ngoài. Kernel cung cấp cho chương trình một interface để chuẩn hóa và đơn giản hóa việc truy cập thiết bị.

- **Networking:** Kernel truyền và nhận các network message (packet) thay mặt cho các user process. Tác vụ này bao gồm định tuyến network packet đến hệ thống đích.

- **Cung cấp system call API:** Các process có thể yêu cầu kernel thực hiện các tác vụ khác nhau bằng cách sử dụng các kernel entry point được gọi là system call. Linux system call API là chủ đề chính của cuốn sách này.

## **Kernel mode và user mode**

Các kiến trúc bộ xử lý hiện đại thường cho phép CPU hoạt động trong ít nhất hai chế độ khác nhau: user mode và kernel mode (đôi khi còn được gọi là supervisor mode). Khi chạy ở user mode, CPU chỉ có thể truy cập bộ nhớ được đánh dấu là nằm trong user space; các nỗ lực truy cập bộ nhớ trong kernel space dẫn đến hardware exception. Khi chạy ở kernel mode, CPU có thể truy cập cả user và kernel memory space.

Một số thao tác nhất định chỉ có thể được thực hiện trong khi bộ xử lý đang hoạt động ở kernel mode. Ví dụ bao gồm thực thi lệnh halt để dừng hệ thống, truy cập hardware quản lý bộ nhớ và khởi tạo các thao tác device I/O.

# **2.2 Shell**

Một shell là chương trình mục đích đặc biệt được thiết kế để đọc các lệnh do người dùng gõ và thực thi các chương trình phù hợp để phản hồi các lệnh đó. Chương trình như vậy đôi khi được gọi là command interpreter.

Thuật ngữ login shell được dùng để chỉ process được tạo ra để chạy shell khi người dùng đăng nhập lần đầu.

Trong khi trên một số hệ điều hành command interpreter là phần không thể tách rời của kernel, trên hệ thống UNIX, shell là một user process. Nhiều shell khác nhau tồn tại. Một số shell quan trọng đã xuất hiện theo thời gian:

- **Bourne shell (sh):** Đây là shell lâu đời nhất trong số các shell được dùng rộng rãi, được viết bởi Steve Bourne. Nó là shell chuẩn cho Seventh Edition UNIX.
- **C shell (csh):** Shell này được viết bởi Bill Joy tại University of California tại Berkeley. Tên xuất phát từ sự giống nhau của nhiều cấu trúc flow-control của shell này với ngôn ngữ lập trình C.
- **Korn shell (ksh):** Shell này được viết như phiên bản kế tiếp của Bourne shell bởi David Korn tại AT&T Bell Laboratories.
- **Bourne again shell (bash):** Shell này là phiên bản cài đặt lại của Bourne shell từ dự án GNU. bash là shell được sử dụng rộng rãi nhất trên Linux.

Các shell được thiết kế không chỉ để sử dụng tương tác, mà còn để thông dịch shell script — các file văn bản chứa các lệnh shell.

## **2.3 Người Dùng và Nhóm**

Mỗi người dùng trên hệ thống được nhận dạng duy nhất, và người dùng có thể thuộc về các nhóm.

#### **Người dùng**

Mỗi người dùng của hệ thống có một tên đăng nhập (username) duy nhất và một user ID (UID) số tương ứng. Đối với mỗi người dùng, chúng được định nghĩa bởi một dòng trong file mật khẩu hệ thống, `/etc/passwd`, bao gồm các thông tin bổ sung sau:

- **Group ID:** numeric group ID của nhóm đầu tiên mà người dùng là thành viên.
- **Home directory:** thư mục ban đầu mà người dùng được đặt vào sau khi đăng nhập.
- **Login shell:** tên chương trình sẽ được thực thi để phiên dịch các lệnh của người dùng.

#### **Nhóm**

Để phục vụ mục đích quản trị — đặc biệt để kiểm soát quyền truy cập file và các tài nguyên hệ thống khác — việc tổ chức người dùng thành nhóm rất hữu ích. Mỗi nhóm được nhận dạng bởi một dòng trong file nhóm hệ thống, `/etc/group`.

#### **Superuser**

Một người dùng, được gọi là superuser, có các đặc quyền đặc biệt trong hệ thống. Tài khoản superuser có user ID là 0, và thường có tên đăng nhập là `root`. Trên các hệ thống UNIX điển hình, superuser bỏ qua tất cả các kiểm tra permission trong hệ thống.

## **2.4 Cây Thư Mục Đơn, Thư Mục, Link và File**

Kernel duy trì một cây thư mục phân cấp đơn để tổ chức tất cả các file trong hệ thống. Ở gốc của cây phân cấp này là *root directory*, có tên `/` (slash). Tất cả file và thư mục đều là con hoặc hậu duệ xa hơn của root directory.

```text
/                    (thư mục)
├── bin              (thư mục)
│   └── bash         (regular file)
├── boot             (thư mục)
│   └── vmlinuz      (regular file)
├── etc              (thư mục)
│   ├── group        (regular file)
│   └── passwd       (regular file)
├── home             (thư mục)
│   ├── avr          (thư mục)
│   │   └── java     (thư mục)
│   │       └── Go.java   (regular file)
│   └── mtk          (thư mục)
│       └── .bashrc  (regular file)
└── usr              (thư mục)
    └── include      (thư mục)
        ├── sys      (thư mục)
        │   └── types.h   (regular file)
        └── stdio.h  (regular file)
```

**Hình 2-1:** Tập hợp con của cây thư mục đơn Linux

#### **Các loại file**

Trong file system, mỗi file được đánh dấu với một *type*, cho biết đó là loại file gì. Một trong các loại file này biểu thị các file dữ liệu thông thường, thường được gọi là *regular* hoặc *plain* file để phân biệt với các loại file khác. Các loại file khác bao gồm device, pipe, socket, thư mục và symbolic link.

#### **Thư mục và link**

Một *directory* là một file đặc biệt có nội dung dạng bảng tên file được ghép với tham chiếu đến các file tương ứng. Sự kết hợp tên file cộng tham chiếu này được gọi là *link*, và file có thể có nhiều link, và do đó nhiều tên, trong cùng một hoặc trong các thư mục khác nhau.

Mỗi thư mục chứa ít nhất hai mục: `.` (dot), là link đến chính thư mục đó, và `..` (dot-dot), là link đến *parent directory*, thư mục phía trên nó trong cây phân cấp.

#### **Symbolic link**

Giống như normal link, symbolic link cung cấp một tên thay thế cho file. Nhưng trong khi normal link là mục tên file cộng pointer trong danh sách thư mục, thì symbolic link là một file được đánh dấu đặc biệt chứa tên của file khác. File sau thường được gọi là target của symbolic link. Khi một pathname được chỉ định trong system call, trong hầu hết các trường hợp, kernel tự động dereference (hay đồng nghĩa là follow) mỗi symbolic link trong pathname, thay thế nó bằng tên file mà nó trỏ đến.

#### **Tên file**

Trên hầu hết các Linux file system, tên file có thể dài tới 255 ký tự. Tên file có thể chứa bất kỳ ký tự nào ngoại trừ dấu gạch chéo (`/`) và ký tự null (`\0`).

### **Pathname**

Một pathname là một chuỗi bao gồm một dấu gạch chéo ban đầu tùy chọn (`/`) theo sau là một chuỗi tên file được phân cách bằng dấu gạch chéo.

Pathname mô tả vị trí của file trong cây thư mục đơn, và là tuyệt đối hay tương đối:

- **Absolute pathname** bắt đầu bằng dấu gạch chéo (`/`) và chỉ định vị trí của file so với root directory.
- **Relative pathname** chỉ định vị trí của file tương đối với current working directory của process, và được phân biệt với absolute pathname bằng sự vắng mặt của dấu gạch chéo ban đầu.

## **Current working directory**

Mỗi process có một current working directory. Đây là "vị trí hiện tại" của process trong cây thư mục đơn, và từ thư mục này, các relative pathname được giải thích cho process. Process kế thừa current working directory từ process cha của nó.

#### **File ownership và permission**

Mỗi file có một user ID và group ID liên kết xác định chủ sở hữu của file và nhóm mà nó thuộc về. Để truy cập file, hệ thống chia người dùng thành ba loại: chủ sở hữu file, người dùng là thành viên của nhóm khớp với group ID của file (group), và phần còn lại của thế giới (other). Ba permission bit có thể được đặt cho mỗi loại người dùng: read, write và execute permission.

# **2.5 Mô Hình File I/O**

Một trong những đặc trưng nổi bật của mô hình I/O trên hệ thống UNIX là khái niệm tính phổ quát của I/O. Điều này có nghĩa là cùng các system call (`open()`, `read()`, `write()`, `close()`, v.v.) được dùng để thực hiện I/O trên tất cả các loại file, bao gồm cả device. Do đó, một chương trình sử dụng các system call này sẽ hoạt động trên bất kỳ loại file nào.

## **File descriptor**

Các I/O system call tham chiếu đến các file đang mở bằng cách sử dụng một file descriptor — một số nguyên không âm (thường nhỏ). File descriptor thường được lấy bằng cách gọi `open()`, nhận một pathname argument chỉ định file cần thực hiện I/O.

Thông thường, một process kế thừa ba open file descriptor khi được khởi động bởi shell: descriptor 0 là standard input; descriptor 1 là standard output; và descriptor 2 là standard error. Trong thư viện stdio, các descriptor này tương ứng với các file stream `stdin`, `stdout` và `stderr`.

## **Thư viện stdio**

Để thực hiện file I/O, các chương trình C thường sử dụng các hàm I/O có trong thư viện C chuẩn. Tập hàm này, được gọi là thư viện stdio, bao gồm `fopen()`, `fclose()`, `scanf()`, `printf()`, `fgets()`, `fputs()`, v.v. Các hàm stdio được xây dựng trên các I/O system call.

# **2.6 Chương Trình**

Chương trình thường tồn tại ở hai dạng. Dạng thứ nhất là source code, văn bản có thể đọc được bao gồm một loạt câu lệnh được viết bằng ngôn ngữ lập trình như C. Để thực thi, source code phải được chuyển đổi sang dạng thứ hai: các lệnh máy nhị phân mà máy tính có thể hiểu.

### **Filter**

Một filter là tên thường được áp dụng cho chương trình đọc input từ stdin, thực hiện một số biến đổi của input đó, và ghi dữ liệu đã biến đổi ra stdout. Ví dụ về filter bao gồm `cat`, `grep`, `tr`, `sort`, `wc`, `sed` và `awk`.

## **Command-line argument**

Trong C, chương trình có thể truy cập các command-line argument, các từ được cung cấp trên command line khi chương trình chạy. Hàm `main()` được khai báo như sau:

```c
int main(int argc, char *argv[])
```

Biến `argc` chứa tổng số command-line argument, và các argument riêng lẻ có sẵn dưới dạng chuỗi được trỏ đến bởi các phần tử của mảng `argv`. Chuỗi đầu tiên, `argv[0]`, xác định tên của chính chương trình.

# **2.7 Process**

Nói đơn giản nhất, một process là một instance của chương trình đang thực thi. Khi một chương trình được thực thi, kernel nạp code của chương trình vào virtual memory, cấp phát không gian cho các biến chương trình, và thiết lập các cấu trúc dữ liệu bookkeeping của kernel để ghi lại các thông tin khác nhau (như process ID, termination status, user ID và group ID) về process.

## **Process memory layout**

Một process được chia thành các phần sau, được gọi là segment:

- **Text:** các lệnh của chương trình.
- **Data:** các biến static được dùng bởi chương trình.
- **Heap:** một vùng từ đó chương trình có thể cấp phát động bộ nhớ thêm.
- **Stack:** một phần bộ nhớ tăng và thu nhỏ khi các hàm được gọi và trả về, được dùng để cấp phát storage cho các biến cục bộ và thông tin liên kết lời gọi hàm.

## **Tạo process và thực thi chương trình**

Một process có thể tạo một process mới bằng cách sử dụng system call `fork()`. Process gọi `fork()` được gọi là parent process, và process mới được gọi là child process. Kernel tạo child process bằng cách sao chép parent process. Child process sau đó tiếp tục thực thi một tập hàm khác trong cùng code như parent, hoặc thường xuyên hơn, sử dụng system call `execve()` để nạp và thực thi một chương trình hoàn toàn mới.

## **Process ID và parent process ID**

Mỗi process có một process identifier (PID) số nguyên duy nhất. Mỗi process cũng có một thuộc tính parent process identifier (PPID), xác định process yêu cầu kernel tạo process này.

## **Kết thúc process và termination status**

Một process có thể kết thúc theo một trong hai cách: bằng cách yêu cầu kết thúc của chính mình bằng system call `_exit()` (hoặc hàm thư viện `exit()` liên quan), hoặc bằng cách bị giết bởi việc nhận signal. Theo quy ước, termination status 0 chỉ ra rằng process đã thành công, và trạng thái khác không chỉ ra rằng đã xảy ra lỗi.

## **Process user và group identifier (credential)**

Mỗi process có một số user ID (UID) và group ID (GID) liên kết, bao gồm:

- **Real user ID và real group ID:** Xác định người dùng và nhóm mà process thuộc về.
- **Effective user ID và effective group ID:** Được sử dụng để xác định permission mà process có khi truy cập các tài nguyên được bảo vệ như file và interprocess communication object.
- **Supplementary group ID:** Xác định các nhóm bổ sung mà process thuộc về.

## **Process đặc quyền**

Theo truyền thống, trên hệ thống UNIX, một process đặc quyền là process có effective user ID là 0 (superuser). Process như vậy bỏ qua các hạn chế permission thường được kernel áp dụng.

## **Capability**

Từ kernel 2.2, Linux chia các đặc quyền truyền thống dành cho superuser thành một tập các đơn vị riêng biệt gọi là capability. Mỗi thao tác đặc quyền được liên kết với một capability cụ thể, và một process chỉ có thể thực hiện một thao tác nếu nó có capability tương ứng. Tên capability bắt đầu bằng tiền tố `CAP_`, như trong `CAP_KILL`.

## **Process init**

Khi khởi động hệ thống, kernel tạo một process đặc biệt gọi là init, "cha của tất cả các process," có nguồn gốc từ file chương trình `/sbin/init`. Tất cả các process trên hệ thống được tạo (bằng `fork()`) bởi init hoặc bởi một trong các hậu duệ của nó. Process init luôn có process ID 1 và chạy với đặc quyền superuser.

#### **Daemon process**

Một daemon là một process mục đích đặc biệt được phân biệt bởi các đặc điểm sau:

- Nó tồn tại lâu dài. Một daemon process thường được khởi động lúc boot hệ thống và tồn tại cho đến khi hệ thống tắt.
- Nó chạy trong nền, và không có controlling terminal để đọc input hay ghi output.

Ví dụ về daemon process bao gồm `syslogd` và `httpd`.

#### **Environment list**

Mỗi process có một environment list, là một tập biến môi trường được duy trì trong user-space memory của process. Mỗi phần tử của danh sách này bao gồm một tên và một giá trị liên kết.

#### **Resource limit**

Mỗi process tiêu thụ tài nguyên, chẳng hạn như open file, bộ nhớ và CPU time. Sử dụng system call `setrlimit()`, một process có thể thiết lập giới hạn trên cho việc tiêu thụ các tài nguyên khác nhau của mình. Mỗi resource limit có hai giá trị liên kết: một soft limit và một hard limit.

## **2.8 Memory Mapping**

System call `mmap()` tạo một memory mapping mới trong virtual address space của process đang gọi.

Mapping thuộc hai loại:

- **File mapping** ánh xạ một vùng của file vào virtual memory của process gọi. Sau khi ánh xạ, nội dung file có thể được truy cập bằng các thao tác trên các byte trong vùng bộ nhớ tương ứng.
- **Anonymous mapping** không có file tương ứng. Thay vào đó, các page của mapping được khởi tạo thành 0.

Memory mapping phục vụ nhiều mục đích khác nhau, bao gồm khởi tạo text segment của process từ executable file, cấp phát bộ nhớ mới, file I/O (memory-mapped I/O) và interprocess communication (thông qua shared mapping).

# **2.9 Static và Shared Library**

Một object library là một file chứa compiled object code cho một tập hàm (thường liên quan đến nhau về mặt logic) có thể được gọi từ các chương trình ứng dụng. Hệ thống UNIX hiện đại cung cấp hai loại object library: static library và shared library.

## **Static library**

Static library (đôi khi còn được gọi là archive) là tập hợp có cấu trúc của các compiled object module. Để sử dụng các hàm từ static library, ta chỉ định library đó trong lệnh link được dùng để build chương trình. Sau khi giải quyết các tham chiếu hàm từ chương trình chính đến các module trong static library, linker trích xuất các bản sao của các object module cần thiết từ library và sao chép chúng vào file executable kết quả.

## **Shared library**

Shared library được thiết kế để giải quyết các vấn đề với static library. Nếu một chương trình được liên kết với shared library, thay vì sao chép object module từ library vào executable, linker chỉ ghi một bản ghi vào executable để chỉ ra rằng lúc runtime executable cần sử dụng shared library đó. Khi executable được nạp vào bộ nhớ lúc runtime, một chương trình gọi là dynamic linker đảm bảo rằng các shared library được tìm thấy và nạp vào bộ nhớ, và thực hiện runtime linking.

## **2.10 Interprocess Communication và Đồng Bộ Hóa**

Một hệ thống Linux đang chạy bao gồm nhiều process, nhiều trong số đó hoạt động độc lập với nhau. Một số process, tuy nhiên, hợp tác để đạt được mục đích dự định, và các process này cần phương pháp để giao tiếp với nhau và đồng bộ hóa các hành động của mình. Linux cung cấp một bộ phong phú các cơ chế interprocess communication (IPC), bao gồm:

- **signal**, được dùng để chỉ ra rằng một sự kiện đã xảy ra;
- **pipe** (quen thuộc với người dùng shell như toán tử `|`) và **FIFO**, có thể được dùng để truyền dữ liệu giữa các process;
- **socket**, có thể được dùng để truyền dữ liệu từ process này sang process khác;
- **file locking**, cho phép process khóa các vùng của file để ngăn các process khác đọc hay cập nhật nội dung file;
- **message queue**, được dùng để trao đổi message giữa các process;
- **semaphore**, được dùng để đồng bộ hóa các hành động của process; và
- **shared memory**, cho phép hai hoặc nhiều process chia sẻ một phần bộ nhớ.

# **2.11 Signal**

Signal thường được mô tả là "software interrupt." Sự đến của một signal thông báo cho process rằng một số sự kiện hoặc điều kiện bất thường đã xảy ra. Có nhiều loại signal, mỗi loại nhận dạng một sự kiện hoặc điều kiện khác nhau. Mỗi loại signal được nhận dạng bằng một số nguyên khác nhau, được định nghĩa với tên tượng trưng có dạng `SIGxxxx`.

Signal được gửi đến process bởi kernel, bởi process khác (với permission phù hợp), hoặc bởi chính process đó.

Khi process nhận được signal, nó thực hiện một trong các hành động sau, tùy thuộc vào signal:

- nó bỏ qua signal;
- nó bị giết bởi signal; hoặc
- nó bị tạm dừng cho đến khi được tiếp tục sau đó bởi việc nhận một signal đặc biệt.

# **2.12 Thread**

Trong các phiên bản UNIX hiện đại, mỗi process có thể có nhiều thread thực thi. Mỗi thread đang thực thi cùng code chương trình và chia sẻ cùng vùng data và heap. Tuy nhiên, mỗi thread có stack riêng chứa các biến cục bộ và thông tin liên kết lời gọi hàm.

Thread có thể giao tiếp với nhau qua các biến toàn cục mà chúng chia sẻ. Threading API cung cấp condition variable và mutex, là các nguyên thủy cho phép các thread của process giao tiếp và đồng bộ hóa các hành động của mình.

## **2.13 Process Group và Shell Job Control**

Mỗi chương trình được thực thi bởi shell được bắt đầu trong một process mới. Trong các shell có job control, tất cả các process trong một pipeline được đặt trong một process group hoặc job mới. Mỗi process trong một process group có cùng một integer process group identifier.

## **2.14 Session, Controlling Terminal và Controlling Process**

Một session là một tập hợp các process group (job). Tất cả các process trong một session có cùng session identifier.

Session thường có một controlling terminal liên kết. Controlling terminal được thiết lập khi process session leader lần đầu tiên mở một terminal device.

# **2.15 Pseudoterminal**

Một pseudoterminal là một cặp virtual device được kết nối, được biết đến là master và slave. Cặp device này cung cấp một kênh IPC cho phép dữ liệu được truyền theo cả hai hướng. Pseudoterminal được dùng trong nhiều ứng dụng, đáng chú ý nhất là trong việc cài đặt các terminal window dưới X Window System và trong các ứng dụng cung cấp dịch vụ network login như telnet và ssh.

## **2.16 Ngày và Giờ**

Hai loại thời gian quan trọng đối với một process:

- **Real time** được đo từ một điểm chuẩn nào đó (calendar time) hoặc từ một điểm cố định trong vòng đời của process (elapsed time hay wall clock time). Trên hệ thống UNIX, calendar time được đo tính bằng giây kể từ nửa đêm ngày 1 tháng 1 năm 1970, UTC. Ngày này được gọi là Epoch.
- **Process time**, còn được gọi là CPU time, là tổng lượng CPU time mà process đã sử dụng kể từ khi bắt đầu. CPU time được chia thành system CPU time và user CPU time.

# **2.17 Kiến Trúc Client-Server**

Ứng dụng client-server là ứng dụng được chia thành hai process thành phần:

- **client**, yêu cầu server thực hiện một dịch vụ nào đó bằng cách gửi cho nó một request message; và
- **server**, kiểm tra request của client, thực hiện các hành động phù hợp, rồi gửi response message trở lại cho client.

## **2.18 Realtime**

Ứng dụng realtime là những ứng dụng cần phản hồi kịp thời với input. Yếu tố xác định là response được đảm bảo được phân phối trong một thời hạn nhất định sau sự kiện kích hoạt.

POSIX.1b định nghĩa một số extension cho POSIX.1 để hỗ trợ ứng dụng realtime, bao gồm asynchronous I/O, shared memory, memory-mapped file, memory locking, realtime clock và timer, các chính sách lập lịch thay thế, realtime signal, message queue và semaphore.

# **2.19 File System /proc**

Giống như một số phiên bản UNIX khác, Linux cung cấp file system `/proc`, bao gồm một tập các thư mục và file được mount dưới thư mục `/proc`.

File system `/proc` là một virtual file system cung cấp interface đến các cấu trúc dữ liệu kernel dưới dạng trông giống như file và thư mục trên file system. Điều này cung cấp một cơ chế dễ dàng để xem và thay đổi các thuộc tính hệ thống khác nhau. Ngoài ra, một tập các thư mục có tên dạng `/proc/PID`, trong đó PID là process ID, cho phép chúng ta xem thông tin về mỗi process đang chạy trên hệ thống.

## **2.20 Tóm Tắt**

Trong chương này, chúng ta đã khảo sát nhiều khái niệm cơ bản liên quan đến lập trình hệ thống Linux. Hiểu biết về các khái niệm này sẽ cung cấp cho độc giả có kinh nghiệm hạn chế trên Linux hoặc UNIX đủ nền tảng để bắt đầu học lập trình hệ thống.
