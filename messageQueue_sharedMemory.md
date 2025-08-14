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
    const char* q = "/demo_mq_eagain";       // Tên queue POSIX phải bắt đầu bằng '/'
    mq_attr a{}; 
    a.mq_maxmsg = 2;                         // Giới hạn tối đa số message trong queue (rất nhỏ để dễ tái hiện)
    a.mq_msgsize = 32;                       // Kích thước tối đa mỗi message (byte)
    a.mq_flags = O_NONBLOCK;                 // Đặt cờ non-block cho *mô tả* queue khi tạo (không bắt buộc)

    // Mở queue để *ghi* và *non-blocking*. Nếu chưa tồn tại thì tạo (O_CREAT).
    mqd_t m = mq_open(q, O_CREAT | O_WRONLY | O_NONBLOCK, 0644, &a);
    if (m == (mqd_t)-1) { perror("mq_open"); return 1; }

    char buf[32] = "x";                      // Payload nhỏ hơn mq_msgsize

    mq_send(m, buf, sizeof(buf), 0);         // 1) Gửi OK
    mq_send(m, buf, sizeof(buf), 0);         // 2) Gửi OK → queue đạt mq_maxmsg (đầy)

    // 3) Gửi *thêm* trong chế độ non-block → không chờ → trả -1 với errno=EAGAIN
    if (mq_send(m, buf, sizeof(buf), 0) == -1) {
        printf("mq_send lỗi (khi đầy): %s\n", strerror(errno)); // Kỳ vọng: EAGAIN
    }

    mq_close(m); 
    mq_unlink(q);                            // Dọn dẹp queue trong /dev/mqueue
} 

#### 1.2 `EAGAIN` khi hàng đợi rỗng (non-blocking receive)

**Tình huống**: Queue trống, gọi `mq_receive` với `O_NONBLOCK`.

**Linux xử lý**:  
- Không `O_NONBLOCK`: block tới khi có message.
- Có `O_NONBLOCK`: trả `-1`, `errno=EAGAIN`.

**Nguyên nhân**:
- Consumer đọc nhanh hơn producer.

**Khắc phục**:
- Dùng blocking hoặc poll/select để chờ.

// mq_empty_eagain.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cerrno>
#include <cstring>

int main(){
  const char* q = "/demo_mq_empty";

  mq_attr a{}; 
  a.mq_maxmsg = 2; 
  a.mq_msgsize = 32;

  // Tạo queue trước (writer mở để giữ queue tồn tại)
  mqd_t w = mq_open(q, O_CREAT | O_WRONLY, 0644, &a);
  if (w == (mqd_t)-1) { perror("mq_open writer"); return 1; }

  // Mở *reader* ở chế độ non-block
  mqd_t r = mq_open(q, O_RDONLY | O_NONBLOCK);
  if (r == (mqd_t)-1) { perror("mq_open reader"); return 1; }

  char buf[32]; 
  unsigned prio = 0;

  // Queue đang *trống* và receive non-block → trả -1, errno=EAGAIN
  if (mq_receive(r, buf, sizeof(buf), &prio) == -1){
    printf("mq_receive lỗi (khi rỗng): %s\n", strerror(errno)); // Kỳ vọng: EAGAIN
  }

  mq_close(r); 
  mq_close(w); 
  mq_unlink(q); // Dọn dẹp
}

#### 1.3 `EMSGSIZE` khi gửi message vượt kích thước


**Tình huống**: Gửi message lớn hơn `mq_msgsize`.

**Linux xử lý**:  
- Kernel kiểm tra → nếu vượt → `errno=EMSGSIZE`.

**Nguyên nhân**:
- Không kiểm tra giới hạn trước khi gửi.

**Khắc phục**:
- Dùng `mq_getattr` lấy `mq_msgsize`.
- Chia nhỏ dữ liệu.

// mq_emsgsize.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cstring>
#include <cerrno>

