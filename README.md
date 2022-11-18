```
通过ansible在CentOS Stream 8 上面安装和配置k8s所需的包和参数等。
前提条件
1. 在ansible控制节点安装ansible version > 2.4 ##测试版本2.13.5
2. 在各个受控节点配置不需密码的sudo 的用户devops，并在控制节点配置ssh互信。

[root@ansible n_k8s_install]# ls -tlra
total 20
-rw-r--r--. 1 root root  169 Nov 16 23:27 ansible.cfg
-rw-r--r--. 1 root root   64 Nov 16 23:27 inventory
-rw-r--r--. 1 root root   84 Nov 16 23:28 requirements.yml
dr-xr-x---. 6 root root 4096 Nov 16 23:29 ..
-rw-r--r--. 1 root root   93 Nov 17 01:41 k8s_install.yml
drwxr-xr-x. 3 root root  102 Nov 17 02:50 .
drwxr-xr-x. 2 root root    6 Nov 17 02:50 roles
====================================================
[root@ansible n_k8s_install]# cat ansible.cfg
[defaults]
inventory=inventory
remote_user=devops
log_path=~/ansible.log



[privilege_escalation]
become=true
become_ask_pass=False
become_method=sudo
become_user=root

=========================================================
[root@ansible n_k8s_install]# cat inventory
[master]
master1

[worker]
worker1
worker2

[newserver]
worker3

######### 配置requirement 文件###############################
[root@ansible n_k8s_install]# cat requirements.yml
---
- name: k8s_install
  src: https://github.com/jingwei269/k8s_install
  scm: git

################# playbook k8s_install.yml ##################
[root@ansible n_k8s_install]# cat k8s_install.yml
---
- name: prepare k8s required tools on new server
  hosts: all

  roles:
   - k8s_install
[root@ansible n_k8s_install]#

========================================================================
运行ansible-galaxy 安装playbook，取放置于github的playbook，有可能install failed，因为网络原因。
[root@ansible n_k8s_install]# ansible-galaxy install -r requirements.yml -p roles/
Starting galaxy role install process
- extracting k8s_install to /root/n_k8s_install/roles/k8s_install
- k8s_install was installed successfully

运行playbook k8s_install.yml
[root@ansible n_k8s_install]# ansible-playbook k8s_install.yml


init k8s
[root@ansible n_k8s_install]# ssh devops@master1 "sudo kubeadm init  --pod-network-cidr 10.244.0.0/16 --kubernetes-version=v1.24.1 --image-repository=registry.aliyuncs.com/google_containers "

token 根据实际情况修改：
[root@ansible n_k8s_install]# ssh devops@worker1 "sudo kubeadm join 192.168.31.22:6443 --token karad7.e2uopy5132ackb96 --discovery-token-ca-cert-hash sha256:b1ab5001fe3569076a7285f4f14bff5d1b0f9b35f5071608af49f42c403cf8ca "
[root@ansible n_k8s_install]# ssh devops@worker3 "sudo kubeadm join 192.168.31.22:6443 --token karad7.e2uopy5132ackb96 --discovery-token-ca-cert-hash sha256:b1ab5001fe3569076a7285f4f14bff5d1b0f9b35f5071608af49f42c403cf8ca "

安装calico
[root@master1 ~]# curl https://docs.projectcalico.org/manifests/calico.yaml | sed  -e '/192/s/# //g' -e '/IPV4POOL_CIDR/s/# //g' -e '/192/s/192\.168/10\.244/g' > calico.yaml
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  229k  100  229k    0     0   335k      0 --:--:-- --:--:-- --:--:--  334k
[root@master1 ~]# kubectl apply -f calico.yaml
```
