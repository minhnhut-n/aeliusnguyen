Command line
============

- ps
- top
- htop
- free
- vmstat
- iostat
- netstat
- ss
- lsof

ps
--

**ps** - Process Status: Hiển thị danh sách processes đang chạy.

.. code-block:: bash

   $ ps aux                    # Xem tất cả processes
   $ ps -ef                    # Format khác
   $ ps -u username            # Processes của user cụ thể

top
---

**top** - Real-time process monitoring: Xem processes theo thời gian thực.

.. code-block:: bash

   $ top                       # Mở giao diện real-time
   # Trong top: nhấn 'q' để thoát, 'k' để kill process

htop
----

**htop** - Enhanced version of top (cần cài đặt riêng): Giao diện đẹp hơn, hỗ trợ
mouse, dễ sử dụng hơn top.

.. code-block:: bash

   # Install htop
   sudo apt-get install htop
   $ htop

free
----

**free** - Xem dung lượng RAM và swap.

.. code-block:: bash

   $ free -h

lsof
----

**lsof** - Liệt kê các file đang được mở bởi process, rất hữu ích khi debug socket,
file descriptor hoặc process khóa file.

.. code-block:: bash

   $ lsof -p <PID>
   $ lsof -i :80
   $ sudo lsof | head

ss
--

**ss** - Xem socket và các kết nối mạng hiện tại.

.. code-block:: bash

   $ ss -tulpn

vmstat và iostat
----------------

- **vmstat**: thống kê tần suất memory, process, I/O và CPU.
- **iostat**: thống kê I/O cho block devices.

.. code-block:: bash

   $ vmstat 1
   $ iostat -x 1

Mẹo thực hành
-------------

Khi làm việc với Linux, bạn nên kết hợp nhiều lệnh với nhau để phân tích nhanh:

.. code-block:: bash

   $ ps aux --forest
   $ ps aux | grep nginx
   $ lsof -iTCP -sTCP:LISTEN
