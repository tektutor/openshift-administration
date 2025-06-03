## Day 1

## Red Hat Openshift System Requirements
Our Training Lab Hardware configurations ( For a multi-node Openshift cluster setup - with 3 masters and 3 worker nodes )
<pre>
Intel Xeon Processor with 48 Cores
755 GB RAM
18 TB HDD/SDD(Storage)
</pre>	

#### Master Nodes (Control Plane)
<pre>
- This can be physical server or a Virtual machine
- this can be server from your private datacenter or an AWS ec2 instance or an azure VMS
- a minimum of 3 master nodes are required for HA
</pre>	
<pre>
Processor - 8 vcpus
RAM - 128 GB RAM
HDD - 500 GB 
</pre>
Sizing considerations
<pre>
- For each 1000 pods about 1.5 GB RAM and 1 CPU core is required as a minimum
- Practically, atleast 16GB RAM and 8 vcpus and about 100GB Storage, in case you plan not to deploy user applications
</pre>	

#### Worker Nodes (Compute Node)
<pre>
Processor - 8 vcpus
RAM - 128 GB RAM
HDD - 500 GB 
</pre>
 
## Installing KVM on Ubuntu v24.04
```
sudo apt update
sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils -y
sudo adduser root kvm
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
sudo apt install virt-manager -y
sudo apt install -y guestfs-tools
```

## Installing KVM on Oracle Linux v9.4
```
sudo dnf install qemu-kvm -y
sudo dnf install libvirt libvirt-client -y
sudo dnf install libguestfs-tools virt-top virt-install virt-manager libguestfs-tools-c guestfs-tools -y
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
virsh list --all
sudo dnf install cockpit cockpit-machines
sudo systemctl enable --now cockpit.socket
sudo systemctl status cockpit.socket
sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --reload
```

## Create a file virt-net.xml
```
<network>
  <name>openshift4</name>
  <forward mode='nat'>
    <nat>
      <port start='1024' end='65535'/>
    </nat>
  </forward>
  <bridge name='openshift4' stp='on' delay='0'/>
  <domain name='openshift4'/>
  <ip address='192.168.100.1' netmask='255.255.255.0'>
  </ip>
</network>  
```

Run the below command
```
sudo virsh net-define --file virt-net.xml
sudo virsh net-autostart openshift4
sudo virsh net-start openshift4
sudo virsh net-list
```

Expected output
<pre>
sudo virsh net-define --file virt-net.xml
Network openshift4 defined from virt-net.xml
</pre>

## Create a bastion vm
```
sudo virt-builder -l
sudo virt-builder fedora-41  --format qcow2 \
  --size 20G -o /var/lib/libvirt/images/ocp-bastion-server.qcow2 \
  --root-password password:Root@123

sudo virt-install \
  --name ocp-bastion-server \
  --ram 4096 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/ocp-bastion-server.qcow2 \
  --os-variant rhel8.0 \
  --network bridge=openshift4 \
  --graphics none \
  --serial pty \
  --console pty \
  --boot hd \
  --import
```

## After signing into Fedora 41 VM
```
ip link show

nmcli con add type ethernet con-name enp1s0 ifname enp1s0 \
  connection.autoconnect yes ipv4.method manual \
  ipv4.address 192.168.100.254/24 ipv4.gateway 192.168.100.1 \
  ipv4.dns 8.8.8.8

ping -c 2 8.8.8.8
ping -c 2 google.com

sudo dnf -y upgrade
sudo dnf -y install git vim wget curl bash-completion tree tar libselinux-python3 firewalld ansible
sudo reboot
```

## On Oracle Linux v9.4
```
sudo virsh list
sudo virsh console ocp-bastion-server
sudo virsh autostart ocp-bastion-server
sudo virsh start ocp-bastion-server
```

## On Fedora 39 VM
```
sudo dnf -y install git ansible vim wget curl bash-completion tree tar libselinux-python3
cd ~/
git clone https://github.com/jmutai/ocp4_ansible.git
cd ~/ocp4_ansible
```

vim ansible.cfg
```
[defaults]
inventory = inventory
command_warnings = False
filter_plugins = filter_plugins
host_key_checking = False
deprecation_warnings=False
retry_files = false
```

Mofify vim vars/main.yml to configure domain name, IP address and network interface name.

Install DHCP Server
```
sudo yum -y install dhcp-server
sudo systemctl enable dhcpd
ansible-playbook tasks/configure_dhcpd.yml
systemctl status dhcpd
cat /etc/dhcp/dhcpd.conf
sudo yum -y install bind bind-utils
sudo systemctl enable named
```

