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

- Dải mạng của calico => sẽ không hoạt động ở các VM trên cloud khi mà clusterIP thường dc assign với dạng 10.x.x.x   

```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

- Dải mạng của flannel => hoạt động được trên các VM ở cloud (Đây là dải mạng sẽ được sử dụng)
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

- Cài đặt file config .kube với config của cụm và cập nhật để đảm bảo rầng mình có đủ quyền để sửa

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

- Sau bước kubeadm init thành công lưu lại token dùng để add worker node vào mạng lưới của master node

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/6ff1d794-d3bf-4a4f-acee-3d8771833437)

#### 5. Cài Network Plugin thích hợp (Calico hỗ trợ máy ảo local, không hỗ trợ cho cloud) => (Flannel hỗ trợ cho cloud)

- Việc cài Calico khiến em tốn tận 2 ngày ngồi debug và mày mò network trước khi nhận ra nó không hỗ trợ cloud nên ví dụ dưới đây sẽ chỉ áp dụng cho việc cài máy ảo ở local
 
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

- Cài đặt triển khai cho các VM trên cloud với cluster IP của pod 10.x.x.x thì ta sẽ sử dụng Flannel nếu không sẽ không thể tạo connection giữa các pod vì khác private network của pod (10.x.x.x và 192.168.x.x)

- Chỉ đơn giản bằng 1 câu lệnh ta sẽ cài xong flannel

```
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

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

# II. Triển khai web application sử dụng các DevOps tools & practices

## K8S Helm Chart

### Request 1:

- Bắt đầu triển khai argoCD lên cụm K8s bằng cách tạo một namespace riêng và install file manifest sau

```
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Chờ khi deploy thành công bắt đầu expose ArgoCD thông qua service NodePort bằng file yaml có nội dung sau:

```
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
  namespace: argocd
spec:
  type: NodePort
  selector:
    app.kubernetes.io/name: argocd-server
  ports:
    - name: http
      port: 80
      targetPort: 8080
      nodePort: 30080
```
- port: 80 là port được exposed của cụm K8S thường để phục vụ http
- targetPort: 8080 : khi traffic đi qua port của cụm K8S sẽ được redirect đến port này của argocd-server pod
- nodePort: 30080: đấy là port exposed ra ở VM host cụm K8s và khi đi qua nodePort sẽ được redirect đến pod này
=> NodePort(30080) => Cluster Port(80) => Pod Port(8080) 

- Kiểm tra việc exposed bằng việc xem các service thông qua
```
kubectl get svc -n argocd
```

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/bbe632b2-52d5-47e8-9347-d800e035bc2c)



- Ảnh chụp giao diện màn hình khi đã deploy thành công và expose qua NodePort

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/0645ed9c-8206-4b97-a746-1e79117cb562)


- Link File Manifest để triển khai ArgoCD lên K8S: https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

### Request 2:

#### Repo chứa Helm Chart và values.yaml (2 repo Helm Chart, 2 repo config values.yaml) cho web và api

- Helm Chart sử dụng để triển khai web Deployment : https://github.com/BaoICTHustK67/VDT_frontend/tree/main/frontend

- Repo chứa file values.yaml của web: https://github.com/BaoICTHustK67/web_values

- Helm Chart sử dụng để triển khai api Deployment : https://github.com/BaoICTHustK67/VDT_backend/tree/main/backend

- Repo chứa file values.yaml của api: https://github.com/BaoICTHustK67/api_values

#### Manifest của ArgoCD Application

- web:

```
project: default
destination:
  server: 'https://kubernetes.default.svc'
  namespace: vdt-web
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
sources:
  - repoURL: 'https://github.com/BaoICTHustK67/VDT_frontend'
    path: frontend
    targetRevision: HEAD
    helm:
      valueFiles:
        - $values/values.yaml
  - repoURL: 'https://github.com/BaoICTHustK67/web_values'
    targetRevision: HEAD
    ref: values
```

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/e9677d51-a650-4442-aebb-459e8c1f32e1)

- api:
```
project: default
destination:
  server: 'https://kubernetes.default.svc'
  namespace: vdt-api
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - CreateNamespace=true
sources:
  - repoURL: 'https://github.com/BaoICTHustK67/VDT_backend'
    path: backend
    targetRevision: HEAD
    helm:
      valueFiles:
        - $values/values.yaml
  - repoURL: 'https://github.com/BaoICTHustK67/api_values'
    targetRevision: HEAD
    ref: values

```

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/7c7a56df-b159-45c2-8021-416da4e8b4bb)



#### Ảnh chụp giao diện 

- Màn hình hệ thống ArgoCD trên trình duyệt (Public có sự thay đổi do với lần cài ArgoCD là do em đã tắt và bật lại VM trên EC2 để tiết kiệm chi phí khi không dùng nên public address mới sẽ được assign)

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/bd635786-aad4-4feb-8cef-c4316588cd2e)


![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/394ecea2-fb34-4c03-801a-457aa384762c)



- Khi truy cập vào Web URL exposed qua NodePort:


![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/cf9c9a34-625b-4686-9a9f-f02c632ef7d5)




- Khi truy cập vào API URL exposed qua NodePort:

![image](https://github.com/BaoICTHustK67/VDT_Final/assets/123657319/4ce31d11-982f-4612-a8fb-73574cd6c19b)

## III. Continuous Delivery









 
