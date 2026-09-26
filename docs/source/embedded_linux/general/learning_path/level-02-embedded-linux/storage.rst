Storage
=======

- MMC
- SD
- eMMC
- Partition
- mount
- fstab

**1. MMC (MultiMediaCard)**

MMC là chuẩn giao tiếp cho thẻ nhớ, dùng giao thức serial 1-bit hoặc
multi-bit (4-bit, 8-bit). Trên Banana Pi, MMC controller kết nối với
khe SD card và eMMC chip.

.. code-block:: bash

   # Xem MMC devices
   $ ls /dev/mmcblk*

   # Thông tin chi tiết về MMC device
   $ cat /sys/block/mmcblk0/device/name
   $ cat /sys/block/mmcblk0/device/manfid

   # Xem kernel log về MMC
   $ dmesg | grep mmc

**2. SD (Secure Digital)**

SD là phiên bản phát triển của MMC, phổ biến trên các board nhúng và
máy ảnh. SD card có thể dùng làm boot device cho Banana Pi.

.. code-block:: bash

   # Kiểm tra tốc độ SD card
   $ dd if=/dev/mmcblk0 of=/dev/null bs=1M count=100
   # Kết quả cho thấy tốc độ đọc (càng cao càng tốt)

   # Ghi image vào SD card (cẩn thận với of=)
   $ sudo dd if=ubuntu.img of=/dev/mmcblk0 bs=4M status=progress

**3. eMMC (embedded MMC)**

eMMC là MMC được hàn cố định trên board (không thể tháo rời như SD card).
eMMC nhanh hơn SD card và tin cậy hơn. Banana Pi Pro có eMMC onboard.

.. code-block:: bash

   # Kiểm tra dung lượng eMMC
   $ fdisk -l /dev/mmcblk1   # eMMC thường là mmcblk1

   # Xem eMMC health (nếu driver hỗ trợ)
   $ cat /sys/block/mmcblk1/device/life_time
   $ cat /sys/block/mmcblk1/device/pre_eol_info

**4. Partition**

Partition là cách chia ổ đĩa thành các vùng riêng biệt. Trên embedded Linux,
bạn thường thấy layout kiểu:

.. code-block:: bash

   # Layout điển hình trên Banana Pi boot từ SD card
   $ fdisk -l /dev/mmcblk0

   # Ví dụ output:
   # Device         Boot  Start      End  Sectors  Size Id Type
   # /dev/mmcblk0p1 *      2048   133119   131072   64M  c W95 FAT32 (boot)
   # /dev/mmcblk0p2      133120 15564799 15431680  7.4G 83 Linux (rootfs)

- Partition 1 (FAT32): Chứa bootloader, kernel, device tree
- Partition 2 (ext4): Chứa root filesystem (Ubuntu/Debian)

Công cụ partition phổ biến:

.. code-block:: bash

   # fdisk - cổ điển, ổn định
   $ sudo fdisk /dev/mmcblk0

   # parted - hỗ trợ GPT
   $ sudo parted /dev/mmcblk0 print

   # gdisk - dành cho GPT
   $ sudo gdisk /dev/mmcblk0

**5. mount**

Mount là quá trình gắn một filesystem vào một thư mục (mount point) để
truy cập dữ liệu. Không mount thì không xài được.

.. code-block:: bash

   # Mount thủ công
   $ sudo mount /dev/mmcblk0p2 /mnt/rootfs
   $ ls /mnt/rootfs

   # Mount với options
   $ sudo mount -o rw,noatime /dev/mmcblk0p1 /mnt/boot

   # Xem tất cả mount points
   $ mount
   $ df -hT

   # Unmount
   $ sudo umount /mnt/rootfs

**6. fstab (File System Table)**

File ``/etc/fstab`` là file cấu hình cho biết partition nào sẽ được mount
tự động vào thư mục nào khi boot.

.. code-block:: bash

   # Ví dụ /etc/fstab trên Banana Pi
   $ cat /etc/fstab

   # Format:
   # <device>    <mount_point>  <fstype>  <options>  <dump>  <pass>
   /dev/mmcblk0p1  /boot         vfat      defaults    0       2
   /dev/mmcblk0p2  /             ext4      defaults,noatime 0 1
   tmpfs           /tmp          tmpfs     defaults    0       0

Giải thích các cột:

- **device**: Block device (``/dev/mmcblk0p1``) hoặc UUID
- **mount_point**: Thư mục mount (``/boot``, ``/``)
- **fstype**: Loại filesystem (ext4, vfat, tmpfs)
- **options**: ``defaults``, ``noatime`` (tăng tốc), ``ro`` (read-only)
- **dump**: Có backup bằng dump không (0 = không)
- **pass**: Thứ tự kiểm tra fsck (1 = root, 2 = other, 0 = không kiểm tra)

Dùng UUID để tránh lỗi khi thay đổi device name:

.. code-block:: bash

   # Lấy UUID của partition
   $ blkid /dev/mmcblk0p2
   # Output: /dev/mmcblk0p2: UUID="abc123..." TYPE="ext4"

   # /etc/fstab với UUID
   UUID=abc123... / ext4 defaults,noatime 0 1
