## Chương 51
# GIỚI THIỆU VỀ POSIX IPC

Các phần mở rộng realtime POSIX.1b đã định nghĩa một tập hợp các cơ chế IPC tương tự như các cơ chế IPC System V được mô tả trong Chương 45 đến 48. (Một trong những mục tiêu của các nhà phát triển POSIX.1b là thiết kế một tập hợp các cơ chế IPC không gặp phải những khiếm khuyết của các cơ sở IPC System V.) Các cơ chế IPC này được gọi chung là POSIX IPC. Ba cơ chế POSIX IPC là:

-  Message queue có thể được sử dụng để truyền message giữa các process. Giống như System V message queue, ranh giới message được bảo toàn, sao cho các reader và writer giao tiếp theo đơn vị message (trái ngược với byte stream không phân định được cung cấp bởi pipe). POSIX message queue cho phép mỗi message được gán một độ ưu tiên, cho phép các message có độ ưu tiên cao được xếp hàng trước các message có độ ưu tiên thấp. Điều này cung cấp một số chức năng giống với chức năng có sẵn thông qua trường `type` của System V message.
-  Semaphore cho phép nhiều process đồng bộ hóa các hành động của chúng. Giống như System V semaphore, một POSIX semaphore là một số nguyên được kernel duy trì mà giá trị của nó không bao giờ được phép giảm xuống dưới 0. POSIX semaphore đơn giản hơn để sử dụng so với System V semaphore: chúng được phân bổ riêng lẻ (trái ngược với System V semaphore set), và chúng được thao tác riêng lẻ bằng hai thao tác tăng và giảm giá trị của semaphore đi một (trái ngược với khả năng của system call `semop()` để cộng hoặc trừ các giá trị tùy ý từ nhiều semaphore trong một System V semaphore set).

 Shared memory cho phép nhiều process chia sẻ cùng một vùng bộ nhớ. Giống như System V shared memory, POSIX shared memory cung cấp IPC nhanh. Sau khi một process đã cập nhật shared memory, thay đổi ngay lập tức hiển thị với các process khác chia sẻ cùng vùng.

Chương này cung cấp tổng quan về các cơ sở POSIX IPC, tập trung vào các tính năng chung của chúng.

## **51.1 Tổng quan API**

Ba cơ chế POSIX IPC có một số tính năng chung. Bảng 51-1 tóm tắt các API của chúng, và chúng ta sẽ đi vào chi tiết về các tính năng chung của chúng trong vài trang tiếp theo.

> Ngoại trừ một đề cập trong Bảng 51-1, trong phần còn lại của chương này, chúng ta sẽ bỏ qua thực tế rằng POSIX semaphore có hai loại: named semaphore và unnamed semaphore. Named semaphore giống như các cơ chế POSIX IPC khác mà chúng ta mô tả trong chương này: chúng được xác định bởi một tên, và có thể truy cập được bởi bất kỳ process nào có quyền phù hợp trên đối tượng. Một unnamed semaphore không có identifier liên kết; thay vào đó, nó được đặt trong một vùng bộ nhớ được chia sẻ bởi một nhóm process hoặc bởi các thread của một process đơn. Chúng ta sẽ đi vào chi tiết về cả hai loại semaphore trong Chương 53.

**Bảng 51-1:** Tóm tắt giao diện lập trình cho các đối tượng POSIX IPC

| Giao diện      | Message queue         | Semaphore                   | Shared memory         |
|----------------|-----------------------|-----------------------------|-----------------------|
| Header file    | `<mqueue.h>`          | `<semaphore.h>`             | `<sys/mman.h>`        |
| Object handle  | `mqd_t`               | `sem_t *`                   | int (file descriptor) |
| Create/open    | `mq_open()`           | `sem_open()`                | `shm_open()` + `mmap()` |
| Close          | `mq_close()`          | `sem_close()`               | `munmap()`            |
| Unlink         | `mq_unlink()`         | `sem_unlink()`              | `shm_unlink()`        |
| Thực hiện IPC  | `mq_send()`,          | `sem_post()`, `sem_wait()`, | thao tác trên các vị trí |
|                | `mq_receive()`        | `sem_getvalue()`            | trong vùng chia sẻ   |
| Thao tác khác  | `mq_setattr()`—thiết lập | `sem_init()`—khởi tạo   | (không)               |
|                | thuộc tính            | unnamed semaphore           |                       |
|                | `mq_getattr()`—lấy    | `sem_destroy()`—hủy        |                       |
|                | thuộc tính            | unnamed semaphore           |                       |
|                | `mq_notify()`—yêu cầu |                             |                       |
|                | thông báo             |                             |                       |

