# Phân tích Shared Memory và Message Queue

- Cả Shared Memory (Bộ nhớ chia sẻ) và Message Queue (Hàng đợi thông điệp) đều là các cơ chế giao tiếp liên tiến trình (Inter-Process Communication - IPC) trong hệ điều hành.
- Chúng được sử dụng để trao đổi dữ liệu giữa các tiến trình. Tuy nhiên, chúng có cách hoạt động, ưu điểm và nhược điểm khác nhau.

## 1. Message Queue (POSIX mq_* trên Linux)
> Trên Linux, POSIX Message Queue được kernel quản lý và lưu trong `/dev/mqueue`.  
> API chính: `mq_open`, `mq_send`, `mq_receive`, `mq_close`, `mq_unlink`.

## Khái niệm
- Message Queue là một cơ chế IPC cho phép các tiến trình gửi và nhận thông điệp thông qua một hàng đợi được quản lý bởi hệ điều hành.
- Dữ liệu được truyền dưới dạng các thông điệp có cấu trúc.

## Cách hoạt động
- Một tiến trình tạo hàng đợi thông điệp bằng cách sử dụng msgget.
- Các tiến trình có thể gửi thông điệp vào hàng đợi bằng msgsnd.
- Các tiến trình khác có thể nhận thông điệp từ hàng đợi bằng msgrcv.
- Hàng đợi có thể được hủy bằng msgctl.

## Ưu điểm
- Đồng bộ hóa tự động: Hệ điều hành quản lý việc gửi và nhận thông điệp, giúp tránh xung đột.
- An toàn hơn: Dữ liệu được truyền qua hàng đợi, không bị truy cập trực tiếp bởi các tiến trình khác.
- Dễ sử dụng: Không cần quản lý đồng bộ hóa thủ công.

## Nhược điểm
- Hiệu suất thấp hơn: Dữ liệu phải được sao chép từ tiến trình gửi vào hàng đợi và từ hàng đợi đến tiến trình nhận.
- Dung lượng giới hạn: Kích thước hàng đợi và thông điệp thường bị giới hạn bởi hệ điều hành.
- Phức tạp hơn: Cần xử lý các trường hợp hàng đợi đầy hoặc rỗng.

```cpp
#include <sys/ipc.h>
#include <sys/msg.h>
#include <stdio.h>
#include <string.h>

struct message {
    long msg_type;
    char msg_text[100];
};

int main() {
    key_t key = 1234; // Khóa để xác định hàng đợi
    int msgid = msgget(key, 0666 | IPC_CREAT); // Tạo hàng đợi thông điệp

    struct message msg;
    msg.msg_type = 1; // Loại thông điệp
    strcpy(msg.msg_text, "Hello, Message Queue!"); // Ghi dữ liệu vào thông điệp

    msgsnd(msgid, &msg, sizeof(msg.msg_text), 0); // Gửi thông điệp
    printf("Message sent: %s\n", msg.msg_text);

    msgrcv(msgid, &msg, sizeof(msg.msg_text), 1, 0); // Nhận thông điệp
    printf("Message received: %s\n", msg.msg_text);

    msgctl(msgid, IPC_RMID, NULL); // Hủy hàng đợi thông điệp
    return 0;
}
```

### Lỗi hay gặp và ví dụ

#### 1.1 `EAGAIN` khi hàng đợi đầy (non-blocking)

**Tình huống**: Hàng đợi đạt số lượng message tối đa (`mq_maxmsg`), gọi `mq_send` với `O_NONBLOCK`.

**Linux xử lý**:  
- Không `O_NONBLOCK`: tiến trình block cho tới khi có chỗ trống.
- Có `O_NONBLOCK`: trả `-1`, `errno=EAGAIN`.

**Nguyên nhân**:
- Producer gửi nhanh hơn consumer đọc.
- `mq_maxmsg` quá nhỏ.

**Khắc phục**:
- Tăng `mq_maxmsg` (giới hạn `/proc/sys/fs/mqueue/msg_max`).
- Xử lý `EAGAIN` bằng retry/backoff.

```cpp
// mq_full_eagain.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cstring>
#include <cerrno>
int main() {
    const char* q = "/demo_mq_eagain";
    mq_attr a{}; a.mq_maxmsg = 2; a.mq_msgsize = 32; a.mq_flags = O_NONBLOCK;
    mqd_t m = mq_open(q, O_CREAT | O_WRONLY | O_NONBLOCK, 0644, &a);
    char buf[32] = "x";
    mq_send(m, buf, sizeof(buf), 0);
    mq_send(m, buf, sizeof(buf), 0);
    if (mq_send(m, buf, sizeof(buf), 0)==-1) {
        printf("mq_send lỗi: %s\n", strerror(errno));
    }
    mq_close(m); mq_unlink(q);
}
```

