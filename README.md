# Kubernetes ConfigMap vs Secret 

   * [Kubernetes ConfigMap vs Secret](#kubernetes-configmap-vs-secret)
      * [场景对比](#场景对比)
      * [使用示例：](#使用示例)
         * [ConfigMap：](#configmap)
            * [创建ConfigMap](#创建configmap)
            * [使用方式：](#使用方式)
               * [1.填充环境变量](#1填充环境变量)
               * [2.设置容器启动命令行参数](#2设置容器启动命令行参数)
               * [3.挂载ConfigMap到文件](#3挂载configmap到文件)
            * [实际使用例子（Redis配置）：](#实际使用例子redis配置)
         * [Secret:](#secret)
            * [创建Secret](#创建secret)
            * [使用方式：](#使用方式-1)
               * [1.挂载到文件](#1挂载到文件)
               * [2.通过环境变量使用](#2通过环境变量使用)
               * [3.拉取镜像](#3拉取镜像)
            * [实际使用例子（挂载证书文件）：](#实际使用例子挂载证书文件)


## 场景对比

- Secret：当你想要存储一些敏感数据时使用Secret，例如（passwords, OAuth tokens, ssh keys, credentials等）
- ConfigMap : 当需要存储一些非敏感配置数据时可以使用ConfigMap，例如应用程序的ini，json等配置文件。
## 使用示例：
### ConfigMap：

#### 创建ConfigMap
``` yaml
kind: ConfigMap
apiVersion: v1
metadata:
  name: example-config
  namespace: default
data:
  example.property.1: hello        #key-value形式，key规则必须满足dns域名规则。value为字符串。
  example.property.2: world
  example.property.file: |-        #配置文件使用方式，直接把文件内容放入value即可
    property.1=value-1
    property.2=value-2
    property.3=value-3
```
#### 使用方式：
##### 1.填充环境变量
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: example-config     #需要使用的ConfigMap名称，必须已经存在
              key: example.property.1  #对应ConfigMap data 的key
  restartPolicy: Never
```
设置结果
```
SPECIAL_LEVEL_KEY=hello
```
##### 2.设置容器启动命令行参数
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh", "-c", "echo $(SPECIAL_LEVEL_KEY) $(SPECIAL_TYPE_KEY)" ]
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: example-config
              key: example.property.1
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: example-config
              key: example.property.2
  restartPolicy: Never
```
运行结果：
```
hello world
```
#####  3.挂载ConfigMap到文件
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: dapi-test-pod
spec:
  containers:
    - name: test-container
      image: gcr.io/google_containers/busybox
      command: [ "/bin/sh","-c","ls -l /etc/config/path/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: example-config
        items:
        - key: example.property.1
          path: path/one/example.property.1
        - key: example.property.2
          path: path/two/example.property.2
        - key: example.property.file
          path: path/file/example.property.file
  restartPolicy: Never
```
创建的文件如下：
```
└── path
    ├── file
    │   └── example.property.file
    ├── one
    │   └── example.property.1
    └── two
        └── example.property.2
        
#cat example.property.1          hello
#cat example.property.2          world
#cat example.property.file  
    property.1=value-1
    property.2=value-2
    property.3=value-3    
```
> **注意：** 
- ConfigMap必须在使用之前被创建，如果引用了一个不存在的configMap，将会导致Pod无法启动
- ConfigMap只能被相同namespace内的应用使用。
#### 实际使用例子（Redis配置）：
例如当我们需要按照如下配置来启动Redis
```
maxmemory 2mb
maxmemory-policy allkeys-lru
```
首先，让我们来创建一个ConfigMap：
```yaml
apiVersion: v1
data:
  redis-config: |
    maxmemory 2mb
    maxmemory-policy allkeys-lru
kind: ConfigMap
metadata:
  name: example-redis-config
  namespace: default
```
下面我们来创建一个Pod来使用它：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
  - name: redis
    image: kubernetes/redis:v1
    env:
    - name: MASTER
      value: "true"
    ports:
    - containerPort: 6379
    volumeMounts:
    - mountPath: /redis-master
      name: config
  volumes:
    - name: config
      configMap:
        name: example-redis-config
        items:
        - key: redis-config
          path: redis.conf     #指定生成的配置文件名称
```
当我们创建完Pod后，进入它：
生成的配置文件如下：
```
redis-master
`-- redis.conf -> ..data/redis.conf
```
我们发现在redis-master 目录下生成了一个文件redis.conf ，对应我们上面path定义的文件名。输出一下redis.conf内容：
```
maxmemory 2mb
maxmemory-policy allkeys-lru
```
下面我们看一下redis的配置：
```
$ kubectl exec -it redis redis-cli
127.0.0.1:6379> CONFIG GET maxmemory
1) "maxmemory"
2) "2097152"
127.0.0.1:6379> CONFIG GET maxmemory-policy
1) "maxmemory-policy"
2) "allkeys-lru"
```
符合我们的预期。
> **注意：** 虽然使用configMap可以很方便的把我们配置文件放入到容器中，但一定注意配置文件的大小，（尽量控制在1M以内）更不能滥用ConfigMap，否则可能会给apiserver和etcd造成较大压力，影响整个集群。

### Secret:
当需要使用一些敏感的配置，比如密码，证书等信息时，建议使用Secret。
#### 创建Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=     
  password: MWYyZDFlMmU2N2Rm
```
> **注意：** Secret 的value必须经过base64， 以上明文为： username: **admin**   password: **1f2d1e2e67df** ，可以直接使用 echo -n "1f2d1e2e67df" | base64   进行base64，或者到[这里](http://tool.chinaz.com/Tools/Base64.aspx)可以进行base64加密解密
#### 使用方式：
#####  1.挂载到文件
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
  namespace: default
spec:
  containers:
  - image: redis
    name: mypod
    volumeMounts:
    - mountPath: /etc/foo
      name: foo
      readOnly: true
  volumes:
  - name: foo
    secret:
      defaultMode: 420      #0644默认文件权限，由于json文件不支持八进制，使用json时应使用十进制
      secretName: mysecret
```
运行结果：在/etc/foo/目录下生成两个文件username和password   
```
foo/
|-- password -> ..data/password
`-- username -> ..data/username
#cat username       admin
#cat password      1f2d1e2e67df
```
> **注意：**  自动更新： 当Secrets被更新时，已经挂载的pod不会立即更新，而要等待kubelete检查周期，kubelet会定期检查secret变化并更新它。
#####  2.通过环境变量使用
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
    - name: mycontainer
      image: redis
      env:
        - name: SECRET_USERNAME
          valueFrom:
            secretKeyRef:
              name: mysecret   #指定secret名称
              key: username    #要使用的key
        - name: SECRET_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysecret
              key: password
```

```
$ echo $SECRET_USERNAME
admin
$ echo $SECRET_PASSWORD
1f2d1e2e67df
```
#####  3.拉取镜像
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: image-test-secret
  namespace: default
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: ew0KCSJhdXRocyI6IHsNCgkJImltYWdlLXRlc3QiOiB7DQoJCQkiYXV0aCI6ICJjbTl2ZERweWIyOTAiLA0KCQkJImVtYWlsIjogIiINCgkJfQ0KCX0NCn0=
```
加密部分的明文为：
```
{
	"auths": {
		"image-test": {
			"auth": "cm9vdDpyb290",   #  密文为"root:rootbase64"的结果
			"email": ""
		}
	}
}
```
部署文件使用如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: foo
  namespace: default
spec:
  containers:
    - name: foo
      image: janedoe/awesomeapp:v1
  imagePullSecrets:
    - name: image-test-secret
```
#### 实际使用例子（挂载证书文件）：
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: client-certs
  namespace: default
data:
  ca.pem: *** #实际使用请替换成经过base64加密后的内容
  kubernetes.pem: ***
  kubernetes-key.pem: ***
```
使用deployment部署：
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis
        ports:
          - containerPort: 8080
        volumeMounts:
        - mountPath: /etc/kubernetes/certs
          name: my-certs
      volumes:
        - name: my-certs
          secret:
            secretName: client-certs
            items:
            - key: ca.pem
              path: ca.pem
            - key: kubernetes.pem
              path: kubernetes.pem
            - key: kubernetes-key.pem
              path: kubernetes-key.pem
      imagePullSecrets:
        - name: image-test-secret
```
