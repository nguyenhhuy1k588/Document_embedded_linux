## Chương 43
# **TỔNG QUAN VỀ GIAO TIẾP LIÊN TIẾN TRÌNH (IPC)**

Chương này trình bày tổng quan ngắn gọn về các cơ chế mà các process và thread có thể sử dụng để giao tiếp với nhau và đồng bộ hóa các hành động của chúng. Các chương tiếp theo sẽ cung cấp thêm chi tiết về các cơ chế này.

## **43.1 Phân loại các cơ chế IPC**

Hình 43-1 tóm tắt sự phong phú của các cơ chế giao tiếp và đồng bộ hóa trên UNIX, chia chúng thành ba nhóm chức năng lớn:

- **Giao tiếp (Communication):** Các cơ chế này liên quan đến việc trao đổi dữ liệu giữa các process.
- **Đồng bộ hóa (Synchronization):** Các cơ chế này liên quan đến việc đồng bộ hóa các hành động của các process hoặc thread.
- **Signal:** Mặc dù signal chủ yếu được dùng cho các mục đích khác, chúng có thể được dùng như một kỹ thuật đồng bộ hóa trong một số trường hợp nhất định. Hiếm hơn, signal còn có thể được dùng như một kỹ thuật giao tiếp: bản thân số hiệu signal là một dạng thông tin, và realtime signal có thể đi kèm dữ liệu liên quan (một số nguyên hoặc con trỏ). Signal được mô tả chi tiết trong các Chương 20 đến 22.

Mặc dù một số cơ chế này liên quan đến đồng bộ hóa, thuật ngữ chung là giao tiếp liên tiến trình (IPC) thường được dùng để mô tả tất cả chúng.

![](_page_1_Figure_1.jpeg)

<span id="page-1-0"></span>**Hình 43-1:** Phân loại các cơ chế IPC trên UNIX

Như Hình 43-1 minh họa, thường có nhiều cơ chế cung cấp chức năng IPC tương tự nhau. Có một vài lý do cho điều này:

- Các cơ chế tương tự nhau đã phát triển trên các biến thể UNIX khác nhau, và sau đó được chuyển sang các hệ thống UNIX khác. Ví dụ, FIFO được phát triển trên System V, trong khi (stream) socket được phát triển trên BSD.
- Các cơ chế mới đã được phát triển để giải quyết các thiếu sót trong thiết kế của các cơ chế tương tự trước đó. Ví dụ, các cơ chế POSIX IPC (message queue, semaphore và shared memory) được thiết kế như một cải tiến so với các cơ chế System V IPC cũ hơn.

Trong một số trường hợp, các cơ chế được nhóm lại trong Hình 43-1 thực sự cung cấp chức năng khác nhau đáng kể. Ví dụ, stream socket có thể được dùng để giao tiếp qua mạng, trong khi FIFO chỉ có thể dùng để giao tiếp giữa các process trên cùng một máy.

## **43.2 Các cơ chế giao tiếp**

Các cơ chế giao tiếp khác nhau được hiển thị trong Hình 43-1 cho phép các process trao đổi dữ liệu với nhau. (Các cơ chế này cũng có thể được dùng để trao đổi dữ liệu giữa các thread của một process duy nhất, nhưng điều này hiếm khi cần thiết vì các thread có thể trao đổi thông tin thông qua các biến toàn cục chia sẻ.)

Chúng ta có thể chia các cơ chế giao tiếp thành hai loại:

- **Các cơ chế truyền dữ liệu (Data-transfer):** Yếu tố phân biệt chính của các cơ chế này là khái niệm ghi và đọc. Để giao tiếp, một process ghi dữ liệu vào cơ chế IPC, và một process khác đọc dữ liệu đó. Các cơ chế này yêu cầu hai lần truyền dữ liệu giữa bộ nhớ user và bộ nhớ kernel: một lần từ bộ nhớ user sang bộ nhớ kernel khi ghi, và một lần từ bộ nhớ kernel sang bộ nhớ user khi đọc. (Hình 43-2 minh họa tình huống này cho một pipe.)
- **Shared memory:** Shared memory cho phép các process trao đổi thông tin bằng cách đặt nó vào một vùng bộ nhớ được chia sẻ giữa các process. (Kernel thực hiện điều này bằng cách làm cho các mục page-table trong mỗi process trỏ đến cùng các trang RAM, như được minh họa trong Hình 49-2.) Một process có thể làm cho dữ liệu có sẵn cho các process khác bằng cách đặt nó vào vùng shared memory. Vì giao tiếp không yêu cầu system call hay truyền dữ liệu giữa bộ nhớ user và bộ nhớ kernel, shared memory có thể cung cấp tốc độ giao tiếp rất nhanh.