int main(){
  const char* q = "/demo_mq_sz";

  mq_attr a{}; 
  a.mq_maxmsg  = 4;     // OK
  a.mq_msgsize = 16;    // Hạn 16 byte/mỗi message

  mqd_t m = mq_open(q, O_CREAT | O_WRONLY, 0644, &a);
  if (m == (mqd_t)-1) { perror("mq_open"); return 1; }

  char big[64]; 
  memset(big, 'A', sizeof(big));             // Chuẩn bị payload 64 byte (> 16)

  // Gửi vượt kích thước → trả -1 với errno=EMSGSIZE
  if (mq_send(m, big, sizeof(big), 0) == -1){
    printf("mq_send lỗi (vượt mq_msgsize): %s\n", strerror(errno)); // Kỳ vọng: EMSGSIZE
  }

  mq_close(m); 
  mq_unlink(q);
}

#### 1.4 `ENOENT` khi mở queue không tồn tại

**Tình huống**: `mq_open` queue chưa tồn tại, không dùng `O_CREAT`.

**Linux xử lý**:  
- Không tìm thấy trong `/dev/mqueue` → `errno=ENOENT`.

**Nguyên nhân**:
- Mở queue trước khi tiến trình tạo queue chạy.

**Khắc phục**:
- Tạo với `O_CREAT` hoặc đảm bảo tiến trình tạo chạy trước.

#include <mqueue.h>
#include <cstdio>

int main() {
  // Không dùng O_CREAT → nếu queue chưa tồn tại trong /dev/mqueue → ENOENT
  mqd_t m = mq_open("/khong_ton_tai", O_RDONLY);
  if (m == (mqd_t)-1) { 
    perror("mq_open"); // Kỳ vọng: ENOENT (No such file or directory)
  }
}

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


// mq_ebadf.cpp
#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cerrno>
#include <cstring>

int main(){
  const char* q = "/demo_mq_badf";

  mq_attr a{}; 
  a.mq_maxmsg = 2; 
  a.mq_msgsize = 8;

  // Writer: chỉ mở WRONLY
  mqd_t w = mq_open(q, O_CREAT | O_WRONLY, 0600, &a);
  if (w == (mqd_t)-1) { perror("mq_open w"); return 1; }

  // Reader: chỉ mở RDONLY
  mqd_t r = mq_open(q, O_RDONLY);
  if (r == (mqd_t)-1) { perror("mq_open r"); return 1; }

  char msg[8] = "hi"; 
  unsigned pr = 0;

  // Dùng descriptor WRONLY để *nhận* → sai chế độ → EBADF
  if (mq_receive(w, msg, sizeof(msg), &pr) == -1)
    printf("mq_receive WRONLY: %s\n", strerror(errno)); // Kỳ vọng: EBADF

  // Dùng descriptor RDONLY để *gửi* → sai chế độ → EBADF
  if (mq_send(r, msg, sizeof(msg), 0) == -1)
    printf("mq_send RDONLY: %s\n", strerror(errno)); // Kỳ vọng: EBADF

  mq_close(r); 
  mq_close(w); 
  mq_unlink(q);
}

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

// Ví dụ minh họa: cố tạo queue với tham số quá lớn so với kernel
// Tham khảo giới hạn hiện tại qua:
//   /proc/sys/fs/mqueue/msg_max        (số message tối đa/queue)
//   /proc/sys/fs/mqueue/msgsize_max    (kích thước message tối đa)

#include <mqueue.h>
#include <fcntl.h>
#include <cstdio>
#include <cerrno>

int main(){
  mq_attr a{}; 
  a.mq_maxmsg  = 1'000'000;             // Quá lớn → có thể EINVAL/ENOSPC
  a.mq_msgsize = 1024 * 1024;           // 1 MiB, có thể vượt msgsize_max

  mqd_t m = mq_open("/demo_mq_big", O_CREAT | O_WRONLY, 0644, &a);
  if (m == (mqd_t)-1) {
    perror("mq_open");                  // Kỳ vọng: EINVAL (tham số sai) hoặc ENOSPC (không cấp phát được)
    return 1;
  }
  mq_close(m); 
  mq_unlink("/demo_mq_big");
  return 0;
}

