# Process and Thread

## Process (Tiến trình)
- Process là một chương trình đang được thực thi. Nó bao gồm mã chương trình, dữ liệu, tài nguyên hệ thống (như bộ nhớ, CPU), và trạng thái thực thi.
- Mỗi tiến trình trong Linux được quản lý bởi kernel và có một PID (Process ID) duy nhất.
- Tiến trình có thể là độc lập hoặc tạo ra các tiến trình con (child processes).
- Đặc điểm của Process:
    - Mỗi tiến trình có không gian địa chỉ riêng biệt (memory space).
    - Tiến trình không chia sẻ dữ liệu trực tiếp với các tiến trình khác (trừ khi sử dụng cơ chế IPC - Inter-Process Communication).
    - Tiến trình được quản lý bởi hệ điều hành thông qua các trạng thái: ready, running, waiting, terminated.
```cpp
#include <iostream>
#include <unistd.h>
int main() {
    pid_t pid = fork(); // Tạo tiến trình con
    if (pid == 0) {
        std::cout << "This is the child process. PID: " << getpid() << std::endl;
    } else if (pid > 0) {
        std::cout << "This is the parent process. PID: " << getpid() << std::endl;
    } else {
        std::cerr << "Failed to create process." << std::endl;
    }
    return 0;
}
```

## Thread (Luồng)
- Thread là đơn vị nhỏ hơn của tiến trình, đại diện cho một luồng thực thi trong tiến trình.
- Một tiến trình có thể chứa nhiều luồng (multithreading), và các luồng trong cùng một tiến trình chia sẻ tài nguyên như bộ nhớ, dữ liệu, và không gian địa chỉ.

- Đặc điểm của Thread:
    - Các luồng trong cùng một tiến trình chia sẻ không gian địa chỉ và tài nguyên.
    - Mỗi luồng có ngăn xếp (stack) riêng để lưu trữ dữ liệu cục bộ.
    - Thread nhẹ hơn process, vì việc tạo và quản lý thread tiêu tốn ít tài nguyên hơn.

```cpp
#include <iostream>
#include <pthread.h>
void* printMessage(void* arg) {
    std::cout << "Thread is running: " << static_cast<char*>(arg) << std::endl;
    return nullptr;
}
int main() {
    pthread_t thread;
    const char* message = "Hello from thread";
    // Tạo thread
    if (pthread_create(&thread, nullptr, printMessage, (void*)message) != 0) {
        std::cerr << "Failed to create thread." << std::endl;
        return 1;
    }
    // Chờ thread kết thúc
    pthread_join(thread, nullptr);
    std::cout << "Thread has finished execution." << std::endl;
    return 0;
}

```

## So sánh Process và Thread
- Process: Là chương trình đang chạy, độc lập và có không gian địa chỉ riêng.
- Thread: Là luồng thực thi trong tiến trình, chia sẻ tài nguyên với các thread khác trong cùng tiến trình.
- Process phù hợp cho các ứng dụng cần sự cô lập, trong khi Thread phù hợp cho các ứng dụng cần hiệu năng cao và chia sẻ tài nguyên.

|Tiêu chí|Process|Thread|
|-|-|-|
|Không gian địa chỉ|Mỗi tiến trình có không gian địa chỉ riêng|Các thread chia sẻ không gian địa chỉ của tiến trình|
|Tài nguyên|Không chia sẻ tài nguyên trực tiếp.|Chia sẻ tài nguyên trong cùng tiến trình.|
|Chi phí tạo|Tốn nhiều tài nguyên hơn.|Tốn ít tài nguyên hơn.|
|Thời gian tạo|Chậm hơn.|Nhanh hơn.|
|Giao tiếp|Sử dụng IPC (Inter-Process Communication).|Giao tiếp dễ dàng qua bộ nhớ chung.|

# Multi Thread

- Multi-threading giúp handle được nhiều request hơn trong một thời điểm
- Với 1 thread thì nếu có 2 request thì sẽ phải chờ chạy lần lươt tung request
- Trong 1 hệ thống linux sẽ có nhiều server hoạt động
- Mỗi Service đó là 1 process
- Mỗi service (process) đó sẽ cần phải thực hiện nhiều nhiệm vụ (task) khác nhau
    - Có thể tạo ra các thread khác nhau để thực hiên mỗi nhiệm vụ đó.

