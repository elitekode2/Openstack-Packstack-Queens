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

Thiết lập IP
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


2.2. Các bước chuẩn bị trên trên Compute1

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
sudo yum install -y centos-release-openstack-queens
yum update -y

sudo yum install -y wget crudini fping
yum install -y openstack-packstack

yum install -y epel-release
sudo yum install -y byobu 
```

2.3. Các bước chuẩn bị trên trên Compute2
Thiết lập hostname

`hostnamectl set-hostname compute2`
Thiết lập IP
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
sudo yum install -y centos-release-openstack-queens 
yum update -y

sudo yum install -y wget crudini fping
yum install -y openstack-packstack

yum install -y epel-release
sudo yum install -y byobu 
```

3. Cài đặt OpenStack Queens
3.1. Chuẩn bị file trả lời cho packstack
Đứng trên controller để thực hiện các bước sau

Gõ lệnh dưới

byobu
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
Nhập mật khẩu đăng nhập ssh của tài khoản root khi được yêu cầu.

Chờ để packstack cài đặt xong.




ssh -o StrictHostKeyChecking=no root@192.168.12.202 "sed -i -e 's/compute1/192.168.12.202/g' /etc/nova/nova.conf"


ssh -o StrictHostKeyChecking=no root@192.168.12.203 "sed -i -e 's/compute2/192.168.12.203/g' /etc/nova/nova.conf"




ssh -o StrictHostKeyChecking=no root@192.168.12.202 "systemctl stop iptables"
ssh -o StrictHostKeyChecking=no root@192.168.12.202 "systemctl disable iptables"

ssh -o StrictHostKeyChecking=no root@192.168.12.203 "systemctl stop iptables"
ssh -o StrictHostKeyChecking=no root@192.168.12.203 "systemctl disable iptables"


ssh -o StrictHostKeyChecking=no root@192.168.12.202 "init 6"

ssh -o StrictHostKeyChecking=no root@192.168.12.203 "init 6"

init 6








openstack subnet create --network provider \
  --allocation-pool start=192.168.12.210,end=192.168.12.250 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.239.1 \
  --subnet-range 192.168.12.0/24 provider



openstack subnet create --network selfservice \
  --dns-nameserver 8.8.8.8 --gateway 10.10.10.1 \
  --subnet-range 10.10.10.0/24 selfservice





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
[root@controller1 ~(keystone_admin)]# neutron net-list
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.

