Process
=======

- fork
- exec
- signal
- pipe

fork
----

Là system call tạo ra một process mới (child process) bằng cách copy chính xác
process hiện tại (parent process). Child process nhận một bản sao của address space,
file descriptors, environment variables,... của parent.

.. code-block:: bash

   # Xem process tree
   $ pstree

   # Xem parent PID (PPID) của process
   $ ps -o pid,ppid,cmd

exec
----

Là system call thay thế process image hiện tại bằng một program mới. Khi một
process gọi exec, nó sẽ load program mới vào memory và bắt đầu chạy program đó,
PID của process vẫn giữ nguyên nhưng code, data, stack đều được thay thế.

Sự kết hợp fork + exec là cách Linux tạo ra process mới chạy program khác:

1. fork() tạo child process
2. exec() thay thế child process bằng program mới

signal
------

Là cơ chế giao tiếp bất đồng bộ (asynchronous) giữa các processes hoặc giữa kernel
và process. Signal thông báo cho process về một sự kiện đặc biệt.

.. code-block:: bash

   # Gửi signal đến process
   $ kill -SIGTERM <PID>       # Yêu cầu process kết thúc (graceful)
   $ kill -SIGKILL <PID>       # Buộc process kết thúc ngay lập tức
   $ kill -SIGUSR1 <PID>       # User-defined signal 1

   # Một số signal phổ biến: (copy no righter :))
   # SIGINT (2)  - Ctrl+C, interrupt từ bàn phím
   # SIGTERM (15)- Yêu cầu kết thúc
   # SIGKILL (9) - Buộc kết thúc
   # SIGSTOP (19)- Tạm dừng process
   # SIGCONT (18)- Tiếp tục process bị tạm dừng

pipe
----

Là cơ chế IPC (Inter-Process Communication) cho phép output của process này trở
thành input của process khác. Pipe được ký hiệu bằng ``|`` trong shell.

.. code-block:: bash

   # Ví dụ: Tìm process đang chạy với tên "nginx"
   $ ps aux | grep nginx

   # Ví dụ: Đếm số file trong thư mục
   $ ls -l | wc -l

   # Anonymous pipe (shell pipe): chỉ dùng giữa parent-child processes
   # Named pipe (FIFO): có thể dùng giữa các processes không liên quan
   $ mkfifo mypipe            # Tạo named pipe
   $ echo "hello" > mypipe    # Ghi vào pipe (blocking)
   $ cat mypipe               # Đọc từ pipe