#### **Tên đối tượng IPC**

Để truy cập một đối tượng POSIX IPC, chúng ta phải có một số phương tiện để xác định nó. Phương tiện portable duy nhất mà SUSv3 chỉ định để xác định một đối tượng POSIX IPC là thông qua một tên bao gồm một dấu gạch chéo ban đầu, theo sau là một hoặc nhiều ký tự không phải dấu gạch chéo; ví dụ, `/myobject`. Linux và một số cài đặt khác (ví dụ: Solaris) cho phép loại đặt tên portable này cho các đối tượng IPC.

Trên Linux, tên cho các đối tượng POSIX shared memory và message queue được giới hạn ở `NAME_MAX` (255) ký tự. Đối với semaphore, giới hạn ít hơn 4 ký tự, vì việc triển khai thêm chuỗi `sem.` vào trước tên semaphore.

SUSv3 không cấm các tên có dạng khác với `/myobject`, nhưng nói rằng ngữ nghĩa của các tên như vậy là implementation-defined. Các quy tắc tạo tên đối tượng IPC trên một số hệ thống là khác nhau. Ví dụ, trên Tru64 5.1, tên đối tượng IPC được tạo dưới dạng tên trong file system chuẩn, và tên được diễn giải như một pathname tuyệt đối hoặc tương đối. Nếu caller không có quyền tạo file trong thư mục đó, thì lời gọi IPC open sẽ thất bại. Điều này có nghĩa là các chương trình không có đặc quyền không thể tạo tên có dạng `/myobject` trên Tru64, vì người dùng không có đặc quyền thông thường không thể tạo file trong thư mục gốc (`/`). Một số cài đặt khác có các quy tắc dành riêng cho cài đặt tương tự cho việc xây dựng các tên được cung cấp cho các lời gọi IPC open. Do đó, trong các ứng dụng portable, chúng ta nên cô lập việc tạo tên đối tượng IPC vào một hàm hoặc header file riêng biệt có thể được điều chỉnh phù hợp với cài đặt mục tiêu.

#### **Tạo hoặc mở một đối tượng IPC**

Mỗi cơ chế IPC có một lời gọi open liên kết (`mq_open()`, `sem_open()` hoặc `shm_open()`), tương tự như system call `open()` truyền thống của UNIX được sử dụng cho file. Với tên đối tượng IPC, lời gọi IPC open sẽ:

-  tạo một đối tượng mới với tên đã cho, mở đối tượng đó và trả về một handle cho nó; hoặc
-  mở một đối tượng hiện có và trả về một handle cho đối tượng đó.

Handle được trả về bởi lời gọi IPC open tương tự như file descriptor được trả về bởi system call `open()` truyền thống — nó được sử dụng trong các lời gọi tiếp theo để tham chiếu đến đối tượng.

Kiểu handle được trả về bởi lời gọi IPC open phụ thuộc vào loại đối tượng. Đối với message queue, đó là message queue descriptor, một giá trị kiểu `mqd_t`. Đối với semaphore, đó là một pointer kiểu `sem_t *`. Đối với shared memory, đó là một file descriptor.

Tất cả các lời gọi IPC open cho phép ít nhất ba đối số — `name`, `oflag` và `mode` — như được minh họa bởi lời gọi `shm_open()` sau:

```
fd = shm_open("/mymem", O_CREAT | O_RDWR, S_IRUSR | S_IWUSR);
```

Các đối số này tương tự như các đối số của system call `open()` truyền thống của UNIX. Đối số `name` xác định đối tượng cần tạo hoặc mở. Đối số `oflag` là một bit mask có thể bao gồm ít nhất các flag sau:

`O_CREAT`

Tạo đối tượng nếu nó chưa tồn tại. Nếu flag này không được chỉ định và đối tượng không tồn tại, một lỗi xảy ra (`ENOENT`).

`O_EXCL`

Nếu `O_CREAT` cũng được chỉ định và đối tượng đã tồn tại, một lỗi xảy ra (`EEXIST`). Hai bước — kiểm tra sự tồn tại và tạo — được thực hiện một cách nguyên tử (Mục 5.1). Flag này không có hiệu lực nếu `O_CREAT` không được chỉ định.

