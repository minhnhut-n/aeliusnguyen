Kernel Memory Management (Draft)
================================

.. rubric:: Giới thiệu

Tài liệu phân tích về **Kernel Memory Management** trong Linux, bao gồm cấu trúc tổng quát, các module quản lý bộ nhớ, và các cơ chế cấp phát/giải phóng bộ nhớ.

---

.. rubric:: 1. Cấu trúc tổng quát và Syntax của từng module

- Mỗi file ``.c`` là một module độc lập, chịu trách nhiệm cho từng bài toán cụ thể trong quản lý bộ nhớ, theo quy tắc.
- Mỗi file một cơ chế riêng.
- Các file phải được build theo có điều kiện, cmake, kernel config.
- Chia sẻ các include nội bộ.

**Rule name của các file:**

- ``Swap*``: phục vụ cho thao tác swap in-out memory giữa ổ cứng và RAM.
- ``Z*``: compress, nén các memzone size để tiết kiệm tài nguyên RAM.
- ``Sl*``: slab allocator, slab (tấm/khổ lớn) là cơ chế quản lý dữ liệu cho các action tái sử dụng các vùng nhớ cố định cho caches, slabs, và individual object. Phục vụ cho các obj nhỏ.

---

.. rubric:: 2. Core task: phân bổ và giải phóng page

.. list-table::
   :header-rows: 1

   * - Module
     - Chức năng
     - Mục đích
   * - Buddy allocator
     - Cấp phát, giải phóng các physical page, quản lý zones, watermark, migration types
     - Giải các physical RAM thành các pool, các page block có thể cấp phát nhanh O(logN), giải quyết vấn đề phân mảnh bộ nhớ
   * - Memblock
     - Boot time memory allocator quản lý memory trước khi buddy system sẵn sàng
     - Cấp phát bộ nhớ trong giai đoạn boot time khi chưa có allocator nào được đăng kí chính thức
   * - Mm init
     - Khởi tạo memory subsystem, thiết lập zones, memmap, per CPU page set
     - Chuẩn bị/khởi tạo memory management package khi kernel boot
   * - Init mm
     - Mm data structure
     - Cung cấp page table gốc cho kernel space
   * - Page counter
     - Bộ đếm lượng sử dụng (usage) được sử dụng bởi memory cgroup
     - Giới hạn tài nguyên cgroup, khi vượt giới hạn tài nguyên được cấp
   * - Shuffle
     - Randomize page allocation order
     - Giảm khả năng dự đoán layout bộ nhớ

---

.. rubric:: 3. SLAB/SLUB/SLOB: Các đối tượng cấp phát nhỏ

.. list-table::
   :header-rows: 1

   * - Module
     - Chức năng
     - Mục đích
   * - Slab common
     - Cấu trúc chung của slab, infrastructure, slab allocators
     - Chuẩn hóa interface kmalloc/kfree cho tất cả backend
   * - Slab
     - Slab allocator, cache per CPU, phân mảnh thấp, performance + caching, phức tạp nhiều metadata
     - Small object (struct/buffer): Tránh lãng phí khi dùng buddy cho các object nhỏ
   * - Slub
     - Đơn giản hóa SLAB + scalability (khả năng mở rộng), lựa chọn chủ đạo, performance tốt
     - Thay thế slab, ít overhead, debug tốt hơn, scalable tốt hơn trên SMP (các hệ thống multiprocessing được kết nối chặt chẽ)
   * - Slob
     - Allocator nhỏ gọn cho hệ thống nhúng
     - Dành cho embedded real-time có RAM ít và để tiết kiệm code size
   * - Mempool
     - Phần nằm trước các allocator, sử dụng các vùng reserve mỗi khi low mem và không thể allocate thêm block cho I/O thực hiện reserve process
     - Đảm bảo cấp phát trong tình huống khẩn cấp (khi hệ thống thiếu RAM)
   * - Dma pool
     - Cấp phát block nhỏ cho các yêu cầu DMA
     - Thiết bị DMA cần allocation block nhỏ liên tục (contiguous) có alignment yêu cầu
   * - Vmalloc
     - Virtual contiguous memory allocator
     - Vùng nhớ virtual, thường dùng cho các app cần cấp vùng nhớ liên tục (không phải physically contiguous memory)

