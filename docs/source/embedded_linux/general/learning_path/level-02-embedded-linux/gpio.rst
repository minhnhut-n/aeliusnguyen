GPIO
====

- GPIO sysfs
- interrupt
- LED
- Button

**1. GPIO sysfs**

GPIO sysfs là interface cũ (nhưng đơn giản) để điều khiển GPIO từ userspace.
Các file nằm trong ``/sys/class/gpio/``. Này đã bị deprecated từ kernel 4.8,
nhưng vẫn còn được dùng trong các project cũ hoặc prototype nhanh.

.. code-block:: bash

   # Export GPIO pin (ví dụ GPIO 7 trên Banana Pi)
   $ echo 7 > /sys/class/gpio/export
   $ ls /sys/class/gpio/gpio7/

   # Cấu hình direction (in/out)
   $ echo out > /sys/class/gpio/gpio7/direction
   $ echo in > /sys/class/gpio/gpio7/direction

   # Ghi giá trị (output)
   $ echo 1 > /sys/class/gpio/gpio7/value  # HIGH (3.3V)
   $ echo 0 > /sys/class/gpio/gpio7/value  # LOW (0V)

   # Đọc giá trị (input)
   $ cat /sys/class/gpio/gpio7/value

   # Unexport khi không dùng nữa
   $ echo 7 > /sys/class/gpio/unexport

.. attention::

   GPIO sysfs cũ, chậm và không real-time. Trong kernel mới, dùng
   ``libgpiod`` (character device interface) thay thế. Nhưng sysfs vẫn
   dễ cho người mới học.

**libgpiod** - GPIO character device interface mới:

.. code-block:: bash

   # Kiểm tra gpiochip
   $ gpiodetect
   # gpiochip0 [sun7i-a20-pinctrl] (128 lines)

   # Xem thông tin từng pin
   $ gpioinfo gpiochip0

   # Set pin 7 làm output và ghi value 1
   $ gpioset gpiochip0 7=1
   $ gpioset gpiochip0 7=0

   # Đọc input
   $ gpioget gpiochip0 7

   # Monitor chờ sự kiện trên pin
   $ gpiomon gpiochip0 7

**2. interrupt (GPIO Interrupt)**

GPIO interrupt cho phép bạn "chờ" sự kiện trên một pin mà không cần
polling (vừa tốn CPU vừa chậm). Khi pin thay đổi trạng thái, kernel
gửi signal đến userspace.

.. code-block:: bash

   # Trên sysfs, config interrupt:
   $ echo 7 > /sys/class/gpio/export
   $ echo in > /sys/class/gpio/gpio7/direction
   $ echo both > /sys/class/gpio/gpio7/edge
   # edge: rising, falling, both

   # Dùng poll() trong C để chờ sự kiện
   # (Xem lại level 1 - poll() system call)

.. code-block:: c

   // GPIO interrupt với libgpiod (C)
   #include <gpiod.h>
   #include <stdio.h>
   #include <unistd.h>

   int main() {
       struct gpiod_chip *chip = gpiod_chip_open_by_name("gpiochip0");
       struct gpiod_line *line = gpiod_chip_get_line(chip, 7);

       // Cấu hình input với falling edge interrupt
       gpiod_line_request_falling_edge_events(line, "myapp");

       struct gpiod_line_bulk bulk;
       gpiod_line_bulk_init(&bulk);
       gpiod_line_bulk_add(&bulk, line);

       while (1) {
           // Chờ sự kiện (blocking)
           int ret = gpiod_line_event_wait_bulk(&bulk, NULL, NULL);
           if (ret > 0) {
               struct gpiod_line_event event;
               gpiod_line_event_read(line, &event);
               printf("Interrupt! Timestamp: %ld.%ld\n",
                      event.ts.tv_sec, event.ts.tv_nsec);
           }
       }
   }

**3. LED**

LED là ứng dụng cơ bản nhất của GPIO output. Nhấp nháy LED là "Hello World"
của embedded Linux.

.. code-block:: bash

   # Cách 1: GPIO sysfs (thủ công)
   $ echo 24 > /sys/class/gpio/export      # GPIO 24 (PH24 trên Banana Pi)
   $ echo out > /sys/class/gpio/gpio24/direction
   $ echo 1 > /sys/class/gpio/gpio24/value  # Bật LED
   $ sleep 1
   $ echo 0 > /sys/class/gpio/gpio24/value  # Tắt LED

   # Cách 2: LED class driver (nếu Device Tree đã config)
   $ echo 1 > /sys/class/leds/bananapi:green:usr/brightness
   $ echo 0 > /sys/class/leds/bananapi:green:usr/brightness

   # Chế độ nhấp nháy (trigger)
   $ cat /sys/class/leds/bananapi:green:usr/trigger
   $ echo heartbeat > /sys/class/leds/bananapi:green:usr/trigger

.. code-block:: c

   // Blinky LED với C
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>
   #include <fcntl.h>

   int main() {
       int fd = open("/sys/class/gpio/gpio24/value", O_WRONLY);
       if (fd < 0) {
           perror("open GPIO failed");
           return -1;
       }

       while (1) {
           write(fd, "1", 1);  // Bật
           usleep(500000);     // 500ms
           write(fd, "0", 1);  // Tắt
           usleep(500000);     // 500ms
       }

       close(fd);
       return 0;
   }

**4. Button**

Button là ứng dụng cơ bản của GPIO input, thường dùng để reset config,
shutdown, hoặc trigger một hành động.

.. code-block:: bash

   # Đọc trạng thái button (polling)
   $ echo 7 > /sys/class/gpio/export
   $ echo in > /sys/class/gpio/gpio7/direction

   while true; do
       val=$(cat /sys/class/gpio/gpio7/value)
       if [ "$val" = "1" ]; then
           echo "Button pressed!"
       fi
       sleep 0.1
   done

   # Dùng gpio-keys driver (config trong Device Tree)
   # Kernel tự xử lý debounce và gửi sự kiện input

.. code-block:: dts

   // Device Tree cho button
   gpio-keys {
       compatible = "gpio-keys";
       #address-cells = <1>;
       #size-cells = <0>;

       button@0 {
           label = "SW1";
           linux,code = <KEY_POWER>;  // Mã key (keycodes)
           gpios = <&pio 7 7 GPIO_ACTIVE_LOW>;  // PH7, active low
           debounce-interval = <50>;  // 50ms debounce
       };
   };

.. code-block:: bash

   # Khi button được nhấn, kernel gửi sự kiện input
   $ evtest
   # Chọn /dev/input/eventX tương ứng
   # Nhấn button và xem output