Install DNS serial number generator
sudo vim /usr/local/bin/set-dns-serial.sh
```
#!/bin/bash
dnsserialfile=/usr/local/src/dnsserial-DO_NOT_DELETE_BEFORE_ASKING_CHRISTIAN.txt
zonefile=/var/named/zonefile.db
if [ -f zonefile ] ; then
	echo $[ $(grep serial ${zonefile}  | tr -d "\t"" ""\n"  | cut -d';' -f 1) + 1 ] | tee ${dnsserialfile}
else
	if [ ! -f ${dnsserialfile} ] || [ ! -s ${dnsserialfile} ]; then
		echo $(date +%Y%m%d00) | tee ${dnsserialfile}
	else
		echo $[ $(< ${dnsserialfile}) + 1 ] | tee ${dnsserialfile}
	fi
fi
##
##
```

Run the script
```
sudo chmod a+x /usr/local/bin/set-dns-serial.sh
ansible-playbook tasks/configure_bind_dns.yml
systemctl start named
systemctl status named
dig @127.0.0.1 -t srv _etcd-server-ssl._tcp.ocp4.alchemy.com
```

## On Fedora 39 VM
```
nmcli connection show
nmcli connection modify enp1s0  ipv4.dns "192.168.100.254"
nmcli connection reload
nmcli connection up enp1s0
host bootstrap.ocp4.alchemy.com
sudo firewall-cmd --add-service={dhcp,tftp,http,dns} --permanent
sudo firewall-cmd --reload
```

Install TFTP on helper node
```
sudo yum -y install tftp-server syslinux
sudo firewall-cmd --add-service=tftp --permanent
sudo firewall-cmd --reload
```
sudo vim /etc/systemd/system/helper-tftp.service
```
[Unit]
Description=Starts TFTP on boot because of reasons
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/start-tftp.sh
TimeoutStartSec=0
Restart=always
RestartSec=30

[Install]
WantedBy=default.target
```

```
sudo tee /usr/local/bin/start-tftp.sh<<EOF
#!/bin/bash
/usr/bin/systemctl start tftp > /dev/null 2>&1
##
##
EOF

sudo chmod a+x /usr/local/bin/start-tftp.sh
sudo systemctl daemon-reload
sudo systemctl enable --now tftp helper-tftp

sudo mkdir -p  /var/lib/tftpboot/pxelinux.cfg
sudo cp -rvf /usr/share/syslinux/* /var/lib/tftpboot
sudo mkdir -p /var/lib/tftpboot/rhcos
```

Download images
```
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-installer-kernel-x86_64
sudo mv rhcos-installer-kernel-x86_64 /var/lib/tftpboot/rhcos/kernel

wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-installer-initramfs.x86_64.img
sudo mv rhcos-installer-initramfs.x86_64.img /var/lib/tftpboot/rhcos/initramfs.img

sudo restorecon -RFv  /var/lib/tftpboot/rhcos
ls /var/lib/tftpboot/rhcos
sudo yum -y install httpd
```

sudo vim /etc/httpd/conf/httpd.conf and change Listen 80 to 8080

```
sudo rm /etc/httpd/conf.d/welcome.conf
sudo systemctl enable httpd
sudo systemctl restart httpd
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo mkdir -p /var/www/html/rhcos

wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/latest/rhcos-live-rootfs.x86_64.img

sudo mv rhcos-live-rootfs.x86_64.img /var/www/html/rhcos/rootfs.img
sudo restorecon -RFv /var/www/html/rhcos
ansible-playbook tasks/configure_tftp_pxe.yml
ls -1 /var/lib/tftpboot/pxelinux.cfg
sudo yum install -y haproxy
sudo setsebool -P haproxy_connect_any 1
sudo mv /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.default
ansible-playbook tasks/configure_haproxy_lb.yml

yum install policycoreutils-python-utils
sudo vim /etc/haproxy/haproxy.cfg

sudo semanage port  -a 6443 -t http_port_t -p tcp
sudo semanage port  -a 22623 -t http_port_t -p tcp
sudo semanage port -a 32700 -t http_port_t -p tcp
sudo firewall-cmd --add-service={http,https} --permanent
sudo firewall-cmd --add-port={6443,22623}/tcp --permanent
sudo firewall-cmd --reload

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
tar xvf openshift-client-linux.tar.gz
sudo mv oc kubectl /usr/local/bin
rm -f README.md LICENSE openshift-client-linux.tar.gz

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux.tar.gz
tar xvf openshift-install-linux.tar.gz
sudo mv openshift-install /usr/local/bin
rm -f README.md LICENSE openshift-install-linux.tar.gz

openshift-install version
oc version --client
kubectl version --client
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
mkdir ~/.openshift
vim  ~/.openshift/pull-secret
mkdir -p ~/ocp4
cd ~/

cat <<EOF > install-config-base.yaml
apiVersion: v1
baseDomain: alchemy.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: '$(< ~/.openshift/pull-secret)'
sshKey: '$(< ~/.ssh/id_rsa.pub)'
EOF
```

