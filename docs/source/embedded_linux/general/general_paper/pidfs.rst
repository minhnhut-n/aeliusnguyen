Pidfs & Process Lifecycle
=========================

.. rubric:: Giới thiệu

Tài liệu phân tích về **pidfs** — virtual file / filesystem hỗ trợ backing cho process ID
dưới dạng file descriptor — và vòng đời (lifecycle) cơ bản của một process trong Linux,
bao gồm scheduler states, exit/zombie/reaped và sự khác biệt giữa ``PID``,
``struct pid`` và ``pidfs``.

---

.. rubric:: 1. Pidfs là gì?

Pidfs là hệ thống filesystem giả trong kernel, hỗ trợ backing (sao lưu) các process ID
dưới dạng một filesystem, xác định identifier của các process thông qua file
thay vì PID truyền thống.

**Vấn đề tại sao phải lưu process ID dưới dạng filesystem kiểu mới thay vì PID thông thường?**

- PID là một con số có thể tái sử dụng khi chương trình own nó kết thúc.
- Linux kernel thu hồi và chuyển PID number cho một process mới.

Việc giữ numeric PID trong userspace rồi lookup lại về sau có nguy cơ trỏ nhầm
sang process mới đã tái sử dụng cùng PID number. Pidfs/pidfd giải quyết bằng
cách giữ một ``stable handle`` (file descriptor) trỏ tới ``struct pid`` cụ thể.

---

.. rubric:: 2. Tổng quát lifecycle cơ bản của một process

.. code-block:: text

                        [ fork / clone / clone3 ]
                                    │
                                    ▼
                        ┌───────────────────────┐
                        │    Process Created    │
                        │      task_struct      │
                        │     PID assigned      │
                        └───────────┬───────────┘
                                    │
                                    ▼
                        ┌───────────────────────┐ ◄──────┐
                        │       RUNNABLE        │        │
                        │     TASK_RUNNING      │        │
                        └───────────┬───────────┘        │
                                    │                    │
                                scheduler                │
                                    │                    │
                                    ▼                    │
                        ┌───────────────────────┐        │
                        │        RUNNING        │        │
                        │    executing on CPU   │        │
                        └─────┬───────────┬─────┘        │
                              │           │              │
                       preempt /          │              │
                      timeslice           │              │
                              │           │ sleep /      │ wakeup
                              │           │ wait         │
                              │           │              │
                              │           ▼              │
                              │ ┌───────────────────┐    │
                              │ │     SLEEPING      │    │
                              │ │  interruptible    │────┘
                              │ │       or          │
                              │ │  uninterruptible  │
                              │ └───────────────────┘
                              │
                    SIGSTOP / │
                  ptrace stop │
                              ▼
                ┌───────────────────────────┐
                │          STOPPED          │
                │   TASK_STOPPED /          │
                │     TASK_TRACED           │
                └─────────────┬─────────────┘
                              │
                           SIGCONT ───► RUNNABLE

.. rubric:: 3. Khi chương trình kết thúc: exit → zombie → reaped

.. code-block:: text

                          RUNNING
                             │
                          exit() /
                     return from main() /
                       fatal signal
                             │
                             ▼
                        ┌───────────┐
                        │  EXITING  │
                        │ do_exit() │ release resources
                        └─────┬─────┘
                              │
                              ▼
                        ┌───────────┐
                        │  ZOMBIE   │
                        │   EXIT_   │ execution gone
                        │  ZOMBIE   │ exit status kept
                        └─────┬─────┘
                              │ parent
                              │ wait() / waitpid() / waitid()
                              ▼
                        ┌───────────┐
                        │  REAPED   │
                        │   EXIT_   │ task cleaned up
                        │   DEAD    │
                        └─────┬─────┘
                              │
                      kernel releases
                       remaining refs
                              │
                              ▼
                    PID eventually reusable
                              │ future
                              │ fork / clone
                              ▼
                   ┌─────────────────────┐
                   │     New Process     │
                   │     may receive     │
                   │   same PID number   │
                   └─────────────────────┘

**Summary sequence:**

.. code-block:: text

   ┌──────────┐
   │  exit()  │
   └─────┬────┘
         │ release resource
         ▼
   ┌───────────────────────────┐
   │ clean up dead/zombie task │
   └──────────────┬────────────┘
                  │ kernel release refs
                  ▼
   ┌──────────────────────────────────────┐
   │ PID available cho task fork() tiếp theo │
   └──────────────────────────────────────┘

---

.. rubric:: 4. Bổ sung về kernel scheduler & process state

Nói cách khác, không có quá trình nào chuyển trực tiếp PID giữa các process,
kernel tự đi collect/reaped để xác định các PID đang free/available để convert
cho process tiếp sau.

**Kernel scheduler:**

- Forward (kernel sched):

  .. code-block:: text

     ┌────────────┐          ┌───────────┐
     │  Runnable  │ ───────► │  Running  │
     └────────────┘          └───────────┘

