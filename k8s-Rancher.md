# k8s布署Rancher管理器

#### 1.部署cert-manager

由于Rancher Manager Server默认需要SSL/TLS配置来保证访问的安全性。所以需要部署cert-manager，用于自动签发证书使用。

也可以使用真实域名及真实域名证书。

```
wget https://github.com/cert-manager/cert-manager/releases/download/v1.11.0/cert-manager.crds.yaml
kubectl apply -f cert-manager.crds.yaml
customresourcedefinition.apiextensions.k8s.io/clusterissuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/challenges.acme.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificaterequests.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/issuers.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/certificates.cert-manager.io created
customresourcedefinition.apiextensions.k8s.io/orders.acme.cert-manager.io created

```

###### 更新helm

```
helm repo add jetstack https://charts.jetstack.io
"jetstack" has been added to your repositories

helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "jetstack" chart repository
...Successfully got an update from the "rancher-stable" chart repository
Update Complete. ⎈Happy Helming!⎈

```

###### 安装cert-manager

```
helm install cert-manager jetstack/cert-manager  --namespace cert-manager --create-namespace --version v1.11.0
```

###### 检查安装

```
kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-6b4d84674-c29gd               1/1     Running   0          8m39s
cert-manager-cainjector-59f8d9f696-trrl5   1/1     Running   0          8m39s
cert-manager-webhook-56889bfc96-59ddj      1/1     Running   0          8m39s

```



#### 2.部署Rancher

创建NS

```
kubectl create namespace cattle-system
namespace/cattle-system created

kubectl get ns
NAME              STATUS   AGE
cattle-system     Active   8s
cert-manager      Active   4m8s
default           Active   23d
ingress-nginx     Active   21d
kube-flannel      Active   23d
kube-node-lease   Active   23d
kube-public       Active   23d
kube-system       Active   23d


```

安装

~~~bash
helm repo add rancher-stable https://releases.rancher.com/server-charts/stable
helm repo update

helm install rancher rancher-stable/rancher \
  --namespace cattle-system \
  --set hostname=local.tradingvisual.com \
  --set bootstrapPassword=admin \
  --set ingress.tls.source=rancher 

NAME: rancher
LAST DEPLOYED: Thu May 16 04:36:22 2024
NAMESPACE: cattle-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
Rancher Server has been installed.

NOTE: Rancher may take several minutes to fully initialize. Please standby while Certificates are being issued, Containers are started and the Ingress rule comes up.

Check out our docs at https://rancher.com/docs/

If you provided your own bootstrap password during installation, browse to https://local.tradingvisual.com to get started.

If this is the first time you installed Rancher, get started by running this command and clicking the URL it generates:

```
echo https://local.tradingvisual.com/dashboard/?setup=$(kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}')
```

To get just the bootstrap password on its own, run:

```
kubectl get secret --namespace cattle-system bootstrap-secret -o go-template='{{.data.bootstrapPassword|base64decode}}{{ "\n" }}'
```


Happy Containering!
~~~

检查（初始化完成可能需要较长时间）

```
kubectl -n cattle-system rollout status deploy/rancher
Waiting for deployment "rancher" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "rancher" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "rancher" rollout to finish: 2 of 3 updated replicas are available...
deployment "rancher" successfully rolled out

kubectl get pods -n cattle-system
rancher-85ff7d6b5-22dl7            1/1     Running     0          30m
rancher-85ff7d6b5-jfjft            1/1     Running     0          30m
rancher-85ff7d6b5-q4jth            1/1     Running     0          30m
rancher-webhook-7cb964c7f6-ws5qc   1/1     Running     0          16m

kubectl -n cattle-system get deploy rancher
NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
rancher   3         3         3            3           3m

```

输出以上信息表示成功

#### 3.访问Rancher

添加对433端口的映射

```
kubectl patch svc rancher -n cattle-system -p '{"spec":{"type":"NodePort","ports":[{"port":443,"targetPort":443,"nodePort":30433}]}}'
```

打开访问：https://192.168.1.61:30433/