---

.. rubric:: 4. Virtual memory và page fault

.. list-table::
   :header-rows: 1

   * - Module
     - Chức năng
     - Mục đích
   * - Memory.c
     - Xử lý page fault, page table manipulation, thực hiện các hành động swap page
     - Heart of "cấp page theo yêu cầu", lazy allocation, swap-in, copy on write
   * - Mmap
     - Là syscall, tạo và quản lý VMA (virtual memory area)
     - Ánh xạ file/anonymous memory vào address space của tiến trình
   * - Pagewalk
     - Page table worker chung (generic)
     - Duyệt qua các page table cho từng callback

---

.. rubric:: 5. Swap & Reclaim: Thu hồi bộ nhớ

.. list-table::
   :header-rows: 1

   * - Module
     - Chức năng
     - Mục đích
   * - Swap
     - Swap core - swap cache, entry management
     - Hạ tầng của swap, trách nhiệm theo dõi những page đang ở swap mode
   * - Kswap
     - Kernel page swapping - swap kernel memory
     - Swap được .data/.bss của kernel module - tiết kiệm RAM trên thiết bị Tizen
   * - Swap file
     - Quản lý các swap device/file
     - Format/active/deactive swap area

---

.. rubric:: 6. Migration, Compaction, CMA

.. list-table::
   :header-rows: 1

   * - Module
     - Chức năng
     - Mục đích
   * - Migrate
     - Page migration - di chuyển các physical page
     - NUMA balancing, compaction, CMA, memory hotplug
   * - Compaction
     - Dồn page để tạo ra contiguous block
     - Giảm external fragmentation → cấp phát được các high-order page
   * - Cma
     - Contiguous memory allocator
     - Thiết bị DMA cần vùng nhớ liên tục, nhưng kernel đã bị phân mảnh. CMA sẽ reserve vùng cho page cache dùng, khi cần thì migrate đi
   * - Express CMA
     - Low-latency CMA variant
     - Giảm latency CMA allocation - predict response time

---

.. rubric:: 7. Package with MM/analysis

**Scenario in use (Key scenario):**

1. Kernel page swapping
.. 2. Express CMA - Predictable Latency
3. Z4 Fold density compression
4. Page fault statistics
5. Memory logger
6. Efficient memory allocator

---

.. rubric:: 8. Kernel page swapping

**API name:** ``kswap_*``

- ``kswap_init``: Khởi tạo các kernel page swapping subsystem.
- ``kswap_module_load``: Đánh dấu các module page có thể được swap.
- ``kswap_fault_handler``: Handles page faults cho các page đã được swap out.
- ``kswap_shrink_module``: Khởi tạo các swapping của module memory (shrink, thu nhỏ).

**Mục đích của swap kernel module:**

- Di chuyển tạm thời các page đang chạy ngầm sang ổ cứng rời, giảm tải cho RAM của PC, đặc biệt hữu ích trong các trường hợp RAM bị giới hạn.
- Tránh lãng phí khi nhiều app cần cung cấp resource nhưng chỉ có một app được chạy thực tế.

**How it works:**

- Kernel module không được gom vào một chỗ mà thay vào đó code và data của kernel sẽ được page out ra một vùng nhớ khác (có thể là SSD).
- Khi module cần (code + data), kernel pages it back in RAM.
- Giảm lượng tiêu thụ RAM khi load các module.

**Impact:**

- Trong các models TV hoặc device có RAM size nhỏ (256MB hoặc 512MB).
- Đánh đổi giữa việc phải I/O (disk) và save RAM memory.