#### 1.2 `EAGAIN` khi hàng đợi rỗng (non-blocking receive)

**Tình huống**: Queue trống, gọi `mq_receive` với `O_NONBLOCK`.

**Linux xử lý**:  
- Không `O_NONBLOCK`: block tới khi có message.
- Có `O_NONBLOCK`: trả `-1`, `errno=EAGAIN`.

**Nguyên nhân**:
- Consumer đọc nhanh hơn producer.

**Khắc phục**:
- Dùng blocking hoặc poll/select để chờ.

```cpp
// mq_empty_eagain.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cerrno>
#include <cstring>
int main(){
  const char* q="/demo_mq_empty";
  mq_attr a{}; a.mq_maxmsg=2; a.mq_msgsize=32;
  mqd_t w = mq_open(q, O_CREAT|O_WRONLY, 0644, &a);
  mqd_t r = mq_open(q, O_RDONLY|O_NONBLOCK);
  char buf[32]; unsigned prio=0;
  if (mq_receive(r, buf, sizeof(buf), &prio)==-1){
    printf("mq_receive lỗi: %s\n", strerror(errno));
  }
  mq_close(r); mq_close(w); mq_unlink(q);
}
```

#### 1.3 `EMSGSIZE` khi gửi message vượt kích thước


**Tình huống**: Gửi message lớn hơn `mq_msgsize`.

**Linux xử lý**:  
- Kernel kiểm tra → nếu vượt → `errno=EMSGSIZE`.

**Nguyên nhân**:
- Không kiểm tra giới hạn trước khi gửi.

**Khắc phục**:
- Dùng `mq_getattr` lấy `mq_msgsize`.
- Chia nhỏ dữ liệu.

```cpp
// mq_emsgsize.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cstring>
#include <cerrno>
int main(){
  const char* q="/demo_mq_sz";
  mq_attr a{}; a.mq_maxmsg=4; a.mq_msgsize=16;
  mqd_t m = mq_open(q, O_CREAT|O_WRONLY, 0644, &a);
  char big[64]; memset(big,'A',sizeof(big));
  if (mq_send(m, big, sizeof(big), 0)==-1){
    printf("mq_send lỗi: %s\n", strerror(errno));
  }
  mq_close(m); mq_unlink(q);
}
```

#### 1.4 `ENOENT` khi mở queue không tồn tại

**Tình huống**: `mq_open` queue chưa tồn tại, không dùng `O_CREAT`.

**Linux xử lý**:  
- Không tìm thấy trong `/dev/mqueue` → `errno=ENOENT`.

**Nguyên nhân**:
- Mở queue trước khi tiến trình tạo queue chạy.

**Khắc phục**:
- Tạo với `O_CREAT` hoặc đảm bảo tiến trình tạo chạy trước.

```cpp
mqd_t m = mq_open("/khong_ton_tai", O_RDONLY);
if (m==(mqd_t)-1) { perror("mq_open"); }
```

#### 1.5 `EBADF` khi dùng sai mode

**Tình huống**:
- Mở `O_WRONLY` nhưng `mq_receive`.
- Mở `O_RDONLY` nhưng `mq_send`.

**Linux xử lý**:
- Kiểm tra quyền mở → sai → `errno=EBADF`.

**Nguyên nhân**:
- Lập trình nhầm mode.

**Khắc phục**:
- Mở `O_RDWR` nếu cần vừa send vừa receive.


```cpp
// mq_ebadf.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cerrno>
#include <cstring>
int main(){
  const char* q="/demo_mq_badf";
  mq_attr a{}; a.mq_maxmsg=2; a.mq_msgsize=8;
  mqd_t w = mq_open(q, O_CREAT|O_WRONLY, 0600, &a);
  mqd_t r = mq_open(q, O_RDONLY);
  char msg[8]="hi"; unsigned pr=0;
  if (mq_receive(w, msg, sizeof(msg), &pr)==-1)
    printf("mq_receive WRONLY: %s\n", strerror(errno));
  if (mq_send(r, msg, sizeof(msg), 0)==-1)
    printf("mq_send RDONLY: %s\n", strerror(errno));
  mq_close(r); mq_close(w); mq_unlink(q);
}
```

#### 1.6 `EINVAL` / `ENOSPC` khi vượt giới hạn hệ thống

**Tình huống**: `mq_maxmsg` hoặc `mq_msgsize` vượt cấu hình kernel.

**Linux xử lý**:
- So với `/proc/sys/fs/mqueue/msg_max` và `/proc/sys/fs/mqueue/msgsize_max`.
- Nếu vượt → `errno=EINVAL` hoặc `ENOSPC`.