![](_page_2_Figure_6.jpeg)

<span id="page-2-0"></span>**Hình 43-2:** Trao đổi dữ liệu giữa hai process sử dụng pipe

### **Truyền dữ liệu**

Chúng ta có thể chia nhỏ hơn các cơ chế truyền dữ liệu thành các phân loại sau:

- **Byte stream:** Dữ liệu được trao đổi qua pipe, FIFO và datagram socket là một byte stream không phân chia. Mỗi thao tác đọc có thể đọc một số byte tùy ý từ cơ chế IPC, bất kể kích thước của các khối được ghi bởi phía ghi. Mô hình này phản ánh mô hình UNIX truyền thống "file là một chuỗi byte".

- **Message:** Dữ liệu được trao đổi qua System V message queue, POSIX message queue và datagram socket có dạng các message phân chia rõ ràng. Mỗi thao tác đọc đọc toàn bộ một message, như được ghi bởi process ghi. Không thể đọc một phần message, để lại phần còn lại trong cơ chế IPC; cũng không thể đọc nhiều message trong một thao tác đọc duy nhất.
- **Pseudoterminal:** Pseudoterminal là một cơ chế giao tiếp dùng trong các tình huống đặc biệt. Chi tiết được cung cấp trong Chương 64.

Một vài đặc điểm chung phân biệt các cơ chế truyền dữ liệu với shared memory:

- Mặc dù cơ chế truyền dữ liệu có thể có nhiều người đọc, các thao tác đọc là phá hủy dữ liệu. Thao tác đọc tiêu thụ dữ liệu, và dữ liệu đó không còn sẵn có cho bất kỳ process nào khác.

> Flag `MSG_PEEK` có thể được dùng để thực hiện đọc không phá hủy từ socket (Mục 61.3). Socket UDP (Internet domain datagram) cho phép một message được broadcast hoặc multicast đến nhiều người nhận (Mục 61.12).

- Đồng bộ hóa giữa các process đọc và ghi là tự động. Nếu một người đọc cố gắng lấy dữ liệu từ cơ chế truyền dữ liệu hiện tại không có dữ liệu, thì (theo mặc định) thao tác đọc sẽ block cho đến khi một process nào đó ghi dữ liệu vào cơ chế.

## **Shared memory**

Hầu hết các hệ thống UNIX hiện đại cung cấp ba loại shared memory: System V shared memory, POSIX shared memory và memory mapping. Chúng ta xem xét sự khác biệt giữa chúng khi mô tả các cơ chế trong các chương sau (xem đặc biệt Mục 54.5).

Lưu ý các điểm chung sau về shared memory:

- Mặc dù shared memory cung cấp tốc độ giao tiếp nhanh, ưu thế tốc độ này bị bù đắp bởi sự cần thiết phải đồng bộ hóa các thao tác trên shared memory. Ví dụ, một process không nên cố gắng truy cập một cấu trúc dữ liệu trong shared memory trong khi một process khác đang cập nhật nó. Semaphore là phương pháp đồng bộ hóa thường được dùng với shared memory.
- Dữ liệu được đặt vào shared memory hiển thị cho tất cả các process chia sẻ bộ nhớ đó. (Điều này trái ngược với ngữ nghĩa đọc phá hủy được mô tả ở trên cho các cơ chế truyền dữ liệu.)

## **43.3 Các cơ chế đồng bộ hóa**

Các cơ chế đồng bộ hóa được hiển thị trong Hình 43-1 cho phép các process phối hợp các hành động của chúng. Đồng bộ hóa cho phép các process tránh thực hiện các việc như cập nhật đồng thời một vùng shared memory hoặc cùng một phần của một file. Nếu không có đồng bộ hóa, các cập nhật đồng thời như vậy có thể khiến ứng dụng tạo ra kết quả không chính xác.

