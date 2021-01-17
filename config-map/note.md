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





