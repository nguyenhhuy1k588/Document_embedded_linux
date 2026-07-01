## Chương 58
# SOCKETS: KIẾN THỨC CƠ BẢN VỀ MẠNG TCP/IP

Chương này cung cấp phần giới thiệu về các khái niệm mạng máy tính và các giao thức mạng TCP/IP. Hiểu biết về các chủ đề này là cần thiết để sử dụng hiệu quả Internet domain socket, được mô tả trong chương tiếp theo.

Bắt đầu từ chương này, chúng ta bắt đầu đề cập đến các tài liệu Request for Comments (RFC). Mỗi giao thức mạng được thảo luận trong cuốn sách này được định nghĩa chính thức trong một tài liệu RFC. Chúng ta cung cấp thêm thông tin về RFC, cũng như danh sách các RFC đặc biệt liên quan đến tài liệu trong cuốn sách này, trong Mục 58.7.

# **58.1 Internet**

Một liên mạng hay phổ biến hơn là internet (viết thường), kết nối các mạng máy tính khác nhau, cho phép các host trên tất cả các mạng giao tiếp với nhau. Nói cách khác, một internet là một mạng của các mạng máy tính. Thuật ngữ subnetwork, hay subnet, được dùng để chỉ một trong các mạng tạo thành một internet. Một internet nhằm mục đích ẩn các chi tiết của các mạng vật lý khác nhau để trình bày một kiến trúc mạng thống nhất cho tất cả các host trên các mạng được kết nối. Điều này có nghĩa là, ví dụ, một định dạng địa chỉ duy nhất được dùng để nhận dạng tất cả các host trong internet.

Mặc dù nhiều giao thức internetworking đã được thiết kế, TCP/IP đã trở thành bộ giao thức chiếm ưu thế, thay thế ngay cả các giao thức mạng độc quyền trước đây phổ biến trên mạng cục bộ và diện rộng. Thuật ngữ Internet (viết hoa) được dùng để chỉ internet TCP/IP kết nối hàng triệu máy tính trên toàn cầu.

Implementation TCP/IP đầu tiên được phổ biến rộng rãi xuất hiện cùng với 4.2BSD năm 1983. Một số implementation của TCP/IP được bắt nguồn trực tiếp từ code BSD; các implementation khác, bao gồm implementation Linux, được viết từ đầu, lấy hoạt động của code BSD làm tiêu chuẩn tham chiếu định nghĩa hoạt động của TCP/IP.

> TCP/IP phát triển từ một dự án được tài trợ bởi Cơ quan Dự án Nghiên cứu Tiên tiến của Bộ Quốc phòng Hoa Kỳ (ARPA, sau là DARPA, với D là Defense) để thiết kế một kiến trúc mạng máy tính được sử dụng trong ARPANET, một mạng diện rộng ban đầu. Trong những năm 1970, một họ giao thức mới được thiết kế cho ARPANET. Chính xác hơn, các giao thức này được gọi là bộ giao thức Internet DARPA, nhưng thường được biết đến hơn là bộ giao thức TCP/IP, hoặc đơn giản là TCP/IP.

Hình 58-1 cho thấy một internet đơn giản. Trong sơ đồ này, máy tekapo là ví dụ về một router, một máy tính có chức năng kết nối một subnet với một subnet khác, truyền dữ liệu giữa chúng. Ngoài việc hiểu giao thức internet đang được sử dụng, một router cũng phải hiểu các giao thức data-link-layer (có thể) khác nhau được sử dụng trên mỗi subnet mà nó kết nối.

Một router có nhiều network interface, một cho mỗi subnet mà nó được kết nối. Thuật ngữ chung hơn là multihomed host được dùng cho bất kỳ host nào—không nhất thiết phải là router—với nhiều network interface. (Cách khác để mô tả router là nói rằng nó là một multihomed host chuyển tiếp packet từ subnet này sang subnet khác.) Một multihomed host có địa chỉ mạng khác nhau cho mỗi interface của nó (tức là, địa chỉ khác nhau trên mỗi subnet mà nó được kết nối).

**Hình 58-1:** Một internet sử dụng router để kết nối hai mạng

# **58.2 Các Giao Thức Mạng và Các Lớp**

Giao thức mạng là một tập hợp các quy tắc định nghĩa cách thông tin được truyền qua mạng. Các giao thức mạng thường được tổ chức thành một chuỗi các lớp, với mỗi lớp xây dựng trên lớp bên dưới để thêm các tính năng được cung cấp cho các lớp cao hơn.

*Bộ giao thức TCP/IP* là một giao thức mạng phân lớp (Hình 58-2). Nó bao gồm *Internet Protocol* (IP) và nhiều giao thức được xếp lớp trên nó. (Code implement các lớp khác nhau này thường được gọi là *protocol stack*.) Tên TCP/IP bắt nguồn từ thực tế là *Transmission Control Protocol* (TCP) là giao thức tầng transport được sử dụng nhiều nhất.

Chúng ta đã bỏ qua một loạt các giao thức TCP/IP khác từ Hình 58-2 vì chúng không liên quan đến chương này. *Address Resolution Protocol* (ARP) liên quan đến việc ánh xạ địa chỉ Internet sang địa chỉ phần cứng (ví dụ: Ethernet). *Internet Control Message Protocol* (ICMP) được dùng để truyền thông tin lỗi và điều khiển qua mạng. (ICMP được dùng bởi chương trình *ping*, thường được dùng để kiểm tra xem một host cụ thể có sống và hiển thị trên mạng TCP/IP không, và bởi *traceroute*, theo dõi đường đi của một IP packet qua mạng.) *Internet Group Management Protocol* (IGMP) được dùng bởi các host và router hỗ trợ multicasting của IP datagram.

**Hình 58-2:** Các giao thức trong bộ TCP/IP