[root@controller1 ~(keystone_admin)]# openstack subnet create --network provider \
>   --allocation-pool start=192.168.239.210,end=192.168.239.250 \
>   --dns-nameserver 8.8.8.8 --gateway 192.168.239.1 \
>   --subnet-range 192.168.239.0/24 provider
No Network found for provider
[root@controller1 ~(keystone_admin)]# openstack network create  --share --external --provider-physical-network provider --provider-network-type flat provider
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-07-29T11:02:53Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | b2c2c68f-e9b3-4a26-8537-411a46d3f21a |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1500                                 |
| name                      | provider                             |
| port_security_enabled     | True                                 |
| project_id                | 4a9cff0aed4a447b87bbdbcd2d3663f0     |
| provider:network_type     | flat                                 |
| provider:physical_network | provider                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 5                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-07-29T11:02:54Z                 |
+---------------------------+--------------------------------------+
[root@controller1 ~(keystone_admin)]# openstack subnet create --network selfservice \
>   --dns-nameserver 8.8.8.8 --gateway 10.10.10.1 \
>   --subnet-range 10.10.10.0/24 selfservice
No Network found for selfservice
[root@controller1 ~(keystone_admin)]# openstack network create selfservice
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2018-07-29T11:04:37Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | 2b8eb80f-37de-43dc-8cdc-cb535c9a3352 |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| is_vlan_transparent       | None                                 |
| mtu                       | 1450                                 |
| name                      | selfservice                          |
| port_security_enabled     | True                                 |
| project_id                | 4a9cff0aed4a447b87bbdbcd2d3663f0     |
| provider:network_type     | vxlan                                |
| provider:physical_network | None                                 |
| provider:segmentation_id  | 65                                   |
| qos_policy_id             | None                                 |
| revision_number           | 2                                    |
| router:external           | Internal                             |
| segments                  | None                                 |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tags                      |                                      |
| updated_at                | 2018-07-29T11:04:37Z                 |
+---------------------------+--------------------------------------+
[root@controller1 ~(keystone_admin)]# openstack subnet create --network selfservice   --dns-nameserver 8.8.8.8 --gateway 10.10.10.1   --subnet-range 10.10.10.0/24 selfservice
+-------------------+-----------------------------------------------+
| Field             | Value                                         |
+-------------------+-----------------------------------------------+
| allocation_pools  | 10.10.10.1-10.10.10.1,10.10.10.3-10.10.10.254 |
| cidr              | 10.10.10.0/24                                 |
| created_at        | 2018-07-29T11:04:47Z                          |
| description       |                                               |
| dns_nameservers   | 8.8.8.8                                       |
| enable_dhcp       | True                                          |
| gateway_ip        | 10.10.10.1                                    |
| host_routes       |                                               |
| id                | a7155084-a815-4008-966f-56cd636df57c          |
| ip_version        | 4                                             |
| ipv6_address_mode | None                                          |
| ipv6_ra_mode      | None                                          |
| name              | selfservice                                   |
| network_id        | 2b8eb80f-37de-43dc-8cdc-cb535c9a3352          |
| project_id        | 4a9cff0aed4a447b87bbdbcd2d3663f0              |
| revision_number   | 0                                             |
| segment_id        | None                                          |
| service_types     |                                               |
| subnetpool_id     | None                                          |
| tags              |                                               |
| updated_at        | 2018-07-29T11:04:47Z                          |
+-------------------+-----------------------------------------------+
[root@controller1 ~(keystone_admin)]# openstack router create router
+-------------------------+--------------------------------------+
| Field                   | Value                                |
+-------------------------+--------------------------------------+
| admin_state_up          | UP                                   |
| availability_zone_hints |                                      |
| availability_zones      |                                      |
| created_at              | 2018-07-29T11:05:31Z                 |
| description             |                                      |
| distributed             | False                                |
| external_gateway_info   | None                                 |
| flavor_id               | None                                 |
| ha                      | False                                |
| id                      | 43b45c4e-264a-4b3d-9b71-009703300c7b |
| name                    | router                               |
| project_id              | 4a9cff0aed4a447b87bbdbcd2d3663f0     |
| revision_number         | 1                                    |
| routes                  |                                      |
| status                  | ACTIVE                               |
| tags                    |                                      |
| updated_at              | 2018-07-29T11:05:32Z                 |
+-------------------------+--------------------------------------+
[root@controller1 ~(keystone_admin)]# neutron router-interface-add router selfservice
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Added interface cf1db525-f954-4468-8a02-4c7bb2b3f97b to router router.
[root@controller1 ~(keystone_admin)]# neutron router-gateway-set router provider
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
Set gateway for router router
[root@controller1 ~(keystone_admin)]# ip netns
qrouter-43b45c4e-264a-4b3d-9b71-009703300c7b (id: 1)
qdhcp-2b8eb80f-37de-43dc-8cdc-cb535c9a3352 (id: 0)
[root@controller1 ~(keystone_admin)]# neutron router-port-list router
neutron CLI is deprecated and will be removed in the future. Use openstack CLI instead.
+--------------------------------------+------+----------------------------------+-------------------+-----------------------------------------------------------------------------------+
| id                                   | name | tenant_id                        | mac_address       | fixed_ips                                                                         |
+--------------------------------------+------+----------------------------------+-------------------+-----------------------------------------------------------------------------------+
| bb254013-5ddc-49a4-818c-0ed3c981354b |      |                                  | fa:16:3e:2a:f3:48 |                                                                                   |
| cf1db525-f954-4468-8a02-4c7bb2b3f97b |      | 4a9cff0aed4a447b87bbdbcd2d3663f0 | fa:16:3e:67:c8:a7 | {"subnet_id": "a7155084-a815-4008-966f-56cd636df57c", "ip_address": "10.10.10.2"} |
+--------------------------------------+------+----------------------------------+-------------------+-----------------------------------------------------------------------------------+
[root@controller1 ~(keystone_admin)]# openstack security group rule create --proto icmp default
More than one SecurityGroup exists with the name 'default'.
[root@controller1 ~(keystone_admin)]# openstack security group list
+--------------------------------------+---------+------------------------+----------------------------------+
| ID                                   | Name    | Description            | Project                          |
+--------------------------------------+---------+------------------------+----------------------------------+
| 07362a7d-96b7-4a5e-b10b-b22c0f766002 | default | Default security group |                                  |
| 6e12c898-b2b8-4aa7-9aa8-49cba279380f | default | Default security group | 4a9cff0aed4a447b87bbdbcd2d3663f0 |
+--------------------------------------+---------+------------------------+----------------------------------+
[root@controller1 ~(keystone_admin)]# openstack security group rule create --proto icmp 6e12c898-b2b8-4aa7-9aa8-49cba279380f
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2018-07-29T11:07:52Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | 2de89574-b98a-45e4-8867-5138b8ff899c |
| name              | None                                 |
| port_range_max    | None                                 |
| port_range_min    | None                                 |
| project_id        | 4a9cff0aed4a447b87bbdbcd2d3663f0     |
| protocol          | icmp                                 |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 0                                    |
| security_group_id | 6e12c898-b2b8-4aa7-9aa8-49cba279380f |
| updated_at        | 2018-07-29T11:07:52Z                 |
+-------------------+--------------------------------------+
[root@controller1 ~(keystone_admin)]# openstack security group rule create --proto tcp --dst-port 22 6e12c898-b2b8-4aa7-9aa8-49cba279380f
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| created_at        | 2018-07-29T11:08:08Z                 |
| description       |                                      |
| direction         | ingress                              |
| ether_type        | IPv4                                 |
| id                | 6a01e082-295b-44ab-af6a-b840d888795c |
| name              | None                                 |
| port_range_max    | 22                                   |
| port_range_min    | 22                                   |
| project_id        | 4a9cff0aed4a447b87bbdbcd2d3663f0     |
| protocol          | tcp                                  |
| remote_group_id   | None                                 |
| remote_ip_prefix  | 0.0.0.0/0                            |
| revision_number   | 0                                    |
| security_group_id | 6e12c898-b2b8-4aa7-9aa8-49cba279380f |
| updated_at        | 2018-07-29T11:08:08Z                 |
+-------------------+--------------------------------------+
[root@controller1 ~(keystone_admin)]#




openstack subnet create --network provider \
  --allocation-pool start=192.168.239.204,end=192.168.239.250 \
  --dns-nameserver 8.8.8.8 --gateway 192.168.239.2 \
  --subnet-range 192.168.239.0/24 provider












































































Tài liệu tham khảo :
https://github.com/congto/ghichep-rdo/blob/master/docs/packstack_OpenStack_Queens_VNPT_LAB.md