- Backward (kernel sched / priority preemption):

  .. code-block:: text

     ┌───────────┐          ┌────────────┐
     │  Running  │ ───────► │  Runnable  │
     └───────────┘          └────────────┘

**Về trạng thái Sleeping:**

- Task Interruptible: task đang ngủ nhưng có thể wake up thông qua signal/event.
- Task Uninterruptible: task không được wake thông thường do đang đợi kernel
  operation/resource cần thiết. Phải thông qua kernel scheduler.

Vòng hồi tiếp RUNNING ↔ RUNNABLE (qua SLEEPING khi đợi I/O):

.. code-block:: text

                           sleep / wait (read, wait_event)
                      +---------------------------------------+
                      |                                       v
                +-------------+  I/O complete / wakeup  +------------+  scheduler / dispatch  +-----------+
                |   SLEEPING  | -----------------------> |  RUNNABLE  | ---------------------> |  RUNNING  |
                |  waiting    |                         | TASK_RUNNING|                         | TASK_RUNNING|
                |  for I/O    | <----------------------- | in runqueue | <--------------------- |  on CPU   |
                +-------------+   sleep / block         +------------+  preempt / timeslice    +-----------+
                      ^                                       |                                       |
                      |                                       +---------------------------------------+
                      +-------------------------------------------------------------------------------+
                                   RUNNING -> read() -> SLEEPING -> RUNNABLE -> RUNNING

Giải thích: cả RUNNABLE và RUNNING trong Linux đều là TASK_RUNNING,
chỉ khác là đã được scheduler cho lên CPU hay còn nằm trong runqueue.
Vì vậy chúng tạo thành một vòng kín: RUNNABLE -> RUNNING -> RUNNABLE,
con đường RUNNING -> SLEEPING -> RUNNABLE -> RUNNING là vòng hồi tiếp
phụ khi task phải đợi I/O.

**Về trạng thái Zombie:**

- Zombie là trạng thái của một task/process đã kết thúc, nhưng vẫn còn tồn tại vài thông
  tin nhỏ vì parent process chưa get được thông tin của child.
- Dead process là hậu trạng thái của Zombie, lúc này tiến trình đang được dọn dẹp
  lần cuối trước khi biến mất.

.. rubric:: 5. Ba khái niệm: PID, struct pid, pidfs

- ``PID``: là một con số xác định, dùng để định nghĩa tiến trình trong quá trình hoạt động.
- ``struct pid``: là cấu trúc chứa thông tin của tiến trình, dành cho kernel layer.
- ``pidfs``: là một userspace-visible filesystem, dùng để reference tới tiến trình.

Thứ tự layer của pidfs trong kernel::

   ┌─────────────────┐
   │    Userspace    │
   └────────┬────────┘
            │ file descriptor / virtual FS
            ▼
   ┌─────────────────┐
   │      pidfs      │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   struct pid    │
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   Process/Task  │
   └─────────────────┘

**Quy trình của một vòng đời PID:**

- Parent ``fork()`` / ``clone()`` cấp phát PID cho Process A:

  .. code-block:: text

     ┌────────────────────────┐                  ┌───────────────────────────────────┐
     │ Parent: fork / clone   │ ───────────────► │ Process A: được cấp phát một PID  │
     └────────────────────────┘                  └───────────────────────────────────┘
- Cách tiếp cận pidfd: thay vì dùng trực tiếp ``PID = 1234``,
  ta dùng ``stable handle``.
- Khi process A chết, nó chuyển thành zombie và chờ dọn dẹp:

  - Parent request chờ thu hồi status của A.
  - ``task_struct`` của A được dọn theo refs rules / lifecycle.
  - ``task_struct`` bị hủy và parent thu hồi status thành công. Trong khi ``struct pid``
    vẫn được giữ nguyên, chỉ là empty (no live task).

**Khác biệt giữa task_struct và struct pid:**

- ``struct pid``: đại diện cho identity/thuộc tính mà PID đó có trong kernel.
- ``task_struct``: là data struct riêng của task.

**Ví dụ truy vấn userspace và kernel:**

- Process A có ``struct pid`` (được allocate bởi kernel):

  .. code-block:: text

     ┌──────────────┐          ┌──────────────┐
     │  Process A   │ ◄────────│  struct pid  │
     └──────────────┘          └──────────────┘
     (được allocate bởi kernel)
- Kernel numeric look-up nhận diện được process A:

  .. code-block:: text

     ┌──────────────────────────┐          ┌──────────────┐          ┌────────────────────────────┐
     │ kernel numeric look-up   │ ───────► │  struct pid  │ ───────► │ process A nhận diện được   │
     └──────────────────────────┘          └──────────────┘          └────────────────────────────┘
- Userspace ``pidfd_open()`` truy vấn tới process A:

  .. code-block:: text

     ┌─────────────────────────┐
     │ userspace pidfd_open    │
     └────────────┬────────────┘
                  │ pidfd
                  ▼
     ┌─────────────────────────┐
     │ struct file             │
     └────────────┬────────────┘
                  │
                  ▼
     ┌─────────────────────────┐
     │ pidfs                   │
     └────────────┬────────────┘
                  │
                  ▼
     ┌─────────────────────────┐
     │ struct pid              │
     └────────────┬────────────┘
                  │
                  ▼
     ┌─────────────────────────┐
     │ process A               │
     └─────────────────────────┘

