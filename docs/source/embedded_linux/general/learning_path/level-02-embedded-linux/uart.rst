UART
====

- serial
- DMA
- baudrate
- flow control

**1. Serial (UART)**

UART (Universal Asynchronous Receiver/Transmitter) là giao thức nối tiếp
cổ điển, dùng để debug console, giao tiếp với module GPS, GSM, LoRa,...
Trên Banana Pi, UART0 thường là debug console (qua pin header).

.. code-block:: bash

   # Xem UART devices
   $ ls /dev/ttyS* /dev/ttyAMA*

   # Banana Pi thường có:
   # /dev/ttyS0 - UART0 (debug console, qua GPIO header)
   # /dev/ttyS1 - UART1
   # /dev/ttyS2 - UART2

   # Kiểm tra baudrate và config
   $ stty -F /dev/ttyS0 -a

   # Gửi dữ liệu qua UART
   $ echo "Hello" > /dev/ttyS0

   # Đọc dữ liệu từ UART
   $ cat /dev/ttyS0

   # Kết nối với serial terminal
   $ sudo apt-get install picocom minicom
   $ picocom -b 115200 /dev/ttyS0
   # Ctrl+A Ctrl+X để thoát picocom

.. code-block:: c

   // Lập trình UART với C
   #include <stdio.h>
   #include <fcntl.h>
   #include <unistd.h>
   #include <termios.h>

   int main() {
       int fd = open("/dev/ttyS0", O_RDWR | O_NOCTTY);
       if (fd < 0) {
           perror("open UART failed");
           return -1;
       }

       struct termios tty;
       tcgetattr(fd, &tty);

       // Set baudrate 115200
       cfsetospeed(&tty, B115200);
       cfsetispeed(&tty, B115200);

       tty.c_cflag |= (CLOCAL | CREAD);    // Enable receiver
       tty.c_cflag &= ~CSIZE;
       tty.c_cflag |= CS8;                  // 8 data bits
       tty.c_cflag &= ~PARENB;              // No parity
       tty.c_cflag &= ~CSTOPB;              // 1 stop bit
       tty.c_cflag &= ~CRTSCTS;             // No hardware flow control

       tty.c_lflag &= ~(ICANON | ECHO | ECHOE | ISIG);  // Raw mode
       tty.c_iflag &= ~(IXON | IXOFF | IXANY);          // No software flow control
       tty.c_oflag &= ~OPOST;                            // Raw output

       tcsetattr(fd, TCSANOW, &tty);

       // Gửi dữ liệu
       write(fd, "AT\r\n", 4);

       // Đọc dữ liệu
       char buf[256];
       int n = read(fd, buf, sizeof(buf));
       if (n > 0) {
           buf[n] = '\0';
           printf("Received: %s\n", buf);
       }

       close(fd);
       return 0;
   }

**2. DMA (Direct Memory Access)**

DMA cho phép UART truyền/nhận dữ liệu trực tiếp với RAM mà không cần CPU
tham gia từng byte. CPU chỉ cần setup và nhận interrupt khi hoàn thành.
Điều này cực kỳ quan trọng cho các ứng dụng cần throughput cao.

.. code-block:: bash

   # Kiểm tra DMA có được enable không
   $ dmesg | grep dma
   $ ls /sys/class/dma/

   # Xem DMA channels đang dùng
   $ cat /sys/kernel/debug/dmaengine/summary

   # Trên Banana Pi, DMA controller là "sun4i-dma"
   $ cat /proc/device-tree/soc/dma-controller@* 2>/dev/null

.. note::

   DMA trên UART thường được kernel driver quản lý tự động. Bạn không cần
   config gì thêm ở userspace. Nhưng nếu dùng UART ở chế độ polling
   (không DMA), CPU sẽ bận rộn cho mỗi byte, rất lãng phí.

**3. Baudrate**

Baudrate là tốc độ truyền dữ liệu qua UART, đơn vị là bits per second (bps).
Các baudrate phổ biến:

- ``9600`` - Tiêu chuẩn cũ, dùng cho terminal cổ điển
- ``115200`` - Phổ biến nhất cho embedded Linux debug console
- ``230400`` - Nhanh hơn
- ``921600`` - Rất nhanh, dùng khi cần throughput cao

.. code-block:: bash

   # Kiểm tra baudrate hiện tại
   $ stty -F /dev/ttyS0 speed

   # Đổi baudrate (trên terminal đã mở)
   $ stty -F /dev/ttyS0 115200

   # Custom baudrate (kernel hỗ trợ)
   $ stty -F /dev/ttyS0 1500000  # 1.5 Mbps

.. warning::

   Baudrate giữa hai thiết bị UART phải giống nhau, nếu không dữ liệu
   sẽ bị lỗi (garbled). Ví dụ nếu board gửi ở 115200 mà terminal nhận
   ở 9600, bạn sẽ thấy toàn ký tự rác.

**4. Flow Control (Lưu lượng điều khiển)**

Flow control ngăn buffer tràn khi một bên gửi nhanh hơn bên kia xử lý.

Có hai loại:

- **Hardware flow control (RTS/CTS)**: Dùng 2 dây riêng (Request To Send /
  Clear To Send) để báo hiệu. Đáng tin cậy, dùng trong modem, GPS modules.
- **Software flow control (XON/XOFF)**: Dùng ký tự đặc biệt (Ctrl+S = XOFF,
  Ctrl+Q = XON) trong data stream. Đơn giản nhưng dễ bị lỗi với binary data.

.. code-block:: bash

   # Bật hardware flow control
   $ stty -F /dev/ttyS0 crtscts

   # Tắt flow control
   $ stty -F /dev/ttyS0 -crtscts

   # Bật software flow control
   $ stty -F /dev/ttyS0 ixon ixoff

.. code-block:: c

   // Cấu hình flow control trong C
   struct termios tty;
   tcgetattr(fd, &tty);

   // Enable hardware flow control (RTS/CTS)
   tty.c_cflag |= CRTSCTS;

   // Disable hardware flow control
   tty.c_cflag &= ~CRTSCTS;

   // Enable software flow control (XON/XOFF)
   tty.c_iflag |= (IXON | IXOFF);

   tcsetattr(fd, TCSANOW, &tty);

Lưu ý: Khi dùng hardware flow control, bạn cần kết nối đúng dây RTS và CTS
giữa hai thiết bị. Nếu không có dây này, tắt flow control để tránh bị treo
(hang) khi một bên chờ tín hiệu RTS/CTS không bao giới đến.
