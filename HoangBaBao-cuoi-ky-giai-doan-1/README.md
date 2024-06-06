# VDT Final Project
## Name: Hoang Ba Bao

## I. Triển khai Kubernetes
### Request: cài đặt Kubernetes thông qua công cụ kubeadm với 1 master node VM + 2 worker node VM

### Các bước cài đặt và tài liệu
#### 1. Chuẩn bị và setup máy ảo trước khi cài đặt Kubernetes
- Tạo 3 con VM ubuntu ec2 trên AWS đảm bảo:
  -  3 VM ở cùng một VPC để có thể ping thông một cách dễ dàng
  -  Tạo ssh-key để đảm bảo Security
  -  Security Group cho phép ssh vào các máy đó từ máy mình
 
![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/f1b15862-2f37-42a8-9640-fabb5f6a77cd)

- Để có thể truy cập dễ dàng hơn ta có thể đổi tên hostname từ ip tương ứng với master, worker1, worker2
```
sudo hostnamectl set-hostname your-desired-hostname
exec bash
```

- Update file /etc/hosts để enable host name với IP address tương ứng

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/d382fceb-3bea-4e84-ae29-6ea6cb6e2d3e)

- Disable Swap bởi vì bộ nhớ swap có thể ảnh hưởng đến scheduling decisions của k8s vì tốc độ đọc viết và truy cập khác biệt với RAM
- Thay đổi file fstab để đảm bảo rằng những thay đổi này được lâu dài   

```
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

- Đảm bảo rằng tất cả package đều up-to-date trước khi cài đặt Kubernetes

```
sudo apt-get update && sudo apt-get upgrade -y 
```

- Cài đặt IP Bridge cho các nodes có thể kết nối với nhau qua mạng

```
cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```

- Cấu hình tham số cần thiết bởi sysctl:

```
cat << EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

- Apply những thay đổi bằng câu lệnh sau:

```
sudo sysctl --system
```

#### 2. Bắt đầu triển khai Kubernetes trên các máy VM đã được cấu hình ở trên
- Cài đặt kubeadm, kubelet và kubectl

- Install ca-certificates để đảm bảo rằng tài liệu mình tải là real và an toàn

```
sudo apt-get install -y apt-transport-https ca-certificates curl
```

- Tạo directory ở /etc/apt/keyrings để chứa public key cho gói cài đặt Kubernetes và curl public key về
  
```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

- Thêm gói cài đặt apt của K8s vào trong source cài đặt
  
```
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

- Update index của apt package, cài đặt kubelet, kubeadm và kubectl đồng thời giữ lại version hiện tại để tránh việc auto-upgrade dẫn đến conflict versions

```
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

#### 3. Cài đặt Docker với containerd là container runtime nền tảng cho K8s
- Cài đặt Docker đơn giản bằng câu lệnh trên các nodes

```
sudo apt install docker.io
```

- Bắt đầu set up containerd trên tất cả các nodes để đảm bảo rằng mọi thứ đều đồng bộ bằng cách tạo thư mục ở /etc/

```
sudo mkdir /etc/containerd
```

- Gen ra một file config mặc định cho containerd

```
sudo sh -c "containerd config default > /etc/containerd/config.toml"
```

- Enable SystemdCgroup để đảm bảo rằng Kubernetes có thể tối ưu việc quản lý tài nguyên cho containers.

```
sudo sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
```

- Apply những thay đổi này bằng cách restart lại các service và enable kubelet service để đảm bảo rằng nó luôn được boot và hoạt động mỗi lần khởi chạy VM

```
sudo systemctl restart containerd.service && sudo systemctl restart kubelet.service
sudo systemctl enable kubelet.service
```

#### 4. Cài đặt cụm Kubernetes trên Node master
- Cài đặt K8s control plane với đầy đủ các component cần thiết bằng cách xử dụng một câu lệnh sau

```
sudo kubeadm config images pull
```

- Chỉ định dải mạng cho pod đảm bảo rằng các pod có thể communicate với nhau một cách dễ dàng

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

- Cài đặt file config .kube với config của cụm và cập nhật để đảm bảo rầng mình có đủ quyền để sửa

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Sau bước kubeadm init thành công lưu lại token dùng để add worker node vào mạng lưới của master node

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/6ff1d794-d3bf-4a4f-acee-3d8771833437)

#### 5. Cài Network Plugin thích hợp (Calico)
- Deploy Calico operator sử dụng

```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
```

- Tải file custom cho resources của Calico, định nghĩa cho việc triển khai tài nguyên của Calico

```
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml -O
```

- Chạy file vừa curl về để triển khai resources

```
kubectl create -f custom-resources.yaml
```

#### 6. Add worker nodes vào cụm

- Sau khi config xong master đầy là thời gian để add những node worker vào cụm
- Generate token dùng để join vào cụm bằng câu lệnh sau ở master node và sử dụng token ở worker node để add worker node vào cụm

```
kubeadm token create --print-join-command
```
- Khi join worker thành công sẽ có thông báo như sau:

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/f09914fc-bebd-4878-b6f5-ed958d524067)

#### 7. Kiểm tra hệ thống đã cài với các câu lệnh

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/c32cc221-11b0-4889-b79c-33030e272989)

Quá tuyệt vời cho một buổi cài đặt K8s qua kubeadm :)








 