Then
```
vim  install-config-base.yaml
apiVersion: v1
baseDomain: alchemy.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ocp4
networking:
  clusterNetworks:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  networkType: OVNKubernetes
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
fips: false
pullSecret: 'paste-as-obtained-from-https://cloud.redhat.com'
sshKey: 'PASTE-SSH-PUBLIC-KEY'
```

Then
```
cd ~/
cp install-config-base.yaml ocp4/install-config.yaml
cd ocp4

openshift-install create manifests
ls
sed -i 's/true/false/' manifests/cluster-scheduler-02-config.yml

openshift-install create ignition-configs
ls
sudo mkdir -p /var/www/html/ignition
sudo cp -v *.ign /var/www/html/ignition
sudo chmod 644 /var/www/html/ignition/*.ign
sudo restorecon -RFv /var/www/html/
ls /var/www/html/ignition/

sudo systemctl enable --now haproxy.service dhcpd httpd tftp named
sudo systemctl restart haproxy.service dhcpd httpd tftp named
sudo systemctl status haproxy.service dhcpd httpd tftp named

systemctl status haproxy
```

## Create the bootstrap VM
```
sudo virt-install -n bootstrap.ocp4.alchemy.com \
  --description "Bootstrap Machine for Openshift 4 Cluster" \
  --ram=8192 \
  --vcpus=4 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=50 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:a4:db:5f

journalctl -f
journalctl -f -u tftp
journalctl -f -u dhcpd

sudo virsh --connect qemu:///system start bootstrap.ocp4.alchemy.com

```

## Create Master nodes
```
# Create Master01 VM
sudo virt-install -n master01.ocp4.alchemy.com \
  --description "Master01 Machine for Openshift 4 Cluster" \
  --ram=131072 \
  --vcpus=8 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=500 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:8b:a1:17

# Create Master02 VM
sudo virt-install -n master02.ocp4.alchemy.com \
  --description "Master02 Machine for Openshift 4 Cluster" \
  --ram=131072 \
  --vcpus=8 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=500 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:ea:8b:9d

# Create Master03 VM
sudo virt-install -n master03.ocp4.alchemy.com \
  --description "Master03 Machine for Openshift 4 Cluster" \
  --ram=131072 \
  --vcpus=8 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=500 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:f8:87:c7

sudo virsh --connect qemu:///system start master01.ocp4.alchemy.com
sudo virsh --connect qemu:///system start master02.ocp4.alchemy.com
sudo virsh --connect qemu:///system start master03.ocp4.alchemy.com
```


## Create Worker Nodes ( Must be created once the master nodes are listed by oc get nodes )
```
# Create Worker01 VM
sudo virt-install -n worker01.ocp4.alchemy.com \
  --description "Worker01 Machine for Openshift 4 Cluster" \
  --ram=131072 \
  --vcpus=8 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=500 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:31:4a:39
 
# Create Worker02 VM
sudo virt-install -n worker02.ocp4.alchemy.com \
  --description "Worker02 Machine for Openshift 4 Cluster" \
  --ram=131072 \
  --vcpus=8 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=500 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:6a:37:32

# Create Worker03 VM
sudo virt-install -n worker03.ocp4.alchemy.com \
  --description "Worker03 Machine for Openshift 4 Cluster" \
  --ram=131072 \
  --vcpus=8 \
  --os-variant=rhel8.0 \
  --noreboot \
  --disk pool=default,bus=virtio,size=500 \
  --graphics none \
  --serial pty \
  --console pty \
  --pxe \
  --network bridge=openshift4,mac=52:54:00:95:d4:ed

sudo virsh --connect qemu:///system start worker01.ocp4.alchemy.com
sudo virsh --connect qemu:///system start worker02.ocp4.alchemy.com
sudo virsh --connect qemu:///system start worker03.ocp4.alchemy.com

for i in {1..3}; do
	sudo virsh autostart master0${i}.ocp4.alchemy.com
done

for i in {1..2}; do
	sudo virsh autostart worker0${i}.ocp4.alchemy.com
done

virsh list --autostart

export KUBECONFIG=/root/ocp4/auth/kubeconfig
mkdir ~/.kube
sudo cp /root/ocp4/auth/kubeconfig ~/.kube/config
sudo chown $USER ~/.kube/config

vim ~/.bashrc
source <(oc completion bash)
source <(kubectl completion bash)
source  ~/.bashrc
```

