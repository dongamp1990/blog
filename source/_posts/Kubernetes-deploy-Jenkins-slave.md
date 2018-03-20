---
title: Kubernetes部署Jenkins并动态资源分配
tags: [kubernetes, jenkins]
---
基于Kubernetes部署Jenkins，JenkinsSlave动态分配

## 部署nfs服务
需要把jenkins的home目录做持久化，解决方案用nfs
  ①用外部nfs
  ②在k8s里面部署一个nfs服务
  我们使用第二种方案
这里还可以把编排到某个node，可以参考<br>https://kubernetes.io/docs/concepts/configuration/assign-pod-node/


### nfs.yaml
```
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: nfs
  labels:
    app: nfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nfs
  template:
    metadata:
      labels:
        app: nfs
    spec:
      containers:
      - name: nfs-server
        image: itsthenetwork/nfs-server-alpine:7
        imagePullPolicy: IfNotPresent
        securityContext:
          privileged: true
        ports:
        - containerPort: 2049
        env:
        - name: SHARED_DIRECTORY
          value: "/nfsshare"
        volumeMounts:
        - name: nfsshare
          mountPath: /nfsshare
      volumes:
      - name: nfsshare
        hostPath:
         # 目录在node节点下
         path: /opt/nfsshare
         # 文件夹如果不存在即创建一个
         type: DirectoryOrCreate
         
---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: nfs
  name: nfs
spec:
  ports:
  - port: 2049
    targetPort: 2049
  selector:
    app: nfs  
```

### 部署nfs
```
$ kubectl apply -f nfs.yaml

#查看nfs服务ip地址，记住ip， 后面用到
$ kubectl get pod -o wide  | grep nfs
nfs-7b689fb468-xsmbf   1/1    Running   0    1h   192.168.166.136   node1

```

## 使用PersistentVolume持久化卷
参考：https://jimmysong.io/kubernetes-handbook/concepts/persistent-volume.html
### jenkins slave 数据持久化卷
jenkins slave 数据持久化卷
cipvc.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: cipv01
  namespace: ci
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /jenkins
    #这里填nfs服务器ip（前面nfs的地址）
    server: 192.168.166.136
    
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cipvc01
  namespace: ci
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  storageClassName: slow
```

部署持久化卷
```      
kubelet apply -f cipvc.yaml
```

### mavan本地仓库久化卷
mvnpvc.yaml
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mvnpv
  namespace: ci
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /mavenLocalRepo
    server: 192.168.166.136

---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mvnpvc
  namespace: ci
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi
  storageClassName: slow
```
部署持久化卷
```
kubelet apply -f mvnpvc.yaml
```

### 查看pcv
```
$ kubectl get pvc -n ci

NAME              CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                 STORAGECLASS   REASON    AGE
cipv01            2Gi        RWX            Recycle          Bound     ci/cipvc01            slow                     16h
mvnpv             2Gi        RWX            Recycle          Bound     ci/mvnpvc             slow                     16h

状态是Bound才可用
```

## 部署jenkins
jenkins.yaml
```
apiVersion: v1
kind: Namespace
metadata:
  name: ci
  
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: ci
  
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: ci
  
---
kind: Deployment
apiVersion: apps/v1beta2
metadata:
  name: jenkins
  namespace: ci
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccount: jenkins
      securityContext:
       runAsUser: 0
      containers:
      - name: jenkins
        image: jenkinsci/blueocean:1.3.5
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8080
          name: web
          protocol: TCP
        env:
        - name: PORT
          value: '8000'
        - containerPort: 50000
          name: agent
          protocol: TCP
        volumeMounts:
        - name: jenkinshome
          mountPath: /var/jenkins_home/
        env:
        - name: JAVA_OPTS
          value: "-Duser.timezone=Asia/Shanghai"
      volumes:
      - name: jenkinshome
      	#pvc模式
        persistentVolumeClaim:
          claimName: cipvc01
        #主机目录模式
        #hostPath:
         # directory location on host
         #path: /opt/jenkins-blueocean-data
         # this field is optional
         #type: DirectoryOrCreate

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jenkine-data-pv
  namespace: ci
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  nfs:
    path: /jenkins-blueocean-data
    server: 192.168.166.136
    
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkine-data-pvc
  namespace: ci
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
  storageClassName: slow

---
kind: Service
apiVersion: v1
metadata:
  labels:
    app: jenkins
  name: jenkins
  namespace: ci
spec:
  ports:
  - port: 8090
    targetPort: 8080
    name: web
  - port: 50000
    targetPort: 50000
    name: agent
  selector:
    app: jenkins
  type: LoadBalancer
```

部署jenkins
```
$ kubectl apply -f jenkins.yaml
```

查看svc
```
$ kubectl get svc -n ci  
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                          AGE
jenkins   LoadBalancer   10.101.147.82   <pending>     8090:31830/TCP,50000:31870/TCP   8h
```

## 配置jenkins
访问master/node:31830 端口，可用访问到jenkins
配置好jenkins后，打开系统管理>插件管理>可选插件
安装Kubernetes plugin
安装好后，系统管理>系统设置>新增一个Kubernetes

配置一下几项即可
```
Name           kubernetes
Kubernetes URL https://kubernetes.default.svc.cluster.local
#用svc的域名访问。端口是jenkins service的端口
Jenkins URL    http://jenkins.ci.svc.cluster.local:8090
```

## 测试

接下来新建一个测试pipeline job

填入以下代码：
```
podTemplate(label: 'slave',  containers: [
    containerTemplate(
            name: 'jnlp',
            image: 'dongamp1990/jenkins-slave-docker-glibc-jdk8-git-maven',
            args: '${computer.jnlpmac} ${computer.name}',
            command: ''
    )
  ]
  ,volumes: [
        persistentVolumeClaim(mountPath: '/home/jenkins', claimName: 'cipvc01', readOnly: false),
        persistentVolumeClaim(mountPath: '/root/.m2/repository', claimName: 'mvnpvc', readOnly: false),
        //hostPathVolume(hostPath: '/jenkins-blueocean-data', mountPath: '/home/jenkins'),
        hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock'),
        hostPathVolume(hostPath: '/tmp/', mountPath: '/tmp/'),
]) 
{    node ('slave') {
        container('jnlp') {
            stage('colne code') {
                try{
                    sh 'git clone --local https://github.com/dongamp1990/demo.git'
                }catch(e){
                    dir('demo/demo'){
                        sh 'git reset --hard && git pull'
                    }
                }
                dir('demo/demo') {
                    sh 'mvn -X clean install'
                    sh 'ls -l ./target'
                }
            }
            stage('dockerinfo') {
                sh 'pwd' 
            }    
        }
    }
}
```

保存后，点立即构建，查看Console Output，不出意外，应该会有maven的编译日志和ls -l ./target的日志输出