Một trong những khái niệm cung cấp sức mạnh và tính linh hoạt lớn cho phân lớp giao thức là *tính trong suốt*—mỗi lớp giao thức che giấu các lớp cao hơn khỏi hoạt động và sự phức tạp của các lớp thấp hơn. Do đó, ví dụ, một ứng dụng sử dụng TCP chỉ cần sử dụng sockets API chuẩn và biết rằng nó đang sử dụng dịch vụ truyền tải byte-stream đáng tin cậy. Nó không cần hiểu các chi tiết hoạt động của TCP. (Khi chúng ta xem xét các tùy chọn socket trong Mục 61.9, chúng ta sẽ thấy rằng điều này không phải lúc nào cũng hoàn toàn đúng; đôi khi, một ứng dụng thực sự cần biết một số chi tiết về hoạt động của giao thức transport bên dưới.) Ứng dụng cũng không cần biết các chi tiết hoạt động của IP hay data-link layer. Từ góc độ của các ứng dụng, có vẻ như chúng đang giao tiếp trực tiếp với nhau qua sockets API, như được hiển thị trong Hình 58-3, nơi các đường ngang đứt nét đại diện cho các đường giao tiếp ảo giữa các thực thể ứng dụng, TCP và IP tương ứng trên hai host.

### **Đóng Gói (Encapsulation)**

Encapsulation là một nguyên tắc quan trọng của giao thức mạng phân lớp. Hình 58-4 cho thấy một ví dụ về encapsulation trong các lớp giao thức TCP/IP. Ý tưởng chính của encapsulation là thông tin (ví dụ: dữ liệu ứng dụng, một TCP segment, hoặc một IP datagram) được truyền từ lớp cao hơn xuống lớp thấp hơn được lớp thấp hơn coi là dữ liệu không trong suốt. Nói cách khác, lớp thấp hơn không cố gắng diễn giải thông tin được gửi từ lớp trên, mà chỉ đặt thông tin đó vào bất kỳ loại packet nào được sử dụng trong lớp thấp hơn và thêm header đặc thù cho lớp của nó trước khi truyền packet xuống lớp thấp hơn tiếp theo. Khi dữ liệu được truyền từ lớp thấp hơn lên lớp cao hơn, một quá trình giải nén ngược lại diễn ra.

> Chúng ta không hiển thị trong Hình 58-4, nhưng khái niệm encapsulation cũng mở rộng xuống đến data-link layer, nơi IP datagram được đóng gói bên trong các network frame. Encapsulation cũng có thể mở rộng lên đến application layer, nơi ứng dụng có thể thực hiện đóng gói dữ liệu của riêng mình.

# **58.3 Data-Link Layer**

Lớp thấp nhất trong Hình 58-2 là data-link layer, bao gồm device driver và phần cứng giao diện (network card) với phương tiện vật lý bên dưới (ví dụ: đường điện thoại, cáp đồng trục, hoặc cáp quang). Data-link layer có liên quan đến việc truyền dữ liệu qua liên kết vật lý trong một mạng.

Để truyền dữ liệu, data-link layer đóng gói datagram từ lớp mạng vào các đơn vị gọi là frame. Ngoài dữ liệu cần truyền, mỗi frame bao gồm một header chứa, ví dụ, địa chỉ đích và kích thước frame. Data-link layer truyền các frame qua liên kết vật lý và xử lý các xác nhận (acknowledgement) từ receiver. (Không phải tất cả data-link layer đều dùng acknowledgement.) Lớp này có thể thực hiện phát hiện lỗi, truyền lại và kiểm soát luồng. Một số data-link layer cũng chia các network packet lớn thành nhiều frame và tổng hợp lại chúng ở receiver.

Từ quan điểm lập trình ứng dụng, chúng ta thường có thể bỏ qua data-link layer, vì tất cả các chi tiết giao tiếp được xử lý trong driver và phần cứng.

Một đặc điểm của data-link layer quan trọng cho việc thảo luận về IP của chúng ta là đơn vị truyền tải tối đa (MTU). MTU của data-link layer là giới hạn trên mà lớp đó đặt ra về kích thước của frame. Các data-link layer khác nhau có MTU khác nhau.

> Lệnh `netstat –i` hiển thị danh sách các network interface của hệ thống, cùng với MTU của chúng.

**Hình 58-3:** Giao tiếp phân lớp qua các giao thức TCP/IP

**Hình 58-4:** Encapsulation trong các lớp giao thức TCP/IP

# **58.4 Network Layer: IP**

Phía trên data-link layer là network layer, liên quan đến việc giao packet (dữ liệu) từ host nguồn đến host đích. Lớp này thực hiện nhiều nhiệm vụ, bao gồm:

- chia dữ liệu thành các fragment đủ nhỏ để truyền qua data-link layer (nếu cần thiết);
- định tuyến dữ liệu qua internet; và
- cung cấp dịch vụ cho tầng transport.

Trong bộ giao thức TCP/IP, giao thức chính trong network layer là IP. Phiên bản IP xuất hiện trong implementation 4.2BSD là IP version 4 (IPv4). Vào đầu những năm 1990, một phiên bản IP sửa đổi đã được thiết kế: IP version 6 (IPv6). Sự khác biệt đáng chú ý nhất giữa hai phiên bản là IPv4 nhận dạng subnet và host bằng địa chỉ 32-bit, trong khi IPv6 dùng địa chỉ 128-bit, do đó cung cấp phạm vi địa chỉ lớn hơn nhiều để gán cho host. Mặc dù IPv4 vẫn là phiên bản IP chiếm ưu thế được sử dụng trên Internet, trong những năm tới, nó sẽ được thay thế bởi IPv6. Cả IPv4 và IPv6 đều hỗ trợ các giao thức tầng transport UDP và TCP cao hơn (cũng như nhiều giao thức khác).

