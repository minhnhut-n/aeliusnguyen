Boot
====

- BootROM
- SPL
- U-Boot
- Device Tree
- Kernel
- init
- systemd

**1. BootROM**

BootROM là một đoạn code nhỏ được nhà sản xuất SoC (System-on-Chip) ghi sẵn
vào ROM (Read-Only Memory) của chip. Nó là thứ đầu tiên chạy khi bạn cấp
nguồn cho board. Nhiệm vụ chính là xác định boot device (SD card, eMMC,
NAND Flash, UART, USB,...) và load SPL vào SRAM.

Có hai dạng Boot ROM:

- **Mask Boot ROM**: Được ghi cứng vào chip tại nhà máy, không thể thay đổi.
- **Writable Boot ROM**: Có thể ghi lại (EEPROM, eMMC,...), cho phép cập nhật
  firmware boot sau này.

Boot ROM sẽ đặt các init code của từng peripheral device (storage device,...)
vào một vùng nhớ cố định ngay lập tức để CPU có thể thực thi.

.. code-block:: bash

   # BootROM thường dùng FEL mode trên Allwinner để boot qua USB
   # Có thể kiểm tra FEL mode bằng cách:
   $ lsusb
   # Nếu thấy "1f3a:efe8" (Onda) hoặc tương tự, board đang ở FEL mode

**2. SPL (Secondary Program Loader)**

SPL là bootloader giai đoạn 2, được load bởi BootROM vào SRAM (vì DRAM chưa
được khởi tạo). Nhiệm vụ của SPL là:

- Khởi tạo clock cơ bản
- Khởi tạo DRAM (bộ nhớ chính)
- Load U-Boot từ boot device vào DRAM
- Jump sang U-Boot

.. code-block:: bash

   # SPL thường là file "sunxi-spl.bin" trên các board Allwinner
   # Xem SPL trong file image:
   $ hexdump -C u-boot-sunxi-with-spl.bin | head -20

**3. U-Boot (Universal Bootloader)**

U-Boot, nguyên gốc là Das-Uboot, là một bootloader open-source được sử dụng
rộng rãi nhất trong thế giới embedded Linux. Nó là một "mini OS" có shell
riêng, cho phép bạn tương tác trước khi kernel được load.

U-Boot là code được chạy trong 1st stage hoặc 2nd stage của bootloader,
được load bởi system ROM từ các thiết bị ngoại vi có thể dùng làm boot
như SD Card, SATA, Flash.

Code U-Boot có thể chia thành 2 stage: First và Second Program Loader (SPL).
Nguyên nhân là do có thể không đủ hoặc giới hạn phần cứng để load hết vào
1 stage. Ví dụ, để khởi tạo DRAM (RAM chính), ban đầu nó sẽ sử dụng CPU
cache làm RAM tạm thời trước khi DRAM được khởi tạo.

U-Boot làm các nhiệm vụ sau:

- Khởi tạo hardware (Ethernet, MMC, USB,...)
- Đọc Device Tree và Kernel từ storage
- Cung cấp môi trường (environment) để cấu hình boot parameters
- Load và jump vào kernel

.. note::

   Chuẩn boot phổ biến trên máy tính hiện đại là UEFI, được định nghĩa trong
   Embedded Base Boot Requirements. UEFI là một binary interface giống với
   GRUB hay Linux kernel, được load/boot thông qua boot manager dưới dạng
   CLI. Trong khi đó, U-Boot là giải pháp đặc thù cho embedded Linux.

.. code-block:: bash

   # Vào U-Boot console (nhấn phím bất kỳ khi boot)
   U-Boot> printenv           # Xem các biến môi trường
   U-Boot> mmc list           # Xem danh sách MMC devices
   U-Boot> mmc dev 0          # Chọn MMC device 0
   U-Boot> fatls mmc 0:1      # Liệt kê file trong partition 1 (FAT)
   U-Boot> bdinfo             # Thông tin board
   U-Boot> boot               # Boot kernel

   # Boot kernel bằng tay:
   U-Boot> fatload mmc 0:1 0x42000000 uImage
   U-Boot> fatload mmc 0:1 0x43000000 sun7i-a20-bananapi.dtb
   U-Boot> bootm 0x42000000 - 0x43000000

**4. Device Tree (Flattened Device Tree - FDT)**

Device Tree là một cấu trúc dữ liệu dạng cây (tree) mô tả toàn bộ phần cứng
trên board: CPU, memory, GPIO, UART, I2C, interrupt controller,...