Tùy thuộc vào loại đối tượng, `oflag` cũng có thể bao gồm một trong các giá trị `O_RDONLY`, `O_WRONLY` hoặc `O_RDWR`, với ý nghĩa tương tự như `open()`. Các flag bổ sung được phép đối với một số cơ chế IPC.

Đối số còn lại, `mode`, là một bit mask chỉ định các quyền cần đặt trên một đối tượng mới, nếu một đối tượng được tạo bởi lời gọi (nghĩa là `O_CREAT` được chỉ định và đối tượng chưa tồn tại). Các giá trị có thể được chỉ định cho `mode` giống như đối với file (Bảng 15-4, trang 295). Giống như system call `open()`, permissions mask trong `mode` được AND với process umask (Mục 15.4.6). Quyền sở hữu và quyền sở hữu nhóm của một đối tượng IPC mới được lấy từ effective user ID và group ID của process thực hiện lời gọi IPC open. (Nói chính xác hơn, trên Linux, quyền sở hữu của một đối tượng POSIX IPC mới được xác định bởi file-system ID của process, thường có cùng giá trị với các effective ID tương ứng. Xem Mục 9.5.)

> Trên các hệ thống mà các đối tượng IPC xuất hiện trong file system chuẩn, SUSv3 cho phép một cài đặt thiết lập group ID của một đối tượng IPC mới theo group ID của thư mục cha.

#### **Đóng một đối tượng IPC**

Đối với POSIX message queue và semaphore, có một lời gọi IPC close cho biết rằng process đang gọi đã hoàn thành việc sử dụng đối tượng và hệ thống có thể giải phóng mọi tài nguyên được liên kết với đối tượng cho process này. Một đối tượng POSIX shared memory được đóng bằng cách unmap nó với `munmap()`.

Các đối tượng IPC tự động được đóng nếu process kết thúc hoặc thực hiện `exec()`.

#### **Quyền đối tượng IPC**

Các đối tượng IPC có permissions mask giống như đối với file. Quyền truy cập vào đối tượng IPC tương tự như các quyền truy cập file (Mục 15.4.3), ngoại trừ quyền thực thi không có ý nghĩa đối với các đối tượng POSIX IPC.

Kể từ kernel 2.6.19, Linux hỗ trợ việc sử dụng access control list (ACL) để thiết lập các quyền trên các đối tượng POSIX shared memory và named semaphore. Hiện tại, ACL không được hỗ trợ cho POSIX message queue.

### **Xóa đối tượng IPC và tính persistence của đối tượng**

Giống như các file đang mở, các đối tượng POSIX IPC được đếm tham chiếu — kernel duy trì số lượng tham chiếu mở đến đối tượng. So với các đối tượng IPC System V, điều này giúp ứng dụng dễ dàng xác định hơn khi nào đối tượng có thể được xóa an toàn.

Mỗi đối tượng IPC có một lời gọi unlink tương ứng mà hoạt động tương tự như system call `unlink()` truyền thống cho file. Lời gọi unlink ngay lập tức xóa tên của đối tượng, và sau đó hủy đối tượng một khi tất cả các process ngừng sử dụng nó (nghĩa là khi reference count giảm xuống còn 0). Đối với message queue và semaphore, điều này có nghĩa là đối tượng bị hủy sau khi tất cả các process đã đóng đối tượng; đối với shared memory, việc hủy xảy ra sau khi tất cả các process đã unmap đối tượng bằng `munmap()`.

Sau khi một đối tượng bị unlink, các lời gọi IPC open chỉ định cùng tên đối tượng sẽ tham chiếu đến một đối tượng mới (hoặc thất bại, nếu `O_CREAT` không được chỉ định).

Giống như IPC System V, các đối tượng POSIX IPC có kernel persistence. Một khi được tạo, một đối tượng tiếp tục tồn tại cho đến khi nó bị unlink hoặc hệ thống bị tắt. Điều này cho phép một process tạo một đối tượng, sửa đổi trạng thái của nó, và sau đó thoát, để lại đối tượng để được truy cập bởi một số process được khởi động vào thời điểm sau này.

#### **Liệt kê và xóa các đối tượng POSIX IPC qua dòng lệnh**

