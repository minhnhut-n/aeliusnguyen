Shell
=====

- ✅ Bash
- ✅ SSH
- ✅ tmux/screen
- ✅ vim/nano
- ✅ systemd
- ✅ cron

1. Bash
-------

Bash (Bourne Again SHell) là một CLI giúp tự động hóa tuần tự các công việc thông qua
một chuỗi các đoạn mã thực thi mà không cần biên dịch trước. Bash chuyển tuần tự các
code để chạy (trình thông dịch - chuyển trực tiếp các mã nguồn code sang ngôn ngữ máy
để chạy thay vì phải biên dịch trước).

Chi tiết hướng dẫn và học về ngôn ngữ bash có thể tham khảo tại:

- https://www.w3schools.com/bash/bash_getstarted.php

Mẹo: Bash chỉ là một kỹ năng phụ trong quá trình học, không cần quá đầu tư ngay từ đầu,
dùng tới đâu học tới đó.

2. SSH (Secure Shell)
---------------------

SSH là giao thức mạng cho phép truyền dữ liệu không dây một cách bảo mật.

Lưu ý: Các kết nối shell thường thấy là ở local (cho test). Để sử dụng shell cho remote
control thì bạn phải thông qua các phần mềm hoặc công cụ port-forwarding như Tailscale
hay WireGuard.

Một số lệnh SSH hữu ích:

.. code-block:: bash

   # Generate keypair, file sẽ được lưu vào ~/.ssh/
   $ ssh-keygen

   # Login/remote vào bất kỳ Linux server/machine nào có SSH
   $ ssh <username>@<ip_addr>
   # Sau đó nhập password

   # File transmission qua SSH - Upload (local -> remote)
   scp <path_of_src> <username>@<ip_addr>:<path_of_destination>

   # Download (remote -> local)
   scp <username>@<ip_addr>:<path_of_source_remote> <path_of_destination>

   # Với folder, thêm flag -r ngay sau lệnh scp
   scp -r <path_of_src> <username>@<ip_addr>:<path_of_destination>

Chi tiết options có thể xem với ``--help``.

3. tmux / screen
----------------

Do nhu cầu của người dùng Linux thường không có GUI, CLI là cách chính để giao tiếp
giữa người dùng và máy tính. Tmux giải quyết vấn đề multi-task, cho phép mở nhiều
cửa sổ terminal trên cùng một terminal app để chạy song song (parallel task).

"Screen" là công cụ được tích hợp sẵn vào trong linux, nhưng lại ít được cập nhật thậm
chí còn bị lược bỏ trong các bản phát hành linux, còn "tmux" hiện đại hơn và có thể dễ
dàng cài đặt thông qua lệnh.

Tmux sẽ mở lại chính session nếu nó bị ngắt kết nối mạng giữa chừng thay vì phải mở và cấu
hình lại. Điểm khác nhau giữa tmux và mobaxterm (một công cụ hỗ trợ ssh) là khi tab moba đóng
các session sẽ bị kill và các service đang hoạt động cũng vậy, nhưng với tmux các tiến trình
vẫn chạy bình thường.

Ứng dụng này đặc biệt quan trọng trong các trường hợp sử dụng là: máy chủ, build code c/c++,
train AI, script run,... Chỉ cần mở mobaxterm hoặc putty từ bất kì máy nào và gõ "tmux attach"
trạng thái làm việc sẽ được khôi phục.

Mô hình phân cấp trong tmux cần nhớ:

- **Session**: Một phiên làm việc độc lập (ví dụ: ``build``, ``server``). Có thể detach để chạy nền.
- **Window**: Giống như các tab trong trình duyệt, một session chứa nhiều window.
- **Pane**: Chia nhỏ một window thành nhiều ô terminal chạy song song.