/*
# Cách khắc phục (tạm thời, cần quyền root):
sudo sh -c 'echo 128   > /proc/sys/fs/mqueue/msg_max'       # tăng số message tối đa
sudo sh -c 'echo 65536 > /proc/sys/fs/mqueue/msgsize_max'   # tăng kích cỡ message tối đa
*/

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

// demo_shm_ftruncate.cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <cerrno>

int main() {
    const char* name = "/demo_shm_nf";

    // Tạo SHM object (file ảo trong /dev/shm)
    int fd = shm_open(name, O_CREAT | O_RDWR, 0644);
    if (fd == -1) { perror("shm_open"); return 1; }

    // SAI: nếu không ftruncate để cấp kích thước trước khi mmap,
    // việc ghi vào vùng map có thể gây SIGBUS (do backing size = 0).
    // ĐÚNG: luôn ftruncate trước.
    if (ftruncate(fd, 4096) == -1) { perror("ftruncate"); return 1; }

    void* p = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
    if (p == MAP_FAILED) { perror("mmap"); return 1; }

    strcpy((char*)p, "Hello SHM");  // An toàn sau khi đã ftruncate

    munmap(p, 4096);
    close(fd);
    shm_unlink(name);                // Dọn dẹp object trong /dev/shm
}

#### 2.2 `EACCES` khi mở không đủ quyền

**Tình huống**: SHM file tạo với mode không cho ghi, mở với `O_RDWR`.

**Linux xử lý**:
- Check quyền file trong `/dev/shm`.

**Nguyên nhân**:
- Cấp quyền sai khi tạo.

**Khắc phục**:
- Dùng mode đủ (`0644` hoặc `0666` nếu chia sẻ user khác).

#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <cstdio>

int main(){
  const char* name = "/demo_shm_ro";

  // Tạo với quyền *chỉ đọc* (0400) rồi thử mở O_RDWR → EACCES
  int fd_rocreate = shm_open(name, O_CREAT | O_RDONLY, 0400);
  if (fd_rocreate == -1) { perror("shm_open create ro"); return 1; }
  close(fd_rocreate);

  // Mở lại với O_RDWR nhưng mode trước đây không cho ghi → EACCES
  int fd = shm_open(name, O_RDWR, 0);
  if (fd == -1) { perror("shm_open O_RDWR"); } // Kỳ vọng: EACCES

  shm_unlink(name);
}

#### 2.3 `EACCES` khi mmap quyền không khớp

**Tình huống**: `shm_open` RDONLY nhưng `mmap` PROT_WRITE.

**Linux xử lý**:
- Kiểm tra quyền fd trước khi map → sai → `errno=EACCES`.

**Nguyên nhân**:
- Yêu cầu ghi khi fd chỉ có read.

**Khắc phục**:
- Mở `O_RDWR` nếu cần PROT_WRITE.

#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <cstdio>

int main(){
  const char* name = "/demo_shm_map";

  // Mở RDONLY
  int fd = shm_open(name, O_CREAT | O_RDONLY, 0644);
  if (fd == -1) { perror("shm_open"); return 1; }

  // Cố mmap với PROT_WRITE trong khi fd là RDONLY → EACCES
  void* p = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
  if (p == MAP_FAILED) { perror("mmap"); } // Kỳ vọng: EACCES

  close(fd);
  shm_unlink(name);
}

#### 2.4 Race condition khi không đồng bộ

**Tình huống**: Nhiều tiến trình truy cập chung mà không khóa.

**Linux xử lý**:
- Không tự đồng bộ, dữ liệu có thể hỏng.

**Nguyên nhân**:
- Không semaphore/mutex.

**Khắc phục**:
- Dùng `pthread_mutex` với `PTHREAD_PROCESS_SHARED` hoặc `sem_*`.

// Minh hoạ race: 2 tiến trình/2 thread ghi/đọc cùng struct không khoá → dữ liệu "rách"
#include <pthread.h>
#include <atomic>
#include <cstdio>
#include <cstring>

