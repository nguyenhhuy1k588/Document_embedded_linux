## Chương 42
# **CÁC TÍNH NĂNG NÂNG CAO CỦA SHARED LIBRARY**

Chương trước đã trình bày những kiến thức cơ bản về shared library. Chương này mô tả một số tính năng nâng cao của shared library, bao gồm những nội dung sau:

-  tải động shared library;
-  kiểm soát tầm nhìn của các symbol được định nghĩa bởi shared library;
-  sử dụng linker script để tạo versioned symbol;
-  sử dụng hàm khởi tạo và hàm kết thúc để tự động thực thi code khi thư viện được tải và gỡ tải;
-  preloading shared library; và
-  sử dụng `LD_DEBUG` để theo dõi hoạt động của dynamic linker.

### <span id="page-126-0"></span>**42.1 Dynamically Loaded Library**

Khi một chương trình thực thi khởi động, dynamic linker sẽ tải tất cả các shared library trong danh sách phụ thuộc động của chương trình. Tuy nhiên, đôi khi việc tải thư viện vào một thời điểm muộn hơn lại hữu ích hơn. Ví dụ, một plug-in chỉ được tải khi nó thực sự cần thiết. Chức năng này được cung cấp thông qua một API cho dynamic linker. API này, thường được gọi là dlopen API, có nguồn gốc từ Solaris, và phần lớn đã được đặc tả trong SUSv3.

dlopen API cho phép một chương trình mở một shared library vào lúc chạy, tìm kiếm một hàm theo tên trong thư viện đó, rồi gọi hàm đó. Một shared library được tải theo cách này lúc chạy thường được gọi là dynamically loaded library, và được tạo ra theo cách giống như bất kỳ shared library nào khác.

Phần cốt lõi của dlopen API bao gồm các hàm sau (tất cả đều được đặc tả trong SUSv3):

-  Hàm `dlopen()` mở một shared library, trả về một handle để sử dụng trong các lần gọi tiếp theo.
-  Hàm `dlsym()` tìm kiếm trong thư viện một symbol (một chuỗi chứa tên của hàm hoặc biến) và trả về địa chỉ của nó.
-  Hàm `dlclose()` đóng một thư viện đã được mở trước đó bằng `dlopen()`.
-  Hàm `dlerror()` trả về một chuỗi thông báo lỗi, và được sử dụng sau khi một trong các hàm trên trả về lỗi.

Phiên bản triển khai glibc cũng bao gồm một số hàm liên quan, một số trong đó sẽ được mô tả bên dưới.

Để biên dịch các chương trình sử dụng dlopen API trên Linux, chúng ta phải chỉ định tùy chọn `–ldl` để liên kết với thư viện `libdl`.

### <span id="page-127-0"></span>**42.1.1 Mở một Shared Library: dlopen()**

Hàm `dlopen()` tải shared library có tên trong `libfilename` vào virtual address space của process đang gọi và tăng bộ đếm số tham chiếu đang mở đến thư viện.

```
#include <dlfcn.h>
void *dlopen(const char *libfilename, int flags);
                              Returns library handle on success, or NULL on error
```

