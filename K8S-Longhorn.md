# 在K8S上部署Longhorn存储



#### 1.在集群所有节点上添加硬盘并挂载目录

```bash
vgcreate lvmvg /dev/sdb
mkfs.ext4 /dev/sdb
mkdir mkdir /mnt/longhorn-sdb
echo '/dev/sdb /mnt/longhorn-sdb ext4 defaults 0 1' >> /etc/fstab
mount -a

```



#### 2.下载longhorn.yaml并安装

```bash
wget https://raw.githubusercontent.com/longhorn/longhorn/v1.6.0/deploy/longhorn.yaml
kubectl apply -f  longhorn.yaml
```



#### 3.检查安装及等待Pod完成（需要较长时间）

```bash
kubectl get pods -n longhorn-system
NAME                                                READY   STATUS    RESTARTS      AGE
csi-attacher-57c5fd5bdf-85j78                       1/1     Running   2 (27m ago)   36m
csi-attacher-57c5fd5bdf-hhjp7                       1/1     Running   3 (23m ago)   36m
csi-attacher-57c5fd5bdf-s42w7                       1/1     Running   6 (28m ago)   36m
csi-provisioner-7b95bf4b87-966vz                    1/1     Running   0             36m
csi-provisioner-7b95bf4b87-tqczj                    1/1     Running   2 (24m ago)   36m
csi-provisioner-7b95bf4b87-vbdlj                    1/1     Running   2 (27m ago)   36m
csi-resizer-6df9886858-4zfv7                        1/1     Running   2 (24m ago)   36m
csi-resizer-6df9886858-c6nrk                        1/1     Running   1 (27m ago)   36m
csi-resizer-6df9886858-k2nhq                        1/1     Running   0             36m
csi-snapshotter-5d84585dd4-8swxh                    1/1     Running   0             36m
csi-snapshotter-5d84585dd4-lxm2k                    1/1     Running   0             36m
csi-snapshotter-5d84585dd4-rwrbt                    1/1     Running   1 (24m ago)   36m
engine-image-ei-acb7590c-c6j7m                      1/1     Running   0             36m
engine-image-ei-acb7590c-pmsd5                      1/1     Running   0             36m
engine-image-ei-acb7590c-x6cwg                      1/1     Running   0             36m
engine-image-ei-acb7590c-x8g8p                      1/1     Running   0             36m
instance-manager-2a9806394e82a26a685d53b0091675a9   1/1     Running   0             36m
instance-manager-34e030845736556d555f6c90a8c6e364   1/1     Running   0             36m
instance-manager-71e376513a004455bd187189ca16c4cc   1/1     Running   0             36m
instance-manager-f756536f9ab93dcf67fb4e3d8caefbea   1/1     Running   0             36m
longhorn-csi-plugin-dvzdw                           3/3     Running   1 (26m ago)   36m
longhorn-csi-plugin-h9ghw                           3/3     Running   0             36m
longhorn-csi-plugin-ltlmv                           3/3     Running   1 (24m ago)   36m
longhorn-csi-plugin-nsqq7                           3/3     Running   1 (27m ago)   36m
longhorn-driver-deployer-85cb455b98-ccp6s           1/1     Running   1 (35m ago)   37m
longhorn-manager-5qt9h                              1/1     Running   0             37m
longhorn-manager-s6djw                              1/1     Running   0             37m
longhorn-manager-ss7lb                              1/1     Running   1 (36m ago)   37m
longhorn-manager-z86ts                              1/1     Running   0             37m
longhorn-ui-59b6f75f5d-7xgg4                        1/1     Running   0             37m
longhorn-ui-59b6f75f5d-cv4dm                        1/1     Running   0             37m

```

确认所有的Pod都处于"Running"状态。表示初始化安装完成。



#### 4.查看存储类

```bash
kubectl get sc
NAME                 PROVISIONER          RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
longhorn (default)   driver.longhorn.io   Delete          Immediate           true                   47m

```



#### 5.配置svc服务

```bash
kubectl get svc -n longhorn-system
longhorn-frontend             ClusterIP   10.106.154.54    <none>        80/TCP      55m

kubectl edit svc longhorn-frontend -n longhorn-system
type: NodePort    #将type的ClusterIP改为 NodePort 即可

#查询刚才变更为 NodePort 暴露的端口
kubectl get svc -n longhorn-system|grep longhorn-frontend 
longhorn-frontend             NodePort    10.106.154.54    <none>        80:32146/TCP   58m
```

安装好后名为longhorn-frontend的svc服务默认是clusterip模式，除了集群之外的网络是访问不到此服务的，所以要将此svc服务改为nodeport模式。

在浏览器访问此端口即可（根据自己的k8s-master宿主机ip地址输入）

[http://192.168.1.61:32146](http://192.168.1.61:32146/) 



#### 6.配置存储卷

存储类配置完成后，你可以通过以下来创建持久卷。

```bash
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: plik-longhorn-pvc
spec:
  storageClassName: longhorn
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
EOF


#查看
kubectl get pvc
NAME                        STATUS    VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS             AGE
plik-longhorn-pvc           Pending                                         mayastor-sc              4s
```



#### 7.使用存储卷

```bash
vim test.yaml
#输入以下内容
kind: Pod
apiVersion: v1
metadata:
  name: longhorn-test
spec:
  volumes:
    - name: ms-volume
      persistentVolumeClaim:
        claimName: plik-longhorn-pvc
  containers:
    - name: fio
      image: nixery.dev/shell/fio
      args:
        - sleep
        - "1000000"
      volumeMounts:
        - mountPath: "/volume"
          name: ms-volume

#使生效
kubectl apply -f test.yaml

```