Test
```
oc whoami
oc whoami --show-server
oc get clusterversions.config.openshift.io
oc get nodes
oc get csr
oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs oc adm certificate approve
oc get nodes
oc get co
oc get ingresscontroller default -n openshift-ingress-operator -o jsonpath='{.status.domain}'
oc whoami --show-console
cat ocp4/auth/kubeadmin-password
```

To access web console

We need to add the below entries in /etc/hosts file
<pre>
192.168.100.254 api.ocp4.alchemy.com oauth-openshift.apps.ocp4.alchemy.com console-openshift-console.apps.ocp4.alchemy.com</pre>

```
oc whoami --show-console
cat ocp4/auth/kubeadmin-password
```

To allow scheduling user pods onto master nodes
```
oc get nodes
oc edit schedulers.config.openshift.io cluster
oc get nodes
```

# Install NFS Server in Ubuntu
```
sudo apt install -y nfs-kernel-server
sudo systemctl start nfs-kernel-server.service
sudo apt install -y nfs-common
sudo systemctl restart nfs-kernel-server
sudo systemctl enable nfs-kernel-server
```

## Installing NFS Server in Oracle Linux
```
sudo dnf install -y nfs-utils
sudo systemctl enable nfs-server
sudo systemctl start nfs-server
sudo mkdir -p /var/nfs/user<xy>/share1
sudo mkdir -p /var/nfs/user<xy>/share2
sudo mkdir -p /var/nfs/user<xy>/share3
sudo mkdir -p /var/nfs/user<xy>/share4
sudo mkdir -p /var/nfs/user<xy>/share5

sudo chmod 777 -R /var/nfs
sudo chown nobody:nobody -R /var/nfs

sudo firewall-cmd --permanent --zone=public --add-service=nfs
sudo firewall-cmd --reload
sudo firewall-cmd --list-all

sudo vim /etc/exports
/var/nfs/user<xy>/share1 192.168.100.0/24(rw,no_root_squash)
/var/nfs/user<xy>/share2 192.168.100.0/24(rw,no_root_squash)
/var/nfs/user<xy>/share3 192.168.100.0/24(rw,no_root_squash)
/var/nfs/user<xy>/share4 192.168.100.0/24(rw,no_root_squash)
/var/nfs/user<xy>/share5 192.168.100.0/24(rw,no_root_squash)

exportfs -a
showmount -e localhost
```

## Importing Container Images into Openshift's Internal registry
```
oc import-image ubi8/openjdk-11:1.20-2.1727147549 --from=registry.access.redhat.com/ubi8/openjdk-11:1.20-2.1727147549 --confirm

https://access.redhat.com/solutions/2072843
https://access.redhat.com/solutions/6159832
```

## Troubleshooting webconsole not accessible
```
oc -n openshift-ingress get pod -o json |
  jq -r '.items[].metadata.name' |
  xargs oc -n openshift-ingress delete pod

oc -n openshift-console get pods -w
```

## Troubleshooting routes
```
oc get ingresses.config/cluster -o jsonpath={.spec.domain}
```

## Configure the Openshift's Internal Registry
<pre>
https://www.ibm.com/docs/en/mas-cd/continuous-delivery?topic=installing-enabling-openshift-internal-image-registry	
</pre>
Create PV and PV in a file storage.yml
<pre>
apiVersion: v1
kind: PersistentVolume
metadata:
  name: image-registry-pv
spec:
  capacity:
    storage: 100Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: image-registry-storage
    namespace: openshift-image-registry
  accessModes:
  - ReadWriteMany
  nfs:
    path: /var/nfs/jegan/image-registry
    server: 192.168.1.127
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  finalizers:
  - kubernetes.io/pvc-protection
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
</pre>