---

.. .. rubric:: 9. Express CMA

.. **API name:** ``express_cma_*``

.. - ``express_cma_init``: Khởi tạo module.
.. - ``express_cma_alloc``: Cấp phát nhanh bộ nhớ cho việc giảm độ trễ.
.. - ``express_cma_free``: Trả lại các vùng nhớ đã cấp phát.
.. - ``express_cma_predict_latency``: Tính toán thời gian allocation time.
.. - ``express_cma_prealloc_pool``: Đặt trước các vùng nhớ liên tục.

.. **Mục đích của express CMA module:**

.. - Giảm các latency không đoán trước được do các vấn đề:
..   - Page migration overhead
..   - Compaction delays (thời gian nén page)
..   - Reclaim operation (quá trình dọn dẹp và thu hồi page)
.. - Giảm hiện tượng stuttering/frame drops trong suốt quá trình video playback và multimedia rendering.

.. **How it works:**

.. - Cấp phát trước và dành riêng các vùng nhớ liên tục (contiguous block).
.. - Tối thiểu hóa các migration trong quá trình cấp phát.
.. - Dự đoán các allocation latency cho việc lập lịch.

.. ---

.. rubric:: 10. Z4 Fold Density Compression (z4fold.c)

**API name:** ``z4fold_*``

- ``z4fold_alloc``: Cấp phát một z4fold page để chứa các compressed object.
- ``z4fold_free``: Giải phóng z4fold page khi không còn object nào sử dụng.
- ``z4fold_map``: Ánh xạ (map) một compressed object trong z4fold page để đọc/ghi.
- ``z4fold_unmap``: Hủy ánh xạ (unmap) compressed object sau khi sử dụng xong.
- ``z4fold_compact``: Dồn các compressed object lại để giải phóng page trống.
- ``z4fold_get_size``: Lấy kích thước thực tế của compressed object trong page.

**Mục đích của z4fold module:**

- Tăng mật độ lưu trữ (storage density) cho các compressed pages trong zswap/zram.
- Giảm số lượng physical pages cần thiết khi lưu trữ nhiều compressed objects nhỏ.
- Giảm memory fragmentation và metadata overhead so với việc mỗi object chiếm một page riêng.
- Tối ưu hóa cho các workload có nhiều page nhỏ sau khi nén (ví dụ: swap, file cache).

**How it works:**

- Z4Fold gom (pack) tối đa **4 compressed objects** vào trong **một physical page** (do đó có tên "4-fold").
- Mỗi z4fold page được chia thành các vùng nhỏ (slots) để chứa các compressed object có kích thước khác nhau.
- Khi một compressed object được lưu vào z4fold page, nó được đánh dấu bằng metadata (header) để có thể truy xuất lại.
- Khi page đầy hoặc cần thêm không gian, z4fold sẽ:
  - Tìm page có đủ chỗ trống để chèn object mới.
  - Nếu không có page phù hợp, cấp phát một z4fold page mới.
  - Khi một object được giải phóng, z4fold có thể dồn (compact) các object còn lại để gom thành page trống và trả về hệ thống.
- Z4Fold hoạt động như một **allocator tầng dưới** cho zswap/zram, giúp tận dụng tối đa không gian RAM khi lưu trữ dữ liệu đã nén.
- So với zsmalloc (cơ chế tương tự), z4fold tập trung vào việc tối ưu hóa cho **density** (mật độ) hơn là **scalability** (khả năng mở rộng), phù hợp với các hệ thống có RAM giới hạn.

---

.. rubric:: 11. Page Fault Statistics (fault_stat.c)

**API name:** ``fault_stat_*``

