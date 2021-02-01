FATE-Manaager部署FATE1.5

#安装ingress controlller
 

参考: NGINX Docs | Installation with Manifests

https://docs.nginx.com/nginx-ingress-controller/installation/installation-with-manifests/

kubernetes-ingress/installation.md at v1.6.0 · nginxinc/kubernetes-ingress

https://github.com/nginxinc/kubernetes-ingress/blob/v1.6.0/docs/installation.md

wget https://github.com/nginxinc/kubernetes-ingress/archive/v1.6.0.tar.gz

kubectl apply -f common/ns-and-sa.yaml

kubectl apply -f common/default-server-secret.yaml

kubectl apply -f common/nginx-config.yaml

kubectl apply -f rbac/rbac.yaml

kubectl apply -f deployment/nginx-ingress.yaml


kubectl get pods --namespace=nginx-ingress


FATEBOARD

http://50000.fateboard.kubefate.net:17657/#/history

NOTEBOOK

http://50000.notebook.kubefate.net:17657/tree?


kubefate

http://kubefate.net:17657/



#fate-manager自动部署站点

1. 加载已手工安装之1.5.0版本的FATE cluster

自动部署界面选择connect链接，输入http://172.32.150.133:35050（参看fate-manager/conf/app.ini)

KubeFateUrl=http://172.32.150.133:35050


修改t_fate_kubenetes_conf表node_list字段内容为：

fm-node-0:172.32.150.134,fm-node-0:172.32.150.135


将demo/config.yaml文件复制到fate_manager启动目录下


修改services/component_deploy_service/component_deploy_service.go如下：

20	//      "bytes" 

...

329	// index := bytes.IndexByte([]byte(result.Body), 0)

330	err = json.Unmarshal([]byte(result.Body), &clusterQueryResp)

重新编译  sudo go build


在t_fate_auto_test表中将status字段值都改成2，跳过Auto-test步骤

修改t_fate_site_info表以下内容：

fate_version: 1.5.0

component_version: {"clustermanager":"1.5.0-release","fateboard":"1.5.0-release","fateflow":"1.5.0-release","mysql":"8","nodemanager":"1.5.0-release","rollsite":"1.5.0-release"}


2. 部署新站点FATE1.4.4

wget https://github.com/FederatedAI/KubeFATE/releases/download/v1.4.4/fate-v1.4.4.tgz

kubefate chart upload -f charts/fate-v1.4.4.tgz

[app@vcapp133 demo]$ kubefate chart ls

UUID                                	NAME	VERSION	APPVERSION
6f36e56f-e1eb-46b0-83a2-244ae2e029af	fate	v1.5.0 	v1.5.0    
74445b08-4ef1-480a-a87d-baf0b9a1fca0	fate	v1.4.4 	v1.4.4 

自动部署界面选择start deploying链接，输入http://172.32.150.133:35050（参看fate-manager/conf/app.ini)

Start

Version选择1.4.4

点击pull下载镜像

Next

Start 启动镜像安装

Next

Start 启动Auto-test（跳过）

Finish


#在线升级FATE1.5，点击Upgrade FATE，


手工将t_fate_deploy_site表对应记录的status字段值改为2. （因为前面手工跳过了Auto-Test）

参考：fate-manager/src/comm/enum/siteRunStatus.go

SITE_RUN_STATUS_UNKNOWN SiteRunStatusType = -1

SITE_RUN_STATUS_STOPPED SiteRunStatusType = 1

SITE_RUN_STATUS_RUNNING SiteRunStatusType = 2



 在entity/kubefate.go中添加以下代码：
 
type ClusterConfig150 struct {
        ChartName    string      `json:"chartName"`
        ChartVersion string      `json:"chartVersion"`
        Istio        Istio       `json:"istio"`
        Modules      []string    `json:"modules"`
        Name         string      `json:"name"`
        NameSpace    string      `json:"nameSpace"`
        SrcPartyId   int         `json:"partyId"`
        Persistence  bool        `json:"persistence"`
        PullPolicy   PullPolicy  `json:"pullPolicy"`
        Registry     string      `json:"registry"`
        Backend	string		`json:"backend"`
}

    修改services/site_deploy_service/site_deploy_service.go：

    151行：
        clusterConfigNew2 := entity.ClusterConfig150{
                ChartName:    "fate",
                ChartVersion: chartVersion,
                Istio:        entity.Istio{Enabled: false},
                Modules:      modules,
                Name:      name,
                NameSpace: nameSpace,
                SrcPartyId:  site.PartyId,
                Persistence: false,
                PullPolicy:  entity.PullPolicy{},
                Registry: setting.KubenetesSetting.Registry,
                Backend: "eggroll"
        }

        var json = jsoniter.ConfigCompatibleWithStandardLibrary
        valBj, err := json.Marshal(clusterConfig)
        if err != nil {
                return nil, err
        }

        if versionIndex >= 150 {
                valBj, err = json.Marshal(clusterConfigNew2)
                if err != nil {
                        return nil, err
                }
        } else {
		    if versionIndex >= 140 {
                		valBj, err = json.Marshal(clusterConfigNew)
                		if err != nil {
                        		return nil, err
                		}
        	    } 
        }

#直接安装1.5FATE

修改fate-manager/src/services/job_service/job_service.go：
72行		result, err := http.GET(http.Url(kubenetesUrl+"/v1/job/"+deployJobList[i].JobId), nil, head)
              if err != nil || result == nil {
                      continue
              }
              var jobQueryResp entity.JobQueryResp
              //index := bytes.IndexByte([]byte(result.Body), 0)
              err = json.Unmarshal([]byte(result.Body), &jobQueryResp)


部署过程：

start deploying -》 http://172.32.150.133:35050 -》Start -》Verion1.5.0 Pull Next -》Start Next -》

t_fate_auto_test表中将status字段值都改成2，跳过Auto-test步骤:) -》Finish

