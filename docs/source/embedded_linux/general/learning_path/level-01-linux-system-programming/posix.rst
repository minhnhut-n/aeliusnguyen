POSIX (Portable Operating System Interface)
===========================================

- pthread
- mutex
- condition variable

POSIX là các interface tách biệt hoàn toàn với OS, platform layer.

**pthread**

Interface cho phép process tạo ra nhiều thread khác nhau để concurrency
handle process flow, tận dụng được multithread trên các thiết bị hiện nay.
Thư viện ``<pthread.h>``.

.. code-block:: c

   #include <pthread.h>

   void *thread_func(void *arg) {
       printf("Thread running, arg = %s\n", (char *)arg);
       return NULL;
   }

   int main() {
       pthread_t tid;
       char *msg = "Hello from main";

       // Tạo thread
       pthread_create(&tid, NULL, thread_func, msg);

       // Chờ thread kết thúc
       pthread_join(tid, NULL);

       return 0;
   }

Các API quan trọng trong ``<pthread.h>``:

- ``pthread_create()`` - Tạo thread mới
- ``pthread_join()`` - Chờ thread kết thúc
- ``pthread_detach()`` - Tách thread (tự động giải phóng khi kết thúc)
- ``pthread_exit()`` - Kết thúc thread hiện tại
- ``pthread_self()`` - Lấy ID của thread hiện tại
- ``pthread_equal()`` - So sánh hai thread IDs
- ``pthread_cancel()`` - Gửi yêu cầu hủy thread
- ``pthread_setcanceltype()`` - Thiết lập kiểu hủy (async/deferred)
- ``pthread_once()`` - Đảm bảo một hàm chỉ chạy một lần
- ``pthread_key_create()`` - Tạo thread-specific data key
- ``pthread_setspecific()`` - Gán dữ liệu riêng cho thread
- ``pthread_getspecific()`` - Lấy dữ liệu riêng của thread

**mutex**

Một cơ chế khóa, đồng bộ giữa các process trên application layer, xuất
hiện thường xuyên trong các ứng dụng multi-thread hoặc realtime process.

.. code-block:: c

   #include <pthread.h>

   pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
   int shared_counter = 0;

   void *increment(void *arg) {
       for (int i = 0; i < 1000000; i++) {
           pthread_mutex_lock(&mutex);
           shared_counter++;  // Critical section
           pthread_mutex_unlock(&mutex);
       }
       return NULL;
   }

Việc lập trình đa nhiệm mà không sử dụng mutex có thể dẫn tới các vấn đề:

- **Deadlock**: Hai thread cùng chờ nhau giải phóng tài nguyên
- **Race condition**: Kết quả phụ thuộc vào thứ tự thực thi không xác định
- **Unpredicted result**: Dữ liệu sai lệch do truy cập đồng thời

Các trường hợp sử dụng mutex:

1. **Bảo vệ shared data**: Khi nhiều thread cùng đọc/ghi một biến toàn cục
2. **Atomic operations**: Đảm bảo một chuỗi thao tác được thực thi liên tục
3. **Producer-Consumer**: Đồng bộ giữa thread sản xuất và tiêu thụ dữ liệu
4. **Resource pool**: Quản lý truy cập vào pool tài nguyên dùng chung

Những điều cần tránh khi sử dụng mutex:

1. **Deadlock**: Không bao giờ khóa nhiều mutex theo thứ tự khác nhau ở
   các thread khác nhau. Luôn lock theo cùng một thứ tự.
2. **Không unlock**: Luôn unlock mutex sau khi hoàn thành critical section.
   Sử dụng ``pthread_mutex_destroy()`` khi không còn dùng nữa.
3. **Recursive lock**: Mặc định mutex không cho phép cùng một thread lock
   lại chính nó (gây deadlock). Dùng ``PTHREAD_MUTEX_RECURSIVE`` nếu cần.
4. **Spin-lock trên mutex**: Tránh giữ mutex quá lâu, gây lãng phí CPU.
5. **Lock ordering violation**: Luôn lock/unlock theo thứ tự cố định để
   tránh deadlock.

**condition variable**

Cơ chế đồng bộ cho phép một thread chờ đợi một điều kiện cụ thể xảy ra,
thay vì phải busy-wait (polling). Condition variable luôn đi kèm với mutex.

.. code-block:: c

   #include <pthread.h>

   pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
   pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
   int data_ready = 0;

   void *producer(void *arg) {
       // Sản xuất dữ liệu...
       pthread_mutex_lock(&mutex);
       data_ready = 1;
       pthread_cond_signal(&cond);  // Báo hiệu cho consumer
       pthread_mutex_unlock(&mutex);
       return NULL;
   }

   void *consumer(void *arg) {
       pthread_mutex_lock(&mutex);
       while (!data_ready) {
           // Tự động unlock mutex và chờ signal
           pthread_cond_wait(&cond, &mutex);
           // Khi được đánh thức, tự động lock lại mutex
       }
       // Xử lý dữ liệu...
       pthread_mutex_unlock(&mutex);
       return NULL;
   }

Các API chính:

- ``pthread_cond_wait()`` - Chờ điều kiện (unlock mutex, chờ signal, lock lại)
- ``pthread_cond_signal()`` - Đánh thức một thread đang chờ
- ``pthread_cond_broadcast()`` - Đánh thức tất cả thread đang chờ
- ``pthread_cond_timedwait()`` - Chờ có timeout

Lưu ý: Luôn kiểm tra điều kiện trong vòng lặp ``while`` (không dùng ``if``)
để tránh *spurious wakeup* (thread bị đánh thức giả).