Phím Prefix mặc định của tmux là ``Ctrl+b``, nghĩa là: nhấn giữ ``Ctrl`` + ``b``,
thả ra rồi nhấn tiếp phím lệnh. Ví dụ ``Ctrl+b d`` là nhấn ``Ctrl+b`` rồi nhấn ``d``.
Có thể đổi thành ``Ctrl+a`` (kiểu screen) trong file cấu hình.

.. code-block:: bash

   # Install tmux
   sudo apt-get update
   sudo apt-get install tmux

   # Kiểm tra phiên bản
   tmux -V

   # ================= SESSION =================
   # Tạo session mới có tên (nên đặt tên gợi nhớ)
   tmux new -s s_name

   # Tạo session mới đơn giản (tên tự sinh 0,1,2...)
   tmux

   # Tạo session và chạy sẵn lệnh bên trong (vd: top, build)
   tmux new -s monitor -d top
   tmux new -s build -d "cd ~/project && make -j$(nproc)"

   # Liệt kê tất cả session đang có
   tmux ls

   # Detach khỏi session, giữ mọi thứ chạy nền (Prefix + d)
   # Ctrl+b, sau đó nhấn d

   # Attach lại session theo tên
   tmux attach -t s_name
   # Viết tắt
   tmux a -t s_name
   # Attach session gần nhất vừa detach
   tmux attach

   # Attach mà “đuổi” client khác đang bám vào cùng session
   tmux attach -d -t s_name

   # Đổi tên session hiện tại (Prefix + $)
   # Ctrl+b, sau đó nhấn $ rồi gõ tên mới

   # Đổi tên session từ ngoài
   tmux rename-session -t old_name new_name

   # Kill/Xóa một session cụ thể
   tmux kill-session -t s_name

   # Kill tất cả session (dừng server tmux)
   tmux kill-server

   # ================= WINDOW (Tab) =================
   # Tạo window mới (Prefix + c): Ctrl+b c
   # Đóng window hiện tại (Prefix + &): Ctrl+b & -> xác nhận y/n
   # Hoặc gõ trực tiếp: exit
   # Chuyển window kế tiếp/trước đó: Ctrl+b n / Ctrl+b p
   # Nhảy tới window số 0-9: Ctrl+b 0, Ctrl+b 1, ...
   # Liệt kê và chọn window: Ctrl+b w -> arrow keys + Enter
   # Đổi tên window hiện tại (Prefix + ,): Ctrl+b ,
   # Đổi tên window từ ngoài:
   tmux rename-window -t s_name:1 new_window_name
   # Đổi vị trí window (swap window 1 và 2):
   tmux swap-window -s 1 -t 2

   # ================= PANE (Chia ô) =================
   # Split dọc (chia trái-phải) (Prefix + %): Ctrl+b %
   # Split ngang (chia trên-dưới) (Prefix + "): Ctrl+b "
   # Di chuyển giữa các pane: Ctrl+b + arrow keys
   # Chuyển nhanh sang pane kế tiếp: Ctrl+b o
   # Hiện số pane để nhảy nhanh: Ctrl+b q -> nhấn số 0-9
   # Phóng to/thu nhỏ pane toàn màn hình (toggle zoom): Ctrl+b z
   # Xoay vòng layout các pane: Ctrl+b Space
   # Chuyển layout có sẵn:
   tmux select-layout even-horizontal
   tmux select-layout even-vertical
   tmux select-layout main-horizontal
   tmux select-layout main-vertical
   tmux select-layout tiled
   # Resize pane: Ctrl+b :resize-pane -D 10 (U/D/L/R + số dòng)
   # Đóng pane hiện tại (Prefix + x): Ctrl+b x -> xác nhận y, hoặc gõ exit
   # Biến pane thành window riêng: Ctrl+b ! (break-pane)
   # Gộp window thành pane, vd gộp window 1 vào pane hiện tại:
   tmux join-pane -s s_name:1

   # ================= COPY MODE (Cuộn, copy, tìm kiếm) =================
   # Vào chế độ cuộn/copy: Ctrl+b [ (cuộn bằng Arrow/PageUp/PageDown, q để thoát)
   # Paste nội dung vừa copy: Ctrl+b ]
   # Mặc định dùng phím emacs. Để dùng vim-style, thêm vào ~/.tmux.conf:
   # setw -g mode-keys vi
   # Với mode-keys vi: Ctrl+b [ -> Space bắt đầu chọn, Enter để copy,
   # / tìm xuống, ? tìm lên
   # Liệt kê buffer đã copy:
   tmux list-buffers
   tmux show-buffer
   # Lưu lịch sử pane ra file (debug log train/build rất hay):
   tmux capture-pane -p -S -3000 > ~/tmux_log.txt

   # ================= CẤU HÌNH NHANH (~/.tmux.conf) =================
   # Bật chuột: cuộn, click chuyển pane, resize bằng chuột:
   # set -g mouse on
   # Split dễ nhớ: | chia dọc, - chia ngang:
   # bind | split-window -h
   # bind - split-window -v
   # Dùng vim-keys di chuyển pane: Prefix + h/j/k/l:
   # bind h select-pane -L
   # bind j select-pane -D
   # bind k select-pane -U
   # bind l select-pane -R
   # Reload config không cần thoát session (Prefix + r):
   # bind r source-file ~/.tmux.conf \; display "Reloaded!"
   # Sau khi sửa file, nạp lại bằng lệnh:
   tmux source-file ~/.tmux.conf
   # Xem tất cả phím tắt: Ctrl+b ? hoặc tmux list-keys