Tại sao cần Device Tree? Hồi xưa (kernel 2.6 trở về trước), thông tin phần
cứng được hardcode trong source code kernel. Mỗi board là một file ``board-*.c``
riêng. Hàng ngàn board ⇒ không thể maintain nổi. Device Tree ra đời để tách
phần mô tả hardware ra khỏi kernel code.

.. code-block:: dts

   // Ví dụ Device Tree cho Banana Pi (sun7i-a20-bananapi.dts)
   /dts-v1/;

   #include "sun7i-a20.dtsi"

   / {
       model = "Banana Pi";
       compatible = "bananapi,bpi-m1", "allwinner,sun7i-a20";

       memory {
           reg = <0x40000000 0x80000000>;  // 2GB RAM
       };

       leds {
           compatible = "gpio-leds";
           pinctrl-0 = <&led_pins>;

           led-green {
               label = "bananapi:green:usr";
               gpios = <&pio 7 24 GPIO_ACTIVE_HIGH>;
               default-state = "on";
           };
       };
   };

   &uart0 {
       pinctrl-0 = <&uart0_pins>;
       status = "okay";
   };

   &mmc0 {
       pinctrl-0 = <&mmc0_pins>;
       vmmc-supply = <&reg_vcc3v3>;
       bus-width = <4>;
       non-removable;
       status = "okay";
   };

Device Tree compiler (DTC) sẽ biên dịch ``.dts`` thành ``.dtb`` (binary).
Kernel đọc ``.dtb`` để biết nó đang chạy trên hardware nào và khởi tạo driver
tương ứng.

.. warning::

   Việc viết Device Tree sai hoặc thiếu có thể dẫn đến các vấn đề nghiêm trọng:
   nếu nhẹ thì mất cấu hình Wi-Fi, RAM thiếu, peripheral không hoạt động; nặng
   thì brick device, không boot được (no boot load). Vì vậy, luôn kiểm tra kỹ
   Device Tree trước khi flash xuống board.

.. code-block:: bash

   # Biên dịch DTS → DTB
   $ dtc -I dts -O dtb -o sun7i-a20-bananapi.dtb sun7i-a20-bananapi.dts

   # Giải ngược DTB → DTS (để xem)
   $ dtc -I dtb -O dts -o output.dts sun7i-a20-bananapi.dtb

   # Xem Device Tree của hệ thống đang chạy
   $ ls /sys/firmware/devicetree/base/
   $ cat /sys/firmware/devicetree/base/model  # Xem model name

**5. Kernel**

Kernel là trái tim của hệ điều hành Linux. Nó quản lý memory, processes,
drivers, filesystem, networking,... U-Boot load kernel image vào memory và
nhảy đến entry point. Kernel khởi tạo các subsystem, đọc Device Tree để
probe drivers, mount root filesystem và cuối cùng chạy ``/sbin/init``.

.. code-block:: bash

   # Kernel image thường là zImage, uImage hoặc Image
   # Xem kernel version đang chạy
   $ uname -a
   $ cat /proc/version

   # Xem kernel modules đã load
   $ lsmod

   # Xem kernel log (dmesg)
   $ dmesg | head -50  # 50 dòng đầu tiên từ kernel ring buffer

   # Kiểm tra kernel config
   $ zcat /proc/config.gz

**6. init**

Init là process đầu tiên được kernel chạy (PID = 1). Nó là "tổ tiên" của
mọi process khác trên hệ thống. Trên các hệ thống embedded nhỏ gọn, init
có thể là BusyBox init hoặc một init script đơn giản.

.. code-block:: bash

   # Kiểm tra PID 1
   $ ps -p 1 -o pid,comm,cmd

   # Trên hệ thống dùng BusyBox, init thường là /sbin/init
   $ ls -l /sbin/init

   # Init script mặc định: /etc/inittab (dùng cho BusyBox init)
   $ cat /etc/inittab

**7. systemd**

Systemd là init system hiện đại, được dùng trên hầu hết các distro Linux
hiện nay (Ubuntu, Debian, Fedora,...). Nó không chỉ là init mà còn quản lý
service, logging (journald), network (networkd), timer (thay cron), v.v.

.. code-block:: bash

   # systemd là PID 1 trên hầu hết distro hiện đại
   $ systemctl status
   $ systemctl list-units --type=service  # Liệt kê services
   $ systemctl get-default                  # Default target (multi-user/graphical)
   $ systemctl isolate multi-user.target    # Chuyển sang chế độ không GUI

   # Phân tích thời gian boot
   $ systemd-analyze
   $ systemd-analyze blame                 # Xem service nào boot lâu nhất

   # Xem log của một service
   $ journalctl -u ssh.service