Tạo thread
```cpp
#include <iostream>
#include <thread>
// Function to be executed by the thread
void threadFunction() {
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread is running: " << i << std::endl;
    }
}
int main() {
    // Create a thread and pass the function to it
    std::thread myThread(threadFunction);
    // Wait for the thread to finish execution
    myThread.join();
    std::cout << "Thread has finished execution." << std::endl;
    return 0;
}
```

# Các vấn đề Multi threading

## 1. Race condition
- Vấn đề: Khi nhiều thread cùng truy cập và sửa đổi một tài nguyên chung (ví dụ: biến toàn cục), có thể xảy ra xung đột dữ liệu.
- Giải pháp: Sử dụng các cơ chế đồng bộ hóa như mutex hoặc atomic để bảo vệ tài nguyên.
```cpp
#include <iostream>
#include <thread>
#include <mutex>
int sharedData = 0;
std::mutex mtx;
void increment() {
    for (int i = 0; i < 1000; ++i) {
        std::lock_guard<std::mutex> lock(mtx); // Bảo vệ sharedData
        ++sharedData;
    }
}
int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Shared Data: " << sharedData << std::endl;
    return 0;
}
```
##  2. Deadlocks
- Vấn đề: Deadlock xảy ra khi hai hoặc nhiều thread chờ nhau (dùng nhiều mutex khác nhau) giải phóng tài nguyên, dẫn đến việc không thread nào tiếp tục được.
- Giải pháp: Tránh giữ nhiều mutex cùng lúc. Sử dụng các kỹ thuật như lock hierarchy hoặc std::lock để tránh deadlock.
```cpp
#include <iostream>
#include <thread>
#include <mutex>
std::mutex mtx1, mtx2;
void thread1() {
    std::lock(mtx1, mtx2); // Tránh deadlock bằng std::lock
    std::lock_guard<std::mutex> lock1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(mtx2, std::adopt_lock);
    std::cout << "Thread 1 acquired both locks\n";
}
void thread2() {
    std::lock(mtx1, mtx2); // Tránh deadlock bằng std::lock
    std::lock_guard<std::mutex> lock1(mtx1, std::adopt_lock);
    std::lock_guard<std::mutex> lock2(mtx2, std::adopt_lock);
    std::cout << "Thread 2 acquired both locks\n";
}
int main() {
    std::thread t1(thread1);
    std::thread t2(thread2);
    t1.join();
    t2.join();
    return 0;
}
```
## 3. Starvation
- Vấn đề: Một thread có thể bị "đói" tài nguyên nếu các thread khác liên tục chiếm quyền truy cập.
- Giải pháp: Sử dụng các cơ chế ưu tiên hoặc thiết kế thuật toán công bằng.
```cpp
#include <iostream>
#include <thread>
#include <mutex>
std::mutex mtx;
void highPriorityTask() {
    std::lock_guard<std::mutex> lock(mtx);
    std::cout << "High priority task running\n";
}
void lowPriorityTask() {
    std::lock_guard<std::mutex> lock(mtx);
    std::cout << "Low priority task running\n";
}
int main() {
    std::thread t1(highPriorityTask);
    std::thread t2(lowPriorityTask);
    t1.join();
    t2.join();
    return 0;
}
```

