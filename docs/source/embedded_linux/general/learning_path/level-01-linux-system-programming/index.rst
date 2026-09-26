Level 1 – Linux System Programming
===================================

Ngôn ngữ C là ngôn ngữ chính cho toàn bộ kernel module, device driver và các ứng dụng hệ thống - không có ngoại lệ. Kiến thức cơ bản về lập trình C là cần thiết để phát triển các ứng dụng Linux nhúng, vì C cho phép tương tác trực tiếp với phần cứng và đơn giản nhất để con người đọc và hiểu được.

Có bạn hỏi tại sao lại không phải là C++? Vì C++ là ngôn ngữ hướng đối tượng, trong khi C lại nghiêng về thiết kế và tối ưu hệ thống. C++ có thể có nhiều class "rác" không tận dụng 100% tài nguyên, hoặc do tính chất của OOP được dùng làm future feature.

Đồng ý rằng code C trong kernel phải được thiết kế theo nguyên tắc tường minh, dễ đọc, dễ mở rộng và tối ưu.

.. toctree::
   :maxdepth: 2
   :caption: Nội dung bài học
   :titlesonly:

   system-calls
   ipc
   posix
   project

.. include:: ../../../../_includes/contact_info.rst