Hệ thống UNIX cung cấp các cơ chế đồng bộ hóa sau:

- **Semaphore:** Semaphore là một số nguyên được kernel duy trì với giá trị không bao giờ được phép giảm xuống dưới 0. Một process có thể giảm hoặc tăng giá trị của semaphore. Nếu có một nỗ lực giảm giá trị semaphore xuống dưới 0, thì kernel chặn thao tác cho đến khi giá trị semaphore tăng lên đến mức cho phép thao tác được thực hiện. (Thay vào đó, process có thể yêu cầu một thao tác không block; trong trường hợp đó, thay vì block, kernel làm cho thao tác trả về ngay lập tức với lỗi cho thấy thao tác không thể thực hiện ngay lập tức.) Ý nghĩa của semaphore được xác định bởi ứng dụng. Một process giảm semaphore (từ, giả sử, 1 xuống 0) để đặt quyền truy cập độc quyền vào một tài nguyên chia sẻ nào đó, và sau khi hoàn thành công việc trên tài nguyên, tăng semaphore để tài nguyên chia sẻ được giải phóng cho process khác sử dụng. Việc sử dụng binary semaphore—semaphore có giá trị bị giới hạn ở 0 hoặc 1—là phổ biến. Tuy nhiên, một ứng dụng xử lý nhiều phiên bản của tài nguyên chia sẻ sẽ sử dụng semaphore có giá trị tối đa bằng số tài nguyên chia sẻ. Linux cung cấp cả System V semaphore và POSIX semaphore, với chức năng về cơ bản tương tự nhau.
- **File lock:** File lock là phương pháp đồng bộ hóa được thiết kế rõ ràng để phối hợp các hành động của nhiều process hoạt động trên cùng một file. Chúng cũng có thể được dùng để phối hợp truy cập vào các tài nguyên chia sẻ khác. File lock có hai loại: read (shared) lock và write (exclusive) lock. Bất kỳ số lượng process nào cũng có thể giữ read lock trên cùng một file (hoặc một vùng của file). Tuy nhiên, khi một process giữ write lock trên một file (hoặc vùng file), các process khác bị ngăn giữ cả read lẫn write lock trên file đó (hoặc vùng file đó). Linux cung cấp cơ chế file-locking thông qua system call `flock()` và `fcntl()`. System call `flock()` cung cấp cơ chế khóa đơn giản, cho phép các process đặt shared hoặc exclusive lock trên toàn bộ file. Vì chức năng hạn chế của nó, cơ chế khóa `flock()` hiếm khi được dùng ngày nay. System call `fcntl()` cung cấp record locking, cho phép các process đặt nhiều read và write lock trên các vùng khác nhau của cùng một file.
- **Mutex và condition variable:** Các cơ chế đồng bộ hóa này thường được sử dụng với POSIX thread, như được mô tả trong Chương 30.

Một số triển khai UNIX, bao gồm các hệ thống Linux với glibc cung cấp triển khai threading NPTL, cũng cho phép mutex và condition variable được chia sẻ giữa các process. SUSv3 cho phép, nhưng không yêu cầu, một triển khai hỗ trợ process-shared mutex và condition variable. Chúng không có sẵn trên tất cả các hệ thống UNIX, và vì vậy không được sử dụng phổ biến để đồng bộ hóa process.

Khi thực hiện đồng bộ hóa liên process, việc lựa chọn cơ chế thường được xác định bởi các yêu cầu chức năng. Khi phối hợp truy cập vào một file, file record locking thường là lựa chọn tốt nhất. Semaphore thường là lựa chọn tốt hơn để phối hợp truy cập vào các loại tài nguyên chia sẻ khác.

Các cơ chế giao tiếp cũng có thể được dùng để đồng bộ hóa. Ví dụ, trong Mục 44.3, chúng ta thấy cách một pipe có thể được dùng để đồng bộ hóa các hành động của process cha với các tiến trình con của nó. Tổng quát hơn, bất kỳ cơ chế truyền dữ liệu nào cũng có thể được dùng để đồng bộ hóa, với thao tác đồng bộ hóa có dạng trao đổi message qua cơ chế.

