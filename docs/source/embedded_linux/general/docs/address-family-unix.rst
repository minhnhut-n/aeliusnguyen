AF_UNIX — Unix Domain Socket
===============================

.. rubric:: Giới thiệu

**AF_UNIX** (còn gọi là **AF_LOCAL**) là họ địa chỉ socket
dùng cho IPC trên **cùng một máy Linux**.

Nếu ``AF_INET`` nói chuyện bằng IP + port, thì ``AF_UNIX``
nói chuyện qua **file** — ví dụ ``/run/my_logger.sock``.

.. code-block:: text

   App client ================> Daemon server
   (AF_UNIX socket file)   không cần TCP/IP

---

.. rubric:: 1. Đặc điểm chính

- **Giao tiếp nội bộ:** kernel copy trực tiếp, không qua TCP/IP.
- **Địa chỉ là đường dẫn file:** ``struct sockaddr_un.sun_path``.
- **Hỗ trợ cả 2 kiểu:**

  - ``SOCK_STREAM`` — như TCP: 2 chiều, có kết nối, tin cậy.
  - ``SOCK_DGRAM`` — như UDP: 1 message / 1 datagram, có thể mất gói.
- **Bảo mật bằng phân quyền file:** ``chmod / chown`` để giới hạn
  process nào được kết nối.

.. list-table::
   :header-rows: 1

   * - Tiêu chí
     - SOCK_STREAM
     - SOCK_DGRAM
   * - Kết nối
     - Có — ``listen()`` / ``accept()``
     - Không — ``bind()`` rồi ``recv()`` ngay
   * - Độ tin cậy
     - Không mất, giữ thứ tự
     - Mất / drop khi tràn buffer
   * - Khi nghẽn
     - Client bị block
     - Server drop gói, mất log

---

.. rubric:: 2. Vòng đời của một Unix Socket

.. code-block:: text

   unlink() -> socket() -> bind() -> [listen() -> accept()] -> recv()/send() -> close()

.. list-table::
   :header-rows: 1

   * - Bước
     - Hàm
     - Ý nghĩa
   * - 0. Dọn socket cũ
     - ``unlink(path)``
     - Xóa file ``.sock`` còn sót, tránh ``EADDRINUSE``
   * - 1. Mở socket
     - ``socket(AF_UNIX, SOCK_DGRAM / SOCK_STREAM, 0)``
     - Tạo FD trong kernel
   * - 2. Gán địa chỉ
     - ``bind(fd, (struct sockaddr_un *)&addr, ...)``
     - Gán FD vào đường dẫn file
   * - 3. Lắng nghe (chỉ STREAM)
     - ``listen(fd, backlog)``
     - Xếp hàng kết nối chờ từ client
   * - 4. Chấp nhận (chỉ STREAM)
     - ``accept(fd, ...)``
     - Tạo FD mới cho từng client
   * - 5. Trao đổi dữ liệu
     - ``recv()`` / ``recvfrom()`` / ``read()``
     - Nhận dữ liệu vào buffer
   * - 6. Đóng
     - ``close(fd)`` + ``unlink(path)``
     - Giải phóng FD và xóa file socket

.. note::

   Địa chỉ luôn là ``struct sockaddr_un`` với 2 trường:
   ``sun_family = AF_UNIX`` và ``sun_path`` là đường dẫn file.
   Nhớ ``memset()`` và ``strncpy()`` an toàn (tối đa 108 bytes).

---

.. rubric:: 3. Case study: Logging Daemon

Mô hình quen thuộc: nhiều app gửi log về **một daemon duy nhất**,
daemon ghi tuần tự ra ``/var/log/my_system.log``.

.. code-block:: text

   App 1 --sendto()-->                    --fprintf()--> my_system.log
   App 2 --sendto()-->  Logging Daemon    --fprintf()--> (trên đĩa)
   App N --sendto()-->  /run/logger.sock

Vòng lặp đơn giản nhất — **1 luồng vừa recv vừa ghi file**:

.. code-block:: c

   while (1) {
       ssize_t n = recv(server_fd, buffer, sizeof(buffer) - 1, 0);
       if (n > 0) {
           buffer[n] = '\0';
           fprintf(log_file, "%s\n", buffer);
           fflush(log_file);
       }
   }

---

.. rubric:: 4. Vấn đề: khi ghi đĩa chậm hơn nhận socket

``fprintf()`` + ``fflush()`` có thể tốn hàng mili-giây vì **I/O wait**.
Trong lúc daemon mải ghi đĩa, nó **không kịp gọi** ``recv()``,
socket buffer phình lên và:

- ``SOCK_DGRAM``: kernel **drop message** — **mất log** mà client không biết.
- ``SOCK_STREAM``: buffer đầy — client gọi ``write()`` bị **block / treo** —
  **deadlock dây chuyền, watchdog reset, crash**.

.. warning::

   Đừng bao giờ để **đường nhanh (socket receive)** phải chờ
   **đường chậm (disk I/O)** trên cùng một luồng.

---

.. rubric:: 5. Giải pháp: 2 luồng + In-Memory Ring Buffer

Nguyên tắc: **ai nhanh đi đường nhanh, ai chậm đi đường chậm**,
nối nhau bằng hàng đợi FIFO trên RAM.

