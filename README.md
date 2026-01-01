# วิธีการติดตั้ง Kubernetes บน Redhat Enterprise 9.7

1. เตรียมระบบพื้นฐาน (ทุก Node)
1.1. ปิด Swap และตั้งค่า SELinux: Kubernetes จะไม่ทำงานหากเปิด Swap และ SELinux อาจบล็อกการทำงานของ Container (ในขั้นตอนเริ่มต้นแนะนำให้เป็น Permissive)

```bash
# ปิด Swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# ตั้งค่า SELinux เป็น Permissive
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

1.2. โหลด Kernel Modules และตั้งค่า Network:
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# ตั้งค่า sysctl
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

2. ติดตั้ง Docker Engine และ cri-dockerd (ทุก Node)
2.1. เพิ่ม Docker Repository และติดตั้ง:
```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y docker-ce docker-ce-cli containerd.io
sudo systemctl enable --now docker
```

2.2. ติดตั้ง cri-dockerd (เนื่องจาก K8s v1.24+ ไม่รองรับ Docker โดยตรง): สำหรับ RHEL 9 ให้ใช้แพ็กเกจของ el9
```bash
# ดึงเวอร์ชันล่าสุด
VER=$(curl -s https://api.github.com/repos/Mirantis/cri-dockerd/releases/latest | grep tag_name | cut -d '"' -f 4 | sed 's/v//g')
wget https://github.com/Mirantis/cri-dockerd/releases/download/v${VER}/cri-dockerd-${VER}.amd64.tgz

# แตกไฟล์และย้ายไปที่ตำแหน่งระบบ
tar xzwf cri-dockerd-${VER}.amd64.tgz
sudo mv cri-dockerd/cri-dockerd /usr/local/bin/
sudo chmod +x /usr/local/bin/cri-dockerd
```

2.3. สร้าง Systemd Unit Files (Socket และ Service)
```bash
cat <<EOF | sudo tee /etc/systemd/system/cri-docker.socket
[Unit]
Description=cri-docker: Docker Application Container Engine API [Socket]

[Socket]
ListenStream=/var/run/cri-dockerd.sock

[Install]
WantedBy=sockets.target
EOF
```
2.4. สร้างไฟล์ Service:
```bash
cat <<EOF | sudo tee /etc/systemd/system/cri-docker.service
[Unit]
Description=cri-docker: Docker Application Container Engine API
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
Requires=cri-docker.socket

[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --container-runtime-endpoint fd://
ExecReload=/bin/kill -s HUP \$MAINPID
TimeoutSec=0
RestartSec=2
Restart=always

[Install]
WantedBy=multi-user.target
EOF
```

3. ติดตั้ง Kubernetes Tools (ทุก Node)
เพิ่ม Repository ของ Kubernetes (เวอร์ชัน 1.31 เป็นต้นไปใช้ URL ใหม่):
```bash
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.31/rpm/repodata/repomd.xml.key
EOF

sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
sudo systemctl enable --now kubelet
```

ติดตั้งแพ็กเกจเสริมที่จำเป็น (ป้องกัน Error เหมือนที่เจอใน Ubuntu):
```bash
sudo yum install -y conntrack-tools socat iproute-tc
```
 
4. เปิดพอร์ตบน Firewall (ทุก Node)
RHEL 9 เปิด Firewall เป็นค่าเริ่มต้น หากไม่เปิดพอร์ต Cluster จะคุยกันไม่ได้:

- Master Node:
```bash
sudo firewall-cmd --permanent --add-port={6443,2379-2380,10250,10251,10252}/tcp
sudo firewall-cmd --reload
```

- Worker Node:
```Bash
sudo firewall-cmd --permanent --add-port={10250,30000-32767}/tcp
sudo firewall-cmd --reload
```

5. เริ่มต้น Cluster (เฉพาะ Master Node)
รันคำสั่ง Init โดยระบุ cri-socket ของ Docker:
```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket unix:///var/run/cri-dockerd.sock
```

หลังรันเสร็จ ให้ตั้งค่าสิทธิ์ Access:
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

6. ติดตั้ง Network Plugin และ Join Worker
6.1 ติดตั้ง Flannel (ที่ Master):
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

6.2 Join Worker Node: นำคำสั่งที่ได้จาก Master มาเติม --cri-socket แล้วรันที่เครื่อง Worker:
```bash
sudo kubeadm join <MASTER_IP>:6443 --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --cri-socket unix:///var/run/cri-dockerd.sock
```