> Mặc dù không gian địa chỉ 32-bit về mặt lý thuyết cho phép hàng tỷ địa chỉ mạng IPv4 được gán, cách thức cấu trúc và phân bổ địa chỉ có nghĩa là số lượng địa chỉ khả dụng thực tế thấp hơn nhiều. Khả năng cạn kiệt không gian địa chỉ IPv4 là một trong những động lực chính cho việc tạo ra IPv6.

Hình 58-2 cho thấy một kiểu socket raw (`SOCK_RAW`), cho phép ứng dụng giao tiếp trực tiếp với lớp IP. Chúng ta không mô tả việc sử dụng raw socket, vì hầu hết các ứng dụng đều dùng socket trên một trong các giao thức tầng transport (TCP hoặc UDP). Raw socket được mô tả trong Chương 28 của [Stevens et al., 2004].

#### **IP truyền datagram**

IP truyền dữ liệu dưới dạng datagram (packet). Mỗi datagram được gửi giữa hai host đi qua mạng một cách độc lập, có thể đi theo các tuyến đường khác nhau. Một IP datagram bao gồm một header, có kích thước từ 20 đến 60 byte. Header chứa địa chỉ của host đích, để datagram có thể được định tuyến qua mạng đến đích của nó, và cũng bao gồm địa chỉ nguồn của packet, để host nhận biết nguồn gốc của datagram.

> Một host gửi có thể giả mạo địa chỉ nguồn của packet, và đây là cơ sở của một cuộc tấn công TCP từ chối dịch vụ được gọi là SYN-flooding.

Một implementation IP có thể đặt giới hạn trên cho kích thước datagram mà nó hỗ trợ. Tất cả các implementation IP phải cho phép datagram ít nhất lớn bằng giới hạn được chỉ định bởi kích thước buffer tổng hợp tối thiểu của IP. Trong IPv4, giới hạn này là 576 byte; trong IPv6, là 1500 byte.

## **IP là connectionless và không đáng tin cậy**

IP được mô tả là một giao thức connectionless, vì nó không cung cấp khái niệm về virtual circuit kết nối hai host. IP cũng là một giao thức không đáng tin cậy: nó thực hiện nỗ lực "tốt nhất" để truyền datagram từ sender đến receiver, nhưng không đảm bảo rằng các packet sẽ đến theo thứ tự chúng được truyền, rằng chúng sẽ không bị trùng lặp, hoặc thậm chí rằng chúng sẽ đến. IP cũng không cung cấp khôi phục lỗi (các packet có lỗi header bị loại bỏ im lặng). Độ tin cậy phải được cung cấp bằng cách dùng giao thức tầng transport đáng tin cậy (ví dụ: TCP) hoặc trong chính ứng dụng.

> IPv4 cung cấp checksum cho IP header, cho phép phát hiện lỗi trong header, nhưng không cung cấp bất kỳ phát hiện lỗi nào cho dữ liệu được truyền bên trong packet. IPv6 không cung cấp checksum trong IP header, dựa vào các giao thức cao hơn để cung cấp kiểm tra lỗi và độ tin cậy khi cần thiết. (UDP checksum là tùy chọn với IPv4, nhưng thường được kích hoạt; UDP checksum là bắt buộc với IPv6. TCP checksum là bắt buộc với cả IPv4 và IPv6.)

#### **IP có thể phân mảnh datagram**

Các IPv4 datagram có thể lên đến 65.535 byte. Theo mặc định, IPv6 cho phép datagram tối đa 65.575 byte (40 byte cho header, 65.535 byte cho dữ liệu), và cung cấp một tùy chọn cho datagram lớn hơn (gọi là jumbogram).

Chúng ta đã lưu ý trước đó rằng hầu hết data-link layer áp đặt giới hạn trên (MTU) cho kích thước của data frame. Ví dụ, giới hạn trên này là 1500 byte trên kiến trúc mạng Ethernet thường được dùng (tức là, nhỏ hơn nhiều so với kích thước tối đa của IP datagram). IP cũng định nghĩa khái niệm path MTU. Đây là MTU tối thiểu trên tất cả data-link layer được đi qua trên tuyến đường từ nguồn đến đích. (Trên thực tế, Ethernet MTU thường là MTU tối thiểu trong một path.)

Khi một IP datagram lớn hơn MTU, IP phân mảnh (chia nhỏ) datagram thành các đơn vị có kích thước phù hợp để truyền qua mạng. Các fragment này sau đó được tổng hợp lại ở đích cuối cùng để tạo lại datagram gốc. (Mỗi IP fragment là một IP datagram chứa trường offset cho biết vị trí của fragment đó trong datagram gốc.)

IP fragmentation xảy ra trong suốt đối với các lớp giao thức cao hơn, nhưng nhìn chung được coi là không mong muốn. Vấn đề là vì IP không thực hiện truyền lại, và datagram chỉ có thể được tổng hợp lại ở đích nếu tất cả các fragment đến, toàn bộ datagram sẽ không thể dùng được nếu bất kỳ fragment nào bị mất hoặc chứa lỗi truyền. Trong một số trường hợp, điều này có thể dẫn đến tỷ lệ mất dữ liệu đáng kể (đối với các lớp giao thức cao hơn không thực hiện truyền lại, chẳng hạn như UDP) hoặc tốc độ truyền bị giảm (đối với các lớp giao thức cao hơn thực hiện truyền lại, chẳng hạn như TCP). Các implementation TCP hiện đại sử dụng các thuật toán (path MTU discovery) để xác định MTU của một path giữa các host, và tương ứng chia nhỏ dữ liệu chúng truyền đến IP, để IP không được yêu cầu truyền datagram vượt quá kích thước này. UDP không cung cấp cơ chế như vậy, và chúng ta xem xét cách các ứng dụng dựa trên UDP có thể đối phó với khả năng IP fragmentation trong Mục 58.6.2.

# **58.5 Địa Chỉ IP**

Một địa chỉ IP bao gồm hai phần: network ID, xác định mạng mà host đang cư trú, và host ID, nhận dạng host trong mạng đó.

## **Địa chỉ IPv4**