```
oc apply -f storage.yml
oc get po -n openshift-image-registry
oc edit configs.imageregistry.operator.openshift.io -n openshift-image-registry
oc get po -n openshift-image-registry -w

firewall-cmd --permanent --add-port=111/tcp --zone=public
firewall-cmd --permanent --add-port=2049/tcp --zone=public
firewall-cmd --permanent --add-port=33333/tcp --zone=public
firewall-cmd --permanent --add-service=nfs --zone=public
firewall-cmd --reload
firewall-cmd --list-all
```

Expected output
![image](https://github.com/user-attachments/assets/702c62fd-6c75-43a3-a53e-07c0708d8ede)
![image](https://github.com/user-attachments/assets/3c1c9183-5940-41f9-9d47-233cfcba7654)


## Troubleshooting logs retrieval errors in commandline and webconsole for every application
Approve all pending certificate signing requests
```
oc get csr -o name | xargs oc adm certificate approve
```
For further details
<pre>
https://access.redhat.com/solutions/4307511	
</pre>

## Info - How does the master node get the ignition file ?
<pre>
- While creating the VM using KVM, we supply an unique mac address for each VM
- KVM sends a DHCP request to provide an IP for the mac address provided in the virt-install
- Master nodes gets an IP address assigned by dhcp server ( in our case, a fixed static address gets assigned 
  based on /etc/dhcpd.conf )
- the bootstrap vm will now be able to resolve the master node using its dns-name master01.ocp4.rps.com based on /etc/named/zonefile.db configuration done in the named(bind) dns server
- While creating the VM using KVM, we supply a switch --pxe, this helps the VM pick the kernel parameters, 
 i.e which ignition file it must be using, the web server url from where it can download the 
 kernel.img, initramfs.img, ignition  files, etc
</pre>

## Info - How does the worker node get the ignition file ?
<pre>
- While creating the VM using KVM, we supply an unique mac address for each VM
- KVM sends a DHCP request to provide an IP for the mac address provided in the virt-install
- Worker nodes gets an IP address assigned by dhcp server ( in our case, a fixed static address gets assigned 
  based on /etc/dhcpd.conf )
- While creating the VM using KVM, we supply a switch --pxe, this is where the tftp server gets involved. 
  The tftpd server will help the vm with network boot configuration. 
  In the /var/lib/pxeboot/pxelinux.cfg folder, for each VM there is a pxe configuration file organized by its mac address.  
  This is how the VM is able to pick the correct path of kernel.img initramfs.img ignition file, etc
</pre>

## Info - What is the role of bootstrap VM in the Openshift installation of process?
<pre>
- BootStrap VM uses COREOS operating system, hence it comes with podman container engine and container runtime, 
  kubelet service
- the kubelet service starts the bootkube process(service)
- bootkube starts a temporary Kubernetes cluster within the bootstrap VM using the manifest files 
  kept at /etc/kubernetes/manifests and other manifests files from 
  /etc/ecc folder (Early Cluster Configuration Manifest files)
- apart from starting the K8s control plane, it also starts a static Pod called Cluster Version Operator (CVO)
- The Cluster Version Operator takes care installing the correct version of Openshift components, images, etc.,
- You may check /etc/kubernetes/manifests/cluster-version-operator-pod.yaml
- The CVO uses the Custom Resource(CR) named ClusterVersion to decide the openshift version, etc.,
- The CVO creates the openshift namespaces, deployments, Cluster Operators, etc.,
- Some of the Cluster operators that are started are
  - Machine Config Operator
  - Kube Control Manager Operator
  - Etcd Operator
  - Network Operator
  - Ingress Operator
- The CVO watches all the Cluster Operators ( Waits for them to become Available ) 

- When the master node is booted, the kubelet service on the master node
- the Kubelet contacts the boostrap API Server via its DNS/IP (api-int.<cluster-name>.<base-domain>)
- the master node kubelet registers with the boostrap node API server using the certificates injected by the ignition
- Via the BootStrap API, the Machine Config Operator (MCO) pushes the manifests files onto the master node 
- the kubelet container agent service in the master nodes, starts detecting the manifests files in the 
  /etc/kubernetes/manifests folder on the master node
- the kubelet then starts the static pods eventually creating the control plane on the master node
- the master nodes joins the BootStrap Control Plane and partially starts serving the control plane responsibilities
- the bookube keeps monitors the master node(s) control plane components and checks if they are healthy
- Once all the masters nodes are up and running, i.e serving the API/etcd traffic the cluster becomes self-hosted
- the bootkube process shuts down the temporary control plane in the bootsrap vm and master nodes take over from here
</pre>