**Nguyên nhân**:
- Đặt giá trị quá lớn.

**Khắc phục**:
- Tăng giới hạn kernel:
  ```bash
  sudo sh -c 'echo 128 > /proc/sys/fs/mqueue/msg_max'
  sudo sh -c 'echo 65536 > /proc/sys/fs/mqueue/msgsize_max'

```cpp
mq_attr a{}; a.mq_maxmsg=1000000; a.mq_msgsize=1024*1024;
mqd_t m = mq_open("/demo_mq_big", O_CREAT|O_WRONLY, 0644, &a);
if (m==(mqd_t)-1) perror("mq_open");
```

---

## 2. Shared Memory (`shm_open` + `mmap`)
> Trên Linux, POSIX shared memory được quản lý qua `/dev/shm` (tmpfs).  
> API chính: `shm_open`, `ftruncate`, `mmap`, `munmap`, `shm_unlink`.

## Khái niệm
- Shared Memory là một vùng bộ nhớ được chia sẻ giữa nhiều tiến trình.
- Các tiến trình có thể đọc/ghi trực tiếp vào vùng bộ nhớ này để trao đổi dữ liệu.
- Đây là một trong những cơ chế IPC nhanh nhất vì không cần thông qua hệ điều hành để truyền dữ liệu.

## Cách hoạt động

- Một tiến trình tạo ra một vùng bộ nhớ chia sẻ bằng cách sử dụng các API như shmget (trong Linux).
- Các tiến trình khác có thể gắn (attach)vào vùng bộ nhớ này bằng cách sử dụng shmat.
- Các tiến trình có thể đọc/ghi dữ liệu trực tiếp vào vùng bộ nhớ.
- Khi không còn cần thiết, vùng bộ nhớ có thể được hủy bằng shmctl.

## Ưu điểm
- Hiệu suất cao: Dữ liệu được trao đổi trực tiếp qua bộ nhớ mà không cần sao chép.
- Dung lượng lớn: Có thể chia sẻ một lượng lớn dữ liệu giữa các tiến trình.
- Đơn giản: Dễ dàng truy cập dữ liệu bằng cách sử dụng con trỏ.

## Nhược điểm
- Đồng bộ hóa: Các tiến trình phải tự quản lý đồng bộ hóa (ví dụ: sử dụng semaphore) để tránh xung đột khi đọc/ghi dữ liệu.
- Bảo mật: Dữ liệu trong bộ nhớ chia sẻ có thể bị truy cập bởi các tiến trình không mong muốn nếu không được bảo vệ đúng cách.
- Khó quản lý: Việc tạo, gắn và hủy vùng bộ nhớ cần được quản lý cẩn thận để tránh rò rỉ tài nguyên.

```cpp
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdio.h>
#include <string.h>

