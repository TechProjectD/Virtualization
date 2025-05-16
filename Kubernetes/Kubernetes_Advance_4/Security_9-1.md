## 配置对多集群的访问

- 下面将展示如何使用配置文件来配置对多个集群的访问。 在将集群、用户和上下文定义在一个或多个配置文件中之后，用户可以使用 kubectl config use-context 命令快速地在集群之间进行切换。

  - **说明:用于配置集群访问的文件有时被称为 kubeconfig 文件。 这是一种引用配置文件的通用方式，并不意味着存在一个名为 kubeconfig 的文件。**


##### 定义集群、用户和上下文
- 假设用户有两个集群，一个用于正式开发工作，一个用于其它临时用途（scratch）。 在 development 集群中，前端开发者在名为 frontend 的名字空间下工作， 存储开发者在名为 storage 的名字空间下工作。在 scratch 集群中， 开发人员可能在默认名字空间下工作，也可能视情况创建附加的名字空间。 访问开发集群需要通过证书进行认证。 访问其它临时用途的集群需要通过用户名和密码进行认证。

- 创建名为 config-exercise 的目录。在 config-exercise 目录中，创建名为 config-demo 的文件，其内容为：

```yaml
apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
  name: development
- cluster:
  name: scratch

users:
- name: developer
- name: experimenter

contexts:
- context:
  name: dev-frontend
- context:
  name: dev-storage
- context:
  name: exp-scratch
```
- 配置文件描述了集群、用户名和上下文。config-demo 文件中含有描述两个集群、 两个用户和三个上下文的框架。
  - 进入 config-exercise 目录。输入以下命令，将集群详细信息添加到配置文件中：

```shell
kubectl config --kubeconfig=config-demo set-cluster development --server=https://1.2.3.4 --certificate-authority=fake-ca-file
kubectl config --kubeconfig=config-demo set-cluster scratch --server=https://5.6.7.8 --insecure-skip-tls-verify
```

- 将用户详细信息添加到配置文件中：

```shell
kubectl config --kubeconfig=config-demo set-credentials developer --client-certificate=fake-cert-file --client-key=fake-key-seefile
kubectl config --kubeconfig=config-demo set-credentials experimenter --username=exp --password=some-password

#删除:
kubectl --kubeconfig=config-demo config unset users.<name>      #删除用户
kubectl --kubeconfig=config-demo config unset clusters.<name>   #删除集群
kubectl --kubeconfig=config-demo config unset contexts.<name>   #删除上下文
```

- 将上下文详细信息添加到配置文件中：

```shell
kubectl config --kubeconfig=config-demo set-context dev-frontend --cluster=development --namespace=frontend --user=developer
kubectl config --kubeconfig=config-demo set-context dev-storage --cluster=development --namespace=storage --user=developer
kubectl config --kubeconfig=config-demo set-context exp-scratch --cluster=scratch --namespace=default --user=experimenter
```

打开 config-demo 文件查看添加的详细信息。也可以使用 config view 命令进行查看：
`kubectl config --kubeconfig=config-demo view`
输出展示了两个集群、两个用户和三个上下文：

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
- cluster:
    insecure-skip-tls-verify: true
    server: https://5.6.7.8
  name: scratch
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
- context:
    cluster: development
    namespace: storage
    user: developer
  name: dev-storage
- context:
    cluster: scratch
    namespace: default
    user: experimenter
  name: exp-scratch
current-context: ""
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
- name: experimenter
  user:
    password: some-password
    username: exp

#其中的 fake-ca-file、fake-cert-file 和 fake-key-file 是证书文件路径名的占位符。 你需要更改这些值，使之对应你的环境中证书文件的实际路径名。
#有时你可能希望在这里使用 BASE64 编码的数据而不是一个个独立的证书文件。 如果是这样，你需要在键名上添加 -data 后缀。例如， certificate-authority-data、client-certificate-data 和 client-key-data。
```

- 每个上下文包含三部分（集群、用户和名字空间），例如， dev-frontend 上下文表明：使用 developer 用户的凭证来访问 development 集群的 frontend 名字空间。

- 设置当前上下文：
`kubectl config --kubeconfig=config-demo use-context dev-frontend`

- 现在当输入 kubectl 命令时，相应动作会应用于 dev-frontend 上下文中所列的集群和名字空间， 同时，命令会使用 dev-frontend 上下文中所列用户的凭证。

```yaml
#使用 --minify 参数，来查看与当前上下文相关联的配置信息。

kubectl config --kubeconfig=config-demo view --minify
#输出结果展示了 dev-frontend 上下文相关的配置信息：
apiVersion: v1
clusters:
- cluster:
    certificate-authority: fake-ca-file
    server: https://1.2.3.4
  name: development
contexts:
- context:
    cluster: development
    namespace: frontend
    user: developer
  name: dev-frontend
current-context: dev-frontend
kind: Config
preferences: {}
users:
- name: developer
  user:
    client-certificate: fake-cert-file
    client-key: fake-key-file
```

```shell
#现在假设用户希望在其它临时用途集群中工作一段时间。

#将当前上下文更改为 exp-scratch：
kubectl config --kubeconfig=config-demo use-context exp-scratch
#现在你发出的所有 kubectl 命令都将应用于 scratch 集群的默认名字空间。 同时，命令会使用 exp-scratch 上下文中所列用户的凭证。

#查看更新后的当前上下文 exp-scratch 相关的配置：

kubectl config --kubeconfig=config-demo view --minify
#最后，假设用户希望在 development 集群中的 storage 名字空间下工作一段时间。

#将当前上下文更改为 dev-storage：
kubectl config --kubeconfig=config-demo use-context dev-storage
#查看更新后的当前上下文 dev-storage 相关的配置：
kubectl config --kubeconfig=config-demo view --minify
```

```yaml
#创建第二个配置文件
#在 config-exercise 目录中，创建名为 config-demo-2 的文件，其中包含以下内容：
apiVersion: v1
kind: Config
preferences: {}

contexts:
- context:
    cluster: development
    namespace: ramp
    user: developer
  name: dev-ramp-up
#上述配置文件定义了一个新的上下文，名为 dev-ramp-up。
```