Những project mình khuyên làm trên chính BPI-M4
===============================================

- Hello Kernel Module – Làm quen với module, insmod, rmmod, printk.
- GPIO LED Driver – Điều khiển LED bằng character driver.
- Button Interrupt Driver – Xử lý ngắt phần cứng và debounce.
- UART Driver/User-space Tool – Giao tiếp với thiết bị ngoại vi.
- Platform Driver + Device Tree – Tạo node mới trong DTS và viết driver probe().
- I2C Sensor Driver – Kết nối một cảm biến (ví dụ MPU6050 hoặc BH1750) và viết driver.
- SPI OLED/LCD Driver – Điều khiển màn hình nhỏ qua SPI.
- DMA Demo – Tìm hiểu truyền dữ liệu bằng DMA nếu BSP hỗ trợ.
- Build Kernel từ Source – Thêm một CONFIG\_* mới, build và boot.
- Chỉnh sửa Device Tree – Thay đổi GPIO, UART hoặc reserved-memory và quan sát ảnh hưởng.
- Buildroot – Tự tạo một root filesystem tối giản và boot được trên BPI-M4.
- Khởi động bằng kernel và DTB tự build – Đây là bước chuyển từ "người dùng BSP" sang "người phát triển BSP".
