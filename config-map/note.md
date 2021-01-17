[TOC]
# Config Map
应用程序的运行可能会依赖配置，往往这些配置又会随着需求的变化而发生变化。如果这些配置和应用是在一起的，
那么当配置发生改变时就需要重新构建镜像才能使得配置生效。那么config map可以避免由于配置的改变而重新构建镜像。
ConfigMap用于保持配置数据的键值对，可以保存单个属性，也可以将文件作为配置进行保存。
但是需要注意的是，config map配置的信息是不加密的。
## 创建
### 命令行创建ConfigMap
#### 从key-value字符串创建
- 在dev namespace下创建名为literal-config的config map:
```
kubectl create configmap literal-config --from-literal=name=root --from-literal=password=123 -n dev
```
- 查看user-config的内容：
```
kubectl get configmap user-config -o go-template='{{.data}}' -n dev
```
#### 从文件创建 
- 创建文件：
```
echo -e "name=123\npwd=456" | tee file-config.env
```
- 在dev namespace下创建名为file-config的config map:
```shell script
kubectl create configmap file-config --from-env-file=file-config.env -n dev
```
#### 从目录创建
- 创建目录
```shell script
mkdir config
```
- 创建文件并设置文件内容
```shell script
echo name>config/a
echo pwd>config/b
```
- 在dev namespace下创建名为directory-config的config map:
```shell script
kubectl create configmap directory-config --from-file=config/ -n dev
```
- 查看config map
```shell script
kubectl get configmap directory-config -o go-template='{{.data}}' -n dev
```
从输出结果可以看出，config map的key是文件名，其值为文件的内容。

### Yaml创建ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: yaml-config
  namespace: dev
data:
  user: user
  pwd: "12345"
  data-file: |
    file-key: 890
```

其中**data**指定config map中的配置，如果为简单键值对则可以直接配置，若配置为文件则value需要使用**｜**。

## ConfigMap的使用
ConfigMap有三种在Pod中使用的方式：
* 设置环境变量
* 设置容器命令行参数
* 在Volume中直接挂载文件或目录

但是需要注意的是：
* ConfigMap必须在Pod引用之前创建
* 只能引用同一个命名空间的ConfigMap
* 在Pod中对ConfigMap进行挂载（volumeMount）操作时，在容器内只能挂载为**目录**，无法挂载为**文件**。在挂载到容器内部后，在目录下将包含
ConfigMap定义的每个item，如果在该目录下原来还有其他文件，则容器内的该目录将被挂载的ConfigMap覆盖。如果要求不覆盖，需要额外的处理。

首先来创建ConfigMap
```shell script
kubectl create configmap literal-config --from-literal=name=root --from-literal=password=123 -n dev
```
#### 设置环境变量
Pod的yml文件：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-pod
spec:
  containers:
    - name: nginx-container
      image: nginx:1.19.6
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      env:
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: name
        - name: SPECIAL_TYPE_KEY
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: password
      envFrom:
        - configMapRef:
            name: literal-config
```
在该yaml文件中给出了两种使用configMap的方式：
* 使用env、valueFrom来显示指出使用ConfigMap的哪个配置
* 使用envFrom来引用ConfigMap中的所有配置

可以使用如下命令的输出结果来验证configMap在pod中被成功引用：
```shell script
kubectl exec config-map-pod -n dev env
```

#### 使用 volume 将 ConfigMap 作为文件或目录直接挂载
##### ConfigMap直接挂载到Pod到目录中
将ConfigMap直接挂载到Pod到目录时，ConfigMap中到每个key-value都会生成一个文件，文件名为key，文件内容为value。
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-pod-volume
  namespace: dev
spec:
  containers:
    - name: nginx-container
      image: nginx:1.19.6
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config/
  volumes:
    - name: config-volume
      configMap:
        name: literal-config
```
**该方式挂载时会将/etc/config目录中的内容清空，然后再进行挂载。**

##### ConfigMap中到key载到Pod的一个相对目录中
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-pod-volume
  namespace: dev
spec:
  containers:
    - name: nginx-container
      image: nginx:1.19.6
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: literal-config
        items:
          - key: name
            path: subPath
```
**该方式挂载时也会将/etc/config目录中的内容清空，然后再进行挂载**
和上述方式的区别：
* 该方式必须在volumes中指定path，也就是挂载的相对目录。
* 该方式只会挂载指定的key，其他的key不会进行挂载

##### 使用 subpath 将 ConfigMap 作为单独的文件挂载到目录
上述的两种方式进行挂载时都会将挂载目录清空。如果不想对Pod中对目录进行覆盖，只是将ConfigMap中对每个key按照文件对方式挂载到目录中，可以使用
subpath参数。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-map-pod-volume
  namespace: dev
spec:
  containers:
    - name: nginx-container
      image: nginx:1.19.6
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      volumeMounts:
        - name: config-volume
          mountPath: /usr/subPath
          subPath: subPath
  volumes:
    - name: config-volume
      configMap:
        name: literal-config
        items:
          - key: name
            path: subPath
```
**使用该方式时volumeMounts对subPath需要和key对应的path保持一致。且该方式在configMap中的值发生改变时无法进行实时更新**