---

.. rubric:: 6. Note về Virtual Filesystem (VFS)

Một folder trong Linux có thể thuộc nhiều kiểu định dạng khác nhau, được biểu
diễn bằng các loại filesystem khác nhau, implement và behavior với data cũng
là khác nhau.

.. code-block:: text

   ┌──────────┐       ┌─────────┐
   │   Ext4   │ ─────►│   SSD   │
   └──────────┘       └─────────┘

   ┌──────────┐       ┌───────────┐       ┌──────────┐
   │   NFS    │ ─────►│  network  │ ─────►│  server  │
   └──────────┘       └───────────┘       └──────────┘

   ┌──────────┐       ┌─────────┐
   │  Tmpfs   │ ─────►│   RAM   │
   └──────────┘       └─────────┘

   ┌──────────┐       ┌──────────────────────┐
   │  Procfs  │ ─────►│ kernel information   │
   └──────────┘       └──────────────────────┘

Và để application/services tương tác được với dữ liệu thì phải thông qua filesystem.
Nếu hệ thống dùng ``if/else`` cho từng loại thì architect của system là cực kỳ tệ.

Lúc này một abstraction layer mới được sinh ra để làm general API chung
cho các application, gọi là **VFS (Virtual Filesystem interface)**. Đồng thời
cung cấp các interface cần thiết như ``open()``, ``read()``, ``write()``, ``stat()``, ...

VFS qua đó được hiểu là một ``ABSTRACTION LAYER`` (logic). Tương tự với pidfs, nó là
một lớp logic như vậy nhưng chỉ dành cho PID/process identity, hỗ trợ backing
cho PID fd.

.. code-block:: text

   ┌───────────────────────────────────────────────────────────┐
   │ USERSPACE                                                 │
   │ pidfd = 5                                                 │
   └─────────────────────────────┬─────────────────────────────┘
                                 │
   ──────────────────────────────┼──────────────────────────────
   KERNEL                        │
                                 ▼
                          ┌─────────────┐
                          │ struct file │
                          └──────┬──────┘
                                 │
                                 ▼
                          ┌─────────────┐
                          │     VFS     │
                          │   generic   │
                          │  abstract   │
                          │ filesystem  │
                          │    layer    │
                          └──────┬──────┘
                                 │
                                 ▼
                          ┌─────────────┐
                          │    pidfs    │
                          │  process-   │
                          │  specific   │
                          │ VFS backing │
                          └──────┬──────┘
                                 │
                                 ▼
                          ┌─────────────┐
                          │ struct pid  │
                          └──────┬──────┘
                                 │
                                 ▼
                      process identity

---

.. rubric:: 7. Old way vs New way (PID lookup vs pidfd)

Trước đây userspace thường giữ numeric PID và mỗi lần muốn thao tác thì gửi PID đó
xuống kernel để kernel lookup task/process hiện tại.

Với pidfd, userspace mở một handle một lần bằng ``pidfd_open(pid, ...)``, nhận về một FD
number; từ đó các operation hỗ trợ pidfd dùng chính handle đó thay vì lookup lại
bằng numeric PID.

**OLD WAY (numeric PID lookup):**

.. code-block:: text

   ┌──────────────────┐
   │    userspace     │
   └────────┬─────────┘
            │ PID = 1234
            ▼
   ┌──────────────────┐
   │     syscall      │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │ kernel lookup    │
   │    PID 1234      │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │  current process │
   │  for PID 1234    │
   └──────────────────┘

**NEW WAY (Step 1: Obtain handle):**

.. code-block:: text

   ┌──────────────────┐
   │    userspace     │
   └────────┬─────────┘
            │ pidfd_open(1234)
            ▼
   ┌──────────────────┐
   │      kernel      │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │  resolve PID     │
   │   1234 once      │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │   struct pid     │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │ pidfs /          │
   │ struct file      │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │ install FD in    │
   │ caller table     │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │ userspace gets   │
   │     fd = 5       │
   └──────────────────┘

**NEW WAY (Step 2: Operations using handle):**

.. code-block:: text

   ┌──────────────────┐
   │    userspace     │
   └────────┬─────────┘
            │ pidfd operation using fd 5
            ▼
   ┌──────────────────┐
   │    FD table      │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │   struct file    │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │      pidfs       │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │   struct pid     │
   └────────┬─────────┘
            │
            ▼
   ┌──────────────────┐
   │ specific process │
   │     identity     │
   └──────────────────┘

---

.. rubric:: Tài liệu tham khảo

- `Linux Kernel Documentation - Virtual Filesystem <https://www.kernel.org/doc/html/latest/filesystems/vfs.html>`_
- `pidfd_open(2) — Linux man page <https://man7.org/linux/man-pages/man2/pidfd_open.2.html>`_
- `Linux man pages <https://man7.org/linux/man-pages/>`_
