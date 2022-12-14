# Argo CD Guide

[Argo CD](https://argo-cd.readthedocs.io/) 是基于 [Kubernetes](https://kubernetes.io/) 的申明式、GitOps 持续部署工具。

## 安装
首先，你需要有一套 [Kubernetes](https://github.com/kubernetes/kubernetes/) 环境。下面的工具可以帮助你快速按照好一套 Kubernetes 环境：

> 推荐使用 [hd](https://github.com/LinuxSuRen/http-downloader) 安装下面的工具
>
> 安装 `hd` 的命令为：`curl https://linuxsuren.github.io/tools/install.sh|bash`

| 工具 | 工具安装 |使用 |
|---|---|---|
| [k3d](https://k3d.io/) | `hd i k3d` | `k3d cluster create` |
| [kubekey](https://github.com/kubesphere/kubekey) | `hd i kk` | `kk create cluster` |
| [minikube](https://github.com/kubernetes/minikube) | `hd i minikube` | `minikube start` |

当 Kubernetes 环境就绪后，就可以通过下面的命令会在命名空间（`argo`）下安装最新版本的 `Argo CD`：

```shell
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

如果你的环境访问 GitHub 时有网络问题，可以使用下面的命令来安装：

```shell
docker run -it --rm -v /root/.kube/:/root/.kube --network host ghcr.io/linuxsuren/argo-cd-guide:master
```

推荐使用的工具：

||||
|---|---|---|
| [k9s](https://k9scli.io/) | `hd i k9s` | K9s is a terminal based UI to interact with your Kubernetes clusters. |
| `argocd` | `hd i argoproj/argo-cd` |  |

## 一个简单的示例
执行下面的命令后

```shell
cat <<EOF | kubectl apply -n argocd -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: learn-pipeline-go
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    repoURL: https://gitee.com/devops-ws/learn-pipeline-go   # 示例工程
    path: kustomize                                           # 从该目录下查找 Kubernetes 文件
    targetRevision: HEAD
    kustomize:
      namePrefix: foo
EOF
```

## 概念
TODO

## 模板工具
TODO

## Webhook
TODO

```
https://ip:port/api/webhook
```

## 配置管理插件
配置管理工具（Config Management Plugin，CMP）使得 Argo CD 可以支持 Helm、Kustomize 以外的（可转化为 Kubernetes 资源）格式。

例如：我们可以将 GitHub Actions 的配置文件转为 Argo Workflows 的文件，从而实现在不了解 Argo Workflows 的 `WorkflowTemplate` 写法的前提下，也可以把 Argo Workflows 作为 CI 工具。

> 下面的例子中需要用到 Argo Workflows，请自行安装，或查看[这篇中文教程](https://github.com/LinuxSuRen/argo-workflows-guide)。

我们只需要将插件作为 sidecar 添加到 `argocd-repo-server` 即可。下面是 sidecar 的配置：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-repo-server
  namespace: argocd
spec:
  template:
    spec:
      containers:
      - args:
        - --loglevel
        - debug
        command:
        - /var/run/argocd/argocd-cmp-server
        image: ghcr.io/linuxsuren/github-action-workflow:master
        imagePullPolicy: IfNotPresent
        name: tool
        resources: {}
        securityContext:
          runAsNonRoot: true
          runAsUser: 999
        volumeMounts:
        - mountPath: /var/run/argocd
          name: var-files
        - mountPath: /home/argocd/cmp-server/plugins
          name: plugins
```

然后，再添加如下 Argo CD Application 后，我们就可以看到已经有多个 Argo Workflows 被创建出来了。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: yaml-readme
  namespace: argocd
spec:
  destination:
    namespace: default
    server: https://kubernetes.default.svc
  project: default
  source:
    path: .github/workflows/                            # It will generate multiple Argo CD application manifests 
                                                        # base on YAML files from this directory.
                                                        # Please make sure the path ends with slash.
    plugin: {}                                          # Argo CD will choose the corresponding CMP automatically
    repoURL: https://gitee.com/linuxsuren/yaml-readme   # a sample project for discovering manifests
    targetRevision: HEAD
  syncPolicy:
    automated:
      selfHeal: true
```

由于用到 PVC 作为 Pod 之间的共享存储，我们还需要安装对应的依赖。如果是测试环境，可以安装 [OpenEBS](https://openebs.io/docs/user-guides/installation)。并设置其中的 为[默认存储卷](https://kubernetes.io/zh-cn/docs/tasks/administer-cluster/change-default-storage-class/)：

```shell
kubectl patch storageclass openebs-hostpath -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

如果需要用到 Git 凭据的话，可以通过下面的命令拿到：

```shell
kubectl create secret generic git-secret --from-file=id_rsa=/root/.ssh/id_rsa --from-file=known_hosts=/root/.ssh/known_hosts --dry-run=client -oyaml
```

这一点对于 Argo Workflows 落地为持续集成（CI）工具时，非常有帮助。如果您觉得 GitHub Actions 的语法足够清晰，那么，可以直接使用上面的插件。或者，您希望能定义出更简单的 YAML，也可以自行实现插件。插件的核心逻辑就是将目标文件（集）转为 Kubernetes 的 YAML 文件，在这里就是 `WorkflowTemplate`。

如果再发散性地思考下，我们也可以通过自定义格式的 YAML（或 JSON 等任意格式）文件转为 Jenkins 可以识别的 Jenkinsfile，或其他持续集成工具的配置文件格式。

## 凭据管理
可以通过下面的命令，生成一个加密后的 Secret：
```shell
kubectl create secret generic test --from-literal=username=admin --from-literal=password=admin --dry-run=client -oyaml -n default | kubeseal -oyaml
```

下面是生成 Docker 认证信息的命令：
```shell
kubectl create secret docker-registry harbor --docker-server='10.121.218.184:30002' \
  --docker-username=admin --docker-password=password \
  --dry-run=client -oyaml -n default | kubeseal -oyaml
```

## 单点登录
TODO

## 组件介绍
TODO

## FAQ

| 组件 | 日志 | 方案 |
|---|---|---|
| `argocd-repo-server` | `gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-k ││ ey-recipe1158238699 failed exit status 2` | 删除 `seccompProfile` |

## 其他资料
* [Argo CD 视频教程](https://www.bilibili.com/video/BV17F411h7Zh/)