System V IPC cung cấp hai lệnh, `ipcs` và `ipcrm`, để liệt kê và xóa các đối tượng IPC. Không có lệnh chuẩn nào được cung cấp để thực hiện các nhiệm vụ tương tự cho các đối tượng POSIX IPC. Tuy nhiên, trên nhiều hệ thống, bao gồm Linux, các đối tượng IPC được triển khai trong một file system thực hoặc ảo, được mount ở đâu đó dưới thư mục gốc (`/`), và các lệnh `ls` và `rm` chuẩn có thể được sử dụng để liệt kê và xóa các đối tượng IPC. (SUSv3 không chỉ định việc sử dụng `ls` và `rm` cho các nhiệm vụ này.) Vấn đề chính khi sử dụng các lệnh này là tính không chuẩn của tên đối tượng POSIX IPC và vị trí của chúng trong file system.

Trên Linux, các đối tượng POSIX IPC được chứa trong các file system ảo được mount dưới các thư mục có sticky bit được thiết lập. Bit này là restricted deletion flag (Mục 15.4.5); thiết lập nó có nghĩa là một process không có đặc quyền chỉ có thể unlink các đối tượng POSIX IPC mà nó sở hữu.

#### **Biên dịch các chương trình sử dụng POSIX IPC trên Linux**

Trên Linux, các chương trình sử dụng các cơ chế POSIX IPC phải được link với thư viện realtime, `librt`, bằng cách chỉ định tùy chọn `–lrt` cho lệnh `cc`.

## **51.2 So sánh System V IPC và POSIX IPC**

Khi chúng ta xem xét các cơ chế POSIX IPC trong các chương sau, chúng ta sẽ so sánh từng cơ chế với System V counterpart của nó. Ở đây, chúng ta xem xét một số so sánh chung cho hai loại IPC này.

POSIX IPC có các ưu điểm chung sau khi so sánh với System V IPC:

-  Giao diện POSIX IPC đơn giản hơn giao diện System V IPC.
-  Mô hình POSIX IPC — sử dụng tên thay vì key, và các hàm open, close và unlink — nhất quán hơn với mô hình file UNIX truyền thống.
-  Các đối tượng POSIX IPC được đếm tham chiếu. Điều này đơn giản hóa việc xóa đối tượng, vì chúng ta có thể unlink một đối tượng POSIX IPC, biết rằng nó sẽ chỉ bị hủy khi tất cả các process đã đóng nó.

Tuy nhiên, có một ưu điểm đáng chú ý nghiêng về phía System V IPC: tính portability. POSIX IPC ít portable hơn System V IPC theo những khía cạnh sau:

 System V IPC được chỉ định trong SUSv3 và được hỗ trợ trên gần như mọi cài đặt UNIX. Ngược lại, mỗi cơ chế POSIX IPC là một thành phần tùy chọn trong SUSv3. Một số cài đặt UNIX không hỗ trợ (tất cả) các cơ chế POSIX IPC. Tình trạng này được phản ánh vi mô trên Linux: POSIX shared memory chỉ được hỗ trợ kể từ kernel 2.4; một cài đặt đầy đủ của POSIX semaphore chỉ có sẵn kể từ kernel 2.6; và POSIX message queue chỉ được hỗ trợ kể từ kernel 2.6.6.

-  Mặc dù có đặc tả SUSv3 cho tên đối tượng POSIX IPC, các cài đặt khác nhau tuân theo các quy ước khác nhau để đặt tên đối tượng IPC. Những khác biệt này yêu cầu chúng ta phải làm thêm một chút công việc để viết các ứng dụng portable.
-  Nhiều chi tiết của POSIX IPC không được chỉ định trong SUSv3. Đặc biệt, không có lệnh nào được chỉ định để hiển thị và xóa các đối tượng IPC tồn tại trên một hệ thống. (Trong nhiều cài đặt, các lệnh file-system chuẩn được sử dụng, nhưng các chi tiết của pathname được sử dụng để xác định các đối tượng IPC là khác nhau.)

## **51.3 Tóm tắt**

POSIX IPC là tên chung được đặt cho ba cơ chế IPC — message queue, semaphore và shared memory — được thiết kế bởi POSIX.1b như là các giải pháp thay thế cho các cơ chế IPC System V tương tự.

Giao diện POSIX IPC nhất quán hơn với mô hình file UNIX truyền thống. Các đối tượng IPC được xác định bằng tên, và được quản lý bằng cách sử dụng các lời gọi open, close và unlink hoạt động theo cách tương tự như các system call liên quan đến file tương tự.

POSIX IPC cung cấp một giao diện vượt trội theo nhiều khía cạnh so với giao diện IPC System V. Tuy nhiên, POSIX IPC ít portable hơn System V IPC.
