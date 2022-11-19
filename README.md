```
运行方式：
1. 准备requirement 文件如下：

[devops@ansible test-dep]$ cat requirements.yml
- name: k8s_ha_install
  src: https://github.com/jingwei269/k8s_ha_install
  scm: git


2. install role 
ansible-galaxy install -r requirements.yml -p /roles 

3. 准备主 playbook 如下: 
hosts放置所有需要安装的节点，其中master节点放置到组master中  
另外需要配置backup组，用于配置keepalived 
[devops@ansible test-dep]$ cat k8s_ha_install.yml
---
- name: prepare k8s required tools on new server
  hosts: k8s

  roles:
   - k8s_ha_install

======inventory============
[master]
master[1:3].example.local

[worker]
worker[1:2].example.local


[k8s:children]
master
worker

[backup]
master[2:3].example.local

[newserver]
master[2:3].example.local

4. 编辑目录roles/k8s_ha_install/files/下的文件
keepalived.master.conf
keepalived.bkp.conf
haproxy.cfg
check_apiserver.sh
修改VIP地址(所有文件)，以及对应各个master的IP地址(haproxy.cfg)。
还需要修改interface (nmcli dev show  |grep -i dev) . 


5. 运行playbook 
ansible-playbook k8s_ha_install.yml


6. 在master节点上初始化k8s集群：
## --control-plane-endpoint 《-- 配置为VIP地址和端口(haproxy.cfg中的配置)
## --upload-certs <--需要加上这个参数为配置多master
kubeadm init  --control-plane-endpoint=192.168.31.50:9443 --pod-network-cidr 10.244.0.0/16 --kubernetes-version=v1.24.1 --image-repository=registry.aliyuncs.com/google_containers --upload-certs

### 运行无误会出现下面提示： 
You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 192.168.31.50:9443 --token o0a6p3.4d0k2kb8dh0kq32g \
	--discovery-token-ca-cert-hash sha256:6cad2a213eb100eacb08c07b36d7b51991bf20b8874d73228331e91ed921d393 \
	--control-plane --certificate-key bfb03b2e461a46e53b060e84d8b0f5fa5c3fc88094cae8c20053900dc189838d

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.31.50:9443 --token o0a6p3.4d0k2kb8dh0kq32g \
	--discovery-token-ca-cert-hash sha256:6cad2a213eb100eacb08c07b36d7b51991bf20b8874d73228331e91ed921d393

### 遇到的问题：pull image时出现解析到的地址时ipv6，且network is unreachable。需要检查DNS配置。
[root@master2 keepalived]# crictl pull registry.aliyuncs.com/google_containers/pause:3.7
E1119 10:04:54.333462   12892 remote_image.go:242] "PullImage from image service failed" err="rpc error: code = Unknown desc = Get \"https://dockerauth.cn-hangzhou.aliyuncs.com/auth?scope=repository%3Agoogle_containers%2Fpause%3Apull&service=registry.aliyuncs.com%3Acn-hangzhou%3A26842\": dial tcp [2408:4005:1000:10::2]:443: connect: network is unreachable" image="registry.aliyuncs.com/google_containers/pause:3.7"
FATA[0000] pulling image: rpc error: code = Unknown desc = Get "https://dockerauth.cn-hangzhou.aliyuncs.com/auth?scope=repository%3Agoogle_containers%2Fpause%3Apull&service=registry.aliyuncs.com%3Acn-hangzhou%3A26842": dial tcp [2408:4005:1000:10::2]:443: connect: network is unreachable


```
