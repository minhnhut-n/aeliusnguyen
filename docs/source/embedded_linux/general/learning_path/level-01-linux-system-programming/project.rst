Project
=======

- ✅ Viết mini shell
- ✅ TCP server
- ✅ UART terminal

**Mini shell**

Viết một shell đơn giản sử dụng ``fork()`` + ``exec()`` để thực thi lệnh,
hỗ trợ pipe (``|``) và redirection (``>``, ``<``).

.. code-block:: c

   // Pseudocode
   while (1) {
       print_prompt();
       read_command(cmd);
       if (fork() == 0) {
           // Child: thực thi lệnh
           execvp(cmd.argv[0], cmd.argv);
       } else {
           // Parent: chờ child kết thúc
           wait(NULL);
       }
   }

**TCP server**

Viết TCP server sử dụng socket API, hỗ trợ multiple clients với
``epoll()`` hoặc ``pthread``.

.. code-block:: c

   // Pseudocode
   int server_fd = socket(AF_INET, SOCK_STREAM, 0);
   bind(server_fd, ...);
   listen(server_fd, 5);

   while (1) {
       int client_fd = accept(server_fd, ...);
       // Xử lý client trong thread riêng
       pthread_create(&tid, NULL, handle_client, &client_fd);
   }

**UART terminal**

Viết chương trình giao tiếp UART sử dụng ``open()``, ``read()``,
``write()``, ``ioctl()`` trên ``/dev/ttyUSB0`` hoặc ``/dev/ttyS0``.

.. code-block:: c

   // Pseudocode
   int uart_fd = open("/dev/ttyUSB0", O_RDWR | O_NOCTTY);
   struct termios tty;
   tcgetattr(uart_fd, &tty);
   cfsetospeed(&tty, B115200);
   cfsetispeed(&tty, B115200);
   tcsetattr(uart_fd, TCSANOW, &tty);

   write(uart_fd, "AT\r\n", 4);
   char resp[256];
   read(uart_fd, resp, sizeof(resp));