Workflow gợi ý khi làm việc remote (SSH + tmux):

1. ``ssh user@server`` vào máy remote.
2. ``tmux new -s work`` tạo session làm việc.
3. ``Ctrl+b c`` mở nhiều window: một window code, một window build, một window log.
4. ``Ctrl+b %`` / ``Ctrl+b "`` chia pane để vừa xem log vừa gõ lệnh.
5. Mất mạng / tắt laptop -> mọi thứ vẫn chạy. Hôm sau ``ssh`` lại rồi ``tmux attach -t work`` là khôi phục 100%.

Mẹo thực tế:

- Luôn đặt tên session theo task: ``tmux new -s api``, ``tmux new -s bot`` để không nhầm.
- Dùng ``tmux ls`` trước khi tạo mới để tránh trùng session chạy nền gây tốn RAM/CPU.
- Khi train AI / build Yocto / bitbake hàng giờ: chạy trong tmux, detach ``Ctrl+b d`` rồi tắt SSH thoải mái.
- Không nên chạy tmux lồng nhau (tmux trong tmux qua SSH 2 tầng) nếu chưa đổi prefix, rất dễ rối phím.

4. vim / nano
-------------

Hai trình soạn thảo văn bản phổ biến trong terminal.
Nano phù hợp cho người mới bắt đầu với các thao tác đơn giản:

.. code-block:: bash

   # Mở file với nano
   nano filename.txt

   # Thoát: Ctrl+X
   # Lưu: Ctrl+O
   # Tìm kiếm: Ctrl+W

Vim mạnh mẽ hơn nhưng có learning curve cao hơn:

.. code-block:: bash

   # Mở file với vim
   vim filename.txt

   # Các chế độ trong vim:
   # - Normal mode: mode mặc định khi mở vim
   # - Insert mode:  nhấn i để vào chế độ soạn thảo
   # - Visual mode: nhấn v để chọn văn bản
   # - Command mode: nhấn : để gõ lệnh

   # Lệnh cơ bản:
   # :q  - thoát
   # :w  - lưu
   # :wq - lưu và thoát
   # :q! - thoát không lưu

5. systemd
----------

Systemd là một bộ các công cụ cơ bản để xây dụng các khối (blocks) cho một hệ thống linux.
Nó cung cấp một hệ thống và là một bộ điều khiển hoạt động của các service khác, với PID là 1
và chạy phần còn lại của cả hệ thống.

