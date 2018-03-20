---
title: Kubernetes Java Client使用
tags: [kubernetes]
---
## 前言
使用Kubernetes 加Jenkins CICD，那么问题来了，Jenkins里面怎么部署Kubernetes应用呢。曾经在Jenkins上搜索过几个插件，其中Kubernetes Continuous Deploy没搞懂怎么用，咨询了一技术哥们说：本质就是配置认证， 调用k8s接口实现。在他的指点下，使用Kubernetes client去调用Kubernetes接口z
k8s java client github地址：https://github.com/kubernetes-client/java。


下面使用springboot + k8s client搭建的一个k8s-api项目

## 创建接口调用token
使用RBAC（基于角色的访问控制)调用接口，关于RBAC阅读以下文档
官网文档：https://v1-8.docs.kubernetes.io/docs/admin/authorization/rbac/
中文翻译文档：https://jimmysong.io/kubernetes-handbook/concepts/rbac.html

首先创建一个ServiceAccount，再绑定ClusterRole角色的cluster-admin权限，获得token，备client调接口用。
deployuser-token.yaml文件
```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
 name: deploy-user
 annotations:
  rbac.authorization.kubernetes.io/autoupdate: "true"
roleRef:
 kind: ClusterRole
 name: cluster-admin
 apiGroup: rbac.authorization.k8s.io
subjects:
 - kind: ServiceAccount
   name: deploy-user
   namespace: kube-system
   
---
apiVersion: v1
kind: ServiceAccount
metadata:
 name: deploy-user
 namespace: kube-system
 labels:
  kubernetes.io/cluster-service: "true"
  addonmanager.kubernetes.io/mode: Reconcile
```

```
$ kubectl apply -f cat deployuser-token.yaml
```

查看用户秘钥
```
$ kubectl get secret -n kube-system|grep deploy-user
NAME                                     TYPE                                  DATA      AGE
deploy-user-token-5g6w6                  kubernetes.io/service-account-token   3         1d
```

#查看deploy-user账户秘钥详情
```
$ kubectl describe secret deploy-user-token-5g6w6  -n kube-system

Name:         deploy-user-token-5g6w6
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name=deploy-user
              kubernetes.io/service-account.uid=0de0540e-2b24-11e8-841c-080027381e88

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:  TOKEN
```

## 新建一个Maven project
pom.xml文件
```
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>

	<groupId>org.kevin</groupId>
	<artifactId>k8s-api</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<packaging>jar</packaging>

	<name>k8s-api</name>
	<url>http://maven.apache.org</url>

	<properties>
		<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
	</properties>
	<parent>
	    <groupId>org.springframework.boot</groupId>
	    <artifactId>spring-boot-starter-parent</artifactId>
	    <version>1.5.10.RELEASE</version>
	</parent>
 

	<dependencies>
		<dependency>
			<groupId>io.kubernetes</groupId>
			<artifactId>client-java</artifactId>
			<version>1.0.0-beta1</version>
		</dependency>
		
		<dependency>
	        <groupId>org.springframework.boot</groupId>
	        <artifactId>spring-boot-starter-web</artifactId>
	    </dependency>
	</dependencies>
	
	<build>
		<finalName>app</finalName>
		<resources>
			<resource>
				<directory>src/main/resources</directory>
				<includes>
					<include>**/*</include>
					<include>**/*/*</include>
				</includes>
			</resource>
		</resources>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				<artifactId>maven-compiler-plugin</artifactId>
				<configuration>
					<source>1.7</source>
					<target>1.7</target>
				</configuration>
			</plugin>
	
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
</project>

```
重点在
```
		<dependency>
			<groupId>io.kubernetes</groupId>
			<artifactId>client-java</artifactId>
			<version>1.0.0-beta1</version>
		</dependency>
```
因为集群是k8s 1.8的，client需要使用1.0.0-beta1版本

### 配置
新建application.properties配置以下内容
```
server.port=8089

#deploy-user的token，可以配置在配置文件，也可以在声明pod的设置env引用secret
#token=TOKEN
```

### springboot启动类
springboot启动类，包括初始化ApiClient。
KUBERNETES_SERVICE_HOST 使用k8s部署后pod的env环境变量，k8s apiserver地址
KUBERNETES_SERVICE_PORT k8s apiserver端口
token 是调接口凭证。
```
@SpringBootApplication
public class App {
	public static void main(String[] args) throws IOException, ApiException {
		SpringApplication.run(App.class, args);
	}
	
	@Value("${KUBERNETES_SERVICE_HOST}")
	private String host;
	@Value("${KUBERNETES_SERVICE_PORT}")
	private String port;
	@Value("${token}")
	private String token;
	
	@Bean
	public ApiClient getClient() {
		try {
			ApiClient client = new ApiClient();
			client.setBasePath("https://" + host + ":" + port);
			client.setApiKeyPrefix("bearer");
			client.setApiKey(token);
			//忽略ssl验证，不然java会报错。
			client.setVerifyingSsl(false);
			Configuration.setDefaultApiClient(client);
			return client;
		} catch (Exception e) {
			e.printStackTrace();
			System.out.println("初始化ApiClient错误");
			System.exit(1);
			return null;
		}
	}
}
```

