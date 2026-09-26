Filesystem
==========

.. contents:: Mục Lục Bài Viết
   :depth: 2
   :local:

Filesystem là gì?
-----------------

.. admonition:: Định nghĩa
   :class: tip

   **File System (FS)** là một mã nguồn code, trong đó phần lớn là các
   *data struct* (cấu trúc dữ liệu) dùng để tổ chức các dữ liệu của hệ thống:
   lưu trữ, truy xuất, quản lý các hệ thống tệp tin. Tóm lại, FS là một nền
   tảng cho việc tương tác với các dữ liệu.

Nói cách khác, FS chính là **khâu trung gian** giữa các app/services và block
devices:

.. code-block:: text
   :caption: Vị trí của File System trong hệ thống

   ┌─────────┐  ┌─────────┐  ┌─────────┐
   │  App 1  │  │  App 2  │  │ Service │
   └────┬────┘  └────┬────┘  └────┬────┘
        │             │             │
        └─────────────┼─────────────┘
                      ▼
        ┌─────────────────────────┐
        │      FILE  SYSTEM       │
        │     ext4, tmpfs, ...    │
        └────────────┬────────────┘
                     ▼
        ┌─────────────────────────┐
        │      BLOCK DEVICE       │
        │    eMMC, SSD, HDD...    │
        └─────────────────────────┘

File system có nhiều loại, tùy thuộc vào hệ thống và hệ điều hành mà nó hỗ trợ:

.. grid:: 2
   :gutter: 2

   .. grid-item-card:: ext4
      :link: fs-ext4
      :link-type: ref

      Lưu trữ lâu dài trên disk (eMMC, SSD, HDD).

   .. grid-item-card:: procfs
      :link: fs-procfs
      :link-type: ref

      Thông tin kernel & processes, mount tại ``/proc``.

   .. grid-item-card:: sysfs
      :link: fs-sysfs
      :link-type: ref

      Thông tin hardware, drivers, kernel objects, mount tại ``/sys``.

   .. grid-item-card:: devtmpfs
      :link: fs-devtmpfs
      :link-type: ref

      Device nodes cho phần cứng, mount tại ``/dev``.

   .. grid-item-card:: tmpfs
      :link: fs-tmpfs
      :link-type: ref

      Lưu trữ tạm trong RAM, mất dữ liệu khi reboot.

Tại sao cần nhiều loại file system?
-----------------------------------

Mỗi loại có những ưu nhược điểm khác nhau, dùng để tối ưu về bộ nhớ hoặc tốc độ
truy xuất dữ liệu theo yêu cầu của target system. File system suy cho cùng cũng
là một **thuật toán**, mà thuật toán thì có giới hạn hoàn hảo của nó cho từng
trường hợp khác nhau.

Một hệ thống có thể có **nhiều file system** phục vụ cho nhiều mục đích khác
nhau; việc mix các thuật toán này theo usecase làm tăng tính linh hoạt và
performance của system về nhiều mặt. **Big-O** là phương pháp tiêu chuẩn để nói
về độ phức tạp của thuật toán — trong tình huống này sẽ xem xét các khía cạnh
như:

- tìm kiếm
- đọc / ghi file
- định vị file

dựa trên số lượng dữ liệu mà thuật toán phải handle.

Cấu tạo cơ bản của một file system
----------------------------------

Một phân vùng disk được tổ chức thành các vùng chức năng như sau:

.. code-block:: text
   :caption: Layout điển hình của một phân vùng

   ┌────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
   │ SUPERBLOCK │BLOCK BITMAP │INODE BITMAP │ INODE TABLE │ DATA BLOCKS │
   ├────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
   │ Kích thước │ Trạng thái  │ Trạng thái  │Metadata của │ Nội dung    │
   │ FS, số     │ của Data    │ của Inodes  │  tất cả file│ thực tế của │
   │inode/block │ Blocks      │             │             │file/thư mục │
   └────────────┴─────────────┴─────────────┴─────────────┴─────────────┘

