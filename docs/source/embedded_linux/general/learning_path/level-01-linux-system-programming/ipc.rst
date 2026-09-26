IPC (Inter-Process Communication)
=================================

- pipe
- shared memory
- semaphore
- message queue

IPC là tập hợp các phương thức truyền tải dữ liệu giữa các process trên
Linux system, trong đó các services/process có thể gửi dữ liệu trực tiếp
cho nhau thông qua các kênh truyền dẫn.

Có hai cách chính để chia sẻ thông tin dựa trên IPC:

- Gửi tin nhắn (message) thông qua PIPE
- Thiết lập memory sharing (dùng chung)

**pipe**

Giao thức đường ống, tưởng tượng như dòng chảy dầu khí/nước.

.. code-block:: c

   #include <unistd.h>

   int pipefd[2];
   pipe(pipefd);  // pipefd[0] = read end, pipefd[1] = write end

   if (fork() == 0) {
       // Child process: đọc từ pipe
       close(pipefd[1]);
       char buf[256];
       read(pipefd[0], buf, sizeof(buf));
       close(pipefd[0]);
   } else {
       // Parent process: ghi vào pipe
       close(pipefd[0]);
       write(pipefd[1], "Hello from parent", 18);
       close(pipefd[1]);
   }

Ưu điểm:

- Đơn giản
- Nhanh chóng
- An toàn

Hạn chế:

- Lưu lượng giới hạn
- Chỉ truyền được một chiều (half-duplex)
- Không có cơ chế đồng bộ

**shared memory**

Cho phép một vùng nhớ được cùng truy cập bởi nhiều process cùng lúc.

.. code-block:: c

   #include <sys/shm.h>

   key_t key = ftok("/tmp", 'A');
   int shmid = shmget(key, 1024, IPC_CREAT | 0666);
   void *data = shmat(shmid, NULL, 0);

   // Ghi dữ liệu vào shared memory
   strcpy((char *)data, "Shared data");

   shmdt(data);  // Detach

Ưu điểm:

- Tiết kiệm bộ nhớ (không cần lưu riêng rẽ, phân mảnh cùng 1 dữ liệu tại
  nhiều local space của process)
- Truy cập vùng nhớ dễ dàng giữa các data khi setup
- Chia sẻ dữ liệu nhanh hơn IPC, tránh được overhead (write in wait)

Hạn chế:

- Khó implement, vì các process cần đồng bộ và quyền truy cập khác nhau
- Memory leak khi last hold process không free đúng cách
- Over wait ⇒ Deadlock (cần tracking bằng watchdog timer)

**semaphore**

Cờ hiệu, để giải quyết vấn đề đồng bộ của shared memory. Semaphore là
một cờ dạng int giúp Linux đồng bộ giữa các process.

.. code-block:: c

   #include <sys/sem.h>

   key_t key = ftok("/tmp", 'B');
   int semid = semget(key, 1, IPC_CREAT | 0666);
   semctl(semid, 0, SETVAL, 1);  // Khởi tạo giá trị = 1

   struct sembuf sb;
   sb.sem_num = 0;
   sb.sem_op = -1;  // Wait (P) - giảm semaphore
   semop(semid, &sb, 1);

   // Truy cập critical section...

   sb.sem_op = 1;   // Signal (V) - tăng semaphore
   semop(semid, &sb, 1);

Semaphore cung cấp cơ chế kiểm soát đơn giản:

- Tối ưu hóa IPC
- Tránh được race condition, sử dụng quá mức resource (excessive resource
  utilization)

Giá trị của một semaphore thường có 2 chiều:

- Tăng = signal/post
- Giảm = wait/acquire

Với positive int, process được cho phép truy cập vào space để thao tác
với vùng nhớ. Ngược lại, negative int, process phải chờ do space đang
bị block/busy.

**message queue**

Phương thức truyền tải dữ liệu IPC phổ biến nhất, thường được triển khai
trên các service đặc biệt là application layer (platform).

Message queue là một chuỗi dữ liệu được lưu trữ dưới dạng linked-list
và lưu trong kernel. Cấu trúc dữ liệu này theo kiểu FIFO và các tin nhắn
được xác định dựa trên Message ID.

.. code-block:: c

   #include <sys/msg.h>

   key_t key = ftok("/tmp", 'C');
   int msqid = msgget(key, IPC_CREAT | 0666);

   struct msgbuf {
       long mtype;       // Message type (> 0)
       char mtext[256];  // Message data
   } msg;

   // Gửi message
   msg.mtype = 1;
   strcpy(msg.mtext, "Hello via message queue");
   msgsnd(msqid, &msg, sizeof(msg.mtext), 0);

   // Nhận message
   msgrcv(msqid, &msg, sizeof(msg.mtext), 1, 0);

   // Xóa message queue
   msgctl(msqid, IPC_RMID, NULL);

Các API chính:

- ``ftok()`` → tạo unique key
- ``msgget()`` → lấy hoặc tạo mới message queue
- ``msgsnd()`` → thêm message vào cuối chuỗi
- ``msgrcv()`` → truy xuất messages từ một chuỗi
- ``msgctl()`` → delete message