Địa chỉ IPv4 bao gồm 32 bit (Hình 58-5). Khi được biểu diễn dưới dạng người đọc được, các địa chỉ này thường được viết theo ký hiệu dotted-decimal, với 4 byte của địa chỉ được viết dưới dạng số thập phân cách nhau bởi dấu chấm, ví dụ như 204.152.189.116.

|                 | 32 bits    |         |
|-----------------|------------|---------|
| Network address | Network ID | Host ID |
|                 |            |         |
| Network mask    | all 1s     | all 0s  |

**Hình 58-5:** Địa chỉ mạng IPv4 và network mask tương ứng

Khi một tổ chức đăng ký một dải địa chỉ IPv4 cho các host của mình, nó nhận được một địa chỉ mạng 32-bit và một network mask 32-bit tương ứng. Ở dạng nhị phân, mask này bao gồm một chuỗi các bit 1 ở các bit ngoài cùng bên trái, theo sau là một chuỗi các bit 0 để điền phần còn lại của mask. Các bit 1 cho biết phần nào của địa chỉ chứa network ID được gán, trong khi các bit 0 cho biết phần nào của địa chỉ có sẵn cho tổ chức để gán làm host ID duy nhất trên mạng của nó. Kích thước của phần network ID của mask được xác định khi địa chỉ được gán. Vì thành phần network ID luôn chiếm phần ngoài cùng bên trái của mask, ký hiệu sau đây đủ để chỉ định dải địa chỉ được gán:

204.152.189.0/24

`/24` cho biết phần network ID của địa chỉ được gán bao gồm 24 bit ngoài cùng bên trái, với 8 bit còn lại xác định host ID. Hoặc, chúng ta có thể nói rằng network mask trong trường hợp này là 255.255.255.0 theo ký hiệu dotted-decimal.

Một tổ chức nắm giữ địa chỉ này có thể gán 254 địa chỉ Internet duy nhất cho máy tính của mình—204.152.189.1 đến 204.152.189.254. Hai địa chỉ không thể được gán. Một trong số đó là địa chỉ có host ID là tất cả các bit 0, được dùng để nhận dạng chính mạng. Địa chỉ kia là địa chỉ có host ID là tất cả các bit 1—204.152.189.255 trong ví dụ này—là địa chỉ broadcast của subnet.

Một số địa chỉ IPv4 có ý nghĩa đặc biệt. Địa chỉ đặc biệt 127.0.0.1 thường được định nghĩa là địa chỉ loopback, và theo quy ước được gán tên host localhost. (Bất kỳ địa chỉ nào trên mạng 127.0.0.0/8 đều có thể được chỉ định là địa chỉ loopback IPv4, nhưng 127.0.0.1 là lựa chọn thông thường.) Một datagram được gửi đến địa chỉ này không bao giờ thực sự đến mạng, mà thay vào đó tự động quay vòng để trở thành đầu vào cho host gửi. Việc sử dụng địa chỉ này thuận tiện để kiểm tra chương trình client và server trên cùng một host. Để sử dụng trong chương trình C, hằng số nguyên `INADDR_LOOPBACK` được định nghĩa cho địa chỉ này.

Hằng số `INADDR_ANY` là cái gọi là địa chỉ wildcard IPv4. Địa chỉ wildcard IP hữu ích cho các ứng dụng bind Internet domain socket trên các multihomed host. Nếu một ứng dụng trên multihomed host bind một socket chỉ vào một trong các địa chỉ IP của host, thì socket đó chỉ có thể nhận UDP datagram hoặc TCP connection request được gửi đến địa chỉ IP đó. Tuy nhiên, chúng ta thường muốn một ứng dụng trên multihomed host có thể nhận datagram hoặc connection request chỉ định bất kỳ địa chỉ IP nào của host, và việc bind socket vào địa chỉ wildcard IP làm điều này có thể. SUSv3 không chỉ định giá trị cụ thể nào cho `INADDR_ANY`, nhưng hầu hết các implementation định nghĩa nó là 0.0.0.0 (tất cả zero).

Thông thường, địa chỉ IPv4 được chia mạng con (subnet). Subnetting chia phần host ID của địa chỉ IPv4 thành hai phần: subnet ID và host ID (Hình 58-6). (Sự lựa chọn cách chia các bit của host ID được thực hiện bởi quản trị viên mạng cục bộ.) Lý do cho subnetting là một tổ chức thường không gắn tất cả các host của mình vào một mạng duy nhất. Thay vào đó, tổ chức có thể vận hành một tập hợp các subnetwork ("internal internetwork"), với mỗi subnetwork được nhận dạng bằng sự kết hợp của network ID cộng với subnet ID. Sự kết hợp này thường được gọi là extended network ID. Trong một subnet, subnet mask phục vụ vai trò tương tự như đã mô tả trước đó cho network mask, và chúng ta có thể dùng ký hiệu tương tự để chỉ định dải địa chỉ được gán cho một subnet cụ thể.

**Hình 58-6:** IPv4 subnetting

#### **Địa chỉ IPv6**

Các nguyên tắc của địa chỉ IPv6 tương tự như địa chỉ IPv4. Sự khác biệt chính là địa chỉ IPv6 bao gồm 128 bit, và một vài bit đầu tiên của địa chỉ là format prefix, cho biết kiểu địa chỉ.

Địa chỉ IPv6 thường được viết như một chuỗi các số hexadecimal 16-bit cách nhau bởi dấu hai chấm, ví dụ:

```
F000:0:0:0:0:0:A:1
```

Địa chỉ IPv6 thường bao gồm một chuỗi các số không và, theo quy ước ký hiệu, hai dấu hai chấm (::) có thể được dùng để chỉ ra chuỗi đó. Do đó, địa chỉ trên có thể được viết lại là:

```
F000::A:1
```

Chỉ một instance của ký hiệu double-colon có thể xuất hiện trong một địa chỉ IPv6; nhiều hơn một instance sẽ mơ hồ.