Nếu `libfilename` chứa dấu gạch chéo (`/`), `dlopen()` sẽ diễn giải nó là một đường dẫn tuyệt đối hoặc tương đối. Nếu không, dynamic linker sẽ tìm kiếm shared library theo các quy tắc được mô tả trong Mục [41.11.](#page-121-1)

Khi thành công, `dlopen()` trả về một handle có thể dùng để tham chiếu đến thư viện trong các lần gọi tiếp theo đến các hàm trong dlopen API. Nếu có lỗi xảy ra (ví dụ: không tìm thấy thư viện), `dlopen()` trả về `NULL`.

Nếu shared library được chỉ định bởi `libfilename` có các phụ thuộc vào các shared library khác, `dlopen()` cũng tự động tải các thư viện đó. Quá trình này diễn ra đệ quy nếu cần thiết. Chúng ta gọi tập hợp các thư viện được tải như vậy là dependency tree của thư viện này.

Có thể gọi `dlopen()` nhiều lần trên cùng một file thư viện. Thư viện chỉ được tải vào bộ nhớ một lần (trong lần gọi đầu tiên), và tất cả các lần gọi đều trả về cùng một giá trị handle. Tuy nhiên, dlopen API duy trì một reference count cho mỗi library handle. Bộ đếm này được tăng lên mỗi lần gọi `dlopen()` và giảm đi mỗi lần gọi `dlclose()`; chỉ khi bộ đếm giảm về 0 thì `dlclose()` mới thực sự gỡ tải thư viện khỏi bộ nhớ.

Đối số `flags` là một bitmask phải chứa đúng một trong hai hằng số `RTLD_LAZY` hoặc `RTLD_NOW`, với ý nghĩa như sau:

#### RTLD\_LAZY

Các function symbol chưa được xác định trong thư viện chỉ cần được giải quyết khi code đang thực thi. Nếu một đoạn code yêu cầu một symbol cụ thể nhưng không được thực thi, symbol đó sẽ không bao giờ được giải quyết. Lazy resolution chỉ áp dụng cho tham chiếu hàm; các tham chiếu đến biến luôn được giải quyết ngay lập tức. Việc chỉ định cờ `RTLD_LAZY` tạo ra hành vi tương ứng với hoạt động bình thường của dynamic linker khi tải các shared library được xác định trong danh sách phụ thuộc động của một chương trình thực thi.

#### RTLD\_NOW

Tất cả các symbol chưa được xác định trong thư viện phải được giải quyết ngay lập tức trước khi `dlopen()` hoàn thành, bất kể chúng có thực sự được sử dụng hay không. Hệ quả là việc mở thư viện sẽ chậm hơn, nhưng bất kỳ lỗi function symbol nào chưa được xác định sẽ được phát hiện ngay lập tức thay vì ở một thời điểm muộn hơn. Điều này hữu ích khi debug một ứng dụng, hoặc đơn giản là để đảm bảo rằng ứng dụng thất bại ngay lập tức khi có symbol chưa được xác định, thay vì chỉ xảy ra sau khi đã chạy được một thời gian dài.

Bằng cách đặt biến môi trường `LD_BIND_NOW` thành một chuỗi không rỗng, chúng ta có thể buộc dynamic linker giải quyết tất cả symbol ngay lập tức (tương tự như `RTLD_NOW`) khi tải các shared library được xác định trong danh sách phụ thuộc động của một chương trình thực thi. Biến môi trường này có hiệu lực trong glibc 2.1.1 trở lên. Đặt `LD_BIND_NOW` sẽ ghi đè hiệu ứng của cờ `RTLD_LAZY` của `dlopen()`.

Ngoài ra, cũng có thể bao gồm các giá trị khác trong `flags`. Các cờ sau được đặc tả trong SUSv3:

#### RTLD\_GLOBAL

Các symbol trong thư viện này và dependency tree của nó được làm sẵn để giải quyết các tham chiếu trong các thư viện khác được tải bởi process này và cũng dùng cho việc tra cứu thông qua `dlsym()`.

#### RTLD\_LOCAL

Đây là phép đối nghịch của `RTLD_GLOBAL` và là giá trị mặc định nếu không chỉ định hằng số nào. Nó chỉ định rằng các symbol trong thư viện này và dependency tree của nó không sẵn có để giải quyết các tham chiếu trong các thư viện được tải sau này.

SUSv3 không chỉ định giá trị mặc định nếu cả `RTLD_GLOBAL` lẫn `RTLD_LOCAL` đều không được chỉ định. Hầu hết các bản triển khai UNIX giả định cùng giá trị mặc định (`RTLD_LOCAL`) như Linux, nhưng một số ít giả định mặc định là `RTLD_GLOBAL`.

Linux cũng hỗ trợ một số cờ không được đặc tả trong SUSv3:

#### RTLD\_NODELETE (từ glibc 2.2)

Không gỡ tải thư viện khi `dlclose()`, ngay cả khi reference count giảm về 0. Điều này có nghĩa là các biến static của thư viện sẽ không bị khởi tạo lại nếu thư viện được tải lại sau đó bằng `dlopen()`. (Chúng ta có thể đạt hiệu ứng tương tự cho các thư viện được tải tự động bởi dynamic linker bằng cách chỉ định tùy chọn `gcc –Wl,–znodelete` khi tạo thư viện.)

#### RTLD\_NOLOAD (từ glibc 2.2)

Không tải thư viện. Cờ này phục vụ hai mục đích. Thứ nhất, chúng ta có thể sử dụng cờ này để kiểm tra xem một thư viện cụ thể có đang được tải như một phần của address space của process hay không. Nếu có, `dlopen()` trả về handle của thư viện; nếu không, `dlopen()` trả về `NULL`. Thứ hai, chúng ta có thể sử dụng cờ này để "nâng cấp" các cờ của một thư viện đã được tải. Ví dụ, chúng ta có thể chỉ định `RTLD_NOLOAD | RTLD_GLOBAL` trong `flags` khi sử dụng `dlopen()` trên một thư viện đã được mở trước đó với `RTLD_LOCAL`.

#### RTLD\_DEEPBIND (từ glibc 2.3.4)

Khi giải quyết các tham chiếu symbol được thực hiện bởi thư viện này, hãy tìm kiếm các định nghĩa trong thư viện trước khi tìm kiếm trong các thư viện đã được tải trước đó. Điều này cho phép một thư viện tự bao hàm, sử dụng các định nghĩa symbol của chính nó ưu tiên hơn so với các global symbol cùng tên được định nghĩa trong các shared library khác đã được tải. (Điều này tương tự như hiệu ứng của tùy chọn linker `–Bsymbolic` được mô tả trong Mục [41.12.](#page-121-2))

Các cờ `RTLD_NODELETE` và `RTLD_NOLOAD` cũng được triển khai trong dlopen API của Solaris, nhưng không có trên nhiều bản triển khai UNIX khác. Cờ `RTLD_DEEPBIND` là đặc thù của Linux.

Theo một trường hợp đặc biệt, chúng ta có thể chỉ định `libfilename` là `NULL`. Điều này khiến `dlopen()` trả về một handle cho chương trình chính. (SUSv3 gọi đây là handle cho "global symbol object".) Chỉ định handle này trong lần gọi tiếp theo đến `dlsym()` sẽ khiến symbol được yêu cầu được tìm kiếm trong chương trình chính, sau đó là tất cả các shared library được tải khi chương trình khởi động, và tiếp theo là tất cả các thư viện được tải động với cờ `RTLD_GLOBAL`.

### **42.1.2 Chẩn đoán Lỗi: dlerror()**

Nếu chúng ta nhận được giá trị trả về lỗi từ `dlopen()` hoặc một trong các hàm khác trong dlopen API, chúng ta có thể sử dụng `dlerror()` để lấy một con trỏ tới chuỗi chỉ ra nguyên nhân của lỗi.

```
#include <dlfcn.h>
const char *dlerror(void);
                              Returns pointer to error-diagnostic string, or NULL if
                             no error has occurred since previous call to dlerror()
```

Hàm `dlerror()` trả về `NULL` nếu không có lỗi nào xảy ra kể từ lần gọi `dlerror()` trước đó. Chúng ta sẽ thấy điều này hữu ích như thế nào trong phần tiếp theo.

### **42.1.3 Lấy Địa Chỉ của một Symbol: dlsym()**

Hàm `dlsym()` tìm kiếm symbol được đặt tên (một hàm hoặc biến) trong thư viện được tham chiếu bởi `handle` và trong các thư viện trong dependency tree của thư viện đó.

```
#include <dlfcn.h>
void *dlsym(void *handle, char *symbol);
                          Returns address of symbol, or NULL if symbol is not found
```

Nếu `symbol` được tìm thấy, `dlsym()` trả về địa chỉ của nó; nếu không, `dlsym()` trả về `NULL`. Đối số `handle` thông thường là một library handle được trả về bởi lần gọi `dlopen()` trước đó. Ngoài ra, nó cũng có thể là một trong các pseudohandle được mô tả bên dưới.

> Một hàm liên quan, `dlvsym(handle, symbol, version)`, tương tự như `dlsym()`, nhưng có thể được sử dụng để tìm kiếm trong một thư viện có symbol versioning một định nghĩa symbol có version khớp với chuỗi được chỉ định trong `version`. (Chúng ta mô tả symbol versioning trong Mục [42.3.2](#page-137-0).) Feature test macro `_GNU_SOURCE` phải được định nghĩa để lấy khai báo của hàm này từ `<dlfcn.h>`.

Giá trị của một symbol được trả về bởi `dlsym()` có thể là `NULL`, điều này không thể phân biệt với kết quả trả về "không tìm thấy symbol". Để phân biệt hai khả năng này, chúng ta phải gọi `dlerror()` trước (để đảm bảo rằng bất kỳ chuỗi lỗi nào được giữ trước đó đều đã được xóa) và sau đó nếu, sau lần gọi `dlsym()`, `dlerror()` trả về một giá trị khác `NULL`, chúng ta biết rằng đã xảy ra lỗi.

Nếu `symbol` là tên của một biến, thì chúng ta có thể gán giá trị trả về của `dlsym()` cho một kiểu con trỏ phù hợp, và lấy giá trị của biến bằng cách dereference con trỏ:

```
int *ip;
ip = (int *) dlsym(symbol, "myvar");
if (ip != NULL)
 printf("Value is %d\n", *ip);
```

Nếu `symbol` là tên của một hàm, thì con trỏ được trả về bởi `dlsym()` có thể được dùng để gọi hàm đó. Chúng ta có thể lưu giá trị trả về bởi `dlsym()` vào một con trỏ có kiểu phù hợp, chẳng hạn như sau:

```
int (*funcp)(int); /* Pointer to a function taking an integer
 argument and returning an integer */
```

Tuy nhiên, chúng ta không thể đơn giản gán kết quả của `dlsym()` cho con trỏ như vậy, như trong ví dụ sau:

```
funcp = dlsym(handle, symbol);
```

Lý do là tiêu chuẩn C99 cấm việc gán giữa một function pointer và `void *`. Giải pháp là sử dụng cú pháp (hơi cồng kềnh) sau đây:

```
*(void **) (&funcp) = dlsym(handle, symbol);
```

Sau khi dùng `dlsym()` để lấy con trỏ đến hàm, chúng ta có thể gọi hàm bằng cú pháp C thông thường để dereference function pointer:

```
res = (*funcp)(somearg);
```

Thay vì cú pháp `*(void **)` được trình bày ở trên, người ta có thể xem xét sử dụng đoạn code có vẻ tương đương sau đây khi gán giá trị trả về của `dlsym()`:

```
(void *) funcp = dlsym(handle, symbol);
```

Tuy nhiên, với code này, `gcc –pedantic` cảnh báo rằng "ANSI C forbids the use of cast expressions as lvalues." Cú pháp `*(void **)` không gây ra cảnh báo này vì chúng ta đang gán cho một địa chỉ được trỏ bởi lvalue của phép gán.

Trên nhiều bản triển khai UNIX, chúng ta có thể sử dụng các casts như sau để loại bỏ cảnh báo từ trình biên dịch C:

```
funcp = (int (*) (int)) dlsym(handle, symbol);
```

Tuy nhiên, đặc tả của `dlsym()` trong SUSv3 Technical Corrigendum Number 1 lưu ý rằng tiêu chuẩn C99 vẫn yêu cầu các trình biên dịch tạo ra cảnh báo cho sự chuyển đổi như vậy, và đề xuất cú pháp `*(void **)` được trình bày ở trên.

> SUSv3 TC1 lưu ý rằng do cần đến cú pháp `*(void **)`, một phiên bản tiêu chuẩn tương lai có thể định nghĩa các API riêng tương tự `dlsym()` để xử lý data pointer và function pointer. Tuy nhiên, SUSv4 không có thay đổi gì liên quan đến điểm này.

### **Sử dụng library pseudohandle với dlsym()**

Thay vì chỉ định một library handle được trả về bởi lần gọi `dlopen()`, một trong hai pseudohandle sau có thể được chỉ định làm đối số `handle` cho `dlsym()`:

RTLD\_DEFAULT

Tìm kiếm `symbol` bắt đầu từ chương trình chính, sau đó theo thứ tự qua danh sách tất cả các shared library đã được tải, bao gồm cả các thư viện được tải động bởi `dlopen()` với cờ `RTLD_GLOBAL`. Điều này tương ứng với mô hình tìm kiếm mặc định của dynamic linker.

RTLD\_NEXT

Tìm kiếm `symbol` trong các shared library được tải sau thư viện đang gọi `dlsym()`. Điều này hữu ích khi tạo một wrapper function cùng tên với một hàm được định nghĩa ở nơi khác. Ví dụ, trong chương trình chính của chúng ta, chúng ta có thể định nghĩa phiên bản `malloc()` của riêng mình (có thể thực hiện một số thống kê cấp phát bộ nhớ), và hàm này có thể gọi `malloc()` thực sự bằng cách trước tiên lấy địa chỉ của nó qua lời gọi `func = dlsym(RTLD_NEXT, "malloc")`.

Các giá trị pseudohandle được liệt kê ở trên không được yêu cầu bởi SUSv3 (mặc dù vẫn được dành để sử dụng trong tương lai), và không có sẵn trên tất cả các bản triển khai UNIX. Để lấy định nghĩa của các hằng số này từ `<dlfcn.h>`, chúng ta phải định nghĩa feature test macro `_GNU_SOURCE`.

#### **Chương trình ví dụ**

[Listing 42-1](#page-132-0) minh họa việc sử dụng dlopen API. Chương trình này nhận hai đối số dòng lệnh: tên của một shared library để tải và tên của một hàm để thực thi trong thư viện đó. Các ví dụ sau minh họa việc sử dụng chương trình này:

```
$ ./dynload ./libdemo.so.1 x1
Called mod1-x1
```

```
$ LD_LIBRARY_PATH=. ./dynload libdemo.so.1 x1
Called mod1-x1
```

Trong lệnh đầu tiên ở trên, `dlopen()` nhận thấy rằng đường dẫn thư viện có chứa dấu gạch chéo và do đó diễn giải nó là một đường dẫn tương đối (trong trường hợp này là đến một thư viện trong thư mục làm việc hiện tại). Trong lệnh thứ hai, chúng ta chỉ định một đường dẫn tìm kiếm thư viện trong `LD_LIBRARY_PATH`. Đường dẫn tìm kiếm này được giải thích theo các quy tắc thông thường của dynamic linker (trong trường hợp này, cũng để tìm thư viện trong thư mục làm việc hiện tại).

<span id="page-132-0"></span>**Listing 42-1:** Sử dụng dlopen API

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– shlibs/dynload.c
#include <dlfcn.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 void *libHandle; /* Handle for shared library */
 void (*funcp)(void); /* Pointer to function with no arguments */
 const char *err;
 if (argc != 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s lib-path func-name\n", argv[0]);
 /* Load the shared library and get a handle for later use */
 libHandle = dlopen(argv[1], RTLD_LAZY);
 if (libHandle == NULL)
 fatal("dlopen: %s", dlerror());
 /* Search library for symbol named in argv[2] */
 (void) dlerror(); /* Clear dlerror() */
 *(void **) (&funcp) = dlsym(libHandle, argv[2]);
 err = dlerror();
 if (err != NULL)
 fatal("dlsym: %s", err);
 /* If the address returned by dlsym() is non-NULL, try calling it
 as a function that takes no arguments */
 if (funcp == NULL)
 printf("%s is NULL\n", argv[2]);
 else
 (*funcp)();
 dlclose(libHandle); /* Close the library */
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– shlibs/dynload.c
```

### **42.1.4 Đóng một Shared Library: dlclose()**

Hàm `dlclose()` đóng một thư viện.

```
#include <dlfcn.h>
int dlclose(void *handle);
                                             Returns 0 on success, or –1 on error
```

Hàm `dlclose()` giảm bộ đếm tham chiếu mở của hệ thống đến thư viện được tham chiếu bởi `handle`. Nếu reference count này giảm về 0, và không có symbol nào trong thư viện được yêu cầu bởi các thư viện khác, thì thư viện đó sẽ được gỡ tải. Quá trình này cũng được thực hiện (đệ quy) đối với các thư viện trong dependency tree của thư viện này. Một `dlclose()` ngầm định cho tất cả các thư viện được thực hiện khi process kết thúc.

> Từ glibc 2.2.3 trở đi, một hàm trong shared library có thể sử dụng `atexit()` (hoặc `on_exit()`) để thiết lập một hàm được gọi tự động khi thư viện được gỡ tải.

### **42.1.5 Lấy Thông Tin Về Các Symbol Đã Tải: dladdr()**

Với một địa chỉ trong `addr` (thông thường là địa chỉ lấy được từ lần gọi `dlsym()` trước đó), `dladdr()` trả về một structure chứa thông tin về địa chỉ đó.

```
#define _GNU_SOURCE
#include <dlfcn.h>
int dladdr(const void *addr, Dl_info *info);
        Returns nonzero value if addr was found in a shared library, otherwise 0
```

Đối số `info` là một con trỏ tới một structure được cấp phát bởi caller, có dạng như sau:

```
typedef struct {
 const char *dli_fname; /* Pathname of shared library
 containing 'addr' */
 void *dli_fbase; /* Base address at which shared
 library is loaded */
 const char *dli_sname; /* Name of nearest run-time symbol
 with an address <= 'addr' */
 void *dli_saddr; /* Actual value of the symbol
 returned in 'dli_sname' */
} Dl_info;
```

Hai trường đầu tiên của structure `Dl_info` chỉ định pathname và base address lúc chạy của shared library chứa địa chỉ được chỉ định trong `addr`. Hai trường cuối trả về thông tin về địa chỉ đó. Giả sử rằng `addr` trỏ đến địa chỉ chính xác của một symbol trong shared library, thì `dli_saddr` trả về cùng giá trị như đã được truyền vào `addr`.

SUSv3 không đặc tả `dladdr()`, và hàm này không có sẵn trên tất cả các bản triển khai UNIX.

### <span id="page-134-0"></span>**42.1.6 Truy Cập Các Symbol Trong Chương Trình Chính**

Giả sử rằng chúng ta sử dụng `dlopen()` để tải động một shared library, dùng `dlsym()` để lấy địa chỉ của một hàm `x()` từ thư viện đó, rồi gọi `x()`. Nếu `x()` lần lượt gọi một hàm `y()`, thì `y()` thông thường sẽ được tìm kiếm trong một trong các shared library được tải bởi chương trình.

Đôi khi, thay vào đó ta muốn `x()` gọi một phiên bản triển khai của `y()` trong chương trình chính. (Điều này tương tự như cơ chế callback.) Để làm điều này, chúng ta phải làm cho các symbol (có phạm vi global) trong chương trình chính sẵn có cho dynamic linker, bằng cách liên kết chương trình sử dụng tùy chọn linker `––export–dynamic`:

```
$ gcc -Wl,--export-dynamic main.c (plus further options and arguments)
```

Tương đương, chúng ta có thể viết như sau:

```
$ gcc -export-dynamic main.c
```

Sử dụng một trong hai tùy chọn này cho phép một dynamically loaded library truy cập các global symbol trong chương trình chính.

> Tùy chọn `gcc –rdynamic` và tùy chọn `gcc –Wl,–E` là các từ đồng nghĩa khác của `–Wl,––export–dynamic`.

# **42.2 Kiểm Soát Tầm Nhìn của Symbol**

Một shared library được thiết kế tốt chỉ nên làm lộ ra những symbol (hàm và biến) tạo thành một phần của application binary interface (ABI) đã được chỉ định của nó. Lý do cho điều này như sau:

-  Nếu người thiết kế shared library vô tình xuất các interface không được chỉ định, thì các tác giả ứng dụng sử dụng thư viện đó có thể chọn sử dụng những interface này. Điều này tạo ra vấn đề tương thích cho các bản nâng cấp tương lai của shared library. Nhà phát triển thư viện kỳ vọng có thể thay đổi hoặc xóa bất kỳ interface nào ngoài những interface trong ABI đã được ghi chép, trong khi người dùng thư viện kỳ vọng tiếp tục sử dụng các interface giống nhau (với cùng ngữ nghĩa) mà họ hiện đang dùng.
-  Trong quá trình giải quyết symbol lúc chạy, bất kỳ symbol nào được xuất bởi shared library đều có thể xen vào (interpose) các định nghĩa được cung cấp trong các shared library khác (Mục [41.12](#page-121-2)).
-  Việc xuất các symbol không cần thiết làm tăng kích thước của dynamic symbol table phải được tải lúc chạy.

Tất cả những vấn đề này có thể được giảm thiểu hoặc tránh hoàn toàn nếu người thiết kế thư viện đảm bảo rằng chỉ những symbol được yêu cầu bởi ABI đã được chỉ định của thư viện mới được xuất. Các kỹ thuật sau có thể được sử dụng để kiểm soát việc xuất symbol:

 Trong một chương trình C, chúng ta có thể sử dụng từ khóa `static` để đặt một symbol thành private cho một module mã nguồn, do đó ngăn nó không thể bị binding bởi các object file khác.

Ngoài việc làm cho một symbol trở thành private cho một module mã nguồn, từ khóa `static` còn có hiệu ứng ngược lại. Nếu một symbol được đánh dấu là `static`, thì tất cả các tham chiếu đến symbol đó trong cùng source file sẽ được bound với định nghĩa đó của symbol. Do đó, các tham chiếu này sẽ không bị subject to interposition lúc chạy bởi các định nghĩa từ các shared library khác (theo cách được mô tả trong Mục [41.12\)](#page-121-2). Hiệu ứng này của từ khóa `static` tương tự như tùy chọn linker `–Bsymbolic` được mô tả trong Mục [41.12](#page-121-2), với điểm khác biệt là từ khóa `static` ảnh hưởng đến một symbol duy nhất trong một source file duy nhất.

 Trình biên dịch GNU C, `gcc`, cung cấp một khai báo attribute đặc thù của trình biên dịch thực hiện tác vụ tương tự như từ khóa `static`:

```
void
__attribute__ ((visibility("hidden")))
func(void) {
 /* Code */
}
```

Trong khi từ khóa `static` giới hạn tầm nhìn của một symbol trong một source file duy nhất, thuộc tính `hidden` làm cho symbol sẵn có trên tất cả các source file tạo nên shared library, nhưng ngăn nó không bị nhìn thấy bên ngoài thư viện.

> Giống như từ khóa `static`, thuộc tính `hidden` cũng có hiệu ứng ngược lại là ngăn symbol interposition lúc chạy.

-  Version script (Mục [42.3](#page-135-0)) có thể được sử dụng để kiểm soát chính xác tầm nhìn của symbol và để chọn phiên bản của một symbol mà tham chiếu sẽ được bound vào.
-  Khi tải động một shared library (Mục [42.1.1](#page-127-0)), cờ `RTLD_GLOBAL` của `dlopen()` có thể được sử dụng để chỉ định rằng các symbol được định nghĩa bởi thư viện phải được làm sẵn để binding bởi các thư viện được tải sau đó, và tùy chọn linker `––export–dynamic` (Mục [42.1.6\)](#page-134-0) có thể được sử dụng để làm cho các global symbol của chương trình chính sẵn có cho các dynamically loaded library.

Để biết thêm chi tiết về chủ đề tầm nhìn của symbol, xem [Drepper, 2004 (b)].

## <span id="page-135-0"></span>**42.3 Linker Version Script**

Một version script là một file văn bản chứa các hướng dẫn cho linker, `ld`. Để sử dụng một version script, chúng ta phải chỉ định tùy chọn linker `––version–script`:

```
$ gcc -Wl,--version-script,myscriptfile.map ...
```

Version script thường (nhưng không phải luôn luôn) được xác định bằng phần mở rộng `.map`. Các phần sau mô tả một số cách sử dụng của version script.

### **42.3.1 Kiểm Soát Tầm Nhìn của Symbol Bằng Version Script**

Một cách sử dụng của version script là kiểm soát tầm nhìn của các symbol mà nếu không có thể vô tình được làm global (tức là hiển thị với các ứng dụng liên kết với thư viện). Như một ví dụ đơn giản, giả sử chúng ta đang xây dựng một shared library từ ba source file `vis_comm.c`, `vis_f1.c`, và `vis_f2.c`, lần lượt định nghĩa các hàm `vis_comm()`, `vis_f1()`, và `vis_f2()`. Hàm `vis_comm()` được gọi bởi `vis_f1()` và `vis_f2()`, nhưng không có nghĩa là để sử dụng trực tiếp bởi các ứng dụng liên kết với thư viện. Giả sử chúng ta xây dựng shared library theo cách thông thường:

```
$ gcc -g -c -fPIC -Wall vis_comm.c vis_f1.c vis_f2.c
$ gcc -g -shared -o vis.so vis_comm.o vis_f1.o vis_f2.o
```

Nếu chúng ta sử dụng lệnh `readelf` sau để liệt kê các dynamic symbol được xuất bởi thư viện, chúng ta thấy như sau:

```
$ readelf --syms --use-dynamic vis.so | grep vis_
 30 12: 00000790 59 FUNC GLOBAL DEFAULT 10 vis_f1
 25 13: 000007d0 73 FUNC GLOBAL DEFAULT 10 vis_f2
 27 16: 00000770 20 FUNC GLOBAL DEFAULT 10 vis_comm
```

Shared library này xuất ba symbol: `vis_comm()`, `vis_f1()`, và `vis_f2()`. Tuy nhiên, chúng ta muốn đảm bảo rằng chỉ các symbol `vis_f1()` và `vis_f2()` được xuất bởi thư viện. Chúng ta có thể đạt được kết quả này bằng cách sử dụng version script sau:

```
$ cat vis.map
VER_1 {
 global:
 vis_f1;
 vis_f2;
 local:
 *;
};
```

Định danh `VER_1` là một ví dụ về version tag. Như chúng ta sẽ thấy trong phần thảo luận về symbol versioning trong Mục [42.3.2,](#page-137-0) một version script có thể chứa nhiều version node, mỗi node được nhóm trong dấu ngoặc nhọn (`{}`) và có tiền tố là một version tag duy nhất. Nếu chúng ta chỉ sử dụng version script cho mục đích kiểm soát tầm nhìn của symbol, thì version tag là dư thừa; tuy nhiên, các phiên bản cũ hơn của `ld` yêu cầu nó. Các phiên bản hiện đại của `ld` cho phép bỏ qua version tag; trong trường hợp này, version node được gọi là có anonymous version tag, và không có version node nào khác có thể có mặt trong script.

Trong version node, từ khóa `global` bắt đầu một danh sách ngăn cách bởi dấu chấm phẩy các symbol được làm hiển thị bên ngoài thư viện. Từ khóa `local` bắt đầu một danh sách các symbol bị ẩn khỏi thế giới bên ngoài. Dấu hoa thị (`*`) ở đây minh họa thực tế là chúng ta có thể sử dụng các wildcard pattern trong các đặc tả symbol này. Các ký tự wildcard giống như những ký tự được sử dụng cho việc khớp tên file trong shell—ví dụ: `*` và `?`. (Xem trang manual của `glob(7)` để biết thêm chi tiết.) Trong ví dụ này, việc sử dụng dấu hoa thị cho đặc tả `local` nghĩa là tất cả những gì không được khai báo rõ ràng là `global` sẽ bị ẩn. Nếu chúng ta không nói điều này, thì `vis_comm()` vẫn sẽ hiển thị, vì mặc định là làm cho các global symbol C hiển thị bên ngoài shared library.

Sau đó, chúng ta có thể xây dựng shared library của mình bằng cách sử dụng version script như sau:

```
$ gcc -g -c -fPIC -Wall vis_comm.c vis_f1.c vis_f2.c
$ gcc -g -shared -o vis.so vis_comm.o vis_f1.o vis_f2.o \
 -Wl,--version-script,vis.map
```

Sử dụng `readelf` một lần nữa cho thấy `vis_comm()` không còn hiển thị bên ngoài:

```
$ readelf --syms --use-dynamic vis.so | grep vis_
 25 0: 00000730 73 FUNC GLOBAL DEFAULT 11 vis_f2
 29 16: 000006f0 59 FUNC GLOBAL DEFAULT 11 vis_f1
```

### <span id="page-137-0"></span>**42.3.2 Symbol Versioning**

Symbol versioning cho phép một single shared library cung cấp nhiều phiên bản của cùng một hàm. Mỗi chương trình sử dụng phiên bản của hàm đang có hiệu lực khi chương trình được liên kết (statically) với shared library. Kết quả là, chúng ta có thể thực hiện một thay đổi không tương thích với shared library mà không cần tăng major version number của thư viện. Ở mức độ cực đoan, symbol versioning có thể thay thế cho sơ đồ versioning shared library truyền thống sử dụng major và minor version number. Symbol versioning được sử dụng theo cách này trong glibc 2.1 và các phiên bản sau, để tất cả các phiên bản của glibc từ 2.0 trở đi đều được hỗ trợ trong một major library version duy nhất (`libc.so.6`).

Chúng ta minh họa việc sử dụng symbol versioning với một ví dụ đơn giản. Chúng ta bắt đầu bằng cách tạo phiên bản đầu tiên của một shared library sử dụng version script:

```
$ cat sv_lib_v1.c
#include <stdio.h>
void xyz(void) { printf("v1 xyz\n"); }
$ cat sv_v1.map
VER_1 {
 global: xyz;
 local: *; # Hide all other symbols
};
$ gcc -g -c -fPIC -Wall sv_lib_v1.c
$ gcc -g -shared -o libsv.so sv_lib_v1.o -Wl,--version-script,sv_v1.map
```

Trong một version script, ký tự hash (`#`) bắt đầu một comment.

(Để giữ ví dụ đơn giản, chúng ta tránh sử dụng soname thư viện rõ ràng và major version number của thư viện.)

Ở giai đoạn này, version script của chúng ta, `sv_v1.map`, chỉ phục vụ mục đích kiểm soát tầm nhìn của các symbol trong shared library; `xyz()` được xuất, nhưng tất cả các symbol khác (không có symbol nào trong ví dụ nhỏ này) đều bị ẩn. Tiếp theo, chúng ta tạo một chương trình, `p1`, sử dụng thư viện này:

```
$ cat sv_prog.c
#include <stdlib.h>
int
main(int argc, char *argv[])
{
 void xyz(void);
 xyz();
 exit(EXIT_SUCCESS);
}
$ gcc -g -o p1 sv_prog.c libsv.so
```

Khi chúng ta chạy chương trình này, chúng ta thấy kết quả như mong đợi:

```
$ LD_LIBRARY_PATH=. ./p1
v1 xyz
```

Bây giờ, giả sử chúng ta muốn sửa đổi định nghĩa của `xyz()` trong thư viện của mình, đồng thời vẫn đảm bảo rằng chương trình `p1` tiếp tục sử dụng phiên bản cũ của hàm này. Để làm điều này, chúng ta phải định nghĩa hai phiên bản của `xyz()` trong thư viện của mình:

```
$ cat sv_lib_v2.c
#include <stdio.h>
__asm__(".symver xyz_old,xyz@VER_1");
__asm__(".symver xyz_new,xyz@@VER_2");
void xyz_old(void) { printf("v1 xyz\n"); }
void xyz_new(void) { printf("v2 xyz\n"); }
void pqr(void) { printf("v2 pqr\n"); }
```

Hai phiên bản của `xyz()` được cung cấp bởi các hàm `xyz_old()` và `xyz_new()`. Hàm `xyz_old()` tương ứng với định nghĩa ban đầu của `xyz()`, là định nghĩa mà chương trình `p1` nên tiếp tục sử dụng. Hàm `xyz_new()` cung cấp định nghĩa của `xyz()` để được sử dụng bởi các chương trình liên kết với phiên bản mới của thư viện.

Hai chỉ thị assembler `.symver` là chất kết dính liên kết hai hàm này với các version tag khác nhau trong version script đã sửa đổi (được trình bày sau đây) mà chúng ta sử dụng để tạo phiên bản mới của shared library. Chỉ thị đầu tiên trong số này nói rằng `xyz_old()` là phiên bản triển khai của `xyz()` được sử dụng cho các ứng dụng liên kết với version tag `VER_1` (tức là chương trình `p1` trong ví dụ của chúng ta), và `xyz_new()` là phiên bản triển khai của `xyz()` được sử dụng bởi các ứng dụng liên kết với version tag `VER_2`.

Việc sử dụng `@@` thay vì `@` trong chỉ thị `.symver` thứ hai chỉ ra rằng đây là định nghĩa mặc định của `xyz()` mà các ứng dụng phải bind khi được liên kết statically với shared library này. Đúng một trong số các chỉ thị `.symver` cho một symbol phải được đánh dấu bằng `@@`.

Version script tương ứng cho thư viện đã sửa đổi của chúng ta như sau:

```
$ cat sv_v2.map
VER_1 {
 global: xyz;
 local: *; # Hide all other symbols
};
VER_2 {
 global: pqr;
} VER_1;
```

Version script này cung cấp một version tag mới, `VER_2`, phụ thuộc vào tag `VER_1`. Sự phụ thuộc này được chỉ ra bởi dòng sau:

```
} VER_1;
```

Các phụ thuộc của version tag chỉ ra mối quan hệ giữa các phiên bản thư viện kế tiếp nhau. Về mặt ngữ nghĩa, hiệu ứng duy nhất của các phụ thuộc version tag trên Linux là một version node kế thừa các đặc tả `global` và `local` từ version node mà nó phụ thuộc vào.

Các phụ thuộc có thể được xâu chuỗi, vì vậy chúng ta có thể có một version node khác được gắn tag `VER_3`, phụ thuộc vào `VER_2`, và cứ như vậy tiếp tục.

Tên của version tag tự nó không có ý nghĩa gì. Mối quan hệ của chúng với nhau chỉ được xác định bởi các phụ thuộc version đã được chỉ định, và chúng ta đã chọn tên `VER_1` và `VER_2` chỉ để gợi ý về các mối quan hệ này. Để hỗ trợ bảo trì, thực hành được khuyến nghị là sử dụng các version tag bao gồm tên package và số phiên bản. Ví dụ, glibc sử dụng các version tag với tên như `GLIBC_2.0`, `GLIBC_2.1`, v.v.

Version tag `VER_2` cũng chỉ định rằng hàm mới `pqr()` phải được xuất bởi thư viện và bound với version tag `VER_2`. Nếu chúng ta không khai báo `pqr()` theo cách này, thì đặc tả `local` mà version tag `VER_2` kế thừa từ version tag `VER_1` sẽ làm cho `pqr()` không hiển thị bên ngoài thư viện. Cũng lưu ý rằng nếu chúng ta bỏ hoàn toàn đặc tả `local`, thì các symbol `xyz_old()` và `xyz_new()` cũng sẽ được xuất bởi thư viện (thường không phải là điều chúng ta muốn).

Bây giờ chúng ta xây dựng phiên bản mới của thư viện theo cách thông thường:

```
$ gcc -g -c -fPIC -Wall sv_lib_v2.c
$ gcc -g -shared -o libsv.so sv_lib_v2.o -Wl,--version-script,sv_v2.map
```

Bây giờ chúng ta có thể tạo một chương trình mới, `p2`, sử dụng định nghĩa mới của `xyz()`, trong khi chương trình `p1` sử dụng phiên bản cũ của `xyz()`.

```
$ gcc -g -o p2 sv_prog.c libsv.so
$ LD_LIBRARY_PATH=. ./p2
v2 xyz Uses xyz@VER_2
$ LD_LIBRARY_PATH=. ./p1
v1 xyz Uses xyz@VER_1
```

Các phụ thuộc version tag của một chương trình thực thi được ghi lại tại thời điểm static link. Chúng ta có thể sử dụng `objdump –t` để hiển thị symbol table của mỗi chương trình thực thi, từ đó cho thấy các phụ thuộc version tag khác nhau của mỗi chương trình:

```
$ objdump -t p1 | grep xyz
08048380 F *UND* 0000002e xyz@@VER_1
$ objdump -t p2 | grep xyz
080483a0 F *UND* 0000002e xyz@@VER_2
```

Chúng ta cũng có thể sử dụng `readelf –s` để lấy thông tin tương tự.

Thông tin thêm về symbol versioning có thể được tìm thấy bằng lệnh `info ld scripts version` và tại http://people.redhat.com/drepper/symbol-versioning.

### **42.4 Hàm Khởi Tạo và Hàm Kết Thúc**

Có thể định nghĩa một hoặc nhiều hàm được thực thi tự động khi một shared library được tải và gỡ tải. Điều này cho phép chúng ta thực hiện các hành động khởi tạo và kết thúc khi làm việc với shared library. Các hàm khởi tạo và kết thúc được thực thi bất kể thư viện được tải tự động hay được tải rõ ràng bằng cách sử dụng giao diện `dlopen` (Mục [42.1\)](#page-126-0).

Các hàm khởi tạo và kết thúc được định nghĩa bằng cách sử dụng các thuộc tính `constructor` và `destructor` của gcc. Mỗi hàm sẽ được thực thi khi thư viện được tải phải được định nghĩa như sau:

```
void __attribute__ ((constructor)) some_name_load(void)
{
 /* Initialization code */
}
```

Các hàm gỡ tải được định nghĩa tương tự:

```
void __attribute__ ((destructor)) some_name_unload(void)
{
 /* Finalization code */
}
```

Tên hàm `some_name_load()` và `some_name_unload()` có thể được thay thế bằng bất kỳ tên nào mong muốn.

> Cũng có thể sử dụng các thuộc tính `constructor` và `destructor` của gcc để tạo các hàm khởi tạo và kết thúc trong một chương trình chính.

### **Các hàm \_init() và \_fini()**

Một kỹ thuật cũ hơn để khởi tạo và kết thúc shared library là tạo hai hàm, `_init()` và `_fini()`, như một phần của thư viện. Hàm `void _init(void)` chứa code sẽ được thực thi khi thư viện lần đầu được tải bởi một process. Hàm `void _fini(void)` chứa code sẽ được thực thi khi thư viện được gỡ tải.

Nếu chúng ta tạo các hàm `_init()` và `_fini()`, thì chúng ta phải chỉ định tùy chọn `gcc –nostartfiles` khi xây dựng shared library, để ngăn linker bao gồm các phiên bản mặc định của những hàm này. (Sử dụng các tùy chọn linker `–Wl,–init` và `–Wl,–fini`, chúng ta có thể chọn tên thay thế cho hai hàm này nếu muốn.)

Việc sử dụng `_init()` và `_fini()` hiện được coi là lỗi thời, thay thế bởi các thuộc tính `constructor` và `destructor` của gcc, những thuộc tính này, trong số những ưu điểm khác, cho phép chúng ta định nghĩa nhiều hàm khởi tạo và kết thúc.

# **42.5 Preloading Shared Library**

Để kiểm thử, đôi khi hữu ích khi có thể ghi đè có chọn lọc các hàm (và các symbol khác) mà thông thường sẽ được tìm thấy bởi dynamic linker theo các quy tắc được mô tả trong Mục [41.11](#page-121-1). Để làm điều này, chúng ta có thể định nghĩa biến môi trường `LD_PRELOAD` là một chuỗi bao gồm các tên shared library được phân tách bởi khoảng trắng hoặc dấu hai chấm, phải được tải trước bất kỳ shared library nào khác. Vì các thư viện này được tải đầu tiên, bất kỳ hàm nào chúng định nghĩa sẽ tự động được sử dụng nếu cần thiết bởi chương trình thực thi, do đó ghi đè bất kỳ hàm nào cùng tên mà dynamic linker đáng lẽ phải tìm kiếm. Ví dụ, giả sử chúng ta có một chương trình gọi các hàm `x1()` và `x2()`, được định nghĩa trong thư viện `libdemo` của chúng ta. Khi chúng ta chạy chương trình này, chúng ta thấy đầu ra sau:

```
$ ./prog
Called mod1-x1 DEMO
Called mod2-x2 DEMO
```

(Trong ví dụ này, chúng ta giả sử rằng shared library nằm trong một trong các thư mục tiêu chuẩn, và do đó chúng ta không cần sử dụng biến môi trường `LD_LIBRARY_PATH`.)

Chúng ta có thể ghi đè có chọn lọc hàm `x1()` bằng cách tạo một shared library khác, `libalt.so`, chứa một định nghĩa khác của `x1()`. Preloading thư viện này khi chạy chương trình sẽ cho kết quả sau:

```
$ LD_PRELOAD=libalt.so ./prog
Called mod1-x1 ALT
Called mod2-x2 DEMO
```

Ở đây, chúng ta thấy rằng phiên bản `x1()` được định nghĩa trong `libalt.so` được gọi, nhưng lời gọi đến `x2()`, mà không có định nghĩa nào được cung cấp trong `libalt.so`, dẫn đến việc gọi hàm `x2()` được định nghĩa trong `libdemo.so`.

Biến môi trường `LD_PRELOAD` kiểm soát preloading theo từng process. Ngoài ra, file `/etc/ld.so.preload`, liệt kê các thư viện được phân tách bởi khoảng trắng, có thể được sử dụng để thực hiện cùng một tác vụ trên toàn hệ thống. (Các thư viện được chỉ định bởi `LD_PRELOAD` được tải trước các thư viện được chỉ định trong `/etc/ld.so.preload`.)

Vì lý do bảo mật, các chương trình set-user-ID và set-group-ID bỏ qua `LD_PRELOAD`.

## **42.6 Theo Dõi Dynamic Linker: LD\_DEBUG**

Đôi khi, hữu ích khi theo dõi hoạt động của dynamic linker để biết, ví dụ, nơi nó đang tìm kiếm các thư viện. Chúng ta có thể sử dụng biến môi trường `LD_DEBUG` để làm điều này. Bằng cách đặt biến này thành một (hoặc nhiều hơn) trong một tập hợp các từ khóa tiêu chuẩn, chúng ta có thể nhận được nhiều loại thông tin tracing từ dynamic linker.

Nếu chúng ta gán giá trị `help` cho `LD_DEBUG`, dynamic linker hiển thị thông tin trợ giúp về `LD_DEBUG`, và lệnh được chỉ định sẽ không được thực thi:

#### \$ **LD\_DEBUG=help date**

Các tùy chọn hợp lệ cho biến môi trường `LD_DEBUG` là:

```
 libs display library search paths
 reloc display relocation processing
 files display progress for input file
 symbols display symbol table processing
 bindings display information about symbol binding
 versions display version dependencies
 all all previous options combined
 statistics display relocation statistics
 unused determine unused DSOs
 help display this help message and exit
```

Để chuyển hướng đầu ra debug vào một file thay vì standard output, có thể chỉ định một tên file bằng cách sử dụng biến môi trường `LD_DEBUG_OUTPUT`. Ví dụ sau đây cho thấy một phiên bản rút gọn của đầu ra được cung cấp khi chúng ta yêu cầu tracing thông tin về việc tìm kiếm thư viện:

#### \$ **LD\_DEBUG=libs date**

```
 10687: find library=librt.so.1 [0]; searching
 10687: search cache=/etc/ld.so.cache
 10687: trying file=/lib/librt.so.1
 10687: find library=libc.so.6 [0]; searching
 10687: search cache=/etc/ld.so.cache
 10687: trying file=/lib/libc.so.6
 10687: find library=libpthread.so.0 [0]; searching
 10687: search cache=/etc/ld.so.cache
 10687: trying file=/lib/libpthread.so.0
 10687: calling init: /lib/libpthread.so.0
 10687: calling init: /lib/libc.so.6
 10687: calling init: /lib/librt.so.1
 10687: initialize program: date
 10687: transferring control: date
Tue Dec 28 17:26:56 CEST 2010
 10687: calling fini: date [0]
 10687: calling fini: /lib/librt.so.1 [0]
 10687: calling fini: /lib/libpthread.so.0 [0]
 10687: calling fini: /lib/libc.so.6 [0]
```

Giá trị `10687` được hiển thị ở đầu mỗi dòng là process ID của process đang được tracing. Điều này hữu ích nếu chúng ta đang theo dõi nhiều process (ví dụ: process cha và process con).

Theo mặc định, đầu ra `LD_DEBUG` được ghi vào standard error, nhưng chúng ta có thể chuyển hướng nó bằng cách gán một pathname cho biến môi trường `LD_DEBUG_OUTPUT`.

Nếu muốn, chúng ta có thể gán nhiều tùy chọn cho `LD_DEBUG` bằng cách phân tách chúng bằng dấu phẩy (không được có khoảng trắng). Đầu ra của tùy chọn `symbols` (tracing việc giải quyết symbol bởi dynamic linker) đặc biệt là rất nhiều.

`LD_DEBUG` có hiệu lực cả cho các thư viện được tải ngầm bởi dynamic linker và cho các thư viện được tải động bởi `dlopen()`.

Vì lý do bảo mật, `LD_DEBUG` (từ glibc 2.2.5) bị bỏ qua trong các chương trình set-user-ID và set-group-ID.

### **42.7 Tóm Tắt**

Dynamic linker cung cấp dlopen API, cho phép các chương trình tải thêm shared library một cách rõ ràng lúc chạy. Điều này cho phép các chương trình triển khai chức năng plug-in.

Một khía cạnh quan trọng trong thiết kế shared library là kiểm soát tầm nhìn của symbol, để thư viện chỉ xuất những symbol (hàm và biến) thực sự nên được sử dụng bởi các chương trình liên kết với thư viện. Chúng ta đã xem xét một loạt các kỹ thuật có thể được sử dụng để kiểm soát tầm nhìn của symbol. Trong số các kỹ thuật này có việc sử dụng version script, cung cấp sự kiểm soát chi tiết đối với tầm nhìn của symbol.

Chúng ta cũng đã chỉ ra cách version script có thể được sử dụng để triển khai một sơ đồ cho phép một single shared library xuất nhiều định nghĩa của một symbol để sử dụng bởi các ứng dụng khác nhau liên kết với thư viện. (Mỗi ứng dụng sử dụng định nghĩa hiện hành khi ứng dụng được liên kết statically với thư viện.) Kỹ thuật này cung cấp một phương án thay thế cho cách tiếp cận versioning thư viện truyền thống sử dụng major và minor version number trong tên thực của shared library.

Việc định nghĩa các hàm khởi tạo và kết thúc trong một shared library cho phép chúng ta tự động thực thi code khi thư viện được tải và gỡ tải.

Biến môi trường `LD_PRELOAD` cho phép chúng ta preloading shared library. Sử dụng cơ chế này, chúng ta có thể ghi đè có chọn lọc các hàm và các symbol khác mà dynamic linker thông thường sẽ tìm thấy trong các shared library khác.

Chúng ta có thể gán các giá trị khác nhau cho biến môi trường `LD_DEBUG` để theo dõi hoạt động của dynamic linker.

### **Thông Tin Thêm**

Tham khảo các nguồn thông tin thêm được liệt kê trong Mục [41.14.](#page-123-0)

### **42.8 Bài Tập**

- **42-1.** Viết một chương trình để xác minh rằng nếu một thư viện được đóng bằng `dlclose()`, nó sẽ không được gỡ tải nếu có bất kỳ symbol nào của nó được sử dụng bởi một thư viện khác.
- **42-2.** Thêm một lời gọi `dladdr()` vào chương trình trong [Listing 42-1](#page-132-0) (dynload.c) để truy xuất thông tin về địa chỉ được trả về bởi `dlsym()`. In ra các giá trị của các trường trong structure `Dl_info` được trả về, và xác minh rằng chúng đúng như mong đợi.
