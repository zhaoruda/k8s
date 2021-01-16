# 属性说明
## metadata元数据
* name：pod的名称
* namespace：pod所属的命名空间，默认值为default
* labels：自定义标签列表

## spec
spec指定Pod中容器的详细定义
### containers
containers指定Pod中的容器列表
* name：容器的名称
* image：容器的镜像名称
* imagePullPolicy：镜像拉取策略，默认值为Always。**如果不设置imagePullPolicy或者镜像tag为latest，则系统将默认设置imagePullPolicy=Always**
    * Always：表示每次都尝试重新拉取镜像
    * IfNotPresent：表示如果本地有该镜像，则使用本地的镜像，本地不存在时拉取镜像。
    * Never：表示仅使用本地镜像。
* ports：容器需要暴露的端口列表。
    * name：端口的名称
    * containerPort：容器需要监听的端口号
    * protocol：端口协议，支持TCP和UDP，默认值为TCP
