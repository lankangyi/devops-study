1.设置virtualBox 代理端口

2.查看ip route
```shell
ip rout
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
192.168.7.0/24 dev eth1 proto kernel scope link src 192.168.7.4 metric 101 
192.168.104.0/26 via 192.168.7.3 dev tunl0 proto bird onlink 
blackhole 192.168.135.0/26 proto bird 
192.168.166.128/26 via 192.168.7.2 dev tunl0 proto bird onlink 
```
3.永久修改代理服务
```shell

vi ~/.bashrc

export https_proxy="http://10.0.2.2:7890"
export http_proxy="http://10.0.2.2:7890"
export no_proxy="127.0.0.1/8,10.0.0.1/8,192.168.7.4/24,192.168.135.0/32,localhost,master.lky.io,node1,node2,node3,localhost.localdomain,::1"
source  ~/.bashrc


```
4.修改containerd 配置
```shell
    vi /etc/systemd/system/containerd.service

#添加以下配置
Environment="HTTP_PROXY=http://10.0.2.2:7890"
Environment="HTTPS_PROXY=http://10.0.2.2:7890"
Environment="NO_PROXY=127.0.0.1/8,10.0.0.0/8,192.168.0.0/16,localhost,master.lky.io,node1,node2,node3,localhost.localdomain,::1,.svc.cluster.local"
systemctl daemon-reload
systemctl enable containerd --now 
systemctl status containerd
```

5.在deployment的时候添加环境变量拉取镜像
```yaml
    - name: http_proxy
      value: http://10.0.2.2:7890
    - name: https_proxy
      value: http://10.0.2.2:7890
    - name: no_proxy
      value: localhost,127.0.0.1,localaddress,.localdomain.com,master.lky.io,node1.cluster.local,node2.cluster.local,node1,node2,node3
```


6.ctr操作
```shell
ctr -n k8s.io image pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.53.0@sha256:6445641a3ea1afa918f92ef51d18edf1bdb40c08878374db33e290221e3564e1
ctr -n k8s.io image pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.53.0@sha256:340e1edd0783bdb86e396ef53499f068a42da1986a1d806ab652b448869637bd
ctr -n k8s.io image pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.53.0@sha256:9cef507c33127c488938fd6af10c0c2242b4b667732e488545338f290025fa08
ctr -n k8s.io image pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.53.0@sha256:da5cefe4b29a2c0904ccdae95d5bf668068c2ac1fc1618c239425fc616360568

crictl pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.53.0@sha256:6445641a3ea1afa918f92ef51d18edf1bdb40c08878374db33e290221e3564e1
crictl pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.53.0@sha256:340e1edd0783bdb86e396ef53499f068a42da1986a1d806ab652b448869637bd
crictl pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.53.0@sha256:9cef507c33127c488938fd6af10c0c2242b4b667732e488545338f290025fa08
crictl pull  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.53.0@sha256:da5cefe4b29a2c0904ccdae95d5bf668068c2ac1fc1618c239425fc616360568

ctr -n k8s.io image  tag gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.53.0@sha256:9cef507c33127c488938fd6af10c0c2242b4b667732e488545338f290025fa08  gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/controller:v0.53.0
ctr -n k8s.io image  tag gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.53.0@sha256:340e1edd0783bdb86e396ef53499f068a42da1986a1d806ab652b448869637bd gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/events:v0.53.0
ctr -n k8s.io image  tag gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.53.0@sha256:6445641a3ea1afa918f92ef51d18edf1bdb40c08878374db33e290221e3564e1 gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/resolvers:v0.53.0
ctr -n k8s.io image  tag gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.53.0@sha256:da5cefe4b29a2c0904ccdae95d5bf668068c2ac1fc1618c239425fc616360568 gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/webhook:v0.53.0
```

7.重新初始化集群
```shell
   kubeadm init --config kubeadm-conf.yaml --upload-certs=true
   kubectl taint nodes --all node-role.kubernetes.io/master-
   ls
   kubectl create -f tigera-operator.yaml 
   kubectl create -f custom-resources.yaml 
```
8.收到拉取镜像
```shell
export https_proxy="http://10.0.2.2:7890"
export http_proxy="http://10.0.2.2:7890"
ctr -n k8s.io image pull gcr.io/tekton-nightly/github.com/tektoncd/pipeline/cmd/entrypoint:v20231108-15c81688ba@sha256:8dccc02cbb20f6209d9a8964c7ae6e65db04da1b8bbf1cca7f30646bce29dc8a
unset http_proxy
unset https_proxy
ctr -n k8s.io image  tag gcr.io/tekton-nightly/github.com/tektoncd/pipeline/cmd/entrypoint:v20231108-15c81688ba@sha256:8dccc02cbb20f6209d9a8964c7ae6e65db04da1b8bbf1cca7f30646bce29dc8a gcr.io/tekton-nightly/github.com/tektoncd/pipeline/cmd/entrypoint:v20231108-15c81688ba
```

9.nfs provider
```shell
helm repo add kubesphere https://charts.kubesphere.io/main
helm install my-nfs-client-provisioner kubesphere/nfs-client-provisioner --version 4.0.11 -f values.yaml
```