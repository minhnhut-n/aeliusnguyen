Introduction — Linux Kernel Scheduler
=====================================

.. rst-class:: lead

   Scheduler là **“bộ não điều phối CPU”** của Linux: quyết định *task nào, chạy ở đâu,
   trong bao lâu* — để vừa **công bằng**, vừa **đúng deadline**, vừa **tiết kiệm pin**.

.. grid:: 1 2 2 3
   :gutter: 3

   .. grid-item-card:: Công bằng
      :text-align: center
      :class-card: sd-shadow-sm

      Chia CPU hợp lý giữa hàng trăm task.
      Đại diện: **CFS, EEVDF**.

   .. grid-item-card:: Đúng giờ
      :text-align: center
      :class-card: sd-shadow-sm

      Đảm bảo task real-time không trễ.
      Đại diện: **RT, Deadline**.

   .. grid-item-card:: Tiết kiệm năng lượng
      :text-align: center
      :class-card: sd-shadow-sm

      Chọn CPU/core “rẻ điện” nhất còn đủ lực.
      Đại diện: **EAS, CAS**.

.. dropdown:: Bạn sẽ học được gì trong chương Scheduler này?
   :color: primary
   :open:

   * **Bức tranh toàn cảnh:** các scheduler class đang có trong mainline.
   * **Bộ máy hỗ trợ:** EAS / Capacity-Aware Scheduling hoạt động kề bên scheduler ra sao.
   * **4 lăng kính đọc code:** computation → algorithm → data structure → vận hành.
   * **Khung phân tích module kernel** chuẩn để áp dụng cho mọi subsystem, không chỉ ``kernel/sched/``.

---

.. rubric:: 1. Bản đồ các Scheduler hiện nay

.. list-table::
   :header-rows: 1
   :widths: 20 45 35

   * - Scheduler
     - Ý tưởng cốt lõi
     - Khi nào quan tâm?
   * - **CFS** — Completely Fair Scheduler
     - Mỗi task có ``vruntime``; ai “thiệt thòi” nhất được chạy trước. Công bằng theo weight.
     - Hệ general-purpose, server, desktop, hiểu nền trước khi học EEVDF.
   * - **EEVDF** — Earliest Eligible Virtual Deadline First
     - Thay ``vruntime`` bằng *virtual deadline*; task nào deadline sớm + đủ điều kiện chạy trước. Thay thế dần CFS từ kernel 6.6+.
     - Latency tốt hơn, công bằng hơn khi task sleep/wake liên tục, cgroup nặng.
   * - **Realtime** — ``RT`` + ``Deadline``
     - ``SCHED_FIFO / SCHED_RR`` theo priority; ``SCHED_DEADLINE`` theo EDF + CBS (runtime/period/deadline).
     - Audio, robot, điều khiển công nghiệp, preempt-rt — nơi trễ 1ms cũng là lỗi.

.. note::
   Muốn hiểu nhanh sự khác nhau **CFS vs EEVDF vs RT**, đọc theo thứ tự:
   công bằng (fairness) → deadline → priority. Đừng nhảy vào code EEVDF khi chưa hình dung
   được ``vruntime`` của CFS.

.. seealso::

   * `Ubuntu Real-Time — Schedulers explanation <https://ubuntu.com/real-time/docs/explanation/schedulers/>`_
   * `kernel/sched/ source — kernel.org <https://elixir.bootlin.com/linux/latest/source/kernel/sched>`_

---

.. rubric:: 2. Cơ chế hỗ trợ bên cạnh Scheduler

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: CAS — Capacity-Aware Scheduling
      :class-card: sd-shadow-sm

      Không coi mọi CPU là như nhau (big.LITTLE, DynamIQ).

      * Dựa vào **capacity** của từng CPU để đặt task nặng lên core khỏe.
      * Liên quan tới ``arch_topology``, ``capacity_orig``, misfit task.

   .. grid-item-card:: EAS — Energy-Aware Scheduling
      :class-card: sd-shadow-sm

      Chọn **nơi đặt task rẻ điện nhất** mà vẫn giữ performance.

      * Dựa vào **Energy Model (EM)** + ``schedutil`` governor.
      * Quyết định tại wake-up và load-balance: có đáng migrate task không?

.. important::
   Scheduler trả lời **“task nào chạy kế tiếp”**, còn **CAS/EAS** trả lời
   **“task đó nên chạy trên CPU nào”**. Hai câu hỏi này luôn đi song song trong code.

---

.. rubric:: 3. Bốn lăng kính để đọc code kernel/sched/

Đừng đọc dàn trải 30k dòng code. Hãy xoay quanh 4 câu hỏi sau:

.. tab-set::

   .. tab-item:: Computations — Tính cái gì?

      * ``vruntime``, deadline, slice (timeslice) tính như thế nào?
      * Load: ``load_avg``, ``runnable_avg``, PELT decay theo thời gian ra sao?
      * Energy: ``compute_energy()`` cộng cost của perf-domain thế nào?
      * Capacity: ``capacity_orig vs capacity_curr``, thermal pressure trừ ở đâu?

   .. tab-item:: Algorithms — Thuật toán nào?

      * **CFS:** cây đỏ-đen + min-vruntime picking.
      * **EEVDF:** eligible + earliest deadline, lag-based compensation.
      * **RT/Deadline:** priority queue + EDF + Constant Bandwidth Server.
      * **Placement:** wakee placement, ``find_idlest_cpu``, ``sched_balance``.
      * **PELT, WALT:** trung bình trượt có trọng số theo thời gian.

   .. tab-item:: Data structures — Dữ liệu ở đâu?

      .. list-table::
         :header-rows: 1

         * - Struct
           - Vai trò ghi nhớ nhanh
         * - ``struct task_struct``
           - Mọi thứ về task: policy, prio, ``se``, ``rt``, ``dl``.
         * - ``struct sched_entity``
           - Thực thể fair: ``vruntime``, ``deadline``, ``lag``.
         * - ``struct cfs_rq / rt_rq / dl_rq``
           - Hàng đợi trên mỗi CPU cho từng class.
         * - ``struct rq``
           - Hàng đợi gốc mỗi CPU: clock, curr, nr_running, lock.
         * - ``struct sched_domain / sched_group``
           - Topology cho load-balance: SMT, core, cluster, die.

   .. tab-item:: Vận hành — Chúng nói chuyện ra sao?

      * **Tick:** ``scheduler_tick()`` → update curr, check preempt.
      * **Wake-up:** ``try_to_wake_up()`` → chọn CPU → enqueue → preempt victim?
      * **Pick-next:** ``__schedule()`` → ``pick_next_task()`` đi qua ``dl → rt → fair → idle``.
      * **Balance:** periodic + idle balance, active balance khi CPU lệch tải.
      * **Hook bên ngoài:** cpufreq (``schedutil``), cpuidle, cgroup, perf, tracepoints.

.. tip::
   Mẹo đọc code cực nhanh: bật ``trace-cmd record -e sched`` hoặc
   ``/sys/kernel/debug/sched/debug`` trước, thấy luồng chạy rồi mới mở code đối chiếu.
   Đọc từ **hook → data → algorithm**, đừng đọc từ algorithm trước.

---

.. rubric:: 4. Khung phân tích mọi Linux Kernel Module

Phần này là “kính lúp” dùng chung — bạn sẽ gặp lại nó ở Memory, Driver, Networking.
Học một lần, tái dùng mọi nơi.

.. grid:: 1 2 2 2
   :gutter: 3

   .. grid-item-card:: A. Phân loại module
      :class-card: sd-shadow-sm

      * Character device
      * Block device driver
      * Network device / manager
      * System / feature module (như ``sched``, ``mm``)

   .. grid-item-card:: B. Vòng đời module
      :class-card: sd-shadow-sm

      * Nạp / khởi tạo / cleanup (exit)
      * ``module_init()``, ``module_exit()``
      * Tham số, dependency, thứ tự init

   .. grid-item-card:: C. User-space interface
      :class-card: sd-shadow-sm

      * Systemcall mechanism
      * Virtual filesystem (``procfs/sysfs/debugfs``)
      * Hardware abstraction (arch hooks)

   .. grid-item-card:: D. Memory, concurrency, device model
      :class-card: sd-shadow-sm

      * Dynamic allocation (slab, buddy, vmalloc)
      * Synchronization và lock (spinlock, rq lock)
      * Platform driver và device, Device Tree

.. dropdown:: Áp thử khung trên vào Scheduler?
   :color: secondary

   * **Loại:** system/feature module, không phải device driver.
   * **Vòng đời:** init lúc boot (``sched_init()``), không rmmod.
   * **Interface:** ``/proc/sched*``, ``/sys/kernel/debug/sched``, syscalls ``sched_setattr``, ``nice``.
   * **Concurrency:** per-CPU ``rq->lock``, RCU, lockless read cho hot-path.
   * **Device model:** gắn với topology + Energy Model + cpufreq/cpuidle.

---

.. rubric:: Đọc tiếp ở đâu?

* Quay lại :doc:`index` để xem mục lục chương Scheduler.
* Đọc sâu Memory trước để hiểu PELT / load tracking? Xem :doc:`../../general_paper/kernel-mm`.
* Chuẩn bị lab: bật ``CONFIG_SCHED_DEBUG=y``, ``CONFIG_SCHEDSTATS=y`` rồi chạy ``stress-ng`` + ``trace-cmd``.

.. seealso::
   Tài liệu tham khảo chính: `Ubuntu Real-Time Schedulers <https://ubuntu.com/real-time/docs/explanation/schedulers/>`_.

