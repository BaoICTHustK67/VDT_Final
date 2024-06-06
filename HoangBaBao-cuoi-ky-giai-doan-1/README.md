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


 
