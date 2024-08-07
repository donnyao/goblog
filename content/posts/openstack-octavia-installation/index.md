+++
author = "kiboud"
title = "OpenStack: LBaaS with Octavia"
date = "2020-11-20"
description = "This post will add LBaaS (Load Balancing as a Service) to the OpenStack cluster."
tags = [
    "opensource",
    "octavia",
]
categories = [
    "OpenStack",
]
series = ["OpenStack"]
image = "images/openstack-octavia.jpg"
+++
Kali ini kita akan menambahkan service loadbalancer pada Openstack. Disini saya menggunakan kolla-ansible untuk mendeploy service Octavia. Tutorial ini menggunakan versi Openstack Ussuri pada Ubuntu 18.04 dengan deployment tool Kolla-Ansible.



## Prerequisite
- Openstack Cluster

## Deploy Octavia
> **Note**: Jalankan perintah dibawah ini pada deployer node

### Generate certificate untuk service octavia

- Download octavia repository 
```bash
cd ~
git clone https://opendev.org/openstack/octavia -b stable/ussuri
cd octavia/bin
```


- Cari password octavia

```bash
grep octavia_keystone /etc/kolla/passwords.yml

ex output:
octavia_keystone_password: VQ2vA5AsFZLzt1t1FK39sMMu2R5BXMSSXtIXOWow
```

- Edit password pada script

```bash
sed -i 's/not-secure-passphrase/<octavia_ca_password>/g' create_single_CA_intermediate_CA.sh
```

- Jalankan script
```bash
./create_single_CA_intermediate_CA.sh openssl.cnf
```

- Copy file certificate ke direktori kolla-ansible

```bash
cd single_ca/etc/octavia/certs/
sudo mkdir -p /etc/kolla/config/octavia
sudo chown -R $USER:$USER /etc/kolla/config
cp * /etc/kolla/config/octavia
cd ~
```

- Selanjutnya adalah deploy octavia, edit file globals kolla-ansible

```bash
enable_octavia: "yes"
```

- Deploy service Octavia Openstack

```bash
kolla-ansible -i multinode deploy -t octavia
```

- Buat file openrc octavia

```bash
grep octavia_keystone /etc/kolla/passwords.yml

ex output:
octavia_keystone_password: VQ2vA5AsFZLzt1t1FK39sMMu2R5BXMSSXtIXOWow
```

- Membuat file openrc octavia

```bash
cp admin-openrc octavia-openrc.sh
nano octavia-openrc.sh
....
export OS_PROJECT_NAME=service
export OS_USERNAME=octavia
export OS_PASSWORD=<octavia_keystone_password>
....
```

- Source file octavia openrc

```bash
source octavia-openrc.sh
```

- Membuat image Amphora

```bash
sudo apt install -y qemu-utils git kpartx debootstrap
```

- Membuat image Amphora, sebelumnya kita harus menginstall paket disk-builder terlebih dahulu

```bash
deactivate
python3 -m venv disk-builder
source disk-builder/bin/activate
pip install diskimage-builder
```

- Membuat image Amphora, default nya akan membuat image ubuntu

```bash
cd ~
cd octavia/diskimage-create
./diskimage-create.sh
```

- Upload image to Glance

```bash
openstack image create amphora-x64-haproxy.qcow2 --container-format bare --disk-format qcow2 --private --tag amphora --file amphora-x64-haproxy.qcow2
```

- Membuat flavor amhora

```bash
openstack flavor create --vcpus 1 --ram 1024 --disk 3 "amphora" --private
```

- Membuat security group untuk instance amphora & health manager

Amphora

```bash
openstack security group create lb-mgmt-sec-grp
openstack security group rule create --protocol icmp lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 22 lb-mgmt-sec-grp
openstack security group rule create --protocol tcp --dst-port 9443 lb-mgmt-sec-grp
```

Health Manager

```bash
openstack security group create lb-health-mgr-sec-grp
openstack security group rule create --protocol udp --dst-port 5555 lb-health-mgr-sec-grp
openstack security group rule create --protocol icmp lb-health-mgr-sec-grp
```

- Membuat keypair untu instance amhora

```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub octavia_ssh_key
```

- Copy file dhclient.conf 

```bash
mkdir -m755 -p /etc/dhcp/octavia
cp octavia/etc/dhcp/dhclient.conf /etc/dhcp/octavia
```

- Membuat network dan subnet octavia

```bash
openstack network create lb-mgmt-net
OCTAVIA_MGMT_SUBNET=172.24.0.0/14
openstack subnet create --subnet-range $OCTAVIA_MGMT_SUBNET --gateway none --network lb-mgmt-net lb-mgmt-subnet
```

- Membuat port untuk health manager

```bash
OCTAVIA_MGMT_PORT_IP=172.24.0.10
SUBNET_ID=$(openstack subnet show lb-mgmt-subnet -f value -c id)
MGMT_PORT_ID=$(openstack port create --security-group \
  lb-health-mgr-sec-grp --device-owner Octavia:health-mgr \
  --host=$(hostname) -c id -f value --network lb-mgmt-net \
  --fixed-ip subnet=$SUBNET_ID,ip-address=$OCTAVIA_MGMT_PORT_IP \
  octavia-health-manager-listen-port)
```

