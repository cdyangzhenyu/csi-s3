## CSI-S3插件修改说明

[CSI-S3](https://github.com/ctrox/csi-s3)插件是利用K8S的CSI接口实现，其功能是将符合S3接口规范的对象存储bucket当做PVC挂载给pod使用，这样做的好处是即满足对象存储方便的读写操作（各种客户端，随处可读写），又满足了k8s pod的文件存储读写（使用标准的文件接口读写，如cp，mv，cat，echo，rm，vim等）。但是目前该插件存在一些问题：

- 无法指定bucket挂载，即通过其他接口创建或者已经存在的bucket无法挂载给pod。只支持新建一个bucket。
- 无法做到共享bucket挂载，即将同一个bucket挂载给多个pod（包括多个不同namespace的pod），来满足文件读写共享的需求。
- 无法挂载bucket的根目录，默认有csi-fs的前缀。

为了实现可以指定bucket挂载，并且可以挂载多个pod的需求，做了以下修改：

- 在创建pv时新增bucket字段，可以指定一个已有的bucket，如果不指定会使用pv的名字作为bucket，如果该名字的bucket不存在则会创建一个新的，如果指定的bucket也不存在，也会创建一个新的。这样就可以在创建pv的时候指定该pv对应的bucket，之后pod启动后就会挂载该bucket到对应的目录中。
- 删除了依赖metadata判断bucket是否存在的逻辑，该逻辑不利于外部创建的bucket挂载，因为在这种情况下，外部创建的bucket必须写入csi-s3需要的metadata文件，紧耦合太严重。这里改成了只判断bucket是否存在，不存在则创建，存在则使用。
- 将默认挂载csi-fs目录改成了挂载根目录

在使用方式上兼容之前的版本，即可以只指定pvc创建，这样就会新建一个随机名称的pv及bucket。另外也可以同时指定pv和pvc的配置，指定新建的bucket名字，或者指定要挂载的bucket名字。

具体修改代码可查看该[提交](https://github.com/cdyangzhenyu/csi-s3/commit/e4423719a010fd3886f5d8d9b77825262aa6c8e2)

另外CSI查看开发参考：
```
https://github.com/kubernetes-csi/csi-driver-host-path
https://kubernetes-csi.github.io/docs/drivers.html#production-drivers
https://github.com/container-storage-interface/spec/blob/master/lib/go/csi/csi.pb.go
https://github.com/kubernetes-csi/csi-driver-nfs
https://cloud-provider-vsphere.sigs.k8s.io/container_storage_interface.html
http://dy.163.com/v2/article/detail/E278D02S05119IE4.html
```

## 安装说明
安装方式与官方的csi-s3一样，区别是使用share-mounter的镜像：`aiven86/csi-s3:v1.1.1`。另外注意：

- 要开启docker的`MountFlags=shared`。
- 驱动推荐使用s3fs，支持的操作多，经过验证没有问题。

## 使用说明

目前支持以下几种使用场景：

- 场景一：已经存在一个bucket，可以把这个bucket挂给不同的pod共享使用，该bucket也同时可以使用外部客户端进行读写。
- 场景二：通过创建pv来新建一个bucket，后续利用指定bucket挂载的特性来共享该bucket。可用在需要共享文件系统的场景。

### 场景一

下面来实验测试一下这种场景如何使用。

- 首先使用s3cmd创建一个bucket

```
export AWS_ACCESS_KEY_ID=1PPZY1HD15E40JJ6R8VK
export AWS_SECRET_ACCESS_KEY=FwiEmRoJthMWZhtQ86eCVuKpDoN7UTOirk0hO9sf

s3cmd --no-ssl --host=10.1.47.23:30362  mb S3://BUCKET
```
- 使用该bucket分别创建不同namespace的pv，pvc，pod

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-bucket
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  csi:
    controllerPublishSecretRef:
      name: csi-s3-secret
      namespace: kube-system
    driver: ch.ctrox.csi.s3-driver
    fsType: ext4
    nodePublishSecretRef:
      name: csi-s3-secret
      namespace: kube-system
    nodeStageSecretRef:
      name: csi-s3-secret
      namespace: kube-system
    volumeAttributes:
      mounter: s3fs
      bucket: BUCKET
    volumeHandle: test-bucket
  persistentVolumeReclaimPolicy: Delete
  storageClassName: csi-s3
  volumeMode: Filesystem

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-bucket
  namespace: default
spec:
  accessModes:
  - ReadWriteMany
  dataSource: null
  resources:
    requests:
      storage: 5Gi
  storageClassName: csi-s3
  volumeMode: Filesystem
  volumeName: test-bucket

---
apiVersion: v1
kind: Pod
metadata:
  name: csi-s3-test-nginx
  namespace: default
spec:
  containers:
   - name: csi-s3-test-nginx
     image: nginx
     imagePullPolicy: IfNotPresent
     volumeMounts:
       - mountPath: /var/lib/www/html
         name: webroot
  volumes:
   - name: webroot
     persistentVolumeClaim:
       claimName: test-bucket
       readOnly: false

```
其他namespace下
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: test-bucket-2
spec:
  accessModes:
  - ReadWriteMany
  capacity:
    storage: 5Gi
  csi:
    controllerPublishSecretRef:
      name: csi-s3-secret
      namespace: kube-system
    driver: ch.ctrox.csi.s3-driver
    fsType: ext4
    nodePublishSecretRef:
      name: csi-s3-secret
      namespace: kube-system
    nodeStageSecretRef:
      name: csi-s3-secret
      namespace: kube-system
    volumeAttributes:
      mounter: s3fs
      bucket: BUCKET
    volumeHandle: test-bucket-2
  persistentVolumeReclaimPolicy: Delete
  storageClassName: csi-s3
  volumeMode: Filesystem

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: test-bucket-2
  namespace: test
spec:
  accessModes:
  - ReadWriteMany
  dataSource: null
  resources:
    requests:
      storage: 5Gi
  storageClassName: csi-s3
  volumeMode: Filesystem
  volumeName: test-bucket-2

---
apiVersion: v1
kind: Pod
metadata:
  name: csi-s3-test-nginx-2
  namespace: test
spec:
  containers:
   - name: csi-s3-test-nginx
     image: nginx
     imagePullPolicy: IfNotPresent
     volumeMounts:
       - mountPath: /var/lib/www/html
         name: webroot
  volumes:
   - name: webroot
     persistentVolumeClaim:
       claimName: test-bucket-2
       readOnly: false
```

- 查看资源创建是否成功

```
[root@cloud ~]# kubectl get pods
NAME                READY   STATUS    RESTARTS   AGE
csi-s3-test-nginx   1/1     Running   0          3m29s

[root@cloud ~]# kubectl get pvc 
NAME           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-bucket    Bound    test-bucket                                5Gi        RWX            csi-s3         5m52s

[root@cloud ~]# kubectl exec -it csi-s3-test-nginx -- df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          47G   21G   27G  44% /
tmpfs            64M     0   64M   0% /dev
tmpfs           7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/vda3        47G   21G   27G  44% /etc/hosts
shm              64M     0   64M   0% /dev/shm
s3fs            256T     0  256T   0% /var/lib/www/html

[root@cloud ~]# kubectl get pods -n test
NAME                  READY   STATUS    RESTARTS   AGE
csi-s3-test-nginx-2   1/1     Running   0          3m36s

[root@cloud ~]# kubectl get pvc -n test
NAME            STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-bucket-2   Bound    test-bucket-2   5Gi        RWX            csi-s3         5m11s

[root@cloud ~]# kubectl -n test exec -it csi-s3-test-nginx-2 -- df -h
Filesystem      Size  Used Avail Use% Mounted on
overlay          47G   21G   27G  44% /
tmpfs            64M     0   64M   0% /dev
tmpfs           7.8G     0  7.8G   0% /sys/fs/cgroup
/dev/vda3        47G   21G   27G  44% /etc/hosts
shm              64M     0   64M   0% /dev/shm
s3fs            256T     0  256T   0% /var/lib/www/html

```

- 验证共享文件读写

```
[root@cloud ~]# kubectl exec -it csi-s3-test-nginx bash
root@csi-s3-test-nginx:/# cd /var/lib/www/html/
root@csi-s3-test-nginx:/var/lib/www/html# ls
root@csi-s3-test-nginx:/var/lib/www/html# echo "test" > test
root@csi-s3-test-nginx:/var/lib/www/html# ls
test
root@csi-s3-test-nginx:/var/lib/www/html# cat test 
test

[root@cloud ~]# kubectl -n test exec -it csi-s3-test-nginx-2 bash
root@csi-s3-test-nginx-2:/# cd /var/lib/www/html/
root@csi-s3-test-nginx-2:/var/lib/www/html# ls
test
root@csi-s3-test-nginx-2:/var/lib/www/html# cat test 
test

[root@cloud ~]# s3cmd ls --no-ssl --host=10.1.47.23:30362 --host-bucket= s3://BUCKET 
2020-03-10 13:56         5   s3://BUCKET/test

[root@cloud ~]# s3cmd get --no-ssl --host=10.1.47.23:30362 --host-bucket= s3://BUCKET/test
download: 's3://BUCKET/test' -> './test'  [1 of 1]
 5 of 5   100% in    0s   368.65 B/s  done
 
[root@cloud ~]# cat test 
test

```

### 场景二

场景二与一的主要区别是不需要事先创建好bucket，csi-s3插件会判断如果不存在bucket的时候自动创建bucket。这里有两种情况：

- 只需要pvc的配置，这种会自动创建pv，pv的名字都是以pvc-开头，这种方式在使用的时候比较方便（不需要自己创建pv），但是不能自定义bucket的名称。
- 需要创建pv，指不指定bucket参数都可以，指定bucket参数，则新创建的bucket就是这个参数的名字。如果不指定则新创建的bucket名字就是pv的名字。

## 问题排查

通过查看对应节点的插件pod日志，可以看到在创建bucket，挂载目录等一系列操作对应的日志。
```
[root@cloud ~]# kubectl -n kube-system logs  csi-s3-9lbpr -c csi-s3 -f
I0310 07:57:58.272078       1 s3-driver.go:80] Driver: ch.ctrox.csi.s3-driver 
I0310 07:57:58.272307       1 s3-driver.go:81] Version: v1.1.1 
I0310 07:57:58.272321       1 driver.go:81] Enabling controller service capability: CREATE_DELETE_VOLUME
I0310 07:57:58.272327       1 driver.go:93] Enabling volume access mode: SINGLE_NODE_WRITER
I0310 07:57:58.272655       1 server.go:108] Listening for connections on address: &net.UnixAddr{Name:"//csi/csi.sock", Net:"unix"}

```

## 遗留问题（TODO）
- 经过测试，目前应该不支持容量的限制，即pv和pvc指定的size应该是没有实际作用的，且pod中的挂载容量也是不正确的。
- 无法指定bucket的某个特定目录来挂载，做到目录之间的隔离。