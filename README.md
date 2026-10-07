# my-k8s-app-deploy

GitOps 部署仓库。仓库保存 Helm chart 和 Argo CD Application 模板；真实 Harbor 地址、凭据只存在于 GitHub Secrets 和集群内，绝不入库。

> 本仓库是**公开仓库**，提交前务必确认没有真实内网 IP、主机名或凭据。

## 目录

- application.yaml：Argo CD Application 模板（含 `${DEPLOY_REPO_URL}`）。
- charts/my-app：应用 Helm chart。
- charts/my-app/values.yaml：副本数、镜像、资源和服务配置，镜像地址是占位符。
- charts/my-app/templates：Deployment 和 Service 模板。
- test-pull.yaml：可选的 Harbor 拉取测试 Pod（镜像地址是占位符）。

## 模板变量

| 变量 | 出现位置 | 由谁注入 |
| --- | --- | --- |
| `${DEPLOY_REPO_URL}` | application.yaml | 本地 `envsubst` 渲染，只渲染一次用于 bootstrap |
| `${HARBOR_REGISTRY}` / `${HARBOR_PROJECT}` | values.yaml、test-pull.yaml | **不渲染**；values.yaml 的真实值由 Argo CD helm.parameters 在集群侧注入 |

## 真实镜像地址怎么注入

values.yaml 里的 `image.repository` 是占位符，不会被渲染。Argo CD Application 通过 helm parameters 覆盖它
（等价于 `helm --set image.repository=...`，优先级高于 values.yaml），真实值只存在于集群里：

    kubectl -n argocd patch application my-app --type=merge -p \
      '{"spec":{"source":{"helm":{"parameters":[{"name":"image.repository","value":"<registry>/<project>/my-app"}]}}}}'

bootstrap 之后，CI 只改 `values.yaml` 的 `image.tag`，`image.repository` 始终由 parameters 提供，二者互不干扰。

## Bootstrap（只在首次搭建集群时做一次）

    cd ~/projects/my-k8s-app-deploy
    export DEPLOY_REPO_URL="https://github.com/OWNER/DEPLOY_REPOSITORY.git"

    envsubst < application.yaml > /tmp/application.rendered.yaml
    kubectl apply -f /tmp/application.rendered.yaml

    # 随后立刻注入镜像仓库真实地址
    kubectl -n argocd patch application my-app --type=merge -p \
      '{"spec":{"source":{"helm":{"parameters":[{"name":"image.repository","value":"'"$HARBOR_REGISTRY"'/'"$HARBOR_PROJECT"'/my-app"}]}}}}'

/tmp 下的渲染文件不要提交到 Git。CI 从不 apply Application，也不做 envsubst —— 这是设计。

## Harbor imagePullSecret

集群中需要名为 harbor-secret 的 Secret。不要把用户名密码写入 YAML：

    kubectl create namespace my-app
    kubectl -n my-app create secret docker-registry harbor-secret --docker-server="$HARBOR_REGISTRY" --docker-username="$HARBOR_USERNAME" --docker-password="$HARBOR_PASSWORD"

HTTP Harbor 还要求 Kubernetes 节点上的 containerd 或 Docker 配置 insecure registry。

## 提交和同步

    git status
    git diff
    git add -A
    git commit -m "describe the deployment change"
    git push origin main

应用仓库 workflow 会自动更新 values.yaml 的 image.tag。Argo CD 根据 automated syncPolicy 同步集群。

提交前检查：

    git grep -n -I -e "172\." -e "password" -e "token" HEAD