IPv6 cũng cung cấp các tương đương của địa chỉ loopback của IPv4 (127 số không, tiếp theo là một số một, tức là ::1) và địa chỉ wildcard (tất cả không, được viết là 0::0 hoặc ::).

Để cho phép các ứng dụng IPv6 giao tiếp với các host chỉ hỗ trợ IPv4, IPv6 cung cấp cái gọi là địa chỉ IPv4-mapped IPv6. Định dạng của các địa chỉ này được hiển thị trong Hình 58-7.

**Hình 58-7:** Định dạng của địa chỉ IPv4-mapped IPv6

Khi viết địa chỉ IPv4-mapped IPv6, phần IPv4 của địa chỉ (tức là 4 byte cuối cùng) được viết theo ký hiệu dotted-decimal IPv4. Do đó, địa chỉ IPv4-mapped IPv6 tương đương với 204.152.189.116 là ::FFFF:204.152.189.116.

# **58.6 Tầng Transport**

Có hai giao thức tầng transport được sử dụng rộng rãi trong bộ TCP/IP:

- User Datagram Protocol (UDP) là giao thức được dùng cho datagram socket.
- Transmission Control Protocol (TCP) là giao thức được dùng cho stream socket.

Trước khi xem xét các giao thức này, chúng ta trước tiên cần mô tả port number, một khái niệm được cả hai giao thức sử dụng.

## **58.6.1 Port Number**

Nhiệm vụ của giao thức transport là cung cấp dịch vụ giao tiếp end-to-end cho các ứng dụng cư trú trên các host khác nhau (hoặc đôi khi trên cùng một host). Để làm được điều này, tầng transport cần một phương pháp để phân biệt các ứng dụng trên một host. Trong TCP và UDP, sự phân biệt này được cung cấp bởi port number 16-bit.

#### **Well-known, registered và privileged port**