## 4. Thread Safety
- Vấn đề: Đảm bảo rằng các hàm hoặc tài nguyên được sử dụng bởi nhiều thread là thread-safe.
- Giải pháp: Sử dụng các hàm thread-safe hoặc bảo vệ tài nguyên bằng mutex. Tránh sử dụng các tài nguyên không an toàn như biến toàn cục mà không có bảo vệ.
```cpp
#include <iostream>
#include <thread>
#include <vector>
#include <mutex>
std::vector<int> sharedVector;
std::mutex mtx;
void addToVector(int value) {
    std::lock_guard<std::mutex> lock(mtx);
    sharedVector.push_back(value);
}
int main() {
    std::thread t1(addToVector, 1);
    std::thread t2(addToVector, 2);
    t1.join();
    t2.join();
    for (int val : sharedVector) {
        std::cout << val << " ";
    }
    return 0;
}
```
## 5. Synchronization
- Vấn đề: Đồng bộ hóa giữa các thread để đảm bảo thứ tự thực thi hoặc chia sẻ dữ liệu đúng cách.
- Giải pháp: Sử dụng condition variables, semaphores, hoặc barriers.
```cpp
#include <iostream>
#include <thread>
#include <condition_variable>
std::mutex mtx;
std::condition_variable cv;
bool ready = false;
void workerThread() {
    std::unique_lock<std::mutex> lock(mtx);
    cv.wait(lock, [] { return ready; }); // Chờ đến khi ready = true
    std::cout << "Worker thread is running\n";
}
int main() {
    std::thread t(workerThread);
    {
        std::lock_guard<std::mutex> lock(mtx);
        ready = true;
    }
    cv.notify_one(); // Thông báo cho thread
    t.join();
    return 0;
}
```
## 6. Performance Issues
- Vấn đề: Việc sử dụng quá nhiều thread có thể dẫn đến chi phí quản lý cao và giảm hiệu năng.
- Giải pháp: Tối ưu số lượng thread dựa trên số lượng lõi CPU. Sử dụng thread pools để quản lý thread hiệu quả.
```cpp
#include <iostream>
#include <thread>
#include <vector>
void task(int id) {
    std::cout << "Thread " << id << " is running\n";
}
int main() {
    const int numThreads = std::thread::hardware_concurrency(); // Số lõi CPU
    std::vector<std::thread> threads;
    for (int i = 0; i < numThreads; ++i) {
        threads.emplace_back(task, i);
    }
    for (auto& t : threads) {
        t.join();
    }
    return 0;
}
```
## 7. Memory Consistency
- Vấn đề: Các thread có thể không nhìn thấy cùng một trạng thái của bộ nhớ do bộ nhớ đệm CPU.
- Giải pháp: Sử dụng std::atomic hoặc các cơ chế đồng bộ hóa để đảm bảo nhất quán bộ nhớ.
```cpp
#include <iostream>
#include <thread>
#include <atomic>
std::atomic<int> sharedData(0);
void increment() {
    for (int i = 0; i < 1000; ++i) {
        ++sharedData;
    }
}
int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Shared Data: " << sharedData.load() << std::endl;
    return 0;
}
```
## 8. Exception Handling
- Vấn đề: Nếu một thread gặp lỗi hoặc ngoại lệ, cần đảm bảo chương trình không bị crash.
- Giải pháp: Bọc code trong thread bằng try-catch. Sử dụng các cơ chế để truyền ngoại lệ giữa các thread.
```cpp
#include <iostream>
#include <thread>
void task() {
    throw std::runtime_error("Error in thread");
}
int main() {
    try {
        std::thread t(task);
        t.join();
    } catch (const std::exception& e) {
        std::cout << "Caught exception: " << e.what() << std::endl;
    }
    return 0;
}
```
## 9. Thread Lifecycle
- Vấn đề: Quản lý vòng đời của thread (tạo, chạy, kết thúc) để tránh các vấn đề như dangling threads hoặc zombie threads.
- Giải pháp: Luôn gọi join() hoặc detach() khi kết thúc thread.
```cpp
#include <iostream>
#include <thread>
void task() {
    std::cout << "Thread is running\n";
}
int main() {
    std::thread t(task);
    t.detach(); // Tách thread khỏi main
    std::this_thread::sleep_for(std::chrono::seconds(1)); // Đợi thread chạy xong
    std::cout << "Main thread finished\n";
    return 0;
}
```
# 10. Shared Data Access
- Vấn đề: Khi nhiều thread truy cập dữ liệu chung, cần đảm bảo dữ liệu không bị hỏng.
- Giải pháp: Sử dụng mutex, read-write locks, hoặc atomic variables
```cpp
#include <iostream>
#include <thread>
#include <mutex>
int sharedData = 0;
std::mutex mtx;
void increment() {
    for (int i = 0; i < 1000; ++i) {
        std::lock_guard<std::mutex> lock(mtx);
        ++sharedData;
    }
}
int main() {
    std::thread t1(increment);
    std::thread t2(increment);
    t1.join();
    t2.join();
    std::cout << "Shared Data: " << sharedData << std::endl;
    return 0;
}
```