struct Shared { 
  int len; 
  char buf[256]; 
};

int main(){
  // Ví dụ *ý tưởng* (không đầy đủ shared memory setup để ngắn gọn):
  Shared s{};                     // Giả sử s nằm trong vùng SHM đã map chung

  // Thread A (writer) - KHÔNG khoá:
  // s.len = 256; memcpy(s.buf, big_data, 256);
  // Trong lúc copy, Thread B (reader) đọc s.len & s.buf → có thể thấy len mới nhưng buf còn dở

  // Cách khắc phục: dùng mutex/semaphore liên tiến trình:
  // - pthread_mutex với thuộc tính PTHREAD_PROCESS_SHARED
  // - hoặc sem_* (sem_open, sem_wait, sem_post)
  // (Xem ví dụ khoá thực sự ở phần 2.5)
  (void)s;
  return 0;
}

#### 2.5 Deadlock do thứ tự khóa sai
/*
Process A: lock(sem1); lock(sem2); ... unlock(sem2); unlock(sem1);
Process B: lock(sem2); lock(sem1); ... unlock(sem1); unlock(sem2);
→ Mắc kẹt vĩnh viễn nếu cả hai giữ một sem và chờ cái còn lại.

Khắc phục:
- Quy ước thứ tự khoá thống nhất (vd: luôn lock sem1 rồi mới lock sem2 ở mọi nơi).
- Hoặc dùng trylock + backoff/timeouts nếu phù hợp.
*/

#### 2.6 Lỗi `sem_open`
- `ENOENT`: mở semaphore chưa tồn tại, không O_CREAT.
- `EEXIST`: tạo với O_EXCL khi đã tồn tại.

**Linux xử lý**:
- Semaphore lưu trong `/dev/shm/sem.<name>`.

// sem_open_examples.cpp
#include <semaphore.h>
#include <fcntl.h>
#include <cstdio>
#include <cerrno>
#include <cstring>
#include <unistd.h>

int main(){
  const char* name = "/demo_sem";

  // ENOENT: Mở semaphore chưa tồn tại và KHÔNG O_CREAT
  errno = 0;
  sem_t* s1 = sem_open(name, 0);            // Không O_CREAT
  if (s1 == SEM_FAILED) {
    printf("sem_open ENOENT: %s\n", strerror(errno)); // Kỳ vọng: ENOENT
  }

  // Tạo semaphore với O_CREAT
  sem_t* s2 = sem_open(name, O_CREAT, 0644, 1);
  if (s2 == SEM_FAILED) { perror("sem_open create"); return 1; }

  // EEXIST: Tạo lại với O_CREAT|O_EXCL khi đã tồn tại
  errno = 0;
  sem_t* s3 = sem_open(name, O_CREAT | O_EXCL, 0644, 1);
  if (s3 == SEM_FAILED) {
    printf("sem_open EEXIST: %s\n", strerror(errno)); // Kỳ vọng: EEXIST
  }

  sem_close(s2);
  sem_unlink(name);                          // /dev/shm/sem.<name>
  return 0;
}

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

// cleanup_examples.cpp
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include <unistd.h>
#include <semaphore.h>
#include <cstdio>

int main(){
  // SHM
  const char* shm_name = "/demo_leak";
  int fd = shm_open(shm_name, O_CREAT | O_RDWR, 0644);
  ftruncate(fd, 4096);
  void* p = mmap(nullptr, 4096, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);

  // SEM
  const char* sem_name = "/demo_sem_leak";
  sem_t* s = sem_open(sem_name, O_CREAT, 0644, 1);

  // ... dùng p và s ...

  // QUÊN shm_unlink / sem_unlink → object vẫn còn trong /dev/shm
  // Khắc phục: unlink khi tiến trình cuối cùng kết thúc
  munmap(p, 4096);
  close(fd);
  shm_unlink(shm_name);   // Xoá lối vào tên (object sẽ biến mất khi không còn tiến trình giữ)
  sem_close(s);
  sem_unlink(sem_name);   // Xoá semaphore có tên
  return 0;
}


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
