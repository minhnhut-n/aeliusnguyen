===================
Kernel to Edge AI
===================

.. contents:: Nội dung chính
   :depth: 2
   :local:

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: :octicon:`rocket` Định hướng cốt lõi
      :shadow: md

      Tận dụng thế mạnh **Kernel** (Scheduler EEVDF/CFS, ZRAM,
      Memory Compression, Boost Service) để trở thành
      **System / Performance AI Engineer** — tối ưu Inference,
      Latency, Memory và Power trên phần cứng thực tế.

   .. grid-item-card:: :octicon:`info` Thông tin
      :shadow: md

      .. list-table::
         :widths: 30 70
         :header-rows: 0

         * - Tác giả
           - Kỹ sư Performance / Kernel / Edge AI Systems
         * - Ngày tạo
           - Tháng 9, 2026
         * - Trạng thái
           - :bdg-warning:`Đang thực hiện`

------

1. Profile and Strengths
------------------------

.. grid:: 1
   :gutter: 2

   .. grid-item-card:: 💡 Lợi thế của bạn
      :shadow: sm
      :class-card: sd-bg-light

      Hầu hết kỹ sư AI đi lên từ Software/Data Science — giỏi training
      nhưng yếu khi **deploy & tối ưu hạ tầng**. Điểm mạnh của bạn nằm ở
      **phần cứng, OS và hiệu năng** — đúng thứ Edge AI đang thiếu.

.. list-table:: Ánh xạ Kỹ năng từ Kernel/System sang Edge AI
   :widths: 35 65
   :header-rows: 1

   * - Kỹ năng Kernel/System hiện tại
     - Ánh xạ sang Kỹ năng Edge AI / AI Systems
   * - **Scheduler (EEVDF / CFS) & Boost Service**
     - **Heterogeneous Scheduling:** chia luồng Inference giữa CPU-GPU-NPU,
       kiểm soát Latency/Jitter, ``isolcpus``, Affinity, ``SCHED_FIFO``.
   * - **ZRAM & Memory Compression**
     - **Memory-bound Optimization:** Memory Footprint/Bandwidth,
       nén KV Cache, Quantization (INT8/INT4), Pruning.
   * - **System Overview & Root-cause Debugging**
     - **Bottleneck Profiling:** Cache Thrashing, Memory Bandwidth,
       Thermal Throttling, Latency Spikes.

2. Strategy & Project Re-alignment
-----------------------------------

Chống quá tải và phân tán nguồn lực — hệ thống lại dự án như sau:

.. grid:: 1 1 1 3
   :gutter: 3

   .. grid-item-card:: :octicon:`package` Đóng gói & Dừng
      :shadow: md

      **ESP32 Dashboard & RF24 HAL** (C, Singleton, Event-driven).
      Đã hoàn thành — đóng gói code, viết README đẹp làm Portfolio
      C Clean / Design Pattern.

   .. grid-item-card:: :octicon:`no-entry` Tạm dừng / Lọc bỏ
      :shadow: md

      **ESP32-P4 Android Auto.** Tốn thời gian cho USB Host/Display
      integration — chệch khỏi mục tiêu AI Systems.

   .. grid-item-card:: :octicon:`rocket` Tập trung 100%
      :shadow: md

      **Edge AI Translate / Audio Processing System** — dự án đinh,
      kết hợp Audio Pipeline + Memory/Latency Optimization + Inference.

3. Main Project: Edge AI Translate / Audio Processing
----------------------------------------------------------

Minh chứng rõ nhất cho khả năng kết hợp **System Engineering + Edge AI**.

