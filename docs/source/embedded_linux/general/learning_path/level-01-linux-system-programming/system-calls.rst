System Calls
============

Các system call cốt lõi để tương tác với kernel Linux.

**open()** - Mở file hoặc device.

.. code-block:: c

   #include <fcntl.h>
   #include <unistd.h>

   int fd = open("/path/to/file", O_RDWR | O_CREAT, 0644);
   if (fd < 0) {
       perror("open failed");
       return -1;
   }

Các flag thường dùng: ``O_RDONLY``, ``O_WRONLY``, ``O_RDWR``, ``O_CREAT``,
``O_TRUNC``, ``O_APPEND``, ``O_NONBLOCK``.

**close()** - Đóng file descriptor.

.. code-block:: c

   close(fd);

Luôn kiểm tra và đóng fd sau khi sử dụng để tránh rò rỉ tài nguyên.

**read()** - Đọc dữ liệu từ file descriptor.

.. code-block:: c

   #include <unistd.h>

   char buf[1024];
   ssize_t n = read(fd, buf, sizeof(buf));
   if (n < 0) {
       perror("read failed");
   }

Trả về số byte đã đọc, 0 nếu EOF, -1 nếu lỗi.

**write()** - Ghi dữ liệu vào file descriptor.

.. code-block:: c

   const char *msg = "Hello, Linux!";
   ssize_t n = write(fd, msg, strlen(msg));
   if (n < 0) {
       perror("write failed");
   }

**ioctl()** - Input/Output Control: Thao tác với thiết bị phần cứng hoặc
các tham số đặc biệt của device driver.

.. code-block:: c

   #include <sys/ioctl.h>

   int baud = B115200;
   ioctl(fd, TIOCSSERIAL, &baud);

Dùng phổ biến trong UART, GPIO, SPI, I2C, network interfaces.

**mmap()** - Memory-Map File: Ánh xạ file hoặc device vào bộ nhớ process
để truy cập trực tiếp như một mảng.

.. code-block:: c

   #include <sys/mman.h>

   void *addr = mmap(NULL, length, PROT_READ | PROT_WRITE,
                     MAP_SHARED, fd, 0);
   if (addr == MAP_FAILED) {
       perror("mmap failed");
   }

   // Truy cập dữ liệu trực tiếp qua addr
   printf("%s", (char *)addr);

   munmap(addr, length);  // Giải phóng khi không dùng nữa

**poll()** - I/O Multiplexing: Theo dõi nhiều file descriptors cùng lúc
để kiểm tra sự kiện đọc/ghi/lỗi.

.. code-block:: c

   #include <poll.h>

   struct pollfd fds[2];
   fds[0].fd = fd1; fds[0].events = POLLIN;
   fds[1].fd = fd2; fds[1].events = POLLIN;

   int ret = poll(fds, 2, 5000);  // timeout 5 giây
   if (ret > 0) {
       if (fds[0].revents & POLLIN) {
           // fd1 có dữ liệu để đọc
       }
   }

**epoll()** - I/O Event Notification (Linux-specific): Phiên bản cải tiến
của poll, hiệu quả hơn với hàng ngàn file descriptors.

.. code-block:: c

   #include <sys/epoll.h>

   int epfd = epoll_create1(0);
   struct epoll_event ev;
   ev.events = EPOLLIN;
   ev.data.fd = fd;
   epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);

   struct epoll_event events[10];
   int n = epoll_wait(epfd, events, 10, -1);
   for (int i = 0; i < n; i++) {
       if (events[i].events & EPOLLIN) {
           // Xử lý dữ liệu từ events[i].data.fd
       }
   }

   close(epfd);