> Kể từ kernel 2.6.22, Linux cung cấp thêm cơ chế đồng bộ hóa phi tiêu chuẩn thông qua system call `eventfd()`. System call này tạo ra một đối tượng eventfd có một số nguyên không dấu 8 byte được kernel duy trì. System call trả về một file descriptor tham chiếu đến đối tượng. Ghi một số nguyên vào file descriptor này sẽ cộng số nguyên đó vào giá trị của đối tượng. Một `read()` từ file descriptor sẽ block nếu giá trị của đối tượng là 0. Nếu đối tượng có giá trị khác không, `read()` trả về giá trị đó và đặt lại nó về 0. Ngoài ra, `poll()`, `select()`, hoặc `epoll` có thể được dùng để kiểm tra xem đối tượng có giá trị khác không hay không; nếu có, file descriptor được chỉ ra là có thể đọc. Ứng dụng muốn sử dụng đối tượng eventfd để đồng bộ hóa trước tiên phải tạo đối tượng bằng `eventfd()`, và sau đó gọi `fork()` để tạo các process liên quan kế thừa các file descriptor tham chiếu đến đối tượng. Để biết thêm chi tiết, xem trang man `eventfd(2)`.

# **43.4 So sánh các cơ chế IPC**

Khi nói đến IPC, chúng ta đối mặt với nhiều lựa chọn có thể lúc đầu trông có vẻ bối rối. Trong các chương sau mô tả từng cơ chế IPC, chúng tôi bao gồm các phần so sánh từng cơ chế với các cơ chế tương tự khác. Trong các trang tiếp theo, chúng ta xem xét một số điểm chung có thể xác định việc lựa chọn cơ chế IPC.

## **Nhận dạng đối tượng IPC và handle cho các đối tượng đã mở**

Để truy cập một đối tượng IPC, một process phải có phương tiện để nhận dạng đối tượng đó, và sau khi đối tượng đã được "mở", process phải sử dụng một loại handle nào đó để tham chiếu đến đối tượng mở. Bảng 43-1 tóm tắt các thuộc tính này cho các loại cơ chế IPC khác nhau.

<span id="page-5-0"></span>**Bảng 43-1:** Định danh và handle cho các loại cơ chế IPC khác nhau

| Loại cơ chế                | Tên dùng để nhận dạng đối tượng | Handle dùng để tham chiếu đối tượng trong chương trình |
|----------------------------|---------------------------------|--------------------------------------------------------|
| Pipe                       | không có tên                    | file descriptor                                        |
| FIFO                       | pathname                        | file descriptor                                        |
| UNIX domain socket         | pathname                        | file descriptor                                        |
| Internet domain socket     | địa chỉ IP + số port            | file descriptor                                        |
| System V message queue     | System V IPC key                | System V IPC identifier                                |
| System V semaphore         | System V IPC key                | System V IPC identifier                                |
| System V shared memory     | System V IPC key                | System V IPC identifier                                |
| POSIX message queue        | POSIX IPC pathname              | `mqd_t` (message queue descriptor)                    |
| POSIX named semaphore      | POSIX IPC pathname              | `sem_t *` (semaphore pointer)                          |
| POSIX unnamed semaphore    | không có tên                    | `sem_t *` (semaphore pointer)                          |
| POSIX shared memory        | POSIX IPC pathname              | file descriptor                                        |
| Anonymous mapping          | không có tên                    | không có                                               |
| Memory-mapped file         | pathname                        | file descriptor                                        |
| `flock()` lock             | pathname                        | file descriptor                                        |
| `fcntl()` lock             | pathname                        | file descriptor                                        |

### **Chức năng**

Có sự khác biệt về chức năng giữa các cơ chế IPC khác nhau có thể liên quan đến việc xác định cơ chế nào sẽ sử dụng. Chúng ta bắt đầu bằng cách tóm tắt sự khác biệt giữa các cơ chế truyền dữ liệu và shared memory:

- Các cơ chế truyền dữ liệu bao gồm các thao tác đọc và ghi, với dữ liệu được truyền chỉ có thể được tiêu thụ bởi một process đọc. Kiểm soát luồng giữa người ghi và người đọc, cũng như đồng bộ hóa (để người đọc bị block khi cố gắng đọc dữ liệu từ cơ chế hiện tại trống) được kernel tự động xử lý. Mô hình này phù hợp tốt với nhiều thiết kế ứng dụng.
- Các thiết kế ứng dụng khác phù hợp tự nhiên hơn với mô hình shared memory. Shared memory cho phép một process làm cho dữ liệu hiển thị với bất kỳ số lượng process nào khác chia sẻ cùng vùng bộ nhớ. Các "thao tác" giao tiếp đơn giản—một process có thể truy cập dữ liệu trong shared memory theo cách tương tự như cách nó truy cập bất kỳ bộ nhớ nào khác trong không gian địa chỉ ảo của nó. Mặt khác, nhu cầu xử lý đồng bộ hóa (và có thể cả kiểm soát luồng) có thể tăng thêm độ phức tạp của thiết kế shared memory. Mô hình này phù hợp tốt với các thiết kế ứng dụng cần duy trì trạng thái chia sẻ (ví dụ, cấu trúc dữ liệu chia sẻ).

Đối với các cơ chế truyền dữ liệu khác nhau, các điểm sau đây đáng chú ý:

- Một số cơ chế truyền dữ liệu truyền dữ liệu dưới dạng byte stream (pipe, FIFO và stream socket); các cơ chế khác theo hướng message (message queue và datagram socket). Cách tiếp cận nào được ưu tiên phụ thuộc vào ứng dụng. (Ứng dụng cũng có thể áp đặt mô hình theo hướng message lên cơ chế byte-stream, bằng cách sử dụng ký tự phân cách, message có độ dài cố định, hoặc header message mã hóa độ dài của toàn bộ message; xem Mục 44.8.)
- Một đặc điểm nổi bật của System V và POSIX message queue, so với các cơ chế truyền dữ liệu khác, là khả năng gán một loại số hoặc mức ưu tiên cho một message, để các message có thể được phân phối theo thứ tự khác với thứ tự chúng được gửi.
- Pipe, FIFO và socket được triển khai sử dụng file descriptor. Tất cả các cơ chế IPC này đều hỗ trợ một loạt các mô hình I/O thay thế mà chúng ta mô tả trong Chương 63: I/O multiplexing (system call `select()` và `poll()`), signaldriven I/O, và Linux-specific epoll API. Lợi ích chính của các kỹ thuật này là chúng cho phép ứng dụng đồng thời theo dõi nhiều file descriptor để xem liệu I/O có thể thực hiện trên bất kỳ cái nào trong số chúng. Ngược lại, System V message queue không dùng file descriptor và không hỗ trợ các kỹ thuật này.

Trên Linux, POSIX message queue cũng được triển khai sử dụng file descriptor và hỗ trợ các kỹ thuật I/O thay thế được mô tả ở trên. Tuy nhiên, hành vi này không được chỉ định trong SUSv3, và không được hỗ trợ trên hầu hết các triển khai khác.

- POSIX message queue cung cấp cơ chế thông báo có thể gửi signal đến một process, hoặc khởi tạo một thread mới, khi một message đến trên một queue trước đó trống.

- UNIX domain socket cung cấp tính năng cho phép file descriptor được chuyển từ một process sang process khác. Điều này cho phép một process mở file và làm cho nó có sẵn cho process khác mà không thể truy cập file. Chúng ta mô tả ngắn gọn tính năng này trong Mục 61.13.3.
- Socket UDP (Internet domain datagram) cho phép người gửi broadcast hoặc multicast message đến nhiều người nhận. Chúng ta mô tả ngắn gọn tính năng này trong Mục 61.12.

Đối với các cơ chế đồng bộ hóa process, các điểm sau đây đáng chú ý:

- Record lock được đặt bằng `fcntl()` được coi là thuộc sở hữu của process đặt lock. Kernel sử dụng thuộc tính sở hữu này để phát hiện deadlock (tình huống trong đó hai hoặc nhiều process đang giữ lock mà chặn các yêu cầu lock tiếp theo của nhau). Nếu một tình huống deadlock xảy ra, kernel từ chối yêu cầu lock của một trong các process, trả về lỗi từ lời gọi `fcntl()` để chỉ ra rằng một deadlock đã xảy ra. System V và POSIX semaphore không có thuộc tính sở hữu; không có phát hiện deadlock xảy ra cho semaphore.
- Record lock được đặt bằng `fcntl()` tự động được giải phóng khi process sở hữu lock kết thúc. System V semaphore cung cấp tính năng tương tự dưới dạng tính năng "undo", nhưng tính năng này không đáng tin cậy trong mọi trường hợp (Mục 47.8). POSIX semaphore không cung cấp tương đương của tính năng này.