.. code-block:: text

   Luồng 1 (nhanh): recv() -> push + signal -> RAM ring buffer
   Luồng 2 (chậm):  RAM ring buffer -> pop -> ghi file (fputs)

   sock receive -> push signal -> RAM ring buffer -> pop -> I/O ra file

- **Luồng 1 — Receiver:** chỉ ``recv()`` và ``push`` vào RAM, không chạm vào đĩa.
- **Luồng 2 — Disk writer:** ``pthread_cond_wait()``, có tín hiệu thì ``pop``
  từng message và ``fputs()`` xuống file. Gom batch + ``fflush()`` định kỳ.
- **Ring Buffer:** mảng vòng ``head`` / ``tail``, đồng bộ bằng
  ``pthread_mutex_t`` + ``pthread_cond_t``, FIFO: đến trước — ra trước.

.. tip::

   Muốn chịu burst tốt hơn: tăng ``SO_RCVBUF`` bằng ``setsockopt()``,
   tăng ``RING_BUFFER_SIZE``, chỉ ``fflush()`` khi queue rỗng hoặc theo timer.

---

.. rubric:: 6. Code mẫu hoàn chỉnh

**6.1. Server — Logging Daemon (Unix Datagram)**

.. code-block:: c
   :linenos:

   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>
   #include <unistd.h>
   #include <sys/socket.h>
   #include <sys/un.h>

   #define SOCKET_PATH   "/run/my_logger.sock"
   #define LOG_FILE_PATH "/var/log/my_system.log"

   int main(void)
   {
       int server_fd;
       struct sockaddr_un addr;
       char buffer[1024];

       unlink(SOCKET_PATH); // 0. Xóa socket cũ
       server_fd = socket(AF_UNIX, SOCK_DGRAM, 0); // 1. Tạo socket
       if (server_fd < 0) { perror("socket"); exit(1); }

       memset(&addr, 0, sizeof(addr)); // 2. Chuẩn bị địa chỉ
       addr.sun_family = AF_UNIX;
       strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

       if (bind(server_fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
           perror("bind"); close(server_fd); exit(1); // 3. Bind
       }

       FILE *log_file = fopen(LOG_FILE_PATH, "a"); // 4. Mở file log
       if (!log_file) { perror("fopen"); close(server_fd); exit(1); }

       printf("Daemon running at %s...\n", SOCKET_PATH);

       while (1) { // 5. Nhận log (bản prod: push vào Ring Buffer)
           ssize_t n = recv(server_fd, buffer, sizeof(buffer) - 1, 0);
           if (n > 0) {
               buffer[n] = '\0';
               fprintf(log_file, "%s\n", buffer);
               fflush(log_file);
           }
       }
       close(server_fd); fclose(log_file); unlink(SOCKET_PATH);
       return 0;
   }

**6.2. Ring Buffer 2 luồng — tách receive khỏi I/O**

.. code-block:: c
   :linenos:

   #include <pthread.h>
   #include <stdio.h>
   #include <string.h>
   #define RING_BUFFER_SIZE 1024
   #define LOG_MAX_LEN 512

   typedef struct {
       char data[RING_BUFFER_SIZE][LOG_MAX_LEN];
       int head; int tail;
       pthread_mutex_t lock; pthread_cond_t cond;
   } log_queue_t;
   static log_queue_t log_q;

   void push_to_queue(const char *msg) // Luồng 1: recv -> push
   {
       pthread_mutex_lock(&log_q.lock);
       strncpy(log_q.data[log_q.head], msg, LOG_MAX_LEN - 1);
       log_q.data[log_q.head][LOG_MAX_LEN - 1] = '\0';
       log_q.head = (log_q.head + 1) % RING_BUFFER_SIZE;
       pthread_cond_signal(&log_q.cond);
       pthread_mutex_unlock(&log_q.lock);
   }

   void *disk_writer_thread(void *arg) // Luồng 2: pop -> ghi file
   {
       FILE *log_file = (FILE *)arg;
       while (1) {
           pthread_mutex_lock(&log_q.lock);
           while (log_q.head == log_q.tail)
               pthread_cond_wait(&log_q.cond, &log_q.lock);
           char msg[LOG_MAX_LEN]; // copy nhanh rồi nhả lock
           strncpy(msg, log_q.data[log_q.tail], LOG_MAX_LEN);
           log_q.tail = (log_q.tail + 1) % RING_BUFFER_SIZE;
           pthread_mutex_unlock(&log_q.lock);
           fputs(msg, log_file); fputc('\n', log_file);
       }
       return NULL;
   }

.. note::

   Copy message ra biến cục bộ trước khi ghi đĩa để
   giữ critical section cực ngắn.

---

.. rubric:: 7. Tóm tắt nhanh

- ``AF_UNIX`` = IPC cùng máy qua **file socket**, nhanh gọn hơn ``AF_INET``.
- ``SOCK_STREAM`` không mất nhưng dễ block dây chuyền.
  ``SOCK_DGRAM`` không block sender nhưng **mất gói khi tràn**.
- Tuân thủ: ``unlink -> socket -> bind -> [listen -> accept] -> recv -> close``.
- Daemon log chuẩn prod **luôn tách** ``recv()`` khỏi ``I/O`` bằng
  **Ring Buffer + 2 luồng** FIFO.

---

.. rubric:: Tài liệu tham khảo

- `unix(7) <https://man7.org/linux/man-pages/man7/unix.7.html>`_
- `socket(2) <https://man7.org/linux/man-pages/man2/socket.2.html>`_