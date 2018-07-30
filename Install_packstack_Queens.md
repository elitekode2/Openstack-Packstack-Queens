# Hương dẫn cài đặt OpenStack Queens bằng Packstack trên CENTOS 7.x


## 1. Các bước chuẩn bị
### 1.1. Giới thiệu

- Lưu ý: Trong tài liệu này chỉ thực hiện cài đặt OpenStack, bước cài đặt CEPH ở tài liệu khác.

- Packstack là một công cụ cài đặt OpenStack nhanh chóng.

- Packstack được phát triển bởi redhat

- Chỉ hỗ trợ các distro: RHEL, Centos

- Tự động hóa các bước cài đặt và lựa chọn thành phần cài đặt.

- Nhanh chóng dựng được môi trường OpenStack để sử dụng làm PoC nội bộ, demo khách hàng, test tính năng.

- Nhược điểm 1 : Đóng kín các bước cài đối với người mới.

- Nhược điểm 2: Khó bug các lỗi khi cài vì đã được đóng gói cùng với các tool cài đặt tự động (puppet)

### 1.2. Môi trường thực hiện 

- Distro: CentOS 7.x

- OpenStack Queens

- NIC1 - ens33: Là dải mạng mà các máy ảo sẽ giao tiếp với bên ngoài. Dải mạng này sử dụng chế độ bridge hoặc NAT của VMware Workstation

- NIC2 - ens34: là dải mạng sử dụng cho các traffic MGNT + API + DATA VM. Dải mạng này sử dụng chế độ hostonly trong VMware Workstation

### 1.3. Mô hình

 - Ở phần này sẽ gộp dải manager +API +DATA vào dải provider network và dải self service network riêng

<img src="/img/1.png">


### 2. Các bước chuẩn bị trên trên Controller


Thiết lập hostname

`hostnamectl set-hostname controller1`


* Thiết lập IP


``` sh
echo "Setup IP  ens34"
nmcli c modify ens34 ipv4.addresses 10.10.10.145/24
nmcli c modify ens34 ipv4.method manual
nmcli con mod ens34 connection.autoconnect yes

echo "Setup IP  ens33"
nmcli c modify ens33 ipv4.addresses 192.168.239.145/24
nmcli c modify ens33 ipv4.gateway 192.168.239.1
nmcli c modify ens33 ipv4.dns 8.8.8.8
nmcli c modify ens33 ipv4.method manual
nmcli con mod ens33 connection.autoconnect yes


sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

Khai báo repos cho OpenStack Queens

```
	yum install centos-release-openstack-queens.x86_64 -y

	sudo yum install -y wget crudini fping   (vi /etc/yum.repos.d/CentOS-QEMU-EV.repo sửa lại đường dẫn repos thay 2 dòng 'baseurl=http://mirror.centos.org/centos-7/7/virt/x86_64/kvm-common/' nếu repos báo lỗi sai)
	yum install -y openstack-packstack
	yum install -y epel-release
	sudo yum install -y byobu 
	init 6
	uname -m | grep -q 'x86_64'  && echo 'centos' >/etc/yum/vars/contentdir || echo 'altarch' >/etc/yum/vars/contentdir (Nếu cần thì chạy thêm dòng này)
```
  
Cài thêm gói yum install -y python-setuptools để fix nếu gặp lỗi dưới:

```
# packstack --gen-answer-file=answers.cfg
ERROR:root:Failed to load plugin from file ssl_001.py
ERROR:root:Traceback (most recent call last):
  File "/usr/lib/python2.7/site-packages/packstack/installer/run_setup.py", line 923, in loadPlugins
    moduleobj = __import__(moduleToLoad)
  File "/usr/lib/python2.7/site-packages/packstack/plugins/ssl_001.py", line 20, in <module>
    from OpenSSL import crypto
  File "/usr/lib/python2.7/site-packages/OpenSSL/__init__.py", line 8, in <module>
    from OpenSSL import rand, crypto, SSL
  File "/usr/lib/python2.7/site-packages/OpenSSL/crypto.py", line 13, in <module>
    from cryptography.hazmat.primitives.asymmetric import dsa, rsa
  File "/usr/lib64/python2.7/site-packages/cryptography/hazmat/primitives/asymmetric/rsa.py", line 14, in <module>
    from cryptography.hazmat.backends.interfaces import RSABackend
  File "/usr/lib64/python2.7/site-packages/cryptography/hazmat/backends/__init__.py", line 7, in <module>
    import pkg_resources
ImportError: No module named pkg_resources

ERROR:root:Traceback (most recent call last):
  File "/usr/lib/python2.7/site-packages/packstack/installer/run_setup.py", line 988, in main
    loadPlugins()
  File "/usr/lib/python2.7/site-packages/packstack/installer/run_setup.py", line 931, in loadPlugins
    raise Exception("Failed to load plugin from file %s" % item)
Exception: Failed to load plugin from file ssl_001.py