Vai trò chi tiết của từng vùng:

.. list-table::
   :header-rows: 1
   :widths: 25 75

   * - Vùng
     - Vai trò & nội dung
   * - ``SUPERBLOCK``
     - **Trái tim của FS.** Lưu metadata tổng quan: tổng số block, số inode
       rảnh, kích thước block.
   * - ``BLOCK BITMAP``
     - Dải bit (ví dụ: ``1011001...``) đánh dấu block nào đã dùng và block
       nào còn trống.
   * - ``INODE BITMAP``
     - Dải bit (ví dụ: ``1110001...``) đánh dấu inode nào đã dùng và inode
       nào còn trống.
   * - ``INODE TABLE``
     - Tập hợp các cấu trúc ``struct inode``. Mỗi inode chứa file type,
       permission, kích thước, timestamps và con trỏ tới data block.
   * - ``DATA BLOCKS``
     - Vùng lưu nội dung thực của file hoặc thư mục.

Một inode điển hình có thể mô tả như sau:

.. code-block:: text
   :caption: Ví dụ một inode

   Inode #1024
   ├─ File Type:               Regular File
   ├─ Permissions:             rwxr-xr-- (0754)
   ├─ Owner/Group:             1000/1000
   ├─ File Size:               12,500 Bytes
   ├─ Timestamps:              atime, mtime, ctime
   └─ Data Pointers / Extents: Block #5001, #5002, #5003

Những gì có thể khác giữa các file system
-----------------------------------------

Các file system đều khác nhau, nhưng về cơ bản chúng được xây dựng theo một
khuôn mẫu cụ thể. Vậy chúng khác nhau ở những điểm nào?

.. list-table:: So sánh các loại file system
   :header-rows: 1
   :widths: 32 32 36

   * - Lưu trữ truyền thống
     - Lưu trữ hiện đại
     - Pseudo / In-Memory
   * - ``ext4``, ``FAT32``
     - ``Btrfs``, ``ZFS``
     - ``procfs``, ``sysfs``, ``tmpfs``
   * - Dùng bảng cố định
     - Cấu trúc cây B-Tree
     - Không có đĩa thật
   * - Cấp phát inode tĩnh
     - Copy-on-Write (CoW)
     - Dữ liệu nằm trên RAM
   * - Cần quét phục lỗi
     - Snapshots tức thì
     - Tạo động bởi kernel

.. note::

   Các khái niệm cốt lõi — superblock, bitmap, inode, data block — xuất hiện ở
   hầu hết các file system, chỉ khác nhau ở cách triển khai. Khi học một FS
   mới, hãy tìm các cấu trúc tương ứng này trước.

.. _fs-ext4:

ext4
----

Fourth extended file system, là gen 4 của loại storage file system extX, vai
trò của nó chính là để tương tác với các hệ thống lưu trữ như eMMC, EEPROM,
SSD, HDD, ...

.. admonition:: Ưu điểm
   :class: tip

   - **Tìm kiếm file:** sử dụng cấu trúc H-Tree (variant của B-Tree)
     → ``O(log N)``
   - **Cấp phát không gian lưu trữ cho file lớn:** theo cơ chế extents thay vì
     block-to-block → ``O(1)``
   - **Metadata look-up:** mỗi file được gắn cho một inode, việc đọc các thuộc
     tính của file diễn ra nhanh → ``O(1)``
   - **Phục hồi sự cố dữ liệu:** journaling được ghi vào metadata trước khi
     lưu xuống ổ đĩa thật → ``O(J)``

.. admonition:: Nhược điểm
   :class: warning

   - Khi đĩa gần tiến đến hết dung lượng, các block lớn liên tiếp sẽ không
     được cấp phát, dẫn đến dữ liệu bị chia thành K extents khác nhau,
     làm **phân mảnh dữ liệu**.
   - Số lượng inode cố định khi vừa format ổ đĩa, không có cơ chế dynamic
     allocator → không thể cấp thêm inode khi cần, dù đôi khi ổ đĩa còn trên
     50% dung lượng.
   - Không có cơ chế CoW: khi một file được tạo, mất ``O(k)`` để backup lại
     file đó, không trả về ngay lập tức như các file system có copy-on-write.

