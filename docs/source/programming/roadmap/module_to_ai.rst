=================================================================
1. Linux Kernel Subsystems Guide for AI Systems & Infrastructure
=================================================================

:Author: Embedded & AI Infrastructure Engineer
:Target Domain: High-Performance AI Inference, Kernel Tuning, Low-Latency Execution
:Kernel Versions: 4.9 LTS (Legacy CFS) & 6.12+ (Modern EEVDF)
:Status: Approved Technical Reference
:Updated: 2026-04

.. contents:: Nội dung chính
   :depth: 2
   :local:

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: :octicon:`info` Mục tiêu tài liệu
      :shadow: md

      Tổng hợp các **Kernel Subsystems & Modules cốt lõi** giúp tối ưu
      *CPU, RAM, DMA, I/O* cho AI/Inference hiệu năng cao.

   .. grid-item-card:: :octicon:`cpu` Hai thế hệ Kernel
      :shadow: md

      .. list-table::
         :widths: 25 75
         :header-rows: 0

         * - :bdg:`4.9 LTS`
           - Legacy · **CFS** · ``sched_setscheduler``
         * - :bdg-success:`6.12+`
           - Modern · **EEVDF** · ``sched_setattr``

------

1. Scheduler & Process Management
---------------------------------

.. grid:: 1
   :gutter: 2

   .. grid-item-card:: 💡 Vì sao quan trọng?
      :shadow: sm
      :class-card: sd-bg-light

      Đảm bảo CPU luôn ưu tiên cho tác vụ AI — giảm **Latency**,
      triệt tiêu **Jitter**, giữ inference ổn định thời gian thực.

kernel/sched — CFS, EEVDF, Real-Time
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. grid:: 1 1 3 3
   :gutter: 2

   .. grid-item-card:: :octicon:`clock` Virtual Runtime / Deadline
      :shadow: sm

      CFS (4.9) cân bằng theo ``vruntime`` — EEVDF (6.12+)
      lập lịch theo **Virtual Deadline / Latency Slice**.

   .. grid-item-card:: :octicon:`zap` Scheduling Policies
      :shadow: sm

      ``SCHED_OTHER`` (task thường) ·
      ``SCHED_FIFO`` / ``SCHED_RR`` (real-time cho inference) ·
      ``SCHED_DEADLINE`` (deadline cứng).

   .. grid-item-card:: :octicon:`terminal` Key APIs
      :shadow: sm

      ``sched_setscheduler()`` (legacy) ·
      ``sched_setattr()`` (modern) ·
      ``sched_setaffinity()`` (pin core).

cpufreq / cpuidle — Power & Governor
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **Governor:** ``performance`` | ``powersave`` | ``schedutil``
* **Ramp-Up tức thì:** ghi vào ``/sys/.../cpufreq/`` để dựng xung tối đa
  ngay khi AI task chạy — triệt tiêu trễ khởi động.

  .. code-block:: bash

     echo performance | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

cgroups v2 — cpu, cpuset, memory
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **Limits:** ``cpu.max`` · ``cpu.weight`` · ``memory.max`` · ``memory.high``
* **Core Isolation:** ``isolcpus`` + ``nohz_full`` để dành core riêng
  cho inference, tránh nhiễu từ OS/IRQs.

  .. code-block:: bash

     isolcpus=2,3 nohz_full=2,3 rcu_nocbs=2,3

------

2. Memory Management & Zero-Copy
--------------------------------

.. grid:: 1
   :gutter: 2

   .. grid-item-card:: 💡 Vì sao quan trọng?
      :shadow: sm
      :class-card: sd-bg-light

      Tối ưu băng thông nhớ, triệt tiêu ``memcpy`` giữa
      Kernel Space ↔ User Space — sống còn với model lớn.

mm — Virtual Memory, Page Allocator, Swap
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

* **Syscalls:** ``mmap()`` · ``madvise()`` · ``mlock()`` / ``mlockall()``
* **Chống page-fault:** khóa weights/tensors trong RAM, chống swap gây sụt FPS:

  .. code-block:: c

     mlockall(MCL_CURRENT | MCL_FUTURE);