Systemd (deamon) này tồn tại cơ chế mạnh mẽ có khả năng chạy song song hiệu quả (aggressive
parallelization capabilities) - xin thứ lỗi tại hạ tiếng anh lởm, thông qua socket và D-Bus
để:

- kích hoạt các services. (service activation).
- theo dõi các process thông qua cơ chế cgroup.
- duy trì các mount point (điểm ghép nối) và auto mount point.
- service control logic (chịu trách nhiệm quản lý vòng đời của một service)

"implements an elaborate transactional dependency-based service control logic."

Service control logic (Lập trình / Logic điều khiển dịch vụ):
Là phần code chịu trách nhiệm quản lý vòng đời của các dịch vụ (bắt đầu, dừng, khởi động lại, kiểm tra trạng thái...).

Dependency-based (Dựa trên sự phụ thuộc):
Dịch vụ này phụ thuộc vào dịch vụ khác để chạy.
Ví dụ: Dịch vụ Web Server chỉ được phép chạy sau khi dịch vụ Database đã khởi động xong.

Transactional (Có tính giao dịch / Giao dịch nguyên tố - Atomicity):
Tuân theo nguyên tắc "được ăn cả, ngã về không". Nếu trong quá trình khởi động hoặc dừng một chuỗi dịch vụ mà có một dịch vụ bị lỗi,
hệ thống sẽ tự động Rollback (hoàn tác) toàn bộ về trạng thái an toàn trước đó.

Elaborate (Phức tạp / Tinh vi / Được thiết kế tỉ mỉ):
Mô tả sự phức tạp của logic, xử lý nhiều trường hợp biên (edge cases), ngoại lệ và luồng điều khiển nâng cao.

Reference: https://systemd.io/

Các hạng mục trong systemd là:

Nhóm 1: Quản lý dịch vụ hệ thống
  - systemctl, quản lý dịch vụ, trong đó có start/stop/restart/enable/disable, điều khiển nguồn reboot/poweroff.
  - systemd-analyze, hiệu năng khởi động của hệ thống (booting time)

Nhóm 2: Nhật ký (Logs)
  - journalctl, xem và lọc các nhật ký hệ thống, nhân kernel, hoặc các khoảng thời gian cụ thể.
  - systemd-cat, chuyển hướng đầu ra của một lệnh bất kì hoặc một file text vào hệ thống của journald.

Nhóm 3: System configuration
  - hostnamectl, thay đổi tên máy tính, thông tin kiến trúc hệ điều hành.
  - timedatectl, quản lý thời gian, múi giờ, đồng bộ thời gian qua mạng NTP.
  - localectl, cấu hình ngôn ngữ hệ thống và sơ đồ bàn phím (keyboard layout).

Nhóm 4: Sandbox
  - coredumpctl, tìm kiếm và phân tích các file coredump(dữ liệu lưu lại khi một chương trình crash/lỗi).
  - machinectl, quản lý và tương tác với các container hoặc máy ảo chạy bằng nspawn.

.. code-block:: bash

   # Kiểm tra trạng thái service
   $ systemctl status <service_name>

   # Tắt service
   $ systemctl stop <service>

   # Reboot service (ví dụ khi zombie task)
   $ systemctl restart <service>

   # Enable boot up with system (khi cold boot)
   $ systemctl enable <service>

   # Xem log của service
   $ journalctl -u <service_name>

6. Cron
-------

Cron là một deamon chạy ngầm, có trách nhiệm dọn rác và quét dọn kĩ càng các tệp tin hệ thống,
chịu trách nhiệm tự động hóa các công việc có tính chất chu kì, ví dụ như backup database, dọn
rác trong thư mục.

Tất cả chỉ cần làm là đặt vào cron tab /opt/... một hoặc vài đoạn script, nó sẽ được trigger mỗi
phút và thực hiện các tác vụ như yêu cầu, tránh phải thực hiện một hành động lặp lại tốn thời gian.