- Menghubungkan port ke controller

```bash
MGMT_PORT_MAC=$(openstack port show -c mac_address -f value $MGMT_PORT_ID)

docker exec -it openvswitch_vswitchd ovs-vsctl -- --may-exist add-port br-int o-hm0 -- set Interface o-hm0 type=internal -- set Interface o-hm0 external-ids:iface-status=active -- set Interface o-hm0 external-ids:attached-mac=$MGMT_PORT_MAC -- set Interface o-hm0 external-ids:iface-id=$MGMT_PORT_ID 

ip link set dev o-hm0 address $MGMT_PORT_MAC
ip link set dev o-hm0 up
dhclient -v o-hm0 -cf /etc/dhcp/octavia/dhclient.conf
```

- Membuat service systemd network untuk health manager

```bash
sudo nano /etc/systemd/network/o-hm0.network
...
[Match]
Name=o-hm0

[Network]
DHCP=yes
```

- Buat script

```bash
sudo tee -a /opt/octavia-interface.sh<<-EOF
#!/bin/bash

set -ex

MGMT_PORT_MAC=$MGMT_PORT_MAC
MGMT_PORT_ID=$MGMT_PORT_ID

if [ "$1" == "start" ]; then
  sudo docker exec -it openvswitch_vswitchd ovs-vsctl -- --may-exist add-port br-int o-hm0 -- set Interface o-hm0 type=internal -- set Interface o-hm0 external-ids:iface-status=active -- set Interface o-hm0 external-ids:attached-mac=$MGMT_PORT_MAC -- set Interface o-hm0 external-ids:iface-id=$MGMT_PORT_ID
  ip link set dev o-hm0 address $MGMT_PORT_MAC
  ip link set o-hm0 up
elif [ "$1" == "stop" ]; then
  sudo docker exec -it openvswitch_vswitchd ovs-vsctl del-port o-hm0
else
  sudo docker exec -it openvswitch_vswitchd ovs-vsctl show br-int
  ip a s dev o-hm0
fi
EOF
```
- Berikan permission pada script file

```bash
sudo chmod +x /opt/octavia-interface.sh
```

- Membuat systemd service

```bash
sudo nano /etc/systemd/system/octavia-interface.service
...
[Unit]
Description=Octavia Interface Creator
Requires=openvswitch-switch.service
After=openvswitch-switch.service

[Service]
Type=oneshot
RemainAfterExit=true
ExecStart=/opt/octavia-interface.sh start
ExecStop=/opt/octavia-interface.sh stop

[Install]
WantedBy=multi-user.target
....
```

- Masukan resource ID ke file globals

```bash
openstack network show lb-mgmt-net | awk '/ id / {print $4}'
openstack security group show lb-mgmt-sec-grp | awk '/ id / {print $4}'
openstack flavor show amphora | awk '/ id / {print $4}'

nano /etc/kolla/globals.yml
....
octavia_amp_boot_network_list: <ID of lb-mgmt-net>
octavia_amp_secgroup_list: <ID of lb-mgmt-sec-grp>
octavia_amp_flavor_id: <ID of amphora flavor>
....
```

- Check ip address interface hm0

```bash
HM_IP=$(openstack port show octavia-health-manager-listen-port | awk '/ fixed_ips / {print $4}' | cut -d "'" -f 2)
echo $HM_IP
```

- Reconfiure servoce octavia

```bash
kolla-ansible -i ./multinode reconfigure -t octavia
```

- Install octavia client

```bash
source ~/kolla-install/bin/activate
source /etc/kolla/admin-openrc.sh
pip install python-octaviaclient
```

- Membuat loadbalancer basic

source menggunakan openrc admin

```bash
source ~/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh
```

Membaut loadbalancer

```bash
LB_VIP=$(openstack loadbalancer create --name lb1 --vip-subnet-id private-subnet | awk  '/ vip_address / {print $4}')
```

Membaut loadbalancer

```bash
LB_VIP=$(openstack loadbalancer create --name lb1 --vip-subnet-id private-subnet | awk  '/ vip_address / {print $4}')
```

Menambahkan floating ip ke loadbalancer

```bash
openstack floating ip set --port $(openstack port list | grep $LB_VIP | cut -d '|' -f 3) 10.20.150.50
```

Membuat HTTP listener

```bash
openstack loadbalancer listener create --name listener1 --protocol HTTP --protocol-port 80 lb1
```

Membuat pool untuk member loadbalancer

```bash
openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol HTTP
```

Membuat health monitor

```bash
openstack loadbalancer healthmonitor create --delay 5 --max-retries 3 --timeout 5 --type HTTP --url-path / pool1
```

Menambahkan instance ke loadbalancer

```bash
openstack loadbalancer member create --subnet-id private-subnet --address 192.168.100.167 --protocol-port 80 pool1
openstack loadbalancer member create --subnet-id private-subnet --address 192.168.100.81 --protocol-port 80 pool1
```