### **Giao tiếp qua mạng**

Trong tất cả các phương thức IPC được hiển thị trong Hình 43-1, chỉ socket cho phép các process giao tiếp qua mạng. Socket thường được dùng trong một trong hai domain: UNIX domain, cho phép giao tiếp giữa các process trên cùng hệ thống, và Internet domain, cho phép giao tiếp giữa các process trên các máy khác nhau được kết nối qua mạng TCP/IP. Thường thì chỉ cần thay đổi nhỏ để chuyển đổi một chương trình sử dụng UNIX domain socket thành một chương trình sử dụng Internet domain socket, vì vậy một ứng dụng được xây dựng bằng UNIX domain socket có thể được làm cho có khả năng mạng với ít công sức tương đối.

## **Khả năng di động**

Các triển khai UNIX hiện đại hỗ trợ hầu hết các cơ chế IPC được hiển thị trong Hình 43-1. Tuy nhiên, các cơ chế POSIX IPC (message queue, semaphore và shared memory) không phổ biến rộng rãi như các cơ chế System V IPC tương đương, đặc biệt trên các hệ thống UNIX cũ hơn. (Một triển khai POSIX message queue và hỗ trợ đầy đủ cho POSIX semaphore đã xuất hiện trên Linux chỉ trong dòng kernel 2.6.x.) Do đó, từ quan điểm khả năng di động, System V IPC có thể được ưu tiên hơn POSIX IPC.

## **Vấn đề thiết kế trong System V IPC**

Các cơ chế System V IPC được thiết kế độc lập với mô hình I/O UNIX truyền thống, và do đó có một số đặc điểm khiến giao diện lập trình của chúng phức tạp hơn để sử dụng. Các cơ chế POSIX IPC tương ứng được thiết kế để giải quyết các vấn đề này. Các điểm đặc biệt đáng chú ý sau:

- Các cơ chế System V IPC là connectionless. Các cơ chế này không cung cấp khái niệm handle (như file descriptor) tham chiếu đến đối tượng IPC đã mở. Trong các chương sau, đôi khi chúng ta nói về "mở" đối tượng System V IPC, nhưng đây thực sự chỉ là cách diễn đạt tắt để mô tả quá trình lấy handle để tham chiếu đến đối tượng. Kernel không ghi lại rằng process đã "mở" đối tượng (không như các loại đối tượng IPC khác). Điều này có nghĩa là kernel không thể duy trì số lượng tham chiếu của các process hiện đang sử dụng đối tượng. Do đó, có thể cần nỗ lực lập trình bổ sung để ứng dụng có thể biết khi nào đối tượng có thể được xóa an toàn.
- Các giao diện lập trình cho các cơ chế System V IPC không nhất quán với mô hình I/O UNIX truyền thống (chúng sử dụng giá trị key số nguyên và IPC identifier thay vì pathname và file descriptor). Các giao diện lập trình cũng quá phức tạp. Điểm cuối này áp dụng đặc biệt cho System V semaphore (tham khảo Mục 47.11 và 53.5).

Ngược lại, kernel đếm số tham chiếu mở cho đối tượng POSIX IPC. Điều này đơn giản hóa các quyết định về khi nào đối tượng có thể được xóa. Hơn nữa, các cơ chế POSIX IPC cung cấp giao diện đơn giản hơn và nhất quán hơn với mô hình UNIX truyền thống.

### **Khả năng truy cập**

Cột thứ hai của Bảng 43-2 tóm tắt một đặc điểm quan trọng của mỗi loại đối tượng IPC: sơ đồ quyền kiểm soát quá trình nào có thể truy cập đối tượng. Danh sách sau đây bổ sung thêm một số chi tiết về các sơ đồ khác nhau:

- Đối với một số cơ chế IPC (ví dụ, FIFO và socket), tên đối tượng tồn tại trong file system, và khả năng truy cập được xác định theo mặt nạ quyền file liên quan, chỉ định quyền cho owner, group và other (Mục 15.4). Mặc dù các đối tượng System V IPC không nằm trong file system, mỗi đối tượng có mặt nạ quyền liên quan với ngữ nghĩa tương tự như file.
- Một vài cơ chế IPC (pipe, anonymous memory mapping) được đánh dấu là chỉ có thể truy cập bởi các process có liên quan. Ở đây, liên quan nghĩa là liên quan thông qua `fork()`. Để hai process truy cập đối tượng, một trong số chúng phải tạo đối tượng và sau đó gọi `fork()`. Kết quả của `fork()`, process con kế thừa handle tham chiếu đến đối tượng, cho phép cả hai process chia sẻ đối tượng.
- Khả năng truy cập của POSIX unnamed semaphore được xác định bởi khả năng truy cập của vùng shared memory chứa semaphore.
- Để đặt lock trên file, một process phải có file descriptor tham chiếu đến file (tức là, trong thực tế, nó phải có quyền mở file).
- Không có hạn chế nào đối với việc truy cập (tức là kết nối hoặc gửi datagram đến) Internet domain socket. Nếu cần thiết, kiểm soát truy cập phải được triển khai trong ứng dụng.

<span id="page-9-0"></span>**Bảng 43-2:** Khả năng truy cập và tính bền vững cho các loại cơ chế IPC khác nhau

| Loại cơ chế               | Khả năng truy cập                    | Tính bền vững |
|---------------------------|--------------------------------------|---------------|
| Pipe                      | chỉ các process liên quan            | process       |
| FIFO                      | mặt nạ quyền                         | process       |
| UNIX domain socket        | mặt nạ quyền                         | process       |
| Internet domain socket    | bất kỳ process nào                   | process       |
| System V message queue    | mặt nạ quyền                         | kernel        |
| System V semaphore        | mặt nạ quyền                         | kernel        |
| System V shared memory    | mặt nạ quyền                         | kernel        |
| POSIX message queue       | mặt nạ quyền                         | kernel        |
| POSIX named semaphore     | mặt nạ quyền                         | kernel        |
| POSIX unnamed semaphore   | quyền của vùng nhớ bên dưới          | phụ thuộc     |
| POSIX shared memory       | mặt nạ quyền                         | kernel        |
| Anonymous mapping         | chỉ các process liên quan            | process       |
| Memory-mapped file        | mặt nạ quyền                         | file system   |
| `flock()` file lock       | `open()` của file                    | process       |
| `fcntl()` file lock       | `open()` của file                    | process       |

## **Tính bền vững**

Thuật ngữ tính bền vững (persistence) đề cập đến vòng đời của đối tượng IPC. (Tham khảo cột thứ ba của Bảng 43-2.) Chúng ta có thể phân biệt ba loại tính bền vững:

- **Process persistence (Tính bền vững process):** Đối tượng IPC bền vững theo process chỉ tồn tại miễn là nó được ít nhất một process giữ mở. Nếu đối tượng bị đóng bởi tất cả các process, thì tất cả tài nguyên kernel liên quan đến đối tượng được giải phóng, và bất kỳ dữ liệu chưa đọc nào bị hủy. Pipe, FIFO và socket là các ví dụ về cơ chế IPC với tính bền vững process.

> Tính bền vững của dữ liệu FIFO không giống với tính bền vững của tên nó. FIFO có tên trong file system tồn tại ngay cả sau khi tất cả file descriptor tham chiếu đến FIFO đã được đóng.

- **Kernel persistence (Tính bền vững kernel):** Đối tượng IPC bền vững theo kernel tồn tại cho đến khi nó được xóa rõ ràng hoặc hệ thống tắt. Vòng đời của đối tượng không phụ thuộc vào việc có process nào giữ đối tượng mở hay không. Điều này có nghĩa là, ví dụ, một process có thể tạo đối tượng, ghi dữ liệu vào nó, và sau đó đóng nó (hoặc kết thúc). Tại thời điểm sau, một process khác có thể mở đối tượng và đọc dữ liệu. Ví dụ về các cơ chế có tính bền vững kernel là System V IPC và POSIX IPC. Chúng ta khai thác tính chất này trong các chương trình ví dụ mà chúng ta trình bày khi mô tả các cơ chế này trong các chương sau: đối với mỗi cơ chế, chúng ta triển khai các chương trình riêng biệt để tạo đối tượng, xóa đối tượng, và thực hiện giao tiếp hoặc đồng bộ hóa.
- **File-system persistence (Tính bền vững file system):** Đối tượng IPC với tính bền vững file system giữ thông tin ngay cả khi hệ thống được khởi động lại. Đối tượng tồn tại cho đến khi nó được xóa rõ ràng. Loại đối tượng IPC duy nhất thể hiện tính bền vững file system là shared memory dựa trên memory-mapped file.