.. code-block:: bash

   # Edit crontab cho user hiện tại
   $ crontab -e

   # List crontab hiện tại
   $ crontab -l

   # Cấu trúc cron:
   # * * * * * command
   # - - - - -
   # | | | | |
   # | | | | +---- Day of week (0-7, 0=Sun)
   # | | | +------ Month (1-12)
   # | | +-------- Day of month (1-31)
   # | +---------- Hour (0-23)
   # +------------ Minute (0-59)

   # Ví dụ: Chạy script mỗi ngày lúc 2:30 sáng
   30 2 * * * /path/to/script.sh

   # ==========================================
   # 1. KHỞI ĐỘNG VÀ KÍCH HOẠT (START & ENABLE)
   # ==========================================

   # Khởi động dịch vụ ngay lập tức
   sudo systemctl start cron

   # Bật tự động khởi động cùng hệ điều hành (khi reboot)
   sudo systemctl enable cron

   # LỆNH GỘP: Vừa bật tự khởi động, vừa khởi động ngay lập tức
   sudo systemctl enable --now cron

   # ==========================================
   # 2. KIỂM TRA TRẠNG THÁI (STATUS & LOGS)
   # ==========================================

   # Xem trạng thái chi tiết (đang chạy hay đã dừng, xem log gần nhất)
   sudo systemctl status cron

   # Kiểm tra nhanh dịch vụ có đang hoạt động hay không (active/inactive)
   sudo systemctl is-active cron

   # Kiểm tra xem dịch vụ đã bật tự khởi động cùng máy chưa (enabled/disabled)
   sudo systemctl is-enabled cron

   # Kiểm tra tiến trình cron có thực sự chạy trên RAM hay không
   ps aux | grep cron

   # ==========================================
   # 3. LÀM MỚI VÀ KHỞI ĐỘNG LẠI (RELOAD & RESTART)
   # ==========================================

   # Tải lại cấu hình (áp dụng file cấu hình mới mà không làm gián đoạn tác vụ đang chạy)
   sudo systemctl reload cron

   # Khởi động lại toàn bộ dịch vụ (Tắt đi rồi Bật lại)
   sudo systemctl restart cron

   # ==========================================
   # 4. DỪNG VÀ VÔ HIỆU HÓA (STOP & DISABLE)
   # ==========================================

   # Dừng dịch vụ ngay lập tức (các lịch trình sẽ không chạy nữa)
   sudo systemctl stop cron

   # Tắt tự động khởi động cùng hệ điều hành (reboot sẽ không tự bật)
   sudo systemctl disable cron

   # LỆNH GỘP: Vừa dừng dịch vụ, vừa tắt tự khởi động cùng hệ điều hành
   sudo systemctl disable --now cron

   # Khóa hoàn toàn dịch vụ (ngăn tất cả các lệnh khác vô tình start nó lên)
   sudo systemctl mask cron

   # Mở khóa dịch vụ (nếu trước đó đã dùng lệnh mask)
   sudo systemctl unmask cron

   # ------------------------------------------
   # *Lưu ý: Nếu dùng hệ điều hành dòng RedHat/CentOS/RHEL/Fedora,
   # hãy thay thế chữ "cron" ở cuối mỗi lệnh thành "crond".
   # ------------------------------------------

Các lỗi thường gặp với Cron là:

- Lỗi môi trường, vì là task chạy ngầm nên trong nhiều trường hợp nên viết dưới dạng đường dẫn tuyệt đối.
- Chưa cấp quyền thực thi cho file.
- Không kiểm tra log khi lỗi, bằng các câu lệnh:

  - journal -u cron
  - tail -f /var/log/syslog | grep cron

Tham khảo thêm: https://viblo.asia/p/cron-job-la-gi-huong-dan-su-dung-cron-tab-E375zLo2ZGW