ERROR : Failed to load plugin from file ssl_001.py
```


### 2.2. Các bước chuẩn bị trên trên Compute1

Thiết lập hostname

`systemctl start NetworkManager`

`hostnamectl set-hostname compute1`
Thiết lập IP
``` sh
echo "Setup IP  ens33"
nmcli c modify ens33 ipv4.addresses 192.168.239.146/24
nmcli c modify ens33 ipv4.gateway 192.168.239.2
nmcli c modify ens33 ipv4.dns 8.8.8.8
nmcli c modify ens33 ipv4.method manual
nmcli c modify ens33 connection.autoconnect yes

echo "Setup IP  ens34"
nmcli c modify ens34 ipv4.addresses 10.10.10.146/24
nmcli c modify ens34 ipv4.method manual
nmcli c modify ens34 connection.autoconnect yes

sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```
Khai báo repos cho OpenStack Queens trên node Compute1

``` sh
yum install centos-release-openstack-queens.x86_64 -y
yum update -y

sudo yum install -y wget crudini fping
yum install -y openstack-packstack

yum install -y epel-release
sudo yum install -y byobu 
```

### 2.3. Các bước chuẩn bị trên trên Compute2
Thiết lập hostname

`hostnamectl set-hostname compute2`


*Thiết lập IP


``` sh
echo "Setup IP  ens33"
nmcli c modify ens33 ipv4.addresses 192.168.239.147/24
nmcli c modify ens33 ipv4.gateway 192.168.239.2
nmcli c modify ens33 ipv4.dns 8.8.8.8
nmcli c modify ens33 ipv4.method manual
nmcli c modify ens33 connection.autoconnect yes

echo "Setup IP  ens34"
nmcli c modify ens34 ipv4.addresses 10.10.10.147/24
nmcli c modify ens34 ipv4.method manual
nmcli c modify ens34 connection.autoconnect yes

sudo systemctl disable firewalld
sudo systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network

sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

Khai báo repos cho OpenStack Queens trên node Compute2


``` sh
yum install centos-release-openstack-queens.x86_64 -y
yum update -y

sudo yum install -y wget crudini fping
yum install -y openstack-packstack

yum install -y epel-release
sudo yum install -y byobu 
```

### 3. Cài đặt OpenStack Queens

### 3.1. Chuẩn bị file trả lời cho packstack

Đứng trên controller để thực hiện các bước sau:

Gõ lệnh dưới: `byobu`

Tạo file trả lời để cài packstack