## **Hiệu suất**

Trong một số trường hợp, các cơ chế IPC khác nhau có thể cho thấy sự khác biệt đáng chú ý về hiệu suất. Tuy nhiên, trong các chương sau, chúng tôi thường không đưa ra so sánh hiệu suất vì các lý do sau:

- Hiệu suất của cơ chế IPC có thể không phải là yếu tố quan trọng trong hiệu suất tổng thể của ứng dụng, và nó có thể không phải là yếu tố duy nhất trong việc xác định lựa chọn cơ chế IPC.
- Hiệu suất tương đối của các cơ chế IPC khác nhau có thể thay đổi giữa các triển khai UNIX hoặc giữa các phiên bản kernel Linux khác nhau.
- Quan trọng nhất, hiệu suất của cơ chế IPC sẽ thay đổi tùy thuộc vào cách thức và môi trường sử dụng chính xác. Các yếu tố liên quan bao gồm kích thước của các đơn vị dữ liệu được trao đổi trong mỗi thao tác IPC, lượng dữ liệu chưa đọc có thể đang chờ trong cơ chế IPC, liệu có cần context switch của process cho mỗi đơn vị dữ liệu được trao đổi hay không, và tải trọng khác trên hệ thống.

Nếu hiệu suất IPC là quan trọng, không có gì thay thế được các benchmark dành riêng cho ứng dụng chạy trong môi trường phù hợp với hệ thống đích. Để đạt được điều này, có thể đáng viết một lớp phần mềm trừu tượng ẩn các chi tiết của cơ chế IPC khỏi ứng dụng và sau đó kiểm tra hiệu suất khi các cơ chế IPC khác nhau được thay thế bên dưới lớp trừu tượng.

## **43.5 Tóm tắt**

Trong chương này, chúng ta đã cung cấp tổng quan về các cơ chế khác nhau mà các process (và thread) có thể sử dụng để giao tiếp với nhau và đồng bộ hóa các hành động của chúng.

Trong số các cơ chế giao tiếp được cung cấp trên Linux có pipe, FIFO, socket, message queue và shared memory. Các cơ chế đồng bộ hóa được cung cấp trên Linux bao gồm semaphore và file lock.

Trong nhiều trường hợp, chúng ta có lựa chọn một số kỹ thuật có thể cho giao tiếp và đồng bộ hóa khi thực hiện một nhiệm vụ nhất định. Trong quá trình của chương này, chúng ta đã so sánh các kỹ thuật khác nhau theo nhiều cách khác nhau, với mục tiêu làm nổi bật một số sự khác biệt có thể ảnh hưởng đến việc chọn một kỹ thuật này hơn một kỹ thuật khác.

Trong các chương tiếp theo, chúng ta đi sâu vào từng cơ chế giao tiếp và đồng bộ hóa với nhiều chi tiết hơn.

## **43.6 Bài tập**

**43-1.** Viết một chương trình đo băng thông được cung cấp bởi pipe. Là các đối số dòng lệnh, chương trình nên chấp nhận số lượng khối dữ liệu được gửi và kích thước của mỗi khối dữ liệu. Sau khi tạo một pipe, chương trình chia thành hai process: một process con ghi các khối dữ liệu vào pipe nhanh nhất có thể, và một process cha đọc các khối dữ liệu. Sau khi tất cả dữ liệu được đọc, process cha nên in thời gian trôi qua và băng thông (byte được truyền mỗi giây). Đo băng thông cho các kích thước khối dữ liệu khác nhau.

**43-2.** Lặp lại bài tập trước cho System V message queue, POSIX message queue, UNIX domain stream socket và UNIX domain datagram socket. Sử dụng các chương trình này để so sánh hiệu suất tương đối của các cơ chế IPC khác nhau trên Linux. Nếu bạn có quyền truy cập vào các triển khai UNIX khác, hãy thực hiện các so sánh tương tự trên các hệ thống đó.
