##### 管理工具：ctr
 - 名字：ctr  （containerd CLI）
 - 用法：ctr [global options] command [command options] [arguments…]
 - 描述：ctr 是一个不受支持的用于交互的调试和管理客户机使用容器守护进程。因为它不受支持，选项和操作不能保证向后兼容或容器项目从一个版本到另一个版本都是稳定的。

##### COMMANDS:

| command                           | 说明                             | 操作示例                          |
| :-------                          | :--------------------------      | :-------------------------------- |
| plugins, plugin                   | 提供关于容器插件的信息           |  |
| version                           | 打印客户端和服务器的版本         |  |
| containers, c, container          | 管理容器                         |  |
| content                           | 管理内容                         |  |
| events, event                     | 事件显示容器事件                 |  |
| images, image, i                  | 管理镜像                         | 查看镜像：ctr i list              |
| leases                            | 管理租赁                         |  |
| namespaces, namespace, ns         | 管理命名空间                     |  |
| pprof                             | 为containerd提供golang Pprof输出 |  |
| run                               | 运行容器                         |  |
| snapshots, snapshot               | 管理快照                         |  |
| tasks, t, task                    | 管理任务                         |  |
| install                           | 安装一个新的包	               | 停止容器：ctr -n k8s.io tasks kill -a -s 9 {id}         |
| oci                               | OCI tools                        | |
| shim                              | 与shim直接交互                   | |
| help                              | 帮助                             | |

---
##### GLOBAL OPTIONS:

| options                   |  说明     |
| :------                   |  :------- |
| --debug                   | 打开日志的调试输出| 
| --address value, -a value | containerd的GRPC服务器地址(默认:"/run/k3s/containerd/containerd.sock") [\$CONTAINERD_ADDRESS] |
| --timeout value           | CTR命令的总超时时间(默认值:0) |
| --connect-timeout value   | 连接到容器的超时时间(默认值:0) |
| --namespace value, -n value | 命名空间与命令一起使用(默认:"k8s.io") [\$CONTAINERD_NAMESPACE] |
| --help, -h |                帮助 |
| --version, -v |             打印版本|


---
##### ctr日常操作示例:

| 操作示例 | 命令                                                                                                   | 备注      |
| :------  | :---------------------------------------------------------                                             | :-------- |
| 查看镜像 | ctr i list                                                                                             |           |
| 镜像标记 | ctr -n k8s.io i tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2 k8s.gcr.io/pause:3.2 |           |
| 删除镜像 | ctr -n k8s.io i rm k8s.gcr.io/pause:3.2      |  |
| 拉取镜像 | ctr -n k8s.io i pull -k k8s.gcr.io/pause:3.2 |  |
| 导出镜像 | ctr -n k8s.io i export pause.tar k8s.gcr.io/pause:3.2 | |
| 导入镜像 | ctr -n k8s.io i import pause.tar | 不支持 build,commit 镜像 |
| 运行容器 | ctr -n k8s.io run --null-io --net-host -d –env PASSWORD=\$drone_password –mount type=bind,src=/etc,dst=/host-etc,options=rbind:rw –mount type=bind,src=/root/.kube,dst=/root/.kube,options=rbind:rw \$image sysreport bash /sysreport/run.sh | –null-io: 将容器内标准输出重定向到/dev/null | -d: 当task执行后就进行下一步shell命令,如没有选项,则会等待用户输入,并定向到容器内 | –net-host: 主机网络 |
| 查看容器 | ctr c ls | |
| 容器日志 |          | |
| 停止容器 | ctr -n k8s.io tasks kill -a -s 9 {id}| |
| 删除容器 | ctr -n k8s.io c rm {id} | 先停止容器，再删除 | |
| 修改tag  | ctr -n k8s.io images tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause-amd64:3.2 k8s.gcr.io/pause:3.2 | |