``` sh
packstack packstack --gen-answer-file=/root/rdotraloi.txt \
    --allinone \
    --default-password=Welcome123 \
    --os-cinder-install=y \
    --os-ceilometer-install=y \
    --os-trove-install=n \
    --os-ironic-install=n \
    --os-swift-install=n \
    --os-panko-install=y \
    --os-heat-install=y \
    --os-magnum-install=n \
    --os-aodh-install=y \
    --os-neutron-ovs-bridge-mappings=extnet:br-ex \
    --os-neutron-ovs-bridge-interfaces=br-ex:ens35 \
    --os-neutron-ovs-bridges-compute=br-ex \
    --os-neutron-ml2-type-drivers=vxlan,flat \
    --os-controller-host=192.168.12.201 \
    --os-compute-hosts=192.168.12.202,192.168.12.203 \
    --os-neutron-ovs-tunnel-if=ens33 \
    --provision-demo=n
```
	
	
Thực thi file trả lời vừa tạo ở trên (nếu cần có thể mở ra để chỉnh lại các tham số cần thiết.

`packstack --answer-file rdotraloi.txt`

Sau khi cài đặt xong, màn hình sẽ hiển thị thông báo như dưới:

``` sh
**** Installation completed successfully ******

Additional information:
 * Time synchronization installation was skipped. Please note that unsynchronized time on server instances might be problem for some OpenStack components.
 * File /root/keystonerc_admin has been created on OpenStack client host 192.168.239.145. To use the command line tools you need to source the file.
 * To access the OpenStack Dashboard browse to http://192.168.239.145/dashboard .
Please, find your login credentials stored in the keystonerc_admin in your home directory.
 * The installation log file is available at: /var/tmp/packstack/20180730-100156-y15EAs/openstack-setup.log
 * The generated manifests are available at: /var/tmp/packstack/20180730-100156-y15EAs/manifests
[root@controller1 ~]#
```


Nhập mật khẩu đăng nhập ssh của tài khoản root khi được yêu cầu.


Đứng trên `Controller1` thực hiện lệnh dưới để sửa các cấu hình cần thiết.

``` sh
sed -i -e 's/enable_isolated_metadata=False/enable_isolated_metadata=True/g' /etc/neutron/dhcp_agent.ini

ssh -o StrictHostKeyChecking=no root@192.168.239.146 "sed -i -e 's/compute1/192.168.239.146/g' /etc/nova/nova.conf"

ssh -o StrictHostKeyChecking=no root@192.168.239.147 "sed -i -e 's/compute2/192.168.239.147/g' /etc/nova/nova.conf"
```

Tắt Iptables trên cả 03 node

``` sh
systemctl stop iptables
systemctl disable iptables

ssh -o StrictHostKeyChecking=no root@192.168.239.146 "systemctl stop iptables"
ssh -o StrictHostKeyChecking=no root@192.168.239.146 "systemctl disable iptables"

ssh -o StrictHostKeyChecking=no root@192.168.239.147 "systemctl stop iptables"
ssh -o StrictHostKeyChecking=no root@192.168.239.147 "systemctl disable iptables"
```

Khởi động lại cả 03 node `Controller1`, `Compute1`, `Compute2`.

``` sh
ssh -o StrictHostKeyChecking=no root@192.168.239.146 "init 6"

ssh -o StrictHostKeyChecking=no root@192.168.239.147 "init 6"

init 6
```

Đăng nhập lại vào `Controller1` bằng quyền` root` và kiểm tra hoạt động của openstack sau khi cài.

Khai báo biến môi trường:

`source keystonerc_admin`


Kiểm tra hoạt động của openstack bằng lệnh dưới (lưu ý: có thể phải mất vài phút để các service của OpenStack khởi động xong).

`openstack token issue`

``` sh
[root@controller1 ~(keystone_admin)]# openstack token issue
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2018-07-30T05:09:13+0000                                                                                                                                                                |
| id         | gAAAAABbXo9phyc_1REQCintFOSoJCIC57rH5d3tYK_p7gHfOKwDMBuOmpwPWDgNdpa775KNWat4CD6KpZ7EvQo3h8i_PlBSER1q8Kfvax3ff3O02aA1BkY3EM55jD1azdDyYIIspr0N89h6tPq6l7Bqw_3Gsvce0U-_0ar6PZ2Z8uu3JTdZNrU |
| project_id | 196c9ee913de4a81a568b30a455e78eb                                                                                                                                                        |
| user_id    | 47ecbfe6a82143fe930b917b82f830e3                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

Kết quả lệnh trên như sau:




``` sh
[root@controller1 ~(keystone_admin)]# openstack user list
+----------------------------------+------------+
| ID                               | Name       |
+----------------------------------+------------+
| 0ad92a3b960b4344ba3e841e411841c2 | panko      |
| 1d8736352c704ebbb4b30d3c1e9fd01c | placement  |
| 1eece16bbe764388ab4502dec8ee81af | ceilometer |
| 1f2be96d87e84252bc6f963c17d1af3c | aodh       |
| 2200f95d835b452d884dc4206dad0c78 | cinder     |
| 24a1f7b9d44b401786455bf62d965f0a | heat_admin |
| 2cc09f5ed1ee4b8daaf6328c9bd0b674 | gnocchi    |
| 3cf66cc2db7d47fbbf39c43753a4011f | nova       |
| 86f1a9ee2bd04db7ba39456e13c16400 | heat-cfn   |
| d807b87a5e164e8ebadabdc82725b080 | admin      |
| dc074540433f499c9c130f7037d82229 | glance     |
| dec7389046f2480398c9dee979781163 | heat       |
| f7f1bdd3a6954219a822900bb37cf8ac | neutron    |
+----------------------------------+------------+
```

### 4. Tạo images, network, subnet, router, mở security group và tạo VM.


### 4.1. Tạo images

Đăng nhập vào node `controller1` với quyền root và thực thi các lệnh sau:

Tải images cirros. Images này dùng để tạo các máy ảo sau này:

`wget http://download.cirros-cloud.net/0.3.5/cirros-0.3.5-x86_64-disk.img`

Tạo images:

``` sh
[root@controller1 ~(keystone_admin)]# openstack image create "cirros" \
>   --file cirros-0.3.5-x86_64-disk.img \
>   --disk-format qcow2 --container-format bare \
>   --public
+------------------+------------------------------------------------------+
| Field            | Value                                                |
+------------------+------------------------------------------------------+
| checksum         | f8ab98ff5e73ebab884d80c9dc9c7290                     |
| container_format | bare                                                 |
| created_at       | 2018-07-30T04:16:18Z                                 |
| disk_format      | qcow2                                                |
| file             | /v2/images/c41b30db-3cb9-467f-8d48-fc07323537bb/file |
| id               | c41b30db-3cb9-467f-8d48-fc07323537bb                 |
| min_disk         | 0                                                    |
| min_ram          | 0                                                    |
| name             | cirros                                               |
| owner            | 196c9ee913de4a81a568b30a455e78eb                     |
| protected        | False                                                |
| schema           | /v2/schemas/image                                    |
| size             | 13267968                                             |
| status           | active                                               |
| tags             |                                                      |
| updated_at       | 2018-07-30T04:16:18Z                                 |
| virtual_size     | None                                                 |
| visibility       | public                                               |
+------------------+------------------------------------------------------+
[root@controller1 ~(keystone_admin)]# 
```

Kiểm tra việc tạo images, kết quả  :

``` sh
[root@controller1 ~(keystone_admin)]# openstack image list
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| c41b30db-3cb9-467f-8d48-fc07323537bb | cirros | active |
+--------------------------------------+--------+--------+
[root@controller1 ~(keystone_admin)]#
```




































































Tài liệu tham khảo :
https://github.com/congto/ghichep-rdo/blob/master/docs/packstack_OpenStack_Queens_VNPT_LAB.md