- ``fault_stat_init``: Khởi tạo hệ thống theo dõi và thống kê lỗi system.
- ``fault_stat_record_fault``: Dùng để log ra một page fault event.
- ``fault_stat_get_stats``: Retrieve (truy xuất) các thống kê bằng các process id, cgroup.
- ``fault_stat_reset``: Dọn dẹp thống kê.
- ``fault_stat_show_debugfs``: Hiển thị trạng thái thông qua debugfs interface.
- ``fault_stat_analyze_pattern``: Phân tích các patterns cho các trường hợp dị thường.

**Mục đích của page fault:**

- Có những truy cập bộ nhớ dị thường làm suy giảm system performance (perf degradation) xảy ra, nguồn của các issue đó đến từ:
  - Background service
  - Streaming buffer management
  - UI rendering
  - Video decoding

**How it works:**

- Theo dõi các page theo loại Type có thể được chia thành major/minor, process/cgroup.
- Lưu trữ các memory latencies và tần suất xảy ra.
- Phân loại các loại lỗi: paging, swap-in/out, file cache, etc.
- Expose ra (phơi ra) các metric (số liệu) thông qua các file system (debugfs/sysfs).

---

.. rubric:: 12. Memory Logger (mem_logger.c)

**API name:** ``mem_logger_*``

- ``mem_logger_init``: Khởi tạo circular buffer logger.
- ``mem_logger_write``: Xuất các event sang các buffer (lock-free, interrupt-safe).
- ``mem_logger_get_entry``: Truy xuất (retrieve) các sự kiện đã được ghi log.
- ``mem_logger_dump``: Xuất tất cả các event đã được log.
- ``mem_logger_clear``: Xóa bỏ buffer.
- ``mem_logger_show_debugfs``: Xuất các log thông qua debugfs (debug file system).

**Mục đích của mem_logger module:**

- ``printk`` là module dùng để xuất các log từ kernel (print kernel), có thể bị deadlock khi interrupt handlers.
- Page fault handlers
- Reclaim/swap paths
- DMA completion handlers

Nếu không có logging, debugging memory là cực kì phức tạp.

**How it works:**

- Logging được lưu theo cơ chế log theo circular buffer trong kernel memory.
- Lock-free logging operations an toàn cho các bối cảnh sử dụng interrupt context.
- Records: page faults, allocation failure, reclaim events, OOM conditions, đều được lưu trữ.
- Dump on crash, các trường hợp crash có thể dump ra để debug được thông qua debugfs interface.

---

.. rubric:: 13. Efficient Memory Allocator (efficient_mem.c)

**API name:** ``efficient_mem_*``

- ``efficient_mem_init``: Khởi tạo bộ cấp phát memory hiệu quả.
- ``efficient_mem_carve_pool``: Trích xuất các memory từ vùng memory đã được để dành.
- ``efficient_mem_alloc``: Cấp phát từ pool của module.
- ``efficient_mem_free``: Có cấp phát thì có giải phóng, API này sẽ giải phóng vùng mem.
- ``efficient_mem_compact``: Chống phân mảnh pool.
- ``efficient_mem_reclaim``: Trả pool về cho general system khi không cần nữa.

**Mục đích của efficient memory allocator:**

- ``printk`` là module dùng để xuất các log từ kernel (print kernel), có thể bị deadlock khi interrupt handlers.
- Page fault handlers
- Reclaim/swap paths
- DMA completion handlers

Nếu không có logging, debugging memory là cực kì phức tạp.

**How it works:**

- Lụm lại memory từ pool dành cho việc boot sau khi hệ thống đã ổn định.
- Cấp phát bộ nhớ hiệu quả cho:
  - zswap buffer (nén swap từ RAM)
  - debug/tracing buffer (dùng để bắt các trạng thái của hệ thống)
  - Temporary staging areas (tạm thời cấp phát vùng nhớ cho pool)
- Lụm lại memory khi không cần nữa.

---

.. rubric:: Tài liệu tham khảo

- `Linux Kernel Documentation - Memory Management <https://www.kernel.org/doc/html/latest/mm/index.html>`_
- `Linux man pages <https://man7.org/linux/man-pages/>`_