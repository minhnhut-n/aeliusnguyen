Networking
==========

- Ethernet
- Wi-Fi
- Bluetooth
- DHCP
- DNS
- SSH
- FTP
- NFS

**1. Ethernet**

Banana Pi có cổng Ethernet 10/100/1000 Mbps (tùy model) sử dụng chip
Realtek hoặc integrated MAC + external PHY.

.. code-block:: bash

   # Kiểm tra interface
   $ ip link show
   $ ifconfig -a

   # Bật/tắt Ethernet
   $ sudo ip link set eth0 up
   $ sudo ip link set eth0 down

   # Gán IP tĩnh
   $ sudo ip addr add 192.168.1.100/24 dev eth0
   $ sudo ip route add default via 192.168.1.1

   # Xem thông tin chi tiết Ethernet (link speed, duplex)
   $ ethtool eth0

   # Test kết nối
   $ ping -c 4 google.com

**2. Wi-Fi**

Wi-Fi trên Banana Pi thường dùng chip USB Wi-Fi hoặc module onboard (như
AP6210 trên Banana Pi Pro).

.. code-block:: bash

   # Quét mạng Wi-Fi
   $ sudo iwlist wlan0 scan

   # Kết nối với WPA/WPA2
   $ sudo nano /etc/wpa_supplicant/wpa_supplicant.conf
   # Thêm:
   # network={
   #     ssid="TEN_WIFI"
   #     psk="mat_khau"
   # }

   $ sudo wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
   $ sudo dhclient wlan0  # Lấy IP

   # Hoặc dùng nmcli (NetworkManager) cho dễ
   $ sudo nmcli dev wifi connect "TEN_WIFI" password "mat_khau"

**3. Bluetooth**

Bluetooth trên Banana Pi thường đi kèm với module Wi-Fi (combo chip).

.. code-block:: bash

   # Kiểm tra Bluetooth adapter
   $ hciconfig -a
   $ bluetoothctl

   # Scan devices
   $ bluetoothctl scan on

   # Pair với device
   $ bluetoothctl pair <MAC_ADDR>
   $ bluetoothctl connect <MAC_ADDR>

**4. DHCP (Dynamic Host Configuration Protocol)**

DHCP tự động cấp IP address, gateway, DNS cho client. Trên embedded Linux,
DHCP client phổ biến là ``dhclient`` (ISC) hoặc ``dhcpcd`` (nhẹ hơn).

.. code-block:: bash

   # Lấy IP tự động từ DHCP server
   $ sudo dhclient eth0

   # Renew IP
   $ sudo dhclient -r eth0   # Release
   $ sudo dhclient eth0      # Renew

   # Cấu hình DHCP tĩnh trong /etc/dhcpcd.conf
   $ cat /etc/dhcpcd.conf
   # interface eth0
   # static ip_address=192.168.1.100/24
   # static routers=192.168.1.1
   # static domain_name_servers=8.8.8.8

**5. DNS (Domain Name System)**

DNS chuyển đổi tên miền (google.com) thành IP address. File cấu hình DNS
là ``/etc/resolv.conf``.

.. code-block:: bash

   # Xem DNS servers
   $ cat /etc/resolv.conf

   # Test DNS resolution
   $ nslookup google.com
   $ dig google.com
   $ host google.com

   # Nếu không có DNS, bạn có thể ping trực tiếp bằng IP:
   $ ping 8.8.8.8  # Google DNS

**6. SSH (Secure Shell)**

SSH là giao thức "cứu cánh" cho embedded Linux vì board thường không có
màn hình. Bạn SSH vào board từ máy tính để làm việc.

.. code-block:: bash

   # Trên board Banana Pi - cài SSH server
   $ sudo apt-get install openssh-server
   $ sudo systemctl enable ssh
   $ sudo systemctl start ssh

   # Kiểm tra SSH đang chạy
   $ sudo systemctl status ssh
   $ ss -tulpn | grep :22

   # Từ máy tính - SSH vào board
   $ ssh pi@192.168.1.100
   # Mặc định password: bananapi

   # Copy SSH key để không cần nhập password
   $ ssh-copy-id pi@192.168.1.100

   # Config SSH cho bảo mật (không cho phép root login)
   $ sudo nano /etc/ssh/sshd_config
   # PermitRootLogin no

**7. FTP (File Transfer Protocol)**

FTP dùng để truyền file giữa máy tính và board. Tuy nhiên, FTP không mã
hóa nên ngày nay ít dùng, thay bằng SFTP (SSH-based).

.. code-block:: bash

   # Cài FTP server (vsftpd)
   $ sudo apt-get install vsftpd
   $ sudo systemctl start vsftpd

   # Hoặc dùng SFTP (qua SSH) - khuyên dùng
   $ sftp pi@192.168.1.100
   sftp> put localfile.txt
   sftp> get remotefile.txt
   sftp> ls
   sftp> exit

**8. NFS (Network File System)**

NFS cho phép mount một thư mục từ máy tính (host) vào board (target) qua
mạng. Rất hữu ích cho development: bạn edit code trên máy tính, board chạy
trực tiếp mà không cần copy file.

.. code-block:: bash

   # Trên máy tính (host) - share thư mục
   $ sudo apt-get install nfs-kernel-server
   $ sudo nano /etc/exports
   # /path/to/rootfs 192.168.1.0/24(rw,no_root_squash,no_subtree_check)
   $ sudo exportfs -a
   $ sudo systemctl restart nfs-kernel-server

   # Trên board Banana Pi - mount NFS
   $ sudo mount -t nfs 192.168.1.100:/path/to/rootfs /mnt/nfs
   $ ls /mnt/nfs

   # NFS root boot (kernel boot thẳng từ NFS, không cần SD card)
   # Cấu hình trong U-Boot:
   # setenv bootargs console=ttyS0,115200 root=/dev/nfs nfsroot=192.168.1.100:/path/to/rootfs ip=192.168.1.100
