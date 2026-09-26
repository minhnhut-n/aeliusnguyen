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

* **CFS — Completely Fair Scheduler**

  * *Ý tưởng cốt lõi:* mỗi task có ``vruntime``; ai “thiệt thòi” nhất được chạy trước.
    Công bằng theo weight.
  * *Khi nào quan tâm:* hệ general-purpose, server, desktop.
    Hiểu nền này trước khi học EEVDF.

* **EEVDF — Earliest Eligible Virtual Deadline First**

  * *Ý tưởng cốt lõi:* thay ``vruntime`` bằng *virtual deadline*;
    task nào deadline sớm + đủ điều kiện chạy trước. Thay thế dần CFS từ kernel 6.6+.
  * *Khi nào quan tâm:* cần latency tốt hơn, công bằng hơn khi task sleep/wake liên tục,
    cgroup nặng.

* **Realtime —** ``RT`` **+** ``Deadline``

  * *Ý tưởng cốt lõi:* ``SCHED_FIFO / SCHED_RR`` chạy theo priority;
    ``SCHED_DEADLINE`` chạy theo EDF + CBS (runtime / period / deadline).
  * *Khi nào quan tâm:* audio, robot, điều khiển công nghiệp, preempt-rt —
    nơi trễ 1ms cũng là lỗi.

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

.. rubric:: 3. Khung phân tích mọi Linux Kernel Module

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

