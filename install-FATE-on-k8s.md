1. 要有一个k8s集群，安装过程略（我是从已有的k8s集群上安装的）

以下操作都是在k8s集群master节点（172.32.150.133）上操作的，除非有特殊说明（如，所有节点.....)

2. 到https://github.com/FederatedAI/KubeFATE/releases下载安装包

fate-v1.3.0-a.tgz

kubefate-k8s-v1.3.0-a.tar.gz

3. 到https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/fate_1.3.0-images.tar.gz下载镜像文件

再传到k8s集群的每个节点上，通过 docker load -i fate_1.3.0-images.tar.gz 加载镜像

包含以下镜像：

# docker images

REPOSITORY                         TAG 

federatedai/egg                    1.3.0-release

federatedai/fateboard              1.3.0-release

federatedai/meta-service           1.3.0-release

federatedai/python                 1.3.0-release

federatedai/roll                   1.3.0-release

federatedai/proxy                  1.3.0-release

federatedai/federation             1.3.0-release

federatedai/serving-server         1.2.2-release

federatedai/serving-proxy          1.2.2-release

redis                              5

mysql                              8

4. # tar -zxvf kubefate-k8s-v1.3.0-a.tar.gz 

kubefate

cluster.yaml

config.yaml

kubefate.yaml

rbac-config.yaml

5. # kubectl apply -f ./rbac-config.yaml

namespace/kube-fate created

serviceaccount/kubefate-admin created

clusterrolebinding.rbac.authorization.k8s.io/kubefate created

6. 在kube-fate命令空间下安装ingress，过程略（集成同学帮忙安装的）

7. # kubectl apply -f ./kubefate.yaml

deployment.apps/kubefate created

deployment.apps/mongo created

service/mongo created

service/kubefate created

ingress.extensions/kubefate created

8. # kubectl get pod,ingress -n kube-fate

NAME                                            READY     STATUS    RESTARTS   AGE

pod/kubefate-7b877c5bd6-vdlpw                   1/1       Running   0          21h

pod/mongo-985bf7548-jh5pz                       1/1       Running   1          6d

pod/nginx-ingress-controller-7d8c8df946-fw5v9   1/1       Running   2          6d


NAME                          HOSTS          ADDRESS         PORTS     AGE

ingress.extensions/kubefate   kubefate.net   10.68.141.234   80        6d

# kubectl get node -n kube-fate

NAME             STATUS    ROLES     AGE       VERSION

172.32.150.133   Ready     master    303d      v1.10.11

172.32.150.134   Ready     node      303d      v1.10.11

172.32.150.135   Ready     node      303d      v1.10.11

9. 在所有节点上设置kubefate.net对应的host,修改/etc/hosts文件，添加

172.32.150.135 kubefate.net

10. # chmod +x ./kubefate && sudo mv ./kubefate /usr/local/bin/kubefate

11. # kubefate version

* kubefate service version=v1.0.2

* kubefate commandLine version=v1.0.2

此处注意，如果本机设置有代理如环境变量http_proxy或https_proxy,则需要同时设置no_proxy,

export no_proxy=kubefate.net,172.32.150.133,172.32.150.134,172.32.150.135

即k8s集群内访问不走代理

12. 假设我们要建的两方是fate-8888和fate-9999,将clusteryaml分别复制成fate-8888.yaml和fate-9999.yaml，并修改如下：

# cat fate-8888.yaml 
name: fate-8888

namespace: fate-8888

chartName: fate

chartVersion: v1.3.0-a

partyId: 8888

modules:

  - proxy

  - egg

  - federation

  - metaService

  - mysql

  - redis

  - roll

  - python


proxy:

  type: NodePort

  nodePort: 30008

  partyList:

    - partyId: 9999

      partyIp: 172.32.150.135

      partyPort: 30009

egg:

  count: 3

# cat fate-9999.yaml 

name: fate-9999

namespace: fate-9999

chartName: fate

chartVersion: v1.3.0-a

partyId: 9999

modules:

  - proxy

  - egg

  - federation

  - metaService

  - mysql

  - redis

  - roll

  - python


proxy:

  type: NodePort

  nodePort: 30009

  partyList:

    - partyId: 8888

      partyIp: 172.32.150.134

      partyPort: 30008

egg:

  count: 3

13. 分别创建命名空间fate-8888和fate-9999:

#  kubectl create namespace fate-8888

#  kubectl create namespace fate-9999

14. 构建私有Chart库（由于官方chart库https://federatedai.github.io/KubeFATE/连不上，网络环境差）

（1）安装helm，详细可参考本目录下另一篇文章《小试helm》：

从https://github.com/helm/helm/releases下载helm-v3.0.0-linux-amd64.tar.gz（其他版本亦可）

# tar -zxvf helm-v3.0.0-linux-amd64.tar.gz

# mv linux-amd64/helm /usr/local/bin/helm

（2）# mkdir charts

# mv fate-v1.3.0-a.tgz ./charts

# cd charts

（3）启一个本地的http server

# python -m SimpleHTTPServer 8879

（4）# cd ..

生成chars/index.yaml文件

# helm repo index charts --url https://172.32.150.133:8879   

（5）# kubefate chart ls

UUID                                	NAME	VERSION 	APPVERSION

61b885d7-e281-4fad-80a9-5f4fd7d0691b	fate	v1.3.0-a	v1.3.0  

13. 安装集群：

# kubefate cluster install -f ./fate-8888.yaml

# kubefate cluster install -f ./fate-9999.yaml

查看 cluster job日志，通过kubefate job describe JOBID (JOBID在上述命令后有提示）

# kubectl logs -f pod/kubefate-7b877c5bd6-vdlpw -n kube-fate

14. 所有主机节点修改/etc/hosts文件：

172.32.150.135   vcapp135 kubefate.net 9999.fateboard.kubefate.net 8888.fateboard.kubefate.net

15. 测试toy-example

# kubectl get pod -n fate-9999

NAME                           READY     STATUS    RESTARTS   AGE

egg0-f59f97ff6-kr759           1/1       Running   0          5h

egg1-57898cd579-lwh77          1/1       Running   0          5h

egg2-858fc99f4-mmt94           1/1       Running   0          5h

federation-58c5867d6b-zmxbb    1/1       Running   0          5h

meta-service-7dd88c8fc-lpq6w   1/1       Running   0          5h

mysql-5f575c7947-6nbvg         1/1       Running   0          5h

proxy-67b8665f75-c6kpw         1/1       Running   0          5h

python-8786bcf8d-q6dpt         2/2       Running   0          5h

redis-79ddf9c8d6-vm6lh         1/1       Running   0          5h

roll-6579778b9c-d9wqx          1/1       Running   0          5h

# kubectl exec -it python-8786bcf8d-q6dpt -c python -n fate-9999 -- bash

(venv) [root@python-8786bcf8d-q6dpt python]# cd examples/toy_example/

(venv) [root@python-8786bcf8d-q6dpt python]# python run_toy_example.py 9999 8888 1

通过FATEBoard查看，对kubeFATE1.3需要修改fateboard服务将type由ClusterIP改成NodePort

# kubectl edit svc/fateboard -n fate-8888

# kubectl edit svc/fateboard -n fate-9999

# kubectl get svc -n fate-8888|grep fateboard

fateboard      NodePort    10.68.97.119    <none>        8080:24692/TCP      5h

可通用 http://8888.fateboard.kubefate.net:24692 访问

# kubectl get svc -n fate-9999|grep fateboard

fateboard      NodePort    10.68.31.55     <none>        8080:15519/TCP      6h

可通用 http://9999.fateboard.kubefate.net:15519 访问