int main() {
    key_t key = 1234; // Khóa để xác định vùng bộ nhớ
    int shmid = shmget(key, 1024, 0666 | IPC_CREAT); // Tạo vùng bộ nhớ chia sẻ
    char *data = (char *)shmat(shmid, NULL, 0); // Gắn vào vùng bộ nhớ

    strcpy(data, "Hello, Shared Memory!"); // Ghi dữ liệu vào bộ nhớ
    printf("Data written: %s\n", data);

    shmdt(data); // Tách vùng bộ nhớ
    shmctl(shmid, IPC_RMID, NULL); // Hủy vùng bộ nhớ
    return 0;
}
```

### Lỗi hay gặp và ví dụ

#### 2.1 Quên `ftruncate` trước `mmap` → SIGBUS

**Tình huống**: Tạo SHM nhưng chưa `ftruncate` set size, rồi `mmap` và truy cập.

**Linux xử lý**:
- Map size=0 hợp lệ, nhưng truy cập vượt page hợp lệ → SIGBUS.

**Nguyên nhân**:
- Không gọi `ftruncate` sau khi `shm_open`.

**Khắc phục**:
- Luôn `ftruncate(fd, size)` trước khi `mmap`.

```cpp
int fd = shm_open("/demo_shm_nf", O_CREAT|O_RDWR, 0644);
// Sai: thiếu ftruncate
ftruncate(fd, 4096); // Sửa
```

#### 2.2 `EACCES` khi mở không đủ quyền

**Tình huống**: SHM file tạo với mode không cho ghi, mở với `O_RDWR`.

**Linux xử lý**:
- Check quyền file trong `/dev/shm`.

**Nguyên nhân**:
- Cấp quyền sai khi tạo.

**Khắc phục**:
- Dùng mode đủ (`0644` hoặc `0666` nếu chia sẻ user khác).

```cpp
int fd = shm_open("/demo_shm_ro", O_CREAT|O_RDONLY, 0400);
int fdw = shm_open("/demo_shm_ro", O_RDWR, 0); // EACCES
```

#### 2.3 `EACCES` khi mmap quyền không khớp

**Tình huống**: `shm_open` RDONLY nhưng `mmap` PROT_WRITE.

**Linux xử lý**:
- Kiểm tra quyền fd trước khi map → sai → `errno=EACCES`.

**Nguyên nhân**:
- Yêu cầu ghi khi fd chỉ có read.

**Khắc phục**:
- Mở `O_RDWR` nếu cần PROT_WRITE.

```cpp
int fd = shm_open("/demo_shm_map", O_CREAT|O_RDONLY, 0644);
void* p = mmap(nullptr, 4096, PROT_READ|PROT_WRITE, MAP_SHARED, fd, 0); // EACCES
```

#### 2.4 Race condition khi không đồng bộ

**Tình huống**: Nhiều tiến trình truy cập chung mà không khóa.

**Linux xử lý**:
- Không tự đồng bộ, dữ liệu có thể hỏng.

**Nguyên nhân**:
- Không semaphore/mutex.

**Khắc phục**:
- Dùng `pthread_mutex` với `PTHREAD_PROCESS_SHARED` hoặc `sem_*`.

```cpp
struct Shared { int len; char buf[256]; };
// Writer và Reader cùng truy cập mà không khóa → dữ liệu rách
```

#### 2.5 Deadlock do thứ tự khóa sai

**Tình huống**: Hai tiến trình khóa resource theo thứ tự khác nhau.

**Linux xử lý**:
- Không timeout (với semaphore thường), treo vĩnh viễn.

**Nguyên nhân**:
- Không quy ước thứ tự khóa.

**Khắc phục**:
- Luôn khóa theo thứ tự cố định.

```
// Process A: lock(sem1); lock(sem2);
// Process B: lock(sem2); lock(sem1);
```

#### 2.6 Lỗi `sem_open`
- `ENOENT`: mở semaphore chưa tồn tại, không O_CREAT.
- `EEXIST`: tạo với O_EXCL khi đã tồn tại.

**Linux xử lý**:
- Semaphore lưu trong `/dev/shm/sem.<name>`.

#### 2.7 Dangling objects

**Tình huống**: Quên `shm_unlink` hoặc `sem_unlink`.

**Linux xử lý**:
- Object tồn tại mãi trong `/dev/shm`.

**Nguyên nhân**:
- Tiến trình chết mà không unlink.

**Khắc phục**:
- Gọi unlink khi tiến trình cuối kết thúc.
- Hoặc xóa thủ công:
  ```bash
  rm /dev/shm/<tên>
  ```

# So sánh Shared Memory và Message Queue
|Tiêu chí |Shared Memory | Message Queue|
|-|-|-|
|Cách hoạt động | Chia sẻ một vùng bộ nhớ giữa các tiến trình. | Gửi và nhận thông điệp qua hàng đợi.|
|Hiệu suất | Nhanh hơn (truy cập trực tiếp vào bộ nhớ). | Chậm hơn (cần sao chép dữ liệu).|
|Đồng bộ hóa | Phải tự quản lý đồng bộ hóa (ví dụ: semaphore). | Hệ điều hành tự quản lý đồng bộ hóa. |
|Dung lượng | Có thể chia sẻ lượng lớn dữ liệu. | Bị giới hạn bởi kích thước hàng đợi. |
|Độ phức tạp | Phức tạp hơn (quản lý đồng bộ và bộ nhớ). | Đơn giản hơn (hệ điều hành quản lý). |
|An toàn | Ít an toàn hơn (dữ liệu có thể bị truy cập trực tiếp). | An toàn hơn (dữ liệu được quản lý bởi hệ điều hành). |
|Ứng dụng | Thích hợp cho trao đổi dữ liệu lớn và nhanh. | Thích hợp cho trao đổi thông điệp nhỏ, có cấu trúc.|

# Khi nào sử dụng?

## Shared Memory:

- Khi cần trao đổi dữ liệu lớn và hiệu suất cao.
- Khi các tiến trình cần truy cập đồng thời vào cùng một dữ liệu.
- Khi bạn có thể quản lý đồng bộ hóa một cách cẩn thận.

## Message Queue:

- Khi cần trao đổi thông điệp nhỏ, có cấu trúc.
- Khi cần đảm bảo đồng bộ hóa tự động giữa các tiến trình.
- Khi muốn tránh rủi ro truy cập trái phép vào dữ liệu.