.. code-block:: bash
   :caption: Lệnh kiểm tra filesystem & dung lượng

   # Xem thông tin filesystem
   $ df -hT

   # Kiểm tra dung lượng ổ đĩa
   $ df -h

   # Kiểm tra dung lượng thư mục
   $ du -sh /path/to/dir

.. _fs-procfs:

procfs
------

Là filesystem ảo (pseudo filesystem) mount tại ``/proc``, cung cấp thông tin về
kernel và processes đang chạy. Mỗi process có một thư mục riêng với PID tương ứng
bên trong ``/proc``.

.. code-block:: bash
   :caption: Đọc thông tin kernel / process từ /proc

   # Xem thông tin CPU
   $ cat /proc/cpuinfo

   # Xem thông tin memory
   $ cat /proc/meminfo

   # Xem thông tin process cụ thể (PID=1 là init/systemd)
   $ cat /proc/1/status

.. _fs-sysfs:

sysfs
-----

Là filesystem ảo mount tại ``/sys``, cung cấp thông tin về hardware devices,
drivers và kernel objects. sysfs xuất hiện từ kernel 2.6 để thay thế cho các
file lộn xộn trong procfs.

.. code-block:: bash
   :caption: Quan sát hardware qua /sys

   # Xem danh sách block devices
   $ ls /sys/block/

   # Xem thông tin về một device cụ thể (vd: sda)
   $ cat /sys/block/sda/size

   # Xem class devices
   $ ls /sys/class/

.. _fs-devtmpfs:

devtmpfs
--------

Là filesystem ảo mount tại ``/dev``, tự động tạo các device nodes cho hardware
được kernel phát hiện. Trước đây, việc tạo device nodes phải làm thủ công với
``mknod``, devtmpfs tự động hóa quá trình này.

.. code-block:: bash
   :caption: Khám phá /dev

   # Xem các device nodes
   $ ls /dev/

   # Một số device nodes phổ biến:
   # /dev/sda   - ổ cứng SCSI/SATA đầu tiên
   # /dev/ttyUSB0 - USB-to-Serial adapter
   # /dev/mmcblk0 - thẻ nhớ SD/MMC
   # /dev/null  - "black hole" device

.. _fs-tmpfs:

tmpfs
-----

Là filesystem ảo lưu trữ dữ liệu trong RAM (volatile memory). Dữ liệu trên tmpfs
sẽ mất khi reboot. Thường được dùng cho các file tạm thời, cache, hoặc shared memory.

.. code-block:: bash
   :caption: Làm việc với tmpfs

   # Xem các tmpfs đang mounted
   $ df -hT | grep tmpfs

   # Mount tmpfs thủ công
   $ sudo mount -t tmpfs -o size=100M tmpfs /mnt/mytmp

Tổng kết
--------

.. list-table:: So sánh nhanh các filesystem trong Linux
   :header-rows: 1
   :widths: 18 20 42 20

   * - Filesystem
     - Mount point
     - Cung cấp gì
     - Mất dữ liệu khi reboot?
   * - ``ext4``
     - ``/``, ``/data``, ...
     - Lưu trữ lâu dài trên disk (eMMC, SSD, HDD)
     - Không
   * - ``procfs``
     - ``/proc``
     - Thông tin kernel & processes
     - FS ảo, không lưu dữ liệu người dùng
   * - ``sysfs``
     - ``/sys``
     - Thông tin hardware, driver, kernel objects
     - FS ảo, không lưu dữ liệu người dùng
   * - ``devtmpfs``
     - ``/dev``
     - Device nodes cho phần cứng được kernel phát hiện
     - Tạo lại tự động sau mỗi lần boot
   * - ``tmpfs``
     - ``/tmp``, ``/run``, ...
     - Lưu trữ tạm trong RAM
     - Có