Một số port number well-known được gán vĩnh viễn cho các ứng dụng cụ thể (còn được gọi là dịch vụ). Ví dụ, daemon `ssh` (secure shell) dùng port well-known 22, và HTTP (giao thức dùng cho giao tiếp giữa web server và trình duyệt) dùng port well-known 80. Các port well-known được gán số trong dải 0 đến 1023 bởi một cơ quan trung ương, Internet Assigned Numbers Authority (IANA, http://www.iana.org/). Việc gán một port number well-known phụ thuộc vào một đặc tả mạng được chấp thuận (thường là dưới dạng RFC).

IANA cũng ghi lại các registered port, được phân bổ cho các nhà phát triển ứng dụng trên cơ sở ít nghiêm ngặt hơn (điều này cũng có nghĩa là một implementation không cần đảm bảo tính khả dụng của các port này cho mục đích đã đăng ký của chúng). Dải IANA registered port là 1024 đến 41951. (Không phải tất cả port number trong dải này đều được đăng ký.)

Trong hầu hết các implementation TCP/IP (bao gồm Linux), các port number trong dải 0 đến 1023 cũng là privileged, có nghĩa là chỉ các process có đặc quyền (`CAP_NET_BIND_SERVICE`) mới có thể bind vào các port này. Điều này ngăn người dùng bình thường triển khai một ứng dụng độc hại có thể, ví dụ, giả mạo là ssh để lấy mật khẩu. (Đôi khi, privileged port được gọi là reserved port.)

Mặc dù TCP và UDP port có cùng số là các thực thể riêng biệt, cùng một port number well-known thường được gán cho một dịch vụ dưới cả TCP và UDP, ngay cả khi, như thường xảy ra, dịch vụ đó chỉ khả dụng dưới một trong hai giao thức này. Quy ước này tránh nhầm lẫn port number trên hai giao thức.

## **Ephemeral port**

Nếu một ứng dụng không chọn một port cụ thể (tức là, theo thuật ngữ socket, nó không `bind()` socket của mình vào một port cụ thể), thì TCP và UDP gán một số ephemeral port (tức là tồn tại trong thời gian ngắn) duy nhất cho socket. Trong trường hợp này, ứng dụng—thường là client—không quan tâm đến port number nào nó dùng, nhưng việc gán port là cần thiết để các giao thức tầng transport có thể nhận dạng các endpoint giao tiếp. Nó cũng có kết quả là ứng dụng peer ở đầu kia của kênh giao tiếp biết cách giao tiếp với ứng dụng này. TCP và UDP cũng gán một số ephemeral port nếu chúng ta bind socket vào port 0.

IANA chỉ định các port trong dải 49152 đến 65535 là dynamic hoặc private, với mục đích là các port này có thể được sử dụng bởi các ứng dụng cục bộ và được gán làm ephemeral port. Tuy nhiên, các implementation khác nhau phân bổ ephemeral port từ các dải khác nhau. Trên Linux, dải này được định nghĩa bởi (và có thể được sửa đổi thông qua) hai số chứa trong file `/proc/sys/net/ipv4/ip_local_port_range`.

# **58.6.2 User Datagram Protocol (UDP)**

UDP chỉ thêm hai tính năng vào IP: port number và một data checksum để cho phép phát hiện lỗi trong dữ liệu được truyền.

Giống như IP, UDP là connectionless. Vì nó không thêm độ tin cậy vào IP, UDP cũng không đáng tin cậy. Nếu một ứng dụng được xếp lớp trên UDP yêu cầu độ tin cậy, thì điều này phải được implement trong ứng dụng. Mặc dù không đáng tin cậy này, đôi khi chúng ta có thể muốn dùng UDP thay vì TCP, vì các lý do được trình bày chi tiết trong Mục 61.12.

Checksum được dùng bởi cả UDP và TCP chỉ dài 16 bit, và là các checksum "cộng lên" đơn giản có thể không phát hiện được một số lớp lỗi. Do đó, chúng không cung cấp phát hiện lỗi cực kỳ mạnh. Các Internet server bận rộn thường gặp trung bình một lỗi truyền không được phát hiện mỗi vài ngày. Các ứng dụng cần bảo đảm tính toàn vẹn dữ liệu mạnh hơn có thể sử dụng giao thức Secure Sockets Layer (SSL), cung cấp không chỉ giao tiếp an toàn mà còn phát hiện lỗi chặt chẽ hơn nhiều. Hoặc, một ứng dụng có thể implement sơ đồ kiểm soát lỗi của riêng mình.

## **Chọn kích thước UDP datagram để tránh IP fragmentation**

Trong Mục 58.4, chúng ta đã mô tả cơ chế IP fragmentation, và lưu ý rằng thường tốt nhất là tránh IP fragmentation. Trong khi TCP có cơ chế để tránh IP fragmentation, UDP thì không. Với UDP, chúng ta có thể dễ dàng gây ra IP fragmentation bằng cách truyền một datagram vượt quá MTU của data link cục bộ.

Một ứng dụng dựa trên UDP thường không biết MTU của path giữa host nguồn và host đích. Các ứng dụng dựa trên UDP nhằm tránh IP fragmentation thường áp dụng một cách tiếp cận thận trọng, là đảm bảo rằng IP datagram được truyền nhỏ hơn kích thước buffer tổng hợp tối thiểu IPv4 là 576 byte. (Giá trị này có thể thấp hơn path MTU.) Từ 576 byte này, 8 byte được yêu cầu bởi chính header UDP, và tối thiểu thêm 20 byte được yêu cầu cho IP header, còn lại 548 byte cho chính UDP datagram. Trên thực tế, nhiều ứng dụng dựa trên UDP chọn một giới hạn thậm chí thấp hơn là 512 byte cho datagram của chúng ([Stevens, 1994]).

# **58.6.3 Transmission Control Protocol (TCP)**

TCP cung cấp một kênh giao tiếp byte-stream, bidirectional, kết nối dựa, đáng tin cậy giữa hai endpoint (tức là, các ứng dụng), như được hiển thị trong Hình 58-8. Để cung cấp các tính năng này, TCP phải thực hiện các nhiệm vụ được mô tả trong phần này. (Mô tả chi tiết về tất cả các tính năng này có thể được tìm thấy trong [Stevens, 1994].)

**Hình 58-8:** Các TCP socket đã kết nối

Chúng ta dùng thuật ngữ TCP endpoint để biểu thị thông tin được kernel duy trì cho một đầu của kết nối TCP. Thông tin này bao gồm các send và receive buffer cho đầu kết nối này, cũng như thông tin trạng thái được duy trì để đồng bộ hóa hoạt động của hai endpoint đã kết nối. (Chúng ta mô tả thông tin trạng thái này chi tiết hơn khi chúng ta xem xét sơ đồ chuyển đổi trạng thái TCP trong Mục 61.6.3.) Trong phần còn lại của cuốn sách này, chúng ta dùng các thuật ngữ receiving TCP và sending TCP để biểu thị các TCP endpoint được duy trì cho các ứng dụng nhận và gửi ở cả hai đầu của kết nối stream socket đang được dùng để truyền dữ liệu theo một hướng cụ thể.

## **Thiết lập kết nối**

Trước khi giao tiếp có thể bắt đầu, TCP thiết lập một kênh giao tiếp giữa hai endpoint. Trong quá trình thiết lập kết nối, sender và receiver có thể trao đổi các tùy chọn để quảng cáo các tham số cho kết nối.

## **Đóng gói dữ liệu vào segment**

Dữ liệu được chia thành các segment, mỗi segment chứa một checksum để cho phép phát hiện lỗi truyền end-to-end. Mỗi segment được truyền trong một IP datagram duy nhất.

### **Acknowledgement, truyền lại và timeout**

Khi một TCP segment đến đích mà không có lỗi, TCP nhận gửi một acknowledgement tích cực đến sender, thông báo cho nó về dữ liệu đã được giao thành công. Nếu một segment đến với lỗi, thì nó bị loại bỏ và không có acknowledgement nào được gửi. Để xử lý khả năng các segment không bao giờ đến hoặc bị loại bỏ, sender khởi động một timer khi mỗi segment được truyền. Nếu acknowledgement không được nhận trước khi timer hết hạn, segment sẽ được truyền lại.

> Vì thời gian cần để truyền một segment và nhận acknowledgement của nó thay đổi theo phạm vi của mạng và tải lưu lượng hiện tại, TCP sử dụng một thuật toán để điều chỉnh động kích thước retransmission timeout (RTO).

> TCP nhận có thể không gửi acknowledgement ngay lập tức, mà thay vào đó đợi một phần của một giây để xem liệu acknowledgement có thể được gói kèm trong bất kỳ phản hồi nào mà receiver có thể gửi thẳng trở lại sender không. (Mỗi TCP segment bao gồm một trường acknowledgement, cho phép piggybacking như vậy.) Mục tiêu của kỹ thuật này, được gọi là delayed ACK, là tiết kiệm việc gửi một TCP segment, do đó giảm số lượng packet trong mạng và giảm tải trên host gửi và nhận.

## **Sequencing**

Mỗi byte được truyền qua kết nối TCP được gán một số thứ tự logic. Con số này cho biết vị trí của byte đó trong luồng dữ liệu cho kết nối. (Mỗi trong hai luồng trong kết nối có số thứ tự riêng.) Khi một TCP segment được truyền, nó bao gồm một trường chứa sequence number của byte đầu tiên trong segment.

Việc gắn sequence number vào mỗi segment phục vụ nhiều mục đích:

- Sequence number cho phép TCP segment được tổng hợp theo đúng thứ tự ở đích, và sau đó được truyền dưới dạng byte stream đến application layer. (Tại bất kỳ thời điểm nào, nhiều TCP segment có thể đang trong quá trình truyền giữa sender và receiver, và các segment này có thể đến không theo thứ tự.)
- Message acknowledgement được truyền từ receiver về sender có thể dùng sequence number để nhận dạng TCP segment nào đã được nhận.
- Receiver có thể dùng sequence number để loại bỏ các segment trùng lặp. Các bản sao như vậy có thể xảy ra vì sự trùng lặp của IP datagram hoặc vì thuật toán truyền lại của TCP, có thể truyền lại một segment đã được giao thành công nếu acknowledgement cho segment đó bị mất hoặc không được nhận kịp thời.

Initial sequence number (ISN) cho một luồng không bắt đầu từ 0. Thay vào đó, nó được tạo ra thông qua một thuật toán làm tăng ISN được gán cho các kết nối TCP liên tiếp (để ngăn khả năng các segment cũ từ incarnation trước của kết nối bị nhầm lẫn với các segment cho kết nối này). Thuật toán này cũng được thiết kế để làm cho việc đoán ISN trở nên khó khăn. Sequence number là giá trị 32-bit được đặt lại về 0 khi đạt đến giá trị tối đa.

### **Flow control**

Flow control ngăn sender nhanh áp đảo receiver chậm. Để implement flow control, TCP nhận duy trì một buffer cho dữ liệu đến. (Mỗi TCP quảng cáo kích thước của buffer này trong quá trình thiết lập kết nối.) Dữ liệu tích lũy trong buffer này khi nó được nhận từ TCP gửi, và được xóa khi ứng dụng đọc dữ liệu. Với mỗi acknowledgement, receiver thông báo cho sender biết bao nhiêu không gian có sẵn trong buffer dữ liệu đến của nó (tức là, bao nhiêu byte sender có thể truyền). Thuật toán flow control TCP sử dụng cái gọi là thuật toán sliding window, cho phép các segment chưa được xác nhận chứa tổng cộng N byte (kích thước cửa sổ được cung cấp) trong quá trình truyền giữa sender và receiver. Nếu buffer dữ liệu đến của TCP nhận đầy hoàn toàn, thì cửa sổ được cho là đóng, và TCP gửi ngừng truyền.

> Receiver có thể ghi đè kích thước mặc định cho buffer dữ liệu đến bằng tùy chọn socket `SO_RCVBUF` (xem trang manual `socket(7)`).

#### **Kiểm soát tắc nghẽn: thuật toán slow-start và congestion-avoidance**

Các thuật toán kiểm soát tắc nghẽn của TCP được thiết kế để ngăn sender nhanh áp đảo mạng. Nếu một TCP gửi truyền packet nhanh hơn chúng có thể được chuyển tiếp bởi router trung gian, router đó sẽ bắt đầu loại bỏ packet. Điều này có thể dẫn đến tỷ lệ mất packet cao và do đó, suy giảm hiệu suất nghiêm trọng, nếu TCP gửi tiếp tục truyền lại các segment bị loại bỏ này ở cùng tốc độ. Các thuật toán kiểm soát tắc nghẽn của TCP quan trọng trong hai trường hợp:

- Sau khi thiết lập kết nối: Tại thời điểm này (hoặc khi quá trình truyền nối lại trên kết nối đã nhàn rỗi một thời gian), sender có thể bắt đầu bằng cách ngay lập tức đưa vào mạng nhiều segment như kích thước window được quảng cáo bởi receiver cho phép. Vấn đề ở đây là nếu mạng không thể xử lý lũ lụt các segment này, sender có nguy cơ áp đảo mạng ngay lập tức.
- Khi phát hiện tắc nghẽn: Nếu TCP gửi phát hiện ra tắc nghẽn đang xảy ra, thì nó phải giảm tốc độ truyền của mình. TCP phát hiện tắc nghẽn đang xảy ra dựa trên giả định rằng mất packet do lỗi truyền là rất thấp; do đó, nếu packet bị mất, nguyên nhân được giả định là do tắc nghẽn.

Chiến lược kiểm soát tắc nghẽn của TCP sử dụng hai thuật toán kết hợp: slow start và congestion avoidance.

Thuật toán slow-start khiến TCP gửi ban đầu truyền segment với tốc độ chậm, nhưng cho phép nó tăng tốc độ theo cấp số nhân khi các segment này được xác nhận bởi TCP nhận. Slow start cố gắng ngăn TCP sender nhanh áp đảo mạng. Tuy nhiên, nếu không được kiểm soát, sự tăng theo cấp số nhân về tốc độ truyền của slow start có thể có nghĩa là sender sẽ sớm áp đảo mạng. Thuật toán congestion-avoidance của TCP ngăn điều này xảy ra, bằng cách đặt bộ điều chỉnh tốc độ tăng.

Với congestion avoidance, ở đầu kết nối, TCP gửi bắt đầu với một congestion window nhỏ, giới hạn lượng dữ liệu chưa được xác nhận mà nó có thể truyền. Khi sender nhận acknowledgement từ TCP peer, congestion window ban đầu phát triển theo cấp số nhân. Tuy nhiên, khi congestion window đạt đến một ngưỡng nhất định được cho là gần với dung lượng truyền của mạng, sự phát triển của nó trở thành tuyến tính, thay vì theo cấp số nhân. (Ước tính về dung lượng của mạng được bắt nguồn từ phép tính dựa trên tốc độ truyền đang hoạt động khi tắc nghẽn được phát hiện, hoặc được đặt ở giá trị cố định sau khi thiết lập ban đầu của kết nối.) Tại mọi thời điểm, lượng dữ liệu mà TCP gửi sẽ truyền vẫn bị hạn chế bổ sung bởi window được quảng cáo của TCP nhận và send buffer của TCP cục bộ.

Kết hợp lại, các thuật toán slow-start và congestion-avoidance cho phép sender nhanh chóng nâng tốc độ truyền lên đến dung lượng khả dụng của mạng, mà không vượt quá dung lượng đó. Hiệu ứng của các thuật toán này là cho phép việc truyền dữ liệu nhanh chóng đạt đến trạng thái cân bằng, nơi sender truyền packet với cùng tốc độ nhận acknowledgement từ receiver.

# **58.7 Requests for Comments (RFC)**

Mỗi giao thức Internet mà chúng ta thảo luận trong cuốn sách này được định nghĩa trong một tài liệu RFC—một đặc tả giao thức chính thức. RFC được xuất bản bởi RFC Editor (http://www.rfc-editor.org/), được tài trợ bởi Internet Society (http://www.isoc.org/). Các RFC mô tả các tiêu chuẩn Internet được phát triển dưới sự bảo trợ của Internet Engineering Task Force (IETF, http://www.ietf.org/), một cộng đồng các nhà thiết kế mạng, nhà điều hành, nhà cung cấp và nhà nghiên cứu liên quan đến sự phát triển và hoạt động suôn sẻ của Internet. Thành viên của IETF mở cửa cho bất kỳ cá nhân quan tâm nào.

Các RFC sau đây đặc biệt liên quan đến tài liệu được đề cập trong cuốn sách này:

- RFC 791, Internet Protocol. J. Postel (ed.), 1981.
- RFC 950, Internet Standard Subnetting Procedure. J. Mogul and J. Postel, 1985.
- RFC 793, Transmission Control Protocol. J. Postel (ed.), 1981.
- RFC 768, User Datagram Protocol. J. Postel (ed.), 1980.
- RFC 1122, Requirements for Internet Hosts—Communication Layers. R. Braden (ed.), 1989.

RFC 1122 mở rộng (và sửa chữa) nhiều RFC trước đó mô tả các giao thức TCP/IP. Đây là một trong một cặp RFC thường được gọi là Host Requirements RFC. Thành viên còn lại của cặp là RFC 1123, bao gồm các giao thức application-layer như telnet, FTP và SMTP.

Trong số các RFC mô tả IPv6 có:

- RFC 2460, Internet Protocol, Version 6. S. Deering and R. Hinden, 1998.
- RFC 4291, IP Version 6 Addressing Architecture. R. Hinden and S. Deering, 2006.
- RFC 3493, Basic Socket Interface Extensions for IPv6. R. Gilligan, S. Thomson, J. Bound, J. McCann, and W. Stevens, 2003.
- RFC 3542, Advanced Sockets API for IPv6. W. Stevens, M. Thomas, E. Nordmark, and T. Jinmei, 2003.

Một số RFC và bài báo cung cấp các cải tiến và phần mở rộng cho đặc tả TCP ban đầu, bao gồm:

- Congestion Avoidance and Control. V. Jacobsen, 1988.
- RFC 1323, TCP Extensions for High Performance. V. Jacobson, R. Braden, and D. Borman, 1992.
- RFC 2018, TCP Selective Acknowledgment Options. M. Mathis, J. Mahdavi, S. Floyd, and A. Romanow, 1996.
- RFC 2581, TCP Congestion Control. M. Allman, V. Paxson, and W. Stevens, 1999.
- RFC 2861, TCP Congestion Window Validation. M. Handley, J. Padhye, and S. Floyd, 2000.
- RFC 2883, An Extension to the Selective Acknowledgement (SACK) Option for TCP. S. Floyd, J. Mahdavi, M. Mathis, and M. Podolsky, 2000.
- RFC 2988, Computing TCP's Retransmission Timer. V. Paxson and M. Allman, 2000.
- RFC 3168, The Addition of Explicit Congestion Notification (ECN) to IP. K. Ramakrishnan, S. Floyd, and D. Black, 2001.
- RFC 3390, Increasing TCP's Initial Window. M. Allman, S. Floyd, and C. Partridge, 2002.

# **58.8 Tóm Tắt**

TCP/IP là một bộ giao thức mạng phân lớp. Ở lớp dưới cùng của TCP/IP protocol stack là giao thức IP network-layer. IP truyền dữ liệu dưới dạng datagram. IP là connectionless, có nghĩa là datagram được truyền giữa host nguồn và host đích có thể đi theo các tuyến đường khác nhau qua mạng. IP không đáng tin cậy, ở chỗ nó không đảm bảo datagram sẽ đến theo thứ tự hoặc không bị trùng lặp, hoặc thậm chí đến nơi. Nếu cần độ tin cậy, thì nó phải được cung cấp thông qua việc sử dụng giao thức cao hơn đáng tin cậy (ví dụ: TCP), hoặc trong một ứng dụng.

Phiên bản gốc của IP là IPv4. Vào đầu những năm 1990, một phiên bản mới của IP, IPv6, đã được thiết kế. Sự khác biệt đáng chú ý nhất giữa IPv4 và IPv6 là IPv4 dùng 32 bit để đại diện cho địa chỉ host, trong khi IPv6 dùng 128 bit, do đó cho phép số lượng host lớn hơn nhiều trên Internet toàn thế giới. Hiện tại, IPv4 vẫn là phiên bản IP được sử dụng rộng rãi nhất, mặc dù trong những năm tới, nó có thể sẽ được thay thế bởi IPv6.

Nhiều giao thức tầng transport được xếp lớp trên IP, trong đó được sử dụng rộng rãi nhất là UDP và TCP. UDP là giao thức datagram không đáng tin cậy. TCP là giao thức byte-stream đáng tin cậy, kết nối định hướng. TCP xử lý tất cả các chi tiết thiết lập và kết thúc kết nối. TCP cũng đóng gói dữ liệu vào segment để IP truyền, và cung cấp sequence number cho các segment này để chúng có thể được xác nhận và tổng hợp theo đúng thứ tự bởi receiver. Ngoài ra, TCP cung cấp flow control, để ngăn sender nhanh áp đảo receiver chậm, và congestion control, để ngăn sender nhanh áp đảo mạng.

#### **Tài liệu tham khảo**

Tham khảo các nguồn thông tin thêm được liệt kê trong Mục 59.15.
