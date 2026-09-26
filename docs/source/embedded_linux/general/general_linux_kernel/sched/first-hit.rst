A First Hit - Introduction
==========================

Scheduler và bài toán điều phối trong hệ điều hành
---------------------------------------------------

.. rubric:: 1. Nếu hệ thống không có scheduler

Nếu một hệ thống không có scheduler, về cơ bản hệ thống sẽ gặp một số vấn đề rất lớn.

Đầu tiên, hệ thống có xu hướng làm việc kiểu tuần tự. Các block công việc chèn lên nhau, task này phải chờ task kia hoàn thành, từ đó tạo ra hiện tượng chờ giữa các task và làm lãng phí tài nguyên.

Tiếp theo, hệ thống phải dựa rất nhiều vào cơ chế interrupt để xử lý các sự kiện và đánh thức những phần công việc cần chạy. Nhưng khi có rất nhiều task hoặc event cần được xử lý tại cùng một thời điểm, vấn đề lại xuất hiện: hệ thống phải quyết định task nào được chạy trước, task nào phải chờ, tài nguyên nào được phép truy cập.

Nếu không có một cơ chế điều phối tốt, hệ thống rất dễ rơi vào tình trạng quá tải, I/O congestion hoặc tạo ra những hành vi không mong muốn như race condition.


.. rubric:: 2. Bài toán interrupt, preemption và thứ tự thực thi

Ngay cả khi đã giải quyết được hai vấn đề trên, hệ thống vẫn còn một bài toán khác: sau khi một task bị interrupt hoặc bị preempt, khi nào nó được chạy lại và thứ tự thực thi giữa các task sẽ được duy trì như thế nào?

Nếu việc chia sẻ tài nguyên và thứ tự thực thi không được kiểm soát tốt, chúng ta lại có thể gặp race condition, starvation, priority inversion hoặc các vấn đề về timing.


.. rubric:: 3. Các cơ chế giải quyết bài toán làm việc tuần tự giữa các task

Để giải quyết bài toán làm việc tuần tự giữa các task, rất nhiều cơ chế đã được sinh ra.

Ví dụ, multithreading và multiprocessing cho phép hệ thống có nhiều execution context hoạt động đồng thời.

Trên hệ thống multicore, nhiều thread có thể thực sự chạy song song trên nhiều CPU core; còn trên một core, scheduler có thể chuyển đổi giữa các thread để tạo ra concurrency.

Sau đó là các cơ chế synchronization như mutex, semaphore, spinlock, atomic operation, condition/wait queue, v.v. Chúng giúp kiểm soát việc nhiều task cùng truy cập vào một tài nguyên dùng chung, từ đó hạn chế race condition và giúp hệ thống kiểm soát được việc đồng bộ hóa.

Tuy nhiên, nếu sử dụng sai, chính những cơ chế này cũng có thể dẫn đến deadlock, starvation hoặc priority inversion.


.. rubric:: 4. Phần cốt lõi phía sau các cơ chế

Nhưng về cơ bản, những cơ chế trên mới chỉ là phần mà chúng ta nhìn thấy ở bên ngoài.

Đằng sau chúng còn có cả một hệ thống rất lớn chịu trách nhiệm điều phối và quản lý tài nguyên. Đó mới là phần cốt lõi khiến một hệ thống có thể được gọi là operating system.

Nhìn một cách tổng quát, OS có thể tồn tại dưới nhiều dạng, ví dụ như RTOS hoặc GPOS.

Nhưng đối với một hệ thống có rất nhiều process, thread và workload nặng, một trong những thành phần cốt lõi chính là scheduler.


.. rubric:: 5. Scheduler làm gì?

Scheduler chịu trách nhiệm quyết định task nào được chạy, khi nào được chạy, chạy trong bao lâu và CPU nào sẽ thực thi task đó.

Nó phân bổ CPU time cho các task, xử lý việc preemption, context switching và phối hợp với các cơ chế khác để giữ cho hệ thống có thể đáp ứng được nhiều workload cùng lúc.

Từ đây, scheduler có thể giúp hệ thống đạt được nhiều mục tiêu khác nhau:

* tăng throughput;
* giảm latency;
* cải thiện responsiveness;
* tận dụng nhiều CPU core tốt hơn;
* trong một số trường hợp còn giúp giảm mức tiêu thụ năng lượng.

Chính những cơ chế này là một trong những lý do một hệ điều hành hiện đại có thể duy trì trải nghiệm mượt mà ngay cả khi hệ thống đang chịu heavy load.


.. rubric:: 6. Scheduler không phải là một "cỗ máy duy nhất"

Dĩ nhiên, scheduler chỉ là tên gọi chung cho cả một nhóm cơ chế, chứ bản thân scheduler không phải là một "cỗ máy duy nhất" giải quyết tất cả mọi vấn đề.

Bên dưới nó còn có rất nhiều cơ chế khác như:

* runqueue;
* scheduling policy;
* context switching;
* preemption;
* timer;
* CPU affinity;
* load balancing;
* synchronization;
* và nhiều thành phần khác phối hợp với nhau.


.. rubric:: 7. Các thành phần vừa độc lập vừa liên kết chặt chẽ

Điều thú vị là những thành phần này vừa được tách biệt để có thể dễ dàng phát triển và bảo trì, nhưng đồng thời chúng lại liên kết rất chặt chẽ với nhau.

Nó giống kiểu "tuy hai mà một, tuy một mà hai" hahhah.

Không thể tách chúng hoàn toàn thành những hệ thống độc lập, nhưng cũng không thể gom tất cả lại thành một đống code khổng lồ.

Nếu làm vậy thì mỗi lần sửa một thứ lại phải nghĩ xem mình đang ảnh hưởng đến cái gì, gọi cái nào trước, cái nào phụ thuộc cái nào, và cuối cùng code sẽ trở thành một đống bùi nhùi gối đầu nhau.


.. rubric:: 8. Cách Linux tổ chức các subsystem

Vì vậy, cái hay của một OS như Linux không chỉ nằm ở việc nó có scheduler, mà nằm ở cách rất nhiều cơ chế được thiết kế thành những subsystem tương đối độc lập.

Mỗi subsystem giải quyết một bài toán riêng, nhưng vẫn có thể phối hợp với nhau để tạo thành một hệ thống thống nhất.