### 使用client调用k8s api
详细API接口文档：https://github.com/kubernetes-client/java/blob/master/kubernetes/README.md
Controller.java
```
@RestController
public class Controller {

	@RequestMapping(method = RequestMethod.GET, path = "list_pod_for_all_namespace")
	@ResponseBody
	public RespObject listPod() {
		CoreV1Api api = new CoreV1Api();
		V1PodList list;
		try {
			list = api.listPodForAllNamespaces(null, null, null, null, null, null, null, null, null);
			List<String> names = new ArrayList<>();
			for (V1Pod item : list.getItems()) {
				names.add(item.getMetadata().getName());
			}
			return new RespObject(names);
		} catch (ApiException e) {
			e.printStackTrace();
			return new RespObject(e.getMessage(), 1);
		}
	}
}
```

RespObject.java
```
public class RespObject {
	private Object result;
	private int code;
	
	public RespObject(Object res) {
		this.code = 0;
		this.result = res;
	}
	
	public RespObject(Object res, int code) {
		this.code = code;
		this.result = res;
	}

	public Object getResult() {
		return result;
	}

	public void setResult(Object result) {
		this.result = result;
	}

	public int getCode() {
		return code;
	}

	public void setCode(int code) {
		this.code = code;
	}
}
```

## 编译、部署到k8s

### 制作docker镜像
k8s-api.dockerfile文件
```
FROM java:8-jre-alpine
WORKDIR /opt/
COPY app.jar /opt/app.jar
CMD ["/bin/sh", "-c", "java -jar app.jar"]
```
```
$ docker build -t k8s-api-demo -f k8s-api.dockerfile .
```

### 部署到k8s
至此Java代码差不多了，打成jar包，docker build成镜像，用k8s部署，最后测试下

k8s-api.yaml文件
```
---
kind: Deployment
apiVersion: apps/v1beta2
metadata:
 name: k8s-api-demo
 labels:
  app: k8s-api-demo
 namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: k8s-api-demo
  template:
    metadata:
      labels:
        app: k8s-api-demo
    spec:
      containers:
      - name: k8s-api-demo
        image: k8s-api-demo
        #配置env
        env: 
        - name: token
          valueFrom: 
            #token的值引用自secret
            secretKeyRef: 
              # 通过kubectl get secret -n kube-system|grep deploy-user 得到名称
              name: deploy-user-token-5g6w6
              key: token
              
---
kind: Service
apiVersion: v1
metadata:
 name: k8s-api-demo
 labels:
  app: k8s-api-demo
 namespace: kube-system
spec:
 ports:
 - port: 8099
   targetPort: 8089
   nodePort: 32180
 selector:
  app: k8s-api-demo
 type: NodePort 

```

```
$ kubectl apply -f k8s-api.yaml
$ kubectl get svc -o wide -n kube-system |grep k8s-api-demo
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE       SELECTOR
k8s-api-demo       NodePort    10.100.62.191    <none>        8099:32180/TCP      1d        app=k8s-api-demo
```

通过访问master/node:32180端口即可访问到该项目
访问http://master/node:32180/list_pod_for_all_namespace
就可以返回所有namespace的pod name列表，结果如下：
```
{
    "result": [
        "jenkins-74bbcdfd9-84hbr",
        "adminserver-7c8c47cb95-ljf6z",
        "dapi-test-pod",
        "jobservice-7d584c99fb-2g2xq",
        "k8s-api-6ccdbcb947-k6cwm",
        "mysql-c87996b55-dhnjl",
        "nfs-7b689fb468-p8pfm",
        "nginx-75f4785b7-2t4kr",
        "registry-c5f6f84dd-kk46k",
        "ui-78599c46-8xpx5",
        "calico-etcd-68mzw",
        "calico-kube-controllers-6ff88bf6d4-57bqw",
        "calico-node-6q2r7",
        "calico-node-dx6cm",
        "default-http-backend-n9st8",
        "etcd-ubuntu-k8s",
        "kube-apiserver-ubuntu-k8s",
        "kube-controller-manager-ubuntu-k8s",
        "kube-dns-545bc4bfd4-bgzg8",
        "kube-proxy-77rlm",
        "kube-proxy-nffh7",
        "kube-scheduler-ubuntu-k8s",
        "kubernetes-dashboard-76894548fd-kkhrj",
        "nginx-ingress-controller-4kkph",
        "nginx-ingress-controller-h9cpd",
        "nginx-75f4785b7-lq4p7",
        "nginx-75f4785b7-qt5sx",
        "nginx-75f4785b7-v4lxg"
    ],
    "code": 0
}
```

## 在jenkins里面使用
可以用pipeline用HttpRequest调用
```
def resp;
def url = "http://192.168.10.93:32180/list_pod_for_all_namespace"
try {
    resp = httpRequest httpMode: 'GET', url: "$url"
    println resp.result
}catch(e){
}

```

文章Java代码：https://github.com/dongamp1990/k8s-api-demo.git