dma-buf — DMA Buffer Sharing
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. important::
   **Module cốt lõi cho Zero-Copy** — chia sẻ Buffer FD trực tiếp giữa
   Camera/Sensor ↔ NPU/GPU ↔ AI Runtime, **không qua** ``memcpy``.

CMA / DMABUF-HEAPS
~~~~~~~~~~~~~~~~~~

Đặt vùng nhớ vật lý liên tục trong ``.dts`` cho tensor allocations:

.. code-block:: dts

   ai_tensor_region: ai@80000000 {
       reg = <0x0 0x80000000 0x0 0x20000000>; /* 512MB */
   };

HUGETLBFS / THP
~~~~~~~~~~~~~~~

Dùng Huge Pages (2MB / 1GB) giảm TLB miss khi load weights lớn (LLM/DNN):

.. code-block:: bash

   echo always | sudo tee /sys/kernel/mm/transparent_hugepage/enabled

------

3. Hardware Interface & IPC
---------------------------

.. grid:: 1
   :gutter: 2

   .. grid-item-card:: 💡 Vì sao quan trọng?
      :shadow: sm
      :class-card: sd-bg-light

      Đưa dữ liệu từ ngoại vi vào pipeline AI với độ trễ thấp nhất.

.. list-table:: Subsystem → Vai trò trong AI Pipeline
   :widths: 30 70
   :header-rows: 1

   * - Subsystem
     - Dùng cho AI như thế nào?
   * - **v4l2 & Media**
     - Frame camera vào RAM chung qua ``V4L2_MEMORY_DMABUF``
   * - **char / UIO / VFIO**
     - Ghi thanh ghi GPU/NPU/FPGA từ user-space, hiệu năng tối đa
   * - **net/core (eBPF/XDP)**
     - Ingest packet ở kernel buffer cho Inference Server

------

4. Profiling, Tracing & Metrics
--------------------------------

.. grid:: 1
   :gutter: 2

   .. grid-item-card:: 💡 Vì sao quan trọng?
      :shadow: sm
      :class-card: sd-bg-light

      Đo chính xác latency, tìm bottleneck, quan sát hệ thống thời gian thực.

.. grid:: 1 1 3 3
   :gutter: 2

   .. grid-item-card:: :octicon:`graph` ftrace
      :shadow: sm

      Trace ``sched_switch``, đo interrupt latency:

      .. code-block:: bash

         echo 1 | sudo tee /sys/kernel/tracing/events/sched/sched_switch/enable

   .. grid-item-card:: :octicon:`pulse` perf_events
      :shadow: sm

      Hardware counters: cache-miss, branch-miss, bus-cycles:

      .. code-block:: bash

         perf stat -e cache-misses,cycles ./ai_infer

   .. grid-item-card:: :octicon:`eye` eBPF
      :shadow: sm

      Gắn ``kprobe`` / ``uprobe`` giám sát latency, I/O
      của AI service mà không làm chậm hệ thống.

------

5. Checklist Thực Hành
----------------------

.. tip::
   Làm tuần tự 3 giai đoạn để đóng gói **Boost Service Wrapper**.

.. grid:: 1 1 1 3
   :gutter: 3

   .. grid-item-card:: :octicon:`rocket` Giai đoạn 1 — Scheduler Tuning
      :shadow: md

      Thử ``sched_setattr()`` · ``sched_setaffinity()`` ·
      ``mlockall()`` trên app mẫu, so sánh jitter trước/sau.

   .. grid-item-card:: :octicon:`package` Giai đoạn 2 — Zero-Copy
      :shadow: md

      Truyền buffer qua ``dma-buf`` giữa 2 process
      **không** ``memcpy``. Đo bandwidth tiết kiệm được.

   .. grid-item-card:: :octicon:`checklist` Giai đoạn 3 — Benchmark
      :shadow: md

      Dùng ``perf`` + ``ftrace`` vẽ latency
      **CFS vs. EEVDF**, chốt governor + affinity tối ưu.