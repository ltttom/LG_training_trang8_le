# Message Queue và Shared Memory trong C++ (Linux) — Kèm ví dụ lỗi

## 1. Message Queue (POSIX mq_* trên Linux)
> Trên Linux, POSIX Message Queue được kernel quản lý và lưu trong `/dev/mqueue`.  
> API chính: `mq_open`, `mq_send`, `mq_receive`, `mq_close`, `mq_unlink`.

### Định nghĩa
Cơ chế IPC (inter-process communication) cho phép tiến trình gửi/nhận thông điệp (buffer có kích thước hữu hạn). POSIX cung cấp API `mq_*` (khác System V `msg*` đời cũ).

### Hoạt động
- Producer `mq_send` → kernel copy dữ liệu từ user-space vào queue.
- Consumer `mq_receive` → kernel copy dữ liệu từ queue ra user-space.
- Queue đầy: block hoặc trả `EAGAIN` nếu non-blocking.
- Queue rỗng: block hoặc trả `EAGAIN` nếu non-blocking.

### Tính chất
- Tách rời dữ liệu: gửi/nhận theo *message*, không cần chia sẻ bộ nhớ.
- Có thứ tự: FIFO hoặc theo **priority**.
- Đệm hữu hạn: khi đầy, gửi sẽ block hoặc trả lỗi (non-blocking).
- An toàn tiến trình: kernel quản lý.
- Đặt tên: bắt đầu bằng `/` (ví dụ: `/myqueue`), lưu dưới `/dev/mqueue`.

### Khi thường dùng
- Kết nối producer/consumer giữa tiến trình.
- Truyền message kích thước vừa phải (từ vài B đến vài KB).
- Cần thứ tự và biên giới thông điệp (Không bị rách gói)

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

### Định nghĩa
- **Shared Memory (SHM)** là cơ chế IPC cho phép nhiều tiến trình **truy cập trực tiếp** cùng một vùng nhớ.
- Trên Linux POSIX SHM được quản lý qua `/dev/shm`.
- Thực chất là file đặc biệt được `mmap` vào bộ nhớ.

### Hoạt động
1. Tạo đối tượng SHM bằng `shm_open`.
2. `ftruncate` để set kích thước.
3. `mmap` vào không gian tiến trình.
4. Tiến trình khác mở cùng tên SHM và `mmap`.
5. Cả hai truy cập chung dữ liệu.
6. **Không có đồng bộ sẵn** → cần mutex/semaphore.

### Tính chất
- Nhanh (trực tiếp RAM).
- Không có đồng bộ sẵn → tự mutex/semaphore.
- Đặt tên: `/ten_shm`.
- Thiết kế layout dữ liệu cẩn thận.

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

## 3. Bảng so sánh

| Tiêu chí              | Message Queue                               | Shared Memory                                  |
|-----------------------|---------------------------------------------|------------------------------------------------|
| **Mô hình dữ liệu**   | Message nguyên khối                         | Vùng nhớ liên tục                              |
| **Đồng bộ**           | Kernel đảm bảo                              | Tự lập trình                                   |
| **Hiệu năng**         | Vừa, có copy                                | Rất cao, truy cập trực tiếp                    |
| **Dung lượng**        | Giới hạn bởi mq_maxmsg, mq_msgsize           | Giới hạn bởi RAM                               |
| **Độ phức tạp**       | Dễ                                           | Khó hơn                                        |
| **Ưu tiên message**   | Có                                           | Không                                          |
| **Vị trí**            | /dev/mqueue/<name>                          | /dev/shm/<name>                                |

## 4. Ví dụ cùng bài toán Producer–Consumer

### 4.1. Message Queue

**Producer (mq_producer.cpp)**
```cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cstring>
#include <unistd.h>

int main() {
    const char* qname = "/mq_demo";
    mq_attr attr{};
    attr.mq_maxmsg = 10;
    attr.mq_msgsize = 64;
    mqd_t mq = mq_open(qname, O_CREAT | O_WRONLY, 0644, &attr);
    if (mq == (mqd_t)-1) { perror("mq_open"); return 1; }

    char msg[64];
    for (int i = 0; i < 5; i++) {
        snprintf(msg, sizeof(msg), "Message %d", i);
        mq_send(mq, msg, sizeof(msg), 0);
        printf("Sent: %s\n", msg);
        usleep(100000);
    }
    mq_close(mq);
    mq_unlink(qname);
}
```

**Consumer (mq_consumer.cpp)**
```cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cstring>

int main() {
    const char* qname = "/mq_demo";
    mqd_t mq = mq_open(qname, O_RDONLY);
    if (mq == (mqd_t)-1) { perror("mq_open"); return 1; }

    char msg[64];
    unsigned prio;
    for (int i = 0; i < 5; i++) {
        mq_receive(mq, msg, sizeof(msg), &prio);
        printf("Received: %s\n", msg);
    }
    mq_close(mq);
}
```

---

### 4.2. Shared Memory

**Producer (shm_producer.cpp)**
```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <semaphore.h>

struct SharedData {
    char msg[64];
    bool ready;
};

int main() {
    const char* shm_name = "/shm_demo";
    const char* sem_name = "/sem_demo";

    int fd = shm_open(shm_name, O_CREAT | O_RDWR, 0644);
    ftruncate(fd, sizeof(SharedData));
    SharedData* data = (SharedData*)mmap(NULL, sizeof(SharedData),
                                         PROT_READ | PROT_WRITE,
                                         MAP_SHARED, fd, 0);
    sem_t* sem = sem_open(sem_name, O_CREAT, 0644, 0);

    for (int i = 0; i < 5; i++) {
        snprintf(data->msg, sizeof(data->msg), "Message %d", i);
        data->ready = true;
        sem_post(sem); // thông báo cho consumer
        printf("Wrote: %s\n", data->msg);
        usleep(100000);
    }

    munmap(data, sizeof(SharedData));
    close(fd);
    shm_unlink(shm_name);
    sem_close(sem);
    sem_unlink(sem_name);
}
```

**Consumer (shm_consumer.cpp)**
```cpp
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstdio>
#include <semaphore.h>

struct SharedData {
    char msg[64];
    bool ready;
};

int main() {
    const char* shm_name = "/shm_demo";
    const char* sem_name = "/sem_demo";

    int fd = shm_open(shm_name, O_RDWR, 0644);
    SharedData* data = (SharedData*)mmap(NULL, sizeof(SharedData),
                                         PROT_READ | PROT_WRITE,
                                         MAP_SHARED, fd, 0);
    sem_t* sem = sem_open(sem_name, 0);

    for (int i = 0; i < 5; i++) {
        sem_wait(sem);
        if (data->ready) {
            printf("Read: %s\n", data->msg);
            data->ready = false;
        }
    }

    munmap(data, sizeof(SharedData));
    close(fd);
}
```

## 5. Checklist chống lỗi

- MQ: kiểm tra `mq_msgsize` trước khi gửi; dọn `mq_unlink`.
- SHM: luôn `ftruncate` trước `mmap`; đồng bộ hóa; tránh false-sharing.