.. grid:: 1 1 1 3
   :gutter: 3

   .. grid-item-card:: :octicon:`device-camera` Tầng 1 — Frontend (MCU/DSP)
      :shadow: md

      ESP32-S3/P4 thu âm qua **I2S Mic** + TinyML
      (TFLite Micro) làm **VAD / Keyword Spotting**, lọc nhiễu đầu vào.

   .. grid-item-card:: :octicon:`pulse` Tầng 2 — Processing Pipeline
      :shadow: md

      Audio qua **PipeWire / ALSA** → Whisper STT đã nén
      (**INT8 / ONNX Runtime /** ``whisper.cpp``).

   .. grid-item-card:: :octicon:`zap` Tầng 3 — System Tuning (Kernel)
      :shadow: md

      **Latency:** ``SCHED_FIFO`` + CPU Affinity (taskset/isolcpus).
      **Memory:** ``mmap`` + Ring Buffer + tư duy ZRAM.

4. Five Paths to Edge AI
------------------------

.. grid:: 1 1 2 2
   :gutter: 3

   .. grid-item-card:: :octicon:`mortar-board` Bước 1 — AI nền tảng
      :shadow: sm

      Python, NumPy, Pandas. Xác suất thống kê, Classification/Regression,
      Overfitting, Metrics (Accuracy, Precision, Recall, Latency).

   .. grid-item-card:: :octicon:`pulse` Bước 2 — Deep Learning tín hiệu
      :shadow: sm

      **PyTorch** để thử nghiệm & export. 1D-CNN, RNN/LSTM (audio),
      CNN (ảnh). Bài sát nhúng: KWS, Gesture từ IMU, Anomaly Detection.

   .. grid-item-card:: :octicon:`package` Bước 3 — Edge Conversion
      :shadow: sm

      ONNX, TFLite / TFLite Micro, ``llama.cpp`` / ``whisper.cpp``.
      Quantization (INT8/INT4), Pruning, Graph Fusing.
      Trade-off: Accuracy | Latency | RAM/Flash | Power.

   .. grid-item-card:: :octicon:`cpu` Bước 4 — Deploy & tối ưu HW
      :shadow: sm

      **MCU:** STM32, ESP32-S3/P4 (TFLite Micro, CMSIS-NN).
      **SoC:** RPi, Jetson, Orange Pi (ONNX RT, TensorRT, OpenVINO).
      NPU/GPU drivers, SIMD/NEON.

   .. grid-item-card:: :octicon:`graph` Bước 5 — MLOps & Profiling
      :shadow: sm

      ``perf``, ``ftrace``, ``valgrind`` đo nghẽn CPU/Mem khi inference.
      Versioning, Secure OTA, Data Drift Detection, Privacy.

.. _linux-roadmap:

5. JIT Learning: 20 Level Linux Roadmap
---------------------------------------

.. grid:: 1
   :gutter: 2

   .. grid-item-card:: 💡 Đừng học tuần tự — học JIT
      :shadow: sm
      :class-card: sd-bg-light

      Đừng học Level 0 → 20. Áp dụng **Just-In-Time (JIT) Learning**,
      chỉ đào sâu Level phục vụ trực tiếp Performance & AI Systems.

.. list-table:: Định hướng phân bổ 20 Level Linux
   :widths: 30 20 50
   :header-rows: 1

   * - Nhóm Level Linux
     - Ưu tiên
     - Mục tiêu cho Edge AI Systems
   * - **Level 1: System Programming**
     - :bdg-danger:`RẤT CAO`
     - Thread, POSIX Mutex, Shared Memory, IPC, Signal cho Pipeline Audio/AI.
   * - **Level 5–8: Module, Char Driver, Interrupt, Platform**
     - :bdg:`TRUNG BÌNH`
     - Viết/sửa driver I2S Mic, Camera, NPU HAL đơn giản.
   * - **Level 9–10: Memory & Synchronization**
     - :bdg-danger:`RẤT CAO`
     - Page Allocation, DMA Buffers, Zero-copy, Lock-free Queues.
   * - **Level 11 & 16: Debug & Performance**
     - :bdg-danger:`RẤT CAO`
     - ``perf``, ``eBPF``, ``ftrace``, ``tracepoints`` trị Latency Spikes.
   * - **Level 15: Power Management**
     - :bdg-danger:`RẤT CAO`
     - DVFS, Thermal Throttling khi model ngốn CPU/NPU trên thiết bị pin.
   * - **Level 17–19: Buildroot, Yocto, Bootloader**
     - :bdg-success:`CƠ BẢN`
     - Vừa đủ để build Minimal Linux OS cho board nhúng.

.. _kanban-board:

6. Kanban Board: Project Progress & Goals
-------------------------------------------

.. list-table:: Nhật ký Tiến độ & Mục tiêu
   :widths: 25 50 25
   :header-rows: 1

   * - Hạng mục / Dự án
     - Chủ đề / Công việc
     - Trạng thái
   * - **Project 1**
     - ESP32 Dashboard & RF24 HAL (C Clean, Event-driven)
     - :bdg-success:`ĐÃ HOÀN THÀNH`
   * - **C++ Upgrade**
     - Modern C++ (Smart Pointers, RAII, Move Semantics, Concurrency)
     - :bdg-warning:`ĐANG THỰC HIỆN`
   * - **LeetCode Core**
     - 30–50 bài System (Bitwise, Sliding Window, Ring Buffer, Graph DAG)
     - :bdg-warning:`ĐANG THỰC HIỆN`
   * - **Capstone Project**
     - Edge AI Audio Translate (Whisper.cpp + ESP32 + Linux Perf)
     - :bdg-danger:`MỤC TIÊU TRỌNG TÂM`
   * - **AI Optimization**
     - Quantization (INT8), Benchmark ONNX Runtime vs TFLite
     - :bdg:`CHƯA BẮT ĐẦU`

.. seealso::

   * :doc:`module_to_ai` — Linux Kernel Subsystems Guide for AI Systems