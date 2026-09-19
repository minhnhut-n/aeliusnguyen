=============================================================
Linux Kernel Subsystems Guide for AI Systems & Infrastructure
=============================================================

:Author: Embedded & AI Infrastructure Engineer
:Target Domain: High-Performance AI Inference, Kernel Tuning, Low-Latency Execution
:Kernel Versions: 4.9 LTS (Legacy CFS) & 6.12+ (Modern EEVDF)
:Status: Approved Technical Reference
:Updated: 2026-04

.. contents:: Table of Contents
   :depth: 2
   :local:
   :backlinks: entry

Overview
========

Tài liệu tổng hợp các **Kernel Subsystems** và **Modules** cốt lõi giúp tối ưu hóa tài nguyên phần cứng (*CPU, RAM, DMA, I/O*) cho các ứng dụng và dịch vụ AI/Inference hiệu năng cao.

.. note::
   Tài liệu này hướng tới việc tối ưu hóa cả hai thế hệ Kernel:
   
   * **Kernel 4.9 LTS**: Nền tảng legacy (chạy CFS Scheduler, `sched_setscheduler`).
   * **Kernel 6.12+**: Nền tảng modern (chạy EEVDF Scheduler, `sched_setattr`, `sched_util_min`).

---

Scheduler & Process Management
==============================

Đảm bảo CPU được ưu tiên cho tác vụ AI, giảm thiểu tối đa độ trễ (**Latency**) và triệt tiêu hiện tượng giật lag (**Jitter**).

kernel/sched (CFS, EEVDF, Real-Time Scheduler)
-----------------------------------------------

* **Virtual Runtime vs. Virtual Deadline**:
  Hiểu cơ chế tính toán Virtual Runtime (``vruntime``) trên CFS (Kernel 4.9) và Virtual Deadline / Latency Slices trên EEVDF (Kernel 6.12+).
* **Scheduling Policies**:
  Phân biệt rõ mục đích sử dụng của các chính sách lập lịch: ``SCHED_OTHER``, ``SCHED_FIFO``, ``SCHED_RR``, và ``SCHED_DEADLINE``.
* **System Calls Key APIs**:
  Làm chủ các System Call điều phối tác vụ: ``sched_setscheduler()``, ``sched_setattr()``, và ``sched_setaffinity()``.

cpufreq / cpuidle (Power & Governor Management)
-----------------------------------------------

* **Governor Selection**:
  Nắm vững cơ chế của các Governors: ``performance``, ``powersave``, và ``schedutil``.
* **Instant Frequency Ramp-Up**:
  Tương tác với sysfs (``/sys/devices/system/cpu/cpu*/cpufreq/``) để nâng xung nhịp CPU lên mức tối đa ngay khi kích hoạt AI Task, triệt tiêu độ trễ khởi động.

cgroups v2 (cpu, cpuset, memory)
--------------------------------

* **Resource Limits**:
  Quản lý giới hạn tài nguyên cứng bằng các thông số: ``cpu.max``, ``cpu.weight``, ``memory.max``, và ``memory.high``.
* **Core Isolation**:
  Cô lập CPU (sử dụng boot args ``isolcpus``, ``nohz_full``) để dành riêng Core vật lý cho AI Inference mà không bị ngắt bởi OS (IRQs).

---

Memory Management & Zero-Copy Subsystem
=======================================

Tối ưu hóa băng thông bộ nhớ, triệt tiêu chi phí copy dữ liệu (Memory Overhead) giữa Kernel Space và User Space.

mm (Virtual Memory, Page Allocator, Paging & Swap)
--------------------------------------------------

* **System Calls Core**:
  Nắm vững ``mmap()``, ``madvise()``, và đặc biệt là ``mlock()`` / ``mlockall()``.
* **Preventing Page Faults**:
  Sử dụng ``mlockall(MCL_CURRENT | MCL_FUTURE)`` để khóa toàn bộ Weights và Tensors trong RAM vật lý, chống Paging/Swap ra ổ đĩa gây sụt giảm FPS.

dma-buf (DMA Buffer Sharing Framework)
--------------------------------------

.. important::
   **Module cốt lõi**: Cơ chế chia sẻ con trỏ bộ nhớ (Buffer FD) trực tiếp giữa Camera/Sensor Driver, NPU/GPU Driver và AI Runtime mà **không qua memcpy (Zero-Copy)**.

CMA (Contiguous Memory Allocator) / DMABUF-HEAPS
------------------------------------------------

* **Device Tree Reservation**:
  Định nghĩa và cấp phát vùng nhớ vật lý liên tục dung lượng lớn trong Device Tree (file ``.dts``) dành riêng cho các Tensor allocations.

HUGETLBFS / Transparent Huge Pages (THP)
----------------------------------------

* **TLB Miss Reduction**:
  Tận dụng Huge Pages (2MB / 1GB) để giảm tỷ lệ trễ Cache Miss (TLB Miss) khi load các file Model Weights dung lượng lớn (LLM, Complex DNNs).

---

Hardware Interface & Inter-Process Communication
================================================

Nạp dữ liệu từ thiết bị ngoại vi vào Pipeline tính toán AI với thời gian thực.

v4l2 (Video for Linux 2) & Media Subsystem
------------------------------------------

* Khai thác luồng dữ liệu từ Camera trực tiếp vào khung RAM chung bằng cơ chế ``V4L2_MEMORY_DMABUF``.

char/mem, UIO (Userspace I/O) & VFIO
------------------------------------

* Cho phép điều khiển thanh ghi phần cứng (GPU/NPU/FPGA) trực tiếp từ User-space để đạt hiệu năng giao tiếp tối đa.

net/core & Socket (eBPF / XDP)
------------------------------

* Xử lý luồng dữ liệu mạng (Network Packet Ingestion) ở cấp độ Kernel Packet Buffer cho các hệ thống AI Inference Server.

---

Profiling, Tracing & Performance Metrics
========================================

Đo đạc chính xác độ trễ, điểm nghẽn (Bottlenecks) và hành vi hệ thống theo thời gian thực.

ftrace & tracepoints
--------------------

* Theo dõi luồng chuyển giao tiến trình (``sched_switch``), đo độ trễ ngắt (Interrupt Latency) và thời gian thực thi của Kernel Functions.

perf_events Subsystem
---------------------

* Ghi nhận và phân tích các sự kiện phần cứng (Hardware Counters): CPU Cache Misses, Branch Mispredictions, Bus Cycles.

eBPF (Extended Berkeley Packet Filter)
--------------------------------------

* Gán các Probe (``kprobe``, ``uprobe``) giám sát hành vi, Latency và I/O của AI Service theo thời gian thực mà không làm chậm hệ thống.

---

Checklist Thực Hành Từng Bước
=============================

.. tip::
   Thực hiện tuần tự theo 3 giai đoạn để đóng gói thành công **Boost Service Wrapper**:

1. **Giai đoạn 1 — Scheduler Tuning**:
   Viết code C/C++ thử nghiệm ``sched_setattr()``, ``sched_setaffinity()`` và ``mlockall()`` trên ứng dụng mẫu.

2. **Giai đoạn 2 — Zero-Copy Data Pipeline**:
   Lập trình truyền mảng dữ liệu qua ``dma-buf`` giữa 2 tiến trình độc lập mà không dùng hàm ``memcpy``.

3. **Giai đoạn 3 — Profiling & Benchmarking**:
   Sử dụng ``perf`` và ``ftrace`` trích xuất biểu đồ Latency của AI Task dưới các chính sách lập lịch khác nhau (CFS vs EEVDF).